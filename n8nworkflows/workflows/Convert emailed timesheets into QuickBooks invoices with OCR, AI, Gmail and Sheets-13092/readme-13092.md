Convert emailed timesheets into QuickBooks invoices with OCR, AI, Gmail and Sheets

https://n8nworkflows.xyz/workflows/convert-emailed-timesheets-into-quickbooks-invoices-with-ocr--ai--gmail-and-sheets-13092


# Convert emailed timesheets into QuickBooks invoices with OCR, AI, Gmail and Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Convert emailed timesheets into QuickBooks invoices with OCR, AI, Gmail and Sheets

**Purpose:**  
This workflow ingests timesheet emails from Gmail (with attachments), OCRs each attachment into text, uses an AI Agent (Gemini model) to extract month-by-month billable hours and key dates, looks up customer/PO configuration in a Google Sheet, creates/organizes a Google Drive folder hierarchy, writes invoice-ready rows into a Google Sheet (new or existing, with duplicate protection), and finally syncs invoice/customer records to QuickBooks Online.

**Target use cases:**
- Services companies receiving weekly/monthly timesheets by email (PDF/image)
- Automatic invoice preparation based on billable hours
- QuickBooks invoicing driven by customer PO rules and structured Google Sheets records

### Logical Blocks
1.1 **Email Intake & Attachment Loop (Gmail → Binary splitting → loop control)**  
1.2 **OCR Extraction (HTTP OCR API)**  
1.3 **AI Parsing & Month Normalization (AI Agent → month array → per-month splitting)**  
1.4 **Monthly Filtering & First-Month Handling (skip zero-hours, “first time this month” routing)**  
1.5 **Mapping & Invoice Sheet Naming + Date Computation**  
1.6 **Customer/PO Lookup (Google Sheets “Customer POs” master)**  
1.7 **Drive Folder Discovery/Creation + Sheet Discovery**  
1.8 **Sheet Creation/Initialization or Append + Duplicate Protection**  
1.9 **QuickBooks Sync (find/create customer → create invoice)**  
1.10 **Manual/Test Path (manual trigger → “Get row(s)” → QuickBooks nodes)**

---

## 2. Block-by-Block Analysis

### 1.1 Email Intake & Attachment Loop
**Overview:** Watches Gmail for matching emails with attachments, splits all attachments into individual items, and processes them one-by-one in a loop so each attachment becomes its own OCR + AI + invoicing run.

**Nodes involved:**
- Gmail Trigger
- Split Binary Attachments (Code)
- Loop: Process Each Attachment (Split in Batches)
- Wait

#### Node: Gmail Trigger
- **Type/role:** `gmailTrigger` — polling trigger for incoming emails.
- **Config choices:**
  - Query: `has:attachment (subject:"Casper timesheet" OR subject:"Timesheets")`
  - `readStatus: read` (important: it is set to *read*, not unread)
  - Downloads attachments enabled.
  - Polling: every minute.
- **Key data used downstream:**
  - `from.value[0].address` used later to match sender in the PO master sheet.
  - Binary attachments provided under `items[0].binary`.
- **Connections:**
  - Output → **Split Binary Attachments**
- **Failure/edge cases:**
  - Gmail OAuth permission issues; API quota.
  - Query may miss variations in subject or match unwanted emails.
  - If `readStatus` intended to be unread, current configuration may re-process already-read emails depending on Gmail trigger behavior.

#### Node: Split Binary Attachments
- **Type/role:** `code` — converts a single email item containing multiple binaries into multiple items (one per attachment).
- **Logic:** Iterates `items[0].binary` keys and emits one new item per attachment with that binary.
- **Connections:**
  - Output → **Loop: Process Each Attachment**
- **Failure/edge cases:**
  - No attachments: `items[0].binary` undefined → code error.
  - Non-standard binary structure.

#### Node: Loop: Process Each Attachment
- **Type/role:** `splitInBatches` — loop controller.
- **Config choices:** `reset: false` (continues batches across executions unless explicitly reset).
- **Connections:**
  - Main output (index 1) → **Wait**
  - After downstream completion, the workflow routes back into the loop via **Sheets: Append Row1 → Loop: Process Each Attachment**.
- **Failure/edge cases:**
  - If no item returns to the loop, it may stop early.
  - If reset is mismanaged, may not restart cleanly across runs.

#### Node: Wait
- **Type/role:** `wait` — delay to throttle OCR/AI calls.
- **Config:** `amount: 3` (units depend on n8n Wait node default mode; not fully specified in JSON excerpt).
- **Connections:**
  - Output → **Extract Text**
- **Failure/edge cases:**
  - Large volume attachments may still exceed OCR/API limits.

---

### 1.2 OCR Extraction
**Overview:** Sends each attachment to an external OCR/text extraction API and receives plain text for AI parsing.

**Nodes involved:**
- Extract Text  (HTTP Request)

#### Node: Extract Text
- **Type/role:** `httpRequest` — POST multipart form-data to OCR service.
- **Endpoint:** `https://universal-file-to-text-extractor.vercel.app/extract`
- **Body parameters:**
  - `mode=single`
  - `output_type=text`
  - `files` = binary field name `{{ Object.keys($binary)[0] }}`
- **Connections:**
  - Output → **AI Agent**
- **Failure/edge cases:**
  - External service downtime/timeouts.
  - File type unsupported (scanned PDFs with poor quality, password-protected PDFs).
  - Large attachments may exceed request size limits.
  - If `$binary` has multiple keys unexpectedly, only the first is used.

---

### 1.3 AI Parsing & Month Normalization
**Overview:** Uses an AI Agent (powered by Google Gemini model) to extract a strict JSON structure containing employee/client and up to two “month” objects (month1/month2), then normalizes them into an array and splits into per-month items.

**Nodes involved:**
- Google Gemini Chat Model
- AI Agent
- Prepare Month array
- Split Each Month

#### Node: Google Gemini Chat Model
- **Type/role:** LangChain chat model wrapper for Gemini.
- **Config:** temperature 0.2 (more deterministic extraction).
- **Credentials:** Google PaLM/Gemini API.
- **Connections:**
  - AI languageModel output → **AI Agent**
- **Failure/edge cases:**
  - API key/permission issues.
  - Model changes may affect JSON formatting reliability.

#### Node: AI Agent
- **Type/role:** LangChain agent — prompt-driven extractor.
- **Config choices:**
  - Prompt instructs: extract **only** “BILLABLE HOURS or DAYS” section, ignore non-billable/holiday/leave, split weeks across months, include months even if total hours = 0, return **valid JSON only**.
  - Injected OCR text: `{{ $json.data[0].text }}`
- **Outputs used downstream:**
  - `output.parseJson()` is repeatedly called later to access fields like `Employee Name`, `Client`, `month1`, `month2`.
- **Connections:**
  - Output → **Prepare Month array**
- **Failure/edge cases:**
  - If OCR output shape differs (not `data[0].text`), expression fails.
  - Model may return invalid JSON → parse failures later.
  - Prompt assumes up to two months (`month1`, `month2`); timesheets spanning >2 months won’t be fully captured.

#### Node: Prepare Month array
- **Type/role:** `set` — constructs a “months” array-like field and derives a target month value for branching.
- **Key expressions:**
  - `months` value is set to: `JSON.stringify([ month1, month2 ])`  
    (Note: it becomes a **string**, not a native array.)
  - `Target month1` derived from month1 week starting date:  
    `{{ $json.output.parseJson().month1['Week Starting Date'].split("/")[0] }}`
- **Connections:**
  - Output → **Split Each Month**
- **Failure/edge cases:**
  - If `month2` missing, array contains `null/undefined` and later split may behave unexpectedly.
  - Because `months` is JSON-stringified, `splitOut` may not split as intended unless n8n auto-converts; this is a likely bug source.

#### Node: Split Each Month
- **Type/role:** `splitOut` — splits items by field `months`.
- **Config:** `fieldToSplitOut = months`
- **Connections:**
  - Output → **If1**
- **Failure/edge cases:**
  - If `months` is a string (not array), node may split by characters or fail depending on n8n version/behavior.
  - Null entries for month2 can create empty items.

---

### 1.4 Monthly Filtering & First-Month Handling
**Overview:** Filters out empty/zero-hour month entries and routes the “first month” through a different path (including a Wait node) to coordinate downstream mapping.

**Nodes involved:**
- If1
- If: First Time This Month?
- Wait1

#### Node: If1
- **Type/role:** `if` — filters valid month entries.
- **Conditions:**
  - `Total Hours` != `"0"` (string comparison; loose validation enabled)
  - `month` is not empty
- **Connections:**
  - True → **If: First Time This Month?**
- **Failure/edge cases:**
  - If hours are numeric `0` vs string `"0"`, loose validation helps but still brittle.
  - Prompt explicitly asked to include zero-hour months; this node contradicts that by filtering them out.

#### Node: If: First Time This Month?
- **Type/role:** `if` — routes based on whether the month entry is the first month extracted.
- **Condition:**
  - `Week Starting Date` month number equals `Target month1`
- **Connections:**
  - True → **Map Timesheet Fields**
  - False → **Wait1** → **Map Timesheet Fields**
- **Purpose of Wait1 path:** likely to ensure ordering or to avoid collisions when creating sheets for subsequent month entries.
- **Failure/edge cases:**
  - Assumes `Week Starting Date` always exists and is `MM/DD/YYYY`.

#### Node: Wait1
- **Type/role:** `wait`
- **Config:** `amount: 2`
- **Connections:**
  - Output → **Map Timesheet Fields**

---

### 1.5 Mapping & Invoice Sheet Naming + Date Computation
**Overview:** Converts parsed AI fields into normalized timesheet fields, then generates a deterministic sheet name and computes “last date of current month” for invoice/due date calculations.

**Nodes involved:**
- Map Timesheet Fields
- Create Sheet Name + Invoice Date

#### Node: Map Timesheet Fields
- **Type/role:** `set` — standardizes fields.
- **Key mappings:**
  - `Employee Name` from `AI Agent output.parseJson()['Employee Name']`
  - `Client Name` from `AI Agent output.parseJson().Client`
  - `Month`, `Year`, `Total Hours`, week start/end taken from current month item (`$json`)
- **Connections:**
  - Output → **Create Sheet Name + Invoice Date**
- **Failure/edge cases:**
  - If AI JSON keys differ in capitalization/spelling, mappings fail.
  - Relies on `$('AI Agent').item.json...` which refers to a specific item; in loops, item pairing can drift if not consistent.

#### Node: Create Sheet Name + Invoice Date
- **Type/role:** `set` — generates sheet name + computes month end date.
- **Sheet Name formula:**
  - `EmployeeNameNoSpaces_MonthNoSpaces_CurrentYear`
- **“last date of current month” logic:**
  - Uses a month name → index map and `Date.UTC(year, monthIndex + 1, 0)` to get the last day.
  - Outputs ISO date `YYYY-MM-DD`.
  - Returns a string `Invalid input: ...` if month/year invalid.
- **Connections:**
  - Output → **Get Customer Info From PO Sheet**
- **Failure/edge cases:**
  - If month is not exact English month name (e.g., “Jul”), it returns Invalid input.
  - Downstream date expressions assume a valid date string; invalid string will break `.toDateTime()` conversions.

---

### 1.6 Customer/PO Lookup
**Overview:** Looks up the email sender in a master Google Sheet to obtain customer account number, PO number, item name, folder naming rules, and due date calculation parameters.

**Nodes involved:**
- Get Customer Info From PO Sheet

#### Node: Get Customer Info From PO Sheet
- **Type/role:** `googleSheets` — lookup rows via filter.
- **Lookup filter:**
  - Column: `Email`
  - Value: sender address from Gmail Trigger: `{{ $('Gmail Trigger').item.json.from.value[0].address }}`
- **Document:** Google Sheet by URL (configured but value redacted in JSON as `=`); cached URL shown:
  - `https://docs.google.com/spreadsheets/d/11iUiyjpjaz6pNtSSncU3hwPvBzI5y2Gp8Tvuamjkd98/edit#gid=0`
- **Connections:**
  - Output → **Search: Client Invoices Folder**
- **Failure/edge cases:**
  - No matching sender row → missing PO fields later (folder name, due date offset, item, account number).
  - Sheet permissions / OAuth issues.
  - Column names must match exactly (Email, PO Number, Item, Folder Name, Due Date Calculation, Customer Account Number, etc.).

---

### 1.7 Drive Folder Discovery/Creation + Sheet Discovery
**Overview:** Ensures a folder hierarchy exists under `01-ClientInvoices`, searches for employee folder and year folder, then checks whether the invoice spreadsheet (named by employee+month+year) already exists.

**Nodes involved:**
- Search: Client Invoices Folder
- Search: Employee Name Folder
- Check Employee Name Folder
- Create Employee Name Folder
- Search: Inside folders in Employee Name Folder
- Search: folder name
- Create  Folder Name
- Create: Year Folder
- Search: Year Folder
- Check if Year Folder Exists
- Create Current Year Folder
- Google Drive (search sheet file)
- If- File is Exist

#### Node: Search: Client Invoices Folder
- **Type/role:** `googleDrive` — search root folder.
- **Query:** `01-ClientInvoices`
- **Connections:** → **Search: Employee Name Folder**
- **Failure/edge cases:** Folder not found → downstream folderId missing.

#### Node: Search: Employee Name Folder
- **Type/role:** `googleDrive` search for a folder named employee.
- **Parent folderId:** from previous node’s `id`.
- **Query:** employee name from mapped fields.
- **Connections:** → **Check Employee Name Folder**
- **Failure/edge cases:** If multiple folders match, `.item.json.id` might not be deterministic.

#### Node: Check Employee Name Folder
- **Type/role:** `if` existence check on `$json.id`
- **True path:** → **Search: Inside folders in Employee Name Folder**
- **False path:** → **Create Employee Name Folder**
- **Edge cases:** If Drive API returns empty array, item may not exist depending on node settings.

#### Node: Create Employee Name Folder
- **Type/role:** `googleDrive` create folder under `01-ClientInvoices`.
- **Connections:** → **Create  Folder Name**
- **Edge cases:** Folder already exists (name collision) can cause duplicates.

#### Node: Search: Inside folders in Employee Name Folder
- **Type/role:** `googleDrive` search folders inside employee folder.
- **Query string:** empty (`=`), returns all folders.
- **Connections:** → **Search: folder name**
- **Edge cases:** Uses `.first().json.id` later; if no folders exist, this breaks.

#### Node: Search: folder name
- **Type/role:** `googleDrive` search for customer folder name (from PO sheet) inside the found folderId.
- **Connections:** → **Search: Year Folder**
- **Edge cases:** If wrong parent folder chosen (due to `.first()`), folder structure may drift.

#### Node: Create  Folder Name
- **Type/role:** `googleDrive` create folder named from PO sheet `Folder Name`.
- **Connections:** → **Create: Year Folder**
- **Edge cases:** Same folder name existing; permissions.

#### Node: Create: Year Folder
- **Type/role:** `googleDrive` create year folder under folder name.
- **Connections:** → **Check if Year Folder Exists**
- **Note:** Despite name “Create: Year Folder”, it creates unconditionally on that branch.

#### Node: Search: Year Folder
- **Type/role:** `googleDrive` search year folder (string year) under a folder id taken from `Search: Inside folders...first()`.
- **Connections:** → **Check if Year Folder Exists**
- **Edge cases:** Same as above; `.first()` dependency.

#### Node: Check if Year Folder Exists
- **Type/role:** `if` checks existence of `$json.id`
- **True path:** → **Google Drive** (search for sheet file)
- **False path:** → **Create Current Year Folder** → **Google Drive**
- **Edge cases:** If multiple year folders exist, returned id may be arbitrary.

#### Node: Create Current Year Folder
- **Type/role:** `googleDrive` create folder named Year.
- **Parent folderId:** `Search: folder name`.json.id
- **Connections:** → **Google Drive**

#### Node: Google Drive
- **Type/role:** `googleDrive` search for an existing spreadsheet file by sheet name.
- **FolderId:** the year folder id from either search or create.
- **Query:** `Sheet Name` produced earlier.
- **Connections:** → **If- File is Exist**
- **Failure/edge cases:** Multiple matches; permission; eventual consistency after creation.

#### Node: If- File is Exist
- **Type/role:** `if` checks if `$json.id` exists.
- **True path:** → **Check for Duplicate Entry**
- **False path:** → **Google Sheets: Create Sheet**
- **Failure/edge cases:** Condition uses `rightValue="=\n"` (odd literal). Still works as “exists” check, but brittle.

---

### 1.8 Sheet Creation/Initialization or Append + Duplicate Protection
**Overview:** If a spreadsheet exists, prevents duplicates and conditionally deletes an initial “January first week” row; if it doesn’t exist, creates and moves the spreadsheet, seeds a header/default row, then appends invoice rows.

**Nodes involved:**
- Google Sheets: Create Sheet
- Move Sheet to Invoice Folder
- Prepare Default Invoice Row
- Google Sheets2 (append template row)
- Set: Spreadsheet (ID & Name)
- Sheets: Append Row1
- Check for Duplicate Entry
- Skip January First Time
- Delete Exiting Row
- Skip If Duplicate Found

#### Node: Google Sheets: Create Sheet
- **Type/role:** `googleSheets` create spreadsheet.
- **Config:** title = generated `Sheet Name`; creates initial sheet “Sheet1”.
- **Connections:** → **Move Sheet to Invoice Folder**
- **Edge cases:** Name collisions; API quota.

#### Node: Move Sheet to Invoice Folder
- **Type/role:** `googleDrive` move file into year folder.
- **folderId:** from `Check if Year Folder Exists` or `Create Current Year Folder`.
- **Connections:** → **Prepare Default Invoice Row**
- **Edge cases:** Move permission errors; shared drives vs My Drive.

#### Node: Prepare Default Invoice Row
- **Type/role:** `set` raw JSON — provides empty columns to ensure schema exists.
- **executeOnce:** true (only once per workflow activation context).
- **Connections:** → **Google Sheets2**
- **Edge cases:** If executeOnce prevents initialization for later created sheets in same run.

#### Node: Google Sheets2
- **Type/role:** `googleSheets` append — seeds the new sheet with default row matching schema.
- **Sheet:** uses sheetId from created spreadsheet’s first sheet.
- **Connections:** → **Set: Spreadsheet (ID & Name)**
- **executeOnce:** true
- **Edge cases:** If schema differs from “Append Row1”, later append may misalign.

#### Node: Set: Spreadsheet (ID & Name)
- **Type/role:** `set` — surfaces `id` and `name` of the spreadsheet for later nodes.
- **Connections:** → **Sheets: Append Row1**

#### Node: Check for Duplicate Entry
- **Type/role:** `googleSheets` lookup with filters.
- **Filter:**
  - lookupColumn: `Decription`
  - lookupValue: `Week ending date {{ Map Timesheet Fields Week Ending Date }}`
- **Connections:** → **Skip January First Time**
- **Edge cases:**
  - Column name typo: `Decription` (missing “s”) is used throughout; must match sheet header exactly.
  - If timesheet description format changes, duplicates will slip through.

#### Node: Skip January First Time
- **Type/role:** `if` — special-case for first week of year.
- **Condition:**
  - `parseInt($json["Invoice Date"].split("-")[0].trim()) == 1`
  - This expects `Invoice Date` formatted `MM-dd-yyyy` (month first).
- **Connections:**
  - True → **Delete Exiting Row**
  - False → **Skip If Duplicate Found**
- **Edge cases:**
  - If Invoice Date format changes (e.g., `YYYY-MM-DD`), this logic breaks.

#### Node: Delete Exiting Row
- **Type/role:** `googleSheets` delete row(s).
- **DocumentId:** from **If- File is Exist** item.json.id
- **Connections:** → **Sheets: Append Row1**
- **Edge cases:** Delete operation may delete wrong row if row reference isn’t properly set (not visible in excerpt). As shown, it lacks a row number/range parameter, so behavior depends on node defaults and may be incorrect.

#### Node: Skip If Duplicate Found
- **Type/role:** `if` — skips appending if duplicate exists.
- **Condition:** checks whether `$json.Decription` exists (from duplicate lookup result).
- **Connections:**
  - True (duplicate exists) → **Loop: Process Each Attachment** (skip append)
  - False → **Sheets: Append Row1**
- **Edge cases:** If lookup node always outputs data (it does), but with empty fields, existence check may be unreliable.

#### Node: Sheets: Append Row1
- **Type/role:** `googleSheets` append — writes the invoice line row.
- **Key mapped columns:**
  - Customer Account Number: from PO sheet
  - Invoice Date: last day of month formatted `MM-dd-yyyy`
  - Due Date: last day of month + “Due Date Calculation” days offset
  - PO Number: from PO sheet
  - Item Name: from PO sheet
  - Quantity: `Total Hours` from loop month item
  - Quantity Column: `"Hours"`
  - Item Column Title: `"Items"`
- **DocumentId selection:**
  - `{{ $json.id || $('If- File is Exist').item.json.id }}`
  - If a new sheet path is used, `$json.id` comes from `Set: Spreadsheet`.
- **Connections:** → **Loop: Process Each Attachment** (to continue loop)
- **Edge cases:**
  - Date conversion uses `.toDateTime()`; requires n8n Luxon support and valid ISO date input.
  - `Due Date Calculation` must be numeric; if string/non-number, `.plus(..., 'days')` may fail.

---

### 1.9 QuickBooks Sync (from Sheet rows)
**Overview:** A separate path (manual trigger) reads rows from a Google Sheet and pushes them into QuickBooks: find customer, create if missing, and create invoice.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Get row(s) in sheet
- QuickBooks Find Customer
- If Customer Exists?
- QuickBooks Create Customer
- QuickBooks Create Invoice

#### Node: When clicking ‘Execute workflow’
- **Type/role:** `manualTrigger` — used for testing/manual invoice sync.
- **Connections:** → **Get row(s) in sheet**

#### Node: Get row(s) in sheet
- **Type/role:** `googleSheets` read rows.
- **Config:** documentId and sheetName are not properly set in provided JSON (both appear empty/`=`).
- **Connections:** → **QuickBooks Find Customer**
- **Edge cases:** This node likely won’t work until you set Spreadsheet ID and sheet name/range.

#### Node: QuickBooks Find Customer
- **Type/role:** `quickbooks` getAll customers with query filter.
- **Filter query:** `WHERE DisplayName = '{{ $json["Customer Name"] }}'`
- **ReturnAll:** expression uses `Customer Account Number` (odd usage; QuickBooks node expects boolean/limit semantics—may be misconfigured).
- **alwaysOutputData:** true
- **Connections:** → **If Customer Exists?**
- **Failure/edge cases:** OAuth token expiry, company realm mismatch, query returning multiple customers.

#### Node: If Customer Exists?
- **Type/role:** `if` checks emptiness of `Customer Account Number` coming from `Get row(s) in sheet`.
- **Logic:** If empty → create customer; else → create invoice.
- **Connections:**
  - True branch → **QuickBooks Create Invoice**
  - False branch → **QuickBooks Create Customer**
- **Edge cases:** The condition appears inverted for “exists”; it checks if account number is empty, not whether QuickBooks search found a customer.

#### Node: QuickBooks Create Customer
- **Type/role:** `quickbooks` create customer.
- **Fields:** DisplayName, CompanyName, PrimaryEmailAddr from sheet row.
- **Connections:** → **QuickBooks Create Invoice**
- **Edge cases:** Duplicate customer names; required fields missing.

#### Node: QuickBooks Create Invoice
- **Type/role:** `quickbooks` create invoice.
- **Line mapping:** Amount and TotalAmt both mapped from `Quantity` (hours), but invoice amount usually should be hours * rate.
- **CustomerRef:** hard-coded `"66"` (critical: not dynamically linked to found/created customer).
- **Custom fields:**
  - Customer Account Number, PO Number
  - DefinitionId values appear incorrectly set as strings like `=Customer Account Number`.
- **Edge cases:**
  - Hard-coded CustomerRef will invoice the wrong customer unless updated.
  - `itemId: "2"` is hard-coded; must match actual QuickBooks Item.
  - Date formats must match QuickBooks expectations (usually `YYYY-MM-DD`).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | gmailTrigger | Poll Gmail for timesheet emails + download attachments | — | Split Binary Attachments | ## Email intake and loop<br>Listens to Gmail for unread timesheet emails, splits all attachments, and processes each one in a loop so multiple timesheets in the same email are handled independently. |
| Split Binary Attachments | code | Split multiple binaries into one item per attachment | Gmail Trigger | Loop: Process Each Attachment | ## Email intake and loop<br>Listens to Gmail for unread timesheet emails, splits all attachments, and processes each one in a loop so multiple timesheets in the same email are handled independently. |
| Loop: Process Each Attachment | splitInBatches | Iterate attachments one-by-one | Split Binary Attachments; Sheets: Append Row1; Skip If Duplicate Found | Wait | ## Email intake and loop<br>Listens to Gmail for unread timesheet emails, splits all attachments, and processes each one in a loop so multiple timesheets in the same email are handled independently. |
| Wait | wait | Throttle before OCR | Loop: Process Each Attachment | Extract Text  |  |
| Extract Text  | httpRequest | OCR/file-to-text extraction | Wait | AI Agent | ## OCR extraction<br>Sends each attachment to the OCR API and returns plain text. This turns PDFs and images into machine readable text before AI parsing. |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM backend for AI Agent | — | AI Agent (ai_languageModel) | ## AI timesheet parsing<br>OpenAI reads the timesheet text and extracts employee name, client name, week start and end dates, total hours, and current year into clean JSON fields for later use. |
| OpenAI Chat Model1 | lmChatOpenAi | Unused model node (not connected) | — | — | ## AI timesheet parsing<br>OpenAI reads the timesheet text and extracts employee name, client name, week start and end dates, total hours, and current year into clean JSON fields for later use. |
| AI Agent | agent | Parse OCR text into structured JSON | Extract Text ; Google Gemini Chat Model | Prepare Month array | ## AI timesheet parsing<br>OpenAI reads the timesheet text and extracts employee name, client name, week start and end dates, total hours, and current year into clean JSON fields for later use. |
| Prepare Month array | set | Build months list + derive target month | AI Agent | Split Each Month | ## normalize and extract timesheet month<br>Prepare Month array and extract month number and split based on month |
| Split Each Month | splitOut | Split each month entry into its own item | Prepare Month array | If1 | ## normalize and extract timesheet month<br>Prepare Month array and extract month number and split based on month |
| If1 | if | Filter valid month entries | Split Each Month | If: First Time This Month? | ## check occurrence than map the data<br>check  If: First Time This Month? and than map the extracted data   and create sheet +invoice date |
| If: First Time This Month? | if | Route first month differently | If1 | Map Timesheet Fields; Wait1 | ## check occurrence than map the data<br>check  If: First Time This Month? and than map the extracted data   and create sheet +invoice date |
| Wait1 | wait | Delay for non-first month path | If: First Time This Month? | Map Timesheet Fields |  |
| Map Timesheet Fields | set | Normalize extracted fields for downstream | If: First Time This Month?; Wait1 | Create Sheet Name + Invoice Date | ## check occurrence than map the data<br>check  If: First Time This Month? and than map the extracted data   and create sheet +invoice date |
| Create Sheet Name + Invoice Date | set | Build sheet name + compute month-end date | Map Timesheet Fields | Get Customer Info From PO Sheet | ## Build invoice dates<br>Calculates the invoice date and due date using the timesheet's week end date and PO due date rules. Outputs clean fields used to name and populate the invoice sheet. |
| Get Customer Info From PO Sheet | googleSheets | Lookup sender configuration (PO/account/item/folder) | Create Sheet Name + Invoice Date | Search: Client Invoices Folder | ## Customer and PO lookup<br>Looks up the sender in the Customer POs sheet and pulls account number, PO number, item name, folder name, invoice range, and due date offset that drive the invoice logic. |
| Search: Client Invoices Folder | googleDrive | Find root invoice folder | Get Customer Info From PO Sheet | Search: Employee Name Folder | ## Client folder discovery<br>Search Client Invoices folder, finds or creates the client level folder based on the timesheet client name and PO configuration. |
| Search: Employee Name Folder | googleDrive | Find employee folder | Search: Client Invoices Folder | Check Employee Name Folder | ## Client folder discovery<br>Search Client Invoices folder, finds or creates the client level folder based on the timesheet client name and PO configuration. |
| Check Employee Name Folder | if | If employee folder exists | Search: Employee Name Folder | Search: Inside folders in Employee Name Folder; Create Employee Name Folder | ## Create new <br>create new client name, employee and year named folder |
| Create Employee Name Folder | googleDrive | Create employee folder | Check Employee Name Folder | Create  Folder Name | ## Create new <br>create new client name, employee and year named folder |
| Search: Inside folders in Employee Name Folder | googleDrive | List/search subfolders inside employee folder | Check Employee Name Folder | Search: folder name | ## ## File naming and search<br> searches Drive for existing sheets with the employee and if it exist get that year's folder |
| Search: folder name | googleDrive | Find customer folder name | Search: Inside folders in Employee Name Folder | Search: Year Folder  | ## ## File naming and search<br> searches Drive for existing sheets with the employee and if it exist get that year's folder |
| Search: Year Folder  | googleDrive | Find year folder | Search: folder name | Check if Year Folder Exists |  |
| Check if Year Folder Exists | if | Route create year folder if missing | Search: Year Folder ; Create: Year Folder | Google Drive; Create Current Year Folder | ## create new year folder |
| Create Current Year Folder | googleDrive | Create year folder under customer folder | Check if Year Folder Exists | Google Drive | ## create new year folder |
| Google Drive | googleDrive | Search for existing spreadsheet by name | Check if Year Folder Exists; Create Current Year Folder | If- File is Exist |  |
| If- File is Exist | if | Branch: existing sheet vs create | Google Drive | Check for Duplicate Entry; Google Sheets: Create Sheet |  |
| Google Sheets: Create Sheet | googleSheets | Create invoice spreadsheet | If- File is Exist | Move Sheet to Invoice Folder |  |
| Move Sheet to Invoice Folder | googleDrive | Move created spreadsheet into correct Drive folder | Google Sheets: Create Sheet | Prepare Default Invoice Row |  |
| Prepare Default Invoice Row | set | Seed schema row for new sheet | Move Sheet to Invoice Folder | Google Sheets2 |  |
| Google Sheets2 | googleSheets | Append default row into new sheet | Prepare Default Invoice Row | Set: Spreadsheet  (ID & Name) |  |
| Set: Spreadsheet  (ID & Name) | set | Store spreadsheet id/name for later appends | Google Sheets2 |  Sheets: Append Row1 |  |
| Check for Duplicate Entry | googleSheets | Look for duplicate by description | If- File is Exist | Skip January First Time | ## check for duplicate and  Skip January First Time  (first week of the year) |
| Skip January First Time | if | Special handling for January first week | Check for Duplicate Entry | Delete Exiting Row; Skip If Duplicate Found | ## check for duplicate and  Skip January First Time  (first week of the year) |
| Delete Exiting Row | googleSheets | Delete an existing row (intended cleanup) | Skip January First Time |  Sheets: Append Row1 | ## add extracted data to sheets |
| Skip If Duplicate Found | if | Skip append if duplicate exists | Skip January First Time | Loop: Process Each Attachment;  Sheets: Append Row1 | ## skip if duplicate is found  |
|  Sheets: Append Row1 | googleSheets | Append invoice line row into Sheet1 | Set: Spreadsheet  (ID & Name); Skip If Duplicate Found; Delete Exiting Row | Loop: Process Each Attachment | ## add extracted data to sheets |
| Create  Folder Name | googleDrive | Create customer folder name | Create Employee Name Folder | Create: Year Folder | ## Create new <br>create new client name, employee and year named folder |
| Create: Year Folder | googleDrive | Create year folder (new structure branch) | Create  Folder Name | Check if Year Folder Exists | ## Create new <br>create new client name, employee and year named folder |
| When clicking ‘Execute workflow’ | manualTrigger | Manual entry for QuickBooks sync testing | — | Get row(s) in sheet |  |
| Get row(s) in sheet | googleSheets | Read invoice rows for QuickBooks | When clicking ‘Execute workflow’ | QuickBooks Find Customer | ## get data from sheets<br>extracted data  from sheets |
| QuickBooks Find Customer | quickbooks | Search customer by display name | Get row(s) in sheet | If Customer Exists? | ## quickbooks customer info <br>from quickbooks find customer info that match customer name in sheets |
| If Customer Exists? | if | Decide create customer vs invoice | QuickBooks Find Customer | QuickBooks Create Invoice; QuickBooks Create Customer | ## check if customer exists<br><br>if exists continue else add new customer  info |
| QuickBooks Create Customer | quickbooks | Create customer in QuickBooks | If Customer Exists? | QuickBooks Create Invoice | ## check if customer exists<br><br>if exists continue else add new customer  info |
| QuickBooks Create Invoice | quickbooks | Create invoice in QuickBooks | If Customer Exists?; QuickBooks Create Customer | — | ## add data into Quickbooks<br>add data from sheets into Quickbooks invoice  |
| Sticky Note | stickyNote | Comment | — | — |  |
| Sticky Note1 | stickyNote | Comment | — | — |  |
| Sticky Note2 | stickyNote | Comment | — | — |  |
| Sticky Note3 | stickyNote | Comment | — | — |  |
| Sticky Note5 | stickyNote | Comment | — | — |  |
| Sticky Note6 | stickyNote | Comment | — | — |  |
| Sticky Note7 | stickyNote | Comment | — | — |  |
| Sticky Note8 | stickyNote | Comment | — | — |  |
| Sticky Note9 | stickyNote | Comment | — | — |  |
| Sticky Note10 | stickyNote | Comment | — | — |  |
| Sticky Note11 | stickyNote | Comment | — | — |  |
| Sticky Note12 | stickyNote | Comment | — | — |  |
| Sticky Note14 | stickyNote | Comment | — | — |  |
| Sticky Note15 | stickyNote | Comment | — | — |  |
| Sticky Note16 | stickyNote | Comment | — | — |  |
| Sticky Note17 | stickyNote | Comment | — | — |  |
| Sticky Note19 | stickyNote | Comment | — | — |  |
| Sticky Note20 | stickyNote | Comment | — | — |  |
| Sticky Note21 | stickyNote | Comment | — | — |  |
| Sticky Note22 | stickyNote | Comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
1. Add **Gmail OAuth2** credentials (scope to read emails + attachments).
2. Add **Google Sheets OAuth2** credentials.
3. Add **Google Drive OAuth2** credentials (same Google account as Drive/Sheets is recommended).
4. Add **Google Gemini / PaLM API** credentials for the LangChain Gemini node.
5. Add **QuickBooks Online OAuth2** credentials (company connected; ensure Items/Customers permissions).

2) **Email intake**
1. Add node **Gmail Trigger**:
   - Query: `has:attachment (subject:"Casper timesheet" OR subject:"Timesheets")`
   - Enable “Download Attachments”.
   - Poll every minute.
2. Add **Code** node “Split Binary Attachments” with JS to emit one item per binary attachment (loop over `items[0].binary` keys).
3. Add **Split In Batches** node “Loop: Process Each Attachment”.
4. Connect: Gmail Trigger → Split Binary Attachments → Loop: Process Each Attachment.
5. Add **Wait** node (e.g., 3 seconds) and connect Loop (batch output) → Wait.

3) **OCR**
1. Add **HTTP Request** node “Extract Text”:
   - Method POST
   - URL `https://universal-file-to-text-extractor.vercel.app/extract`
   - Content-Type: multipart/form-data
   - Body params:
     - `mode=single`
     - `output_type=text`
     - `files` = binary property name expression: `{{ Object.keys($binary)[0] }}`
2. Connect: Wait → Extract Text.

4) **AI parsing**
1. Add **Google Gemini Chat Model** node:
   - Temperature 0.2
2. Add **AI Agent** node:
   - Prompt type “define”
   - Paste the extraction prompt (billable-only, split weeks, JSON only)
   - Append OCR text variable: `{{ $json.data[0].text }}`
   - Set Gemini node as the agent’s language model input.
3. Connect: Extract Text → AI Agent.

5) **Month normalization**
1. Add **Set** node “Prepare Month array”:
   - Create field `months` as an **array** (recommended fix): `{{ [ $json.output.parseJson().month1, $json.output.parseJson().month2 ].filter(Boolean) }}`
   - Create field `Target month1`: `{{ $json.output.parseJson().month1['Week Starting Date'].split("/")[0] }}`
2. Add **Split Out** node “Split Each Month” on field `months`.
3. Connect: AI Agent → Prepare Month array → Split Each Month.

6) **Filter and route first month**
1. Add **IF** node “If1”:
   - Conditions: Total Hours != 0 AND month not empty.
2. Add **IF** node “If: First Time This Month?”:
   - Compare `Week Starting Date` month number to `Target month1`.
3. Add **Wait** node “Wait1” (2 seconds).
4. Wire:
   - Split Each Month → If1
   - If1 true → If: First Time This Month?
   - If: First Time This Month? true → Map Timesheet Fields
   - If: First Time This Month? false → Wait1 → Map Timesheet Fields

7) **Map fields + compute sheet naming and month-end**
1. Add **Set** node “Map Timesheet Fields” mapping:
   - Employee Name, Client Name from AI Agent parsed JSON
   - Month/Year/Total Hours/Week dates from current month item
2. Add **Set** node “Create Sheet Name + Invoice Date”:
   - Sheet Name = `EmployeeNoSpaces_MonthNoSpaces_CurrentYear`
   - last date of current month = compute from Month + Year (Luxon/JS expression)
3. Connect: Map Timesheet Fields → Create Sheet Name + Invoice Date.

8) **Lookup PO/customer rules**
1. Add **Google Sheets** node “Get Customer Info From PO Sheet”:
   - Document: your master sheet
   - Sheet tab: “Sheet1” (or your actual)
   - Filter: Email column equals `{{ $('Gmail Trigger').item.json.from.value[0].address }}`
2. Connect: Create Sheet Name + Invoice Date → Get Customer Info From PO Sheet.

9) **Drive folder hierarchy and sheet discovery**
1. Add **Google Drive** node “Search: Client Invoices Folder” query `01-ClientInvoices`.
2. Add **Google Drive** node “Search: Employee Name Folder” inside that folder, query employee name.
3. Add **IF** “Check Employee Name Folder” to create employee folder if missing.
4. Add folder creation nodes:
   - Create Employee Name Folder (under `01-ClientInvoices`)
   - Create Folder Name (from PO sheet “Folder Name”)
   - Create Year Folder (year)
5. Add year folder search:
   - Search inside employee folder → search folder name → search year folder
   - IF “Check if Year Folder Exists” → create year folder if not found
6. Add **Google Drive** node “Google Drive” to search for spreadsheet named `Sheet Name` inside year folder.
7. Add **IF** “If- File is Exist” to branch:
   - Exists → duplicate check
   - Missing → create new spreadsheet

10) **If missing: create/move/init spreadsheet**
1. Add **Google Sheets** “Google Sheets: Create Sheet” (resource: spreadsheet) title = Sheet Name; initial tab “Sheet1”.
2. Add **Google Drive** “Move Sheet to Invoice Folder” to move spreadsheet into year folder.
3. Add **Set** “Prepare Default Invoice Row” (empty schema object).
4. Add **Google Sheets** “Google Sheets2” append to the new sheet (Sheet1) to seed headers/shape if needed.
5. Add **Set** “Set: Spreadsheet (ID & Name)” to store new spreadsheet id for later append.

11) **If exists: duplicate protection**
1. Add **Google Sheets** “Check for Duplicate Entry”:
   - DocumentId = found spreadsheet id
   - Filter column `Decription` equals `Week ending date {{ Week Ending Date }}`
2. Add **IF** “Skip If Duplicate Found” based on existence of returned `Decription`.

12) **Append invoice row**
1. Add **Google Sheets** “Sheets: Append Row1” to append invoice data:
   - Invoice Date = last date of current month formatted `MM-dd-yyyy`
   - Due Date = invoice date + PO sheet “Due Date Calculation” days
   - Quantity = Total Hours
   - Item Name/PO/Account from PO sheet
2. Ensure DocumentId uses either the newly created spreadsheet id or the existing one.
3. Connect append node back to **Loop: Process Each Attachment** to continue with next attachment.

13) **QuickBooks sync path (manual)**
1. Add **Manual Trigger**.
2. Add **Google Sheets** read node “Get row(s) in sheet” pointing to the invoice sheet where rows exist.
3. Add **QuickBooks** node “Find Customer” using DisplayName from sheet.
4. Add **IF** “If Customer Exists?” (recommended: base it on QuickBooks search results, not sheet account number).
5. Add **QuickBooks Create Customer** if missing; then **QuickBooks Create Invoice**.
6. Critical fixes you must apply to match real QuickBooks behavior:
   - Set `CustomerRef` dynamically from found/created customer id, not hard-coded `66`.
   - Compute invoice `Amount` = hours * unit price (or use QuickBooks rate/item).
   - Ensure custom field DefinitionIds match QuickBooks custom field IDs (numeric), not labels.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OCR API endpoint used for attachment-to-text | https://universal-file-to-text-extractor.vercel.app/extract |
| Customer PO master sheet reference (cached URL shown in node) | https://docs.google.com/spreadsheets/d/11iUiyjpjaz6pNtSSncU3hwPvBzI5y2Gp8Tvuamjkd98/edit#gid=0 |
| Root Drive folder required | Create `01-ClientInvoices` in Google Drive (“My Drive”) |
| Workflow “How it works” and setup checklist | Included in the workflow sticky note “How it works” (connect Drive/Sheets/Gmail/QuickBooks/Gemini; update master sheet ID; verify OCR URL; test duplicate logic) |

