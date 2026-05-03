Scrape verified decision-maker emails from Google with GPT-4.1 and Apify

https://n8nworkflows.xyz/workflows/scrape-verified-decision-maker-emails-from-google-with-gpt-4-1-and-apify-12877


# Scrape verified decision-maker emails from Google with GPT-4.1 and Apify

## 1. Workflow Overview

**Purpose:**  
This workflow searches Google (via Apify’s Google Search Scraper), uses GPT-4.1-mini to generate likely decision-maker email addresses for each result, filters out low-quality patterns, verifies the remaining emails via ValidKit, and logs the verification results into Google Sheets.

**Typical use cases:**
- Building prospect lists for private equity firms (or similar B2B niches) from Google results
- Generating and validating likely executive/partner emails when direct emails are not available
- Creating a repeatable pipeline to go from “Google query → verified emails → spreadsheet”

### 1.1 Trigger & Execution
Starts manually, then runs the full pipeline once.

### 1.2 Google SERP Collection (Apify)
Calls Apify’s Google Search Scraper actor and retrieves dataset items (JSON), including organic results.

### 1.3 Result Normalization
Splits the organic results array into individual items so each search result is processed independently.

### 1.4 AI Email Generation (GPT + Structured Output)
For each organic result, an AI Agent generates **exactly 4** candidate decision-maker emails (structured JSON output).

### 1.5 Low-quality Email Filtering & Deduplication
A JavaScript step removes generic/low-signal local parts and deduplicates by “first name part” per domain.

### 1.6 Email Verification (ValidKit)
Each remaining email is checked via ValidKit’s verification endpoint.

### 1.7 Logging to Google Sheets
Verification output is normalized and appended/updated in a Google Sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Execution
**Overview:** Starts the workflow manually from the n8n UI.  
**Nodes involved:**  
- When clicking ‘Execute workflow’

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point.
- **Configuration choices:** No parameters; runs only when manually executed.
- **Input/Output:**
  - **Input:** None (trigger)
  - **Output →** `HTTP Request`
- **Edge cases / failures:** None (except disabled workflow or UI execution permissions).
- **Sticky note:** “## Trigger”

---

### Block 2 — Google SERP Collection (Apify)
**Overview:** Runs Apify’s Google Search Scraper actor synchronously and fetches dataset items as JSON.  
**Nodes involved:**  
- HTTP Request

#### Node: HTTP Request
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Apify Actor API.
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.apify.com/v2/acts/apify~google-search-scraper/run-sync-get-dataset-items`
  - **Query params:**  
    - `format=json` (requests JSON output)  
    - `clean=true` (returns “cleaned” dataset items when supported)
  - **Headers:**
    - `Authorization: Bearer (apify-api)` (expects your Apify API token)
    - `Content-Type: application/json`
  - **Body (JSON):** runs Google scraping with:
    - `countryCode: us`, `languageCode: en`
    - `queries: "private equity firm US leadership team page"`
    - `maxPagesPerQuery: 3`, `resultsPerPage: 10`
    - includes organic/unfiltered results
- **Key expressions/variables:** Body is provided as an n8n expression string but effectively static JSON.
- **Input/Output:**
  - **Input ←** Manual Trigger
  - **Output →** `Split Out Organic Results`
- **Version-specific notes:** Node typeVersion `4.3` (modern HTTP node; supports structured headers/query/body).
- **Edge cases / failures:**
  - 401/403 if Apify token is missing/invalid
  - Actor run failures (quota, blocked scraping, invalid input)
  - Large datasets can increase runtime and memory usage
- **Sticky note:** “## Apify Scraper (API)”

---

### Block 3 — Result Normalization
**Overview:** Converts each result’s `organicResults` array into individual n8n items for per-result processing.  
**Nodes involved:**  
- Split Out Organic Results

#### Node: Split Out Organic Results
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — itemizer / array splitter.
- **Configuration choices:**
  - Splits the field: `organicResults`
- **Input/Output:**
  - **Input ←** `HTTP Request` (Apify dataset items)
  - **Output →** `AI Extract Owner Emails` (one item per organic result)
- **Edge cases / failures:**
  - If `organicResults` is missing or not an array, it may produce zero items or error (depending on runtime behavior).
- **Sticky note:** “## Apify Scraper (API)”

---

### Block 4 — AI Email Generation (GPT + Structured Output)
**Overview:** Uses a LangChain Agent node with an OpenAI chat model and a structured output parser to produce a consistent JSON object containing company, domain, and exactly four candidate decision-maker emails.  
**Nodes involved:**  
- AI Extract Owner Emails  
- OpenAI Chat Model  
- Structured Output Parser

#### Node: AI Extract Owner Emails
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt + model + parser.
- **Configuration choices:**
  - **Prompt (“text”)** builds a context payload per organic result using expressions:
    - `Company: {{ $('Split Out Organic Results').item.json.title }}`
    - `URL: {{ $('Split Out Organic Results').item.json.url }}`
    - `Domain: {{ $('Split Out Organic Results').item.json.displayedUrl }}`
    - `Website Text Content: {{ $json.description }}`
    - Instruction fallback: if no decision makers, generate patterns like `firstname@website`, `owner@website`, etc. (note: contains typos like “abnd”; doesn’t affect execution but can affect model behavior)
  - **System message** instructs the assistant to:
    - identify owner/managing partner/CEO/founder names
    - extract domain
    - generate **exactly 4** possible email addresses
    - exclude generic emails (info@, contact@, etc.)
    - return in a specified JSON format: `{ company, domain, owner_emails: [...] }`
  - **Output parsing enabled** (`hasOutputParser: true`) and connected to Structured Output Parser.
- **Input/Output:**
  - **Input ←** `Split Out Organic Results`
  - **AI language model input ←** `OpenAI Chat Model` (via `ai_languageModel`)
  - **AI output parser input ←** `Structured Output Parser` (via `ai_outputParser`)
  - **Output →** `Code in JavaScript`
- **Version-specific notes:** typeVersion `3` (agent node behavior and port names matter: `ai_languageModel`, `ai_outputParser`).
- **Edge cases / failures:**
  - Model may fail to follow “exactly 4 emails” unless parser enforces it (parser does enforce min/max=4, causing failures if output deviates)
  - If `displayedUrl` is not a clean domain, generated emails may be malformed (e.g., includes paths)
  - `description` is not actual HTML; it’s SERP snippet text—name extraction may be weak

- **Sticky note:** “## AI Emails”

#### Node: OpenAI Chat Model
- **Type / role:** OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — LLM provider.
- **Configuration choices:**
  - **Model:** `gpt-4.1-mini`
  - Default options (no explicit temperature/max tokens set here)
- **Credentials:**
  - Uses `OpenAi account 2` (OpenAI API key must be configured)
- **Input/Output:**
  - **Output →** feeds `AI Extract Owner Emails` on `ai_languageModel` port
- **Version-specific notes:** typeVersion `1.3`
- **Edge cases / failures:**
  - 401 if API key invalid
  - Rate limits / timeouts
  - Cost spikes with large runs
- **Sticky note:** “## AI Emails”

#### Node: Structured Output Parser
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces JSON schema.
- **Configuration choices:**
  - Manual JSON schema:
    - `company`: string
    - `domain`: string
    - `owner_emails`: array of strings with **minItems=4 and maxItems=4**
- **Input/Output:**
  - **Output →** feeds `AI Extract Owner Emails` on `ai_outputParser` port
- **Version-specific notes:** typeVersion `1.3`
- **Edge cases / failures:**
  - If the model returns invalid JSON or wrong number of emails, the parser will fail and stop that item’s execution.
- **Sticky note:** “## AI Emails”

---

### Block 5 — Low-quality Email Filtering & Deduplication
**Overview:** Filters out unwanted local-part patterns (including many generics) and deduplicates candidates per domain before verification.  
**Nodes involved:**  
- Code in JavaScript

#### Node: Code in JavaScript
- **Type / role:** Code (`n8n-nodes-base.code`) — custom filtering/transform.
- **Configuration choices (interpreted logic):**
  - Defines `fillers` array including: `firstname`, `lastname`, `owner`, `founder`, `ceo`, `admin`, `info`, `office`, `contact`, `support`, `help`, `our`, `managing.partner`, `team`, `blastname`
  - Reads AI output from: `item.json.output.company`, `item.json.output.domain`, `item.json.output.owner_emails`
  - Skips items with no emails
  - For each email:
    - lowercases local part (before `@`)
    - **filters out** any local part containing any filler substring
    - deduplicates per domain using first token split on `.` `-` `_` (e.g., `john.smith` → `john`)
  - Outputs items shaped as: `{ company, domain, email }`
- **Input/Output:**
  - **Input ←** `AI Extract Owner Emails`
  - **Output →** `Verify Email`
- **Version-specific notes:** typeVersion `2`
- **Edge cases / failures:**
  - If the AI output is not located at `json.output` (e.g., different agent output structure), this will produce “Unknown Company/unknown.com” and likely skip emails
  - The filler list includes `ceo`, `founder`, `owner`, etc., which contradicts the agent fallback instruction (it may remove many fallback emails intentionally)
  - Deduping by first token can wrongly drop distinct people sharing the same first name
- **Sticky note:** “## Remove Low-Quality Emails”

---

### Block 6 — Email Verification (ValidKit)
**Overview:** Calls ValidKit to verify each email and returns deliverability-related fields.  
**Nodes involved:**  
- Verify Email

#### Node: Verify Email
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — ValidKit verification call.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://api.validkit.com/api/v1/verify`
  - **Headers:** `X-API-Key: (api key)`
  - **Body params:** `email = {{ $json.email }}`
- **Input/Output:**
  - **Input ←** `Code in JavaScript` (one item per email)
  - **Output →** `Format ValidKit Output`
- **Version-specific notes:** typeVersion `4.3`
- **Edge cases / failures:**
  - 401/403 if API key invalid
  - Provider may throttle or return temporary SMTP check failures
  - Some domains block verification probes → `smtp_check` may be false/unknown even for valid emails
- **Sticky note:** “## Verify Email (ValidKit API)”

---

### Block 7 — Logging to Google Sheets
**Overview:** Normalizes the verification response together with company/domain/email and appends or updates a Google Sheet row.  
**Nodes involved:**  
- Format ValidKit Output  
- Log Verified Emails

#### Node: Format ValidKit Output
- **Type / role:** Set (`n8n-nodes-base.set`) — field mapping/normalization.
- **Configuration choices:**
  - Pulls `company/domain/email` from the **Code node item**:
    - `company = {{ $('Code in JavaScript').item.json.company }}`
    - `domain = {{ $('Code in JavaScript').item.json.domain }}`
    - `email = {{ $('Code in JavaScript').item.json.email }}`
  - Pulls verification fields from ValidKit response (`$json`):
    - `status`, `syntax_valid`, `mx_found`, `smtp_check`, `disposable`
- **Input/Output:**
  - **Input ←** `Verify Email`
  - **Output →** `Log Verified Emails`
- **Version-specific notes:** typeVersion `3.4`
- **Edge cases / failures:**
  - If ValidKit response shape differs (field names changed), mappings will be empty
  - Cross-node item referencing (`$('Code in JavaScript').item...`) assumes item alignment remains consistent; in complex branching this can break, but here it’s linear so it’s stable
- **Sticky note:** “## Log Emails”

#### Node: Log Verified Emails
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — persistence layer.
- **Configuration choices:**
  - **Operation:** `appendOrUpdate`
  - **Document:** Spreadsheet “Leads” (ID: `1_Wfe7SNs97UEN2zSc_IIEbHww_0V2hqh_Nni2o__dsI`)
  - **Sheet/tab:** “Decision Makers” (`gid=0`)
  - **Mapping mode:** auto-map input data to columns (schema includes: company, domain, email, status, syntax_valid, mx_found, smtp_check, disposable)
- **Credentials:**
  - `Google Sheets account 3` (OAuth2)
- **Input/Output:**
  - **Input ←** `Format ValidKit Output`
  - **Output:** None (end)
- **Version-specific notes:** typeVersion `4.7`
- **Edge cases / failures:**
  - OAuth token expiration / missing permissions
  - AppendOrUpdate requires a clear matching strategy; here `matchingColumns` is empty, so behavior may effectively append (or rely on internal defaults). Consider configuring matching columns explicitly (e.g., `email`) to avoid duplicates.
- **Sticky note:** “## Log Emails”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | HTTP Request | ## Trigger |
| HTTP Request | HTTP Request | Run Apify Google Search Scraper actor + fetch results | When clicking ‘Execute workflow’ | Split Out Organic Results | ## Apify Scraper (API) |
| Split Out Organic Results | Split Out | Split `organicResults` into items | HTTP Request | AI Extract Owner Emails | ## Apify Scraper (API) |
| AI Extract Owner Emails | LangChain Agent | Generate 4 candidate decision-maker emails (structured) | Split Out Organic Results | Code in JavaScript | ## AI Emails |
| OpenAI Chat Model | OpenAI Chat Model | LLM backend for agent | — | AI Extract Owner Emails | ## AI Emails |
| Structured Output Parser | Structured Output Parser | Enforce output JSON schema (4 emails) | — | AI Extract Owner Emails | ## AI Emails |
| Code in JavaScript | Code | Filter low-quality emails + dedupe per domain | AI Extract Owner Emails | Verify Email | ## Remove Low-Quality Emails |
| Verify Email | HTTP Request | Verify email deliverability via ValidKit | Code in JavaScript | Format ValidKit Output | ## Verify Email (ValidKit API) |
| Format ValidKit Output | Set | Normalize verification + lead fields | Verify Email | Log Verified Emails | ## Log Emails |
| Log Verified Emails | Google Sheets | Append/update results to spreadsheet | Format ValidKit Output | — | ## Log Emails |
| Sticky Note | Sticky Note | Comment container | — | — |  |
| Sticky Note1 | Sticky Note | Comment container | — | — |  |
| Sticky Note2 | Sticky Note | Comment container | — | — |  |
| Sticky Note3 | Sticky Note | Comment container | — | — |  |
| Sticky Note4 | Sticky Note | Comment container | — | — |  |
| Sticky Note5 | Sticky Note | Comment container | — | — |  |
| Sticky Note6 | Sticky Note | Comment container | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Apify Scraper* (or your preferred name).

2. **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add Apify HTTP Request**
   - Node: **HTTP Request**
   - Name: `HTTP Request`
   - Method: **POST**
   - URL: `https://api.apify.com/v2/acts/apify~google-search-scraper/run-sync-get-dataset-items`
   - Query parameters:
     - `format` = `json`
     - `clean` = `true`
   - Headers:
     - `Authorization` = `Bearer <YOUR_APIFY_TOKEN>`
     - `Content-Type` = `application/json`
   - Body content type: **JSON**
   - JSON body (edit as needed):
     - `countryCode: "us"`
     - `languageCode: "en"`
     - `queries: "private equity firm US leadership team page"`
     - `maxPagesPerQuery: 3`
     - `resultsPerPage: 10`
     - `includeUnfilteredResults: true`
     - (keep other booleans consistent with your use)
   - **Connect:** Manual Trigger → HTTP Request

4. **Split organic results**
   - Node: **Split Out**
   - Name: `Split Out Organic Results`
   - Field to split out: `organicResults`
   - **Connect:** HTTP Request → Split Out Organic Results

5. **Add OpenAI model credential**
   - In **Credentials**, create **OpenAI API** credential with your API key.

6. **Add OpenAI Chat Model node**
   - Node: **OpenAI Chat Model** (LangChain)
   - Name: `OpenAI Chat Model`
   - Model: `gpt-4.1-mini`
   - Select your OpenAI credential.

7. **Add Structured Output Parser**
   - Node: **Structured Output Parser**
   - Name: `Structured Output Parser`
   - Schema (manual):
     - object with:
       - `company` (string)
       - `domain` (string)
       - `owner_emails` (array of strings, min 4, max 4)

8. **Add AI Agent**
   - Node: **AI Agent** (LangChain Agent)
   - Name: `AI Extract Owner Emails`
   - Prompt text (use expressions):
     - Company from `title`
     - URL from `url`
     - Domain from `displayedUrl`
     - Website text content from the item’s `description`
   - System message: instruct to extract decision-maker name, use the real domain, and output **exactly 4** emails in JSON.
   - **Connect:**
     - Split Out Organic Results → AI Extract Owner Emails (main)
     - OpenAI Chat Model → AI Extract Owner Emails (ai_languageModel)
     - Structured Output Parser → AI Extract Owner Emails (ai_outputParser)

9. **Add filtering Code node**
   - Node: **Code** (JavaScript)
   - Name: `Code in JavaScript`
   - Implement logic to:
     - read `item.json.output.company`, `item.json.output.domain`, `item.json.output.owner_emails`
     - filter out unwanted local parts
     - dedupe per domain
     - output `{company, domain, email}`
   - **Connect:** AI Extract Owner Emails → Code in JavaScript

10. **Add ValidKit verification request**
   - Node: **HTTP Request**
   - Name: `Verify Email`
   - Method: **POST**
   - URL: `https://api.validkit.com/api/v1/verify`
   - Header: `X-API-Key = <YOUR_VALIDKIT_KEY>`
   - Body parameter: `email = {{$json.email}}`
   - **Connect:** Code in JavaScript → Verify Email

11. **Add Set node to format output**
   - Node: **Set**
   - Name: `Format ValidKit Output`
   - Set fields:
     - `company`, `domain`, `email` (from Code node item via expression)
     - `status`, `syntax_valid`, `mx_found`, `smtp_check`, `disposable` (from ValidKit response)
   - **Connect:** Verify Email → Format ValidKit Output

12. **Configure Google Sheets credentials**
   - Create/select **Google Sheets OAuth2** credential with access to your target spreadsheet.

13. **Add Google Sheets node**
   - Node: **Google Sheets**
   - Name: `Log Verified Emails`
   - Operation: **Append or Update**
   - Select Document (spreadsheet) and Sheet/tab
   - Ensure columns exist: `company, domain, email, status, syntax_valid, mx_found, smtp_check, disposable`
   - Mapping: Auto-map input data (or map explicitly)
   - **Connect:** Format ValidKit Output → Log Verified Emails

14. (Optional) **Add Sticky Notes** to annotate blocks as in the original workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Main**: “This workflow finds and verifies email addresses from Google search results using AI. It processes organic search results, extracts company domains, generates likely email patterns, validates them, and saves verified emails to Google Sheets.” | Overall workflow intent (Sticky Note “Main”) |
| **Setup**: “1. Add your Apify API key 2. Add your OpenAI API key 3. Add your email verification API key 4. Connect Google Sheets 5. Update the search query 6. Run the workflow” | Required configuration steps (Sticky Note “Main”) |
| Disclaimer (provided): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | User-provided compliance/disclaimer context |

