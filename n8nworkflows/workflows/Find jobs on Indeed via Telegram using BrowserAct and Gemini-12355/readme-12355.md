Find jobs on Indeed via Telegram using BrowserAct and Gemini

https://n8nworkflows.xyz/workflows/find-jobs-on-indeed-via-telegram-using-browseract-and-gemini-12355


# Find jobs on Indeed via Telegram using BrowserAct and Gemini

## 1. Workflow Overview

**Purpose:**  
This workflow lets a user send a Telegram message like “Marketing Manager in Austin” and receive a set of formatted job posts sourced from Indeed. It uses AI to (1) extract search parameters (role + location), (2) run a BrowserAct scraping workflow on Indeed, and (3) convert the scraped raw job JSON into Telegram-ready HTML messages.

**Primary use cases:**
- Quick job discovery from a Telegram chat without opening Indeed manually
- Consistent search even when the user forgets to specify a location (defaults to **Brooklyn**)
- Mobile-friendly delivery of job results with short, scannable formatting

### 1.1 Telegram Intake
Receives user messages via Telegram trigger and passes the raw text into the AI extraction step.

### 1.2 Intent & Parameter Extraction (AI)
Uses a Gemini chat model + structured output parser to extract `{ role, location }` from the user message (location defaults to “Brooklyn” if missing).

### 1.3 User Feedback + Live Scraping (BrowserAct)
Sends an immediate confirmation back to Telegram, then calls a BrowserAct workflow (“the Indeed Smart Job Scout”) with extracted role/location to scrape job listings.

### 1.4 AI Post-processing & Telegram Formatting
Uses an agent LLM (via OpenRouter Gemini 2.5 Pro) + structured output parser to turn raw job JSON into a JSON array of Telegram HTML messages (under 3000 chars each), then splits and posts each message to Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Intake
**Overview:** Listens for Telegram messages and starts the workflow with the received chat message payload.  
**Nodes involved:** `User Sends Message to Bot`

#### Node: User Sends Message to Bot
- **Type / role:** Telegram Trigger (`n8n-nodes-base.telegramTrigger`) — workflow entry point.
- **Key configuration (interpreted):**
  - Listens for **update type: `message`**.
  - Uses Telegram bot credentials (`Telegram account`).
- **Key data used later:**
  - `{{$json.message.text}}` (user message content)
  - `{{$json.message.chat.id}}` (chat to reply to), referenced later via `$('User Sends Message to Bot').first()...`
- **Connections:**
  - **Output →** `Analyze user Input`
- **Potential failures / edge cases:**
  - Telegram credential invalid/revoked.
  - Bot not added or user never initiated chat (Telegram bots must be started by user).
  - Non-text messages (stickers, images) may not have `message.text`, causing expression failures downstream unless guarded.

---

### Block 2 — Intent & Parameter Extraction (AI)
**Overview:** Extracts `role` and `location` from the Telegram text using an AI agent, enforcing a default location of “Brooklyn” when missing. A structured output parser ensures valid JSON.  
**Nodes involved:** `Analyze user Input`, `Validate inputs`, `Structured Output1`

#### Node: Validate inputs
- **Type / role:** Google Gemini Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides the LLM used by downstream LangChain nodes in this block.
- **Key configuration:**
  - Uses Google Gemini (PaLM) API credential (`Google Gemini(PaLM) Api account`).
  - No special options set.
- **Connections:**
  - **AI language model →** `Analyze user Input`
  - **AI language model →** `Structured Output1` (so the parser can auto-fix with model help)
- **Potential failures / edge cases:**
  - Google PaLM/Gemini API key invalid, quota exceeded, or model access disabled.
  - Latency/timeouts on model calls.

#### Node: Analyze user Input
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — extracts structured parameters from user text.
- **Key configuration:**
  - **Input text:** `={{ $json.message.text }}`
  - **System message instructs:**
    - Extract role + location
    - If missing location → force `"Brooklyn"`
    - Output **only** a raw JSON string with schema: `{ "role": "String", "location": "String" }`
  - `promptType: define`
  - `hasOutputParser: true` (paired with `Structured Output1`)
- **Connections:**
  - **Input ←** `User Sends Message to Bot`
  - **AI model supplied by ←** `Validate inputs`
  - **Output →** `Process Initialization Alert` and `Extract Job Data` (in parallel)
  - **Output parser →** `Structured Output1`
- **Potential failures / edge cases:**
  - If `message.text` is missing (non-text updates), the agent receives `undefined` and may output invalid JSON.
  - Model may output non-JSON despite instructions; mitigated by structured parser with `autoFix=true`, but not guaranteed.
  - Ambiguous user text (“Find jobs”) may cause vague roles; location still defaults to Brooklyn.

#### Node: Structured Output1
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces `{role, location}` output.
- **Key configuration:**
  - `autoFix: true` (can ask the attached model to correct malformed JSON)
  - JSON schema example: `{ "role": "String", "location": "String" }`
- **Connections:**
  - **Parser input ←** `Analyze user Input`
  - **Model assistance ←** `Validate inputs`
- **Potential failures / edge cases:**
  - If the model response is too malformed or missing required fields, auto-fix can still fail.
  - If role/location are returned but as null/empty strings, downstream BrowserAct may search poorly.

---

### Block 3 — User Feedback + Live Job Scraping (BrowserAct)
**Overview:** Sends the user a confirmation message, then calls BrowserAct to scrape Indeed using extracted parameters.  
**Nodes involved:** `Process Initialization Alert`, `Extract Job Data`

#### Node: Process Initialization Alert
- **Type / role:** Telegram Send Message (`n8n-nodes-base.telegram`) — immediate feedback to the user.
- **Key configuration:**
  - **Text:** `I got you — I will search for {{ $json.output.role }} jobs located {{ $json.output.location }}`
  - **Chat ID:** `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
- **Connections:**
  - **Input ←** `Analyze user Input`
  - No downstream outputs (informational step)
- **Potential failures / edge cases:**
  - Telegram API error (bot blocked, invalid chat id, revoked token).
  - If `Analyze user Input` output doesn’t contain `output.role` / `output.location` due to parsing failures, message may render with blanks or expression error.

#### Node: Extract Job Data
- **Type / role:** BrowserAct (`n8n-nodes-browseract.browserAct`) — runs an external BrowserAct workflow to scrape Indeed.
- **Key configuration:**
  - Runs BrowserAct workflow type: **WORKFLOW**
  - **BrowserAct workflowId:** `70715577277982972`  
    (per sticky note: expected to be the template **“the Indeed Smart Job Scout”**)
  - Passes inputs:
    - `input-Role = {{$json.output.role}}`
    - `input-Location = {{$json.output.location}}`
  - Incognito mode: `false`
  - The BrowserAct node schema also contains an `input-Indeed` field marked removed (not used here).
- **Connections:**
  - **Input ←** `Analyze user Input`
  - **Output →** `Analyze Job Data and Generate Response`
- **Potential failures / edge cases:**
  - BrowserAct credential invalid / revoked.
  - Workflow ID missing or not accessible in the BrowserAct account.
  - Indeed page layout changes break scraping.
  - Rate limiting / bot detection on Indeed.
  - Output format mismatch (if BrowserAct template changes its JSON fields, the next AI step may fail or hallucinate).
- **Sub-workflow reference:**
  - Invokes BrowserAct workflow **ID `70715577277982972`**, described as **“the Indeed Smart Job Scout”** (must exist in BrowserAct).

---

### Block 4 — AI Post-processing & Telegram Formatting
**Overview:** Converts raw scraped job JSON into clean Telegram HTML messages, ensures message length < 3000 chars, splits them into individual items, and posts to Telegram.  
**Nodes involved:** `Analyze Job Data and Generate Response`, `OpenRouter Chat Model`, `Structured Output`, `Gemini`, `Split Out Generated Data`, `Send Job Post to Telegram`

#### Node: OpenRouter Chat Model
- **Type / role:** OpenRouter Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) — provides the LLM used for formatting.
- **Key configuration:**
  - Model: `google/gemini-2.5-pro`
  - Credential: `OpenRouter account`
- **Connections:**
  - **AI language model →** `Analyze Job Data and Generate Response`
- **Potential failures / edge cases:**
  - OpenRouter API key invalid, insufficient credits, model unavailable.
  - Model may generate HTML that Telegram rejects if tags unsupported or malformed.

#### Node: Analyze Job Data and Generate Response
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — transforms raw job JSON into Telegram-ready message strings.
- **Key configuration:**
  - **Input text:** `={{ $json.output.string }}`
    - Assumes BrowserAct returns job data in a field compatible with `output.string`.
  - **System message rules:**
    - Input: JSON list of jobs (job_title, location, benefits, full_job_description)
    - Output: JSON object with a list of message strings under `Telegram`
    - Use **Telegram-supported HTML only**: `<b>, <i>, <a>, <code>, <pre>`
    - Add emojis for scannability
    - Extract salary from description if possible; else “Not specified”
    - Remove boilerplate EOE statements
    - Hard limit: **< 3000 chars per message**
    - If too long, split into multiple parts with “(Part 1) / (Part 2)” in title and ensure tags closed
  - `hasOutputParser: true` (paired with `Structured Output`)
- **Connections:**
  - **Input ←** `Extract Job Data`
  - **AI model supplied by ←** `OpenRouter Chat Model`
  - **Output parser →** `Structured Output`
  - **Output →** `Split Out Generated Data`
- **Potential failures / edge cases:**
  - If BrowserAct output doesn’t contain `output.string`, the agent input is empty and output becomes unreliable.
  - Even with instructions, the model may exceed length or produce invalid HTML; Telegram may reject the message.
  - “Remove duplicates / filter spam” is mentioned in a sticky note, but the system message does not explicitly define deduplication logic; results may still contain duplicates unless the model infers it.

#### Node: Structured Output
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces output schema for Telegram messages.
- **Key configuration:**
  - `autoFix: true`
  - JSON schema example:
    ```json
    { "Telegram": ["User friendly telegram API post 1", "User friendly telegram API post 2"] }
    ```
- **Connections:**
  - **Parser input ←** `Analyze Job Data and Generate Response`
  - **Model assistance ←** `Gemini` (note: parser is wired to Gemini, not OpenRouter)
- **Potential failures / edge cases:**
  - If the agent returns content that cannot be auto-fixed into the expected JSON shape, parsing fails.
  - Using a different model for auto-fix (Gemini PaLM) than generation (OpenRouter Gemini 2.5 Pro) can cause slight formatting drift during repairs.

#### Node: Gemini
- **Type / role:** Google Gemini Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides the LLM used by `Structured Output` for auto-fix.
- **Key configuration:**
  - Credential: `Google Gemini(PaLM) Api account`
- **Connections:**
  - **AI language model →** `Structured Output`
- **Potential failures / edge cases:**
  - Same as other Gemini node: auth/quota/latency issues.

#### Node: Split Out Generated Data
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — converts an array of messages into one item per message.
- **Key configuration:**
  - Field to split: `output.Telegram`
- **Connections:**
  - **Input ←** `Analyze Job Data and Generate Response`
  - **Output →** `Send Job Post to Telegram`
- **Potential failures / edge cases:**
  - If `output.Telegram` is missing or not an array, split fails or yields zero items.

#### Node: Send Job Post to Telegram
- **Type / role:** Telegram Send Message (`n8n-nodes-base.telegram`) — posts each formatted message.
- **Key configuration:**
  - **Text:** `={{ $json["output.Telegram"] }}`
    - After split, each item holds the single message in `output.Telegram`.
  - **Chat ID:** `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
  - **parse_mode:** `HTML` (enables Telegram HTML rendering)
- **Connections:**
  - **Input ←** `Split Out Generated Data`
- **Potential failures / edge cases:**
  - Telegram rejects invalid HTML (unsupported tags, unclosed tags, malformed entities).
  - Message length too long (Telegram hard limit 4096; workflow targets < 3000 but model may fail).
  - Flood control if many messages are sent quickly (Telegram rate limits).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point: receive Telegram messages | — | Analyze user Input | ### 🗣️ Step 1: Intent & Parameter Extraction<br><br>The workflow analyzes Telegram messages to identify the desired job role and location. If the user does not specify a city, the AI agent automatically defaults the search location to "Brooklyn" to ensure the scraper always receives valid inputs. |
| Analyze user Input | langchain.agent | Extract role/location from user text | User Sends Message to Bot | Process Initialization Alert; Extract Job Data | ### 🗣️ Step 1: Intent & Parameter Extraction<br><br>The workflow analyzes Telegram messages to identify the desired job role and location. If the user does not specify a city, the AI agent automatically defaults the search location to "Brooklyn" to ensure the scraper always receives valid inputs. |
| Validate inputs | lmChatGoogleGemini | LLM for extraction + parser autofix | — | (AI) Analyze user Input; (AI) Structured Output1 | ### 🗣️ Step 1: Intent & Parameter Extraction<br><br>The workflow analyzes Telegram messages to identify the desired job role and location. If the user does not specify a city, the AI agent automatically defaults the search location to "Brooklyn" to ensure the scraper always receives valid inputs. |
| Structured Output1 | outputParserStructured | Enforce `{role, location}` | Analyze user Input; (AI) Validate inputs | — | ### 🗣️ Step 1: Intent & Parameter Extraction<br><br>The workflow analyzes Telegram messages to identify the desired job role and location. If the user does not specify a city, the AI agent automatically defaults the search location to "Brooklyn" to ensure the scraper always receives valid inputs. |
| Process Initialization Alert | telegram | Notify user search has started | Analyze user Input | — | ### 🕵️ Step 2: Live Job Scraping<br><br>BrowserAct executes an automated session on Indeed, inputting the extracted Role and Location. It scrapes the latest job results, capturing key details like job titles, salaries, and benefit summaries. |
| Extract Job Data | browserAct | Run BrowserAct workflow to scrape Indeed | Analyze user Input | Analyze Job Data and Generate Response | ### 🕵️ Step 2: Live Job Scraping<br><br>BrowserAct executes an automated session on Indeed, inputting the extracted Role and Location. It scrapes the latest job results, capturing key details like job titles, salaries, and benefit summaries. |
| Analyze Job Data and Generate Response | langchain.agent | Transform scraped JSON → Telegram HTML messages | Extract Job Data | Split Out Generated Data | ### 🧠 Step 3: AI Analysis & Formatting<br><br>A specialized AI agent processes the raw job data. It removes duplicate entries, filters out spam, and formats the valid listings into engaging, emoji-rich HTML messages suitable for mobile reading. |
| OpenRouter Chat Model | lmChatOpenRouter | LLM for job formatting | — | (AI) Analyze Job Data and Generate Response | ### 🧠 Step 3: AI Analysis & Formatting<br><br>A specialized AI agent processes the raw job data. It removes duplicate entries, filters out spam, and formats the valid listings into engaging, emoji-rich HTML messages suitable for mobile reading. |
| Structured Output | outputParserStructured | Enforce `{ Telegram: [...] }` | Analyze Job Data and Generate Response; (AI) Gemini | — | ### 🧠 Step 3: AI Analysis & Formatting<br><br>A specialized AI agent processes the raw job data. It removes duplicate entries, filters out spam, and formats the valid listings into engaging, emoji-rich HTML messages suitable for mobile reading. |
| Gemini | lmChatGoogleGemini | LLM for parser autofix | — | (AI) Structured Output | ### 🧠 Step 3: AI Analysis & Formatting<br><br>A specialized AI agent processes the raw job data. It removes duplicate entries, filters out spam, and formats the valid listings into engaging, emoji-rich HTML messages suitable for mobile reading. |
| Split Out Generated Data | splitOut | Split Telegram message array into items | Analyze Job Data and Generate Response | Send Job Post to Telegram | ### 🧠 Step 3: AI Analysis & Formatting<br><br>A specialized AI agent processes the raw job data. It removes duplicate entries, filters out spam, and formats the valid listings into engaging, emoji-rich HTML messages suitable for mobile reading. |
| Send Job Post to Telegram | telegram | Send each HTML message to Telegram | Split Out Generated Data | — | ### 🧠 Step 3: AI Analysis & Formatting<br><br>A specialized AI agent processes the raw job data. It removes duplicate entries, filters out spam, and formats the valid listings into engaging, emoji-rich HTML messages suitable for mobile reading. |
| Documentation | stickyNote | Workspace note (setup/requirements) | — | — | ## ⚡ Workflow Overview & Setup<br><br>**Summary:** This automation allows users to search for jobs on Indeed via Telegram. It scrapes the latest listings using BrowserAct, filters them with AI, and delivers a formatted digest back to the user.<br><br>### Requirements<br>* **Credentials:** Telegram, BrowserAct, OpenRouter (GPT-4), Google Gemini (PaLM).<br>* **Mandatory:** BrowserAct API (Template: **the Indeed Smart Job Scout**)<br><br>### How to Use<br>1.  **Credentials:** Set up your Telegram Bot, BrowserAct, and AI model credentials in n8n.<br>2.  **BrowserAct Template:** Ensure you have the **the Indeed Smart Job Scout** template saved in your BrowserAct account.<br>3.  **Interaction:** Send a message like "Marketing Manager in Austin" to your Telegram bot. If no location is provided, it defaults to **Brooklyn**.<br><br>### Need Help?<br>[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)<br>[How to Connect n8n to BrowserAct](https://docs.browseract.com)<br>[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | stickyNote | Workspace note (step 1) | — | — | ### 🗣️ Step 1: Intent & Parameter Extraction<br><br>The workflow analyzes Telegram messages to identify the desired job role and location. If the user does not specify a city, the AI agent automatically defaults the search location to "Brooklyn" to ensure the scraper always receives valid inputs. |
| Step 2 Explanation | stickyNote | Workspace note (step 2) | — | — | ### 🕵️ Step 2: Live Job Scraping<br><br>BrowserAct executes an automated session on Indeed, inputting the extracted Role and Location. It scrapes the latest job results, capturing key details like job titles, salaries, and benefit summaries. |
| Step 3 Explanation | stickyNote | Workspace note (step 3) | — | — | ### 🧠 Step 3: AI Analysis & Formatting<br><br>A specialized AI agent processes the raw job data. It removes duplicate entries, filters out spam, and formats the valid listings into engaging, emoji-rich HTML messages suitable for mobile reading. |
| Sticky Note | stickyNote | Workspace note (video) | — | — | @[youtube](X8GQS8nF9j0) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Find specific jobs on Indeed using Telegram, BrowserAct & Gemini**
   - Ensure workflow setting **Execution Order** is `v1` (default in many n8n versions; match if needed).

2. **Add Telegram Trigger**
   - Add node: **Telegram Trigger**
   - Updates: **message**
   - Credentials: connect a **Telegram Bot** (Telegram API token)
   - This is the entry node.

3. **Add Gemini Chat Model for extraction (“Validate inputs”)**
   - Add node: **Google Gemini Chat Model** (LangChain)
   - Credentials: **Google Gemini (PaLM) / Palm API** key
   - Leave options default.

4. **Add LangChain Agent for parameter extraction (“Analyze user Input”)**
   - Add node: **AI Agent** (LangChain Agent)
   - Text input expression: `{{$json.message.text}}`
   - Prompt type: **Define**
   - System message: configure it to:
     - Extract `role` and `location`
     - Default `location` to `"Brooklyn"` if missing
     - Output **only** raw JSON with schema `{ "role": "...", "location": "..." }`
   - Enable/attach output parser (the UI usually offers “Structured Output Parser” connection).

5. **Add Structured Output Parser for `{role, location}` (“Structured Output1”)**
   - Add node: **Structured Output Parser**
   - Enable **Auto-fix**
   - Provide schema example: `{ "role": "String", "location": "String" }`
   - Connect:
     - `Validate inputs` → (AI language model) → `Analyze user Input`
     - `Validate inputs` → (AI language model) → `Structured Output1`
     - `Analyze user Input` → (AI output parser) → `Structured Output1`

6. **Add Telegram “processing” message (“Process Initialization Alert”)**
   - Add node: **Telegram**
   - Operation: **Send Message**
   - Chat ID expression: `{{$('User Sends Message to Bot').first().json.message.chat.id}}`
   - Text: `I got you — I will search for {{$json.output.role}} jobs located {{$json.output.location}}`
   - Connect: `Analyze user Input` → `Process Initialization Alert`

7. **Add BrowserAct node to scrape Indeed (“Extract Job Data”)**
   - Add node: **BrowserAct**
   - Credentials: BrowserAct API key
   - Mode/type: **WORKFLOW**
   - Workflow ID: set to your BrowserAct workflow id (in the provided workflow it is `70715577277982972`)
   - Configure workflow inputs (mapping mode “define below”):
     - `input-Role` = `{{$json.output.role}}`
     - `input-Location` = `{{$json.output.location}}`
   - Connect: `Analyze user Input` → `Extract Job Data`
   - Requirement: In BrowserAct, ensure the template/workflow **“the Indeed Smart Job Scout”** exists and returns job data in the expected JSON form.

8. **Add OpenRouter chat model for formatting**
   - Add node: **OpenRouter Chat Model** (LangChain)
   - Credentials: OpenRouter API key
   - Model: `google/gemini-2.5-pro`

9. **Add LangChain Agent to format job data (“Analyze Job Data and Generate Response”)**
   - Add node: **AI Agent** (LangChain Agent)
   - Text input expression: `{{$json.output.string}}` (adjust if your BrowserAct output field differs)
   - Prompt type: **Define**
   - System message: instruct it to:
     - Convert raw job JSON to Telegram-supported **HTML**
     - Produce JSON object: `{ "Telegram": ["msg1", "msg2", ...] }`
     - Keep each message < 3000 chars; split into parts if needed; close HTML tags
   - Connect models/parsers:
     - `OpenRouter Chat Model` → (AI language model) → `Analyze Job Data and Generate Response`
     - `Extract Job Data` → `Analyze Job Data and Generate Response`

10. **Add Gemini chat model for parser auto-fix (“Gemini”)**
   - Add node: **Google Gemini Chat Model** (LangChain)
   - Credentials: same Gemini/PaLM credential as earlier (or another, but must be valid)

11. **Add Structured Output Parser for `{Telegram:[...]}` (“Structured Output”)**
   - Add node: **Structured Output Parser**
   - Auto-fix: **enabled**
   - Schema example:
     - `{"Telegram":["User friendly telegram API post 1","User friendly telegram API post 2"]}`
   - Connect:
     - `Analyze Job Data and Generate Response` → (AI output parser) → `Structured Output`
     - `Gemini` → (AI language model) → `Structured Output`

12. **Add Split Out node (“Split Out Generated Data”)**
   - Add node: **Split Out**
   - Field to split out: `output.Telegram`
   - Connect: `Analyze Job Data and Generate Response` → `Split Out Generated Data`

13. **Add Telegram sender for final posts (“Send Job Post to Telegram”)**
   - Add node: **Telegram** (Send Message)
   - Chat ID: `{{$('User Sends Message to Bot').first().json.message.chat.id}}`
   - Text: `{{$json["output.Telegram"]}}`
   - Additional fields: `parse_mode = HTML`
   - Connect: `Split Out Generated Data` → `Send Job Post to Telegram`

14. **Credentials checklist**
   - Telegram bot credential (Telegram Trigger + Telegram send nodes)
   - BrowserAct API credential (BrowserAct node)
   - OpenRouter API credential (OpenRouter model node)
   - Google Gemini/PaLM credential (both Gemini model nodes)

15. **Test**
   - Send to the bot: “Data analyst” (should default location to Brooklyn)
   - Send: “Java developer in Austin Texas” (should use provided location)
   - If formatting fails, inspect:
     - BrowserAct output field names (especially whether `output.string` exists)
     - Parser failures in Structured Output nodes

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow summary and setup requirements: Telegram, BrowserAct, OpenRouter, Google Gemini (PaLM). Mandatory BrowserAct template: **the Indeed Smart Job Scout**. | Included in sticky note “Documentation” |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | @[youtube](X8GQS8nF9j0) |