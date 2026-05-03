Scrape Apollo leads with BrowserAct, draft Gemini emails, and save to Sheets

https://n8nworkflows.xyz/workflows/scrape-apollo-leads-with-browseract--draft-gemini-emails--and-save-to-sheets-13170


# Scrape Apollo leads with BrowserAct, draft Gemini emails, and save to Sheets

## 1. Workflow Overview

**Title:** Scrape Apollo leads with BrowserAct, draft Gemini emails, and save to Sheets  
**Workflow name (in JSON):** Apollo Business Hunter

**Purpose:**  
A fully automated B2B lead collection + outreach drafting pipeline. It collects search criteria via an n8n Form, runs a BrowserAct real-browser scrape on Apollo.io to extract lead details, uses Google Gemini (via an n8n LangChain LLM Chain) to generate a short personalized cold email for each lead, appends results to Google Sheets, then sends a Gmail notification summary when finished.

**Target use cases:**
- Building a prospect list for a specific **role/category** and **location**
- Generating **first-touch cold email drafts** at scale
- Centralizing scraped/enriched lead data into **Google Sheets** for review and outreach

### 1.1 Input Reception (Form)
Collects criteria (Location + Business Category) from a user submission.

### 1.2 Browser Scraping (BrowserAct → Apollo.io)
Triggers a BrowserAct workflow (remote browser automation) to search Apollo.io and return contact data.

### 1.3 Data Normalization (Code)
Parses the BrowserAct output (string or object), flattens it into one n8n item per lead, and standardizes fields.

### 1.4 AI Email Drafting (Gemini via LLM Chain)
For each lead, Gemini generates a short personalized cold email draft.

### 1.5 Merge + Storage (Merge → Set → Google Sheets)
Merges original lead data with the AI-generated text, maps columns to the sheet schema, and appends rows to Google Sheets.

### 1.6 Notification (Gmail)
Sends a final summary email listing the number of processed leads and their names/companies.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Form)

**Overview:**  
Captures user-defined lead criteria and starts the workflow execution.

**Nodes involved:**
- Receive Lead Criteria (Form Trigger)

#### Node: Receive Lead Criteria
- **Type / role:** `Form Trigger` — workflow entry point; provides a hosted form and triggers execution on submission.
- **Key configuration:**
  - **Form title:** “Lead Gen Request”
  - **Description:** “Fill this to tailor your request”
  - **Fields (required):**
    - `Location` (placeholder “Austin ”)
    - `Bussines_Category` (placeholder “accountant”) *(note spelling: Bussines)*
- **Key variables produced:** `$json.Location`, `$json.Bussines_Category`
- **Connections:**
  - **Out:** to “Scrape Apollo via BrowserAct”
- **Edge cases / failures:**
  - Users may enter inconsistent location formats (e.g., “NYC” vs “New York”).
  - Field name typo (`Bussines_Category`) must be used consistently downstream; changing it requires updating expressions in BrowserAct mapping.
- **Version notes:** Node typeVersion `2.2` (behavior consistent with form-based triggers; field naming is critical).

---

### Block 2 — Browser Scraping (BrowserAct → Apollo.io)

**Overview:**  
Calls a BrowserAct hosted workflow that performs a real-browser scrape on Apollo.io using the submitted criteria.

**Nodes involved:**
- Scrape Apollo via BrowserAct

#### Node: Scrape Apollo via BrowserAct
- **Type / role:** `browserAct` — executes a BrowserAct Workflow and returns its output.
- **Key configuration choices:**
  - **Mode:** WORKFLOW
  - **Workflow ID:** `70144352397874162` (must match your BrowserAct side)
  - **Workflow input mapping:**
    - `input-Location` = `{{ $json.Location }}`
    - `input-Bussines_Category` = `{{ $json.Bussines_Category }}`
  - Two other possible BrowserAct inputs exist in schema but are **removed** in n8n mapping UI:
    - `input-ApoloAI_Search` (removed)
    - `input-ApoloAi_Login` (removed)
- **Credentials:** BrowserAct API credential (“BrowserAct account”)
- **Connections:**
  - **In:** from “Receive Lead Criteria”
  - **Out:** to “Clean & Flatten JSON”
- **Edge cases / failures:**
  - BrowserAct workflow may fail due to:
    - Apollo UI changes / selectors breaking
    - Login/session expiry (especially if BrowserAct workflow requires authentication)
    - Rate limits / anti-bot protections
    - Timeouts on large result sets
  - Output format variability: output might be JSON string, object, or unexpected structure—handled partially by the next Code node.
- **Version notes:** typeVersion `1` (BrowserAct node; ensure node package `n8n-nodes-browseract` is installed and compatible).

---

### Block 3 — Data Normalization (Parse, Clean, Flatten)

**Overview:**  
Converts BrowserAct output into a clean list of lead items with consistent keys used by downstream AI and storage nodes.

**Nodes involved:**
- Clean & Flatten JSON (Code)

#### Node: Clean & Flatten JSON
- **Type / role:** `Code` — JavaScript transformation.
- **Key logic (interpreted):**
  - Reads BrowserAct output from `$json.output.string` **or** `$json.output`.
  - Tries to `JSON.parse()` if it’s a string; if parsing fails, returns an **empty list**.
  - Normalizes output into `blocks[]`:
    - If parsed is an array → use as-is
    - If parsed is object → wrap into array
    - Else → empty
  - Expects each block to have `block.item` as an array of people.
  - For each person, builds:
    - `id`: `email` OR `links` OR `${name}-${company}` (fallback)
    - `name`, `job_title`, `email`, `profile_url` (from `links`), `company`, `location`
  - Returns one n8n item per lead.
- **Key fields output:**  
  `id, name, job_title, email, profile_url, company, location`
- **Connections:**
  - **Out 1:** to “Basic LLM Chain” (for AI drafting)
  - **Out 2:** to “Merge” input index 0 (to preserve original lead info for merging)
- **Edge cases / failures:**
  - If BrowserAct output schema changes (e.g., `item` renamed), the result becomes empty.
  - If `links` is not a string (array/object), `profile_url` may become `[object Object]` or empty.
  - Duplicate IDs: `id` is not used later for Sheets matching (Sheets node generates a new id), but duplicates may still affect reporting/expectations.
- **Version notes:** Code node typeVersion `2` (modern n8n Code node semantics: must `return [{json: ...}]`).

---

### Block 4 — AI Email Drafting (Gemini via LLM Chain)

**Overview:**  
Generates a personalized cold email per lead using Google Gemini as the chat model.

**Nodes involved:**
- Google Gemini Chat Model
- Basic LLM Chain

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini` — provides a Gemini chat model to LangChain nodes.
- **Key configuration:** default options (no explicit model parameters shown).
- **Credentials:** Google Gemini(PaLM) API credential
- **Connections:**
  - **Out (AI languageModel):** to “Basic LLM Chain” language model input.
- **Edge cases / failures:**
  - API key invalid / project not enabled / billing issues.
  - Safety filters or policy blocks may return empty/blocked content.
  - Quota/rate limits when processing many leads.
- **Version notes:** typeVersion `1` (verify your n8n LangChain integration package supports this node).

#### Node: Basic LLM Chain
- **Type / role:** `chainLlm` — runs an LLM prompt per incoming item.
- **Key configuration choices:**
  - **Prompt (define mode):**  
    Uses lead fields to generate an email:
    - `{{ $json.name }}`
    - `{{ $json.job_title }}`
    - `{{ $json.company }}`
  - Constraints: “Keep it under 100 words. Be polite and professional.”
  - **Batching:** present but empty (defaults apply; typically means it processes each item).
- **Expected output:**  
  Produces an item containing generated text (commonly `text` in LangChain nodes). This workflow later references `$json.text`.
- **Connections:**
  - **In:** from “Clean & Flatten JSON”
  - **AI Model input:** from “Google Gemini Chat Model”
  - **Out:** to “Merge” input index 1
- **Edge cases / failures:**
  - If upstream fields are missing (name/job/company empty), the email becomes generic.
  - If the node outputs a different field than `text` (depends on node version/settings), downstream mapping to `$json.text` will fail (email_draft becomes blank).
- **Version notes:** typeVersion `1.7` (output schema can vary by version; confirm generated text field name).

---

### Block 5 — Merge + Column Mapping + Google Sheets Storage

**Overview:**  
Combines the original lead data with the AI-generated email content, then appends each combined row to Google Sheets with consistent column names.

**Nodes involved:**
- Merge
- Map Columns
- Save to Google Sheets

#### Node: Merge
- **Type / role:** `Merge` — combines two streams into one item set.
- **Key configuration:**
  - **Mode:** Combine
  - **Combine by:** Position (combineByPosition)
  - This means item 1 from input 0 merges with item 1 from input 1, etc.
- **Connections:**
  - **Input 0:** from “Clean & Flatten JSON”
  - **Input 1:** from “Basic LLM Chain”
  - **Out:** to “Map Columns”
- **Edge cases / failures:**
  - If LLM chain outputs fewer items (error on one lead), positional merge misaligns lead data and email drafts.
  - If execution order changes or one branch filters items, positional combine becomes unsafe. Consider combining by a stable key if available.
- **Version notes:** typeVersion `3.2`

#### Node: Map Columns
- **Type / role:** `Set` — normalizes final field names for storage.
- **Key configuration choices:**
  - Assigns:
    - `name, job_title, email, profile_url, company, location` from `$json.<field>`
    - `email_draft` from `{{ $json.text }}`
  - Effectively ensures the downstream Sheets node can reference `$json.email_draft`.
- **Connections:**
  - **In:** from “Merge”
  - **Out:** to “Save to Google Sheets”
- **Edge cases / failures:**
  - If `text` isn’t present (LLM output schema differs), `email_draft` becomes empty.
  - If Merge produces nested objects (name collisions), you may lose fields if not carefully merged (depends on merge behavior).
- **Version notes:** typeVersion `3.4`

#### Node: Save to Google Sheets
- **Type / role:** `Google Sheets` — appends one row per item.
- **Key configuration choices:**
  - **Authentication:** Service Account
  - **Operation:** Append
  - **Document:** spreadsheet “n8n” (ID `1znh4t8qgqvRVe1ZGCteM5F0mapxCyjEHegsK2K5yF24`)
  - **Sheet tab:** `sheet002` (gid `971059340`)
  - **Columns mapped (define below):**
    - `id`: generated unique-ish value: `{{ $now.format('x') }}-{{ Math.random().toString(36).substr(2, 9) }}`
    - `name, email, company, location, job_title, email_draft, profile_url`: from `$json`
  - Matching columns list includes `id`, but since operation is append, matching is not used for upsert (still must exist as a header).
- **Connections:**
  - **In:** from “Map Columns”
  - **Out:** to “Send a message”
- **Edge cases / failures:**
  - Sheet header mismatch (missing/renamed headers) will cause mapping errors or blank columns.
  - Service account must have access to the spreadsheet (shared with the service account email).
  - API limits for large batches.
- **Version notes:** typeVersion `4.7` (Google Sheets node has multiple auth modes; service account requires correct JSON key setup in n8n credentials).

---

### Block 6 — Notification (Gmail)

**Overview:**  
After all rows are appended, sends a single summary email listing the leads processed and a link to the spreadsheet.

**Nodes involved:**
- Send a message

#### Node: Send a message
- **Type / role:** `Gmail` — sends an email via Gmail OAuth2.
- **Key configuration choices:**
  - **To:** `user@example.com` (replace with your email)
  - **Subject:** `[n8n Notification] Leads Successfully Collected & Outreach Emails Generated!`
  - **Body:** Uses expressions:
    - `{{ $input.all().length }}` to count items reaching this node
    - Lists leads: `{{ $input.all().map(item => item.json.name + " (" + item.json.company + ")").join('\n') }}`
    - Includes a fixed Google Sheets URL to the configured doc/tab
  - **Execute Once:** enabled → ensures only one email is sent (not one per item)
- **Connections:**
  - **In:** from “Save to Google Sheets”
  - **Out:** none (end)
- **Edge cases / failures:**
  - If the node receives zero items, it still sends a “0 leads” email (may be desired or not).
  - Gmail OAuth token expiry/revocation.
  - Message content uses a hardcoded sheet link—if you change sheet, update link.
- **Version notes:** typeVersion `2.2`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Lead Criteria | Form Trigger | Collect lead criteria and start workflow | — | Scrape Apollo via BrowserAct | Title: Data collection. Text: Accepts target role and location inputs via a form, then triggers a BrowserAct workflow to perform a real-browser search and scrape contact data from Apollo.io. |
| Scrape Apollo via BrowserAct | BrowserAct | Run remote browser scraping workflow on Apollo | Receive Lead Criteria | Clean & Flatten JSON | Title: Data collection. Text: Accepts target role and location inputs via a form, then triggers a BrowserAct workflow to perform a real-browser search and scrape contact data from Apollo.io. |
| Clean & Flatten JSON | Code | Parse/standardize BrowserAct output into lead items | Scrape Apollo via BrowserAct | Basic LLM Chain; Merge | **Title:** AI Enrichment & Processing. **Text:** 1. **Clean:** Standardizes the raw scraped JSON data. 2. **AI Write:** Uses **Google Gemini** to analyze each lead and generate a personalized cold email draft automatically. 3. **Merge:** Combines the original contact info with the AI-generated email content into a single dataset. |
| Google Gemini Chat Model | LangChain Chat Model (Google Gemini) | Provide Gemini model to LLM Chain | — | Basic LLM Chain (ai_languageModel) | **Title:** AI Enrichment & Processing. **Text:** 1. **Clean:** Standardizes the raw scraped JSON data. 2. **AI Write:** Uses **Google Gemini** to analyze each lead and generate a personalized cold email draft automatically. 3. **Merge:** Combines the original contact info with the AI-generated email content into a single dataset. |
| Basic LLM Chain | LangChain LLM Chain | Generate personalized cold email draft per lead | Clean & Flatten JSON; Google Gemini Chat Model | Merge | **Title:** AI Enrichment & Processing. **Text:** 1. **Clean:** Standardizes the raw scraped JSON data. 2. **AI Write:** Uses **Google Gemini** to analyze each lead and generate a personalized cold email draft automatically. 3. **Merge:** Combines the original contact info with the AI-generated email content into a single dataset. |
| Merge | Merge | Combine lead info with AI output (positional) | Clean & Flatten JSON; Basic LLM Chain | Map Columns | **Title:** AI Enrichment & Processing. **Text:** 1. **Clean:** Standardizes the raw scraped JSON data. 2. **AI Write:** Uses **Google Gemini** to analyze each lead and generate a personalized cold email draft automatically. 3. **Merge:** Combines the original contact info with the AI-generated email content into a single dataset. |
| Map Columns | Set | Normalize field names for sheet storage | Merge | Save to Google Sheets | **Title:** AI Enrichment & Processing. **Text:** 1. **Clean:** Standardizes the raw scraped JSON data. 2. **AI Write:** Uses **Google Gemini** to analyze each lead and generate a personalized cold email draft automatically. 3. **Merge:** Combines the original contact info with the AI-generated email content into a single dataset. |
| Save to Google Sheets | Google Sheets | Append enriched leads + email drafts to Sheets | Map Columns | Send a message | **Title:** Storage & Notification. **Text:** 1. **Save:** Appends the verified lead profile + AI email draft to **Google Sheets**. 2. **Notify:** Sends a summary report to your **Gmail** once the batch is finished, listing how many leads were captured. |
| Send a message | Gmail | Send completion summary email | Save to Google Sheets | — | **Title:** Storage & Notification. **Text:** 1. **Save:** Appends the verified lead profile + AI email draft to **Google Sheets**. 2. **Notify:** Sends a summary report to your **Gmail** once the batch is finished, listing how many leads were captured. |
| Sticky Note | Sticky Note | Documentation / setup guidance | — | — | This workflow creates a fully automated B2B lead generation & outreach pipeline. It combines **BrowserAct** (for scraping) with **Google Gemini AI** (for writing) to automate the entire prospecting process. \| Set up steps include BrowserAct Workflow ID, Google Sheets headers: `id`, `name`, `email`, `job_title`, `profile_url`, `company`, `location`, `email_draft`, and credentials for BrowserAct, Gemini(PaLM), Sheets, Gmail. |
| Sticky Note1 | Sticky Note | Documentation: Data collection block | — | — | Title: Data collection. Text: Accepts target role and location inputs via a form, then triggers a BrowserAct workflow to perform a real-browser search and scrape contact data from Apollo.io. |
| Sticky Note2 | Sticky Note | Documentation: AI enrichment & processing | — | — | **Title:** AI Enrichment & Processing. **Text:** 1. **Clean:** Standardizes the raw scraped JSON data. 2. **AI Write:** Uses **Google Gemini** to analyze each lead and generate a personalized cold email draft automatically. 3. **Merge:** Combines the original contact info with the AI-generated email content into a single dataset. |
| Sticky Note3 | Sticky Note | Documentation: Storage & notification | — | — | **Title:** Storage & Notification. **Text:** 1. **Save:** Appends the verified lead profile + AI email draft to **Google Sheets**. 2. **Notify:** Sends a summary report to your **Gmail** once the batch is finished, listing how many leads were captured. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it (e.g.) **“Apollo Business Hunter”**
- Ensure you have installed/available nodes:
  - **BrowserAct node package** (`n8n-nodes-browseract`)
  - **LangChain nodes** in n8n (for LLM Chain + Gemini model)

2) **Add the entry form**
- Add node: **Form Trigger**
- Configure:
  - Title: `Lead Gen Request`
  - Description: `Fill this to tailor your request`
  - Fields (both required):
    - Field Label: `Location`
    - Field Label: `Bussines_Category` (keep this exact spelling unless you update mappings)
- This node is the start.

3) **Add BrowserAct workflow execution**
- Add node: **BrowserAct** (execute workflow)
- Set:
  - Type: `WORKFLOW`
  - Workflow ID: *(paste your BrowserAct Workflow ID; in the provided workflow it is `70144352397874162`)*
  - Workflow Config inputs:
    - `input-Location` → expression `{{ $json.Location }}`
    - `input-Bussines_Category` → expression `{{ $json.Bussines_Category }}`
- Credentials:
  - Create/select **BrowserAct API** credential.
- Connect: **Form Trigger → BrowserAct**

4) **Add “Clean & Flatten JSON” transformation**
- Add node: **Code**
- Paste logic equivalent to:
  - Read from `$json.output.string` or `$json.output`
  - Parse JSON if needed
  - For each person, output one item with:
    - `id, name, job_title, email, profile_url, company, location`
- Connect: **BrowserAct → Code**

5) **Add Gemini model provider**
- Add node: **Google Gemini Chat Model** (LangChain)
- Credentials:
  - Create/select **Google Gemini (PaLM) API** credential (API key / project setup per n8n credential type)
- No main connection needed; it connects via the **ai_languageModel** port.

6) **Add LLM Chain for email drafting**
- Add node: **Basic LLM Chain**
- Set prompt (define prompt text) similar to:
  - “You are a professional B2B sales expert… Write a short, engaging cold email to {{name}}…”
  - Keep under 100 words.
- Connect:
  - **Code (main) → LLM Chain (main)**
  - **Gemini Chat Model (ai_languageModel) → LLM Chain (ai_languageModel)**

7) **Merge original lead data + LLM output**
- Add node: **Merge**
- Configure:
  - Mode: `Combine`
  - Combine by: `Position`
- Connect:
  - **Code → Merge (Input 0)**
  - **LLM Chain → Merge (Input 1)**

8) **Map fields to final schema**
- Add node: **Set** (rename to “Map Columns”)
- Add/set fields:
  - `name` = `{{ $json.name }}`
  - `job_title` = `{{ $json.job_title }}`
  - `email` = `{{ $json.email }}`
  - `profile_url` = `{{ $json.profile_url }}`
  - `company` = `{{ $json.company }}`
  - `location` = `{{ $json.location }}`
  - `email_draft` = `{{ $json.text }}` *(adjust if your LLM output field differs)*
- Connect: **Merge → Set**

9) **Prepare Google Sheet**
- Create a Google Sheet and add headers in row 1 exactly:
  - `id`, `name`, `email`, `job_title`, `profile_url`, `company`, `location`, `email_draft`
- Share the sheet with your **Google Service Account email** (if using service account auth).

10) **Append to Google Sheets**
- Add node: **Google Sheets**
- Configure:
  - Operation: `Append`
  - Authentication: `Service Account` (or OAuth2 if you prefer, but then match credentials)
  - Select the **Document** and **Sheet tab**
  - Map columns:
    - `id` = `{{ $now.format('x') }}-{{ Math.random().toString(36).substr(2, 9) }}`
    - `name/email/company/location/job_title/email_draft/profile_url` from the Set node fields
- Connect: **Set → Google Sheets**
- Credentials:
  - Create/select **Google API** credential (Service Account JSON uploaded in n8n).

11) **Send completion email via Gmail**
- Add node: **Gmail** (Send a message)
- Configure:
  - To: your email address
  - Subject: notification subject
  - Message body:
    - Count items: `{{ $input.all().length }}`
    - List leads: `{{ $input.all().map(item => item.json.name + " (" + item.json.company + ")").join('\n') }}`
    - Include your sheet link
  - Enable **Execute Once** (so it sends one summary email per run)
- Connect: **Google Sheets → Gmail**
- Credentials:
  - Create/select **Gmail OAuth2** credential.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow creates a fully automated B2B lead generation & outreach pipeline combining BrowserAct scraping + Google Gemini email drafting + Google Sheets storage + Gmail notification. | Sticky note (overview) |
| BrowserAct setup: open “Scrape Apollo via BrowserAct” and paste your BrowserAct Workflow ID. | Sticky note (setup steps) |
| Google Sheets must include headers: `id`, `name`, `email`, `job_title`, `profile_url`, `company`, `location`, `email_draft`. | Sticky note (setup steps) |
| Required credentials: BrowserAct, Google Gemini(PaLM) API, Google Sheets (Service Account or OAuth2), Gmail OAuth2. | Sticky note (setup steps) |
| Spreadsheet used in this workflow: https://docs.google.com/spreadsheets/d/1znh4t8qgqvRVe1ZGCteM5F0mapxCyjEHegsK2K5yF24/edit?gid=971059340#gid=971059340 | Used in Gmail message + Sheets node cached URL |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.