Track invoice spending vs budget from Google Drive with GPT-4o and Telegram alerts

https://n8nworkflows.xyz/workflows/track-invoice-spending-vs-budget-from-google-drive-with-gpt-4o-and-telegram-alerts-13115


# Track invoice spending vs budget from Google Drive with GPT-4o and Telegram alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Track invoice spending vs budget from Google Drive with GPT-4o and Telegram alerts

## 1. Workflow Overview

This workflow automates invoice intake from a Google Drive folder, performs OCR extraction, uses GPT‑4o to classify/extract invoice fields and category, stores invoice records and rolling summaries in Ainoflow JSON storage, and sends Telegram notifications (success, duplicates, errors, and budget alerts). It also includes a Telegram “budget management agent” to set/list/remove budgets, plus scheduled weekly and monthly summary reports, and a manual “reset” function to wipe stored invoice data.

### 1.1 Scheduled Invoice Intake & Validation (Google Drive + App Settings)
- Runs on a schedule.
- Loads bot/app settings from Ainoflow to ensure the bot was initialized via `/start`.
- Fetches PDFs from a configured Google Drive “inbox” folder and loops through them.
- Skips files already marked `[REVIEW] -` or `[DUPLICATE] -`.

### 1.2 Document Content Extraction (Download → OCR → Hash)
- Downloads each file from Drive.
- Sends the binary file to Ainoflow OCR conversion and extracts plain text.
- Computes SHA256 hash of the file for duplicate detection.

### 1.3 AI Invoice Classification & Data Extraction (GPT‑4o)
- Loads budget configuration for context.
- GPT‑4o agent determines if the document is a single invoice and extracts structured fields (vendor, amount, date, category, etc.).
- Invalid/non-invoice documents are routed to manual review.

### 1.4 Storage, Deduplication, Summary Update, Alerts, and File Organization
- Uses the file hash to check for duplicates in Ainoflow storage.
- Saves invoice record if new; otherwise marks as duplicate.
- Updates monthly summary totals per category and overall.
- Sends budget alerts when threshold is exceeded.
- Renames the file using extracted invoice metadata and moves it to a monthly subfolder.

### 1.5 Telegram Budget Management Agent (Chat-triggered)
- `/start` registers the bot owner chat_id and sets allowed categories.
- Subsequent messages are authorized only for the registered chat_id.
- GPT‑4o agent + MCP storage tool reads/writes budgets and replies in Telegram.

### 1.6 Weekly & Monthly Scheduled Reports
- Weekly (Fridays): pulls current month summary and sends a formatted Telegram report.
- Monthly (10th): fetches previous month report text via an internal sub-workflow call (same workflow), then sends it.

### 1.7 Manual Data Reset (Dangerous)
- Manual trigger requests Telegram approval (double approval).
- Deletes all Ainoflow storage categories starting with `invoice-`.
- Sends a “reset completed, run /start” message.

---

## 2. Block-by-Block Analysis

### Block A — Documentation / Operator Notes
**Overview:** Provides embedded operational instructions and setup requirements (credentials, folder structure, commands).  
**Nodes involved:** `README`, multiple sticky notes (`Budget Agent`, `Weekly Summary`, `Monthly Summary`, `Budget Agent1`, `Budget Agent2`, etc.).  
**Node details (representative):**
- **README (Sticky Note)**
  - **Type/role:** Sticky note for setup + usage.
  - **Key content:** Credential links, folder structure, command list, thresholds, reset warnings, support link.
  - **Failure modes:** None (non-executable), but operationally critical (missing `/start` prevents most functions).

(Sticky notes are fully enumerated in the Summary Table.)

---

### Block B — Scheduled Invoice Intake & Registration Guard
**Overview:** Runs on a schedule, loads configuration (chat_id and categories) from Ainoflow, and proceeds only if the bot has been registered.  
**Nodes involved:** `HourlyTrigger`, `GetAppSettings`, `IfRegistrationExists`, `GetFiles`, `LoopInput`, `WorkflowConfig`, `IfNotForReviewOrDuplicate`.

#### Node: HourlyTrigger
- **Type/role:** Schedule Trigger; entry point for invoice processing.
- **Config:** Cron `0 8-21 * * *` (hourly between 08:00 and 21:00).
- **Output:** Triggers `GetAppSettings`.
- **Failure modes:** n8n schedule misconfiguration; timezone surprises.

#### Node: GetAppSettings
- **Type/role:** HTTP Request to Ainoflow JSON storage.
- **Config:** GET `https://api.ainoflow.io/api/v1/storage/json/invoice-config/settings`
  - Auth: HTTP Bearer (`Ainoflow`).
  - `onError: continueRegularOutput`, `alwaysOutputData: false` (so errors may stop output in some cases).
- **Output:** To `IfRegistrationExists`.
- **Failure modes:** 401/403 invalid token; API downtime; non-JSON response.

#### Node: IfRegistrationExists
- **Type/role:** IF gate to ensure a `chat_id` exists in settings.
- **Condition:** “exists” check on `{{$json.chat_id}}`.
- **True path:** Continue to `GetFiles`.
- **False path:** Stops (no invoice processing without initial `/start`).
- **Edge cases:** If settings object is empty `{}` or missing `chat_id`, workflow quietly does nothing.

#### Node: GetFiles
- **Type/role:** Google Drive list/search files in the inbox folder.
- **Config:** Search `*.pdf` in folderId `1YuASChwxTDENnfcwfvrA_WFUJxmu2CNo`. Fields: `*`.
- **Output:** `LoopInput`.
- **Failure modes:** OAuth token expired; folder not accessible; query misses non-PDF invoices.

#### Node: LoopInput (Split in Batches)
- **Type/role:** Iterates files one-by-one.
- **Connections:** First output is unused; second output loops back to itself and to `WorkflowConfig`.
- **Edge cases:** If many files, long runs; consider batch size / execution time limits.

#### Node: WorkflowConfig (Set)
- **Type/role:** Normalizes configuration and propagates settings to downstream nodes.
- **Key fields:**
  - `chat_id` from `GetAppSettings.item.json.chat_id`
  - `input_folder_id` = `LoopInput.item.json.parents[0]`
  - `alert_threshold` = 80 (percent)
  - `allowed_categories` from settings or `'Other'`
  - `default_currency` = `EUR`
  - `review_prefix` = `[REVIEW] - `
  - `duplicate_prefix` = `[DUPLICATE] - `
- **Output:** `IfNotForReviewOrDuplicate`.
- **Edge cases:** `parents[0]` absent if file metadata incomplete.

#### Node: IfNotForReviewOrDuplicate
- **Type/role:** Prevents re-processing.
- **Conditions:** filename does **not** start with review prefix and does **not** start with duplicate prefix.
- **True path:** `DownloadInvoice`.
- **False path:** skipped.
- **Edge cases:** If user renames files manually, reprocessing can happen.

---

### Block C — Download, OCR, Hashing, and OCR Validation
**Overview:** Downloads the invoice, extracts text via Ainoflow OCR, hashes the file for deduplication, and ensures OCR text exists.  
**Nodes involved:** `DownloadInvoice`, `OcrExtract`, `OcrOutput`, `Crypto`, `InputFile`, `Merge`, `IfOcrSuccess`.

#### Node: DownloadInvoice
- **Type/role:** Google Drive download.
- **Config:** `fileId = {{$('LoopInput').item.json.id}}`
- **Output:** Sends binary to `OcrExtract` and also to `Crypto`.
- **Failure modes:** Drive permission issues; large file download timeouts.

#### Node: OcrExtract
- **Type/role:** HTTP multipart upload to Ainoflow converter.
- **Config:** POST `https://api.ainoflow.io/api/v1/convert/submit-file`
  - multipart fields: `languages=en,de,pt`, `outputs=text`, `response=direct`, `file` from binary `data`
  - `onError: continueRegularOutput` (OCR failures won’t hard-stop; they route via empty output)
- **Output:** `OcrOutput`.
- **Failure modes:** OCR service unavailable; unsupported PDF; extraction returns empty.

#### Node: OcrOutput (Set)
- **Type/role:** Extracts OCR text into `invoice_ocr_text`.
- **Expression:** `{{$('OcrExtract').item.json.content?.[0]?.text}}`
- **Output:** Into `Merge` input 0.
- **Edge cases:** `content` missing → `invoice_ocr_text` undefined.

#### Node: Crypto
- **Type/role:** Compute SHA256 on binary file.
- **Config:** SHA256; `binaryData: true`; outputs to json field `input_file_hash`.
- **Output:** `InputFile`.
- **Failure modes:** Missing binary data; memory constraints for large files.

#### Node: InputFile (Set)
- **Type/role:** Consolidates file identifiers and metadata.
- **Fields:**
  - `input_file_id` from Drive id
  - `input_file_hash` from Crypto
  - `input_file_filename` from downloaded binary filename fallback `invoice.pdf`
  - `input_file_extension` derived from filename
- **Output:** To `Merge` input 1.
- **Edge cases:** filename missing or has no extension.

#### Node: Merge
- **Type/role:** Combine OCR output + file metadata (`combineAll`).
- **Output:** `IfOcrSuccess`.
- **Failure modes:** If one branch missing due to errors, downstream may receive partial data.

#### Node: IfOcrSuccess
- **Type/role:** Gate based on presence of OCR text.
- **Condition:** `invoice_ocr_text` exists.
- **True path:** `GetBudgets` (continue AI extraction).
- **False path:** `RenameToReview` (mark file for manual review and alert).

---

### Block D — Load Budget Config Context
**Overview:** Loads budgets from Ainoflow storage so the categorizer can consider current budgets and generate warnings/insights.  
**Nodes involved:** `GetBudgets`, `ProcessingInput`.

#### Node: GetBudgets
- **Type/role:** HTTP GET budgets JSON.
- **Config:** GET `.../storage/json/invoice-config/budgets?allowEmpty=true`
  - `alwaysOutputData: true`, `onError: continueRegularOutput`
- **Output:** `ProcessingInput`.
- **Edge cases:** No budgets set → empty object; downstream builds “No budgets set”.

#### Node: ProcessingInput (Set)
- **Type/role:** Builds AI input payload.
- **Key fields:**
  - `ocr_text` from OCR content
  - `allowed_categories` from `WorkflowConfig`
  - `budgets_context` formatted list of budgets or “No budgets set”
  - carries folder/file/hash metadata
- **Output:** To `AICategorizer`.
- **Edge cases:** Allowed categories string splitting inconsistencies later (see CalcSummary uses `split(', ')`).

---

### Block E — AI Categorization & Structured Parsing
**Overview:** GPT‑4o agent checks if the file is a valid single invoice and extracts structured invoice data; invalid content is routed to review.  
**Nodes involved:** `Gpt4oCategorizer`, `StructuredOutputParser`, `AICategorizer`, `Think1`, `IfInvalidContent`.

#### Node: Gpt4oCategorizer
- **Type/role:** OpenRouter chat model for the invoice agent and the output parser.
- **Config:** Model `openai/gpt-4o`.
- **Connections:** Provides `ai_languageModel` to `AICategorizer` and `StructuredOutputParser`.
- **Failure modes:** OpenRouter auth/rate limit; model unavailable; cost spikes.

#### Node: StructuredOutputParser
- **Type/role:** Enforces JSON schema with autofix.
- **Config:** Manual JSON schema:
  - If `is_invoice=true`: requires vendor, vendor_normalized, amount, currency (3 letters), date (format date), invoice_number, category, confidence, etc.
  - If `is_invoice=false`: requires document_type and skip_reason.
  - `autoFix: true`, `additionalProperties: false`.
- **Connections:** Receives `ai_languageModel` from `Gpt4oCategorizer`; outputs parser to `AICategorizer` (`ai_outputParser`).
- **Edge cases:** Autofix may “repair” ambiguous outputs incorrectly; strict schema can drop fields.

#### Node: AICategorizer (LangChain Agent)
- **Type/role:** Main invoice analysis agent.
- **Input text:** Includes filename + OCR content.
- **System message:** Detailed invoice-vs-non-invoice rules; extraction tasks; category list; anomaly detection; output JSON schema.
- **Output:** `IfInvalidContent`.
- **Failure modes:** Hallucinated fields; wrong date format; misclassification of multi-invoice PDFs; token limits for long OCR text.

#### Node: Think1 (Tool)
- **Type/role:** Optional reasoning tool connected to AICategorizer as `ai_tool`.
- **Config:** Default.
- **Edge cases:** None; depends on agent usage.

#### Node: IfInvalidContent
- **Type/role:** Route invalid documents.
- **Condition:** `{{$json.output.is_invoice}}` is boolean false.
- **True path (invalid/non-invoice):** `RenameToReview`
- **False path (valid invoice):** `StorageInput`
- **Edge cases:** If agent output missing `.output.is_invoice`, strict validation could fail earlier; ensure parser always returns.

---

### Block F — Deduplication, Save Invoice Record, Update Monthly Summary, Budget Alerts
**Overview:** Prevents duplicates via file hash, writes invoice record, updates monthly summary totals, and triggers budget alerts when thresholds are met.  
**Nodes involved:** `StorageInput`, `CheckInvoiceExists`, `IfNewInvoice`, `SaveInvoice`, `GetSummary`, `CalcSummary`, `SummaryInput`, `SaveSummary`, `IfNeedsAlert`, `AlertMessage`, `RenameAsDuplicate`, `DuplicateLog`.

#### Node: StorageInput (Set)
- **Type/role:** Builds storage namespace and keys.
- **Fields:**
  - `storage_category = invoice-data-YYYY-MM` (from invoice date)
  - `invoice_month = YYYY-MM`
  - file hash/id/name/ext passthrough
- **Output:** `CheckInvoiceExists`.
- **Edge cases:** If date missing/invalid, substring fails.

#### Node: CheckInvoiceExists (HTTP Request)
- **Type/role:** Checks if a record exists for this file hash.
- **Config:** GET `.../storage/json/{{storage_category}}/{{input_file_hash}}?allowEmpty=true`
  - `alwaysOutputData: true` (empty object if not found)
- **Output:** `IfNewInvoice`.
- **Failure modes:** Ainoflow storage errors; network timeouts.

#### Node: IfNewInvoice (IF)
- **Type/role:** Determines whether Ainoflow returned empty.
- **Condition:** `Object.keys($json).length == 0`
- **True path:** `SaveInvoice`
- **False path:** `RenameAsDuplicate`
- **Edge cases:** If API returns `{error:...}` it is non-empty and treated as “duplicate” incorrectly.

#### Node: SaveInvoice (HTTP PUT)
- **Type/role:** Writes invoice record to `invoice-data-YYYY-MM/<hash>`.
- **Body:** vendor, vendor_normalized, amount, currency, date, invoice_number, category, file_id, processed_at.
- **Dependencies:** Reads from `AICategorizer.output` and `StorageInput`.
- **Output:** `GetSummary`.
- **Failure modes:** PUT denied; schema mismatch; missing fields due to AI errors.

#### Node: GetSummary (HTTP GET)
- **Type/role:** Loads monthly summary totals `invoice-summary-YYYY-MM/totals`.
- **Config:** GET with `allowEmpty=true`, `alwaysOutputData: true`.
- **Output:** `CalcSummary`.
- **Edge cases:** Summary absent → `{}` handled by CalcSummary initializer.

#### Node: CalcSummary (Code)
- **Type/role:** Mutates/creates summary object and computes per-category %.
- **Key logic:**
  - `allowedCategories = WorkflowConfig.allowed_categories.split(', ')`
  - category fallback to `Other` if not in allowed list
  - updates `summary.categories[category].spent`, `.budget`, `.pct`
  - computes `summary.total_spent`, `summary.invoice_count`, `summary.total_budget`
  - appends insights (last 10) and anomalies
- **Output:** `SummaryInput`.
- **Edge cases:**
  - Category split requires comma+space; if stored categories are comma-only, membership check fails causing many “Other”.
  - If budgets contain strings, numeric math may break.

#### Node: SummaryInput (Set)
- **Type/role:** Prepares downstream payload:
  - `summary`, `summary_category`
  - `needs_alert` = invoice.budget_warning OR (budget>0 and pct>=80)
  - `file_info` includes `new_filename` and `month_folder`
- **Output:** Two outputs: `SaveSummary` and `RenameFile`.
- **Edge cases:** Filename may include forbidden characters (`/`, `:`) from vendor/invoice_number.

#### Node: SaveSummary (HTTP PUT)
- **Type/role:** Saves updated summary to `invoice-summary-YYYY-MM/totals`.
- **Output:** `IfNeedsAlert`.
- **Failure modes:** PUT errors; overwriting concurrency if multiple invoices processed simultaneously.

#### Node: IfNeedsAlert (IF)
- **Type/role:** Budget alert gate.
- **Condition:** `SummaryInput.needs_alert == true`
- **True path:** `AlertMessage`.
- **Edge cases:** Alert threshold hardcoded in SummaryInput logic at 80; WorkflowConfig.alert_threshold is not referenced here (potential config drift).

#### Node: AlertMessage (Telegram)
- **Type/role:** Sends “Budget Alert” message to owner chat.
- **Key expressions:** Uses `SummaryInput` data; Drive file view link.
- **Failure modes:** Telegram auth errors; Markdown formatting issues due to special chars.

#### Node: RenameAsDuplicate (Google Drive update)
- **Type/role:** Renames file with duplicate prefix.
- **Output:** `DuplicateLog`.
- **Failure modes:** Permission errors; name too long.

#### Node: DuplicateLog (Telegram)
- **Type/role:** Notifies user about duplicate.
- **Edge cases:** Mentions hash prefix; might confuse users vs. invoice name.

---

### Block G — Error / Manual Review Path (OCR failure or invalid content)
**Overview:** Marks failed files with `[REVIEW] -` and sends a Telegram error notification.  
**Nodes involved:** `RenameToReview`, `ErrorAlert`.

#### Node: RenameToReview (Google Drive update)
- **Type/role:** Renames input file with review prefix.
- **Expression:** `[REVIEW] - {{original filename}}`
- **Output:** `ErrorAlert`.
- **Failure modes:** Drive permission; collisions; repeated prefixing.

#### Node: ErrorAlert (Telegram)
- **Type/role:** Sends failure message with skip reason if invalid content.
- **Depends on:** `IfInvalidContent.item.json.output.skip_reason` (may be undefined if OCR failed rather than AI skip).
- **Edge cases:** If OCR failed, `IfInvalidContent` path might not have executed; the expression can be undefined (Telegram sends blank reason).

---

### Block H — Rename, Create/Find Month Folder, Move, Success Notification
**Overview:** Applies final canonical filename, ensures month folder exists, moves the file, and logs success.  
**Nodes involved:** `RenameFile`, `SearchMonthFolder`, `IfFolderExists`, `CreateMonthFolder`, `ResultFolder`, `MoveToMonth`, `SuccessLog`.

#### Node: RenameFile (Google Drive update)
- **Type/role:** Renames file to `[YYYY-MM-DD] - Vendor (INV?, amount CUR).ext`
- **Source:** `SummaryInput.file_info.new_filename`
- **Output:** `SearchMonthFolder`.
- **Edge cases:** Illegal filename chars; very long vendor names.

#### Node: SearchMonthFolder (Google Drive search)
- **Type/role:** Searches for month folder under inbox folder.
- **Query:** `month_folder` (e.g., `2026-01`)
- **alwaysOutputData: true** so empty output still passes.
- **Output:** `IfFolderExists`.

#### Node: IfFolderExists (IF)
- **Condition:** `$json.id` exists.
- **True path:** `ResultFolder` (id is the existing folder)
- **False path:** `CreateMonthFolder` then `ResultFolder`
- **Edge cases:** Search can return unexpected folder if same name exists elsewhere; ensure folderId filter is correct (it is).

#### Node: CreateMonthFolder (Google Drive folder create)
- **Type/role:** Creates month folder inside input folder.
- **Output:** `ResultFolder`.

#### Node: ResultFolder (Set)
- **Type/role:** Normalizes target folder id into `target_folder_id = {{$json.id}}`.
- **Output:** `MoveToMonth`.

#### Node: MoveToMonth (Google Drive move)
- **Type/role:** Moves file to month folder.
- **Output:** `SuccessLog`.
- **Failure modes:** File already in folder; permissions; shared drive constraints.

#### Node: SuccessLog (Telegram)
- **Type/role:** Sends success message with vendor/amount/category and final path.
- **Markdown enabled.**
- **Edge cases:** Vendor/category special characters break Markdown.

---

### Block I — Telegram Budget Management Agent (Owner-locked)
**Overview:** Telegram trigger receives messages; first `/start` registers chat_id and defaults; later messages require authorization and are handled by a GPT‑4o agent that reads/writes Ainoflow budgets via MCP.  
**Nodes involved:** `BudgetTrigger`, `GetAppSettings1`, `IfFirstRun`, `FirstRunOnlyStart`, `IfAuthorized`, `NotAuthorizedMessage`, `IfStartMessage`, `SetDefaults`, `WelcomeMessage`, `SaveAppSettings`, `BudgetInput`, `Gpt4oBudgetAgent`, `BudgetMemory`, `JsonStorageMcp`, `Calculator`, `Think`, `BudgetAgent`, `BudgetReply`, plus tool node `MonthlySummary` (workflow tool).

#### Node: BudgetTrigger (Telegram Trigger)
- **Type/role:** Entry point for chat commands (updates: `message`).
- **Output:** `GetAppSettings1`.
- **Failure modes:** Telegram webhook not set; bot token invalid.

#### Node: GetAppSettings1 (HTTP GET)
- **Type/role:** Loads settings to determine first run and allowed categories.
- **alwaysOutputData: true**.

#### Node: IfFirstRun (IF)
- **Condition:** `GetAppSettings1.chat_id` “notExists”.
- **True path:** `FirstRunOnlyStart` (must start with /start)
- **False path:** `IfAuthorized`

#### Node: FirstRunOnlyStart (IF)
- **Condition:** message text equals `/start`.
- **True:** `IfStartMessage` (which routes start vs other handling)
- **False:** stops (user must /start first).

#### Node: IfAuthorized (IF)
- **Condition:** incoming chat id equals stored `chat_id`.
- **True:** `IfStartMessage`
- **False:** `NotAuthorizedMessage`
- **Edge cases:** If settings missing chat_id (corruption), authorization check compares to null.

#### Node: NotAuthorizedMessage (Telegram)
- **Role:** Tells other users the bot is locked to owner.

#### Node: IfStartMessage (IF)
- **Condition:** message equals `/start`.
- **True:** `SetDefaults` (register/init flow)
- **False:** `BudgetInput` (normal agent message)

#### Node: SetDefaults (Set)
- **Role:** Defines default `allowed_categories` string used at first registration.

#### Node: WelcomeMessage (Telegram)
- **Role:** Sends command list + categories list (builds bullets by splitting on commas).
- **Output:** `SaveAppSettings`.

#### Node: SaveAppSettings (HTTP PUT)
- **Role:** Writes `invoice-config/settings`:
  - `chat_id`, `registered_at`, `allowed_categories`
- **Failure modes:** Without this, scheduled invoice processing will never run (registration guard).

#### Node: BudgetInput (Set)
- **Role:** Prepares agent inputs:
  - `text`, `chat_id`, `allowed_categories` from settings

#### Node: Gpt4oBudgetAgent (OpenRouter model)
- **Role:** Language model for `BudgetAgent`.
- **Model:** `openai/gpt-4o`.

#### Node: BudgetMemory (Memory Buffer Window)
- **Role:** Persists conversation context per chat_id.
- **Session key:** `budget_agent_{{chat_id}}`

#### Node: JsonStorageMcp (MCP Client Tool)
- **Role:** Tool access to MCP endpoint `https://mcp.ainoflow.io/mcp/v1/storage/json`
- **Auth:** bearerAuth via Ainoflow bearer token.
- **Exposed ops (per system prompt):** get/upsert JSON.

#### Node: Calculator (Tool)
- **Role:** For totals/percent calculations in agent responses.

#### Node: Think (Tool)
- **Role:** Optional internal reasoning tool for agent planning.

#### Node: BudgetAgent (LangChain Agent)
- **Role:** Parses natural language commands (“Set budget…”, “Show budgets”, “Budget status”, “Remove…”) and uses MCP tool calls.
- **System message:** Defines storage structure `invoice-config/budgets`, validates categories, default currency EUR, and formatting rules.
- **Output:** `BudgetReply`.
- **Edge cases:** If user uses category spelling not matching allowed list; ambiguous amounts/currencies.

#### Node: BudgetReply (Telegram)
- **Role:** Sends agent output back to user.

#### Node: MonthlySummary (ToolWorkflow)
- **Role:** Exposes “Get monthly summary” as a callable tool to the BudgetAgent.
- **WorkflowId:** Self reference (`{{$workflow.id}}`).
- **Inputs:** `year` and `month` via `$fromAI`.
- **Edge cases:** Tool depends on internal sub-workflow entry `GetMonthlySummaryTrigger` being correct.

---

### Block J — Weekly Summary (Scheduled)
**Overview:** Every Friday, fetches current month summary, computes presentation fields, and sends Telegram report.  
**Nodes involved:** `WeeklyTrigger`, `GetAppSettings2`, `GetWeeklySummary`, `IfHasData`, `CalcWeeklySummary`, `BuildWeeklyMessage`, `WeeklySummaryMessage`.

#### Node: WeeklyTrigger
- **Type/role:** Schedule Trigger (weekly).
- **Config:** `weeks`, day=5 (Friday), hour=17.

#### Node: GetAppSettings2
- **Type/role:** Loads settings (chat_id).
- **onError:** continueErrorOutput.

#### Node: GetWeeklySummary
- **Type/role:** GET summary for `{{$now.format('yyyy-MM')}}`.
- **alwaysOutputData: true** with `allowEmpty=true`.

#### Node: IfHasData
- **Condition:** `GetWeeklySummary.invoice_count > 0`
- **True:** proceed; False: stop.

#### Node: CalcWeeklySummary (Code)
- **Role:** Calculates `total_pct`, maps categories to list, limits insights/anomalies.

#### Node: BuildWeeklyMessage (Set)
- **Role:** Formats Markdown message (traffic-light icons based on pct thresholds).

#### Node: WeeklySummaryMessage (Telegram)
- **Role:** Sends weekly message to `GetAppSettings2.chat_id`.

---

### Block K — Monthly Summary (Scheduled + Internal Sub-workflow)
**Overview:** On the 10th, sends the previous month report. Internally, the workflow can be called with `year` and `month` inputs to produce formatted monthly report text.  
**Nodes involved:** `MonthlyTrigger`, `GetAppConfig2`, `GetMonthlySummaryText`, `MonthlySummaryMessage`, plus the internal callable path: `GetMonthlySummaryTrigger`, `WorkflowInput`, `GetMonthlySummary`, `IfHasMonthlyData1`, `CalcMonthlySummary`, `BuildMonthlyMessage`, `ErrorOutput`.

#### Node: MonthlyTrigger
- **Type/role:** Schedule Trigger (monthly).
- **Config:** day 10, hour 9.

#### Node: GetAppConfig2
- **Type/role:** Loads settings for chat_id.
- **onError:** continueErrorOutput.

#### Node: GetMonthlySummaryText (Execute Workflow)
- **Type/role:** Calls same workflow by id (self-call) with inputs set to previous month:
  - `year = now - 1 month`
  - `month = now - 1 month`
  - `convertFieldsToString: true`
- **Output:** `MonthlySummaryMessage`.
- **Failure modes:** Self-call recursion risk is limited because it calls the ExecuteWorkflowTrigger entry path, not the same schedule trigger; still, incorrect wiring can cause loops.

#### Node: MonthlySummaryMessage (Telegram)
- **Role:** Sends `$json.message ?? $json.error` to settings chat_id.

#### Node: GetMonthlySummaryTrigger (Execute Workflow Trigger)
- **Role:** Defines the sub-workflow entry point with inputs `year` and `month`.
- **Output:** `WorkflowInput`.

#### Node: WorkflowInput (Set)
- **Role:** Normalizes inputs (defaults to current year/month if absent).

#### Node: GetMonthlySummary (HTTP GET)
- **Role:** Fetch summary for given `year/month` as `invoice-summary-YYYY-MM/totals`.
- **alwaysOutputData: true** with `allowEmpty=true`.

#### Node: IfHasMonthlyData1 (IF)
- **Condition:** `GetMonthlySummary.invoice_count > 0`
- **True:** `CalcMonthlySummary`
- **False:** `ErrorOutput`

#### Node: CalcMonthlySummary (Code)
- **Role:** Calculates totals, sorts categories by spent, builds progress bars and flags over/near budget.

#### Node: BuildMonthlyMessage (Set)
- **Role:** Formats Markdown monthly report and passes along period/total/count.

#### Node: ErrorOutput (Set)
- **Role:** Returns error string “Monthly summary does not exist for selected month (YYYY-MM)”.

---

### Block L — Manual Reset / Cleanup (Deletes Ainoflow invoice data)
**Overview:** Manual trigger requests Telegram approval and deletes all Ainoflow JSON categories beginning with `invoice-`.  
**Nodes involved:** `ManualResetTrigger`, `GetAppSettings3`, `ApproveRemove`, `IfApproved`, `ForEachCategory`, `If`, `ForEachCategoryItem`, `DeleteItem`, `RestartMessage`.

#### Node: ManualResetTrigger
- **Type/role:** Manual trigger entry point.

#### Node: GetAppSettings3
- **Role:** Loads settings to get chat_id for approval prompt.

#### Node: ApproveRemove (Telegram sendAndWait)
- **Role:** Sends approval request to chat_id and waits (double approval).
- **Failure modes:** Workflow paused awaiting approval; Telegram not reachable.

#### Node: IfApproved
- **Condition:** `$json.data.approved == true`
- **True:** Starts deletion loop and then sends restart message.
- **False:** stops.

#### Node: ForEachCategory (HTTP)
- **Role:** GET list of storage categories at `https://api.ainoflow.io/api/v1/storage/json`
- **onError:** continueErrorOutput.

#### Node: If (IF)
- **Role:** Filters only categories whose name starts with `invoice-`.

#### Node: ForEachCategoryItem (HTTP)
- **Role:** Lists keys/items in a given category `.../storage/json/{{category}}`.

#### Node: DeleteItem (HTTP DELETE)
- **Role:** Deletes each item `.../storage/json/{{category}}/{{key}}`.

#### Node: RestartMessage (Telegram)
- **Role:** Informs user reset completed; instructs `/start`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| README | stickyNote | Embedded setup/usage notes | — | — | # Invoice Budget Tracker (setup steps, credentials, commands, reset warning, link: https://ainovasystems.com/) |
| Budget Agent | stickyNote | Section header (budget management) | — | — | ## 2. Budget Management AI Agent Telegram-based budget configuration via AI Agent + MCP Storage |
| Weekly Summary | stickyNote | Section header (weekly) | — | — | ## 3. Weekly Budget Check Weekly check on current month's budget progress (every Friday) |
| Monthly Summary | stickyNote | Section header (monthly) | — | — | ## 4. Monthly Summary Scheduled report generation on 10th of each month |
| Sticky Note12 | stickyNote (disabled) | Cleanup warning | — | — | ## Cleanup / Reset This is **MANUAL** trigger to full data cleanup. **Warning - this will DELETE ALL DATA** stored in your **Ainoflow** account |
| Sticky Note | stickyNote | Visual label (scheduled trigger) | — | — | ## Scheduled Trigger |
| Sticky Note1 | stickyNote | Requirement: bot config exists | — | — | ## Ensure Tool Configuration Exists Before starting processing, we should have a Telegram bot successfully configured and linked to the workflow via the `/start` message. |
| Sticky Note2 | stickyNote | Loop note | — | — | ## Loop over unprocessed files - Get all files in folder, iterate one by one - Make sure you have selected right (inbox) directory here |
| Sticky Note3 | stickyNote | Visual label (chat trigger) | — | — | ## Chat Trigger |
| Sticky Note4 | stickyNote | Bot privacy note | — | — | ## Ensure Bot Privacy On first use, the bot stores the user ID and locks access to that user. Any other user will receive an “unauthorized” message. |
| Sticky Note10 | stickyNote (disabled) | Future enhancement | — | — | ## Add processing started message Notify user about processing start (with message ... ) |
| Sticky Note6 | stickyNote (disabled) | Placeholder | — | — | ## Build Agent Input |
| Sticky Note7 | stickyNote (disabled) | Placeholder | — | — | ## Budget Agent Processing - answer to user message - ensure budget is set as expected |
| Sticky Note5 | stickyNote (disabled) | Placeholder | — | — | ## Result / Reply Message Sends reply to the user |
| Budget Agent1 | stickyNote | Section header (scheduled document extraction) | — | — | ## 1.1 Document Processing Loop / Data Extraction (Scheduled) |
| Sticky Note8 | stickyNote | Config + file validation note | — | — | ## Get Config and Validate Input File - Document not marked for manual review - Document not marked as duplicate |
| Sticky Note9 | stickyNote | Content extraction note | — | — | ## Get Document Content - Downloads file - Extract text from file - Calculate file hash |
| Sticky Note11 | stickyNote | Budget data load note | — | — | ## Load Budget Data - Budget json loaded from config storage |
| Sticky Note14 | stickyNote | Manual review criteria note | — | — | ## Require manual review - If OCR / text extraction failed - If categorizer identified invalid content |
| Sticky Note15 | stickyNote | AI extraction note | — | — | ## Extract Invoice Data |
| Budget Agent2 | stickyNote | Section header (invoice processing loop) | — | — | ## 1.2 Document Processing Loop / Invoice Processing |
| Sticky Note16 | stickyNote (disabled) | Duplicate handling note | — | — | ## Handle Duplicate Invoice - Rename as duplicate (for manual processing) - Notify user about duplicate |
| Sticky Note17 | stickyNote | Duplicate check label | — | — | ## Duplicate Check |
| Sticky Note18 | stickyNote | Save record label | — | — | ## Save Record |
| Sticky Note19 | stickyNote | Update summary label | — | — | ## Update Summary |
| Sticky Note20 | stickyNote | Alerts label | — | — | ## Budget Alerts |
| Sticky Note21 | stickyNote | Rename doc label | — | — | ## Rename Document - Use invoice details to build new document name |
| Sticky Note22 | stickyNote | Ensure folder exists label | — | — | ## Ensure Month Folder Exists |
| Sticky Note23 | stickyNote | Move file label | — | — | ## Move Document to Month Folder |
| Sticky Note25 | stickyNote | Success notification label | — | — | ## Success Notification - Send message to user with invoice details |
| Sticky Note24 | stickyNote | Weekly trigger label | — | — | ## Weekly Trigger |
| Sticky Note26 | stickyNote | Weekly summary fetch label | — | — | ## Get Weekly Summary - Do not proceed if summary does not exist yet |
| Sticky Note27 | stickyNote (disabled) | Placeholder | — | — | ## Send Weekly Notification |
| Sticky Note28 | stickyNote | Monthly trigger label | — | — | ## Monthly Trigger |
| Sticky Note29 | stickyNote | Monthly summary fetch label | — | — | ## Get Monthly Summary - Do not proceed if summary does not exist yet |
| Sticky Note30 | stickyNote (disabled) | Placeholder | — | — | ## Send Monthly Notification |
| Sticky Note31 | stickyNote | Monthly subworkflow logic label | — | — | ## Monthly Summary Calculation Logic - Respond with summary or error message - Uses subworkflow (workflow with same id) |
| Monthly Summary1 | stickyNote | Reset section label | — | — | ## 5. Data Reset - Removes all account / project data |
| HourlyTrigger | scheduleTrigger | Scheduled entry for invoice processing | — | GetAppSettings | ## Scheduled Trigger |
| GetAppSettings | httpRequest | Load app settings (chat_id/categories) | HourlyTrigger | IfRegistrationExists | ## Ensure Tool Configuration Exists… |
| IfRegistrationExists | if | Gate: only process when chat_id exists | GetAppSettings | GetFiles | ## Ensure Tool Configuration Exists… |
| GetFiles | googleDrive | List PDFs in inbox folder | IfRegistrationExists | LoopInput | ## Loop over unprocessed files… |
| LoopInput | splitInBatches | Iterate files | GetFiles | LoopInput, WorkflowConfig | ## Loop over unprocessed files… |
| WorkflowConfig | set | Build runtime config (prefixes, chat_id, categories) | LoopInput | IfNotForReviewOrDuplicate | ## Get Config and Validate Input File… |
| IfNotForReviewOrDuplicate | if | Skip files already marked review/duplicate | WorkflowConfig | DownloadInvoice | ## Get Config and Validate Input File… |
| DownloadInvoice | googleDrive | Download file binary | IfNotForReviewOrDuplicate | OcrExtract, Crypto | ## Get Document Content… |
| OcrExtract | httpRequest | OCR conversion (multipart) | DownloadInvoice | OcrOutput | ## Get Document Content… |
| OcrOutput | set | Map OCR text to invoice_ocr_text | OcrExtract | Merge | ## Get Document Content… |
| Crypto | crypto | SHA256 hash of binary for dedupe | DownloadInvoice | InputFile | ## Get Document Content… |
| InputFile | set | Normalize file metadata (id/hash/name/ext) | Crypto | Merge | ## Get Document Content… |
| Merge | merge | Combine OCR + file metadata | OcrOutput, InputFile | IfOcrSuccess | ## Get Document Content… |
| IfOcrSuccess | if | Gate: OCR succeeded | Merge | GetBudgets, RenameToReview | ## Require manual review… |
| GetBudgets | httpRequest | Load budgets config | IfOcrSuccess | ProcessingInput | ## Load Budget Data… |
| ProcessingInput | set | Build AI prompt context | GetBudgets | AICategorizer | ## Extract Invoice Data |
| Gpt4oCategorizer | lmChatOpenRouter | GPT‑4o model provider (invoice agent) | — | AICategorizer, StructuredOutputParser (ai_languageModel) | ## Extract Invoice Data |
| StructuredOutputParser | outputParserStructured | Enforce JSON schema for AI output | Gpt4oCategorizer | AICategorizer (ai_outputParser) | ## Extract Invoice Data |
| Think1 | toolThink | Optional tool for AICategorizer | — | AICategorizer (ai_tool) | ## Extract Invoice Data |
| AICategorizer | langchain.agent | Classify & extract invoice fields | ProcessingInput | IfInvalidContent | ## Extract Invoice Data |
| IfInvalidContent | if | Route: non-invoice → review | AICategorizer | RenameToReview, StorageInput | ## Require manual review… |
| StorageInput | set | Build storage category + keys | IfInvalidContent | CheckInvoiceExists | ## Duplicate Check |
| CheckInvoiceExists | httpRequest | Dedupe check (GET by hash) | StorageInput | IfNewInvoice | ## Duplicate Check |
| IfNewInvoice | if | New invoice if empty response | CheckInvoiceExists | SaveInvoice, RenameAsDuplicate | ## Duplicate Check |
| RenameAsDuplicate | googleDrive | Prefix filename with [DUPLICATE] | IfNewInvoice | DuplicateLog | ## Duplicate Check |
| DuplicateLog | telegram | Notify duplicate | RenameAsDuplicate | — | ## Duplicate Check |
| SaveInvoice | httpRequest | PUT invoice record | IfNewInvoice | GetSummary | ## Save Record |
| GetSummary | httpRequest | GET monthly summary totals | SaveInvoice | CalcSummary | ## Update Summary |
| CalcSummary | code | Update monthly summary math | GetSummary | SummaryInput | ## Update Summary |
| SummaryInput | set | Prepare summary + file naming + alert flag | CalcSummary | SaveSummary, RenameFile | ## Update Summary |
| SaveSummary | httpRequest | PUT updated summary totals | SummaryInput | IfNeedsAlert | ## Update Summary |
| IfNeedsAlert | if | Gate: send budget alert | SaveSummary | AlertMessage | ## Budget Alerts |
| AlertMessage | telegram | Telegram “Budget Alert” | IfNeedsAlert | — | ## Budget Alerts |
| RenameFile | googleDrive | Apply canonical invoice filename | SummaryInput | SearchMonthFolder | ## Rename Document… |
| SearchMonthFolder | googleDrive | Find month subfolder | RenameFile | IfFolderExists | ## Ensure Month Folder Exists |
| IfFolderExists | if | Create folder if missing | SearchMonthFolder | ResultFolder, CreateMonthFolder | ## Ensure Month Folder Exists |
| CreateMonthFolder | googleDrive | Create YYYY-MM folder | IfFolderExists | ResultFolder | ## Ensure Month Folder Exists |
| ResultFolder | set | Normalize target_folder_id | IfFolderExists / CreateMonthFolder | MoveToMonth | ## Move Document to Month Folder |
| MoveToMonth | googleDrive | Move file to month folder | ResultFolder | SuccessLog | ## Move Document to Month Folder |
| SuccessLog | telegram | Success notification | MoveToMonth | — | ## Success Notification… |
| RenameToReview | googleDrive | Prefix filename with [REVIEW] | IfOcrSuccess / IfInvalidContent | ErrorAlert | ## Require manual review… |
| ErrorAlert | telegram | Notify processing failure | RenameToReview | — | ## Require manual review… |
| BudgetTrigger | telegramTrigger | Telegram entry for commands | — | GetAppSettings1 | ## Chat Trigger |
| GetAppSettings1 | httpRequest | Load settings for authorization/first run | BudgetTrigger | IfFirstRun | ## Ensure Bot Privacy… |
| IfFirstRun | if | Gate: first run registration path | GetAppSettings1 | FirstRunOnlyStart, IfAuthorized | ## Ensure Bot Privacy… |
| FirstRunOnlyStart | if | On first run, require /start | IfFirstRun | IfStartMessage | ## Ensure Bot Privacy… |
| IfAuthorized | if | Owner-lock based on chat_id | IfFirstRun | IfStartMessage, NotAuthorizedMessage | ## Ensure Bot Privacy… |
| NotAuthorizedMessage | telegram | Deny access to non-owner | IfAuthorized | — | ## Ensure Bot Privacy… |
| IfStartMessage | if | /start vs normal message routing | IfAuthorized / FirstRunOnlyStart | SetDefaults, BudgetInput | ## Ensure Bot Privacy… |
| SetDefaults | set | Default allowed categories | IfStartMessage | WelcomeMessage | ## Ensure Bot Privacy… |
| WelcomeMessage | telegram | Welcome + categories list | SetDefaults | SaveAppSettings | ## Ensure Bot Privacy… |
| SaveAppSettings | httpRequest | PUT settings (chat_id + allowed_categories) | WelcomeMessage | — | ## Ensure Bot Privacy… |
| BudgetInput | set | Build agent input fields | IfStartMessage | BudgetAgent | ## 2. Budget Management AI Agent… |
| Gpt4oBudgetAgent | lmChatOpenRouter | GPT‑4o model provider (budget agent) | — | BudgetAgent (ai_languageModel) | ## 2. Budget Management AI Agent… |
| BudgetMemory | memoryBufferWindow | Conversation memory per chat | — | BudgetAgent (ai_memory) | ## 2. Budget Management AI Agent… |
| JsonStorageMcp | mcpClientTool | MCP tool to read/write Ainoflow JSON | — | BudgetAgent (ai_tool) | ## 2. Budget Management AI Agent… |
| Calculator | toolCalculator | Agent math helper | — | BudgetAgent (ai_tool) | ## 2. Budget Management AI Agent… |
| Think | toolThink | Agent reasoning helper | — | BudgetAgent (ai_tool) | ## 2. Budget Management AI Agent… |
| MonthlySummary | toolWorkflow | Tool: get monthly summary via self workflow | — | BudgetAgent (ai_tool) | ## 2. Budget Management AI Agent… |
| BudgetAgent | langchain.agent | Budget ops agent (set/list/status/delete) | BudgetInput | BudgetReply | ## 2. Budget Management AI Agent… |
| BudgetReply | telegram | Reply with agent output | BudgetAgent | — | ## 2. Budget Management AI Agent… |
| WeeklyTrigger | scheduleTrigger | Weekly summary schedule | — | GetAppSettings2 | ## Weekly Trigger |
| GetAppSettings2 | httpRequest | Load chat_id for weekly message | WeeklyTrigger | GetWeeklySummary | ## Get Weekly Summary… |
| GetWeeklySummary | httpRequest | GET current month summary totals | GetAppSettings2 | IfHasData | ## Get Weekly Summary… |
| IfHasData | if | Gate: only send if invoice_count > 0 | GetWeeklySummary | CalcWeeklySummary | ## Get Weekly Summary… |
| CalcWeeklySummary | code | Weekly summary calculations | IfHasData | BuildWeeklyMessage | ## Get Weekly Summary… |
| BuildWeeklyMessage | set | Format weekly Telegram message | CalcWeeklySummary | WeeklySummaryMessage | ## Get Weekly Summary… |
| WeeklySummaryMessage | telegram | Send weekly report | BuildWeeklyMessage | — | ## Get Weekly Summary… |
| MonthlyTrigger | scheduleTrigger | Monthly report schedule | — | GetAppConfig2 | ## Monthly Trigger |
| GetAppConfig2 | httpRequest | Load chat_id for monthly report | MonthlyTrigger | GetMonthlySummaryText | ## Get Monthly Summary… |
| GetMonthlySummaryText | executeWorkflow | Self-call workflow to generate report text | GetAppConfig2 | MonthlySummaryMessage | ## Get Monthly Summary… |
| MonthlySummaryMessage | telegram | Send monthly report (message or error) | GetMonthlySummaryText | — | ## Get Monthly Summary… |
| GetMonthlySummaryTrigger | executeWorkflowTrigger | Sub-workflow entry with year/month inputs | — | WorkflowInput | ## Monthly Summary Calculation Logic… |
| WorkflowInput | set | Normalize year/month inputs | GetMonthlySummaryTrigger | GetMonthlySummary | ## Monthly Summary Calculation Logic… |
| GetMonthlySummary | httpRequest | GET summary for requested month | WorkflowInput | IfHasMonthlyData1 | ## Monthly Summary Calculation Logic… |
| IfHasMonthlyData1 | if | Gate: summary exists | GetMonthlySummary | CalcMonthlySummary, ErrorOutput | ## Monthly Summary Calculation Logic… |
| CalcMonthlySummary | code | Monthly calculations and sorting | IfHasMonthlyData1 | BuildMonthlyMessage | ## Monthly Summary Calculation Logic… |
| BuildMonthlyMessage | set | Format monthly report | CalcMonthlySummary | — | ## Monthly Summary Calculation Logic… |
| ErrorOutput | set | Error string when summary missing | IfHasMonthlyData1 | — | ## Monthly Summary Calculation Logic… |
| ManualResetTrigger | manualTrigger | Manual entry for destructive reset | — | GetAppSettings3 | ## 5. Data Reset… |
| GetAppSettings3 | httpRequest | Load chat_id for reset approval | ManualResetTrigger | ApproveRemove | ## 5. Data Reset… |
| ApproveRemove | telegram (sendAndWait) | Double approval prompt | GetAppSettings3 | IfApproved | ## 5. Data Reset… |
| IfApproved | if | Gate: only delete if approved | ApproveRemove | ForEachCategory, RestartMessage | ## 5. Data Reset… |
| ForEachCategory | httpRequest | List all storage categories | IfApproved | If | ## 5. Data Reset… |
| If | if | Filter categories starting with invoice- | ForEachCategory | ForEachCategoryItem | ## 5. Data Reset… |
| ForEachCategoryItem | httpRequest | List items in selected category | If | DeleteItem | ## 5. Data Reset… |
| DeleteItem | httpRequest | Delete each item | ForEachCategoryItem | — | ## 5. Data Reset… |
| RestartMessage | telegram | Reset completion message | IfApproved | — | ## 5. Data Reset… |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites (Credentials / External Services)
1. **Google Drive OAuth2 credential** in n8n
   - Used by: `GetFiles`, `DownloadInvoice`, `RenameFile`, `SearchMonthFolder`, `CreateMonthFolder`, `MoveToMonth`, `RenameToReview`, `RenameAsDuplicate`.
2. **Telegram bot credential** in n8n
   - Used by: `BudgetTrigger`, `AlertMessage`, `ErrorAlert`, `SuccessLog`, `DuplicateLog`, `BudgetReply`, `WelcomeMessage`, `NotAuthorizedMessage`, `WeeklySummaryMessage`, `MonthlySummaryMessage`, `ApproveRemove`, `RestartMessage`.
3. **OpenRouter credential** in n8n
   - Used by: `Gpt4oCategorizer`, `Gpt4oBudgetAgent`.
4. **Ainoflow HTTP Bearer credential** in n8n
   - Used by all Ainoflow HTTP request nodes + MCP tool client.

---

### Step-by-step build

#### A) Scheduled Invoice Processing (Google Drive → OCR → AI → Storage → Move)
1. **Add `Schedule Trigger`** named `HourlyTrigger`
   - Cron: `0 8-21 * * *`
2. **Add `HTTP Request`** named `GetAppSettings`
   - Method: GET
   - URL: `https://api.ainoflow.io/api/v1/storage/json/invoice-config/settings`
   - Auth: **Generic Credential Type → HTTP Bearer Auth** (Ainoflow)
   - Enable “Continue On Fail” (equivalent behavior to `onError: continueRegularOutput`) as desired.
3. **Add `IF`** named `IfRegistrationExists`
   - Condition: `{{$json.chat_id}}` exists.
   - True → continue; False → end.
4. **Add `Google Drive`** node `GetFiles`
   - Resource: File/Folder
   - Operation: Search/List (files in folder)
   - Folder: select your invoices inbox folder (equivalent to `/Invoices/`)
   - Query: `*.pdf`
5. **Add `Split In Batches`** node `LoopInput`
   - Use default batch size (or 1).
   - Connect its “continue” output back to itself (loop) and forward to next nodes.
6. **Add `Set`** node `WorkflowConfig`
   - Fields:
     - `chat_id` = settings chat id
     - `input_folder_id` = file parent id
     - `alert_threshold` = 80
     - `allowed_categories` = settings allowed_categories or `Other`
     - `default_currency` = `EUR`
     - `review_prefix` = `[REVIEW] - `
     - `duplicate_prefix` = `[DUPLICATE] - `
7. **Add `IF`** node `IfNotForReviewOrDuplicate`
   - Condition: filename does not start with `review_prefix` AND does not start with `duplicate_prefix`.
8. **Add `Google Drive`** node `DownloadInvoice`
   - Operation: Download
   - File ID: current loop item id
9. **Add `HTTP Request`** node `OcrExtract`
   - Method: POST
   - URL: `https://api.ainoflow.io/api/v1/convert/submit-file`
   - Auth: Ainoflow bearer
   - Body: multipart-form-data with:
     - `languages` = `en,de,pt`
     - `outputs` = `text`
     - `response` = `direct`
     - `file` = binary field from download (`data`)
   - Configure to continue on error (so OCR failures don’t crash the whole run).
10. **Add `Set`** node `OcrOutput`
    - `invoice_ocr_text` = OCR response text (content[0].text)
11. **Add `Crypto`** node `Crypto`
    - Operation: SHA256
    - Input: binary
    - Output field: `input_file_hash`
12. **Add `Set`** node `InputFile`
    - `input_file_id` = drive file id
    - `input_file_hash` = from Crypto
    - `input_file_filename` = downloaded binary filename (with fallback)
    - `input_file_extension` = derived from filename
13. **Add `Merge`** node `Merge`
    - Mode: Combine → Combine All
    - Input 0: OCR output; Input 1: file metadata
14. **Add `IF`** node `IfOcrSuccess`
    - Condition: `invoice_ocr_text` exists
    - True → budgets/AI
    - False → review handling
15. **Add `HTTP Request`** node `GetBudgets`
    - GET `https://api.ainoflow.io/api/v1/storage/json/invoice-config/budgets`
    - Query param: `allowEmpty=true`
16. **Add `Set`** node `ProcessingInput`
    - Include:
      - `ocr_text`
      - `allowed_categories`
      - `budgets_context` (formatted string of budgets)
      - carry `input_folder_id`, `input_file_hash`, `input_file_id`, `input_file_filename`, `input_file_extension`
17. **Add OpenRouter `LM Chat`** node `Gpt4oCategorizer`
    - Model: `openai/gpt-4o`
18. **Add `Structured Output Parser`** node `StructuredOutputParser`
    - Manual schema matching the workflow (invoice vs non-invoice branches)
    - Auto-fix enabled
    - Connect language model input from `Gpt4oCategorizer`
19. **Add `LangChain Agent`** node `AICategorizer`
    - Input text: filename + OCR text
    - System message: invoice analyzer rules + output schema (as in workflow)
    - Connect:
      - Language model: `Gpt4oCategorizer`
      - Output parser: `StructuredOutputParser`
      - Optional tool: `Think` tool node if desired
20. **Add `IF`** node `IfInvalidContent`
    - Condition: `output.is_invoice` is false
    - True → review path
    - False → dedupe/storage path
21. **Add `Set`** node `StorageInput`
    - `storage_category` = `invoice-data-YYYY-MM` from invoice date
    - `invoice_month` = `YYYY-MM`
    - pass file ids/hash/name/ext
22. **Add `HTTP Request`** node `CheckInvoiceExists`
    - GET `https://api.ainoflow.io/api/v1/storage/json/{{storage_category}}/{{input_file_hash}}`
    - Query: `allowEmpty=true`
    - Always output data
23. **Add `IF`** node `IfNewInvoice`
    - Condition: `Object.keys($json).length == 0`
    - True → save invoice
    - False → duplicate handling
24. **Add `HTTP Request`** node `SaveInvoice`
    - PUT `.../storage/json/{{storage_category}}/{{input_file_hash}}`
    - JSON body: vendor, normalized vendor, amount, currency, date, invoice_number, category, file_id, processed_at
25. **Add `HTTP Request`** node `GetSummary`
    - GET `.../storage/json/invoice-summary-YYYY-MM/totals?allowEmpty=true`
26. **Add `Code`** node `CalcSummary`
    - Implement summary mutation logic (category totals, pct, invoice_count, insights, anomalies).
27. **Add `Set`** node `SummaryInput`
    - Build:
      - `summary`, `summary_category`, `invoice`, `budget_status`
      - `needs_alert` (budget_warning OR pct>=80)
      - `file_info` including `new_filename`, `month_folder`, folder/file ids
28. **Add `HTTP Request`** node `SaveSummary`
    - PUT `.../storage/json/{{summary_category}}/totals`
    - Body: summary object
29. **Add `IF`** node `IfNeedsAlert`
    - Condition: `needs_alert == true`
30. **Add `Telegram`** node `AlertMessage`
    - Markdown message with category pct/spent/budget and Drive link
    - chatId = `WorkflowConfig.chat_id`
31. **Add `Google Drive`** node `RenameFile`
    - Update file name to `SummaryInput.file_info.new_filename`
32. **Add `Google Drive`** node `SearchMonthFolder`
    - Search folders within `input_folder_id` for `month_folder`
33. **Add `IF`** node `IfFolderExists`
    - Condition: found folder id exists
34. **Add `Google Drive`** node `CreateMonthFolder` (false path)
    - Create folder named `month_folder` under `input_folder_id`
35. **Add `Set`** node `ResultFolder`
    - `target_folder_id` = folder id from either search/create
36. **Add `Google Drive`** node `MoveToMonth`
    - Move file into `target_folder_id`
37. **Add `Telegram`** node `SuccessLog`
    - Markdown message with vendor/amount/category and final folder path
38. **Duplicate path:**
    - Add `Google Drive` node `RenameAsDuplicate` (from IfNewInvoice false)
    - Add `Telegram` node `DuplicateLog`
39. **Review path (OCR fail or invalid doc):**
    - Add `Google Drive` node `RenameToReview`
    - Add `Telegram` node `ErrorAlert`

---

#### B) Budget Management via Telegram (Owner-locked AI agent)
40. **Add `Telegram Trigger`** node `BudgetTrigger` (updates: message)
41. **Add `HTTP Request`** node `GetAppSettings1` (GET settings)
42. **Add `IF`** node `IfFirstRun` (chat_id not exists)
43. **Add `IF`** node `FirstRunOnlyStart` (text equals `/start`)
44. **Add `IF`** node `IfAuthorized` (incoming chat id equals stored chat_id)
45. **Add `Telegram`** node `NotAuthorizedMessage` (false path)
46. **Add `IF`** node `IfStartMessage` (text equals `/start`)
47. **Start branch:**
    - `SetDefaults` (allowed_categories string)
    - `WelcomeMessage` (show commands and categories)
    - `SaveAppSettings` (PUT settings with chat_id + allowed_categories)
48. **Normal message branch:**
    - `BudgetInput` (text/chat_id/allowed_categories)
    - `Gpt4oBudgetAgent` (OpenRouter GPT‑4o)
    - `BudgetMemory` (session key `budget_agent_<chat_id>`)
    - `JsonStorageMcp` (MCP endpoint + bearer)
    - `Calculator`, `Think` tools
    - `BudgetAgent` (system instructions + tool usage)
    - `BudgetReply` (Telegram send output)

---

#### C) Weekly & Monthly Scheduled Reports
49. **Weekly:**
    - `WeeklyTrigger` (Friday 17:00)
    - `GetAppSettings2` (GET settings)
    - `GetWeeklySummary` (GET current month summary)
    - `IfHasData` (invoice_count > 0)
    - `CalcWeeklySummary` (code)
    - `BuildWeeklyMessage` (format string)
    - `WeeklySummaryMessage` (Telegram)
50. **Monthly:**
    - `MonthlyTrigger` (day 10, hour 9)
    - `GetAppConfig2` (GET settings)
    - `GetMonthlySummaryText` (Execute Workflow → self) with inputs = previous month
    - `MonthlySummaryMessage` (Telegram)

---

#### D) Monthly Summary Internal Entry (Sub-workflow within same workflow)
51. **Add `Execute Workflow Trigger`** node `GetMonthlySummaryTrigger`
    - Define workflow inputs: `year` (number), `month` (number)
52. **Add `Set`** node `WorkflowInput` to normalize inputs (default to current year/month)
53. **Add `HTTP Request`** node `GetMonthlySummary` to fetch summary by computed `YYYY-MM`
54. **Add `IF`** node `IfHasMonthlyData1` (invoice_count > 0)
55. **Add `Code`** node `CalcMonthlySummary`
56. **Add `Set`** node `BuildMonthlyMessage` (message formatting)
57. **Add `Set`** node `ErrorOutput` on false path (no data)

---

#### E) Manual Reset (Destructive)
58. **Add `Manual Trigger`** node `ManualResetTrigger`
59. **Add `GetAppSettings3`** (GET settings)
60. **Add `ApproveRemove`** (Telegram sendAndWait with double approval)
61. **Add `IfApproved`** gate
62. **Add `ForEachCategory`** (GET categories index)
63. **Add `If`** filter for category startsWith `invoice-`
64. **Add `ForEachCategoryItem`** (GET keys in category)
65. **Add `DeleteItem`** (DELETE each key)
66. **Add `RestartMessage`** (Telegram reset completed)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google OAuth setup instructions | https://docs.n8n.io/integrations/builtin/credentials/google/ |
| Telegram bot creation guide | https://blog.n8n.io/create-telegram-bot/ |
| OpenRouter credential docs | https://docs.n8n.io/integrations/builtin/credentials/openrouter/ |
| Ainoflow signup | https://www.ainoflow.io/signup |
| Support / contact | https://ainovasystems.com/ |
| Operational requirement: must send `/start` before any use | Bot registration stores chat_id and locks access to a single owner; scheduled processing won’t run until settings exist. |
| Folder behavior | Inbox `/Invoices/`; monthly folders `/Invoices/YYYY-MM/`; failed files get `[REVIEW] - ` prefix; duplicates get `[DUPLICATE] - `. |
| Threshold note | The alert threshold is set in `WorkflowConfig` (80) but alert decision is hardcoded in `SummaryInput` to 80% unless invoice sets `budget_warning`. Consider wiring threshold consistently. |