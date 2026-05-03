Moderate Facebook comments with AI and send reports to Slack & Telegram

https://n8nworkflows.xyz/workflows/moderate-facebook-comments-with-ai-and-send-reports-to-slack---telegram-12028


# Moderate Facebook comments with AI and send reports to Slack & Telegram

## 1. Workflow Overview

**Purpose:** This workflow periodically retrieves recent Facebook Page comments, moderates them with OpenAI (intent, toxicity, spam, bad words), stores normalized moderation logs in Supabase, aggregates summary statistics, and sends a concise report to **Slack** and **Telegram**.

**Typical use cases:**
- Community management and content moderation at scale
- Team visibility on comment health (toxicity/spam trends) without logging into Facebook
- Building an auditable moderation history in a database (Supabase)

### 1.1 Trigger & Comment Collection
Runs every 6 hours, fetches comments via Facebook Graph API.

### 1.2 Comment Preparation & AI Moderation
Prepares items and calls OpenAI to return strict JSON moderation results per comment.

### 1.3 Normalization, Flagging & Storage
Cleans/parses AI output, computes a `flagged` boolean, and stores each comment’s moderation record in Supabase.

### 1.4 Aggregation & Notifications
Aggregates the run into counts and samples; posts the summary to Slack and Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Data Collection
**Overview:** Starts the workflow on a schedule and pulls Facebook Page comments through an HTTP request.  
**Nodes involved:** Scheduled Workflow Trigger → Fetch Facebook Page Comments

#### Node: Scheduled Workflow Trigger
- **Type / role:** Cron Trigger (`n8n-nodes-base.cron`) — scheduled entry point.
- **Configuration (interpreted):** Runs **every 6 hours** (`everyX` with value `6`).
- **Inputs / outputs:** No inputs (trigger). Output goes to **Fetch Facebook Page Comments**.
- **Edge cases / failures:**
  - n8n timezone and Cron interpretation can affect run timing (server timezone vs expected).
  - If workflow is inactive, it won’t trigger (workflow is currently `active: false`).

#### Node: Fetch Facebook Page Comments
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Facebook Graph API.
- **Configuration (interpreted):**
  - **URL:** `https://graph.facebook.com/v22.0/{{ Your Page ID}}/comments`
  - `continueOnFail: true` (workflow continues even if the request fails).
  - No explicit method shown; HTTP Request defaults to **GET** unless changed.
  - Missing in JSON: **access token** configuration (must be added via query param or headers).
- **Inputs / outputs:** Input from Cron. Output to **Prepare Comment Items for Processing**.
- **Key requirements:**
  - A valid Facebook **Page ID** and a **Page access token** with permissions to read comments.
- **Edge cases / failures:**
  - Auth errors (expired token, missing permissions) → Facebook returns 400/401; with `continueOnFail`, downstream nodes may receive an error-shaped item or empty data.
  - Pagination: Graph API typically returns paged results; this node as configured does not handle pagination, so only the first page may be processed.
  - Response shape mismatch: downstream assumes comments have a `message` field; some comment objects may not include `message` (attachments-only, deleted, etc.).

**Sticky note content applied to this block:**
- “# Workflow Trigger & Data Collection … triggers every six hours and fetches recent comments…”

---

### Block 2 — Comment Preparation & AI Moderation
**Overview:** Takes fetched comment data, ensures items are shaped for per-comment processing, then sends each comment to OpenAI with instructions to return strict JSON moderation results.  
**Nodes involved:** Prepare Comment Items for Processing → AI-Based Comment Moderation

#### Node: Prepare Comment Items for Processing
- **Type / role:** Code (`n8n-nodes-base.code`) — transforms incoming items.
- **Configuration (interpreted):**
  - Loops through `$input.all()` and sets `item.json.myNewField = 1` for each item, then returns all items unchanged otherwise.
  - As written, it **does not** actually split a Facebook response array into individual comment items.
- **Inputs / outputs:** Input from **Fetch Facebook Page Comments**. Output to **AI-Based Comment Moderation**.
- **Key expressions / variables:** `$input.all()`.
- **Edge cases / failures (important):**
  - Facebook Graph responses are typically `{ data: [ ...comments ] }`. If the HTTP node returns a **single item** containing `json.data` array, this code will only add `myNewField` to that single wrapper item, not create one item per comment.  
  - Downstream OpenAI prompt references `{{$json.message}}`; if the item is the wrapper and not a comment object, `message` will be undefined.

#### Node: AI-Based Comment Moderation
- **Type / role:** OpenAI via LangChain (`@n8n/n8n-nodes-langchain.openAi`) — performs moderation analysis.
- **Configuration (interpreted):**
  - **Model:** `gpt-4.1-mini`
  - Prompt requests **STRICT JSON** with:
    - `comment_text` (must match input exactly)
    - `intent` (positive|neutral|negative)
    - `toxicity_score` (0–100)
    - `spam` (boolean)
    - `bad_words` (boolean)
    - `summary` (short explanation)
  - Uses `{{$json.message}}` as the comment text.
- **Inputs / outputs:** Input from **Prepare Comment Items…**. Output to **Normalize AI Moderation Results**.
- **Credential requirements:** OpenAI API credential must be configured.
- **Edge cases / failures:**
  - If `message` is missing/undefined, the model may produce nonsense or an invalid JSON.
  - The model may still wrap JSON in markdown fences despite instructions; downstream normalization attempts to strip fences.
  - Rate limits / timeouts for large batches (many comments every 6 hours).
  - Cost control: one OpenAI call per comment item (unless data is not split correctly, in which case it might be one call per run, but with missing `message`).

**Sticky note content applied to this block:**
- “# Comment Preparation & AI Moderation … prepares … sends each comment to OpenAI…”

---

### Block 3 — Normalize, Flag & Store Results
**Overview:** Parses the AI response into structured fields, calculates a `flagged` status, timestamps the record, and writes each moderation record to Supabase.  
**Nodes involved:** Normalize AI Moderation Results → Store Moderation Logs in Database

#### Node: Normalize AI Moderation Results
- **Type / role:** Code (`n8n-nodes-base.code`) — cleaning + JSON parsing + field mapping.
- **Configuration (interpreted):**
  - `cleanJson()` removes ```json fences and trims.
  - For each incoming item:
    - Reads `item.json.output?.[0]`
    - Extracts AI text at `outputBlock?.content?.[0]?.text`
    - `JSON.parse()` after cleaning; if parsing fails → throws `Invalid AI JSON`
  - Emits items shaped as:
    - `comment_id: outputBlock.id`
    - `comment_text, intent, toxicity_score, spam, bad_words, summary` (from AI JSON)
    - `flagged`: `toxicity_score > 60 OR spam OR bad_words`
    - `comment_created_at`: current ISO timestamp (note: this is *processing time*, not Facebook creation time)
- **Inputs / outputs:** Input from **AI-Based Comment Moderation**. Output to **Store Moderation Logs in Database**.
- **Key expressions / variables:**
  - `item.json.output?.[0]`, `outputBlock?.content?.[0]?.text`
- **Edge cases / failures:**
  - Output schema dependency: This assumes the OpenAI node returns a specific structure (`output[0].content[0].text`). If n8n/OpenAI node output changes, parsing breaks.
  - If `continueOnFail` earlier produced error items or empty items, `aiText` may be missing; code currently **skips** those items (`if (!aiText) continue;`) silently (no alerting).
  - `comment_id` assignment likely incorrect: `outputBlock.id` is probably the OpenAI “output” block id, not the Facebook comment ID. If you need the real comment ID, it should be carried from the original comment item.

#### Node: Store Moderation Logs in Database
- **Type / role:** Supabase (`n8n-nodes-base.supabase`) — persists records.
- **Configuration (interpreted):**
  - **Table:** `facebook_comment_moderation`
  - **Data mapping:** `autoMapInputData` (writes all incoming JSON fields into columns with matching names)
- **Inputs / outputs:** Input from **Normalize AI Moderation Results**. Output to **Aggregate Moderation Statistics**.
- **Credential requirements:** Supabase credential (URL + service role key or appropriate API key).
- **Edge cases / failures:**
  - Table schema mismatch: if columns don’t exist (e.g., `flagged`, `toxicity_score`), inserts fail.
  - Data types: ensure `toxicity_score` numeric, `spam/bad_words/flagged` boolean.
  - Row volume: frequent runs + many comments may require indexing (e.g., on `comment_created_at`, `flagged`).

**Sticky note content applied to this block:**
- “# Normalize, Flag & Store Results … cleans and normalizes … stores … in Supabase…”

---

### Block 4 — Aggregate Statistics & Team Notifications
**Overview:** Computes totals and category counts for the run, generates a Slack/Telegram-friendly message, then sends it to Slack and Telegram.  
**Nodes involved:** Aggregate Moderation Statistics → (Send Moderation Reports to Team via Slack + Send Moderation Reports to Telegarm)

#### Node: Aggregate Moderation Statistics
- **Type / role:** Code (`n8n-nodes-base.code`) — aggregation + message formatting.
- **Configuration (interpreted):**
  - Reads all items (`$input.all()`) and computes:
    - `total`, counts of `positive/neutral/negative`
    - `flagged`, `spam`, `toxic` (`toxicity_score > 60`)
    - `samples`: up to 5 flagged comment texts
  - Produces a **single output item** containing stats + `slack_message` in Slack mrkdwn style.
- **Inputs / outputs:** Input from **Store Moderation Logs in Database**. Output splits to Slack + Telegram.
- **Edge cases / failures:**
  - If upstream produces zero items (no comments / parsing skipped), totals are zero; message will show “None” samples (works).
  - Assumes `intent` values are exactly `positive|neutral|negative`. Anything else won’t be counted.

#### Node: Send Moderation Reports to Team via Slack
- **Type / role:** Slack (`n8n-nodes-base.slack`) — posts message to a channel.
- **Configuration (interpreted):**
  - Sends `{{$json.slack_message}}`
  - Target: **channel** with ID `C09S57E2JQ2` (cached name “n8n”)
  - `includeLinkToWorkflow: false`
- **Inputs / outputs:** Input from **Aggregate Moderation Statistics**. No downstream nodes.
- **Credential requirements:** Slack API credential with permission to post to the channel.
- **Edge cases / failures:**
  - Channel access denied, missing scopes (e.g., `chat:write`), bot not in channel.
  - Slack formatting length limits if samples become too large (currently max 5 samples).

#### Node: Send Moderation Reports to Telegarm
- **Type / role:** Telegram (`n8n-nodes-base.telegram`) — sends message to a chat.
- **Configuration (interpreted):**
  - Sends `{{$json.slack_message}}` as `text`
  - `chatId: 5758325294`
- **Inputs / outputs:** Input from **Aggregate Moderation Statistics**. No downstream nodes.
- **Credential requirements:** Telegram bot token credential.
- **Edge cases / failures:**
  - Bot not allowed to message the chat/user, chat ID mismatch, bot blocked.
  - Telegram message length limits (usually generous, but not infinite).

**Sticky note content applied to this block:**
- “# Aggregate Statistics & Team Notifications … aggregates … sends … to Slack and Telegram…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scheduled Workflow Trigger | Cron Trigger | Schedule workflow every 6 hours | — | Fetch Facebook Page Comments | # Workflow Trigger & Data Collection… fetches recent comments… |
| Fetch Facebook Page Comments | HTTP Request | Call Facebook Graph API to fetch page comments | Scheduled Workflow Trigger | Prepare Comment Items for Processing | # Workflow Trigger & Data Collection… fetches recent comments… |
| Prepare Comment Items for Processing | Code | Light transformation (adds field); intended to prep per-comment items | Fetch Facebook Page Comments | AI-Based Comment Moderation | # Comment Preparation & AI Moderation… sends each comment to OpenAI… |
| AI-Based Comment Moderation | OpenAI (LangChain) | Analyze comment and return strict JSON moderation fields | Prepare Comment Items for Processing | Normalize AI Moderation Results | # Comment Preparation & AI Moderation… sends each comment to OpenAI… |
| Normalize AI Moderation Results | Code | Strip fences, parse AI JSON, compute `flagged`, shape records | AI-Based Comment Moderation | Store Moderation Logs in Database | # Normalize, Flag & Store Results… stores… in Supabase… |
| Store Moderation Logs in Database | Supabase | Insert moderation logs into `facebook_comment_moderation` | Normalize AI Moderation Results | Aggregate Moderation Statistics | # Normalize, Flag & Store Results… stores… in Supabase… |
| Aggregate Moderation Statistics | Code | Compute totals + build Slack/Telegram message | Store Moderation Logs in Database | Send Moderation Reports to Team via Slack; Send Moderation Reports to Telegarm | # Aggregate Statistics & Team Notifications… sends… to Slack and Telegram… |
| Send Moderation Reports to Team via Slack | Slack | Post report to Slack channel | Aggregate Moderation Statistics | — | # Aggregate Statistics & Team Notifications… sends… to Slack and Telegram… |
| Send Moderation Reports to Telegarm | Telegram | Send report to Telegram chat | Aggregate Moderation Statistics | — | # Aggregate Statistics & Team Notifications… sends… to Slack and Telegram… |
| Sticky Note | Sticky Note | Documentation block | — | — | # Workflow Trigger & Data Collection… |
| Sticky Note1 | Sticky Note | Documentation block | — | — | # Comment Preparation & AI Moderation… |
| Sticky Note2 | Sticky Note | Documentation block | — | — | # Normalize, Flag & Store Results… |
| Sticky Note3 | Sticky Note | Documentation block | — | — | # Aggregate Statistics & Team Notifications… |
| Sticky Note4 | Sticky Note | Documentation block | — | — | # How it works… # Setup Steps… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Facebook Page Comment Moderation Scoreboard → Telegram Team Report** (or your preferred title).

2. **Add Cron trigger**
   - Node: **Cron**
   - Configure: *Every X hours* = **6**
   - This is the workflow entry node.

3. **Add Facebook fetch node**
   - Node: **HTTP Request**
   - Method: **GET**
   - URL: `https://graph.facebook.com/v22.0/<PAGE_ID>/comments`
   - Add authentication (recommended approach):
     - Add query parameter: `access_token = <YOUR_PAGE_ACCESS_TOKEN>`
     - Optionally add `fields=message,created_time,from` to ensure `message` is present.
   - Enable **Continue On Fail** (matches current workflow behavior).
   - Connect: **Cron → HTTP Request**

4. **Prepare items for per-comment processing (important)**
   - Node: **Code**
   - Goal: output **one n8n item per Facebook comment**.
   - Configure code (conceptually):
     - Read `items[0].json.data` (array)
     - Map to `[{json: comment}, ...]`
   - Connect: **HTTP Request → Code**
   - Note: the provided workflow’s current code only adds `myNewField` and does not split the array; you will likely want to implement the split for correct moderation per comment.

5. **Add OpenAI moderation node**
   - Node: **OpenAI** (the `@n8n/n8n-nodes-langchain.openAi` node)
   - Credentials: set **OpenAI API** credential.
   - Model: **gpt-4.1-mini**
   - Prompt: include the strict JSON instruction and reference the comment text with an expression like `{{$json.message}}`.
   - Connect: **Code (split comments) → OpenAI**

6. **Normalize and flag results**
   - Node: **Code**
   - Implement:
     - Strip markdown fences if present
     - `JSON.parse()` the AI response
     - Compute `flagged` using your threshold (current: `toxicity_score > 60 || spam || bad_words`)
     - Add timestamp (`comment_created_at`) and preserve the real Facebook `comment_id` by carrying it from the comment input item
   - Connect: **OpenAI → Code (normalize)**

7. **Create Supabase table and configure Supabase node**
   - In Supabase, create table: `facebook_comment_moderation`
   - Suggested columns (aligning with workflow fields):
     - `comment_id` (text)
     - `comment_text` (text)
     - `intent` (text)
     - `toxicity_score` (int)
     - `spam` (bool)
     - `bad_words` (bool)
     - `flagged` (bool)
     - `summary` (text)
     - `comment_created_at` (timestamp/text)
   - Node: **Supabase**
   - Operation: insert (as per node defaults)
   - Table: `facebook_comment_moderation`
   - Mapping: **Auto-map input data**
   - Credentials: Supabase URL + API key
   - Connect: **Normalize Code → Supabase**

8. **Aggregate statistics**
   - Node: **Code**
   - Compute totals and build a single `slack_message` string (mrkdwn).
   - Ensure the node returns **exactly one item** (Slack/Telegram nodes will send one message).
   - Connect: **Supabase → Aggregate Code**

9. **Send to Slack**
   - Node: **Slack**
   - Resource/operation: send message to channel
   - Channel: select your target channel
   - Text: `{{$json.slack_message}}`
   - Credentials: Slack OAuth/app with `chat:write` and channel access
   - Connect: **Aggregate Code → Slack**

10. **Send to Telegram**
   - Node: **Telegram**
   - Operation: send message
   - Chat ID: set your target chat/user/group ID
   - Text: `{{$json.slack_message}}`
   - Credentials: Telegram bot token
   - Connect: **Aggregate Code → Telegram**

11. **Activate workflow**
   - Turn the workflow **Active** and monitor first runs in Executions.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically monitors comments from a Facebook Page… aggregates all results into a summary report… sent to Slack and Telegram…” | Sticky note “How it works.” (embedded in the workflow canvas) |
| Setup steps listed: Cron frequency, Facebook token, OpenAI key, Supabase table, Slack/Telegram credentials, activate workflow | Sticky note “Setup Steps” (embedded in the workflow canvas) |
| Disclaimer: “Le texte fourni provient exclusivement d’un workflow automatisé…” | Provided by the user in the request context |

