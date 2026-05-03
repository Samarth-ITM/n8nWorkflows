Auto-post curated remote jobs to Telegram with BrowserAct and Gemini

https://n8nworkflows.xyz/workflows/auto-post-curated-remote-jobs-to-telegram-with-browseract-and-gemini-12436


# Auto-post curated remote jobs to Telegram with BrowserAct and Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs daily, scrapes remote job listings from **SimplyHired** and **Remotive** using **BrowserAct**, aggregates and curates them with **Google Gemini** (deduplication + spam/low-quality filtering), formats them as **Telegram HTML posts**, and then **posts each job sequentially** to a Telegram channel with throttling to reduce rate-limit risk.

**Target use cases:**
- Automated “remote jobs” Telegram channels
- Daily content pipelines that require multi-source scraping + AI quality control
- Any scenario where raw scraped listings need strict filtering, deduplication, and consistent social formatting

### 1.1 Trigger & Multi-Source Scraping
Runs on a daily schedule, triggers two BrowserAct scraping workflows in parallel (SimplyHired + Remotive).

### 1.2 Parsing & Normalization (Split to Items)
Each BrowserAct output is a JSON string containing an array. Two Code nodes parse/split into individual n8n items.

### 1.3 Join, Aggregate & AI Curation
Merge both branches, aggregate all items into a single array, send to a Gemini-based AI Agent that returns a curated Telegram-ready JSON array (validated by a structured output parser).

### 1.4 Delivery to Telegram with Throttling
Split the AI-produced array into individual post items, loop through them, wait between sends, and post to the Telegram channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Multi-Source Scraping
**Overview:**  
Starts the workflow on a schedule and launches two BrowserAct scraping workflows concurrently to fetch raw job data from SimplyHired and Remotive.

**Nodes involved:**
- Schedule Daily
- Scrape Jobs Data (SimplyHired)
- Scrape Jobs Data (Remotive)

#### Node: Schedule Daily
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration choices:** Runs on an interval rule (daily intent). The JSON shows `rule.interval` with an empty object; in UI this typically maps to a schedule configuration (ensure it’s actually set to “every day”).
- **Connections:**
  - Outputs to **Scrape Jobs Data (Remotive)** and **Scrape Jobs Data (SimplyHired)** in parallel.
- **Edge cases / failures:**
  - Misconfigured schedule interval could result in never running (or running too frequently).

#### Node: Scrape Jobs Data (SimplyHired)
- **Type / role:** BrowserAct node (`n8n-nodes-browseract.browserAct`) — runs a BrowserAct hosted workflow to scrape job listings.
- **Configuration choices:**
  - Runs a BrowserAct **WORKFLOW** with `workflowId: 68979670614019565`
  - **Timeout:** 7200 seconds (2 hours), suitable for browser automation.
  - Workflow config schema includes an input “Simplyhired” but marked removed; default value in BrowserAct is expected.
  - `open_incognito_mode: false`
- **Credentials:** BrowserAct API credential required.
- **Output expectation:** Produces something like `$json.output.string` containing a JSON array as a string.
- **Connections:** To **Splitting SimplyHired Data**
- **Edge cases / failures:**
  - BrowserAct auth errors / invalid workflowId
  - Timeouts if site is slow or blocked
  - Output shape changes (missing `output.string`)
  - Anti-bot measures / captchas causing empty or malformed output

#### Node: Scrape Jobs Data (Remotive)
- **Type / role:** BrowserAct node — same pattern as above for Remotive.
- **Configuration choices:**
  - `workflowId: 68986628591810026`
  - Timeout 7200 seconds
  - Schema includes “Remotive” input marked removed; defaults expected on BrowserAct side.
- **Credentials:** BrowserAct API credential required.
- **Connections:** To **Splitting Remotive Data**
- **Edge cases / failures:** Same as SimplyHired.

---

### Block 2 — Parsing & Normalization (Split BrowserAct JSON string into items)
**Overview:**  
Converts each scraper’s JSON string output into multiple n8n items (one per job entry) so both feeds can be merged and aggregated consistently.

**Nodes involved:**
- Splitting SimplyHired Data
- Splitting Remotive Data

#### Node: Splitting SimplyHired Data
- **Type / role:** Code node (`n8n-nodes-base.code`) — parses JSON string and emits one item per array element.
- **Key logic & variables:**
  - Reads: `const jsonString = $input.first().json.output.string;`
  - Validates existence; throws if missing.
  - `JSON.parse(jsonString)`; validates it’s an array.
  - Returns: `parsedData.map(item => ({ json: item }))`
- **Error handling:** `alwaysOutputData: true` (but **no onError specified** in this node; unlike Remotive node). If it throws, the workflow run may fail unless globally handled.
- **Connections:** To **Wait for Both Path Outputs** (input index 0).
- **Edge cases / failures:**
  - `output.string` missing or not JSON
  - JSON parses to non-array
  - BrowserAct returning already-parsed JSON (would break because it expects a string)

#### Node: Splitting Remotive Data
- **Type / role:** Code node — same as above.
- **Key logic & variables:** Same path: `$input.first().json.output.string`
- **Error handling:** `onError: continueRegularOutput` and `alwaysOutputData: true`
  - In practice: this node attempts to continue even on errors, but the code explicitly `throw new Error(...)`; behavior depends on n8n’s onError mode. Expect “regular output” to still proceed, potentially with no items.
- **Connections:** To **Wait for Both Path Outputs** (input index 1).
- **Edge cases / failures:** Same as SimplyHired.

---

### Block 3 — Join, Aggregate & AI Curation
**Overview:**  
Synchronizes both scraped feeds, aggregates all jobs into a single payload, then uses a Gemini-powered AI agent to deduplicate/filter/format jobs into Telegram-ready HTML post objects. A structured output parser enforces valid JSON output.

**Nodes involved:**
- Wait for Both Path Outputs
- Merge Branch Outputs
- Analyze Job Data & Generate Response
- Google Gemini
- Structured Output Parser
- Fix Output (configured but effectively unused in this workflow graph)

#### Node: Wait for Both Path Outputs
- **Type / role:** Merge node (`n8n-nodes-base.merge`) — synchronization barrier.
- **Configuration choices:** Default merge behavior (v3.2). In this design it’s used primarily to **wait** until both branches deliver output.
- **Connections:**
  - Input 0 from **Splitting SimplyHired Data**
  - Input 1 from **Splitting Remotive Data**
  - Output to **Merge Branch Outputs**
- **Edge cases / failures:**
  - If one branch outputs zero items (or fails and produces none), downstream aggregation may only include the other source.
  - Depending on merge mode, mismatched item counts can lead to pairing behavior; here it’s followed by an aggregate-all node, which mitigates most pairing concerns but you should confirm the merge node is set to a “pass-through/wait” style in UI.

#### Node: Merge Branch Outputs
- **Type / role:** Aggregate node (`n8n-nodes-base.aggregate`) — collects all items into a single combined structure.
- **Configuration choices:**
  - `aggregate: "aggregateAllItemData"` meaning: all incoming items become one aggregated object/array.
- **Connections:** To **Analyze Job Data & Generate Response**
- **Edge cases / failures:**
  - Large input arrays may cause memory usage spikes.
  - If upstream produced mixed schemas, AI prompt will receive inconsistent fields.

#### Node: Analyze Job Data & Generate Response
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — performs AI reasoning + formatting.
- **Configuration choices (interpreted):**
  - **Prompt input text:** `={{ $json.data }}`  
    This assumes the Aggregate node outputs combined data under `$json.data`. If the actual aggregate output differs (some versions use `$json.data` vs `$json` fields), this is a critical coupling.
  - **System message:** detailed job curator instructions:
    - Deduplicate (prefer entries with `full_job_description`)
    - Remove empty/no-description
    - Remove spam/gig-work patterns (examples given)
    - Salary threshold: discard under $20/hour (with internship exception)
    - Produce **a JSON array** of `{ text, parse_mode, disable_web_page_preview }`
    - `parse_mode = "HTML"`, only allowed tags: `<b>, <i>, <a>, <code>`
    - No markdown/code blocks; must start with `[` and end with `]`
  - **hasOutputParser:** true, connected to a Structured Output Parser.
- **Language model used:** provided via the **Google Gemini** node connection (AI language model input).
- **Output:** expected in `$json.output` (commonly) as structured JSON array.
- **Connections:** To **Split Generated Job Items**
- **Edge cases / failures:**
  - Prompt/data mismatch (if `$json.data` is not what you think)
  - Model may output invalid JSON or include disallowed tags → parser may fail
  - Very large input list may exceed model context limits → truncated or partial output
  - Hallucinated fields (missing `job_link`) or broken HTML escaping for Telegram

#### Node: Google Gemini
- **Type / role:** LLM Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides Gemini model access.
- **Configuration choices:**
  - `modelName: "models/gemini-2.5-pro"`
- **Credentials:** Google Gemini (PaLM) API account.
- **Connections:**
  - Feeds as `ai_languageModel` into **Analyze Job Data & Generate Response**
- **Edge cases / failures:**
  - Invalid API key / quota exceeded
  - Model name not available for your account/region
  - Safety filters or blocked content (unlikely here but possible on scraped text)

#### Node: Structured Output Parser
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — validates/coerces AI output into the required JSON format.
- **Configuration choices:**
  - `autoFix: true` (attempts to repair near-valid JSON)
  - Provides a **JSON schema example** shaped as:
    ```text
    [
      {
        "text": "<b>Senior Developer</b>...",
        "parse_mode": "HTML",
        "disable_web_page_preview": true
      }
    ]
    ```
- **Connections:**
  - Attached to the agent as `ai_outputParser`
  - Uses **Fix Output** as its `ai_languageModel` to help auto-fix.
- **Edge cases / failures:**
  - If output is too malformed, auto-fix may still fail.
  - Telegram HTML requires proper escaping and correct attribute quoting; parser validates JSON, not HTML semantics.

#### Node: Fix Output
- **Type / role:** LLM Chat Model (Gemini) used by the parser’s auto-fix mechanism.
- **Configuration choices:** `models/gemini-2.5-pro`
- **Connections:**
  - Provides `ai_languageModel` to **Structured Output Parser**
- **Edge cases / failures:**
  - If the parser invokes fixes repeatedly, can increase token usage/cost.
  - Same credential/quota constraints as the main Gemini model.

---

### Block 4 — Split & Throttled Telegram Posting
**Overview:**  
Takes the AI-generated array of Telegram post objects, splits it into separate n8n items, then posts each item sequentially to Telegram with a wait between posts to reduce rate limit errors.

**Nodes involved:**
- Split Generated Job Items
- Loop Over Items
- Avoid Rate Limits
- Send Travel List to User Channel

#### Node: Split Generated Job Items
- **Type / role:** Code node — converts the agent output array into one item per job post.
- **Key logic & variables:**
  - Reads: `const rawData = $input.first().json.output;`
  - If `rawData` is a string, `JSON.parse`; otherwise uses it directly.
  - Ensures array, maps to `{ json: item }`
- **Connections:** To **Loop Over Items**
- **Edge cases / failures:**
  - If agent output is stored under a different key than `output`, this will throw.
  - If agent returns an object rather than an array, splitting fails.

#### Node: Loop Over Items
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — processes items sequentially.
- **Configuration choices:** Default options; batch size not explicitly shown, so it may default to 1 (verify in UI).
- **Connections:**
  - Receives items from **Split Generated Job Items**
  - “Continue” output (index 1) goes to **Avoid Rate Limits**
  - After sending, **Send Travel List to User Channel** loops back into **Loop Over Items** to fetch next batch/item.
- **Edge cases / failures:**
  - If batch size > 1, you may send multiple messages without waits unless you place the wait inside the loop correctly.
  - If Telegram send fails and workflow stops, loop will not continue.

#### Node: Avoid Rate Limits
- **Type / role:** Wait node (`n8n-nodes-base.wait`) — throttling between posts.
- **Configuration choices:** Parameters are empty in JSON, meaning it may be configured in UI (or left default). You should set a specific delay (e.g., 1–2 seconds).
- **Connections:** To **Send Travel List to User Channel**
- **Edge cases / failures:**
  - No wait configured → offers no real protection from Telegram rate limits.
  - Excessively long waits may cause runs to exceed execution time limits on some n8n plans.

#### Node: Send Travel List to User Channel
- **Type / role:** Telegram node (`n8n-nodes-base.telegram`) — sends a message to a channel/chat.
- **Configuration choices:**
  - **Text:** `={{ $json.text }}`
  - **chatId:** placeholder string: `parameters.chatId==@Channel ID (Use Channel ID or Chat ID)`
    - This is not a valid expression as-is; you must replace with `@yourchannel` or a numeric chat ID.
  - **additionalFields:**
    - `parse_mode: ={{ $json.parse_mode }}`
    - `disable_web_page_preview: ={{ $json.disable_web_page_preview }}`
- **Credentials:** Telegram Bot token credential required.
- **Connections:** Output loops back to **Loop Over Items** (to send the next job).
- **Edge cases / failures:**
  - Wrong `chatId` format or bot not admin in the channel → 403 errors.
  - Telegram HTML parse errors if AI generates invalid HTML.
  - Rate limits (429) if posting too fast or too many messages.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Daily | Schedule Trigger | Daily entry point | — | Scrape Jobs Data (Remotive); Scrape Jobs Data (SimplyHired) | ### 🕵️ Step 1: Multi-Source Scraping<br>The workflow triggers daily to run dual BrowserAct sessions, scraping the latest remote job listings from SimplyHired and Remotive. The raw data from both sources is then parsed into individual job items for processing. |
| Scrape Jobs Data (SimplyHired) | BrowserAct | Run BrowserAct scraper workflow for SimplyHired | Schedule Daily | Splitting SimplyHired Data | ### 🕵️ Step 1: Multi-Source Scraping<br>The workflow triggers daily to run dual BrowserAct sessions, scraping the latest remote job listings from SimplyHired and Remotive. The raw data from both sources is then parsed into individual job items for processing. |
| Scrape Jobs Data (Remotive) | BrowserAct | Run BrowserAct scraper workflow for Remotive | Schedule Daily | Splitting Remotive Data | ### 🕵️ Step 1: Multi-Source Scraping<br>The workflow triggers daily to run dual BrowserAct sessions, scraping the latest remote job listings from SimplyHired and Remotive. The raw data from both sources is then parsed into individual job items for processing. |
| Splitting SimplyHired Data | Code | Parse `$json.output.string` into individual job items | Scrape Jobs Data (SimplyHired) | Wait for Both Path Outputs | ### 🕵️ Step 1: Multi-Source Scraping<br>The workflow triggers daily to run dual BrowserAct sessions, scraping the latest remote job listings from SimplyHired and Remotive. The raw data from both sources is then parsed into individual job items for processing. |
| Splitting Remotive Data | Code | Parse `$json.output.string` into individual job items | Scrape Jobs Data (Remotive) | Wait for Both Path Outputs | ### 🕵️ Step 1: Multi-Source Scraping<br>The workflow triggers daily to run dual BrowserAct sessions, scraping the latest remote job listings from SimplyHired and Remotive. The raw data from both sources is then parsed into individual job items for processing. |
| Wait for Both Path Outputs | Merge | Synchronize both scrape branches | Splitting SimplyHired Data; Splitting Remotive Data | Merge Branch Outputs | ### 🕵️ Step 1: Multi-Source Scraping<br>The workflow triggers daily to run dual BrowserAct sessions, scraping the latest remote job listings from SimplyHired and Remotive. The raw data from both sources is then parsed into individual job items for processing. |
| Merge Branch Outputs | Aggregate | Combine all job items into one aggregated dataset | Wait for Both Path Outputs | Analyze Job Data & Generate Response | ### 🧠 Step 2: AI Curation & Quality Control<br>An AI agent aggregates the job feeds, performing deduplication and strict filtering. It discards low-tier gig work, spam, and roles below a specific salary threshold while generating professional HTML-formatted posts for Telegram. |
| Analyze Job Data & Generate Response | LangChain Agent | Deduplicate/filter and format Telegram-ready posts | Merge Branch Outputs | Split Generated Job Items | ### 🧠 Step 2: AI Curation & Quality Control<br>An AI agent aggregates the job feeds, performing deduplication and strict filtering. It discards low-tier gig work, spam, and roles below a specific salary threshold while generating professional HTML-formatted posts for Telegram. |
| Google Gemini | Gemini Chat Model | LLM provider for the agent | — | Analyze Job Data & Generate Response (ai_languageModel) | ### 🧠 Step 2: AI Curation & Quality Control<br>An AI agent aggregates the job feeds, performing deduplication and strict filtering. It discards low-tier gig work, spam, and roles below a specific salary threshold while generating professional HTML-formatted posts for Telegram. |
| Structured Output Parser | Structured Output Parser (LangChain) | Validate/repair AI output to required JSON array format | Fix Output (ai_languageModel) | Analyze Job Data & Generate Response (ai_outputParser) | ### 🧠 Step 2: AI Curation & Quality Control<br>An AI agent aggregates the job feeds, performing deduplication and strict filtering. It discards low-tier gig work, spam, and roles below a specific salary threshold while generating professional HTML-formatted posts for Telegram. |
| Fix Output | Gemini Chat Model | Provides model for parser auto-fix | — | Structured Output Parser (ai_languageModel) | ### 🧠 Step 2: AI Curation & Quality Control<br>An AI agent aggregates the job feeds, performing deduplication and strict filtering. It discards low-tier gig work, spam, and roles below a specific salary threshold while generating professional HTML-formatted posts for Telegram. |
| Split Generated Job Items | Code | Split curated array into per-message items | Analyze Job Data & Generate Response | Loop Over Items | ### 🚀 Step 3: Throttled Delivery<br>The curated job list is split into individual items and sent sequentially to the Telegram channel. A wait node is included between posts to respect Telegram's rate limits and ensure consistent delivery. |
| Loop Over Items | Split In Batches | Sequential iteration over messages | Split Generated Job Items; Send Travel List to User Channel | Avoid Rate Limits (continue output) | ### 🚀 Step 3: Throttled Delivery<br>The curated job list is split into individual items and sent sequentially to the Telegram channel. A wait node is included between posts to respect Telegram's rate limits and ensure consistent delivery. |
| Avoid Rate Limits | Wait | Delay between Telegram sends | Loop Over Items | Send Travel List to User Channel | ### 🚀 Step 3: Throttled Delivery<br>The curated job list is split into individual items and sent sequentially to the Telegram channel. A wait node is included between posts to respect Telegram's rate limits and ensure consistent delivery. |
| Send Travel List to User Channel | Telegram | Send formatted job post to Telegram channel | Avoid Rate Limits | Loop Over Items | ### 🚀 Step 3: Throttled Delivery<br>The curated job list is split into individual items and sent sequentially to the Telegram channel. A wait node is included between posts to respect Telegram's rate limits and ensure consistent delivery. |
| Sticky Note | Sticky Note | YouTube link note | — | — | @[youtube](DEBF0ILrM5E) |
| Documentation | Sticky Note | Setup and requirements | — | — | ## ⚡ Workflow Overview & Setup<br><br>**Summary:** This automation daily scrapes remote job listings from SimplyHired and Remotive using BrowserAct, uses AI to filter spam and curate high-quality roles, and auto-posts them to a Telegram channel.<br><br>### Requirements<br>* **Credentials:** BrowserAct, Google Gemini (PaLM), Telegram.<br>* **Mandatory:** BrowserAct API (Template: **Automated Remote Job Fetching & Filtering for Telegram Feed**)<br><br>### How to Use<br>1. **Credentials:** Set up your BrowserAct, Google Gemini, and Telegram Bot API keys in n8n.<br>2. **BrowserAct Template:** Ensure you have the **Automated Remote Job Fetching & Filtering for Telegram Feed** template saved in your BrowserAct account.<br>3. **Configuration:** Update the Telegram node with your target Channel ID (e.g., `@yourchannel`).<br><br>### Need Help?<br>[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)<br>[How to Connect n8n to BrowserAct](https://docs.browseract.com)<br>[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | Sticky Note | Block description | — | — | (content shown above in Step 1) |
| Step 2 Explanation | Sticky Note | Block description | — | — | (content shown above in Step 2) |
| Step 3 Explanation | Sticky Note | Block description | — | — | (content shown above in Step 3) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1. Add node: **Schedule Trigger**
   2. Configure to run **daily** at your desired time (verify it’s not left blank).

2. **Add BrowserAct scrapers (parallel)**
   1. Add node: **BrowserAct** → name it `Scrape Jobs Data (SimplyHired)`
      - Operation/type: **WORKFLOW**
      - Workflow ID: `68979670614019565`
      - Timeout: `7200`
      - Credentials: create/select **BrowserAct API** credential
   2. Add node: **BrowserAct** → name it `Scrape Jobs Data (Remotive)`
      - Operation/type: **WORKFLOW**
      - Workflow ID: `68986628591810026`
      - Timeout: `7200`
      - Credentials: same BrowserAct credential
   3. Connect **Schedule Daily → both BrowserAct nodes** (two outgoing connections).

3. **Parse each BrowserAct output into items**
   1. Add node: **Code** → `Splitting SimplyHired Data`
      - Paste logic that reads `$input.first().json.output.string`, parses JSON array, maps to `{json: item}`
   2. Add node: **Code** → `Splitting Remotive Data`
      - Same logic
      - Optionally set **On Error:** “Continue (regular output)” if you want the workflow to proceed even if one source fails.
   3. Connect:
      - `Scrape Jobs Data (SimplyHired) → Splitting SimplyHired Data`
      - `Scrape Jobs Data (Remotive) → Splitting Remotive Data`

4. **Synchronize both streams**
   1. Add node: **Merge**
      - Use it as a “wait for both” junction (ensure mode supports the behavior you want).
   2. Connect:
      - `Splitting SimplyHired Data → Merge (input 0)`
      - `Splitting Remotive Data → Merge (input 1)`

5. **Aggregate all items into one dataset**
   1. Add node: **Aggregate**
      - Mode: **Aggregate All Item Data** (collect all items)
   2. Connect: `Merge → Aggregate`

6. **Configure Gemini model + AI Agent**
   1. Add node: **Google Gemini Chat Model**
      - Model: `models/gemini-2.5-pro`
      - Credentials: **Google Gemini (PaLM) API**
   2. Add node: **AI Agent (LangChain Agent)**
      - Prompt type: “Define”
      - Text input: reference the aggregated data (match your aggregate output; in this workflow it is `={{ $json.data }}`)
      - System message: paste the curation + formatting rules (dedupe, filter, HTML formatting, JSON array constraints)
      - Enable output parser (has output parser).
   3. Connect Gemini model to the Agent via **AI language model** connection.

7. **Add Structured Output Parser (with auto-fix)**
   1. Add node: **Structured Output Parser**
      - Enable **Auto-fix**
      - Provide an example schema matching:
        - array of objects with keys: `text`, `parse_mode`, `disable_web_page_preview`
   2. Add another **Google Gemini Chat Model** node (or reuse, if supported) named `Fix Output`
      - Same model `models/gemini-2.5-pro`
      - Same credentials
   3. Connect:
      - `Fix Output` → Structured Output Parser (AI language model)
      - Structured Output Parser → Agent (AI output parser)

8. **Split AI output into individual Telegram messages**
   1. Add node: **Code** → `Split Generated Job Items`
      - Read `$input.first().json.output`
      - If string, parse; else use directly; validate array; map to `{json:item}`
   2. Connect: `AI Agent → Split Generated Job Items`

9. **Loop and throttle**
   1. Add node: **Split In Batches** → `Loop Over Items`
      - Set batch size to **1** for safest throttling.
   2. Add node: **Wait** → `Avoid Rate Limits`
      - Configure a delay (commonly 1–3 seconds; increase if you hit 429s).
   3. Connect:
      - `Split Generated Job Items → Loop Over Items`
      - `Loop Over Items (continue output) → Avoid Rate Limits`

10. **Send to Telegram**
   1. Add node: **Telegram**
      - Operation: “Send Message”
      - **chatId:** set to `@yourchannel` (channel username) or numeric chat ID  
        (Replace the placeholder text from the JSON; do not keep it.)
      - **text:** `={{ $json.text }}`
      - Additional fields:
        - `parse_mode`: `={{ $json.parse_mode }}` (should be `HTML`)
        - `disable_web_page_preview`: `={{ $json.disable_web_page_preview }}`
      - Credentials: Telegram Bot token (bot must be admin in the channel).
   2. Connect:
      - `Avoid Rate Limits → Telegram`
      - `Telegram → Loop Over Items` (to continue sending next item)

11. **Activate**
   - Test with manual execution first.
   - Then activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| @[youtube](DEBF0ILrM5E) | Included as a sticky note link in the canvas |
| BrowserAct docs (API key, connecting n8n, templates) | https://docs.browseract.com |
| Workflow requires credentials: BrowserAct, Google Gemini (PaLM), Telegram | Setup note from the “Documentation” sticky note |
| BrowserAct template dependency: “Automated Remote Job Fetching & Filtering for Telegram Feed” | Must exist in your BrowserAct account (per sticky note) |

