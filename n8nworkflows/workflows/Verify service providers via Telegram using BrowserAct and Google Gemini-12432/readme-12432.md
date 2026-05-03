Verify service providers via Telegram using BrowserAct and Google Gemini

https://n8nworkflows.xyz/workflows/verify-service-providers-via-telegram-using-browseract-and-google-gemini-12432


# Verify service providers via Telegram using BrowserAct and Google Gemini

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow receives Telegram messages asking whether a given service provider/business can be trusted in a specific location. It extracts the provider + location using Google Gemini, triggers a BrowserAct workflow to gather **Google local presence data** and **OpenCorporates registry data**, then uses Gemini again to produce a final **verification report** and sends it back via Telegram. If the user message is casual chat or missing location, it falls back to a conversational assistant that asks for the missing location or replies normally.

**Target use cases:**
- Vendor/business legitimacy checks (“Can I trust X in Y?”)
- Quick due diligence using local listings + corporate registry
- Telegram-based interface for verification automation

### Logical blocks
**1.1 Telegram input reception**  
Trigger on Telegram “message” update, pass message into the classifier.

**1.2 Input classification & structured extraction (Gemini + parser)**  
Gemini classifies user intent into `Service | Chat | NoData` and (for Service) extracts `Provider` and `Location` as strict JSON.

**1.3 Branching: Service vs Chat/NoData**  
Switch routes Service requests to BrowserAct verification; Chat/NoData goes to a conversational fallback agent.

**1.4 Data acquisition via BrowserAct (incl. human verification handling)**  
Launch BrowserAct “Vendor Vetting and verification bot” workflow, handle returned status (`finished|paused|failed`). If paused, ask user to complete human verification and wait, then fetch results.

**1.5 Analysis & report generation (Gemini agent)**  
Cross-reference Google results vs OpenCorporates results and produce a Telegram-HTML formatted report.

**1.6 Delivery / user communication**  
Send progress message, human verification instructions if needed, final report, or failure alert. Chat/NoData responses are sent back via Telegram too.

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Input Reception
**Overview:** Receives inbound Telegram messages and exposes chat/user text to the rest of the workflow.  
**Nodes involved:** `User Sends Message to Bot`

#### Node: User Sends Message to Bot
- **Type / role:** `telegramTrigger` (Telegram Trigger) – entry point on new messages.
- **Key configuration (interpreted):**
  - Updates: `message`
  - Uses Telegram credentials (`Telegram account`)
- **Key data used downstream:**
  - `{{$json.message.text}}` (user message)
  - `{{$json.message.chat.id}}` (chat for responses)
- **Connections:**
  - Output → `Validate user inputs`
- **Failure/edge cases:**
  - Telegram webhook/credentials misconfigured → no triggers.
  - Non-text messages (stickers, photos) may not have `message.text`; expressions referencing it can fail unless n8n tolerates missing fields.

---

### 2.2 Input Classification & Structured Extraction (Gemini + Structured Output)
**Overview:** Uses a Gemini-powered agent to classify the user’s intent and extract provider/location into structured JSON; auto-fixes to schema if needed.  
**Nodes involved:** `Validate user inputs`, `Validation bot`, `Structured Output Parser`

#### Node: Validation bot
- **Type / role:** `lmChatGoogleGemini` – provides the LLM for the classification agent.
- **Key configuration:**
  - Uses Google Gemini (PaLM) credentials.
  - Default model selection (no explicit modelName set here).
- **Connections:**
  - AI Language Model → `Validate user inputs` and → `Structured Output Parser`
- **Failure/edge cases:**
  - Invalid/expired API key, quota limits, regional model restrictions.
  - Output quality may vary if the model drifts from “raw JSON only” requirement.

#### Node: Structured Output Parser
- **Type / role:** LangChain structured output parser – ensures JSON schema compliance.
- **Key configuration:**
  - `autoFix: true` (attempt to repair malformed JSON).
  - JSON schema example: `{"Type":"Service","Provider":"...","Location":"..."}`
- **Connections:**
  - AI Output Parser → `Validate user inputs`
- **Failure/edge cases:**
  - If the LLM output is too malformed, auto-fix may still fail.
  - If “Type” differs in casing/spelling, downstream Switch comparisons can miss.

#### Node: Validate user inputs
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` – classification/extraction agent.
- **Key configuration:**
  - Input text: `={{ $json.message.text }}`
  - System message defines strict rules:
    - If service provider question AND location present → output `Type: Service` and extract `Provider` (without location) + `Location`.
    - If missing location → force `Type: NoData`.
    - Greetings/small talk → `Type: Chat`.
    - Output must be raw JSON only.
  - `hasOutputParser: true` (feeds into Structured Output Parser).
- **Connections:**
  - Main output → `Validation Type Switch`
- **Failure/edge cases:**
  - Messages containing ambiguous location (“near me”, “in my area”) may be treated as missing and become `NoData`.
  - Provider names containing geographic tokens may be incorrectly split (rule tries to mitigate).
  - Non-text input will make `text` expression empty/undefined.

---

### 2.3 Routing: Service vs Chat/NoData
**Overview:** Routes structured intent to either the verification pipeline (Service) or conversational fallback (Chat/NoData).  
**Nodes involved:** `Validation Type Switch`

#### Node: Validation Type Switch
- **Type / role:** `switch` – branching on `{{$json.output.Type}}`.
- **Key configuration:**
  - Case-sensitive equals checks:
    - `"Service"` → Service pipeline
    - `"Chat"` → fallback
    - `"NoData"` → fallback
- **Connections:**
  - Service (Output 1) → `Process Initialization Alert` and `Get Vendor Data & Stats` (two parallel outgoing connections)
  - Chat (Output 2) → `Chat bot Agent`
  - NoData (Output 3) → `Chat bot Agent`
- **Failure/edge cases:**
  - If the parsed JSON uses unexpected casing (`"Nodata"`) it will not match (note: later prompt says “Nodata” but switch expects `NoData`).
  - If `output.Type` is missing, no route will match (node will output nothing).

---

### 2.4 Service Verification: Launch BrowserAct & Notify User
**Overview:** Notifies the user that verification has started, then runs a BrowserAct workflow using extracted provider/location.  
**Nodes involved:** `Process Initialization Alert`, `Get Vendor Data & Stats`

#### Node: Process Initialization Alert
- **Type / role:** `telegram` – sends a progress message.
- **Key configuration:**
  - Text uses extracted fields:  
    `Ok, I will Search for {{ $json.output.Provider }} located in {{ $json.output.Location }}...`
  - Chat ID: `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - HTML parse mode enabled.
- **Connections:**
  - No downstream connections (informational).
- **Failure/edge cases:**
  - If user trigger item is not accessible (multi-item execution), the `$('User Sends Message to Bot')...` selector could mismatch in some complex runs.
  - Telegram message send failures (blocked bot, invalid chat id).

#### Node: Get Vendor Data & Stats
- **Type / role:** `browserAct` – triggers a BrowserAct workflow run (remote automation/scraping).
- **Key configuration:**
  - Type: `WORKFLOW`
  - Timeout: `7200` seconds (2 hours)
  - `workflowId: "69094846509475311"` (BrowserAct template instance; sticky note says template should be “Vendor Vetting and verification bot”)
  - Workflow inputs mapped:
    - `input-Provider = {{$json.output.Provider}}`
    - `input-Location = {{$json.output.Location}}`
  - Incognito mode: false
- **Connections:**
  - Main output → `Human verification Switch`
- **Failure/edge cases:**
  - BrowserAct credential/API key invalid.
  - Workflow ID missing/not accessible.
  - BrowserAct may return status `paused` due to CAPTCHA/human verification.
  - Large provider names/odd characters could break downstream scraping depending on template robustness.

---

### 2.5 Human Verification Handling + Retrieval of BrowserAct Results
**Overview:** Reacts to BrowserAct task status: proceed if finished, prompt user if paused, or report failure. If paused, waits 10 minutes then fetches task output via BrowserAct API.  
**Nodes involved:** `Human verification Switch`, `Ask For human verification`, `Give Time to Complete Verification`, `Get Data From BrowserAct`, `Send Failure Alert`

#### Node: Human verification Switch
- **Type / role:** `switch` – checks `{{$json.status}}` from BrowserAct run.
- **Key configuration:**
  - `finished` → go directly to analysis
  - `paused` → prompt user and wait
  - `failed` → send failure alert
- **Connections:**
  - finished → `Analyze Data & Verify Providers`
  - paused → `Ask For human verification`
  - failed → `Send Failure Alert`
- **Failure/edge cases:**
  - If BrowserAct returns other statuses (e.g., `running`), no path matches.
  - Strict string equals: whitespace/casing differences can break routing.

#### Node: Ask For human verification
- **Type / role:** `telegram` – instruct user to complete verification.
- **Key configuration:**
  - Text: `Please complete the human varification.`
  - Chat ID from trigger.
  - HTML parse mode enabled.
- **Connections:**
  - Output → `Give Time to Complete Verification`
- **Failure/edge cases:**
  - User cannot actually complete verification unless BrowserAct provides a mechanism (out-of-band). This workflow assumes the user can do it, but doesn’t provide a link/session context.

#### Node: Give Time to Complete Verification
- **Type / role:** `wait` – delays flow to allow verification to be completed.
- **Key configuration:**
  - Wait: 10 minutes
- **Connections:**
  - Output → `Get Data From BrowserAct`
- **Failure/edge cases:**
  - Human verification may take longer than 10 minutes; then fetch might still show paused.
  - Wait nodes depend on n8n being able to resume executions (requires queue/DB persistence configured properly).

#### Node: Get Data From BrowserAct
- **Type / role:** `httpRequest` – calls BrowserAct API to retrieve task result.
- **Key configuration:**
  - GET `https://api.browseract.com/v2/workflow/get-task`
  - Query parameter: `task_id = {{$('Get Vendor Data & Stats').item.json.id}}`
  - Auth: predefined credential type `browserActApi`
- **Connections:**
  - Output → `Analyze Data & Verify Providers`
- **Failure/edge cases:**
  - Task ID missing if BrowserAct node did not return `id`.
  - Network timeouts, 4xx/5xx responses.
  - If task still `paused` or `running`, analysis may receive incomplete output.

#### Node: Send Failure Alert
- **Type / role:** `telegram` – informs the user of an error.
- **Key configuration:**
  - Text: `Sorry we facing problem right now.`
  - Chat ID from trigger.
- **Connections:** none
- **Failure/edge cases:** Telegram send errors.

---

### 2.6 Analysis & Verification Report Generation (Gemini Agent)
**Overview:** Uses an AI agent to cross-reference Google local listing stats vs OpenCorporates registry data and produce a Telegram-HTML formatted report (Verified / Unverified / Not Found).  
**Nodes involved:** `Analyze Data & Verify Providers`, `Verify Data`

#### Node: Verify Data
- **Type / role:** `lmChatGoogleGemini` – provides the LLM for the analysis agent.
- **Key configuration:**
  - Model: `models/gemini-2.5-pro`
  - Google Gemini credentials
- **Connections:**
  - AI Language Model → `Analyze Data & Verify Providers`
- **Failure/edge cases:**
  - Model availability differs by account/region; `gemini-2.5-pro` may not be enabled for all projects.
  - Output may include unsupported HTML tags despite instruction.

#### Node: Analyze Data & Verify Providers
- **Type / role:** LangChain agent – performs the actual reasoning + report composition.
- **Key configuration:**
  - Input text combines:
    - BrowserAct output: `google and opencorporates data : {{ $json.output.string }}`
    - User query provider: `{{ $('Validate user inputs').first().json.output.Provider }}`
  - System message defines:
    - How to compute market presence (locations, avg rating, review volume, top 3 branches).
    - How to classify: Verified (matched corp registry), Unverified/local only (Google present, OpenCorporates missing), Not found (no Google data).
    - Output must be Telegram-supported HTML only.
- **Connections:**
  - Main output → `Result Post Delivery`
- **Failure/edge cases:**
  - Assumes BrowserAct returns a field at `$json.output.string`; if the BrowserAct API returns differently, this will be empty and analysis may hallucinate.
  - Use of `$('Validate user inputs').first()` assumes a single relevant item; in batch scenarios may mismatch.
  - Telegram HTML requirements: unescaped `<`/`&` can break message rendering.

---

### 2.7 Delivery & Conversational Fallback
**Overview:** Sends the final verification report to the user, or handles Chat/NoData by generating a conversational reply or prompting for missing location.  
**Nodes involved:** `Result Post Delivery`, `Chat bot Agent`, `Chat bot`, `Answer the User`

#### Node: Result Post Delivery
- **Type / role:** `telegram` – sends final report.
- **Key configuration:**
  - Text: `={{ $json.output.Text }}`
  - Chat ID from trigger
  - HTML parse mode enabled
- **Connections:** none
- **Failure/edge cases:**
  - Assumes analysis output contains `output.Text`. If the agent returns plain text (not wrapped), Telegram node may send empty text.
  - Telegram message length limit (~4096 chars): long reports may fail or truncate.

#### Node: Chat bot
- **Type / role:** `lmChatGoogleGemini` – provides LLM for fallback agent.
- **Key configuration:** default Gemini settings/credentials.
- **Connections:**
  - AI Language Model → `Chat bot Agent`
- **Failure/edge cases:** same as other Gemini nodes.

#### Node: Chat bot Agent
- **Type / role:** LangChain agent – fallback response generator.
- **Key configuration:**
  - Input text: `Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
  - System instructions:
    - If input type is `"Nodata"` ask user for location (note mismatch vs `"NoData"` used elsewhere).
    - If `"chat"` respond normally.
    - Output raw text, avoid code block markers.
- **Connections:**
  - Main output → `Answer the User`
- **Failure/edge cases:**
  - Type mismatch: switch routes `NoData`, but agent instruction checks `"Nodata"`; behavior depends on agent’s own logic.
  - If user message is not text, it may generate an irrelevant response.

#### Node: Answer the User
- **Type / role:** `telegram` – sends fallback response.
- **Key configuration:**
  - Text: `={{ $json.output }}`
  - Chat ID from trigger
  - HTML parse mode enabled (even though prompt says “raw text”; HTML mode is fine if output has no tags)
- **Connections:** none
- **Failure/edge cases:**
  - If agent returns an object (e.g., `{"text":...}`) instead of a string, message may be `[object Object]` or empty depending on n8n coercion.

---

### 2.8 Documentation / Sticky Notes (non-executing)
**Overview:** Provides usage requirements, step explanations, and a YouTube reference.  
**Nodes involved:** `Workflow Overview`, `Step 1 Explanation`, `Step 2 Explanation`, `Step 3 Explanation`, `Step 4 Explanation`, `Sticky Note`

These nodes do not connect to execution flow.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point (Telegram message trigger) | — | Validate user inputs | ### 🔍 Step 1: Input Validation\n\nThe workflow analyzes incoming Telegram messages to extract the service provider name and geographic location. If essential data is missing or the input is casual chat, the bot responds accordingly via an AI agent. |
| Validate user inputs | langchain agent | Classify input + extract Provider/Location JSON | User Sends Message to Bot; Validation bot (AI model); Structured Output Parser (parser) | Validation Type Switch | ### 🔍 Step 1: Input Validation\n\nThe workflow analyzes incoming Telegram messages to extract the service provider name and geographic location. If essential data is missing or the input is casual chat, the bot responds accordingly via an AI agent. |
| Validation bot | Google Gemini Chat Model | LLM for validation/extraction | — | Validate user inputs; Structured Output Parser | ### 🔍 Step 1: Input Validation\n\nThe workflow analyzes incoming Telegram messages to extract the service provider name and geographic location. If essential data is missing or the input is casual chat, the bot responds accordingly via an AI agent. |
| Structured Output Parser | LangChain Structured Output Parser | Enforce/repair JSON structure | Validation bot (AI model) | Validate user inputs | ### 🔍 Step 1: Input Validation\n\nThe workflow analyzes incoming Telegram messages to extract the service provider name and geographic location. If essential data is missing or the input is casual chat, the bot responds accordingly via an AI agent. |
| Validation Type Switch | Switch | Route Service vs Chat vs NoData | Validate user inputs | Process Initialization Alert; Get Vendor Data & Stats; Chat bot Agent | ### 🔍 Step 1: Input Validation\n\nThe workflow analyzes incoming Telegram messages to extract the service provider name and geographic location. If essential data is missing or the input is casual chat, the bot responds accordingly via an AI agent. |
| Process Initialization Alert | Telegram | Notify user verification has started | Validation Type Switch | — | ### 🌐 Step 2: Automated Data Extraction\n\nBrowserAct performs a dual search, scraping market presence data (reviews/ratings) from Google and corporate registry status from OpenCorporates. It handles human verification challenges if they arise during the session. |
| Get Vendor Data & Stats | BrowserAct | Run BrowserAct workflow to scrape Google + OpenCorporates | Validation Type Switch | Human verification Switch | ### 🌐 Step 2: Automated Data Extraction\n\nBrowserAct performs a dual search, scraping market presence data (reviews/ratings) from Google and corporate registry status from OpenCorporates. It handles human verification challenges if they arise during the session. |
| Human verification Switch | Switch | Branch on BrowserAct task status | Get Vendor Data & Stats | Analyze Data & Verify Providers; Ask For human verification; Send Failure Alert | ### 🌐 Step 2: Automated Data Extraction\n\nBrowserAct performs a dual search, scraping market presence data (reviews/ratings) from Google and corporate registry status from OpenCorporates. It handles human verification challenges if they arise during the session. |
| Ask For human verification | Telegram | Ask user to complete CAPTCHA/human check | Human verification Switch | Give Time to Complete Verification | ### 🌐 Step 2: Automated Data Extraction\n\nBrowserAct performs a dual search, scraping market presence data (reviews/ratings) from Google and corporate registry status from OpenCorporates. It handles human verification challenges if they arise during the session. |
| Give Time to Complete Verification | Wait | Pause execution for manual verification window | Ask For human verification | Get Data From BrowserAct | ### 🌐 Step 2: Automated Data Extraction\n\nBrowserAct performs a dual search, scraping market presence data (reviews/ratings) from Google and corporate registry status from OpenCorporates. It handles human verification challenges if they arise during the session. |
| Get Data From BrowserAct | HTTP Request | Fetch BrowserAct task results by task_id | Give Time to Complete Verification | Analyze Data & Verify Providers | ### 🌐 Step 2: Automated Data Extraction\n\nBrowserAct performs a dual search, scraping market presence data (reviews/ratings) from Google and corporate registry status from OpenCorporates. It handles human verification challenges if they arise during the session. |
| Send Failure Alert | Telegram | Inform user of BrowserAct failure | Human verification Switch | — | ### 🌐 Step 2: Automated Data Extraction\n\nBrowserAct performs a dual search, scraping market presence data (reviews/ratings) from Google and corporate registry status from OpenCorporates. It handles human verification challenges if they arise during the session. |
| Analyze Data & Verify Providers | langchain agent | Cross-reference Google vs OpenCorporates and draft report | Human verification Switch OR Get Data From BrowserAct; Verify Data (AI model) | Result Post Delivery | ### 📊 Step 3: Analysis & Verification\n\nAn AI agent cross-references the local map listings with official corporate data. It calculates branch ratings, review volumes, and issues a final verification status: Verified, Unverified (Local Only), or Not Found. |
| Verify Data | Google Gemini Chat Model | LLM for verification analysis | — | Analyze Data & Verify Providers | ### 📊 Step 3: Analysis & Verification\n\nAn AI agent cross-references the local map listings with official corporate data. It calculates branch ratings, review volumes, and issues a final verification status: Verified, Unverified (Local Only), or Not Found. |
| Result Post Delivery | Telegram | Send final HTML report | Analyze Data & Verify Providers | — | ### 📊 Step 3: Analysis & Verification\n\nAn AI agent cross-references the local map listings with official corporate data. It calculates branch ratings, review volumes, and issues a final verification status: Verified, Unverified (Local Only), or Not Found. |
| Chat bot Agent | langchain agent | Fallback: chat or ask for location | Validation Type Switch; Chat bot (AI model) | Answer the User | ### 💬 Step 2-2: Conversational Fallback\n\nIf no location is present in the user's message, this branch engages the user in natural conversation or prompts them to provide a location for processing. |
| Chat bot | Google Gemini Chat Model | LLM for fallback agent | — | Chat bot Agent | ### 💬 Step 2-2: Conversational Fallback\n\nIf no location is present in the user's message, this branch engages the user in natural conversation or prompts them to provide a location for processing. |
| Answer the User | Telegram | Send fallback response | Chat bot Agent | — | ### 💬 Step 2-2: Conversational Fallback\n\nIf no location is present in the user's message, this branch engages the user in natural conversation or prompts them to provide a location for processing. |
| Workflow Overview | Sticky Note | Requirements + usage notes | — | — | ## ⚡ Workflow Overview & Setup\n\n**Summary:** This automation verifies service providers by cross-referencing local business listings from Google against official corporate registries via OpenCorporates, delivering a legitimacy report through Telegram.\n\n### Requirements\n* **Credentials:** Telegram, BrowserAct, Google Gemini (PaLM).\n* **Mandatory:** BrowserAct API (Template: **Vendor Vetting and verification bot**)\n\n### How to Use\n1.  **Credentials:** Set up your Telegram, BrowserAct, and Google Gemini credentials in n8n.\n2.  **BrowserAct Template:** Ensure you have the **Vendor Vetting and verification bot** template saved in your BrowserAct account.\n3.  **Interaction:** Send a message to your bot containing a provider name and location (e.g., \"Can I trust Mr Rooter Plumbing in Michigan?\").\n\n### Need Help?\n[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)\n[How to Connect n8n to BrowserAct](https://docs.browseract.com)\n[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | Sticky Note | Step explanation | — | — | ### 🔍 Step 1: Input Validation\n\nThe workflow analyzes incoming Telegram messages to extract the service provider name and geographic location. If essential data is missing or the input is casual chat, the bot responds accordingly via an AI agent. |
| Step 2 Explanation | Sticky Note | Step explanation | — | — | ### 🌐 Step 2: Automated Data Extraction\n\nBrowserAct performs a dual search, scraping market presence data (reviews/ratings) from Google and corporate registry status from OpenCorporates. It handles human verification challenges if they arise during the session. |
| Step 3 Explanation | Sticky Note | Step explanation | — | — | ### 📊 Step 3: Analysis & Verification\n\nAn AI agent cross-references the local map listings with official corporate data. It calculates branch ratings, review volumes, and issues a final verification status: Verified, Unverified (Local Only), or Not Found. |
| Step 4 Explanation | Sticky Note | Step explanation | — | — | ### 💬 Step 2-2: Conversational Fallback\n\nIf no location is present in the user's message, this branch engages the user in natural conversation or prompts them to provide a location for processing. |
| Sticky Note | Sticky Note | External reference | — | — | @[youtube](OS3pWKptxDw) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1) **Telegram API** credential in n8n (bot token).  
   2) **Google Gemini (PaLM) API** credential (GooglePalmApi). Ensure models you plan to use are enabled.  
   3) **BrowserAct API** credential (API key). Confirm you can access BrowserAct workflow runs.

2. **Create Trigger node**
   - Add **Telegram Trigger** node named **“User Sends Message to Bot”**
   - Updates: `message`
   - Select your Telegram credentials
   - This becomes the workflow entry.

3. **Add Gemini model node for validation**
   - Add **Google Gemini Chat Model** node named **“Validation bot”**
   - Select Gemini credentials
   - Leave default model (as in original) unless you want to pin a model.

4. **Add Structured Output Parser**
   - Add **Structured Output Parser** node named **“Structured Output Parser”**
   - Enable **Auto-fix**
   - Provide schema example: `{"Type":"Service","Provider":"extracted_provider","Location":"extracted_location"}`

5. **Add validation/extraction agent**
   - Add **AI Agent** node named **“Validate user inputs”**
   - Text input: `{{$json.message.text}}`
   - Prompt type: “Define”
   - Paste the system message rules (Service with location → JSON; missing location → NoData; chat → Chat; output raw JSON).
   - Enable output parsing and connect:
     - **Validation bot** → (AI Language Model) → **Validate user inputs**
     - **Validation bot** → (AI Language Model) → **Structured Output Parser**
     - **Structured Output Parser** → (AI Output Parser) → **Validate user inputs**

6. **Add routing switch**
   - Add **Switch** node named **“Validation Type Switch”**
   - Add 3 rules (string equals, strict, case-sensitive):
     1) `{{$json.output.Type}} == "Service"`
     2) `{{$json.output.Type}} == "Chat"`
     3) `{{$json.output.Type}} == "NoData"`
   - Connect **Validate user inputs** → **Validation Type Switch**

7. **Service branch: notify user**
   - Add **Telegram** node named **“Process Initialization Alert”**
   - Chat ID: `{{$('User Sends Message to Bot').item.json.message.chat.id}}`
   - Text: `Ok, I will Search for {{ $json.output.Provider }} located in {{ $json.output.Location }} and varify them...`
   - Parse mode: HTML; disable attribution (optional)
   - Connect from Switch “Service” output to this node.

8. **Service branch: start BrowserAct workflow**
   - Add **BrowserAct** node named **“Get Vendor Data & Stats”**
   - Type: WORKFLOW
   - Timeout: 7200 seconds
   - Workflow ID: your saved BrowserAct template run ID (original uses `69094846509475311`)
   - Map inputs:
     - `input-Provider = {{$json.output.Provider}}`
     - `input-Location = {{$json.output.Location}}`
   - Connect from Switch “Service” output to this node as well (parallel with the alert).

9. **Handle BrowserAct task status**
   - Add **Switch** node named **“Human verification Switch”**
   - Rules:
     - `{{$json.status}} == "finished"`
     - `{{$json.status}} == "paused"`
     - `{{$json.status}} == "failed"`
   - Connect **Get Vendor Data & Stats** → **Human verification Switch**

10. **Paused path: request human verification**
   - Add Telegram node **“Ask For human verification”**
     - Chat ID from trigger
     - Text: `Please complete the human varification.`
     - HTML parse mode on
   - Connect “paused” output → this node

11. **Paused path: wait**
   - Add **Wait** node **“Give Time to Complete Verification”**
   - Wait 10 minutes
   - Connect **Ask For human verification** → **Give Time to Complete Verification**

12. **Paused path: fetch task result via API**
   - Add **HTTP Request** node **“Get Data From BrowserAct”**
   - Method: GET (or use “Send Query” enabled as in original)
   - URL: `https://api.browseract.com/v2/workflow/get-task`
   - Auth: BrowserAct predefined credential
   - Query param:
     - `task_id = {{$('Get Vendor Data & Stats').item.json.id}}`
   - Connect **Give Time to Complete Verification** → **Get Data From BrowserAct**

13. **Failed path: send failure**
   - Add Telegram node **“Send Failure Alert”**
   - Chat ID from trigger
   - Text: `Sorry we facing problem right now.`
   - Connect “failed” output → this node

14. **Add Gemini model node for analysis**
   - Add **Google Gemini Chat Model** node named **“Verify Data”**
   - Set model name: `models/gemini-2.5-pro` (or available equivalent)
   - Select Gemini credentials

15. **Add analysis agent**
   - Add **AI Agent** node named **“Analyze Data & Verify Providers”**
   - Text input combining BrowserAct results + provider:
     - `google and opencorporates data : {{ $json.output.string }}, service name (given by user) : {{ $('Validate user inputs').first().json.output.Provider }}`
   - System message: paste the provided HTML-only reporting rules (Verified/Unverified/Not Found).
   - Connect:
     - **Verify Data** → (AI Language Model) → **Analyze Data & Verify Providers**
     - From **Human verification Switch** “finished” output → **Analyze Data & Verify Providers**
     - From **Get Data From BrowserAct** → **Analyze Data & Verify Providers**

16. **Send final report**
   - Add Telegram node **“Result Post Delivery”**
   - Chat ID from trigger
   - Text: `{{$json.output.Text}}`
   - HTML parse mode on
   - Connect **Analyze Data & Verify Providers** → **Result Post Delivery**

17. **Chat/NoData fallback**
   - Add Gemini model node **“Chat bot”** (Gemini credentials, default model OK)
   - Add AI Agent node **“Chat bot Agent”**
     - Text: `Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
     - System message: ask for location if missing; otherwise respond conversationally; output raw text.
     - Connect **Chat bot** → (AI Language Model) → **Chat bot Agent**
   - Add Telegram node **“Answer the User”**
     - Text: `{{$json.output}}`
     - Chat ID from trigger
     - HTML parse mode on
   - Connect Switch outputs:
     - “Chat” → **Chat bot Agent**
     - “NoData” → **Chat bot Agent**
   - Connect **Chat bot Agent** → **Answer the User**

**Important reproduction note:**  
Fix the `"Nodata"` vs `"NoData"` inconsistency by aligning the fallback agent instruction to `"NoData"` (or change the switch to match `"Nodata"`). Otherwise the agent may not reliably prompt for location.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| BrowserAct API key & workflow ID help | https://docs.browseract.com |
| How to connect n8n to BrowserAct | https://docs.browseract.com |
| How to use & customize BrowserAct templates | https://docs.browseract.com |
| YouTube reference (as provided in sticky note): `@[youtube](OS3pWKptxDw)` | May refer to a setup/overview video; not expanded in workflow metadata. |
| Workflow requires BrowserAct template “Vendor Vetting and verification bot” to exist in your BrowserAct account | Mentioned in Workflow Overview sticky note. |