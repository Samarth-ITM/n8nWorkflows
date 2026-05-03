Qualify CSV leads, enrich emails, and deliver results with Google Sheets, Drive, WhatsApp, and GPT-5-NANO

https://n8nworkflows.xyz/workflows/qualify-csv-leads--enrich-emails--and-deliver-results-with-google-sheets--drive--whatsapp--and-gpt-5-nano-11382


# Qualify CSV leads, enrich emails, and deliver results with Google Sheets, Drive, WhatsApp, and GPT-5-NANO

## 1. Workflow Overview

**Workflow title:** *Qualify CSV leads, enrich emails, and deliver results with Google Sheets, Drive, WhatsApp, and GPT-5-NANO*  
**Purpose:** Ingest a CSV/XLSX of leads, normalize column names into a standard schema, deduplicate, validate/scrape websites to discover emails, verify emails via Reoon, generate AI personalization (GPT‑5‑NANO or Gemini), then deliver results via **Google Drive**, **WhatsApp**, or **Google Sheets (categorized)**.

### 1.1 Input Reception (Form Upload)
Users upload a file and provide two prompts used later for AI personalization, plus choose the delivery mode.

### 1.2 File Parsing & Normalization
The workflow converts uploaded binary into per-file items, parses CSV/XLSX rows, standardizes many column variants into a fixed set of canonical fields, and deduplicates by Website and Name.

### 1.3 Delivery Mode Routing
A switch routes the flow to:
- **Google Drive**: export file → upload → email link
- **WhatsApp**: export XLSX → send as WhatsApp document
- **Google Sheets with Data Processing**: run enrichment + categorization + personalization, then write to Sheets and notify.

### 1.4 Data Processing Routing (Website/Email Presence)
Records are flagged for “wrong website” patterns (social/link shorteners) and routed into:
- **x Website** bucket (bad/undesired domains)
- **No Website** bucket (Website is “-”)
- **Website + Email** bucket (validate existing emails; scrape website metadata)
- **Website but no Email** bucket (scrape website to find emails, pick best with GPT, verify)

### 1.5 Website Scraping & Metadata Extraction
HTTP fetch + HTML extraction (Title/meta description/H1) with fallback nodes on errors.

### 1.6 Email Discovery & Verification
- Extract emails from website HTML using regex.
- Use **GPT‑5‑NANO** to choose one email if multiple found.
- Verify via **Reoon Email Verifier API** (deliverability/SMTP/safety/MX checks).
- Personal Email is verified separately when present.

### 1.7 Filtering, Personalization, and Sorting
Only items with some valid email signal pass forward. AI chain generates:
- New Name
- New Company (short)
- Personalization 1 & 2 (from user prompts)

Then data is routed into **+1 Email** vs **+2 Email** Google Sheets outputs (based on combinations of valid business/personal/new email), with normalization of empty fields to “-”, and a completion email.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Form Intake & Binary Item Preparation
**Overview:** Receives the uploaded CSV/XLSX and user settings, then reshapes binary payload into one item per uploaded file.
**Nodes involved:**
- **On form submission**
- **Code**
- **CSV Loop**

#### Node: On form submission
- **Type/role:** `n8n-nodes-base.formTrigger` — entry point; collects file + options + prompts.
- **Key configuration:**
  - Path: `lead-qualifier`
  - Fields:
    - File: accepts `.csv, .xlsx` (required)
    - Dropdown “Save Into?”: Google Drive / WhatsApp / Google Sheets with Data Processing (required)
    - Textareas: Prompt 1 & Prompt 2 (required)
- **Output:** One item containing `json` (form fields) + `binary` file(s).
- **Edge cases:**
  - Multiple files uploaded → binary has multiple properties (handled by next Code node).
  - XLSX may parse differently downstream depending on Extract From File settings.

#### Node: Code
- **Type/role:** `n8n-nodes-base.code` — splits multi-binary input into separate items.
- **Logic:** Iterates over `$input.last().binary` keys and produces one item per binary file, preserving the same form `json`.
- **Inputs/outputs:** Input from trigger → output to **CSV Loop**.
- **Edge cases:** If no binary keys exist, output becomes empty; ensure file field is required (it is).

#### Node: CSV Loop
- **Type/role:** `n8n-nodes-base.splitInBatches` — begins processing stream in batches (batch size not set → defaults to n8n behavior).
- **Connections:** Sends data to **Remove Duplicate Websites** and **Extract from File** in parallel.
- **Edge cases:** With default batch size, large datasets may require explicit batch sizing to prevent memory pressure.

---

### Block 2.2 — File Extraction, Column Standardization, Deduplication
**Overview:** Parses the file into rows, converts arbitrary column names into standardized fields, then removes duplicates by Website and Name.
**Nodes involved:**
- **Extract from File**
- **Replace Value**
- **Remove Duplicate Websites**
- **Remove Duplicate Name**
- **Sticky Note (Filter & Formitize Data)**

#### Node: Extract from File
- **Type/role:** `n8n-nodes-base.extractFromFile` — reads binary and outputs row items.
- **Key configuration:**
  - Binary property: `data`
  - BOM handling enabled
- **Outputs:** Item per row (`json` = row columns).
- **Edge cases:** XLSX parsing depends on n8n’s Extract From File support; ensure correct sheet/tab selection if needed (not configured here).

#### Node: Replace Value
- **Type/role:** `n8n-nodes-base.code` — standardizes columns into canonical schema.
- **Key configuration (interpreted):**
  - Maintains a large `columnMappings` dictionary (canonical field → list of variants).
  - Creates a case-insensitive lookup map.
  - Initializes all canonical fields to `"-"` for each row.
  - Copies values from any mapped variant if non-empty.
  - Drops all unmapped columns (only canonical fields are kept).
- **Canonical fields include (examples):**
  - Name, First Name, Last Name, Email, Personal Email, Phone Number, LinkedIn, Job Title, Website, Company, City/State/Country, Industry, Keywords, etc.
- **Failure modes:**
  - Malformed input rows can cause unexpected types; code converts values to strings safely.
  - Throws a new error on failure, stopping the workflow at this point.

#### Node: Remove Duplicate Websites
- **Type/role:** `n8n-nodes-base.removeDuplicates` — dedupe by `Website`.
- **Edge cases:** Many items may have Website = `"-"`; those will dedupe into one record unless Website routing happens before dedupe (it does not). This can unintentionally drop many “no website” rows.

#### Node: Remove Duplicate Name
- **Type/role:** `n8n-nodes-base.removeDuplicates` — dedupe by `Name`.
- **Edge cases:** Name = `"-"` will collapse many entries.

#### Sticky Note: “## Filter & Formitize Data”
- Applies conceptually to: Extract/standardize/dedupe section.

---

### Block 2.3 — Delivery Mode Switch (Drive / WhatsApp / Sheets Processing)
**Overview:** Based on the form dropdown, chooses output channel.
**Nodes involved:**
- **Send to Switch**
- **Convert to File** (Drive path)
- **Upload file**
- **Send a message**
- **Convert to File.** (WhatsApp path)
- **Send CSV**
- **Detect Wrong Website URLs** (Sheets processing path)
- **Sticky Note (Save Filtered Data into Google Drive)**
- **Sticky Note (Send Filtered Data to WhatsApp)**

#### Node: Send to Switch
- **Type/role:** `n8n-nodes-base.switch` — routes by `$('On form submission').item.json['Save Into?']`.
- **Outputs:**
  - “Google Drive” → **Convert to File**
  - “WhatsApp” → **Convert to File.**
  - “Google Sheets with Data Processing” → **Detect Wrong Website URLs**
- **Edge cases:** Exact string matching; dropdown values must match rules.

#### Node: Convert to File (Drive)
- **Type/role:** `n8n-nodes-base.convertToFile` — converts JSON items into a file for Drive upload.
- **Operation:** default (not explicitly set; typically CSV unless configured).  
- **Output:** binary file.

#### Node: Upload file
- **Type/role:** `n8n-nodes-base.googleDrive` — uploads file to a specific folder.
- **Key configuration:**
  - File name: current datetime `dd-MM-yyyy | HH:mm`
  - Drive: “My Drive”
  - Folder: “Good CSV” (ID `1DlS21EpQWfBhQVdTvAePkGFgCXdFa2Fe`)
- **Output:** includes `webViewLink` used in the Gmail message.
- **Failure modes:** OAuth/service account permission issues; folder ID invalid; rate limits.

#### Node: Send a message
- **Type/role:** `n8n-nodes-base.gmail` — notifies user with Drive link.
- **Key configuration:**
  - To: `user@example.com` (placeholder)
  - Subject: “⚡LQ:  Your CSV is Ready”
  - Body includes `{{ $json.webViewLink }}`
- **Edge cases:** Needs valid Gmail credential; sendTo should be dynamic (currently static placeholder).

#### Node: Convert to File. (WhatsApp)
- **Type/role:** `n8n-nodes-base.convertToFile` — creates an XLSX (`operation: xlsx`) for WhatsApp.
- **Output:** binary XLSX.

#### Node: Send CSV
- **Type/role:** `n8n-nodes-base.whatsApp` — sends document via WhatsApp Business.
- **Key configuration:**
  - Operation: send document
  - Phone Number ID: `764045186781201`
  - Recipient: `+1234567890` (placeholder)
  - Filename: timestamp `dd LLL yyyy | HH:mm`
- **Failure modes:** WhatsApp token/auth, media upload size limits, recipient formatting.

#### Node: Detect Wrong Website URLs
- **Type/role:** `n8n-nodes-base.set` — flags undesired website domains.
- **Key expression:**
  - `is_wrong_website = $json.Website !== "" && ($json.Website.extractDomain().toLowerCase()).match(/ikea|instagram|facebook|linktr.ee|wa.me/) !== null`
- **Edge cases:**
  - `extractDomain()` fails if Website is not a URL; ignoreConversionErrors is enabled.
  - Logic checks Website != `""` but upstream uses `"-"` for missing; `"-".extractDomain()` may fail but is ignored, producing falsey behavior.

---

### Block 2.4 — Data Processing Switch (Website/Email Scenarios)
**Overview:** Routes each lead record to the correct enrichment path depending on Website presence/validity and Email presence.
**Nodes involved:**
- **Data Processing Switch**
- **x Website Data**
- **No Website Loop** → No Website path
- **Loop** → Website+Email path
- **Web Scrap Loop** → Website-no-email path
- **Sticky Note: If Website & Email both exist**
- **Sticky Note: Website not exist but Email exist**
- **Sticky Note: Website & Email not exist**

#### Node: Data Processing Switch
- **Type/role:** `n8n-nodes-base.switch` with **allMatchingOutputs** enabled.
- **Rules (interpreted):**
  1. **x Website** if `is_wrong_website` is true
  2. **- Website +- Email +- Phone Number** if `Website == "-"` (no website)
  3. **+ Website + Email** if website is not wrong AND Website != "-" AND (Email != "-" OR Personal Email != "-")  
     *Note:* expression precedence (`&&` vs `||`) makes this effectively:  
     `(!$is_wrong && Website != "-" && Email != "-") OR (Personal Email != "-")` which may route items with Personal Email even if Website is wrong/missing.
  4. **+ Website - Email** if website is not wrong AND Email == "-" AND Personal Email == "-" AND Website != "-"
- **Outputs:**
  - x Website → **x Website Data**
  - no website → **No Website Loop**
  - website+email → **Loop**
  - website-no-email → **Web Scrap Loop**
- **Edge cases:** Rule #3 likely needs parentheses to avoid misrouting.

#### Node: x Website Data
- **Type/role:** `n8n-nodes-base.googleSheets` append — writes “bad website” records to the “x Website” spreadsheet.
- **Document:** `1t447EMkE_2TxZIL-Wr97e2LNS-oGcBpPxWYfnZRziWE`
- **Failure modes:** Google Sheets auth, missing columns, rate limits.

#### Node: No Website Loop
- **Type/role:** `n8n-nodes-base.splitInBatches` — processes records without Website.
- **Connections:** Second output (loop continuation) connects to **Email exist or not** and **No Website Merge** (unusual wiring; see edge case note below).
- **Edge cases:** The first output is unused; suggests configuration mismatch.

---

### Block 2.5 — No Website Path (Email Verification Only)
**Overview:** For records without a website, verify existing Email if present and mark it valid/invalid; then write to “- Website +- Email +- Phone Number”.
**Nodes involved:**
- **Email exist or not**
- **Verify.**
- **Right or Wrong.**
- **Set Valid Email**
- **Set invalid Email.**
- **Set invalid Email**
- **No Website Merge**
- **- Website +- Email +- Phone Number**
- **Sticky Note: Website not exist but Email exist**

#### Node: Email exist or not
- **Type/role:** `n8n-nodes-base.if` — checks `Email != "-"`.
- **True:** go to **Verify.**
- **False:** go to **Set invalid Email** (marks invalid)

#### Node: Verify.
- **Type/role:** `n8n-nodes-base.httpRequest` — Reoon verification.
- **URL template:** `https://emailverifier.reoon.com/api/v1/verify?email={{ $json['Email'].trim() }}&key=<key>&mode=power`
- **Failure modes:** invalid API key, rate limits, network errors.

#### Node: Right or Wrong.
- **Type/role:** `n8n-nodes-base.if` — checks Reoon booleans:
  - `is_deliverable`, `can_connect_smtp`, `is_safe_to_send`, `mx_accepts_mail`
- **True:** **Set Valid Email** (`is_valid_mail = true`)
- **False:** **Set invalid Email.** (`is_valid_mail = false`)

#### Node: No Website Merge
- **Type/role:** `n8n-nodes-base.merge` combineAll — merges “verified result” + original record flow.
- **Output:** **- Website +- Email +- Phone Number** (Google Sheet append)

#### Node: - Website +- Email +- Phone Number
- **Type/role:** Google Sheets append — stores no-website records (with validity flag).
- **Document:** `1vmtaV8pDzPBKht68ApC3KeLjqX2IgIBemYhlh3dIdfQ`
- **Edge cases:** Column schema includes `is_valid_mail` but is typed as string in schema; upstream sets boolean.

---

### Block 2.6 — Website + Email Path (Verify Email + Scrape Metadata)
**Overview:** For records that already have an email, verify it and scrape website title/meta/H1 for context; merge results back into the main stream.
**Nodes involved:**
- **Loop**
- **E: exist or not**
- **E: Verify**
- **E: Verify Backup**
- **E: Right or Wrong**
- **E: Scrap Website HTML**
- **E: Extract Tag & Attribute**
- **E: Website Scrapping Error**
- **E: Email not exist**
- **E: Seng Process Data**
- **E: Right or Wrong (IF)**
- **E: Merge**
- **Sticky Note: If Website & Email both exist**

#### Node: Loop
- **Type/role:** Split in batches — iterates through leads for enrichment.
- **Connections:**
  - Output 0 → **Processed Data Merge**
  - Output 1 → fan-out to **E: Merge**, **E: exist or not**, **PR: Exist or not**
- **Edge cases:** The same item is used for multiple parallel actions; combineAll merges can misalign item pairing if counts differ.

#### Node: E: exist or not
- **Type/role:** IF — checks Email is not `"-"`.
- **True:** **E: Verify**
- **False:** **E: Email not exist** (sets empty metadata and `is_valid_mail=false`)

#### Node: E: Verify / E: Verify Backup
- **Type/role:** HTTP requests to Reoon.
- **Behavior:** Primary has `onError: continueErrorOutput` and also routes to Backup in parallel; Backup then routes back into **E: Right or Wrong**.
- **Edge cases:** Double-calling Reoon (cost/rate) even when primary succeeds, because both outputs are connected.

#### Node: E: Right or Wrong
- **Type/role:** IF — checks Reoon verification booleans (deliverable/SMTP/safe/MX).
- **True path:** **E: Scrap Website HTML** (scrape metadata)
- **False path:** **WE: Scrap Website HTML..** (note: this is odd—invalid email leads into website-no-email scraping block)

#### Node: E: Scrap Website HTML
- **Type/role:** HTTP request — fetches `$('Loop').item.json.Website`.
- **onError:** continueErrorOutput → fallback to **E: Website Scrapping Error**.

#### Node: E: Extract Tag & Attribute
- **Type/role:** HTML extraction — extracts Title, Meta Description, H1.
- **Output:** **E: Seng Process Data**.

#### Node: E: Seng Process Data
- **Type/role:** Set — ensures Title/Meta/H1 present; sets `is_valid_mail=true`.
- **Expressions:** `Title ?? ""`, `Meta Description ?? ""`, etc.

#### Node: E: Website Scrapping Error / E: Email not exist
- **Type/role:** Set — fills Title/Meta/H1 as empty, sets `is_valid_mail=false`.

#### Node: E: Merge
- **Type/role:** Merge combineAll — merges results for downstream “Full Merge”.
- **Output:** to **Full Merge**.

---

### Block 2.7 — Website but No Email Path (Scrape Emails, Pick One with GPT, Verify)
**Overview:** For leads missing Email and Personal Email but having a Website, scrape website HTML, extract candidate emails, pick the best one via GPT‑5‑NANO, then verify and attach `New Email` if valid.
**Nodes involved:**
- **Web Scrap Loop**
- **HTTP Request**
- **Extract Website Details**
- **Email found or not**
- **Find One Correct mail..**
- **Verify**
- **Right or Wrong**
- **Send New Email**
- **Email is Wrong**
- **Extract Tag & Attributee**
- **Tag & Attribute Error**
- **Set empty if not available**
- **Website Scrapping Error**
- **Attribute Merge**
- **Merge**
- **Sticky Note: IF website exist but Email doesn't**

#### Node: Web Scrap Loop
- **Type/role:** Split in batches — iterates over website-only leads.
- **Connections:**
  - Output 0 → **Processed Data Merge** (parallel)
  - Output 1 → **HTTP Request** and **Merge**

#### Node: HTTP Request
- **Type/role:** HTTP GET `{{ $json.Website }}`
- **onError:** continueErrorOutput → **Website Scrapping Error**
- **Success outputs:** feeds both:
  - **Extract Website Details** (regex email extraction)
  - **Extract Tag & Attributee** (metadata extraction)

#### Node: Extract Website Details
- **Type/role:** Code — regex extracts emails from `$json.data` (HTML), dedupes.
- **Output:** `{ emails: [...] }`

#### Node: Email found or not
- **Type/role:** IF (strict) — condition is `operation:false` on `$json.emails.length > 0`  
  Meaning:
  - **True branch** = emails length is **not > 0** (no emails found)
  - **False branch** = emails exist → **Find One Correct mail..**
- **Connections:**
  - True → **Send Email found Data** (sets invalid mail)
  - False → **Find One Correct mail..**

#### Node: Find One Correct mail..
- **Type/role:** LangChain OpenAI node — calls GPT‑5‑NANO to select a single email from list.
- **Prompt:** “Find Only one Correct Email address… return only single email nothing other…”
- **Output:** `message.content` used by **Verify**

#### Node: Verify / Right or Wrong / Send New Email / Email is Wrong
- **Type/role:** Reoon verification + validity gate.
- **Verify URL:** uses `{{ $json.message.content.trim() }}`
- **Right or Wrong IF:** checks Reoon booleans.
- **True:** **Send New Email** sets:
  - `New Email = $json.email`
  - `is_valid_mail = true`
- **False:** **Email is Wrong** sets `is_valid_mail=false`
- **Edge cases:**
  - **Send New Email** references `$json.email`, but the verify response likely doesn’t contain `email` field; it may be `email` or may not—this is a probable bug. It should use the original chosen email (`$json.message.content`) or carry it forward explicitly.

#### Node: Extract Tag & Attributee / Set empty if not available / Tag & Attribute Error
- **Type/role:** HTML metadata extraction with fallbacks.
- **Extract Tag & Attributee keys:** `title`, `meta_description`, `H1` (note key casing differs from other blocks).
- **Set empty if not available:** maps `Title`, `Meta Description`, `H1` but input keys might be `title`/`meta_description`. Potential mismatch.

#### Node: Attribute Merge / Merge
- **Type/role:** Merge combineAll — attempts to merge:
  - Metadata extraction stream
  - Email verification results
  - Original row
- **Edge cases:** combineAll can produce cartesian-like combinations if input sizes differ.

---

### Block 2.8 — Secondary Website Scrape Path (WE: prefix)
**Overview:** A second, parallel implementation of “website scrape for emails” that runs after certain verification outcomes, using `Loop` item context.
**Nodes involved:**
- **WE: Scrap Website HTML..**
- **WE: Scrap Website mails**
- **WE: Extract Tag & Attributee**
- **WE: Set empty if not available**
- **WE: Tag & Attribute Error**
- **WE: Website Scrapping Error**
- **WE: Email found or not**
- **Find One Correct mail**
- **WE: Verify**
- **WE: Right or Wrong**
- **WE: Send New Email**
- **WE: Email is Wrong**
- **WE: Send Email found Data**
- **WP: Merge**

#### Key notes
- This block largely duplicates Block 2.7 but uses different node names and some different key names (Title/Meta Description casing is more consistent here).
- **WE: Email found or not** has the same inverted IF logic (`false` operator with condition `$json.emails.length > 0`).

---

### Block 2.9 — Personal Email Verification (PR/PE)
**Overview:** If Personal Email exists, verify it and set `is_valid_personal_email`; otherwise mark it false. Merged into the same main record.
**Nodes involved:**
- **PR: Exist or not**
- **PE: Not Exist**
- **PE: Verify**
- **PE: Send Data**
- **Full Merge**

#### Node: PR: Exist or not
- **Type/role:** IF — checks `Personal Email == "-"`.
- **True (no personal email):** **PE: Not Exist**
- **False:** **PE: Verify**

#### Node: PE: Verify
- **Type/role:** HTTP Reoon verify.
- **Bug risk:** URL uses `email={{ $json['Email'].trim() }}` (business email), not `Personal Email`. Likely intended to verify Personal Email.

#### Node: PE: Send Data
- **Type/role:** Set — `is_valid_personal_email = is_deliverable && can_connect_smtp && ...`
- **Edge cases:** Downstream nodes sometimes reference misspelled `is_vaid_personal_email` (typo).

---

### Block 2.10 — Full Merge, Filter to “Has Email”, and Personalization Loop
**Overview:** Combines business email validation, personal email validation, and scraped “new email” results; filters to records that have at least one valid email signal; then loops through items to generate AI personalization.
**Nodes involved:**
- **Full Merge**
- **Processed Data Merge**
- **Remove No Email Records**
- **Personalization Loop**
- **Email Switch**
- **Rename New Email Keys**
- **Rename PE Keys**
- **Rename PE + New Email Keys**
- **+1 Email Merge**
- **+2 Email Merge**
- **Sticky Note: Personalize Message & Data Sorting**

#### Node: Full Merge
- **Type/role:** Merge — combineAll; output to **Loop** (forming a processing cycle with batching).
- **Edge cases:** This merge drives the iterative enrichment; misalignment can cause repeated/incorrect combinations.

#### Node: Processed Data Merge
- **Type/role:** Merge — collects processed items and sends to **Remove No Email Records**.

#### Node: Remove No Email Records
- **Type/role:** Filter — keeps items where any of:
  - `$json.is_valid_mail` is true
  - `$json.is_vaid_personal_email` is true (**typo**; should be `is_valid_personal_email`)
  - `$json['New Email']` exists
- **Edge cases:** Because of the typo, valid personal emails may be dropped.

#### Node: Personalization Loop
- **Type/role:** Split in batches — iterates through filtered items.
- **Connections:**
  - Output 0 → **Email Switch**
  - Output 1 → **Personalize Message** and **P: Merge**

#### Node: Email Switch
- **Type/role:** Switch — chooses how to unify email fields before writing to +1/+2 sheets:
  - New Email only
  - Email only
  - Personal Email only
  - E + PE
  - PE + New Email (uses typo `is_vaid_personal_email`)
- **Edge cases:** Several rules depend on typo fields; route may fail or misroute.

#### Nodes: Rename New Email Keys / Rename PE Keys / Rename PE + New Email Keys
- **Type/role:** Rename Keys — standardizes `Email` field and preserves old one as `Wrong Email` in some paths.

#### Nodes: +1 Email Merge / +2 Email Merge
- **Type/role:** Merge — prepares data for writing to target sheet, then runs Replace-with-“-” Code nodes.

---

### Block 2.11 — AI Personalization (LangChain) and Output Shaping
**Overview:** Uses a LangChain LLM chain with structured output parsing to generate cleaned fields and two personalization messages from website/company context and user prompts.
**Nodes involved:**
- **Personalize Message**
- **5 nano**
- **B: Gemini**
- **Output Parser**
- **OpenAI Chat Model**
- **Set AI Response**
- **P: Merge**

#### Node: Personalize Message
- **Type/role:** `@n8n/n8n-nodes-langchain.chainLlm` — orchestrates LLM call, prompt, and structured output parsing.
- **Input text context:** Includes Name, job title, company, industry, keywords, website Title/Meta/H1, etc.
- **System/user prompt:** Uses form prompts:
  - `$('On form submission').item.json['Email Personalization Prompt 1']`
  - `$('On form submission').item.json['Email Personalization Prompt 2']`
  Also asks for “New Name” and shortened “New Company”.
- **needsFallback:** true (Gemini configured as fallback model input).
- **Output:** sent to **Set AI Response**.

#### Node: 5 nano / B: Gemini
- **Type/role:** LLM model providers.
  - 5 nano: OpenAI chat model `gpt-5-nano`
  - Gemini: Google Gemini chat model
- **Connection:** Both feed into Personalize Message as language model options.

#### Node: Output Parser / OpenAI Chat Model
- **Type/role:** Structured output parser with auto-fix; uses `OpenAI Chat Model` for parsing/repair.
- **Schema example:** `{ "New Name": "...", "New Company": "...", "Personalization 1": "...", "Personalization 2": "..." }`

#### Node: Set AI Response
- **Type/role:** Set — maps parsed `output` fields into top-level JSON keys.

#### Node: P: Merge
- **Type/role:** Merge — merges AI output back with original enriched record.

---

### Block 2.12 — Google Sheets Outputs (+1 / +2) and Completion Notification
**Overview:** Writes final personalized rows to Google Sheets, limits to one notification send, and emails completion links.
**Nodes involved:**
- **+1: Replace with "-"**
- **+2: Replace with "-"**
- **+1 Email**
- **+2 Email**
- **Limit**
- **Send a Success Message**

#### Nodes: +1: Replace with "-" / +2: Replace with "-"
- **Type/role:** Code — replaces `""`, `null`, `undefined` with `"-"` across all keys.

#### Nodes: +1 Email / +2 Email
- **Type/role:** Google Sheets append — writes records to separate documents.
  - **+1 Email doc:** `1VJ3OxQh806yQoFqch9ZSN3wPoHiMtX17le-o5qe1x0Y`
  - **+2 Email doc:** `1KHe9AoxR3sgI9-AtqD9mgzcC70MBmzeO368gKkB_nOc`
- **Columns:** extensive mapping including personalization fields.

#### Node: Limit
- **Type/role:** Limit — ensures only one item triggers the success email (prevents multiple sends).

#### Node: Send a Success Message
- **Type/role:** Gmail — sends links to the two spreadsheets.
- **To:** `user@example.com` (placeholder)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Collect file, delivery option, prompts | — | Code | # CSV Lead Qualification & Email Enrichment Workflow (full description block) |
| Code | Code | Split multi-binary into per-file items | On form submission | CSV Loop | # CSV Lead Qualification & Email Enrichment Workflow (full description block) |
| CSV Loop | Split In Batches | Batch/stream file rows & branches | Code; Replace Value | Remove Duplicate Websites; Extract from File | # CSV Lead Qualification & Email Enrichment Workflow (full description block) |
| Extract from File | Extract From File | Parse CSV/XLSX into row items | CSV Loop | Replace Value | ## Filter & Formitize Data |
| Replace Value | Code | Standardize columns to canonical schema | Extract from File | CSV Loop | ## Filter & Formitize Data |
| Remove Duplicate Websites | Remove Duplicates | Deduplicate by Website | CSV Loop | Remove Duplicate Name | ## Filter & Formitize Data |
| Remove Duplicate Name | Remove Duplicates | Deduplicate by Name | Remove Duplicate Websites | Send to Switch | ## Filter & Formitize Data |
| Send to Switch | Switch | Choose Drive / WhatsApp / Sheets processing | Remove Duplicate Name | Convert to File; Convert to File.; Detect Wrong Website URLs  | # CSV Lead Qualification & Email Enrichment Workflow (full description block) |
| Convert to File | Convert To File | Create file for Drive upload | Send to Switch | Upload file | ## Save Filtered Data into Google Drive |
| Upload file | Google Drive | Upload file to Drive folder | Convert to File | Send a message | ## Save Filtered Data into Google Drive |
| Send a message | Gmail | Email Drive link | Upload file | — | ## Save Filtered Data into Google Drive |
| Convert to File. | Convert To File | Create XLSX for WhatsApp | Send to Switch | Send CSV | ## Send Filtered Data to WhatsApp |
| Send CSV | WhatsApp | Send XLSX as document | Convert to File. | — | ## Send Filtered Data to WhatsApp |
| Detect Wrong Website URLs  | Set | Flag unwanted website domains | Send to Switch | Data Processing Switch | # CSV Lead Qualification & Email Enrichment Workflow (full description block) |
| Data Processing Switch | Switch | Route by website/email conditions | Detect Wrong Website URLs  | x Website Data; No Website Loop; Loop; Web Scrap Loop | # CSV Lead Qualification & Email Enrichment Workflow (full description block) |
| x Website Data | Google Sheets | Store “bad website” leads | Data Processing Switch | — | ## Website & Email not exist |
| No Website Loop | Split In Batches | Process records with Website = “-” | Data Processing Switch | Email exist or not; No Website Merge | ## Website not exist but Email exist |
| Email exist or not | IF | Check Email present | No Website Loop | Verify.; Set invalid Email | ## Website not exist but Email exist |
| Verify. | HTTP Request | Reoon verify existing email | Email exist or not | Right or Wrong. | ## Website not exist but Email exist |
| Right or Wrong. | IF | Gate by Reoon deliverability flags | Verify. | Set Valid Email; Set invalid Email. | ## Website not exist but Email exist |
| Set Valid Email | Set | Mark is_valid_mail=true | Right or Wrong. | No Website Merge | ## Website not exist but Email exist |
| Set invalid Email. | Set | Mark is_valid_mail=false | Right or Wrong. | No Website Merge | ## Website not exist but Email exist |
| Set invalid Email | Set | Mark invalid when no email | Email exist or not | No Website Merge | ## Website not exist but Email exist |
| No Website Merge | Merge | Combine and forward to sheet | No Website Loop; Set Valid/invalid | - Website +- Email +- Phone Number | ## Website not exist but Email exist |
| - Website +- Email +- Phone Number | Google Sheets | Store no-website records | No Website Merge | — | ## Website not exist but Email exist |
| Loop | Split In Batches | Main enrichment batching loop | Data Processing Switch; Full Merge | Processed Data Merge; E: Merge/E: exist or not/PR: Exist or not | ##  If Website & Email both exist |
| E: exist or not | IF | Check Email present | Loop | E: Verify; E: Email not exist | ##  If Website & Email both exist |
| E: Verify | HTTP Request | Reoon verify business email | E: exist or not | E: Right or Wrong; E: Verify Backup | ##  If Website & Email both exist |
| E: Verify Backup | HTTP Request | Backup Reoon call | E: Verify | E: Right or Wrong | ##  If Website & Email both exist |
| E: Right or Wrong | IF | Gate by Reoon flags | E: Verify / Backup | E: Scrap Website HTML; WE: Scrap Website HTML.. | ##  If Website & Email both exist |
| E: Scrap Website HTML | HTTP Request | Fetch website HTML | E: Right or Wrong | E: Extract Tag & Attribute; E: Website Scrapping Error | ##  If Website & Email both exist |
| E: Extract Tag & Attribute | HTML | Extract Title/Meta/H1 | E: Scrap Website HTML | E: Seng Process Data | ##  If Website & Email both exist |
| E: Seng Process Data | Set | Set metadata + is_valid_mail=true | E: Extract Tag & Attribute | E: Merge | ##  If Website & Email both exist |
| E: Website Scrapping Error | Set | Fallback empty metadata | E: Scrap Website HTML | E: Merge | ##  If Website & Email both exist |
| E: Email not exist | Set | No email fallback | E: exist or not | E: Merge | ##  If Website & Email both exist |
| E: Merge | Merge | Combine enrichment streams | Loop; E: Email not exist; E: Seng Process Data; WE errors | Full Merge | ##  If Website & Email both exist |
| PR: Exist or not | IF | Check personal email presence | Loop | PE: Not Exist; PE: Verify | ##  If Website & Email both exist |
| PE: Not Exist | Set | is_valid_personal_email=false | PR: Exist or not | Full Merge | ##  If Website & Email both exist |
| PE: Verify | HTTP Request | Reoon verify (intended personal email) | PR: Exist or not | PE: Send Data | ##  If Website & Email both exist |
| PE: Send Data | Set | Set is_valid_personal_email from Reoon | PE: Verify | Full Merge | ##  If Website & Email both exist |
| Full Merge | Merge | Combine all enrichment results | E: Merge; PE: Not Exist/Send Data | Loop | ##  If Website & Email both exist |
| Web Scrap Loop | Split In Batches | Process website-no-email records | Data Processing Switch | Processed Data Merge; HTTP Request; Merge | ## IF website exist but Email doesn't |
| HTTP Request | HTTP Request | Fetch Website URL | Web Scrap Loop | Extract Website Details & Extract Tag & Attributee; Website Scrapping Error | ## IF website exist but Email doesn't |
| Extract Website Details | Code | Regex extract emails from HTML | HTTP Request | Email found or not | ## IF website exist but Email doesn't |
| Email found or not | IF | Branch if emails found | Extract Website Details | Send Email found Data; Find One Correct mail.. | ## IF website exist but Email doesn't |
| Find One Correct mail.. | OpenAI (LangChain) | Pick one email from list | Email found or not | Verify | ## IF website exist but Email doesn't |
| Verify | HTTP Request | Reoon verify chosen email | Find One Correct mail.. | Right or Wrong | ## IF website exist but Email doesn't |
| Right or Wrong | IF | Gate by Reoon flags | Verify | Send New Email; Email is Wrong | ## IF website exist but Email doesn't |
| Send New Email | Set | Set New Email + is_valid_mail | Right or Wrong | Attribute Merge | ## IF website exist but Email doesn't |
| Email is Wrong | Set | is_valid_mail=false | Right or Wrong | Attribute Merge | ## IF website exist but Email doesn't |
| Extract Tag & Attributee | HTML | Extract title/meta/H1 (lowercase keys) | HTTP Request | Set empty if not available; Tag & Attribute Error | ## IF website exist but Email doesn't |
| Set empty if not available | Set | Normalize missing metadata | Extract Tag & Attributee | Attribute Merge | ## IF website exist but Email doesn't |
| Tag & Attribute Error | Set | Metadata fallback | Extract Tag & Attributee | Attribute Merge | ## IF website exist but Email doesn't |
| Website Scrapping Error | Set | Website fetch fallback | HTTP Request | Merge | ## IF website exist but Email doesn't |
| Attribute Merge | Merge | Merge metadata + email validation | Set empty/Tag error; Send New/Email wrong | Merge | ## IF website exist but Email doesn't |
| Merge | Merge | Feeds Web Scrap Loop continuation | Web Scrap Loop; Attribute Merge; Website Scrapping Error | Web Scrap Loop | ## IF website exist but Email doesn't |
| WE: Scrap Website HTML.. | HTTP Request | WE branch website fetch | E: Right or Wrong | WE: Scrap Website mails & WE: Extract Tag & Attributee; WE: Website Scrapping Error | ## IF website exist but Email doesn't |
| WE: Scrap Website mails | Code | WE regex extract emails | WE: Scrap Website HTML.. | WE: Email found or not | ## IF website exist but Email doesn't |
| WE: Email found or not | IF | WE branch if emails found | WE: Scrap Website mails | WE: Send Email found Data; Find One Correct mail | ## IF website exist but Email doesn't |
| Find One Correct mail | OpenAI (LangChain) | WE pick one email | WE: Email found or not | WE: Verify | ## IF website exist but Email doesn't |
| WE: Verify | HTTP Request | WE Reoon verify chosen email | Find One Correct mail | WE: Right or Wrong | ## IF website exist but Email doesn't |
| WE: Right or Wrong | IF | WE deliverability gate | WE: Verify | WE: Send New Email; WE: Email is Wrong | ## IF website exist but Email doesn't |
| WE: Send New Email | Set | WE set New Email | WE: Right or Wrong | WP: Merge | ## IF website exist but Email doesn't |
| WE: Email is Wrong | Set | WE mark invalid | WE: Right or Wrong | WP: Merge | ## IF website exist but Email doesn't |
| WE: Extract Tag & Attributee | HTML | WE extract Title/Meta/H1 | WE: Scrap Website HTML.. | WE: Set empty if not available; WE: Tag & Attribute Error | ## IF website exist but Email doesn't |
| WE: Set empty if not available | Set | WE normalize metadata | WE: Extract Tag & Attributee | WP: Merge | ## IF website exist but Email doesn't |
| WE: Tag & Attribute Error | Set | WE metadata fallback | WE: Extract Tag & Attributee | WP: Merge | ## IF website exist but Email doesn't |
| WE: Website Scrapping Error | Set | WE fetch fallback | WE: Scrap Website HTML.. | E: Merge | ## IF website exist but Email doesn't |
| WE: Send Email found Data | Set | WE no-email-found mark | WE: Email found or not | WP: Merge | ## IF website exist but Email doesn't |
| WP: Merge | Merge | Combine WE outputs into E: Merge | WE metadata and email nodes | E: Merge | ## IF website exist but Email doesn't |
| Processed Data Merge | Merge | Collect processed items | Web Scrap Loop; Loop | Remove No Email Records | # CSV Lead Qualification & Email Enrichment Workflow (full description block) |
| Remove No Email Records | Filter | Keep records that have valid email signal | Processed Data Merge | Personalization Loop | ## Personalize Message & Data Sorting |
| Personalization Loop | Split In Batches | Iterate records for AI personalization | Remove No Email Records | Email Switch; Personalize Message; P: Merge | ## Personalize Message & Data Sorting |
| Email Switch | Switch | Decide email field normalization route | Personalization Loop | Rename New Email Keys; Rename PE Keys; Rename PE + New Email Keys; +1/+2 merges | ## Personalize Message & Data Sorting |
| Rename New Email Keys | Rename Keys | Swap New Email into Email | Email Switch | +1 Email Merge | ## Personalize Message & Data Sorting |
| Rename PE Keys | Rename Keys | Swap Personal Email into Email | Email Switch | +1 Email Merge | ## Personalize Message & Data Sorting |
| Rename PE + New Email Keys | Rename Keys | Swap New Email into Email | Email Switch | +2 Email Merge | ## Personalize Message & Data Sorting |
| +1 Email Merge | Merge | Prepare +1 path payload | Email Switch; Rename nodes | +1: Replace with "-" | ## Personalize Message & Data Sorting |
| +2 Email Merge | Merge | Prepare +2 path payload | Email Switch; Rename nodes | +2: Replace with "-" | ## Personalize Message & Data Sorting |
| +1: Replace with "-" | Code | Normalize empties to “-” | +1 Email Merge | +1 Email | ## Personalize Message & Data Sorting |
| +2: Replace with "-" | Code | Normalize empties to “-” | +2 Email Merge | +2 Email | ## Personalize Message & Data Sorting |
| Personalize Message | LangChain Chain LLM | Generate New Name/Company + personalizations | Personalization Loop; 5 nano/Gemini models; Output Parser | Set AI Response | ## Personalize Message & Data Sorting |
| 5 nano | OpenAI Chat Model | Primary LLM model | — | Personalize Message (ai_languageModel) | ## Personalize Message & Data Sorting |
| B: Gemini | Google Gemini Chat Model | Fallback LLM model | — | Personalize Message (ai_languageModel) | ## Personalize Message & Data Sorting |
| Output Parser | Structured Output Parser | Enforce JSON schema output | OpenAI Chat Model | Personalize Message (ai_outputParser) | ## Personalize Message & Data Sorting |
| OpenAI Chat Model | OpenAI Chat Model | Used by output parser auto-fix | — | Output Parser (ai_languageModel) | ## Personalize Message & Data Sorting |
| Set AI Response | Set | Map parsed AI outputs to top-level keys | Personalize Message | P: Merge | ## Personalize Message & Data Sorting |
| P: Merge | Merge | Merge AI outputs back into record | Personalization Loop; Set AI Response | Personalization Loop | ## Personalize Message & Data Sorting |
| +1 Email | Google Sheets | Append to +1 Email spreadsheet | +1: Replace with "-" | Limit | ## Personalize Message & Data Sorting |
| +2 Email | Google Sheets | Append to +2 Email spreadsheet | +2: Replace with "-" | Limit | ## Personalize Message & Data Sorting |
| Limit | Limit | Prevent multiple completion emails | +1 Email; +2 Email | Send a Success Message | ## Personalize Message & Data Sorting |
| Send a Success Message | Gmail | Notify completion + sheet links | Limit | — | ## Personalize Message & Data Sorting |
| Sticky Note7 | Sticky Note | Comment | — | — | ## IF website exist but Email doesn't |
| Sticky Note | Sticky Note | Comment | — | — | ## Personalize Message & Data Sorting |
| Sticky Note8 | Sticky Note | Comment | — | — | ## Website not exist but Email exist |
| Sticky Note9 | Sticky Note | Comment | — | — | ## Website & Email not exist |
| Sticky Note10 | Sticky Note | Comment | — | — | ## Send Filtered Data to WhatsApp |
| Sticky Note11 | Sticky Note | Comment | — | — | ## Save Filtered Data into Google Drive |
| Sticky Note12 | Sticky Note | Comment | — | — | ## Filter & Formitize Data |
| Sticky Note1 | Sticky Note | Comment | — | — | ##  If Website & Email both exist |
| Sticky Note2 | Sticky Note | Comment | — | — | # CSV Lead Qualification & Email Enrichment Workflow (full description block) |
| (Unused) 6030179e Convert to File | Convert To File | Unused node (not connected) | — | — |  |
| (Unused) 196fff75 Merge | Merge | Connected (part of web scraping loop) | HTTP Request; Attribute Merge; Website Scrapping Error | Web Scrap Loop | ## IF website exist but Email doesn't |
| (Unused/extra) 031ce387 No Website Loop | Split In Batches | Used but partially wired | Data Processing Switch | Email exist or not; No Website Merge | ## Website not exist but Email exist |
| (Unused/extra) 02882e23 Limit | Limit | Used | +1 Email; +2 Email | Send a Success Message | ## Personalize Message & Data Sorting |
| (Unused) 6f11b8ad Set AI Response | Set | Used | Personalize Message | P: Merge | ## Personalize Message & Data Sorting |

> Note: The workflow contains a few nodes that are present but not meaningfully used or have questionable wiring (noted in edge cases above).

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it as provided.

2. **Add trigger: Form Trigger**
   - Node: **Form Trigger**
   - Path: `lead-qualifier`
   - Title: “Bad CSV - Good CSV”
   - Fields:
     1) File “CSV” required; accept `.csv, .xlsx`  
     2) Dropdown “Save Into?” required; options: Google Drive, WhatsApp, Google Sheets with Data Processing  
     3) Textarea “Email Personalization Prompt 1” required  
     4) Textarea “Email Personalization Prompt 2” required

3. **Add Code node to split binary files**
   - Node: **Code**
   - Paste logic that iterates `$input.last().binary` and emits one item per binary property as `binary.data`.
   - Connect: **Form Trigger → Code**

4. **Add Split in Batches for file processing**
   - Node: **Split In Batches** named **CSV Loop**
   - Connect: **Code → CSV Loop**

5. **Add Extract From File**
   - Node: **Extract From File**
   - Binary property: `data`
   - Enable BOM handling
   - Connect: **CSV Loop → Extract From File**

6. **Add Code node for column standardization**
   - Node: **Code** named **Replace Value**
   - Implement canonical schema mapping (as in workflow): initialize all canonical fields with `"-"`, map variants case-insensitively, drop unmapped columns.
   - Connect: **Extract From File → Replace Value**

7. **Deduplicate**
   - Add **Remove Duplicates** node “Remove Duplicate Websites”, compare selected field `Website`.
   - Add **Remove Duplicates** node “Remove Duplicate Name”, compare selected field `Name`.
   - Connect: **CSV Loop → Remove Duplicate Websites → Remove Duplicate Name**
   - Also connect: **Replace Value → CSV Loop** (to feed standardized rows into the loop)

8. **Add Delivery Mode Switch**
   - Node: **Switch** “Send to Switch”
   - Rules based on `$('On form submission').item.json['Save Into?']`:
     - equals “Google Drive”
     - equals “WhatsApp”
     - equals “Google Sheets with Data Processing”
   - Connect: **Remove Duplicate Name → Send to Switch**

### Branch A — Google Drive delivery
9. **Convert to File**
   - Node: **Convert To File** “Convert to File”
   - Keep defaults (or set CSV explicitly if desired).
   - Connect: **Send to Switch (Google Drive output) → Convert to File**

10. **Upload to Google Drive**
   - Node: **Google Drive** “Upload file”
   - Operation: Upload
   - Folder: select your “Good CSV” folder
   - Name: timestamp expression
   - Credentials: **Google Drive OAuth2** (or Service Account with Drive scope)
   - Connect: **Convert to File → Upload file**

11. **Notify via Gmail**
   - Node: **Gmail** “Send a message”
   - Configure Gmail credentials
   - To: change from placeholder to the desired recipient (often form submitter email; not captured currently)
   - Body uses `{{$json.webViewLink}}`
   - Connect: **Upload file → Send a message**

### Branch B — WhatsApp delivery
12. **Convert to XLSX**
   - Node: **Convert To File** “Convert to File.”
   - Operation: `xlsx`
   - Connect: **Send to Switch (WhatsApp output) → Convert to File.**

13. **Send via WhatsApp Business**
   - Node: **WhatsApp** “Send CSV”
   - Operation: send document
   - Configure WhatsApp credentials (token, phone number id, etc.)
   - Set recipient phone number
   - Connect: **Convert to File. → Send CSV**

### Branch C — Google Sheets with Data Processing
14. **Flag wrong websites**
   - Node: **Set** “Detect Wrong Website URLs”
   - Add boolean `is_wrong_website` using domain regex (instagram/facebook/linktr.ee/wa.me etc.)
   - Enable “ignore conversion errors”
   - Connect: **Send to Switch (Sheets processing output) → Detect Wrong Website URLs**

15. **Add Data Processing Switch**
   - Node: **Switch** “Data Processing Switch”
   - Enable **All matching outputs**
   - Create outputs:
     - x Website: `is_wrong_website == true`
     - No Website: `Website == "-"`  
     - Website+Email: `Website != "-" and not is_wrong_website and (Email != "-" or Personal Email != "-")` (recommend parentheses)
     - Website-no-email: `Website != "-" and not is_wrong_website and Email == "-" and Personal Email == "-"`

16. **x Website → Google Sheets**
   - Node: **Google Sheets** “x Website Data” append
   - Create spreadsheet and sheet; map columns as needed
   - Connect: **Data Processing Switch (x Website) → x Website Data**
   - Credentials: Google Sheets OAuth2

17. **No Website path**
   - Use: **Split In Batches** “No Website Loop”
   - IF “Email exist or not” (Email != “-”)
   - HTTP “Verify.” to Reoon (store key in credentials/env var)
   - IF “Right or Wrong.” with Reoon booleans
   - Set nodes to mark `is_valid_mail` true/false
   - Merge “No Website Merge” combineAll
   - Write to Google Sheets “- Website +- Email +- Phone Number”
   - Connect in order:
     - **Data Processing Switch (no website) → No Website Loop → Email exist or not → Verify. → Right or Wrong. → Set Valid/invalid → No Website Merge → - Website +- Email +- Phone Number**

18. **Website+Email path**
   - Add **Split In Batches** “Loop”
   - IF “E: exist or not”
   - HTTP “E: Verify” (+ optional Backup)
   - IF “E: Right or Wrong”
   - HTTP “E: Scrap Website HTML” → HTML “E: Extract Tag & Attribute” → Set “E: Seng Process Data”
   - Fallback Set nodes on errors (E: Website Scrapping Error, E: Email not exist)
   - Merge “E: Merge”
   - Personal email verification:
     - IF “PR: Exist or not”
     - HTTP “PE: Verify” (should verify Personal Email)
     - Set “PE: Send Data” or “PE: Not Exist”
     - Merge with “Full Merge”
   - Feed “Full Merge” back into “Loop” to continue batching (as in original)

19. **Website-no-email path**
   - Add **Split In Batches** “Web Scrap Loop”
   - HTTP fetch website
   - Code extract emails
   - IF email found
   - OpenAI (LangChain) pick one email (gpt-5-nano)
   - HTTP verify chosen email via Reoon
   - Set “New Email” and `is_valid_mail` if good
   - HTML extract metadata and merge with results
   - Merge results into processed stream

20. **Processed stream → filter**
   - Merge node “Processed Data Merge” collects outputs.
   - Filter “Remove No Email Records” keep items with valid email flags / New Email.
   - Fix typos: ensure you consistently use `is_valid_personal_email`.

21. **Personalization**
   - Split In Batches “Personalization Loop”
   - Switch “Email Switch” to normalize Email field scenarios.
   - Rename Keys nodes to standardize `Email` (and store original as “Wrong Email” where needed).
   - Add LangChain **Chain LLM** “Personalize Message”:
     - Provide context text (name/company/metadata).
     - Use the two prompts from the form fields.
     - Attach **Structured Output Parser** with schema keys: New Name, New Company, Personalization 1, Personalization 2.
     - Configure **OpenAI Chat Model gpt-5-nano** as primary; **Gemini** as optional fallback.
   - Set “Set AI Response” to map parser output to fields.
   - Merge back into the record.

22. **Write to Google Sheets (+1 / +2)**
   - Merge routes “+1 Email Merge” and “+2 Email Merge”.
   - Code nodes replace empties with “-”.
   - Append to the appropriate Sheets documents.
   - Add **Limit** then **Gmail** “Send a Success Message” with links.

23. **Credentials to configure**
   - **Google Drive OAuth2** (for Upload file)
   - **Google Sheets OAuth2**
   - **Gmail OAuth2**
   - **WhatsApp Business API** credentials
   - **OpenAI API** (gpt-5-nano) for LangChain OpenAI nodes
   - **Google Gemini API** (optional fallback)
   - **Reoon API key** stored securely (env var/credential); replace `<key>` placeholders

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Full embedded workflow description (purpose, stages, prerequisites, setup steps) | Included in sticky note “CSV Lead Qualification & Email Enrichment Workflow” |
| Google Drive folder “Good CSV” | https://drive.google.com/drive/folders/1DlS21EpQWfBhQVdTvAePkGFgCXdFa2Fe |
| +1 Email Google Sheet | https://docs.google.com/spreadsheets/d/1VJ3OxQh806yQoFqch9ZSN3wPoHiMtX17le-o5qe1x0Y |
| +2 Email Google Sheet | https://docs.google.com/spreadsheets/d/1KHe9AoxR3sgI9-AtqD9mgzcC70MBmzeO368gKkB_nOc |
| Reoon verification endpoint (used multiple times) | `https://emailverifier.reoon.com/api/v1/verify?...&mode=power` |

**Important implementation cautions (from analysis):**
- Deduping on `Website` and `Name` will collapse many rows where values are `"-"`.
- Several conditions use a misspelled variable `is_vaid_personal_email` (should be `is_valid_personal_email`), causing filtering/routing issues.
- Personal Email verification node appears to verify `Email` instead of `Personal Email`.
- Some IF logic is inverted (`operation:false` with `emails.length > 0`)—works, but is easy to misread.
- Some “Set New Email” nodes reference `$json.email` although the verified email is in `$json.message.content`; this likely breaks New Email assignment unless the response contains `email`.