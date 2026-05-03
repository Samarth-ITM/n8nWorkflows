Extract TV ad slots from Gmail attachments to Google Sheets using GPT-4o Vision

https://n8nworkflows.xyz/workflows/extract-tv-ad-slots-from-gmail-attachments-to-google-sheets-using-gpt-4o-vision-13004


# Extract TV ad slots from Gmail attachments to Google Sheets using GPT-4o Vision

## 1. Workflow Overview

**Purpose:** This workflow periodically scans Gmail for unprocessed emails, classifies them into categories (A/B/C), extracts structured “TV ad slot”-like data from attachments (Excel/PPTX/PDF/images) or from the email text using OpenAI (GPT-4o Vision / GPT-4o-mini), appends normalized rows to Google Sheets, then labels the processed emails in Gmail.

**Target use cases:**
- Operations teams receiving booking/cancellation/update sheets via email.
- Automated ingestion of semi-structured schedules from office files and images.
- Standardizing heterogeneous formats into a single Google Sheets table.

### 1.1 Scheduling & Email Intake
- Runs every 30 minutes.
- Pulls recent emails (last 60 days) that do **not** already have the “Processed_*” labels.

### 1.2 Domain Filtering & Classification
- A Code node filters emails by allowed sender domains or identifier codes in subject/body.
- Classifies emails into `CategoryA`, `CategoryB`, or default `CategoryC`.

### 1.3 Attachment Download & File-Type Routing
- Downloads attachments and detects whether the email contains:
  - Excel eligible for direct parsing (Category C only)
  - PPTX/PDF (or Excel in Category A/B) that must be converted to images for vision
  - Images for direct vision
  - Otherwise fallback to text-only GPT extraction

### 1.4 Data Extraction Paths
- **Direct Excel parse** (Category C Excel): Extract rows using Extract From File + custom parser.
- **PPTX/PDF (and variable Excel A/B)**: ConvertAPI → images → upload to S3 → GPT-4o Vision.
- **Image attachments**: Upload to S3 → GPT-4o Vision.
- **Text-only**: GPT-4o-mini.

### 1.5 Normalization & Storage
- Merges all extraction outputs.
- Parses GPT responses into a unified schema.
- Appends rows to Google Sheets.

### 1.6 Gmail Post-Processing (Labeling)
- Ensures labels exist (`Processed_CategoryA/B/C`) and creates missing ones.
- Applies the correct label to each processed email.

---

## 2. Block-by-Block Analysis

### Block 0 — Documentation / Operator Notes (Sticky Notes)
**Overview:** Provides embedded operator guidance: requirements, variable names, label behavior, and where to customize parsing prompts/schema.  
**Nodes involved:**  
- Workflow Description (Sticky Note)  
- Email Classification Info (Sticky Note)  
- File Processing Info (Sticky Note)  
- ConvertAPI Info (Sticky Note)  
- AWS S3 Info (Sticky Note)  
- Google Sheets Info (Sticky Note)  
- Gmail Labels Info (Sticky Note)  
- Configuration Variables (Sticky Note)

**Node details (all Sticky Notes):**
- **Type/role:** `n8n-nodes-base.stickyNote` – documentation only; no runtime effect.
- **Edge cases:** None.

---

### Block 1 — Schedule Trigger & Gmail Fetch
**Overview:** Runs on a schedule, fetches candidate emails not yet labeled as processed.  
**Nodes involved:** Schedule Trigger → Gmail - Get Emails

#### 1) Schedule Trigger
- **Type/role:** `scheduleTrigger` – workflow entry point.
- **Config:** Every 30 minutes.
- **Outputs:** Triggers the workflow run.
- **Failure modes:** n8n scheduler disabled; timezone expectations.

#### 2) Gmail - Get Emails
- **Type/role:** `gmail` node (operation: getAll) – retrieves a list of messages.
- **Config choices:**
  - `limit: 1` (only processes one email per run).
  - Search query: excludes any email already labeled `Processed_CategoryA/B/C`.
  - `receivedAfter: {{$now.minus({ days: 60 }).toISO()}}`.
- **Outputs:** Email metadata (id, subject, snippet, from, etc.).
- **Failure modes:** OAuth errors; Gmail API quota; query syntax issues; returns 0 items.

---

### Block 2 — Domain Filter & Email Classification
**Overview:** Applies allow-list logic and classifies the email into CategoryA/B/C; rejects irrelevant emails by flagging `_skip`.  
**Nodes involved:** Domain Filter → Filter Valid Emails

#### 3) Domain Filter
- **Type/role:** `code` – classification + gating.
- **Key configuration choices:**
  - `allowedDomains`: empty by default → means “allow all domains” unless you add entries.
  - Keyword arrays:
    - `categoryAKeywords`, `categoryBKeywords`, `categoryCKeywords` (all empty by default).
  - Optional `identifierCodes` matches bracketed forms like `【ABC】` or `[ABC]` after normalizing full-width characters.
- **Key outputs/fields:**
  - `_skip: boolean`
  - `_email_type: 'CategoryA'|'CategoryB'|'CategoryC'`
- **Logic notes / edge cases:**
  - If **all keyword lists are empty** and `allowedDomains` is empty, it effectively processes everything (since domain check passes).
  - If you populate `allowedDomains` but keep all keywords empty, it processes all allowed-domain emails.
  - If domain doesn’t match and no identifier code match → `_skip: true`.

#### 4) Filter Valid Emails
- **Type/role:** `filter` – removes items flagged `_skip: true`.
- **Condition:** `$json._skip != true`.
- **Failure modes:** If `_skip` missing, condition evaluates as not-equals true → typically passes; ensure Domain Filter always sets `_skip`.

---

### Block 3 — Download Attachments & Detect File Types
**Overview:** Fetches full email content and attachments, then determines routing flags and binary keys for downstream steps.  
**Nodes involved:** Gmail - Get Attachments → Detect File Types → IF - Has Excel?

#### 5) Gmail - Get Attachments
- **Type/role:** `gmail` node (operation: get) – retrieves message details and downloads attachments.
- **Config choices:**
  - `downloadAttachments: true`
  - Attachments are stored under binary properties prefixed with `attachment_`.
  - `messageId: {{$json.id}}`
- **Outputs:** Email JSON plus binaries (`item.binary.attachment_0`, etc.).
- **Failure modes:** Large attachments; Gmail API errors; missing permissions to download.

#### 6) Detect File Types
- **Type/role:** `code` – inspects binary attachments and sets routing flags.
- **Key outputs:**
  - `has_excel`: true if Excel exists **and** email type is not variable format (CategoryC only).
  - `has_pptx_pdf`: true if PPTX/PDF exists OR Excel exists for CategoryA/B (variable format → needs conversion/vision).
  - `has_image`: true if image exists.
  - `excel_binary_key`, `conversion_binary_key`, `conversion_type`, `image_binary_key`
  - Copies `_email_type` forward.
- **Important logic:**
  - CategoryA/B forces Excel into conversion/vision path (`conversion_type: 'xlsx'`).
- **Edge cases/failure modes:**
  - Multiple attachments: it only stores the *last matching* type in `fileInfo.*` (single slot per type). If you expect multiple excels/images, this node would need expansion.
  - MIME/type detection depends on `fileName` and `mimeType`.
  - Attempts to recover `_email_type` from “Filter Valid Emails” items by matching `id`; may fail if item ordering differs.

#### 7) IF - Has Excel?
- **Type/role:** `if` – branches into Excel direct extraction vs other formats.
- **Condition:** `has_excel == true`
- **Outputs:**
  - True branch → Extract from Excel
  - False branch → IF - Has PPTX/PDF?

---

### Block 4 — Direct Excel Extraction (Category C only)
**Overview:** Extracts rows from Excel and formats them into a GPT-like `choices[].message.content` payload so downstream parsing is unified.  
**Nodes involved:** Extract from Excel → Parse Excel Data → Merge All Paths

#### 8) Extract from Excel
- **Type/role:** `extractFromFile` – reads spreadsheet binary and outputs rows as JSON.
- **Config:**
  - Operation: `xlsx`
  - Binary property: `{{$json.excel_binary_key || 'attachment_0'}}`
- **Failure modes:** Corrupt Excel; wrong binary key; password-protected file; very large sheets.

#### 9) Parse Excel Data
- **Type/role:** `code` – custom parser mapping spreadsheet row structure into structured objects.
- **Key behavior:**
  - Builds `dataBySource` keyed by detected `sourceName`.
  - Attempts to infer year/month from text using regex `(20\d{2})[-\/年]\s*(\d{1,2})`.
  - Extracts source codes from brackets `【CODE】`.
  - Converts Excel time fractions into `HH:MM`.
  - Emits items shaped like OpenAI responses:
    - `choices: [{ message: { content: JSON.stringify(dataForEmail) } }]`
  - Adds metadata fields (`_meta_subject`, `_meta_from`, `_meta_id`, `_email_type`).
- **Edge cases/failure modes:**
  - Parsing is highly dependent on column names like `__EMPTY_1`, `__EMPTY_2`, etc.
  - If “IF - Has Excel?” produced multiple emails over time, this code tries to map results back via `$('IF - Has Excel?').all()`; ordering mismatches can mis-attribute rows.
  - If no data detected, outputs `[{notes: '⚠️No data found'}]`.

---

### Block 5 — PPTX/PDF (and variable Excel) → Convert → S3 → GPT-4o Vision
**Overview:** Converts PPTX/PDF (or Excel A/B) to PNG images, uploads images to S3, then sends all pages to GPT-4o Vision for structured extraction.  
**Nodes involved:** IF - Has PPTX/PDF? → ConvertAPI to PNG → Prepare for S3 Upload → S3 Upload (PPTX) → Collect S3 URLs → Prepare GPT Request (PPTX) → GPT Vision Analysis → Merge All Paths

#### 10) IF - Has PPTX/PDF?
- **Type/role:** `if` – routes to conversion vs image/text checks.
- **Condition:** `has_pptx_pdf == true`
- **Outputs:**
  - True → ConvertAPI to PNG
  - False → IF - Has Image?

#### 11) ConvertAPI to PNG
- **Type/role:** `httpRequest` – calls ConvertAPI to convert file to PNG.
- **Config:**
  - URL: `https://v2.convertapi.com/convert/{{conversion_type}}/to/png`
  - Method: POST, multipart form-data
  - Form field `File`: binary from `conversion_binary_key`
  - Auth: “genericCredentialType” with `httpHeaderAuth` (ConvertAPI secret)
- **Failure modes:**
  - ConvertAPI auth/credits; unsupported file; large files; API latency/timeouts.

#### 12) Prepare for S3 Upload
- **Type/role:** `code` – transforms ConvertAPI response into up to 15 image binaries for upload.
- **Key behavior:**
  - Reads `response.Files[].FileData` and creates new items with `binary.data`.
  - Limits pages to 15.
  - Adds `_page_filename`, `_page_index`, `_total_pages`, plus email metadata.
- **Edge cases:**
  - Assumes ConvertAPI returns `Files` array with `FileData`.
  - Metadata mapping uses positional index correlation with items from Detect File Types; if ConvertAPI outputs differ in ordering/count, association may break.

#### 13) S3 Upload (PPTX)
- **Type/role:** `awsS3` – uploads each generated PNG to S3.
- **Config:**
  - Operation: upload
  - Bucket: `{{$vars.S3_BUCKET_NAME || 'your-bucket-name'}}`
  - File name: `{{$json._page_filename}}`
  - Binary property: defaults to `data` (because Prepare for S3 Upload sets `binary.data`)
- **Failure modes:** AWS credentials; bucket policy; region mismatch; object ACL/ownership constraints.

#### 14) Collect S3 URLs
- **Type/role:** `code` – builds public S3 URLs and groups them by email id.
- **Key expressions:**
  - `bucket = $vars.S3_BUCKET_NAME`, `region = $vars.AWS_REGION`
  - URL format: `https://{bucket}.s3.{region}.amazonaws.com/{filename}`
- **Important:** This assumes objects are publicly accessible *or* OpenAI can fetch them. If the bucket is private, GPT image_url fetch will fail.
- **Failure modes:** Wrong region; non-public bucket; filename collisions.

#### 15) Prepare GPT Request (PPTX)
- **Type/role:** `code` – constructs OpenAI Chat Completions request with multiple `image_url` entries.
- **Config highlights:**
  - Model: `gpt-4o`
  - System prompt demands: “JSON array only”.
  - User content: text prompt + images with `detail: 'high'`
  - `temperature: 0.1`, `max_tokens: 4096`
- **Edge cases:** Too many pages/images may exceed OpenAI limits; image URLs not reachable; prompt/schema mismatch.

#### 16) GPT Vision Analysis
- **Type/role:** `httpRequest` – calls OpenAI `/v1/chat/completions`.
- **Config:**
  - Auth: `openAiApi` credential
  - Timeout: 300000 ms (5 minutes)
  - Body: `{{$json.gpt_request_body}}` (already JSON-stringified)
- **Failure modes:** OpenAI auth; rate limits; payload too large; non-JSON response; network timeout.

---

### Block 6 — Image Attachment → S3 → GPT-4o Vision
**Overview:** Uploads a single image attachment to S3 and sends it to GPT-4o Vision.  
**Nodes involved:** IF - Has Image? → Prepare Image Metadata → S3 Upload (Image) → Prepare GPT Request (Image) → GPT Image Analysis → Merge All Paths

#### 17) IF - Has Image?
- **Type/role:** `if` – selects image path vs text path.
- **Condition:** `has_image == true`
- **Outputs:**
  - True → Prepare Image Metadata
  - False → Prepare GPT Request (Text)

#### 18) Prepare Image Metadata
- **Type/role:** `code` – selects correct binary key and prepares S3 filename + metadata.
- **Outputs:**
  - `_image_key` (defaults to `attachment_0`)
  - `_s3_filename` as `email-image-{emailId}-{timestamp}.png`
  - `_email_type`, subject/from/id, attachments list
- **Edge cases:** If image is JPG but filename ends .png; content-type mismatch isn’t fatal but may confuse downstream.

#### 19) S3 Upload (Image)
- **Type/role:** `awsS3` – uploads the email image to S3.
- **Config:**
  - Bucket: `{{$vars.S3_BUCKET_NAME || 'your-bucket-name'}}`
  - File name: `{{$json._s3_filename}}`
  - Binary property: `{{$json._image_key}}`
- **Failure modes:** Same as other S3 upload; missing binary property if attachment key differs.

#### 20) Prepare GPT Request (Image)
- **Type/role:** `code` – builds GPT-4o vision request for a single image URL.
- **Key dependency:** Reads S3 upload response field `Location` for `imageUrl`.
- **Failure modes:**
  - If S3 node doesn’t return `Location` (depends on node/version and AWS response), this returns `{_skip: true}` and nothing is extracted.
  - If bucket is private, OpenAI cannot fetch the image.

#### 21) GPT Image Analysis
- **Type/role:** `httpRequest` – calls OpenAI chat completions.
- **Config:** model set inside request body to `gpt-4o`.
- **Failure modes:** same as GPT Vision Analysis (but usually smaller payload).

---

### Block 7 — Text-only Email → GPT-4o-mini
**Overview:** If no usable attachments are detected, extracts structured fields from email plain text using GPT-4o-mini.  
**Nodes involved:** Prepare GPT Request (Text) → GPT Text Analysis → Merge All Paths

#### 22) Prepare GPT Request (Text)
- **Type/role:** `code` – creates OpenAI request from subject + first 3000 chars of body.
- **Config:**
  - Model: `gpt-4o-mini`
  - Output must be JSON array only.
  - `max_tokens: 2048`
- **Edge cases:** Email body may be missing `textPlain` and `text`; extraction quality depends on email formatting.

#### 23) GPT Text Analysis
- **Type/role:** `httpRequest` – OpenAI chat completions.
- **Failure modes:** OpenAI auth/limits; long text; non-JSON output.

---

### Block 8 — Merge, Parse, Normalize
**Overview:** Consolidates results from Excel/GPT paths, parses the JSON payload into final rows, flags unreadable/parse-failed items for review.  
**Nodes involved:** Merge All Paths → Parse All Results → Filter Valid Data

#### 24) Merge All Paths
- **Type/role:** `merge` – joins multiple incoming branches.
- **Config:** `alwaysOutputData: true` to avoid stopping when one branch is empty.
- **Edge cases:** Ordering across branches is not guaranteed; downstream tries to recover metadata using indices and model name.

#### 25) Parse All Results
- **Type/role:** `code` – the main normalization layer.
- **Key behaviors:**
  - Extracts metadata either directly (if `_meta_id` present) or by aligning with “Prepare GPT Request” nodes via indices.
  - Pulls GPT response from `choices[0].message.content`.
  - Detects “unreadable” by checking for patterns like `sorry`, `cannot read`, etc.
  - Parses JSON array from:
    - fenced ```json blocks, or
    - first `[...]` array match
  - On parse error/unreadable: emits a “Needs Review” row with warning notes and `_unreadable: true`.
  - Derives `category` from `start_time` if missing (Morning/Afternoon/Evening/Night).
  - Output schema includes: `email_type,date,day_of_week,start_time,end_time,source_name,name,amount,category,type,status,client,notes,created_at,email_subject,sender,email_id,attachments`.
- **Failure modes / edge cases:**
  - If GPT returns non-JSON or partial JSON, parse fails → “Parse failed” row.
  - Index-based metadata association can mismatch when some branches produce fewer results than others.
  - Category derivation assumes `HH:MM` formatting.

#### 26) Filter Valid Data
- **Type/role:** `filter` – ensures rows have `email_type`.
- **Condition:** `email_type` not empty.
- **Edge cases:** Rows produced with `_no_data` won’t pass.

---

### Block 9 — Persist to Google Sheets
**Overview:** Appends normalized rows to a Google Sheet.  
**Nodes involved:** Save to Google Sheets

#### 27) Save to Google Sheets
- **Type/role:** `googleSheets` – append rows.
- **Config:**
  - Operation: append
  - Document ID: `{{$vars.GOOGLE_SHEET_ID || 'your-google-sheet-id'}}`
  - Sheet: `Sheet1`
  - Mapping: auto-map input keys to columns
- **Failure modes:** OAuth; wrong document ID; missing sheet; column mismatches (auto-map may ignore unknown fields).

---

### Block 10 — Determine Email IDs & Ensure Gmail Labels Exist
**Overview:** Collects unique email IDs processed, fetches Gmail labels, creates missing labels, and prepares label IDs for application.  
**Nodes involved:** Extract Email IDs → Get Gmail Labels → Check Labels → IF - Need Labels? → (Prepare Label Creation → Filter Labels to Create → Create Gmail Label → Get New Label IDs) OR (Pass Existing Labels) → Merge Label IDs

#### 28) Extract Email IDs
- **Type/role:** `code` – collects unique `email_id` and final category for labeling.
- **Logic:**
  - Builds a Map of email_id → type; if any row `_unreadable: true`, marks type as `NeedsReview`.
  - Also adds IDs from “Filter Valid Emails” (to label emails even if no rows were saved).
- **Edge cases:**
  - `NeedsReview` is produced but **no label mapping exists** for it later, so these emails may receive no label unless you extend mapping.

#### 29) Get Gmail Labels
- **Type/role:** `httpRequest` – lists Gmail labels via Gmail API.
- **Auth:** `gmailOAuth2` credential.
- **Failure modes:** OAuth, API quota.

#### 30) Check Labels
- **Type/role:** `code` – checks whether required labels exist.
- **Hardcoded label names:**
  - `Processed_CategoryA`, `Processed_CategoryB`, `Processed_CategoryC`
- **Output:** `existing_labels` (name→id), `missing_labels`, `needs_creation`.

#### 31) IF - Need Labels?
- **Type/role:** `if` – branches to label creation if missing labels exist.

#### 32) Prepare Label Creation
- **Type/role:** `code` – expands missing label names into one item per label to create.

#### 33) Filter Labels to Create
- **Type/role:** `filter` – passes only labels where `_skip_creation != true`.

#### 34) Create Gmail Label
- **Type/role:** `httpRequest` – POST to Gmail labels endpoint.
- **Body:** `{ name, labelListVisibility: 'labelShow', messageListVisibility: 'show' }`
- **Failure modes:** Label already exists (race condition); insufficient permissions.

#### 35) Get New Label IDs
- **Type/role:** `code` – merges newly created labels with existing labels into `all_labels`.

#### 36) Pass Existing Labels
- **Type/role:** `code` – used when no creation needed; passes `existing_labels` as `all_labels`.

#### 37) Merge Label IDs
- **Type/role:** `code` – consolidates outputs from either label-creation branch or pass-through branch.

---

### Block 11 — Apply Gmail Labels to Processed Emails
**Overview:** Determines the correct label for each email and applies it via Gmail modify endpoint.  
**Nodes involved:** Prepare Label Application → Has Emails? → Apply Gmail Labels

#### 38) Prepare Label Application
- **Type/role:** `code` – builds one item per email with the label ID and request body.
- **Type-to-label mapping (hardcoded):**
  - `CategoryA` → `Processed_CategoryA`
  - `CategoryB` → `Processed_CategoryB`
  - `CategoryC` → `Processed_CategoryC`
- **Output fields:**
  - `email_id`, `label_id`, `request_body: {"addLabelIds":[...]}`
- **Edge cases:**
  - Emails marked `NeedsReview` won’t be labeled because mapping does not include it.
  - If `all_labels[labelName]` missing, that email is silently skipped.

#### 39) Has Emails?
- **Type/role:** `filter` – removes `_no_emails == true`.

#### 40) Apply Gmail Labels
- **Type/role:** `httpRequest` – POST to `.../messages/{id}/modify`.
- **Auth:** `gmailOAuth2`.
- **Failure modes:** Gmail modify permission missing; message ID invalid; rate limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Description | stickyNote | Documentation | — | — | ##  Email Attachment Processor Template / Monitors Gmail, classifies, processes attachments with GPT Vision, saves to Sheets, labels emails, optional API/Slack; lists requirements and setup steps |
| Email Classification Info | stickyNote | Documentation | — | — | ###  Email Type Detection / Customize keywords for categories in “Domain Filter” |
| File Processing Info | stickyNote | Documentation | — | — | ###  File Processing Routes / Excel direct vs GPT, PPTX/PDF convert to images, image direct, text-only fallback |
| ConvertAPI Info | stickyNote | Documentation | — | — | ###  ConvertAPI Setup / https://www.convertapi.com / Replace HTTP Header Auth credential with ConvertAPI secret |
| AWS S3 Info | stickyNote | Documentation | — | — | ###  AWS S3 Setup / Create bucket, configure AWS creds, set bucket variable; images may be cleaned up periodically |
| Google Sheets Info | stickyNote | Documentation | — | — | ###  Google Sheets Output / Customize output schema in “Parse All Results”; update Document ID |
| Gmail Labels Info | stickyNote | Documentation | — | — | ###  Gmail Labels / Processed_CategoryA/B/C; created automatically; customize names in code nodes |
| Configuration Variables | stickyNote | Documentation | — | — | ###  Required Variables: S3_BUCKET_NAME, AWS_REGION, GOOGLE_SHEET_ID; Required Credentials: Gmail OAuth2, OpenAI API, AWS S3, Google Sheets OAuth2, ConvertAPI header auth |
| Schedule Trigger | scheduleTrigger | Entry point (time-based) | — | Gmail - Get Emails |  |
| Gmail - Get Emails | gmail | Fetch candidate emails | Schedule Trigger | Domain Filter |  |
| Domain Filter | code | Domain allow-list + classification | Gmail - Get Emails | Filter Valid Emails | ###  Email Type Detection / Customize keywords for categories in “Domain Filter” |
| Filter Valid Emails | filter | Drop skipped emails | Domain Filter | Gmail - Get Attachments |  |
| Gmail - Get Attachments | gmail | Download attachments | Filter Valid Emails | Detect File Types |  |
| Detect File Types | code | Determine routing & binary keys | Gmail - Get Attachments | IF - Has Excel? | ###  File Processing Routes / Excel direct vs GPT, PPTX/PDF convert to images, image direct, text-only fallback |
| IF - Has Excel? | if | Route Excel direct parse | Detect File Types | Extract from Excel; IF - Has PPTX/PDF? |  |
| Extract from Excel | extractFromFile | Read XLSX rows | IF - Has Excel? (true) | Parse Excel Data |  |
| Parse Excel Data | code | Custom Excel parsing → normalized JSON payload | Extract from Excel | Merge All Paths |  |
| IF - Has PPTX/PDF? | if | Route to ConvertAPI | IF - Has Excel? (false) | ConvertAPI to PNG; IF - Has Image? | ###  File Processing Routes / Excel direct vs GPT, PPTX/PDF convert to images, image direct, text-only fallback |
| ConvertAPI to PNG | httpRequest | Convert file to PNG pages | IF - Has PPTX/PDF? (true) | Prepare for S3 Upload | ###  ConvertAPI Setup / https://www.convertapi.com / Replace HTTP Header Auth credential with ConvertAPI secret |
| Prepare for S3 Upload | code | Split conversion output into page images | ConvertAPI to PNG | S3 Upload (PPTX) |  |
| S3 Upload (PPTX) | awsS3 | Upload converted pages | Prepare for S3 Upload | Collect S3 URLs | ###  AWS S3 Setup / Create bucket, configure AWS creds, set bucket variable; images may be cleaned up periodically |
| Collect S3 URLs | code | Build public URLs and group by email | S3 Upload (PPTX) | Prepare GPT Request (PPTX) |  |
| Prepare GPT Request (PPTX) | code | Build GPT-4o vision request (multi-image) | Collect S3 URLs | GPT Vision Analysis |  |
| GPT Vision Analysis | httpRequest | OpenAI vision call | Prepare GPT Request (PPTX) | Merge All Paths |  |
| IF - Has Image? | if | Route to image S3+vision or text | IF - Has PPTX/PDF? (false) | Prepare Image Metadata; Prepare GPT Request (Text) | ###  File Processing Routes / Excel direct vs GPT, PPTX/PDF convert to images, image direct, text-only fallback |
| Prepare Image Metadata | code | Prepare S3 filename + binary key | IF - Has Image? (true) | S3 Upload (Image) |  |
| S3 Upload (Image) | awsS3 | Upload original image attachment | Prepare Image Metadata | Prepare GPT Request (Image) | ###  AWS S3 Setup / Create bucket, configure AWS creds, set bucket variable; images may be cleaned up periodically |
| Prepare GPT Request (Image) | code | Build GPT-4o vision request (single image) | S3 Upload (Image) | GPT Image Analysis |  |
| GPT Image Analysis | httpRequest | OpenAI vision call | Prepare GPT Request (Image) | Merge All Paths |  |
| Prepare GPT Request (Text) | code | Build GPT-4o-mini request from email text | IF - Has Image? (false) | GPT Text Analysis |  |
| GPT Text Analysis | httpRequest | OpenAI text extraction call | Prepare GPT Request (Text) | Merge All Paths |  |
| Merge All Paths | merge | Consolidate extraction outputs | Parse Excel Data; GPT Vision Analysis; GPT Image Analysis; GPT Text Analysis | Parse All Results |  |
| Parse All Results | code | Parse/normalize all outputs to sheet rows | Merge All Paths | Filter Valid Data | ###  Google Sheets Output / Customize output schema in “Parse All Results”; update Document ID |
| Filter Valid Data | filter | Keep only valid output rows | Parse All Results | Save to Google Sheets; Extract Email IDs |  |
| Save to Google Sheets | googleSheets | Append rows | Filter Valid Data | Get Gmail Labels | ###  Google Sheets Output / Customize output schema in “Parse All Results”; update Document ID |
| Extract Email IDs | code | Unique email ids for labeling | Filter Valid Data | (used indirectly by Prepare Label Application) |  |
| Get Gmail Labels | httpRequest | List labels | Save to Google Sheets | Check Labels | ###  Gmail Labels / Processed_CategoryA/B/C; created automatically; customize names in code nodes |
| Check Labels | code | Determine missing labels | Get Gmail Labels | IF - Need Labels? | ###  Gmail Labels / Processed_CategoryA/B/C; created automatically; customize names in code nodes |
| IF - Need Labels? | if | Branch: create labels or not | Check Labels | Prepare Label Creation; Pass Existing Labels |  |
| Prepare Label Creation | code | Expand missing labels | IF - Need Labels? (true) | Filter Labels to Create |  |
| Filter Labels to Create | filter | Keep creatable labels | Prepare Label Creation | Create Gmail Label |  |
| Create Gmail Label | httpRequest | Create label(s) | Filter Labels to Create | Get New Label IDs |  |
| Get New Label IDs | code | Merge new+existing IDs | Create Gmail Label | Merge Label IDs |  |
| Pass Existing Labels | code | Pass label IDs as-is | IF - Need Labels? (false) | Merge Label IDs |  |
| Merge Label IDs | code | Consolidate label IDs | Get New Label IDs; Pass Existing Labels | Prepare Label Application |  |
| Prepare Label Application | code | Build modify requests per email | Merge Label IDs | Has Emails? | ###  Gmail Labels / Processed_CategoryA/B/C; created automatically; customize names in code nodes |
| Has Emails? | filter | Guard when no emails | Prepare Label Application | Apply Gmail Labels |  |
| Apply Gmail Labels | httpRequest | Apply label via modify | Has Emails? | — | ###  Gmail Labels / Processed_CategoryA/B/C; created automatically; customize names in code nodes |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow** named: *Extract TV ad slots from Gmail attachments to Google Sheets using GPT-4o Vision*.

2) **Add Variables (n8n Settings → Variables)**
   - `S3_BUCKET_NAME` = your bucket
   - `AWS_REGION` = e.g. `us-east-1`
   - `GOOGLE_SHEET_ID` = the Google Sheets document ID

3) **Create credentials**
   - **Gmail OAuth2** credential with permissions to read messages and modify labels.
   - **OpenAI API** credential (n8n OpenAI credential type) with a valid API key.
   - **AWS S3** credential (Access key/secret or IAM role) with permission to `s3:PutObject` on the bucket.
   - **Google Sheets OAuth2** credential with permission to edit the target sheet.
   - **HTTP Header Auth** credential for ConvertAPI:
     - Header should match ConvertAPI requirements (typically `Authorization: Bearer <secret>` or `x-api-secret: <secret>` depending on your setup).

4) **Add “Schedule Trigger”**
   - Interval: every **30 minutes**.

5) **Add “Gmail - Get Emails” (Gmail node)**
   - Operation: **Get Many (getAll)**
   - Limit: **1**
   - Filters:
     - Query: `-label:Processed_CategoryA -label:Processed_CategoryB -label:Processed_CategoryC`
     - Received after: expression `{{$now.minus({ days: 60 }).toISO()}}`
   - Connect: **Schedule Trigger → Gmail - Get Emails**

6) **Add “Domain Filter” (Code node, Run Once For Each Item)**
   - Paste the domain/classification script.
   - Customize:
     - `allowedDomains` (add domains)
     - `categoryAKeywords`, `categoryBKeywords`, optional `identifierCodes`
   - Connect: **Gmail - Get Emails → Domain Filter**

7) **Add “Filter Valid Emails” (Filter node)**
   - Condition: boolean `{{$json._skip}}` **not equals** `true`
   - Connect: **Domain Filter → Filter Valid Emails**

8) **Add “Gmail - Get Attachments” (Gmail node)**
   - Operation: **Get**
   - Message ID: `{{$json.id}}`
   - Options:
     - Download attachments: **true**
     - Attachment prefix: `attachment_`
   - Connect: **Filter Valid Emails → Gmail - Get Attachments**

9) **Add “Detect File Types” (Code node, Run Once For Each Item)**
   - Paste the detection script.
   - Connect: **Gmail - Get Attachments → Detect File Types**

10) **Add “IF - Has Excel?” (IF node)**
   - Condition: `{{$json.has_excel}}` equals `true`
   - Connect: **Detect File Types → IF - Has Excel?**

### Excel branch (true)
11) Add **“Extract from Excel”** (Extract From File)
   - Operation: `xlsx`
   - Binary property: `{{$json.excel_binary_key || 'attachment_0'}}`
   - Connect: **IF - Has Excel? (true) → Extract from Excel**

12) Add **“Parse Excel Data”** (Code node)
   - Paste the Excel parsing script and adapt column logic to your Excel layout.
   - Connect: **Extract from Excel → Parse Excel Data**

### Non-Excel branch (false): PPTX/PDF/variable Excel or Image/Text
13) Add **“IF - Has PPTX/PDF?”** (IF node)
   - Condition: `{{$json.has_pptx_pdf}}` equals `true`
   - Connect: **IF - Has Excel? (false) → IF - Has PPTX/PDF?**

#### PPTX/PDF branch (true)
14) Add **“ConvertAPI to PNG”** (HTTP Request)
   - Method: POST
   - URL: `https://v2.convertapi.com/convert/{{$json.conversion_type}}/to/png`
   - Content type: multipart/form-data
   - Body: form binary field named `File` from `{{$json.conversion_binary_key}}`
   - Auth: HTTP Header Auth (ConvertAPI secret)
   - Connect: **IF - Has PPTX/PDF? (true) → ConvertAPI to PNG**

15) Add **“Prepare for S3 Upload”** (Code node)
   - Paste script (splits pages up to 15).
   - Connect: **ConvertAPI to PNG → Prepare for S3 Upload**

16) Add **“S3 Upload (PPTX)”** (AWS S3 node)
   - Operation: Upload
   - Bucket: `{{$vars.S3_BUCKET_NAME}}`
   - File name: `{{$json._page_filename}}`
   - Connect: **Prepare for S3 Upload → S3 Upload (PPTX)**

17) Add **“Collect S3 URLs”** (Code node)
   - Paste script (constructs URLs using `S3_BUCKET_NAME` + `AWS_REGION`).
   - Connect: **S3 Upload (PPTX) → Collect S3 URLs**

18) Add **“Prepare GPT Request (PPTX)”** (Code node)
   - Paste script and customize:
     - system prompt fields
     - category-specific instructions
   - Connect: **Collect S3 URLs → Prepare GPT Request (PPTX)**

19) Add **“GPT Vision Analysis”** (HTTP Request)
   - POST `https://api.openai.com/v1/chat/completions`
   - Auth: OpenAI credential
   - Body (JSON): `{{$json.gpt_request_body}}`
   - Timeout: 300000 ms
   - Connect: **Prepare GPT Request (PPTX) → GPT Vision Analysis**

#### PPTX/PDF branch (false): Image/Text
20) Add **“IF - Has Image?”** (IF node)
   - Condition: `{{$json.has_image}}` equals `true`
   - Connect: **IF - Has PPTX/PDF? (false) → IF - Has Image?**

##### Image branch (true)
21) Add **“Prepare Image Metadata”** (Code node)
   - Paste script (sets `_image_key`, `_s3_filename`).
   - Connect: **IF - Has Image? (true) → Prepare Image Metadata**

22) Add **“S3 Upload (Image)”** (AWS S3 node)
   - Bucket: `{{$vars.S3_BUCKET_NAME}}`
   - File name: `{{$json._s3_filename}}`
   - Binary property: `{{$json._image_key}}`
   - Connect: **Prepare Image Metadata → S3 Upload (Image)**

23) Add **“Prepare GPT Request (Image)”** (Code node)
   - Paste script (uses S3 `Location` output).
   - Connect: **S3 Upload (Image) → Prepare GPT Request (Image)**

24) Add **“GPT Image Analysis”** (HTTP Request)
   - POST OpenAI chat completions
   - Auth: OpenAI credential
   - Body: `{{$json.gpt_request_body}}`
   - Connect: **Prepare GPT Request (Image) → GPT Image Analysis**

##### Text branch (false)
25) Add **“Prepare GPT Request (Text)”** (Code node)
   - Paste script; uses `textPlain` / `text`.
   - Connect: **IF - Has Image? (false) → Prepare GPT Request (Text)**

26) Add **“GPT Text Analysis”** (HTTP Request)
   - POST OpenAI chat completions
   - Auth: OpenAI credential
   - Body: `{{$json.gpt_request_body}}`
   - Connect: **Prepare GPT Request (Text) → GPT Text Analysis**

### Merge & normalize
27) Add **“Merge All Paths”** (Merge node)
   - Enable “Always Output Data” (or equivalent setting)
   - Connect into it from:
     - **Parse Excel Data**
     - **GPT Vision Analysis**
     - **GPT Image Analysis**
     - **GPT Text Analysis**

28) Add **“Parse All Results”** (Code node)
   - Paste normalization script.
   - Customize output schema and time-to-category logic as needed.
   - Connect: **Merge All Paths → Parse All Results**

29) Add **“Filter Valid Data”** (Filter node)
   - Condition: string `{{$json["email_type"]}}` not empty
   - Connect: **Parse All Results → Filter Valid Data**

30) Add **“Save to Google Sheets”** (Google Sheets node)
   - Operation: Append
   - Document ID: `{{$vars.GOOGLE_SHEET_ID}}`
   - Sheet name: `Sheet1`
   - Mapping: Auto-map input data
   - Connect: **Filter Valid Data → Save to Google Sheets**

### Gmail labeling
31) Add **“Extract Email IDs”** (Code node)
   - Paste script.
   - Connect: **Filter Valid Data → Extract Email IDs**  
   (In the provided workflow, Filter Valid Data outputs to both Sheets and this node.)

32) Add **“Get Gmail Labels”** (HTTP Request)
   - GET `https://gmail.googleapis.com/gmail/v1/users/me/labels`
   - Auth: Gmail OAuth2 credential
   - Connect: **Save to Google Sheets → Get Gmail Labels**

33) Add **“Check Labels”** (Code node)
   - Paste script; customize label names if desired.
   - Connect: **Get Gmail Labels → Check Labels**

34) Add **“IF - Need Labels?”** (IF node)
   - Condition: `{{$json.needs_creation}}` equals `true`
   - Connect: **Check Labels → IF - Need Labels?**

35) (True path) Add **Prepare Label Creation → Filter Labels to Create → Create Gmail Label → Get New Label IDs**
   - Use the same logic as in JSON; connect sequentially.

36) (False path) Add **Pass Existing Labels** (Code node)

37) Add **Merge Label IDs** (Code node) to unify both paths into `all_labels`.
   - Connect outputs of **Get New Label IDs** and **Pass Existing Labels** to it.

38) Add **Prepare Label Application** (Code node)
   - Paste script; adjust `typeToLabel` mapping if you add categories (e.g., NeedsReview).
   - Connect: **Merge Label IDs → Prepare Label Application**

39) Add **Has Emails?** (Filter node)
   - Condition: `{{$json._no_emails}}` not equals `true`
   - Connect: **Prepare Label Application → Has Emails?**

40) Add **Apply Gmail Labels** (HTTP Request)
   - POST `https://gmail.googleapis.com/gmail/v1/users/me/messages/{{$json.email_id}}/modify`
   - JSON body: `{{$json.request_body}}`
   - Auth: Gmail OAuth2 credential
   - Connect: **Has Emails? → Apply Gmail Labels**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ConvertAPI is used to convert PPTX/PDF to images for GPT Vision. | https://www.convertapi.com |
| Images are uploaded to S3 to be referenced by OpenAI via URL. Bucket accessibility must allow OpenAI to fetch `image_url`. | AWS S3 setup note |
| Processed emails are labeled `Processed_CategoryA/B/C`; labels are auto-created if missing. | Gmail Labels note |
| Customize parsing/output schema in `Parse All Results`; customize Excel parsing in `Parse Excel Data`; customize classification keywords/domains in `Domain Filter`. | Workflow sticky notes |
| Disclaimer (provided): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | User-provided compliance statement |