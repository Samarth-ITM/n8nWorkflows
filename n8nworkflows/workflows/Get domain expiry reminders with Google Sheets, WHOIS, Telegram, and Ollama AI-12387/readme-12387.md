Get domain expiry reminders with Google Sheets, WHOIS, Telegram, and Ollama AI

https://n8nworkflows.xyz/workflows/get-domain-expiry-reminders-with-google-sheets--whois--telegram--and-ollama-ai-12387


# Get domain expiry reminders with Google Sheets, WHOIS, Telegram, and Ollama AI

## disclaimer  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---

# 1. Workflow Overview

**Purpose:**  
This workflow monitors a list of domains stored in **Google Sheets**, fetches their **WHOIS** information from **whois.com**, uses **Ollama (LLM)** to extract key fields (domain, owner, status, expiry date), calculates how many days remain until expiry, and sends a **Telegram** reminder if the domain expires within **90 days**. It then updates a “last_notified” field in the sheet (intended to reduce repeated notifications).

**Target use cases:**
- Individuals/teams managing multiple domains who want daily expiry monitoring.
- Automated reminders via Telegram with AI-based parsing of WHOIS data.

### 1.1 Domain Source & Schedule
Runs daily and loads domain rows from Google Sheets.

### 1.2 Fetch WHOIS HTML & Extract Relevant Text
Queries whois.com per domain and extracts the relevant HTML region.

### 1.3 AI Extraction (WHOIS → Structured Fields)
Uses Ollama + Information Extractor to convert the extracted WHOIS text into structured JSON fields.

### 1.4 Expiration Calculation & Decision
Computes remaining days until expiry and decides whether to notify (≤ 90 days).

### 1.5 Notify & Record
Sends Telegram reminder and updates Google Sheets, then loops to the next domain.

---

# 2. Block-by-Block Analysis

## Block 1 — Domain Source & Schedule
**Overview:** Triggers daily and reads the domain list from Google Sheets, then processes rows in batches to reduce rate-limit risks.

**Nodes involved:**  
- Sticky Note (Domain Source & Schedule)  
- Cron (Daily 08:00)  
- Google Sheets (Read Domains)  
- Split In Batches

### Sticky Note (Domain Source & Schedule)
- **Type/role:** Sticky Note (documentation)
- **Content:** “## Domain Source & Schedule…”
- **Connections:** none
- **Failure modes:** none

### Cron (Daily 08:00)
- **Type/role:** `n8n-nodes-base.cron` – scheduled trigger
- **Config choices:** Runs every day at **08:00** (server timezone)
- **Outputs:** Main → Google Sheets (Read Domains)
- **Edge cases / failures:**
  - Timezone mismatch vs intended local time (depends on n8n instance settings).

### Google Sheets (Read Domains)
- **Type/role:** `n8n-nodes-base.googleSheets` – reads rows from a spreadsheet
- **Config choices:**
  - Document ID: `PUT_YOUR_GOOGLE_SHEET_ID_HERE` (must be replaced)
  - Sheet name: `domains`
  - Notes specify expected columns:  
    `domain`, `expiry_date (YYYY-MM-DD)`, `owner (optional)`, `email (optional)`, `last_notified (optional, YYYY-MM-DD)`
  - Operation is not explicitly shown in the summary, but node is used as a “read” source for rows.
- **Credentials:** Google Sheets OAuth2 (`googleSheetsOAuth2Api`)
- **Outputs:** Main → Split In Batches
- **Edge cases / failures:**
  - Auth/consent problems (OAuth expired, missing scopes).
  - Sheet/tab name mismatch.
  - Missing `domain` column causes downstream URL expression to break or query invalid URL.

### Split In Batches
- **Type/role:** `n8n-nodes-base.splitInBatches` – iterates items in controlled batches
- **Config choices:** Default batch settings (not specified; uses node defaults)
- **Inputs:** from Google Sheets (Read Domains)
- **Outputs:**
  - Main (index 0) → Get raw data html from whois.com (process next item/batch)
  - Main (index 1) is unused in this workflow JSON
- **Edge cases / failures:**
  - If batch size defaults are too large, you may still hit whois.com throttling or Telegram rate limits.
  - If the loop back is incorrect, can stop after first batch; here it loops via “Replace Me”.

---

## Block 2 — Fetch & Extract Domain Data
**Overview:** For each domain row, the workflow fetches WHOIS HTML from whois.com and extracts the `.whois-data` section.

**Nodes involved:**  
- Sticky Note2 (Fetch & Extract Domain Data)  
- Get raw data html from whois.com  
- Extract HTML

### Sticky Note2 (Fetch & Extract Domain Data)
- **Type/role:** Sticky Note (documentation)
- **Content:** “## Fetch & Extract Domain Data…”
- **Failure modes:** none

### Get raw data html from whois.com
- **Type/role:** `n8n-nodes-base.httpRequest` – HTTP fetch
- **Config choices:**
  - URL expression: `https://whois.com/whois/{{$json.domain}}`
  - Uses node “options” defaults (no headers/user-agent configured).
- **Inputs:** from Split In Batches (each item should contain `domain`)
- **Outputs:** Main → Extract HTML
- **Edge cases / failures:**
  - whois.com may block automated requests (CAPTCHA, 403), rate limit, or return different HTML.
  - If `$json.domain` is empty/invalid, request fails or returns irrelevant page.
  - Without a browser-like User-Agent, some providers throttle more aggressively.

### Extract HTML
- **Type/role:** `n8n-nodes-base.html` – extracts content from HTML via CSS selector
- **Config choices:**
  - Operation: Extract HTML Content
  - Extracts key `data` with selector `.whois-data`
- **Inputs:** from HTTP Request (HTML response)
- **Outputs:** Main → Information Extractor
- **Edge cases / failures:**
  - If whois.com changes markup, `.whois-data` may not exist → `data` becomes empty/null.
  - Large HTML payloads can slow parsing.

---

## Block 3 — AI Extraction (WHOIS → Structured Fields)
**Overview:** Uses Ollama as the chat model for an Information Extractor node that transforms raw WHOIS text into a structured JSON object.

**Nodes involved:**  
- Ollama Chat Model  
- Information Extractor

### Ollama Chat Model
- **Type/role:** `@n8n/n8n-nodes-langchain.lmChatOllama` – provides LLM backend to LangChain-based nodes
- **Config choices:**
  - Model: `llama3.1:8b`
  - Options: default
- **Credentials:** `ollamaApi` (must point to reachable Ollama endpoint)
- **Connections:** AI Language Model output → Information Extractor (ai_languageModel)
- **Edge cases / failures:**
  - Ollama not running/reachable from n8n network.
  - Model not pulled locally (`llama3.1:8b` missing) → runtime error.
  - Slow inference causing timeouts under load.

### Information Extractor
- **Type/role:** `@n8n/n8n-nodes-langchain.informationExtractor` – extracts structured info from text using an LLM
- **Config choices:**
  - Text input: `{{ $json.data }}` (from Extract HTML)
  - Schema type: “fromJson”
  - JSON schema example (guides extraction):
    ```json
    {
      "domain": "google.com",
      "owner": "google",
      "status": "client transfer prohibited",
      "expired_date": "2010-01-01"
    }
    ```
- **Inputs:**
  - Main: extracted `data` from HTML node
  - AI Language Model: from Ollama Chat Model
- **Outputs:** Main → Get Date & Time Diff
- **Edge cases / failures:**
  - If `data` is empty/garbled, extraction may hallucinate or return missing fields.
  - Output date format may not match `YYYY-MM-DD`, breaking later date calculations.
  - The node produces `output.*` fields; downstream assumes `output.expired_date` exists.

**Version notes:** Information Extractor is `typeVersion: 1.2`; behavior may differ across n8n versions, especially around schema enforcement and output format.

---

## Block 4 — Expiration Check
**Overview:** Computes the day difference between “today” and the extracted expiry date, then maps final fields into a consistent structure for the notification decision.

**Nodes involved:**  
- Sticky Note6 (Expiration Check)  
- Get Date & Time Diff  
- Mappings row data  
- IF (Should Notify)

### Sticky Note6 (Expiration Check)
- **Type/role:** Sticky Note (documentation)
- **Content:** “## Expiration Check…”
- **Failure modes:** none

### Get Date & Time Diff
- **Type/role:** `n8n-nodes-base.dateTime` – calculates time between dates
- **Config choices:**
  - Operation: `getTimeBetweenDates`
  - Start date: `{{$now.format('yyyy-MM-dd')}}`
  - End date: `{{ $json.output.expired_date }}`
- **Inputs:** from Information Extractor
- **Outputs:** Main → Mappings row data
- **Key variables/fields:**
  - Produces a `timeDifference` object; workflow later uses `timeDifference.days`.
- **Edge cases / failures:**
  - If `output.expired_date` is missing or not parseable, node errors or returns unexpected result.
  - If expiry date is in the past, `days` may be negative (workflow still notifies because negative is `<= 90`).

### Mappings row data
- **Type/role:** `n8n-nodes-base.set` – shapes data into a notification-friendly format
- **Config choices:** Assigns:
  - `domain` = `{{ $('Information Extractor').item.json.output.domain }}`
  - `owner` = `...output.owner`
  - `status` = `...output.status`
  - `expired_date` = `...output.expired_date`
  - `day_diff_expired` = `{{ $json.timeDifference.days }}`
- **Inputs:** from Get Date & Time Diff
- **Outputs:** Main → IF (Should Notify)
- **Edge cases / failures:**
  - Cross-node reference `$('Information Extractor').item...` assumes item alignment remains consistent; this is usually OK in a linear chain, but can break if branching/merging is introduced later.
  - `day_diff_expired` is stored as **string** type in the node config, but used numerically later; coercion usually works, but can be risky if it becomes non-numeric.

### IF (Should Notify)
- **Type/role:** `n8n-nodes-base.if` – conditional routing
- **Config choices:**
  - Condition: `{{ $json.timeDifference.days }} <= 90`
  - Strict type validation enabled (`typeValidation: "strict"`)
- **Inputs:** from Mappings row data
- **Outputs:**
  - **True** → Google Sheets (Update last_notified) and Telegram (Send Reminder) (both in parallel)
  - **False** → no further action (ends for that item)
- **Edge cases / failures:**
  - Because strict validation is enabled, if `timeDifference.days` is not a number (undefined/null/string not coercible), the IF may error.
  - Already-expired domains (negative days) will pass (`-10 <= 90`) and will trigger reminders; if undesired, add a lower bound (e.g., `days >= 0`).

---

## Block 5 — Notify & Update Records (and Loop)
**Overview:** If the domain is within the threshold, send a Telegram message and update the sheet, then loop back to process the next domain.

**Nodes involved:**  
- Sticky Note3 (Notify & Update Records)  
- Telegram (Send Reminder)  
- Google Sheets (Update last_notified)  
- Replace Me (NoOp loop helper)

### Sticky Note3 (Notify & Update Records)
- **Type/role:** Sticky Note (documentation)
- **Content:** “## Notify & Update Records…”
- **Failure modes:** none

### Telegram (Send Reminder)
- **Type/role:** `n8n-nodes-base.telegram` – sends a Telegram message
- **Config choices:**
  - Chat ID: `PUT_YOUR_TELEGRAM_CHAT_ID_HERE` (must be replaced)
  - Text (expression) includes fields from Information Extractor:
    - Domain: `{{ $('Information Extractor').item.json.output.domain }}`
    - Expiry Date: `{{ $('Information Extractor').item.json.output.expired_date }}`
    - Owner: `{{ $('Information Extractor').item.json.output.owner }}`
    - Status: `{{ $('Information Extractor').item.json.output.status }}`
- **Inputs:** from IF (Should Notify) true branch
- **Outputs:** Main → Replace Me
- **Edge cases / failures:**
  - Telegram bot token/credentials missing or invalid.
  - Chat ID wrong (bot cannot message user/group).
  - Rate limits if many domains trigger at once (Split In Batches helps, but no explicit delays are set).

### Google Sheets (Update last_notified)
- **Type/role:** `n8n-nodes-base.googleSheets` – updates a row
- **Config choices:**
  - Operation: `update`
  - Sheet: `domains`
  - Document ID: `PUT_YOUR_GOOGLE_SHEET_ID_HERE`
  - Columns mapping is empty in the exported JSON (`columns.value: {}`), despite note “Updates last_notified based on key=domain”.
- **Inputs:** from IF (Should Notify) true branch
- **Outputs:** Main → Replace Me
- **Edge cases / failures (important):**
  - **As configured, this node is incomplete**: no update fields and no matching key column are defined in the parameters shown. In n8n, an update typically needs:
    - A row identifier (row number) or
    - A “matching column” + value to locate the row
    - Plus the columns to update (e.g., `last_notified = {{$now.format('yyyy-MM-dd')}}`)
  - If left unconfigured, it will fail at runtime or update nothing.

### Replace Me
- **Type/role:** `n8n-nodes-base.noOp` – placeholder/connector node
- **Config choices:** no operation; used to merge both branches and loop
- **Inputs:** from Telegram and from Google Sheets (Update last_notified)
- **Outputs:** Main → Split In Batches (to continue processing remaining items)
- **Edge cases / failures:**
  - If either upstream branch errors, the loop may stop for that item.
  - NoOp is often replaced with a Merge node if synchronization is required (here it just continues when each path reaches it; could cause double-looping if both paths trigger separately in some designs—n8n typically treats each incoming item independently).

---

## Global Documentation Sticky Note
### Sticky Note7
- **Type/role:** Sticky Note (documentation)
- **Main content highlights:**
  - Explains daily schedule, Sheets source, WHOIS fetch, AI extraction, Telegram reminder, and recording notification date.
  - Setup steps + links:
    - Google Sheets credentials: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets  
    - Telegram credentials: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/chat-operations  
  - Contact: LinkedIn https://www.linkedin.com/in/dwicahyas/
- **Failure modes:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Cron (Daily 08:00) | n8n-nodes-base.cron | Daily trigger | — | Google Sheets (Read Domains) | ## Domain Source & Schedule<br>Loads domain data daily from Google Sheets. |
| Google Sheets (Read Domains) | n8n-nodes-base.googleSheets | Read domain list | Cron (Daily 08:00) | Split In Batches | ## Domain Source & Schedule<br>Loads domain data daily from Google Sheets. |
| Split In Batches | n8n-nodes-base.splitInBatches | Iterate domains safely | Google Sheets (Read Domains); Replace Me | Get raw data html from whois.com | ## Domain Source & Schedule<br>Loads domain data daily from Google Sheets. |
| Get raw data html from whois.com | n8n-nodes-base.httpRequest | Fetch WHOIS HTML | Split In Batches | Extract HTML | ## Fetch & Extract Domain Data<br>Retrieves WHOIS data and extracts expiration details using AI. |
| Extract HTML | n8n-nodes-base.html | Extract `.whois-data` text | Get raw data html from whois.com | Information Extractor | ## Fetch & Extract Domain Data<br>Retrieves WHOIS data and extracts expiration details using AI. |
| Ollama Chat Model | @n8n/n8n-nodes-langchain.lmChatOllama | LLM provider for extraction | — | Information Extractor (AI input) | ## Fetch & Extract Domain Data<br>Retrieves WHOIS data and extracts expiration details using AI. |
| Information Extractor | @n8n/n8n-nodes-langchain.informationExtractor | Parse WHOIS text into JSON | Extract HTML; Ollama Chat Model | Get Date & Time Diff | ## Fetch & Extract Domain Data<br>Retrieves WHOIS data and extracts expiration details using AI. |
| Get Date & Time Diff | n8n-nodes-base.dateTime | Compute days until expiry | Information Extractor | Mappings row data | ## Expiration Check<br>Calculates remaining days and decides whether a reminder is needed. |
| Mappings row data | n8n-nodes-base.set | Normalize fields for next steps | Get Date & Time Diff | IF (Should Notify) | ## Expiration Check<br>Calculates remaining days and decides whether a reminder is needed. |
| IF (Should Notify) | n8n-nodes-base.if | Decide if reminder needed | Mappings row data | Google Sheets (Update last_notified); Telegram (Send Reminder) | ## Expiration Check<br>Calculates remaining days and decides whether a reminder is needed. |
| Telegram (Send Reminder) | n8n-nodes-base.telegram | Send reminder message | IF (Should Notify) | Replace Me | ## Notify & Update Records<br>Sends a Telegram reminder and updates the last notification date. |
| Google Sheets (Update last_notified) | n8n-nodes-base.googleSheets | Update notification tracking | IF (Should Notify) | Replace Me | ## Notify & Update Records<br>Sends a Telegram reminder and updates the last notification date. |
| Replace Me | n8n-nodes-base.noOp | Loop connector / placeholder | Telegram (Send Reminder); Google Sheets (Update last_notified) | Split In Batches | ## Notify & Update Records<br>Sends a Telegram reminder and updates the last notification date. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation | — | — | This workflow automatically monitors domain expiration dates and sends reminders…<br>Setup steps with links:<br>https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets<br>https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/chat-operations<br>LinkedIn: https://www.linkedin.com/in/dwicahyas/ |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation | — | — |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation | — | — |  |

*(Note: Sticky notes are documentation-only nodes; they do not connect to the execution flow.)*

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Domain Expired Automated Reminder*.

2. **Add Cron node**
   - Node type: **Cron**
   - Set trigger time: **Every day at 08:00**
   - Connect Cron → Google Sheets (Read Domains)

3. **Add Google Sheets node (Read Domains)**
   - Node type: **Google Sheets**
   - Credentials: configure **Google Sheets OAuth2** (select account)
   - Document ID: your spreadsheet ID
   - Sheet name: `domains`
   - Configure it to **read/get all rows** from the `domains` sheet.
   - Ensure the sheet has at minimum a `domain` column.
   - Connect Google Sheets → Split In Batches

4. **Add Split In Batches**
   - Node type: **Split In Batches**
   - (Optional but recommended) set a conservative batch size (e.g., 10–25) depending on list size.
   - Connect Split In Batches (output 0) → HTTP Request node

5. **Add HTTP Request (WHOIS fetch)**
   - Node type: **HTTP Request**
   - Method: GET
   - URL: `https://whois.com/whois/{{$json.domain}}`
   - (Recommended) add headers like a User-Agent if you face blocking.
   - Connect HTTP Request → HTML node

6. **Add HTML node (Extract HTML)**
   - Node type: **HTML**
   - Operation: **Extract HTML Content**
   - Extraction values:
     - Key: `data`
     - CSS selector: `.whois-data`
   - Connect HTML → Information Extractor

7. **Add Ollama Chat Model**
   - Node type: **Ollama Chat Model** (LangChain)
   - Credentials: configure Ollama API (base URL to your Ollama server)
   - Model: `llama3.1:8b` (ensure it is pulled/available in Ollama)
   - Connect Ollama Chat Model (AI output) → Information Extractor (AI language model input)

8. **Add Information Extractor**
   - Node type: **Information Extractor**
   - Text: `{{ $json.data }}`
   - Schema type: **From JSON**
   - Provide a JSON example schema with keys:
     - `domain`, `owner`, `status`, `expired_date` (YYYY-MM-DD)
   - Connect Information Extractor → Date & Time node

9. **Add Date & Time node (Get Time Between Dates)**
   - Node type: **Date & Time**
   - Operation: **Get Time Between Dates**
   - Start date: `{{$now.format('yyyy-MM-dd')}}`
   - End date: `{{ $json.output.expired_date }}`
   - Connect Date & Time → Set node

10. **Add Set node (Mappings row data)**
    - Node type: **Set**
    - Add fields (expressions):
      - `domain` = `{{ $('Information Extractor').item.json.output.domain }}`
      - `owner` = `{{ $('Information Extractor').item.json.output.owner }}`
      - `status` = `{{ $('Information Extractor').item.json.output.status }}`
      - `expired_date` = `{{ $('Information Extractor').item.json.output.expired_date }}`
      - `day_diff_expired` = `{{ $json.timeDifference.days }}`
    - Connect Set → IF node

11. **Add IF node (Should Notify)**
    - Node type: **IF**
    - Condition (Number): `{{ $json.timeDifference.days }}` **is less than or equal to** `90`
    - (Optional improvement) add `>= 0` to avoid notifying already-expired domains.
    - True output → Telegram and Google Sheets Update (parallel)

12. **Add Telegram node (Send Reminder)**
    - Node type: **Telegram**
    - Credentials: configure Telegram bot token
    - Operation: **Send Message**
    - Chat ID: your chat or group ID
    - Message text: include domain/owner/status/expired_date (as in workflow)
    - Connect Telegram → NoOp (Replace Me)

13. **Add Google Sheets node (Update last_notified)**  
    - Node type: **Google Sheets**
    - Credentials: same OAuth2 as read
    - Operation: **Update**
    - Sheet: `domains`
    - **Important:** configure how to find the row and what to update:
      - Use a matching column (e.g., `domain`) equals `{{$json.domain}}` **or** ensure you keep the row number from the read step.
      - Set `last_notified` to `{{$now.format('yyyy-MM-dd')}}`
    - Connect Google Sheets Update → NoOp (Replace Me)

14. **Add NoOp node (Replace Me)**
    - Node type: **NoOp**
    - Purpose: consolidate the flow and loop back
    - Connect NoOp → Split In Batches (so the next domain row is processed)

15. **Activate the workflow** after testing with a small sample.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Sheets node documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets |
| Telegram node documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/chat-operations |
| Author contact | https://www.linkedin.com/in/dwicahyas/ |
| Design intent | Daily run, WHOIS fetch, AI extraction, Telegram reminder, write-back to prevent duplicates |

---