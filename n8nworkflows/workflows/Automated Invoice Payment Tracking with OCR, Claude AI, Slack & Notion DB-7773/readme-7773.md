Automated Invoice Payment Tracking with OCR, Claude AI, Slack & Notion DB

https://n8nworkflows.xyz/workflows/automated-invoice-payment-tracking-with-ocr--claude-ai--slack---notion-db-7773


# Automated Invoice Payment Tracking with OCR, Claude AI, Slack & Notion DB

### 1. Workflow Overview

This workflow automates the process of invoice payment tracking by integrating OCR (Optical Character Recognition), AI-based data extraction (using Claude AI), Slack notifications, and Notion databases for record keeping and status updates. The primary use case is to receive invoice files via Slack, extract and parse invoice data, detect duplicates, verify and update invoice status in a Notion database, and notify the relevant teams for manual review or confirmation.

The workflow is logically divided into these major blocks:

- **1.1 Input Reception & Format Detection:** Receives invoice files from Slack, detects file types (images, PDFs, emails).
- **1.2 OCR Processing:** Downloads files from Slack, sends them to OCR.space API for text extraction.
- **1.3 AI Parsing & Data Extraction:** Uses Claude AI (via Langchain node) to parse extracted OCR text into structured invoice data following a strict schema.
- **1.4 Duplicate Detection:** Runs internal clustering logic to detect duplicate invoices and select a master record.
- **1.5 Database Query & Invoice Matching:** Queries Notion database to check if invoice already exists, matches incoming data with DB records.
- **1.6 Decision Logic for Invoice Status:** Determines next action (create, update, archive, manual review) based on payment status, duplicates, and data mismatches.
- **1.7 Notion Database Operations:** Creates or updates invoice records and related cashflow entries in Notion.
- **1.8 Slack Notifications and Manual Review:** Sends notifications to Slack channels for new invoices, updates, duplicates, or manual review requests. Supports approval interactions via Slack buttons.
- **1.9 Error Handling:** Catches errors and logs them into a Notion error database.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Format Detection

- **Overview:** Triggered by new messages in a specific Slack channel. Extracts attached files and detects their types (image, document, email).
- **Nodes Involved:**
  - Slack Trigger
  - Format Check (Code)
  - Check Format (Switch)
- **Node Details:**

  - **Slack Trigger**
    - Type: Slack Trigger (Webhook)
    - Config: Listens for messages in Slack channel "expenses" (ID: C09701BLY2Z)
    - Output: Message JSON including attached file metadata.
    - Potential failures: Slack API connectivity, missing file attachments.

  - **Format Check**
    - Type: Code node (JavaScript)
    - Role: Parses Slack file metadata, identifies file type and flags if image, document, or email.
    - Uses expressions to extract mimetype and filename.
    - Output: Array of files with added flags for processing.
    - Edge cases: No files attached; unknown mimetype.

  - **Check Format**
    - Type: Switch node
    - Role: Routes files into two streams based on type flags: image or document.
    - Conditions: Checks `isImage` and `isDocument` boolean flags.
    - Edge cases: Files that are neither image nor document are ignored.

---

#### 1.2 OCR Processing

- **Overview:** Downloads files from Slack and sends them to OCR.space API for text extraction.
- **Nodes Involved:**
  - Take Binary Files (for images)
  - OCR Space Parse (for images)
  - Take Binary Files for Document (for PDFs, docs)
  - OCR Space Parse1 (for documents)
  - Check parsing Error2 (if OCR for images errored)
  - Check parsing Error3 (if OCR for documents errored)
  - Stop and Error / Stop and Error1 (stop workflow on OCR failure)
- **Node Details:**

  - **Take Binary Files / Take Binary Files for Document**
    - Type: HTTP Request
    - Role: Downloads the private file URL from Slack using Slack API credentials.
    - Config: Response format set to file binary data.
    - Edge cases: OAuth token expiration, file not found, permissions error.

  - **OCR Space Parse / OCR Space Parse1**
    - Type: HTTP Request
    - Role: Sends binary file data to OCR.space API with multipart/form-data.
    - Config: Filetype is set according to document or image, engine 2 selected for better accuracy.
    - Retry enabled on failure.
    - Edge cases: OCR API rate limits, timeouts, invalid file data.

  - **Check parsing Error2 / Check parsing Error3**
    - Type: If nodes
    - Role: Checks if OCR response indicates an error.
    - On error: Routes to Stop and Error nodes.
    - Edge cases: Unexpected response format.

  - **Stop and Error / Stop and Error1**
    - Type: Stop and Error node
    - Role: Halts the workflow and logs error message.
    - Configured to include OCR error message from API response.

---

#### 1.3 AI Parsing & Data Extraction

- **Overview:** Uses Claude AI (Anthropic) to parse OCR text into a structured invoice JSON matching a strict schema.
- **Nodes Involved:**
  - Merge Data Value into One Key (Code)
  - Basic LLM Chain (Langchain - Claude AI)
  - Anthropic Chat Model4 (AI Model node)
  - Cleans AI Response (Code)
- **Node Details:**

  - **Merge Data Value into One Key**
    - Type: Code node
    - Role: Normalizes OCR text input from either PDF or Image lanes into a single key `OCRResult`.
    - Handles cases with missing or multiple OCR text sources.

  - **Basic LLM Chain**
    - Type: Langchain Chain LLM node
    - Role: Sends OCR text to Claude AI with a detailed prompt instructing extraction of invoice fields into a schema.
    - Prompt includes comprehensive rules on dates, currency, discounts, partial payments, line items, and status.
    - Output: JSON matching EXACT schema for invoice data.

  - **Anthropic Chat Model4**
    - Type: Langchain AI Chat node
    - Role: Specifies usage of Claude-3.5-haiku model for the AI completions.
    - Credential: Anthropic API key.
    - Linked as AI provider for Basic LLM Chain.

  - **Cleans AI Response**
    - Type: Code node
    - Role: Parses the AI response; strips markdown/formatting; enforces schema consistency.
    - Normalizes numeric values, dates, discount math, partial payments, and status logic.
    - Attaches Slack file metadata URLs for traceability.
    - Edge cases: Malformed JSON from AI, missing fields, inconsistent status values.

---

#### 1.4 Duplicate Detection

- **Overview:** Detects duplicate invoices by clustering similar records and picking a master record to keep.
- **Nodes Involved:**
  - Internal Check Duplicate Invoice (Code)
  - Merge with Original Data (Code)
- **Node Details:**

  - **Internal Check Duplicate Invoice**
    - Type: Code node
    - Role: Applies fuzzy canonicalization on vendor and invoice numbers; clusters invoices by vendor+invoice or fallback by vendor+currency+amount+date.
    - Scores clusters preferring fully paid and recent invoices as masters.
    - Outputs clusters with one "keep" and multiple "drop" items.

  - **Merge with Original Data**
    - Type: Code node
    - Role: Merges duplicate cluster results back onto original invoices.
    - Tags each invoice with deduplication role: keep, drop, or unique.
    - Uses signature matching for robustness.

---

#### 1.5 Database Query & Invoice Matching

- **Overview:** Queries Notion invoice database to check if each invoice already exists, using invoice number or heuristic filters.
- **Nodes Involved:**
  - Decide Fate for Data (Code)
  - Create Query for DB (Code)
  - Check DB Invoice (HTTP Request)
  - Merge Items Invoice (Code)
- **Node Details:**

  - **Create Query for DB**
    - Type: Code node
    - Role: Builds Notion database query filters based on invoice number, vendor, currency, amount total, and date windows.
    - Uses ±30 days window for date filtering.
    - Supports fallback queries when invoice number is missing.

  - **Check DB Invoice**
    - Type: HTTP Request
    - Role: Sends POST query to Notion API to search invoices database.
    - Uses Notion API credentials.
    - Edge cases: Notion API rate limits, query errors.

  - **Merge Items Invoice**
    - Type: Code node
    - Role: Compares incoming invoice data with DB results for mismatches (vendor, currency, amount).
    - Establishes flags for partial payments, full payments, and next routing actions.
    - Determines if invoice requires manual review, update, creation, or archiving.

  - **Decide Fate for Data**
    - Type: Code node
    - Role: Entry point for all invoices after deduplication and DB query.
    - Routes invoices for next processing steps based on database presence and payment status.

---

#### 1.6 Decision Logic for Invoice Status

- **Overview:** Routes invoices to create new records, update existing ones, archive duplicates, or manual review.
- **Nodes Involved:**
  - Decide Fate (Switch)
- **Node Details:**

  - **Decide Fate**
    - Type: Switch node
    - Routes invoices based on `next_action` field which can be:
      - create_unpaid
      - create_paid
      - update_mark_paid
      - update_partial
      - create_partial
      - archive
      - manual_review
    - Ensures proper path for each invoice status scenario.

---

#### 1.7 Notion Database Operations

- **Overview:** Creates or updates invoice entries, cashflow ledger entries, and archives duplicates in Notion databases.
- **Nodes Involved:**
  - Create new Invoice Unpaid, Paid Full, Paid Partially (Notion)
  - Update Invoice to Paid Fully, Paid Partial or Fully (Notion)
  - Archive Invoice, Archive Invoice Duplicate (Notion)
  - Add Receipt into Cashflow (Notion)
  - Send to Source File Invoice (multiple variants)
- **Node Details:**

  - **Create new Invoice...**
    - Type: Notion databasePage creation
    - Maps invoice JSON fields into Notion database properties.
    - Uses relations to link source files and receipts.
    - Separate nodes for unpaid, fully paid, and partially paid invoices.

  - **Update Invoice to Paid Fully / Partial**
    - Type: Notion databasePage update
    - Updates payment status, paid amount, and receipt relations.

  - **Archive Invoice / Archive Invoice Duplicate**
    - Saves duplicates or obsolete invoices to separate archival Notion databases.

  - **Add Receipt into Cashflow**
    - Creates cashflow ledger entries for payments.
    - Links to invoice and source file pages.

  - **Send to Source File Invoice**
    - Creates or updates Notion pages to record source file metadata and summaries.

---

#### 1.8 Slack Notifications and Manual Review

- **Overview:** Notifies teams in Slack about new invoices, updates, duplicate alerts, and requests manual review with interactive buttons.
- **Nodes Involved:**
  - Send a message (Slack)
  - Notify New Invoice, Notify Update Invoice, Notify Invoice Archived, Notify Duplicate Notification (Slack)
  - Wait (Wait for manual response)
  - Approval Check (Switch)
  - Take Invoice Details 1/2/3 (Code)
- **Node Details:**

  - **Send a message**
    - Sends detailed Slack message with invoice summary and buttons for Approve Paid, Approve Unpaid, or Archive.
    - Includes invoice metadata and file image preview.
    - Uses Slack API credentials.
    - Edge cases: Slack API rate limits or failures.

  - **Notify ... (Slack nodes)**
    - Send notifications to relevant Slack channels for invoice creation, updates, duplicates, and archive actions.
    - Aggregates URLs and names for clickable summaries.

  - **Wait**
    - Pauses workflow for 2 days, awaiting user interaction via Slack buttons.

  - **Approval Check**
    - Routes based on user choice from Slack buttons (approve paid, unpaid, archive).

  - **Take Invoice Details 1/2/3**
    - Extracts reviewed invoice from prepared list matching invoice number from Slack button click.
    - Annotates invoice with review choice for downstream processing.

---

#### 1.9 Error Handling

- **Overview:** Captures any errors triggered during workflow execution and logs them in a Notion database for error tracking.
- **Nodes Involved:**
  - Error Trigger
  - Create a database page (Notion)
- **Node Details:**

  - **Error Trigger**
    - Captures any node failures in the workflow.

  - **Create a database page**
    - Creates an entry in a dedicated "ERROR CHECK" Notion database with error message and workflow name.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                           | Input Node(s)                       | Output Node(s)                        | Sticky Note                                      |
|-----------------------------------|----------------------------|-----------------------------------------|-----------------------------------|-------------------------------------|-------------------------------------------------|
| Slack Trigger                     | Slack Trigger              | Starts workflow on Slack message        | -                                 | Format Check                        | # 🔤 Input & Detect Format                        |
| Format Check                     | Code                       | Detects file type from Slack data       | Slack Trigger                     | Check Format                       | # 🔤 Input & Detect Format                        |
| Check Format                    | Switch                     | Routes files to image or document path  | Format Check                     | Take Binary Files, Take Binary Files for Document | # 🔤 Input & Detect Format                        |
| Take Binary Files               | HTTP Request               | Downloads image files from Slack        | Check Format                     | OCR Space Parse                   | # 👨‍💻 Parse Data                                 |
| OCR Space Parse                | HTTP Request               | OCR for images via OCR.space API         | Take Binary Files                | Check parsing Error2              | # 👨‍💻 Parse Data                                 |
| Check parsing Error2           | If                         | Checks for OCR errors on image OCR      | OCR Space Parse                 | Stop and Error / Merge Data Value into One Key | # 👨‍💻 Parse Data                                 |
| Stop and Error                | Stop and Error             | Stops workflow on OCR error              | Check parsing Error2            | -                                 |                                                   |
| Take Binary Files for Document | HTTP Request               | Downloads document files from Slack     | Check Format                     | OCR Space Parse1                 | # 👨‍💻 Parse Data                                 |
| OCR Space Parse1              | HTTP Request               | OCR for documents via OCR.space API      | Take Binary Files for Document  | Check parsing Error3              | # 👨‍💻 Parse Data                                 |
| Check parsing Error3           | If                         | Checks for OCR errors on document OCR   | OCR Space Parse1                | Stop and Error1 / Code           | # 👨‍💻 Parse Data                                 |
| Stop and Error1               | Stop and Error             | Stops workflow on OCR error              | Check parsing Error3            | -                                 |                                                   |
| Code                          | Code                       | Merges parsed OCR text                   | Check parsing Error3            | Merge Data Value into One Key     | # 🤖 AI Format & Detect Duplicate                 |
| Merge Data Value into One Key | Code                       | Normalizes OCR text into single key      | Code                          | Basic LLM Chain                  | # 🤖 AI Format & Detect Duplicate                 |
| Basic LLM Chain               | Langchain Chain LLM        | Extracts invoice fields from OCR text    | Merge Data Value into One Key   | Cleans AI Response              | # 🤖 AI Format & Detect Duplicate                 |
| Anthropic Chat Model4         | Langchain AI Chat          | Claude AI model for invoice parsing      | Basic LLM Chain (ai_languageModel) | -                              | # 🤖 AI Format & Detect Duplicate                 |
| Cleans AI Response            | Code                       | Cleans and normalizes AI JSON output     | Basic LLM Chain                | Internal Check Duplicate Invoice  | # 🤖 AI Format & Detect Duplicate                 |
| Internal Check Duplicate Invoice | Code                       | Detects duplicate invoices                | Cleans AI Response            | Merge with Original Data          | # 🤖 AI Format & Detect Duplicate                 |
| Merge with Original Data      | Code                       | Tags invoices with deduplication results | Internal Check Duplicate Invoice | Decide Fate for Data             | # ❌ Archive Duplicates                            |
| Decide Fate for Data          | Code                       | Prepares DB query and routes duplicates  | Merge with Original Data       | Create Query for DB / Prepare Archive Duplicate Items | # 🟢 Check Invoice on DB                           |
| Prepare Archive Duplicate Items | Code                       | Summarizes line items for archiving      | Decide Fate for Data           | Send to Archive Source File Invoice Duplicate | # ❌ Archive Duplicates                            |
| Create Query for DB           | Code                       | Builds Notion DB query filters            | Decide Fate for Data           | Check DB Invoice                 | # 🟢 Check Invoice on DB                           |
| Check DB Invoice              | HTTP Request               | Queries Notion DB for matching invoices  | Create Query for DB            | Merge Items Invoice             | # 🟢 Check Invoice on DB                           |
| Merge Items Invoice           | Code                       | Compares input with DB and decides next action | Check DB Invoice              | Prepare Line Items before send    | # 🟢 Check Invoice on DB                           |
| Prepare Line Items before send | Code                       | Prepares line items text and computes paid amount | Merge Items Invoice           | Decide Fate                     | # 🟢 Sends Data & Notify                           |
| Decide Fate                  | Switch                     | Routes invoices to create/update/archive/manual review | Prepare Line Items before send | Multiple Notion and Slack nodes | # 🟢 Sends Data & Notify                           |
| Create new Invoice Unpaid    | Notion                     | Creates new unpaid invoice in Notion DB  | Send to Source File Invoice 1  | Aggregate                       | # 🟢 Sends Data & Notify                           |
| Create new Invoice Paid Full | Notion                     | Creates new fully paid invoice in Notion DB | Add Receipt into Cashflow Paid 1 | Aggregate2                     | # 🟢 Sends Data & Notify                           |
| Create new Invoice Paid Partially | Notion                  | Creates new partially paid invoice in Notion DB | Add Receipt into Cashflow Paid Partially | Aggregate5              | # 🟢 Sends Data & Notify                           |
| Update Invoice to Paid Fully | Notion                     | Updates existing invoice to fully paid    | Check DB Exist 1              | Aggregate3                      | # 🟢 Sends Data & Notify                           |
| Update Invoice to Paid Partial or Fully | Notion             | Updates existing invoice payment status   | Add Receipt into Cashflow2    | Aggregate6                      | # 🟢 Sends Data & Notify                           |
| Archive Invoice             | Notion                     | Archives duplicate invoice                 | Send to Archive Source File Invoice | Aggregate4                  | # ❌ Archive Duplicates                            |
| Archive Invoice Duplicate   | Notion                     | Archives duplicate invoice copies          | Send to Archive Source File Invoice Duplicate | Aggregate1                | # ❌ Archive Duplicates                            |
| Add Receipt into Cashflow   | Notion                     | Adds payment receipt entry to cashflow DB | Decide Fate                  | Update Invoice to Paid Fully     | # 🟢 Sends Data & Notify                           |
| Add Receipt into Cashflow1  | Notion                     | Adds payment receipt entry for fully paid invoices | Check DB Exist 1            | Update Invoice to Paid Fully 1   | # 🟢 Sends Data & Notify                           |
| Add Receipt into Cashflow2  | Notion                     | Adds payment receipt entry for partial payments | Decide Fate                  | Update Invoice to Paid Partial or Fully | # 🟢 Sends Data & Notify                           |
| Send to Source File Invoice 1/2/3/4 | Notion              | Creates/updates source file metadata pages | Decide Fate / Check DB Exist | Create new Invoice / Add Receipt | # 🟢 Sends Data & Notify                           |
| Send a message              | Slack                      | Sends Slack notification with invoice details and buttons | Decide Fate                 | Wait                           | # 👤 Manual Review & Sends Data                    |
| Wait                       | Wait                       | Waits 2 days for manual review response   | Send a message                | Approval Check                 | # 👤 Manual Review & Sends Data                    |
| Approval Check             | Switch                     | Routes based on Slack review button choice | Wait                         | Take Invoice Details 1/2/3     | # 👤 Manual Review & Sends Data                    |
| Take Invoice Details 1/2/3 | Code                       | Picks reviewed invoice according to Slack choice | Approval Check              | Update / Create / Archive nodes | # 👤 Manual Review & Sends Data                    |
| Error Trigger              | Error Trigger              | Captures workflow errors                   | -                            | Create a database page          |                                                   |
| Create a database page     | Notion                     | Logs error details into Notion error DB    | Error Trigger                | -                             |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up Slack Trigger**  
   - Create Slack Trigger node listening to channel "expenses"  
   - Authenticate with Slack API credentials

2. **File Format Detection**  
   - Add Code node ("Format Check") to parse Slack message files and flag file types  
   - Add Switch node ("Check Format") routing files into two outputs: images and documents

3. **Download Files from Slack**  
   - For images: HTTP Request node ("Take Binary Files") to download file with Slack OAuth  
   - For documents: HTTP Request node ("Take Binary Files for Document") similarly

4. **OCR Processing**  
   - HTTP Request nodes ("OCR Space Parse" for images, "OCR Space Parse1" for documents)  
   - Configure multipart/form-data, pass binary file data, set appropriate filetype  
   - Add If nodes to check OCR success ("Check parsing Error2" and "Check parsing Error3")  
   - Add Stop and Error nodes to halt workflow on OCR failures

5. **Merge OCR Text**  
   - Add Code node ("Code") to join OCR parsed texts  
   - Add Code node ("Merge Data Value into One Key") to unify OCR text sources into key `OCRResult`

6. **AI Parsing with Claude**  
   - Add Langchain Chain LLM node ("Basic LLM Chain")  
     - Configure prompt with target schema and detailed instructions  
     - Set text input to `OCRResult`  
   - Add Anthropic Chat Model node ("Anthropic Chat Model4") to provide Claude 3.5 AI  
   - Add Code node ("Cleans AI Response") to parse and normalize AI JSON output

7. **Duplicate Detection**  
   - Add Code node ("Internal Check Duplicate Invoice") to cluster duplicates  
   - Add Code node ("Merge with Original Data") to tag invoices as keep/drop/unique

8. **Prepare DB Queries**  
   - Add Code node ("Decide Fate for Data") to prepare Notion query filters and route duplicates  
   - Add Code node ("Create Query for DB") to build Notion API queries  
   - Add HTTP Request node ("Check DB Invoice") to query Notion invoice database  
   - Add Code node ("Merge Items Invoice") to compare with DB results and decide next actions

9. **Prepare Line Items and Decide Fate**  
   - Add Code node ("Prepare Line Items before send") to format line items for Notion  
   - Add Switch node ("Decide Fate") to route invoices based on `next_action` field

10. **Notion Database Operations**  
    - Create nodes to:
      - Create unpaid invoice (`Create new Invoice Unpaid`)  
      - Create fully paid invoice (`Create new Invoice Paid Full`)  
      - Create partially paid invoice (`Create new Invoice Paid Partially`)  
      - Update existing invoices (`Update Invoice to Paid Fully`, `Update Invoice to Paid Partial or Fully`)  
      - Archive invoices (`Archive Invoice`, `Archive Invoice Duplicate`)  
      - Add payment receipts to cashflow (`Add Receipt into Cashflow*`)  
      - Send source file metadata (`Send to Source File Invoice *`)  
    - Configure each with appropriate Notion database IDs and map invoice properties accordingly  
    - Use Notion API credentials

11. **Slack Notifications & Manual Review**  
    - Add Slack node ("Send a message") to notify invoice review channel with invoice info and interactive buttons  
    - Add Wait node (2 days delay) waiting for Slack button response  
    - Add Switch node ("Approval Check") to route based on user approval/unpaid/archive selection  
    - Add Code nodes ("Take Invoice Details 1/2/3") to select reviewed invoice from prepared list  
    - Route to appropriate Notion update/create/archive nodes accordingly  
    - Add Slack notification nodes for new invoice, updates, duplicates, archived notices

12. **Error Handling**  
    - Add Error Trigger node to catch runtime errors  
    - Add Notion node ("Create a database page") to log errors in a dedicated Notion error database

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow respects strict content policies; no illegal or offensive data processed.                             | Disclaimer                                                                                          |
| Slack Channels used: "expenses" for input, "required-invoice-review" for manual review, "notification" for alerts | Configured Slack channel IDs in Slack Trigger and Slack nodes                                      |
| Notion Databases: Multiple (Invoice Ledger, Cashflow Ledger, Source File, Archive, Error Check)                 | Notion database IDs embedded in nodes; ensure correct IDs and schema alignment                       |
| AI Model: Claude 3.5 Haiku via Anthropic API integrated through Langchain nodes                                | Requires Anthropic API key credentials                                                             |
| OCR Service: OCR.space API with HTTP header authentication                                                    | Requires OCR.space API key configured under HTTP Header Auth credentials                            |
| Slack API: OAuth2 credentials for file download and messaging                                                 | Requires Slack API credentials set up in n8n                                                       |
| The workflow uses extensive custom JavaScript code nodes for normalization, duplicate detection, and decision logic | Review and test code nodes carefully; edge cases may arise with unexpected invoice formats or missing data |
| Slack manual review buttons use workflow resume URLs with query parameters for approval flow                   | Requires workflow to be exposed to internet with webhook URLs                                      |
| Partial Payment handling toggled by `ENABLE_PARTIAL` flag in code node                                         | Can be modified to disable auto routing partial payments to manual review                           |
| Useful external references: Notion API docs, OCR.space API docs, Anthropic Claude API docs                     | Consult for API updates or credential issues                                                        |

---

This document provides a detailed and structured reference of the Automated Invoice Payment Tracking workflow, enabling knowledgeable users and AI agents to understand, reproduce, and maintain the automation efficiently.