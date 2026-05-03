Create WordPress posts from Reddit content via Telegram bot and Perplexity

https://n8nworkflows.xyz/workflows/create-wordpress-posts-from-reddit-content-via-telegram-bot-and-perplexity-13042


# Create WordPress posts from Reddit content via Telegram bot and Perplexity

## 1. Workflow Overview

**Purpose:**  
This workflow periodically fetches new Reddit posts, converts each post title into an article topic idea using Google Gemini, sends the idea to a user via a Telegram bot for approval, and—if approved—researches the topic with Perplexity, generates a SEO title + long-form HTML article + slug via Gemini, then creates a **draft** WordPress post and notifies the user.

**Target use cases:**
- Content teams sourcing article ideas from Reddit communities
- Semi-automated editorial pipelines with human approval via Telegram
- WordPress sites needing research-backed long-form posts

### 1.1 Reddit ingestion & deduplication (scheduled)
Fetch “new” Reddit posts on a schedule and prevent re-processing previously seen titles.

### 1.2 Topic idea generation & storage
Turn each Reddit title into an article topic idea and store it in an n8n Data Table for tracking.

### 1.3 Human approval via Telegram
Send each topic idea to Telegram with inline “Approve/Disapprove” buttons.

### 1.4 Approval handling & duplicate prevention (callback webhook)
On approval callback, check if the topic was already approved/written; if not, mark approved and start article creation.

### 1.5 Research, generation (title/content/slug) & WordPress publishing
Use Perplexity for a research summary, Gemini to generate title/content/slug, create a WordPress draft post, update tracking table with title+URL, then notify the user.

---

## 2. Block-by-Block Analysis

### Block A — Reddit ingestion & deduplication (scheduled)

**Overview:**  
Runs daily at a configured hour, pulls the newest Reddit posts, and filters out titles already processed in prior executions.

**Nodes involved:**
- Periodic Check for Reddit Posts
- Get Reddit Posts
- Remove Duplicate Reddit Posts Fetched Earlier

#### Node: **Periodic Check for Reddit Posts**
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point (timer-based).
- **Configuration:** Runs at `triggerAtHour: 19` (daily schedule at 19:00; timezone depends on n8n instance settings).
- **Outputs:** Triggers the workflow into “Get Reddit Posts”.
- **Failure modes / edge cases:**
  - Instance timezone mismatch can cause unexpected run times.
  - If n8n is down at scheduled time, run may be missed unless additional scheduling/backfill exists.

#### Node: **Get Reddit Posts**
- **Type / role:** Reddit (`n8n-nodes-base.reddit`) — fetch content from Reddit.
- **Configuration choices:**
  - Operation: **getAll**
  - Filter category: **new**
  - Limit: **10**
  - Requires Reddit OAuth2 credentials.
- **Inputs:** From Schedule Trigger.
- **Outputs:** List of Reddit posts (items).
- **Failure modes / edge cases:**
  - OAuth token expiry / invalid credentials.
  - Subreddits are not explicitly configured in this node JSON; in n8n UI you must set the target subreddit(s) or listing parameters. If none are set, the node may fail or return unexpected defaults.
  - Rate limits from Reddit API.

#### Node: **Remove Duplicate Reddit Posts Fetched Earlier**
- **Type / role:** Remove Duplicates (`n8n-nodes-base.removeDuplicates`) — persistence-based dedupe across executions.
- **Configuration choices:**
  - Operation: `removeItemsSeenInPreviousExecutions`
  - Dedupe value: `{{ $json.title }}`
- **Inputs:** Reddit posts.
- **Outputs:** Only posts whose `title` hasn’t been seen in previous runs.
- **Failure modes / edge cases:**
  - If Reddit titles repeat (common on Reddit), genuine new posts with same title will be skipped.
  - If `title` is missing/null for any item, expression may dedupe incorrectly.

**Sticky note context (applies to this block):**
- “Add relevant SubReddits available in your niche… Add at least 10-15 subreddits relevant to your niche”

---

### Block B — Topic idea generation (Gemini) & normalization

**Overview:**  
Converts each Reddit post title into an article topic idea using a LangChain Agent powered by Google Gemini, then normalizes the data into `{post_id, article_topic}`.

**Nodes involved:**
- Google Gemini Chat Model
- Generate Article Idea
- Set Topic Data

#### Node: **Google Gemini Chat Model**
- **Type / role:** LangChain chat model connector (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides the LLM to the agent.
- **Configuration choices:** Default options; uses Google PaLM/Gemini credentials.
- **Connections:**
  - Output (AI languageModel) → “Generate Article Idea” as its model.
- **Failure modes / edge cases:**
  - Invalid/expired API key.
  - Model availability / quota errors.

#### Node: **Generate Article Idea**
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — generates topic idea text.
- **Configuration choices (interpreted):**
  - Prompt is **defined** via `promptType: define`.
  - Uses Reddit title: `{{ $json.title }}`
  - Instruction text says “generate exactly 1 article topic idea” but later says “output three article topic ideas”; the prompt is internally inconsistent.
  - Expected output is stored under `output` (used later as `$json.output`).
- **Inputs:** Items from “Remove Duplicate…” (each item is a Reddit post).
- **Outputs:** Agent result (including `output`).
- **Failure modes / edge cases:**
  - Output format inconsistency due to contradictory prompt (1 idea vs 3 ideas).
  - If agent returns a structured response not in `output`, downstream nodes will break.
  - Hallucinated content risk (mitigated by instructions, but not guaranteed).

#### Node: **Set Topic Data**
- **Type / role:** Set (`n8n-nodes-base.set`) — creates consistent fields for storage/sending.
- **Configuration choices:**
  - `post_id` = `{{ $node["Get Reddit Posts"].json["id"] }}`
  - `article_topic` = `{{ $json.output }}`
- **Inputs:** From “Generate Article Idea”.
- **Outputs:** Object with `post_id` + `article_topic` (and whatever else is kept by n8n depending on Set node settings).
- **Failure modes / edge cases:**
  - If Reddit node doesn’t provide `id` under `json.id` (some Reddit APIs use `name` like `t3_xxx`), this may be empty.
  - If `output` is missing, `article_topic` becomes empty.

**Sticky note context (applies to this block):**
- “Add relevant information about the author & your niche… replace placeholder info in []”

---

### Block C — Persist ideas and send to Telegram for approval

**Overview:**  
Stores each generated topic in a Data Table (“Reddit Article Ideas”) and sends it to a Telegram chat with inline approve/disapprove buttons containing callback data tied to the post_id.

**Nodes involved:**
- Add Topic Data to DataTable
- Send Article Ideas to User

#### Node: **Add Topic Data to DataTable**
- **Type / role:** Data Table (`n8n-nodes-base.dataTable`) — inserts a row into “Reddit Article Ideas”.
- **Configuration choices:**
  - Operation: create/insert (no explicit “operation” field shown besides `dataTableId`; in UI it’s an **insert** action)
  - Values:
    - `topic` = `{{ $json.article_topic }}`
    - `status` = `not approved`
    - `post_id` = `{{ $json.post_id }}`
  - Target table: **Reddit Article Ideas**
- **Inputs:** From “Set Topic Data”.
- **Outputs:** Insert result + stored values (depends on Data Table behavior).
- **Failure modes / edge cases:**
  - Table schema mismatch (missing columns or different names).
  - Duplicate topics may be inserted multiple times (no uniqueness constraints shown).

#### Node: **Send Article Ideas to User**
- **Type / role:** Telegram (`n8n-nodes-base.telegram`) — sends approval message with inline keyboard.
- **Configuration choices:**
  - Message text includes:
    - `{{ $node["Set Topic Data"].json["article_topic"] }}`
  - Inline keyboard buttons:
    - Approve callback_data: `approve_{{ post_id }}`
    - Disapprove callback_data: `disapprove_{{ post_id }}`
  - `appendAttribution: false`
  - Requires Telegram bot credentials.
  - **Chat ID is not shown in JSON**; must be configured in node UI or via credential defaults.
- **Inputs:** From “Add Topic Data to DataTable”.
- **Outputs:** Telegram send response.
- **Failure modes / edge cases:**
  - Missing chat ID → message won’t deliver.
  - Bot not started by user / blocked bot.
  - Callback data length limits (Telegram limit is 64 bytes); long IDs could break, though Reddit IDs are usually short.

**Sticky note context (applies to this block):**
- “Add your Telegram chat ID and API credentials…”

---

### Block D — Telegram callback handling (approve/disapprove routing)

**Overview:**  
Receives Telegram callback queries and routes only approvals into the publishing pipeline; disapprovals are ignored (no-op).

**Nodes involved:**
- Telegram Webhook
- Approval Check
- No Operation, do nothing

#### Node: **Telegram Webhook**
- **Type / role:** Telegram Trigger (`n8n-nodes-base.telegramTrigger`) — entry point (event-based).
- **Configuration choices:**
  - Updates: `callback_query`
- **Outputs:** Callback payload to “Approval Check”.
- **Failure modes / edge cases:**
  - Webhook not registered correctly in Telegram.
  - n8n public URL misconfigured (Telegram cannot reach it).
  - If callback_query payload differs (e.g., from other bot interactions), downstream expressions may fail.

#### Node: **Approval Check**
- **Type / role:** IF (`n8n-nodes-base.if`) — checks whether callback indicates approval.
- **Configuration choices:**
  - Condition: `{{ $node["Telegram Webhook"].json["callback_query"]["data"] }}` **startsWith** `approve_`
- **Outputs:**
  - True → “Get Approved Topic”
  - False → “No Operation, do nothing”
- **Failure modes / edge cases:**
  - If `callback_query.data` is missing, expression evaluation may throw or evaluate false depending on n8n behavior.

#### Node: **No Operation, do nothing**
- **Type / role:** NoOp (`n8n-nodes-base.noOp`) — sink node for disapprovals.
- **Purpose:** Explicitly ends the disapprove path.
- **Failure modes:** None (but disapproved topics are not written back to Data Tables, so they remain “not approved” forever unless handled elsewhere).

---

### Block E — Duplicate prevention against previously approved topics

**Overview:**  
Checks whether the approved topic already exists in the “Reddit Approved Topics” table to prevent generating the same article again.

**Nodes involved:**
- Get Approved Topic
- Check for Already Written Topic
- No Operation, do nothing1

#### Node: **Get Approved Topic**
- **Type / role:** Data Table get (`n8n-nodes-base.dataTable`) — lookup topic in “Reddit Approved Topics”.
- **Configuration choices:**
  - Operation: `get`
  - Filter condition:
    - key: `topic`
    - value: `{{ $json.callback_query.message.text.split('\n\n')[1] || '' }}`
      - This parses the Telegram message text and assumes the topic is the second block separated by a blank line.
  - `alwaysOutputData: true` (important): even if no record is found, node outputs something.
- **Inputs:** Approval path from IF.
- **Outputs:** Matching row (if found) or empty-ish output (implementation-specific).
- **Failure modes / edge cases:**
  - Parsing is fragile: if Telegram message formatting changes, `split('\n\n')[1]` may be undefined/empty.
  - If multiple rows match same topic, returned record ambiguity.

#### Node: **Check for Already Written Topic**
- **Type / role:** IF (`n8n-nodes-base.if`) — decide whether to proceed with article creation.
- **Configuration choices:**
  - Condition (string notEquals):
    - Left: `{{ $('Telegram Webhook').item.json.callback_query.message.text.split('\n\n')[1] || '' }}`
    - Right: `{{ $json.topic }}`
  - Interpretation:
    - If the topic from Telegram message is **not equal** to the topic returned by “Get Approved Topic”, proceed.
    - If it equals, it means the topic already exists in approved table → stop.
- **Outputs:**
  - True → “Set Status to Approved”
  - False → “No Operation, do nothing1”
- **Failure modes / edge cases:**
  - If “Get Approved Topic” returns no row, `$json.topic` may be undefined → notEquals likely evaluates true → proceeds (desired).
  - If table returns an empty string topic, logic may misbehave.

#### Node: **No Operation, do nothing1**
- **Type / role:** NoOp — sink node when topic already exists.
- **Purpose:** Prevent duplicate article generation.

---

### Block F — Mark approved, create approved-topic record, research with Perplexity

**Overview:**  
Marks the idea as approved in “Reddit Article Ideas”, inserts it into “Reddit Approved Topics” as “in process”, then retrieves a research summary from Perplexity.

**Nodes involved:**
- Set Status to Approved
- Add Approved Topic
- Research about Approved Topic

#### Node: **Set Status to Approved**
- **Type / role:** Data Table update (`n8n-nodes-base.dataTable`) — updates tracking status.
- **Configuration choices:**
  - Table: **Reddit Article Ideas**
  - Operation: `update`
  - Filter: `topic` equals `{{ Telegram message parsed topic }}`
  - Sets `status` to `approved`
- **Inputs:** From “Check for Already Written Topic” (true path).
- **Outputs:** Update result.
- **Failure modes / edge cases:**
  - If topic parsing fails, it may update nothing.
  - If multiple identical topics exist, it may update multiple rows.

#### Node: **Add Approved Topic**
- **Type / role:** Data Table insert (`n8n-nodes-base.dataTable`) — inserts into “Reddit Approved Topics”.
- **Configuration choices:**
  - Values:
    - `topic` = `{{ $json.topic }}`
    - `status` = `in process`
    - `post_id` = `{{ $json.post_id }}`
- **Inputs:** From “Set Status to Approved”.
- **Important mapping issue:** “Set Status to Approved” node’s configured schema shows `post_id` and `topic` as “removed” fields in the schema UI snapshot. If the update node output doesn’t include `post_id`/`topic`, then `{{ $json.topic }}` and `{{ $json.post_id }}` may be missing here.
- **Outputs:** New approved-topic row.
- **Failure modes / edge cases:**
  - Missing `post_id` or `topic` due to upstream output not carrying them.
  - Duplicate inserts if user clicks approve multiple times.

#### Node: **Research about Approved Topic**
- **Type / role:** Perplexity (`n8n-nodes-base.perplexity`) — generates research summary.
- **Configuration choices:**
  - Model: `sonar-pro`
  - Prompt: “Research on this topic {{ $json.topic }} and create a detailed 700 words summary… no fluff”
  - `simplify: true` (returns simplified message field)
- **Inputs:** From “Add Approved Topic” (expects `topic`).
- **Outputs:** Simplified result; later referenced as `$('Research about Approved Topic').item.json.message`
- **Failure modes / edge cases:**
  - Perplexity auth errors / quota.
  - Prompt asks for exactly “700 words” but models may not comply.
  - Output may be too short/too long; downstream content generation may still work.

**Sticky note context (applies to this block):**
- “Add your Perplexity API credentials…”

---

### Block G — AI generation (title, article HTML, slug) using Gemini

**Overview:**  
Uses Perplexity research + approved topic to generate a SEO title, then a 2000-word HTML article, then a slug.

**Nodes involved:**
- Google Gemini Chat Model4
- Title Generation
- Google Gemini Chat Model1
- Content Generation
- Google Gemini Chat Model5
- Slug Generation

#### Node: **Google Gemini Chat Model4**
- **Type / role:** Gemini model connector for title generation.
- **Configuration choices:** Uses Gemini credentials named `Gemini(NewsLeaks)` in JSON.
- **Connections:** feeds Title Generation (ai_languageModel).
- **Failure modes:** credential/quota/model errors.

#### Node: **Title Generation**
- **Type / role:** Chain LLM (`@n8n/n8n-nodes-langchain.chainLlm`) — produces one SEO title.
- **Configuration choices:**
  - Prompt uses topic from Data Table: `{{ $('Add Approved Topic').item.json.topic }}`
  - Output field used later: `$('Title Generation').item.json.text`
- **Inputs:** From “Research about Approved Topic” (main path), and Gemini model via ai_languageModel.
- **Outputs:** A single title in `text`.
- **Failure modes / edge cases:**
  - If `Add Approved Topic` didn’t output `topic`, title prompt fails.
  - Model may include quotes or extra lines; downstream WordPress title still accepts but may be undesirable.

#### Node: **Google Gemini Chat Model1**
- **Type / role:** Gemini model connector for content generation.
- **Connections:** feeds Content Generation.
- **Failure modes:** as above.

#### Node: **Content Generation**
- **Type / role:** LangChain Agent — produces the full HTML article.
- **Configuration choices (interpreted):**
  - Uses Perplexity summary: `{{ $('Research about Approved Topic').item.json.message }}`
  - Uses “Article Topic”: `Topic : {{ $json.text }}`
    - **This is likely a bug:** `$json.text` at this point comes from the **Title Generation** node output (because Title Generation connects to Content Generation). So the “Article Topic” becomes the generated **title**, not the approved topic. It may still work, but it’s semantically mismatched.
  - Very long style/spec rules; requires replacing placeholders like `[Author's Name]` and `[Add Author Writing Style Summary]`.
  - Output referenced later as `$('Content Generation').item.json.output`
- **Inputs:** From “Title Generation” main output; model via ai_languageModel.
- **Outputs:** HTML article content.
- **Failure modes / edge cases:**
  - Output might include non-HTML or include preamble despite instructions.
  - Very long prompt can increase cost and latency; may hit token limits.
  - If Perplexity output is missing, content may be low quality.

#### Node: **Google Gemini Chat Model5**
- **Type / role:** Gemini model connector for slug generation.
- **Connections:** feeds Slug Generation.

#### Node: **Slug Generation**
- **Type / role:** Chain LLM — returns a single URL slug.
- **Configuration choices:**
  - Prompt uses `{{ $json.output }}` (input is from Content Generation, whose main output includes `output`).
  - Returns one slug in `text`, later used as `{{ $json.text }}` in WordPress node (because Slug Generation outputs `text`).
- **Inputs:** From “Content Generation”; model via ai_languageModel.
- **Outputs:** slug text.
- **Failure modes / edge cases:**
  - Model may output spaces, uppercase, or leading/trailing punctuation; WordPress may sanitize but URLs may differ.
  - If content output is huge, slug prompt is based on it; better to base slug on title/topic.

**Sticky note context (applies to this block):**
- “Add Author Name & Writing Style…”

---

### Block H — WordPress post creation, tracking update, Telegram notification

**Overview:**  
Creates a WordPress post (draft), updates the approved topics Data Table with title and URL, then notifies the user in Telegram.

**Nodes involved:**
- WP Post Creation
- Add Title & URL to Approved Topic
- Notify User about Article Creation

#### Node: **WP Post Creation**
- **Type / role:** WordPress (`n8n-nodes-base.wordpress`) — creates a post.
- **Configuration choices:**
  - Title: `{{ $('Title Generation').item.json.text }}`
  - Slug: `{{ $json.text }}` (from Slug Generation output)
  - Status: `draft`
  - Content: `{{ $('Content Generation').item.json.output }}`
- **Inputs:** From “Slug Generation”.
- **Outputs:** WP API response (post object).
- **Failure modes / edge cases:**
  - WordPress credentials invalid / application password revoked.
  - REST API disabled or blocked by security plugin.
  - Content too large or blocked by server limits.
  - Slug collisions (WP will auto-adjust).

#### Node: **Add Title & URL to Approved Topic**
- **Type / role:** Data Table update — enriches approved-topic row with generated title and URL.
- **Configuration choices:**
  - Table: **Reddit Approved Topics**
  - Filter: `topic` equals `{{ $('Add Approved Topic').item.json.topic }}`
  - Sets:
    - `title` = `{{ $('Title Generation').item.json.text }}`
    - `url` = `yourwebsite.com/{{ $('Slug Generation').item.json.text }}`
      - **Hardcoded domain placeholder**: must be replaced with your real domain and preferably include protocol (`https://`).
- **Inputs:** From “WP Post Creation”.
- **Outputs:** Updated row (or update result).
- **Failure modes / edge cases:**
  - If topic filter doesn’t match exactly, row won’t update.
  - URL may not match actual WordPress permalink settings.

#### Node: **Notify User about Article Creation**
- **Type / role:** Telegram send message — informs user of completion.
- **Configuration choices:**
  - Text includes title from Title Generation and `{{ $json.url }}` from the Data Table update output.
  - `appendAttribution: false`
- **Inputs:** From “Add Title & URL to Approved Topic”.
- **Failure modes / edge cases:**
  - If Data Table update doesn’t return `url`, message shows blank URL.
  - Chat ID must be configured.

**Sticky note context (applies to this block):**
- “Configure WordPress credentials and fields as per the requirements”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Periodic Check for Reddit Posts | Schedule Trigger | Scheduled entry point | — | Get Reddit Posts | # Create wordpress posts with reddit content via telegram bot & perplexity… (overall description + setup + help links) |
| Get Reddit Posts | Reddit | Fetch latest Reddit posts | Periodic Check for Reddit Posts | Remove Duplicate Reddit Posts Fetched Earlier | Same overall note; “Add relevant SubReddits… 10-15” |
| Remove Duplicate Reddit Posts Fetched Earlier | Remove Duplicates | Cross-execution dedupe on title | Get Reddit Posts | Generate Article Idea | Same overall note; “Add relevant SubReddits… 10-15” |
| Google Gemini Chat Model | Gemini Chat Model (LangChain) | LLM provider for idea agent | — (AI port) | Generate Article Idea (AI) | Same overall note; “Add relevant information about the author & your niche… replace placeholder info in []” |
| Generate Article Idea | LangChain Agent | Convert Reddit title to article topic idea | Remove Duplicate Reddit Posts Fetched Earlier | Set Topic Data | Same overall note; “Add relevant information about the author & your niche… replace placeholder info in []” |
| Set Topic Data | Set | Normalize fields (post_id, article_topic) | Generate Article Idea | Add Topic Data to DataTable | Same overall note |
| Add Topic Data to DataTable | Data Table | Store idea as not approved | Set Topic Data | Send Article Ideas to User | “Create 2 Data Tables… columns…” |
| Send Article Ideas to User | Telegram | Send topic with approve/disapprove buttons | Add Topic Data to DataTable | — | “Add your Telegram chat ID and API credentials…” |
| Telegram Webhook | Telegram Trigger | Callback entry point | — | Approval Check | Same overall note |
| Approval Check | IF | Route approve vs disapprove | Telegram Webhook | Get Approved Topic / No Operation, do nothing | Same overall note |
| No Operation, do nothing | NoOp | Sink for disapprove path | Approval Check (false) | — | Same overall note |
| Get Approved Topic | Data Table (get) | Lookup topic in Approved Topics table | Approval Check (true) | Check for Already Written Topic | “Create 2 Data Tables… columns…” |
| Check for Already Written Topic | IF | Prevent duplicate article creation | Get Approved Topic | Set Status to Approved / No Operation, do nothing1 | Same overall note |
| No Operation, do nothing1 | NoOp | Sink for already-written topics | Check for Already Written Topic (false) | — | Same overall note |
| Set Status to Approved | Data Table (update) | Mark idea as approved in ideas table | Check for Already Written Topic (true) | Add Approved Topic | “Create 2 Data Tables… columns…” |
| Add Approved Topic | Data Table (insert) | Insert into Approved Topics as in-process | Set Status to Approved | Research about Approved Topic | “Create 2 Data Tables… columns…” |
| Research about Approved Topic | Perplexity | Produce research summary | Add Approved Topic | Title Generation | “Add your Perplexity API credentials…” |
| Google Gemini Chat Model4 | Gemini Chat Model (LangChain) | LLM provider for title chain | — (AI port) | Title Generation (AI) | Same overall note |
| Title Generation | Chain LLM | Generate SEO title | Research about Approved Topic | Content Generation | Same overall note |
| Google Gemini Chat Model1 | Gemini Chat Model (LangChain) | LLM provider for content agent | — (AI port) | Content Generation (AI) | “Add Author Name & Writing Style…” |
| Content Generation | LangChain Agent | Generate 2000-word WP HTML article | Title Generation | Slug Generation | “Add Author Name & Writing Style…” |
| Google Gemini Chat Model5 | Gemini Chat Model (LangChain) | LLM provider for slug chain | — (AI port) | Slug Generation (AI) | Same overall note |
| Slug Generation | Chain LLM | Generate URL slug | Content Generation | WP Post Creation | Same overall note |
| WP Post Creation | WordPress | Create WP post (draft) | Slug Generation | Add Title & URL to Approved Topic | “Configure WordPress credentials and fields…” |
| Add Title & URL to Approved Topic | Data Table (update) | Store final title + URL in approved table | WP Post Creation | Notify User about Article Creation | “Create 2 Data Tables… columns…” |
| Notify User about Article Creation | Telegram | Notify user of created post | Add Title & URL to Approved Topic | — | “Add your Telegram chat ID and API credentials…” |
| Sticky Note | Sticky Note | Documentation / setup notes | — | — | (content includes links: https://cal.com/mattakshitij/workflow-troubleshoot and https://x.com/matta_kshitij) |
| Sticky Note1 | Sticky Note | Subreddits guidance | — | — | Add at least 10-15 subreddits relevant to your niche |
| Sticky Note2 | Sticky Note | Data tables schema | — | — | Create 2 Data Tables… columns listed |
| Sticky Note3 | Sticky Note | Author/niche placeholders | — | — | Replace placeholder info in [] |
| Sticky Note4 | Sticky Note | Telegram config | — | — | Add Telegram chat ID and API credentials |
| Sticky Note5 | Sticky Note | Perplexity config | — | — | Configure Perplexity API credentials |
| Sticky Note6 | Sticky Note | Author style | — | — | Add Author Name & Writing Style |
| Sticky Note7 | Sticky Note | WordPress config | — | — | Configure WordPress credentials and fields |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Reddit OAuth2 credentials (Reddit app + secret + redirect URI in n8n).
   2. Telegram Bot credentials (create bot via BotFather; paste token into n8n Telegram credential).
   3. Perplexity API credential (API key).
   4. WordPress credential (WordPress REST API access; often Application Password).
   5. Google Gemini/PaLM credentials for **three** Gemini model nodes (or reuse one credential).

2) **Create Data Tables**
   1. Data Table: **Reddit Article Ideas** with columns:
      - `post_id` (string)
      - `topic` (string)
      - `status` (string)
   2. Data Table: **Reddit Approved Topics** with columns:
      - `post_id` (string)
      - `topic` (string)
      - `status` (string)
      - `title` (string)
      - `url` (string)

3) **Build Block A (schedule → Reddit → dedupe)**
   1. Add **Schedule Trigger** node named “Periodic Check for Reddit Posts”.
      - Set schedule to run daily at hour **19** (adjust to your needs).
   2. Add **Reddit** node “Get Reddit Posts”.
      - Operation: **Get All**
      - Category/filter: **new**
      - Limit: **10**
      - Configure subreddit(s)/source listing in the node UI (add 10–15 relevant subreddits as per note).
      - Select Reddit OAuth2 credential.
   3. Add **Remove Duplicates** node “Remove Duplicate Reddit Posts Fetched Earlier”.
      - Operation: **Remove items seen in previous executions**
      - Dedupe value expression: `{{$json.title}}`
   4. Connect: Schedule Trigger → Reddit → Remove Duplicates.

4) **Build Block B (Gemini idea → Set fields)**
   1. Add **Google Gemini Chat Model** node “Google Gemini Chat Model”.
      - Select Gemini credential.
   2. Add **LangChain Agent** node “Generate Article Idea”.
      - Prompt type: **Define**
      - Use the provided prompt (replace placeholders like `[your niche]`, `[Your Author Persona]`).
      - Connect the Gemini model node to the agent via the **AI language model** connector.
   3. Add **Set** node “Set Topic Data”.
      - Add fields:
        - `post_id` = `{{$node["Get Reddit Posts"].json["id"]}}`
        - `article_topic` = `{{$json.output}}`
   4. Connect: Remove Duplicates → Generate Article Idea → Set Topic Data.

5) **Build Block C (store idea → send to Telegram)**
   1. Add **Data Table** node “Add Topic Data to DataTable”.
      - Target table: **Reddit Article Ideas**
      - Insert values:
        - `post_id` = `{{$json.post_id}}`
        - `topic` = `{{$json.article_topic}}`
        - `status` = `not approved`
   2. Add **Telegram** node “Send Article Ideas to User”.
      - Configure **Chat ID** (your Telegram user/chat).
      - Message text:
        - `New article idea:\n\n{{$node["Set Topic Data"].json["article_topic"]}}\n\nWould you like to approve or disapprove this idea?`
      - Reply Markup: **Inline Keyboard**
      - Buttons:
        - Approve → callback_data: `approve_{{$node["Set Topic Data"].json["post_id"]}}`
        - Disapprove → callback_data: `disapprove_{{$node["Set Topic Data"].json["post_id"]}}`
   3. Connect: Set Topic Data → Add Topic Data to DataTable → Send Article Ideas to User.

6) **Build Block D (Telegram trigger → approve routing)**
   1. Add **Telegram Trigger** node “Telegram Webhook”.
      - Updates: **callback_query**
   2. Add **IF** node “Approval Check”.
      - Condition: `{{$node["Telegram Webhook"].json["callback_query"]["data"]}}` starts with `approve_`
   3. Add **NoOp** node “No Operation, do nothing” for disapprovals.
   4. Connect: Telegram Webhook → Approval Check (false output) → NoOp.

7) **Build Block E (check already approved)**
   1. Add **Data Table** node “Get Approved Topic”.
      - Table: **Reddit Approved Topics**
      - Operation: **Get**
      - Filter where `topic` equals:
        - `{{$json.callback_query.message.text.split('\n\n')[1] || ''}}`
      - Enable “Always output data”.
   2. Add **IF** node “Check for Already Written Topic”.
      - Condition: left (Telegram parsed topic) **notEquals** right (`{{$json.topic}}`)
   3. Add **NoOp** node “No Operation, do nothing1” for already-written topics.
   4. Connect: Approval Check (true) → Get Approved Topic → Check for Already Written Topic (false) → NoOp.

8) **Build Block F (update status → add approved topic → Perplexity)**
   1. Add **Data Table** node “Set Status to Approved”.
      - Table: **Reddit Article Ideas**
      - Operation: **Update**
      - Filter: `topic` equals Telegram parsed topic
      - Set `status` = `approved`
   2. Add **Data Table** node “Add Approved Topic”.
      - Table: **Reddit Approved Topics**
      - Insert values:
        - `topic` = `{{$json.topic}}`
        - `status` = `in process`
        - `post_id` = `{{$json.post_id}}`
      - (If your update node does not pass `post_id/topic`, add an intermediate Set node to carry these fields.)
   3. Add **Perplexity** node “Research about Approved Topic”.
      - Model: `sonar-pro`
      - Prompt includes `{{$json.topic}}` and requests ~700-word detailed summary
      - Select Perplexity credential.
   4. Connect: Check for Already Written Topic (true) → Set Status to Approved → Add Approved Topic → Research about Approved Topic.

9) **Build Block G (title → content → slug with Gemini)**
   1. Add **Gemini model** node “Google Gemini Chat Model4” (for title).
   2. Add **Chain LLM** node “Title Generation”.
      - Prompt uses: `{{$('Add Approved Topic').item.json.topic}}`
      - Connect Gemini Chat Model4 to Title Generation via AI port.
   3. Add **Gemini model** node “Google Gemini Chat Model1” (for content).
   4. Add **LangChain Agent** node “Content Generation”.
      - Use provided long prompt.
      - Replace placeholders: `[Author's Name]`, `## Author writing summary`, niche placeholders.
      - Ensure it references Perplexity summary:
        - `{{$('Research about Approved Topic').item.json.message}}`
      - Connect Gemini Chat Model1 via AI port.
   5. Add **Gemini model** node “Google Gemini Chat Model5” (for slug).
   6. Add **Chain LLM** node “Slug Generation”.
      - Prompt uses input (currently `{{$json.output}}` from Content Generation output).
      - Connect Gemini Chat Model5 via AI port.
   7. Connect: Research about Approved Topic → Title Generation → Content Generation → Slug Generation.

10) **Build Block H (WordPress → update table → notify Telegram)**
   1. Add **WordPress** node “WP Post Creation”.
      - Title: `{{$('Title Generation').item.json.text}}`
      - Content: `{{$('Content Generation').item.json.output}}`
      - Slug: `{{$json.text}}` (from Slug Generation)
      - Status: `draft` (or change to `publish`)
      - Select WordPress credential.
   2. Add **Data Table** node “Add Title & URL to Approved Topic”.
      - Table: **Reddit Approved Topics**
      - Operation: **Update**
      - Filter: `topic` equals `{{$('Add Approved Topic').item.json.topic}}`
      - Set:
        - `title` = `{{$('Title Generation').item.json.text}}`
        - `url` = `yourwebsite.com/{{$('Slug Generation').item.json.text}}` (replace with your real domain, ideally `https://example.com/...`)
   3. Add **Telegram** node “Notify User about Article Creation”.
      - Text: `Your article with title: {{$('Title Generation').item.json.text}} has been published.\n\nUrl: {{$json.url}}`
   4. Connect: Slug Generation → WP Post Creation → Add Title & URL to Approved Topic → Notify User about Article Creation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Create wordpress posts with reddit content via telegram bot & perplexity” + explanation of blocks and setup steps | Sticky note covering the full workflow canvas |
| Schedule a 1:1 Troubleshooting Call | https://cal.com/mattakshitij/workflow-troubleshoot |
| Ping on X | https://x.com/matta_kshitij |
| Data Tables required: “Reddit Article Ideas” and “Reddit Approved Topics” with specified columns | Workflow notes |
| Replace placeholders in prompts (author persona, niche, writing style) | Workflow notes (“Add relevant information…” / “Add Author Name & Writing Style”) |

**Disclaimer (as provided):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.