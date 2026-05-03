Pick a daily Facebook comment contest winner with OpenAI, Airtable and Telegram

https://n8nworkflows.xyz/workflows/pick-a-daily-facebook-comment-contest-winner-with-openai--airtable-and-telegram-12375


# Pick a daily Facebook comment contest winner with OpenAI, Airtable and Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Pick a daily Facebook comment contest winner with OpenAI, Airtable and Telegram  
**Workflow name (in JSON):** Community Contest Tracker (FB Comments) →Sentiment Analysis-> Telegram Winner Alerts + Airtable Proof

**Purpose:**  
Automatically pick a daily contest winner from Facebook post comments. The workflow runs every day at 21:00, pulls comments from the Facebook Graph API, excludes recent winners using an Airtable “winners” table (fairness), removes low-quality/spam comments, uses an OpenAI-powered sentiment node to keep only “Positive” comments, randomly selects a winner, stores the result in Airtable, and notifies via Telegram. If saving fails, it logs to Supabase and alerts an admin channel.

### 1.1 Input Reception & Scheduling
- Nightly cron trigger (21:00) initiates the run.

### 1.2 Data Ingestion (Facebook + Airtable history)
- Fetches Facebook comments for a specific Post ID.
- Pulls past winners from Airtable to build a blocklist.

### 1.3 Pre-processing & Fairness Filtering
- Code node filters out:
  - Users found in the winners table (note: code comments mention “last 30 days” but implementation blocks **all** past winners returned).
  - Comments that are missing or too short.
- If no candidates remain, the workflow stops early.

### 1.4 AI Sentiment Analysis & Winner Selection
- LangChain Sentiment Analysis node classifies each eligible comment (Positive/Neutral/Negative) using an OpenAI chat model.
- A code node filters for Positive only and picks one at random.

### 1.5 Storage & Notifications + Error Path
- Writes winner to Airtable.
- If Airtable write succeeded, notifies Telegram “winner” chat.
- If Airtable write failed, logs to Supabase then notifies an admin Telegram channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Start
**Overview:** Triggers the workflow once per day at 21:00 server time, starting the ingestion chain.  
**Nodes involved:** `Daily Trigger`

#### Node: Daily Trigger
- **Type / Role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Config choices:**
  - Cron expression: `0 21 * * *` (daily at 21:00).
- **Inputs/Outputs:**
  - **Output →** `Get FB Comments`
- **Edge cases / failures:**
  - Timezone ambiguity: n8n uses instance/workflow timezone settings; if the server timezone differs from desired locale, trigger time may be off.
- **Version notes:** Node typeVersion `1` (standard scheduling behavior).

Sticky note coverage:  
- **Overview** (general workflow description + setup steps)  
- **Sticky Note - Ingestion** (explains scheduled fetch + context retrieval)

---

### Block 2 — Data Ingestion (Facebook + Past Winners)
**Overview:** Pulls contest entries (Facebook comments) and historical winners (Airtable) required for fairness filtering.  
**Nodes involved:** `Get FB Comments`, `Get Past Winners`

#### Node: Get FB Comments
- **Type / Role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Facebook Graph API to fetch post comments.
- **Config choices (interpreted):**
  - Method: default GET (not explicitly set, typical for this node when only URL/query used).
  - URL: `https://graph.facebook.com/v19.0/YOUR_POST_ID/comments`
  - Query parameters:
    - `fields=message,from` (needs commenter name/id and message)
    - `limit=100`
  - Authentication: `httpHeaderAuth` via “genericCredentialType” (a header-based token, typically `Authorization: Bearer ...` or Facebook `access_token` header depending on your credential setup).
- **Key variables/expressions:** none.
- **Inputs/Outputs:**
  - **Input ←** `Daily Trigger`
  - **Output →** `Get Past Winners` (note: this is sequential in this workflow, not truly parallel)
- **Edge cases / failures:**
  - Invalid/expired Facebook token → 401/403.
  - Wrong `YOUR_POST_ID` → 400 or empty results.
  - Facebook paging: `limit=100` may not capture all comments; Graph API may return `paging.next`. This workflow does not paginate.
  - Missing permissions: may not access comments unless proper permissions and app review are in place.
- **Version notes:** typeVersion `4.1` (HTTP Request node UI/behavior differs by version).

#### Node: Get Past Winners
- **Type / Role:** Airtable node (`n8n-nodes-base.airtable`) — searches existing winners.
- **Config choices:**
  - Base: “Community Contest Tracker” (`appMplJTSjazXzICD`)
  - Table: “Contest Winners” (`tblxGqThycjphLBG9`)
  - Operation: `search`
  - Options: none configured (no explicit filter formula in JSON)
  - Credential: Airtable Personal Access Token (PAT)
- **Key variables/expressions:** none.
- **Inputs/Outputs:**
  - **Input ←** `Get FB Comments`
  - **Output →** `Pre-Filter (Blocklist)`
- **Edge cases / failures:**
  - Airtable PAT missing scopes (needs read for base/table) → auth errors.
  - “Search” without filters can return many records; potential performance issues and also changes fairness behavior (see next block).
- **Version notes:** typeVersion `2.1`.

Sticky note coverage:  
- **Sticky Note - Ingestion** (context retrieval).

---

### Block 3 — Pre-processing & Fairness Filtering
**Overview:** Builds a blocklist of users from past winners and filters Facebook comments down to eligible entries; aborts if none remain.  
**Nodes involved:** `Pre-Filter (Blocklist)`, `Any Eligible?`

#### Node: Pre-Filter (Blocklist)
- **Type / Role:** Code node (`n8n-nodes-base.code`) — transforms and filters data.
- **Config choices (what it does):**
  1. Reads Facebook comments from `Get FB Comments` at: `...first().json.data` (expects Graph API shape `{ data: [...] }`).
  2. Reads winners from `Get Past Winners` as an array of items; if access fails, uses empty array.
  3. Creates a `Set` blocklist from Airtable field **`Facebook ID`** (note the space; uses bracket access `w["Facebook ID"]`).
  4. Filters comments:
     - Exclude commenters whose `c.from.id` is in blocklist.
     - Exclude missing messages or messages shorter than 2 chars.
  5. Returns:
     - If none eligible: a single item `{ abort: true, reason: "No eligible new users found" }`
     - Else: one item per eligible comment with `{ abort:false, id, name, text }`
- **Key expressions/variables:**
  - References other nodes via n8n Code helpers:
    - `$("Get FB Comments").first()`
    - `$("Get Past Winners").all()`
  - Output schema:
    - `abort`, `id`, `name`, `text`, optional `reason`
- **Inputs/Outputs:**
  - **Input ←** `Get Past Winners`
  - **Output →** `Any Eligible?`
- **Important logic mismatch / edge cases:**
  - Sticky note says “won in the last 30 days”, but code does **not** filter by date. It blocks **any Facebook ID** present in the Airtable search results. If Airtable search returns all winners ever, previous winners are blocked forever.
  - Facebook comments can include non-user “from” structures (pages, missing fields) → `c.from.id` could be undefined; the code will treat it as not blocked but may produce items with missing id/name if Graph payload differs.
  - Graph API responses can be empty or have errors; code assumes `.json.data` exists or defaults to `[]`.
  - Multi-comment per user: user can have multiple eligible comments; they become multiple entries, increasing their chance to win. (Not necessarily desired.)
- **Version notes:** typeVersion `1`.

#### Node: Any Eligible?
- **Type / Role:** IF node (`n8n-nodes-base.if`) — routes based on abort flag.
- **Config choices:**
  - Condition: boolean `={{ $json.abort }}` equals `true`
- **Inputs/Outputs:**
  - **Input ←** `Pre-Filter (Blocklist)`
  - **True branch (abort=true):** not connected (workflow ends silently)
  - **False branch (abort=false):** **Output →** `Sentiment Analysis`
- **Edge cases / failures:**
  - If `abort` is missing/non-boolean, comparison may behave unexpectedly; in practice the code always sets it.
  - Since the “true” branch is not connected, aborts do not notify anyone.
- **Version notes:** typeVersion `1`.

Sticky note coverage:  
- **Sticky Note - Filter** (fairness + spam protection description).

---

### Block 4 — AI Sentiment Analysis & Random Winner Selection
**Overview:** Uses an OpenAI chat model to classify sentiment for each eligible comment, then selects a random winner among “Positive” entries only.  
**Nodes involved:** `OpenAI Chat Model`, `Sentiment Analysis`, `Pick Random Winner`

#### Node: OpenAI Chat Model
- **Type / Role:** LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM backend to the sentiment node.
- **Config choices:**
  - Model: `gpt-4o-mini`
  - Options: default (none set)
  - Credential: OpenAI API credential
- **Inputs/Outputs:**
  - Special output connection: `ai_languageModel` → `Sentiment Analysis` (as the model provider)
- **Edge cases / failures:**
  - Missing/invalid OpenAI key, quota exceeded, model unavailable.
  - Latency/timeouts when analyzing many comments.
- **Version notes:** typeVersion `1.2` (LangChain node versions matter for connection types like `ai_languageModel`).

#### Node: Sentiment Analysis
- **Type / Role:** LangChain Sentiment Analysis (`@n8n/n8n-nodes-langchain.sentimentAnalysis`) — classifies each comment into categories.
- **Config choices:**
  - Categories: `Positive, Neutral, Negative`
  - Input text: `={{ $json.text }}` (expects the `text` field from Pre-Filter output)
  - Uses `OpenAI Chat Model` via `ai_languageModel` connection.
- **Inputs/Outputs:**
  - **Main input ←** `Any Eligible?` (false branch)
  - **AI language model input ←** `OpenAI Chat Model`
  - **Main output →** `Pick Random Winner`
- **Output expectations (used later):**
  - Produces something like `sentimentAnalysis.category` on each item (the next code node explicitly reads this nested path).
- **Edge cases / failures:**
  - If the node output schema changes or category path differs, downstream selection fails.
  - Non-English or emoji-only messages might yield unexpected categories.
- **Version notes:** typeVersion `1.1`.

#### Node: Pick Random Winner
- **Type / Role:** Code node (`n8n-nodes-base.code`) — filters positives and randomly selects a winner.
- **Config choices (what it does):**
  1. Reads all incoming analyzed items: `$input.all()`.
  2. Keeps only items where `item.json.sentimentAnalysis?.category` includes `"positive"` (case-insensitive).
  3. If none: returns `{ error: true, message: "AI found no positive human comments" }`.
  4. Else: chooses random index and returns a single consolidated “winner” object:
     - `winner_id`, `winner_name`, `winner_comment`, `sentiment`
- **Inputs/Outputs:**
  - **Input ←** `Sentiment Analysis`
  - **Output →** `Create a record`
- **Edge cases / failures:**
  - If sentiment node returns category labels that don’t contain the substring “positive” (e.g., “Pos” or localized), all candidates may be filtered out.
  - If the “no positive” error object flows into Airtable creation (it will), the Airtable node may create incomplete records or fail depending on required fields. In this workflow, Airtable fields are not marked required in mapping, so it may create a record with blanks or typecast issues.
- **Version notes:** typeVersion `1`.

Sticky note coverage:  
- **Sticky Note - AI** (sentiment + random draw).

---

### Block 5 — Storage, Winner Announcement, and Error Handling
**Overview:** Writes the selected winner to Airtable, then either announces in Telegram (success) or logs to Supabase and alerts admin (failure).  
**Nodes involved:** `Create a record`, `Saved?`, `Send a text message1`, `Log Error (Supabase) `, `Notify Admin (Error)1`

#### Node: Create a record
- **Type / Role:** Airtable node (`n8n-nodes-base.airtable`) — persists winner as proof/ledger.
- **Config choices:**
  - Base/Table: same as `Get Past Winners` (Community Contest Tracker → Contest Winners)
  - Operation: `create`
  - Fields mapped (“define below”):
    - `Date` = `={{ $today }}`
    - `Name` = `={{ $json.winner_name }}`
    - `Facebook ID` = `={{ $json.winner_id }}`
  - Options: `typecast: true` (Airtable attempts to coerce types)
  - Credential: Airtable PAT
- **Inputs/Outputs:**
  - **Input ←** `Pick Random Winner`
  - **Output →** `Saved?`
- **Edge cases / failures:**
  - If `Pick Random Winner` produced an error object (no winner), fields may be empty → may still create a junk record unless Airtable enforces constraints.
  - Airtable rate limits, schema mismatches, base/table access issues.
- **Version notes:** typeVersion `2.1`.

#### Node: Saved?
- **Type / Role:** IF node — checks whether Airtable returned a created record id.
- **Config choices:**
  - Condition: `={{ $json.id ? true : false }}` equals `true`
- **Inputs/Outputs:**
  - **Input ←** `Create a record`
  - **True branch →** `Send a text message1`
  - **False branch →** `Log Error (Supabase) `
- **Edge cases / failures:**
  - If Airtable returns different output shape, `id` may not exist even on success, sending to error path.
- **Version notes:** typeVersion `1`.

#### Node: Send a text message1
- **Type / Role:** Telegram node (`n8n-nodes-base.telegram`) — announces winner.
- **Config choices:**
  - Chat ID: `123456789` (likely a group/channel/user id; must be replaced)
  - Message text uses expression:
    - `{{ $('Pick Random Winner').item.json.winner_name }}`
  - `appendAttribution: false`
- **Inputs/Outputs:**
  - **Input ←** `Saved?` (true branch)
  - No downstream nodes.
- **Edge cases / failures:**
  - Telegram bot not in chat or missing permission to post.
  - Wrong chat id format (channels often use `@channelname` or numeric id).
  - If `Pick Random Winner` produced error but Airtable still returned an id (junk record created), Telegram could announce blank/undefined name.
- **Version notes:** typeVersion `1.2`.

#### Node: Log Error (Supabase)
- **Type / Role:** Supabase node (`n8n-nodes-base.supabase`) — intended to persist error details.
- **Config choices:**
  - **Not configured in JSON** (no operation/table/project shown). This node will not function until configured.
- **Inputs/Outputs:**
  - **Input ←** `Saved?` (false branch)
  - **Output →** `Notify Admin (Error)1`
- **Edge cases / failures:**
  - As-is, will likely fail due to missing credentials/parameters, which may prevent admin notification depending on n8n error behavior (node error stops execution unless error handling is enabled).
- **Version notes:** typeVersion `1`.

#### Node: Notify Admin (Error)1
- **Type / Role:** Telegram node — alerts admin channel on failures.
- **Config choices:**
  - Chat ID: `@youradminchannel` (placeholder)
  - Text:
    - `⚠️ Contest Error\n\nDetails: {{ $json.error_message || $json.message }}`
  - `appendAttribution: false`
- **Inputs/Outputs:**
  - **Input ←** `Log Error (Supabase) `
  - No downstream nodes.
- **Edge cases / failures:**
  - If Supabase logging fails and doesn’t output `error_message/message`, admin message may be empty.
  - Same Telegram permission/chat id concerns as above.
- **Version notes:** typeVersion `1`.

Sticky note coverage:  
- **Sticky Note - Storage** (database sync + alerting).

---

### Sticky Notes (Documentation Nodes)
These are non-executing nodes but provide important context and setup instructions:

#### Node: Overview (Sticky Note)
- Contains: workflow explanation, setup steps (credentials, Post ID, Airtable blocklist/winners DB, activate).

#### Node: Sticky Note - Ingestion (Sticky Note)
- Describes scheduled fetch and parallel retrieval concept (note: actual connections are sequential).

#### Node: Sticky Note - Filter (Sticky Note)
- Mentions 30-day blocklist and spam patterns (code only implements “short comment” + “past winners list”).

#### Node: Sticky Note - AI (Sticky Note)
- States GPT-4o-mini positivity filtering + random draw.

#### Node: Sticky Note - Storage (Sticky Note)
- States Airtable write + Telegram notification + Supabase error logging.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documentation / context | — | — | # Facebook Community Contest Automator … (includes Setup Steps: configure Facebook Graph API, Airtable, OpenAI, Telegram, Supabase; update Post ID in Get FB Comments; ensure Get Past Winners points to Airtable; activate) |
| Sticky Note - Ingestion | Sticky Note | Documentation: ingestion stage | — | — | ## 1. Data Ingestion … triggers daily 21:00; fetches FB comments + past winners |
| Sticky Note - Filter | Sticky Note | Documentation: filtering stage | — | — | ## 2. Pre-Processing & Filter … blocklist last 30 days; filters short/spam; halt if none |
| Sticky Note - AI | Sticky Note | Documentation: AI stage | — | — | ## 3. AI Analysis & Selection … GPT-4o-mini sentiment; keep Positive; random winner |
| Sticky Note - Storage | Sticky Note | Documentation: storage/notify stage | — | — | ## 4. Storage & Notifications … write winner to Airtable; Telegram announce; if save fails log to Supabase + alert admin |
| Daily Trigger | Schedule Trigger | Starts workflow daily at 21:00 | — | Get FB Comments | # Facebook Community Contest Automator … / ## 1. Data Ingestion … |
| Get FB Comments | HTTP Request | Fetch comments from Facebook Graph API | Daily Trigger | Get Past Winners | # Facebook Community Contest Automator … / ## 1. Data Ingestion … |
| Get Past Winners | Airtable (search) | Retrieve historical winners for blocklist | Get FB Comments | Pre-Filter (Blocklist) | # Facebook Community Contest Automator … / ## 1. Data Ingestion … |
| Pre-Filter (Blocklist) | Code | Build blocklist + filter eligible comments | Get Past Winners | Any Eligible? | ## 2. Pre-Processing & Filter … |
| Any Eligible? | IF | Stop if no eligible candidates | Pre-Filter (Blocklist) | (false) Sentiment Analysis | ## 2. Pre-Processing & Filter … |
| OpenAI Chat Model | LangChain Chat Model (OpenAI) | Provides LLM for sentiment node | — | Sentiment Analysis (ai_languageModel) | ## 3. AI Analysis & Selection … |
| Sentiment Analysis | LangChain Sentiment | Classify sentiment of each comment | Any Eligible? (false) + OpenAI Chat Model | Pick Random Winner | ## 3. AI Analysis & Selection … |
| Pick Random Winner | Code | Keep Positive items, select random winner | Sentiment Analysis | Create a record | ## 3. AI Analysis & Selection … |
| Create a record | Airtable (create) | Store winner proof/ledger | Pick Random Winner | Saved? | ## 4. Storage & Notifications … |
| Saved? | IF | Check Airtable create success | Create a record | (true) Send a text message1; (false) Log Error (Supabase)  | ## 4. Storage & Notifications … |
| Send a text message1 | Telegram | Announce winner to Telegram | Saved? (true) | — | ## 4. Storage & Notifications … |
| Log Error (Supabase)  | Supabase | Log error details (currently unconfigured) | Saved? (false) | Notify Admin (Error)1 | ## 4. Storage & Notifications … |
| Notify Admin (Error)1 | Telegram | Notify admin about failure | Log Error (Supabase)  | — | ## 4. Storage & Notifications … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Schedule Trigger**
   - Node: *Schedule Trigger*
   - Set **Cron** to: `0 21 * * *`
   - Confirm timezone in workflow/global settings matches your intended “21:00”.

3. **Add HTTP Request: “Get FB Comments”**
   - Node: *HTTP Request*
   - Method: GET
   - URL: `https://graph.facebook.com/v19.0/<POST_ID>/comments`
   - Query parameters:
     - `fields` = `message,from`
     - `limit` = `100`
   - Authentication:
     - Use **Header Auth** (or equivalent) with a Facebook Graph API access token.
     - Typical approaches:
       - `Authorization: Bearer <token>` header, or
       - add `access_token=<token>` query param (not used here; keep consistent with your org’s security practice).
   - Connect: **Daily Trigger → Get FB Comments**
   - Replace `<POST_ID>` with the actual Facebook post ID.

4. **Add Airtable: “Get Past Winners”**
   - Node: *Airtable*
   - Credentials: Airtable Personal Access Token (PAT) with read/write as needed.
   - Base: select your base (e.g., “Community Contest Tracker”)
   - Table: select “Contest Winners”
   - Operation: **Search**
   - (Optional but recommended) Add a filter formula to limit winners to last 30 days if you want the behavior described in the notes.
   - Connect: **Get FB Comments → Get Past Winners**

5. **Add Code: “Pre-Filter (Blocklist)”**
   - Node: *Code*
   - Paste logic equivalent to:
     - Read `Get FB Comments` → `json.data`
     - Read all `Get Past Winners` items
     - Build Set from `["Facebook ID"]`
     - Filter comments by non-blocklisted user and message length >= 2
     - Output either abort item or mapped items `{abort,id,name,text}`
   - Connect: **Get Past Winners → Pre-Filter (Blocklist)**

6. **Add IF: “Any Eligible?”**
   - Node: *IF*
   - Condition (Boolean):
     - Value1: `={{ $json.abort }}`
     - Equals: `true`
   - Connect: **Pre-Filter (Blocklist) → Any Eligible?**
   - Leave **true** branch unconnected (to silently stop) or connect it to an alert node if you want visibility.
   - Connect **false** branch to the sentiment node (next step).

7. **Add OpenAI model node: “OpenAI Chat Model”**
   - Node: *OpenAI Chat Model* (LangChain)
   - Credentials: OpenAI API key
   - Model: `gpt-4o-mini`

8. **Add “Sentiment Analysis” (LangChain)**
   - Node: *Sentiment Analysis*
   - Categories: `Positive, Neutral, Negative`
   - Input text: `={{ $json.text }}`
   - Connect:
     - **Any Eligible? (false) → Sentiment Analysis (main)**
     - **OpenAI Chat Model → Sentiment Analysis (ai_languageModel)**

9. **Add Code: “Pick Random Winner”**
   - Node: *Code*
   - Implement:
     - Read all items from input
     - Filter where `sentimentAnalysis.category` contains “positive”
     - If none, output `{ error:true, message:"AI found no positive human comments" }`
     - Else output consolidated winner object: `winner_id`, `winner_name`, `winner_comment`, `sentiment`
   - Connect: **Sentiment Analysis → Pick Random Winner**

10. **Add Airtable: “Create a record”**
   - Node: *Airtable*
   - Operation: **Create**
   - Base/Table: same as winners table
   - Map fields:
     - Date = `={{ $today }}`
     - Name = `={{ $json.winner_name }}`
     - Facebook ID = `={{ $json.winner_id }}`
   - Enable **Typecast** if you want Airtable to coerce types.
   - Connect: **Pick Random Winner → Create a record**

11. **Add IF: “Saved?”**
   - Node: *IF*
   - Condition (Boolean):
     - Value1: `={{ $json.id ? true : false }}`
     - Equals: `true`
   - Connect: **Create a record → Saved?**

12. **Add Telegram: “Send a text message1” (success path)**
   - Node: *Telegram*
   - Credentials: Telegram bot token
   - Chat ID: set to your target chat/channel id
   - Message text, for example:
     - `We have a winner!\nName: {{ $('Pick Random Winner').item.json.winner_name }}\nCongrats on the positive vibes!`
   - Connect: **Saved? (true) → Send a text message1**

13. **Add Supabase: “Log Error (Supabase)” (failure path)**
   - Node: *Supabase*
   - Configure credentials (Supabase URL + service role key or appropriate key).
   - Choose an operation (commonly **Insert**) into an `errors` table with columns like:
     - `workflow`, `timestamp`, `node`, `error_message`, `payload`
   - Connect: **Saved? (false) → Log Error (Supabase)**

14. **Add Telegram: “Notify Admin (Error)1”**
   - Node: *Telegram*
   - Chat ID: your admin channel (e.g., `@youradminchannel`)
   - Message:
     - `⚠️ Contest Error\n\nDetails: {{ $json.error_message || $json.message }}`
   - Connect: **Log Error (Supabase) → Notify Admin (Error)1**

15. **(Optional) Add Sticky Notes**
   - Add sticky notes to document setup steps, the 4 functional blocks, and credential requirements.

16. **Activate workflow**
   - Ensure all credentials are valid and the Facebook Post ID is set.
   - Run once manually to verify:
     - Facebook returns `data[]` with `message` and `from`.
     - Airtable search returns items with field `Facebook ID`.
     - Sentiment node outputs `sentimentAnalysis.category`.
     - Airtable create returns `id`.
     - Telegram posts successfully.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Configure credentials for Facebook Graph API, Airtable, OpenAI, Telegram, Supabase. | From sticky note “Overview” |
| Update the Post ID in “Get FB Comments” (`YOUR_POST_ID`). | From sticky note “Overview” |
| Ensure “Get Past Winners” points to your winners database to enforce fairness. | From sticky note “Overview” |
| Workflow description: nightly run, fetch comments, exclude past winners, filter spam, use OpenAI positivity, random select, log to Airtable, notify Telegram; log errors to Supabase. | Sticky notes “Overview”, “Ingestion”, “Filter”, “AI”, “Storage” |
| Design note: documentation claims “last 30 days” blocklist, but current code blocks all winners returned by Airtable search unless you filter Airtable results by date. | Implementation detail derived from `Pre-Filter (Blocklist)` code |

