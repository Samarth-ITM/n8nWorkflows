Create Telegram podcast digests from Apple Podcasts with BrowserAct and Gemini

https://n8nworkflows.xyz/workflows/create-telegram-podcast-digests-from-apple-podcasts-with-browseract-and-gemini-12434


# Create Telegram podcast digests from Apple Podcasts with BrowserAct and Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create Telegram podcast digests from Apple Podcasts using BrowserAct & Gemini  
**Purpose:** Each day, the workflow scrapes Apple Podcasts Top Charts via **BrowserAct**, uses **Google Gemini** to curate and format the results into **Telegram HTML** messages that respect Telegram’s length limits, then posts them sequentially to a Telegram channel/group with a delay to avoid rate limiting.

**Target use cases**
- Daily automated “Top podcasts” digest for Telegram channels
- AI-curated summaries from a scraped chart/list source
- Multi-part Telegram posts with pagination and safe character limits

### 1.1 Logical Blocks
1. **Daily Trigger & Chart Extraction**
   - Runs on a daily schedule and executes a BrowserAct workflow that returns structured scraped data.
2. **AI Curation & Telegram Formatting (Gemini + Structured parsing)**
   - Gemini agent transforms the scraped JSON into an array of Telegram-ready HTML message strings (≤ ~3500 chars each).
3. **Message Splitting, Throttling, and Telegram Delivery**
   - Splits the generated array into items, loops through them sequentially, waits between sends, and posts each message to Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Daily Trigger & Chart Extraction
**Overview:** Triggers once per day and runs a BrowserAct automation (“Top Charts Podcast”) to scrape Apple Podcasts top chart data.

**Nodes involved**
- **Schedule Daily**
- **Run Top Chart Podcast workflow**

#### Node: Schedule Daily
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):** Runs on an interval defined by the schedule rule (the JSON shows an interval object but without explicit timing details; this typically means it needs to be set in the UI).
- **Inputs / outputs:** No inputs. Output goes to **Run Top Chart Podcast workflow**.
- **Edge cases / failures:**
  - Misconfigured schedule rule (no effective trigger time)
  - Workflow inactive (this workflow is currently `active: false`)
- **Version notes:** TypeVersion `1.3` (standard schedule node behavior).

#### Node: Run Top Chart Podcast workflow
- **Type / role:** `BrowserAct` (`browserAct`) — executes a BrowserAct “WORKFLOW” automation.
- **Configuration (interpreted):**
  - **Mode:** Runs a BrowserAct workflow by ID: `72057695044160531`
  - **Workflow config schema:** Contains an optional input called `Apple_Podcast` (marked removed=true in schema). If blank, BrowserAct uses its internal default.
  - **Incognito:** `open_incognito_mode: false`
- **Key variables / expressions:** None in n8n; mapping is defined but no explicit value is provided.
- **Inputs / outputs:**
  - Input: from **Schedule Daily**
  - Output: scraped results in `output` (later referenced as `{{$json.output.string}}` in the AI node)
- **Credentials:** BrowserAct API credential required.
- **Edge cases / failures:**
  - Invalid BrowserAct API key / revoked access
  - Workflow ID not found or not accessible
  - BrowserAct run fails due to Apple Podcasts page layout changes, geo/consent popups, captchas, or network timeouts
  - Output format changes causing downstream parsing failures
- **Version notes:** TypeVersion `1`.

**Sticky notes relevant to this block**
- “### 📻 Step 1: Chart Extraction …” (explains the purpose of this stage)
- The general “Workflow Overview & Setup” note applies to the entire workflow.

---

### Block 2 — AI Curation & Telegram Formatting (Gemini + Structured parsing)
**Overview:** Sends the scraped data to a LangChain Agent powered by Gemini, which generates Telegram HTML-formatted digest messages with pagination. A structured output parser enforces JSON structure and can auto-fix minor formatting issues.

**Nodes involved**
- **Analyze Podcast & Generate Post**
- **Google Gemini Chat Model**
- **Structured Output Parser**

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini` — provides the LLM backing model to the agent and parser pipeline.
- **Configuration (interpreted):**
  - Model: `models/gemini-2.5-pro`
  - Default options (none explicitly set)
- **Inputs / outputs:**
  - Connected via **ai_languageModel** to:
    - **Analyze Podcast & Generate Post**
    - **Structured Output Parser** (as the model provider in this AI chain)
- **Credentials:** Google Gemini (PaLM) API credential required.
- **Edge cases / failures:**
  - Invalid API key / quota exceeded
  - Model name not available in the account/region
  - Latency/timeouts on large payloads (scraped charts can be long)
- **Version notes:** TypeVersion `1`.

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — validates and parses the model output into a defined JSON schema.
- **Configuration (interpreted):**
  - **Auto-fix enabled:** attempts to repair malformed JSON returned by the model.
  - **Schema example:** Expects an object with a single key:
    - `"Telegram": [ "<message1>", "<message2>", ... ]`
  - Each array entry is a Telegram HTML message chunk (intended ≤ 3500 chars).
- **Inputs / outputs:**
  - Receives model context via `ai_languageModel` connection.
  - Produces parsed structure via `ai_outputParser` connection to **Analyze Podcast & Generate Post**.
- **Edge cases / failures:**
  - If Gemini output deviates significantly (wrong keys, extra keys, non-JSON), auto-fix may fail.
  - If the agent returns markdown code fences despite instructions, parsing may fail (auto-fix sometimes removes, but not guaranteed).
- **Version notes:** TypeVersion `1.3`.

#### Node: Analyze Podcast & Generate Post
- **Type / role:** `LangChain Agent` — transforms scraped chart items into Telegram-ready formatted digest messages.
- **Configuration (interpreted):**
  - **Prompt input text:** `={{ $json.output.string }}`  
    Expects BrowserAct output to contain `output.string` with the raw JSON/string representation of scraped items.
  - **Prompt type:** “define” (custom system message is provided).
  - **System message key requirements:**
    - Produce Telegram HTML using only `<b>`, `<i>`, `<a href="...">`
    - Per-item block format with emojis, title link, show name italic, summary, separator line
    - Enforce **3500-char safety limit** (Telegram hard limit 4096)
    - Pagination: add `<i>(To be continued...)</i>` and optionally `<i>(...Continued)</i>`
    - Output must be exactly: `{ "Telegram": [ "...", "..." ] }` (no extra keys, no code fences)
  - **Output parser:** enabled (`hasOutputParser: true`) and connected to **Structured Output Parser**.
- **Inputs / outputs:**
  - Input: from **Run Top Chart Podcast workflow**
  - AI model: from **Google Gemini Chat Model**
  - Output parser: from **Structured Output Parser**
  - Output: to **Split Generated Items**
- **Edge cases / failures:**
  - `output.string` missing or not a string → agent prompt gets empty/invalid content.
  - Excessively large input may exceed model context limits.
  - Hallucinated/invalid HTML tags or unclosed tags can break Telegram rendering.
  - Character counting can be off if HTML entities/encoding differs; using 3500 helps, but not foolproof.
- **Version notes:** TypeVersion `3` (LangChain agent node; behavior can differ across versions).

**Sticky notes relevant to this block**
- “### 🧠 Step 2: AI Curation & Formatting …”

---

### Block 3 — Message Splitting, Throttling, and Telegram Delivery
**Overview:** Splits the AI-produced `Telegram` array into individual items, loops through them sequentially, waits between sends, and posts each chunk to a Telegram channel/group with HTML parse mode.

**Nodes involved**
- **Split Generated Items**
- **Loop Over Items**
- **Avoid Rate Limits**
- **Send Podcast List to User**

#### Node: Split Generated Items
- **Type / role:** `Split Out` — converts an array field into multiple items (1 item per message chunk).
- **Configuration (interpreted):**
  - Field to split: `output.Telegram`
- **Inputs / outputs:**
  - Input: from **Analyze Podcast & Generate Post**
  - Output: to **Loop Over Items**
- **Edge cases / failures:**
  - If `output.Telegram` is missing/not an array (parser failed or wrong structure), node produces zero items or errors.
- **Version notes:** TypeVersion `1`.

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — loops through items sequentially.
- **Configuration (interpreted):**
  - Batch options not set explicitly (defaults apply). Commonly this means it processes one item per iteration unless configured otherwise in UI.
- **Inputs / outputs:**
  - Input: from **Split Generated Items**
  - Output (loop): second output branch goes to **Avoid Rate Limits** (the connections show index `1` used).
  - The node also receives a “continue loop” signal from **Send Podcast List to User**, which is connected back into **Loop Over Items** (standard looping pattern).
- **Edge cases / failures:**
  - If batch size is > 1, Telegram node expression referencing loop item might behave unexpectedly (depending on how items are handled). Typically you want batch size = 1 for sequential posting.
- **Version notes:** TypeVersion `3`.

#### Node: Avoid Rate Limits
- **Type / role:** `Wait` — delay between messages to reduce Telegram rate-limit risk and preserve posting order.
- **Configuration (interpreted):**
  - No explicit wait duration is shown in parameters; it may be configured in UI defaults or left unset (which would be an issue).
- **Inputs / outputs:**
  - Input: from **Loop Over Items**
  - Output: to **Send Podcast List to User**
- **Edge cases / failures:**
  - If wait time is not set, node may not behave as intended.
  - Very short waits can still hit Telegram flood limits for large digests.
- **Version notes:** TypeVersion `1.1`.

#### Node: Send Podcast List to User
- **Type / role:** `Telegram` — sends each digest chunk to a chat/channel.
- **Configuration (interpreted):**
  - **Text:** `={{ $('Loop Over Items').item.json["output.Telegram"] }}`  
    This pulls the current loop item’s `output.Telegram` string.
  - **Chat ID:** placeholder string: `parameters.chatId==@Channel_ID (Use Channel ID or Chat ID )`  
    This must be replaced with an actual numeric chat ID (often negative for channels) or a resolved variable.
  - **Parse mode:** `HTML` (crucial for `<b>`, `<i>`, `<a>` formatting)
- **Inputs / outputs:**
  - Input: from **Avoid Rate Limits**
  - Output: loops back to **Loop Over Items** to continue sending remaining chunks.
- **Credentials:** Telegram Bot API credential required.
- **Edge cases / failures:**
  - Wrong chat ID / bot not admin in channel → 403 errors
  - Invalid HTML (unclosed tags) → Telegram may reject message or render incorrectly
  - Message still exceeds Telegram limit (4096) despite 3500 target → send fails
  - Flood control/rate limiting → 429 errors (mitigate with longer waits)
- **Version notes:** TypeVersion `1.2`.

**Sticky notes relevant to this block**
- “### 🚀 Step 3: Sequential Delivery …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Daily | n8n-nodes-base.scheduleTrigger | Daily workflow trigger | — | Run Top Chart Podcast workflow | ### 📻 Step 1: Chart Extraction\n\nThe workflow triggers daily to execute a BrowserAct automation that scrapes the latest trending episodes and shows from the Apple Podcasts top charts. |
| Run Top Chart Podcast workflow | n8n-nodes-browseract.browserAct | Execute BrowserAct chart scraping workflow | Schedule Daily | Analyze Podcast & Generate Post | ### 📻 Step 1: Chart Extraction\n\nThe workflow triggers daily to execute a BrowserAct automation that scrapes the latest trending episodes and shows from the Apple Podcasts top charts. |
| Analyze Podcast & Generate Post | @n8n/n8n-nodes-langchain.agent | Gemini-driven summarization + Telegram HTML formatting + pagination | Run Top Chart Podcast workflow | Split Generated Items | ### 🧠 Step 2: AI Curation & Formatting\n\nAn AI agent analyzes the scraped podcast data to create engaging HTML-formatted summaries. It automatically calculates character counts to ensure each message stays within Telegram's limits, creating a multi-part series if needed. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider (Gemini) | — (AI connection) | Analyze Podcast & Generate Post; Structured Output Parser | ### 🧠 Step 2: AI Curation & Formatting\n\nAn AI agent analyzes the scraped podcast data to create engaging HTML-formatted summaries. It automatically calculates character counts to ensure each message stays within Telegram's limits, creating a multi-part series if needed. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce `{ "Telegram": [...] }` JSON output, auto-fix minor formatting | Google Gemini Chat Model (AI) | Analyze Podcast & Generate Post (AI output parser) | ### 🧠 Step 2: AI Curation & Formatting\n\nAn AI agent analyzes the scraped podcast data to create engaging HTML-formatted summaries. It automatically calculates character counts to ensure each message stays within Telegram's limits, creating a multi-part series if needed. |
| Split Generated Items | n8n-nodes-base.splitOut | Split Telegram message array into individual items | Analyze Podcast & Generate Post | Loop Over Items | ### 🚀 Step 3: Sequential Delivery\n\nThe generated digest parts are split into individual messages and sent to Telegram. A short wait period is implemented between messages to avoid rate limits and maintain a clean posting order. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Sequential iteration over message chunks | Split Generated Items; Send Podcast List to User | Avoid Rate Limits | ### 🚀 Step 3: Sequential Delivery\n\nThe generated digest parts are split into individual messages and sent to Telegram. A short wait period is implemented between messages to avoid rate limits and maintain a clean posting order. |
| Avoid Rate Limits | n8n-nodes-base.wait | Delay between sends | Loop Over Items | Send Podcast List to User | ### 🚀 Step 3: Sequential Delivery\n\nThe generated digest parts are split into individual messages and sent to Telegram. A short wait period is implemented between messages to avoid rate limits and maintain a clean posting order. |
| Send Podcast List to User | n8n-nodes-base.telegram | Send one Telegram HTML message chunk | Avoid Rate Limits | Loop Over Items | ### 🚀 Step 3: Sequential Delivery\n\nThe generated digest parts are split into individual messages and sent to Telegram. A short wait period is implemented between messages to avoid rate limits and maintain a clean posting order. |
| Documentation | n8n-nodes-base.stickyNote | Global setup notes | — | — | ## ⚡ Workflow Overview & Setup\n\n**Summary:** This automation daily scrapes top podcast charts from Apple Podcasts using BrowserAct and utilizes AI to generate beautifully formatted, multi-part digests for Telegram.\n\n### Requirements\n* **Credentials:** BrowserAct, Google Gemini (PaLM), Telegram.\n* **Mandatory:** BrowserAct API (Template: **Top Charts Podcast**)\n\n### How to Use\n1. **Credentials:** Set up your BrowserAct, Google Gemini, and Telegram Bot API keys in n8n.\n2. **BrowserAct Template:** Ensure you have the **Top Charts Podcast** template saved in your BrowserAct account.\n3. **Configuration:** Update the Telegram node with your target Channel or Group ID.\n\n### Need Help?\nhttps://docs.browseract.com\nhttps://docs.browseract.com\nhttps://docs.browseract.com |
| Step 1 Explanation | n8n-nodes-base.stickyNote | Visual annotation | — | — | ### 📻 Step 1: Chart Extraction\n\nThe workflow triggers daily to execute a BrowserAct automation that scrapes the latest trending episodes and shows from the Apple Podcasts top charts. |
| Step 2 Explanation | n8n-nodes-base.stickyNote | Visual annotation | — | — | ### 🧠 Step 2: AI Curation & Formatting\n\nAn AI agent analyzes the scraped podcast data to create engaging HTML-formatted summaries. It automatically calculates character counts to ensure each message stays within Telegram's limits, creating a multi-part series if needed. |
| Step 3 Explanation | n8n-nodes-base.stickyNote | Visual annotation | — | — | ### 🚀 Step 3: Sequential Delivery\n\nThe generated digest parts are split into individual messages and sent to Telegram. A short wait period is implemented between messages to avoid rate limits and maintain a clean posting order. |
| Sticky Note | n8n-nodes-base.stickyNote | External resource link | — | — | @[youtube](jR_EjiLTIgg) |

> Note: Sticky notes are included as nodes by n8n; they do not connect to the execution graph.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (keep it inactive until credentials and chat IDs are configured).

2. **Add “Schedule Trigger”**
   - Node: **Schedule Trigger** (name it “Schedule Daily”)
   - Configure it to run **daily** at your preferred time (set timezone as needed).

3. **Add BrowserAct node to scrape Apple Podcasts charts**
   - Node: **BrowserAct**
   - Operation/type: **WORKFLOW**
   - Set **Workflow ID** to your BrowserAct workflow (this workflow uses `72057695044160531`)
   - Ensure your BrowserAct workflow returns a payload that includes a field accessible as `output.string` (stringified JSON is acceptable).
   - Credentials: add **BrowserAct API** credentials in n8n and select them here.
   - Connect: **Schedule Daily → Run Top Chart Podcast workflow**

4. **Add Gemini model provider**
   - Node: **Google Gemini Chat Model**
   - Set model to **`models/gemini-2.5-pro`**
   - Credentials: configure **Google Gemini (PaLM) API** in n8n.

5. **Add Structured Output Parser**
   - Node: **Structured Output Parser**
   - Enable **Auto-fix**
   - Provide a schema/example that enforces:
     - an object with exactly one key: `"Telegram"`
     - value is an array of strings (each string is a Telegram message chunk)

6. **Add LangChain Agent to format the digest**
   - Node: **AI Agent** (name it “Analyze Podcast & Generate Post”)
   - Input text: set to an expression pulling BrowserAct output, e.g.:
     - `{{ $json.output.string }}`
   - Provide the **system message** instructions (Telegram HTML format, 3500-char limit, pagination markers, output JSON shape).
   - Enable **Output Parser** and connect the **Structured Output Parser** to the agent.
   - Connect AI wiring:
     - **Google Gemini Chat Model** → Agent (language model connection)
     - **Google Gemini Chat Model** → Structured Output Parser (language model connection), if your n8n version requires it for parser assistance
     - **Structured Output Parser** → Agent (output parser connection)
   - Connect main flow:
     - **Run Top Chart Podcast workflow → Analyze Podcast & Generate Post**

7. **Split the generated messages into items**
   - Node: **Split Out** (name it “Split Generated Items”)
   - Field to split out: `output.Telegram`
   - Connect: **Analyze Podcast & Generate Post → Split Generated Items**

8. **Loop through each message sequentially**
   - Node: **Split In Batches** (name it “Loop Over Items”)
   - Set **batch size to 1** (recommended for ordered posting).
   - Connect: **Split Generated Items → Loop Over Items**

9. **Add a wait/delay node**
   - Node: **Wait** (name it “Avoid Rate Limits”)
   - Configure a wait duration (commonly 1–3 seconds; increase if you post many chunks).
   - Connect: **Loop Over Items → Avoid Rate Limits** (use the looping output branch as appropriate in your pattern)

10. **Add Telegram send node**
   - Node: **Telegram**
   - Operation: **Send Message**
   - **Chat ID:** set your channel/group ID (bot must be admin for channels).
   - **Text:** use the loop item’s message string, e.g.:
     - `{{ $('Loop Over Items').item.json["output.Telegram"] }}`
   - Additional field: **parse_mode = HTML**
   - Credentials: configure **Telegram Bot API** in n8n and select it here.
   - Connect: **Avoid Rate Limits → Send Podcast List to User**

11. **Close the loop**
   - Connect: **Send Podcast List to User → Loop Over Items**  
     This continues until all message chunks are sent.

12. **Test & activate**
   - Run manually once.
   - Confirm:
     - BrowserAct returns expected data
     - Agent output is valid `{ "Telegram": [...] }`
     - Each Telegram message renders correctly (links, bold/italic)
   - Activate the workflow.

**Sub-workflow / external dependency**
- BrowserAct workflow “Top Charts Podcast” must exist in BrowserAct and be accessible by the API key.
- Expected output for n8n: a field reachable as `output.string` containing the scraped items (or adapt the agent input expression to match your BrowserAct output structure).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow runs daily, scrapes Apple Podcasts top charts via BrowserAct, uses Gemini to generate Telegram multi-part digests. Requires BrowserAct, Google Gemini (PaLM), Telegram credentials. Must set Telegram Channel/Group ID. | Included in the “Workflow Overview & Setup” sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| YouTube resource | `@[youtube](jR_EjiLTIgg)` |

