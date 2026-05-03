Process incoming files and notify via email with GitHub storage

https://n8nworkflows.xyz/workflows/process-incoming-files-and-notify-via-email-with-github-storage-13192


# Process incoming files and notify via email with GitHub storage

## 1. Workflow Overview

**Purpose:**  
This workflow runs on a daily schedule, polls an external HTTP endpoint for newly uploaded files, processes each file (CSV-only), validates the parsed records, stores valid output in a GitHub repository, and emails stakeholders with either a success or failure/exception outcome.

**Target use cases:**
- Daily ingestion of externally uploaded CSV files (e.g., exports, form uploads, partner drops)
- Lightweight ETL with simple validation rules
- “Always notify” operations model (success/failure emails)

### 1.1 Trigger & File Queue Retrieval
Runs on a fixed schedule, fetches a list of pending files, counts them, and branches based on whether any work exists.

### 1.2 Per-file Processing & Validation
Expands the queue into items, iterates through files, downloads each, ensures it’s CSV, parses CSV text into records, validates required fields.

### 1.3 Storage (GitHub) & Notifications (Email)
If validation succeeds, prepares commit data and creates/updates a file in GitHub, then sends a success email. Any failure path (no files, wrong file type, validation failure) sends an error email.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & File Queue Retrieval

**Overview:**  
Launches daily, calls an external API to retrieve pending uploads, computes the number of files, and decides whether to proceed or send an “error/empty” notification.

**Nodes involved:**
- Daily File Check
- Fetch File List
- Check File Count
- Any New Files?

#### Node: **Daily File Check**
- **Type / role:** `Schedule Trigger` — entry point; initiates workflow on a schedule.
- **Configuration (interpreted):** Runs every **24 hours** (interval rule).
- **Connections:**  
  - Output → **Fetch File List**
- **Edge cases / failures:**
  - n8n instance downtime can skip runs unless you implement catch-up logic externally.
  - Timezone is determined by n8n instance settings.

#### Node: **Fetch File List**
- **Type / role:** `HTTP Request` — retrieves pending file metadata.
- **Configuration (interpreted):**
  - GET `https://api.example.com/uploads/pending`
  - Expects JSON response shaped like: `{ "files": [ { id, url, fileName, mimeType }, ... ] }` (as assumed by downstream code).
- **Key expressions/variables:** none.
- **Connections:**  
  - Input ← Daily File Check  
  - Output → **Check File Count**
- **Edge cases / failures:**
  - Non-200 responses, timeouts, DNS/TLS errors.
  - Response shape mismatch (e.g., endpoint returns an array directly instead of `{files: ...}`) will cause downstream logic to treat it as empty.

#### Node: **Check File Count**
- **Type / role:** `Code` — normalizes response and computes `fileCount`.
- **Configuration (interpreted):**
  - Reads `const files = $json.files || []`
  - Outputs a single item: `{ files, fileCount: files.length }`
- **Key expressions/variables:**
  - `$json.files`
- **Connections:**  
  - Input ← Fetch File List  
  - Output → **Any New Files?**
- **Edge cases / failures:**
  - If the API returns `files` but it isn’t an array, `files.length` may not behave as expected.
  - If API returns the list at root (e.g., `[...]`), `files` becomes `[]` and the workflow will treat it as “no files”.

#### Node: **Any New Files?**
- **Type / role:** `IF` — branching: proceed only if files exist.
- **Configuration (interpreted):**
  - Condition: `fileCount > 0` using expression `={{ $json.fileCount }}`
- **Connections:**
  - **True** → Prepare File Items
  - **False** → Error Email Content (used here to notify “nothing to do” even though it’s not a technical error)
- **Edge cases / failures:**
  - If `fileCount` is missing or non-numeric, comparison can fail or default unexpectedly.

---

### Block 2 — Per-file Processing & Validation

**Overview:**  
Transforms the fetched queue into individual items, iterates through each file, downloads it, checks MIME type, parses CSV content, and validates records.

**Nodes involved:**
- Prepare File Items
- Iterate Files
- Download File
- Is CSV?
- Parse CSV
- Validate Data
- Validation Passed?

#### Node: **Prepare File Items**
- **Type / role:** `Code` — expands an array of files into individual n8n items (one item per file).
- **Configuration (interpreted):**
  - `return ($json.files || []).map(f => ({ json: f }));`
- **Key expressions/variables:**
  - `$json.files`
- **Connections:**
  - Input ← Any New Files? (true)
  - Output → Iterate Files
- **Edge cases / failures:**
  - If `files` is very large, execution memory/time may increase.
  - If items lack required fields (e.g., `url`), downstream nodes will fail.

#### Node: **Iterate Files**
- **Type / role:** `Split In Batches` — processes file items sequentially in batches.
- **Configuration (interpreted):**
  - Batch options left as default (so batch size defaults apply).
- **Connections:**
  - Input ← Prepare File Items
  - Output → Download File
- **Edge cases / failures:**
  - As wired in this workflow JSON, there is **no explicit loop-back connection** from downstream nodes to SplitInBatches to request the next batch. In many patterns, you connect the last node back to SplitInBatches’ “Next Batch” input. Without that, behavior may depend on n8n’s current batch defaults and may process only the first batch.
  - Consider setting an explicit batch size and adding the loop-back for deterministic iteration.

#### Node: **Download File**
- **Type / role:** `HTTP Request` — downloads file content from the file’s `url`.
- **Configuration (interpreted):**
  - URL: `={{ $json.url }}`
  - Output is assumed by the CSV parser to contain CSV text in `$json.body`.
- **Connections:**
  - Input ← Iterate Files
  - Output → Is CSV?
- **Edge cases / failures:**
  - If the endpoint returns binary, `$json.body` may not be plain text.
  - Missing/expired signed URLs, auth requirements, or large files can cause timeouts.
  - If response is not mapped to `body` as a string, Parse CSV will fail.

#### Node: **Is CSV?**
- **Type / role:** `IF` — routes only CSVs to parsing; everything else to error notification.
- **Configuration (interpreted):**
  - Checks: `={{ $json.mimeType || $json.type || 'unknown' }}` equals `"text/csv"`
- **Connections:**
  - **True** → Parse CSV
  - **False** → Error Email Content
- **Edge cases / failures:**
  - MIME type may be `"application/csv"`, `"text/plain"`, or include charset (`text/csv; charset=utf-8`), causing false negatives.
  - Note: after **Download File**, the item data may now be the HTTP response, not the original metadata. Unless the HTTP node merges original fields, `mimeType`/`fileName` may be missing at this point.

#### Node: **Parse CSV**
- **Type / role:** `Code` — parses CSV text into an array of record objects.
- **Configuration (interpreted):**
  - Reads `const csvText = $json.body;`
  - Splits lines by newline; first line = headers (comma-separated)
  - For each line: splits by comma, trims values, maps to headers
  - Outputs: `{ originalFileName: $json.fileName, records }`
- **Key expressions/variables:**
  - `$json.body`
  - `$json.fileName` (assumes fileName still present after Download File)
- **Connections:**
  - Input ← Is CSV? (true)
  - Output → Validate Data
- **Edge cases / failures:**
  - This is a **naïve CSV parser**:
    - Breaks on commas inside quoted fields
    - Breaks on multiline quoted fields
    - Doesn’t handle escaped quotes
  - If `$json.body` is undefined or not a string, `.trim()` throws.
  - If file is empty, `lines.shift()` can be undefined.

#### Node: **Validate Data**
- **Type / role:** `Code` — checks parsed records against business rules.
- **Configuration (interpreted):**
  - Uses `const items = $input.item.records;` and filters invalid records where `!r.id || !r.id.length`
  - Emits:
    - `data`: all records
    - `isValid`: boolean (true if no invalid records)
    - `errors`: array of invalid records
    - `originalFileName`
- **Key expressions/variables:**
  - `$input.item.records` and `$input.item.originalFileName` (non-standard access pattern; see edge case below)
- **Connections:**
  - Input ← Parse CSV
  - Output → Validation Passed?
- **Edge cases / failures:**
  - In n8n Code node, the most typical access is via `$json` (current item). Using `$input.item` is version/context dependent and may not work as intended. If `$input.item` is undefined, validation will throw.
  - Validation currently only checks presence of `id`. Other required fields/types are not enforced.

#### Node: **Validation Passed?**
- **Type / role:** `IF` — routes valid data to GitHub commit, invalid to error email.
- **Configuration (interpreted):**
  - Condition: `={{ $json.isValid }}` is true
- **Connections:**
  - **True** → Prepare GitHub Commit
  - **False** → Error Email Content
- **Edge cases / failures:**
  - If `isValid` missing/non-boolean, routing may be incorrect.

---

### Block 3 — Storage (GitHub) & Notifications (Email)

**Overview:**  
Formats content for GitHub storage, commits it to a repository, and then sends a success email. All failure branches converge into a set-and-send error email path.

**Nodes involved:**
- Prepare GitHub Commit
- Create/Update File
- Success Email Content
- Send Success Email
- Error Email Content
- Send Error Email

#### Node: **Prepare GitHub Commit**
- **Type / role:** `Set` — intended to assemble commit payload (path, content, message, branch).
- **Configuration (interpreted):**
  - No fields are actually defined in the provided JSON (node has default/empty parameters).
- **Connections:**
  - Input ← Validation Passed? (true)
  - Output → Create/Update File
- **Edge cases / failures:**
  - As configured, this node does not create required fields for the GitHub node (e.g., file path/content/message). The GitHub node may fail or do nothing depending on its defaults.
  - You will likely need to set:
    - repository file path (e.g., `data/{{$json.originalFileName}}.json`)
    - file content (base64 or text depending on node operation)
    - commit message

#### Node: **Create/Update File**
- **Type / role:** `GitHub` — commits a file to a GitHub repository.
- **Configuration (interpreted):**
  - Owner: `{{YOUR_GITHUB_USERNAME}}`
  - Repository: `{{YOUR_REPOSITORY_NAME}}`
  - Operation not fully specified in JSON snippet (but node name implies create/update file contents).
  - Requires GitHub OAuth/PAT credentials in n8n.
- **Connections:**
  - Input ← Prepare GitHub Commit
  - Output → Success Email Content
- **Version-specific notes:**
  - Node is `typeVersion: 1` (older); available operations/fields differ from newer versions.
- **Edge cases / failures:**
  - Auth failures (scopes for `repo` / contents write).
  - Branch/file path conflicts.
  - Missing required parameters (path/content/message).
  - Rate limits.

#### Node: **Success Email Content**
- **Type / role:** `Set` — intended to generate `toEmail`, `fromEmail`, `subject` (and typically `text`/`html`) for success.
- **Configuration (interpreted):**
  - No fields are defined in provided JSON (empty Set node).
- **Connections:**
  - Input ← Create/Update File
  - Output → Send Success Email
- **Edge cases / failures:**
  - If `subject/toEmail/fromEmail` aren’t set, Send Success Email will fail expression evaluation or send blank fields.
  - Consider also setting email body fields (`text` or `html`) explicitly.

#### Node: **Send Success Email**
- **Type / role:** `Email Send` — sends the success notification.
- **Configuration (interpreted):**
  - Subject: `={{ $json.subject }}`
  - To: `={{ $json.toEmail }}`
  - From: `={{ $json.fromEmail }}`
  - SMTP credentials required.
- **Connections:**
  - Input ← Success Email Content
- **Edge cases / failures:**
  - SMTP auth failures, relay restrictions.
  - Missing/invalid email addresses.
  - If no body is configured, some SMTP setups may reject.

#### Node: **Error Email Content**
- **Type / role:** `Set` — intended to craft error/empty-queue email fields.
- **Where it is used:** receives flow from:
  - Any New Files? (false) → “no files”
  - Is CSV? (false) → wrong type
  - Validation Passed? (false) → validation errors
- **Configuration (interpreted):**
  - No fields are defined in provided JSON (empty Set node).
- **Connections:**
  - Output → Send Error Email
- **Edge cases / failures:**
  - Without setting `subject/toEmail/fromEmail` (and ideally body), Send Error Email will fail or send unusable notifications.
  - Consider adding context fields like `$json.originalFileName`, `$json.errors`, and a reason code.

#### Node: **Send Error Email**
- **Type / role:** `Email Send` — sends failure/exception notification.
- **Configuration (interpreted):**
  - Subject/To/From from expressions (same pattern as success).
- **Connections:**
  - Input ← Error Email Content
- **Edge cases / failures:**
  - Same SMTP/address issues as success path.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / usage notes | — | — | ## How it works… (includes setup steps 1–6) |
| Trigger & Fetch | Sticky Note | Documentation for retrieval block | — | — | ## Trigger & Retrieval… |
| Processing & Validation | Sticky Note | Documentation for processing block | — | — | ## Processing & Validation… |
| Storage & Notification | Sticky Note | Documentation for storage/notification block | — | — | ## Storage & Notification… |
| Daily File Check | Schedule Trigger | Scheduled entry point | — | Fetch File List | ## Trigger & Retrieval… |
| Fetch File List | HTTP Request | Pull pending files list | Daily File Check | Check File Count | ## Trigger & Retrieval… |
| Check File Count | Code | Normalize list + count | Fetch File List | Any New Files? | ## Trigger & Retrieval… |
| Any New Files? | IF | Branch on fileCount | Check File Count | Prepare File Items; Error Email Content | ## Trigger & Retrieval… |
| Prepare File Items | Code | Expand array → items | Any New Files? (true) | Iterate Files | ## Processing & Validation… |
| Iterate Files | SplitInBatches | Per-file iteration control | Prepare File Items | Download File | ## Processing & Validation… |
| Download File | HTTP Request | Download file contents | Iterate Files | Is CSV? | ## Processing & Validation… |
| Is CSV? | IF | Route only CSV | Download File | Parse CSV; Error Email Content | ## Processing & Validation… |
| Parse CSV | Code | Parse CSV text → records | Is CSV? (true) | Validate Data | ## Processing & Validation… |
| Validate Data | Code | Validate record rules | Parse CSV | Validation Passed? | ## Processing & Validation… |
| Validation Passed? | IF | Branch valid/invalid | Validate Data | Prepare GitHub Commit; Error Email Content | ## Processing & Validation… |
| Prepare GitHub Commit | Set | Build commit payload | Validation Passed? (true) | Create/Update File | ## Storage & Notification… |
| Create/Update File | GitHub | Commit file to repo | Prepare GitHub Commit | Success Email Content | ## Storage & Notification… |
| Success Email Content | Set | Build success email fields | Create/Update File | Send Success Email | ## Storage & Notification… |
| Send Success Email | Email Send | Send success notification | Success Email Content | — | ## Storage & Notification… |
| Error Email Content | Set | Build error email fields | Any New Files? (false); Is CSV? (false); Validation Passed? (false) | Send Error Email | ## Storage & Notification… |
| Send Error Email | Email Send | Send failure/empty notification | Error Email Content | — | ## Storage & Notification… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **File Processing Pipeline with Email and GitHub**
   - (Optional) Add sticky notes with the same four headings to document each section.

2. **Add trigger**
   - Add **Schedule Trigger** node named **Daily File Check**
   - Set interval to **every 24 hours** (Hours interval = 24)

3. **Fetch pending file list**
   - Add **HTTP Request** node named **Fetch File List**
   - Method: GET
   - URL: `https://api.example.com/uploads/pending`
   - Connect: **Daily File Check → Fetch File List**
   - Ensure the endpoint returns JSON shaped like:  
     `{"files":[{"id":"...","url":"...","fileName":"...","mimeType":"text/csv"}]}`

4. **Count files**
   - Add **Code** node named **Check File Count**
   - Paste code (adapt if your API returns an array directly):
     - Reads `$json.files || []`
     - Outputs `{ files, fileCount }`
   - Connect: **Fetch File List → Check File Count**

5. **Branch if any files exist**
   - Add **IF** node named **Any New Files?**
   - Condition (Number): `{{$json.fileCount}}` **is larger than** `0`
   - Connect: **Check File Count → Any New Files?**

6. **Expand into per-file items**
   - Add **Code** node named **Prepare File Items**
   - Code: map `files` array into items (`{json: f}`)
   - Connect: **Any New Files? (true) → Prepare File Items**

7. **Iterate items**
   - Add **Split In Batches** node named **Iterate Files**
   - Keep defaults or set Batch Size explicitly (e.g., 1 or 10)
   - Connect: **Prepare File Items → Iterate Files**
   - Recommended for reliability: add a “next batch” loop-back later (see note below).

8. **Download each file**
   - Add **HTTP Request** node named **Download File**
   - URL: `{{$json.url}}`
   - Configure response format so the CSV content is accessible to parsing (commonly “String”/text, not binary).
   - Connect: **Iterate Files → Download File**

9. **Check file type**
   - Add **IF** node named **Is CSV?**
   - Condition (String): `{{$json.mimeType || $json.type || 'unknown'}}` equals `text/csv`
   - Connect: **Download File → Is CSV?**

10. **Parse CSV**
    - Add **Code** node named **Parse CSV**
    - Implement CSV parsing (or replace with a more robust approach).
    - Ensure it outputs:
      - `originalFileName`
      - `records` array
    - Connect: **Is CSV? (true) → Parse CSV**

11. **Validate records**
    - Add **Code** node named **Validate Data**
    - Implement rule(s) (current rule: record must have `id`)
    - Output:
      - `isValid` boolean
      - `errors` list
      - `data` list
      - `originalFileName`
    - Connect: **Parse CSV → Validate Data**

12. **Route valid vs invalid**
    - Add **IF** node named **Validation Passed?**
    - Condition (Boolean): `{{$json.isValid}}` is true
    - Connect: **Validate Data → Validation Passed?**

13. **Prepare GitHub commit payload**
    - Add **Set** node named **Prepare GitHub Commit**
    - Add fields you will use in the GitHub node, typically:
      - `path` (example: `processed/{{$json.originalFileName}}.json`)
      - `content` (example: `{{ JSON.stringify($json.data, null, 2) }}`)
      - `message` (example: `Processed {{$json.originalFileName}}`)
      - `branch` (example: `main`)
    - Connect: **Validation Passed? (true) → Prepare GitHub Commit**

14. **Commit to GitHub**
    - Add **GitHub** node named **Create/Update File**
    - Configure credentials: **GitHub OAuth2** (or PAT) with permission to write repository contents.
    - Set Owner/Repository (replace placeholders):
      - Owner: your GitHub username/org
      - Repository: target repo name
    - Select the operation that creates/updates a file and map in:
      - file path, content, commit message, branch (as required by your GitHub node version)
    - Connect: **Prepare GitHub Commit → Create/Update File**

15. **Success email content**
    - Add **Set** node named **Success Email Content**
    - Set:
      - `toEmail`, `fromEmail`, `subject`
      - (Recommended) `text` or `html` body with details (file name, record count, commit path)
    - Connect: **Create/Update File → Success Email Content**

16. **Send success email**
    - Add **Email Send** node named **Send Success Email**
    - Configure SMTP credentials in n8n (host/port/user/pass or OAuth2 depending on your provider).
    - Map:
      - To: `{{$json.toEmail}}`
      - From: `{{$json.fromEmail}}`
      - Subject: `{{$json.subject}}`
      - Body: map from your `text/html` field
    - Connect: **Success Email Content → Send Success Email**

17. **Error notification path**
    - Add **Set** node named **Error Email Content**
    - Populate:
      - `toEmail`, `fromEmail`, `subject`
      - `text/html` including context, e.g.:
        - “No new files” (from Any New Files? false)
        - “Wrong MIME type” (from Is CSV? false)
        - “Validation errors” plus `$json.errors`
    - Connect error branches:
      - **Any New Files? (false) → Error Email Content**
      - **Is CSV? (false) → Error Email Content**
      - **Validation Passed? (false) → Error Email Content**

18. **Send error email**
    - Add **Email Send** node named **Send Error Email**
    - Map To/From/Subject (and body) from Error Email Content output.
    - Connect: **Error Email Content → Send Error Email**

19. **(Strongly recommended) Make SplitInBatches deterministic**
    - Add a connection from the end of each per-file branch back to **Iterate Files** “Next Batch” input (pattern depends on your exact n8n version/UI).  
    - This ensures every file in the queue is processed, not just the first batch.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Create an HTTP endpoint that returns a JSON array of pending files (id, url, fileName, mimeType).” | From sticky note “Workflow Overview” (setup step 1). (Implementation note: current code expects `{ files: [...] }`, so align response or update `Check File Count`.) |
| “Add GitHub OAuth credentials in n8n and replace the owner/repo placeholders.” | From sticky note “Workflow Overview” (setup step 2). |
| “Configure an SMTP credential for the Email Send node and update the to/from addresses.” | From sticky note “Workflow Overview” (setup step 3). |
| “Adjust the schedule trigger interval to suit your cadence.” | From sticky note “Workflow Overview” (setup step 4). |
| “Edit validation rules inside the Validate Data Code node as needed.” | From sticky note “Workflow Overview” (setup step 5). |
| “(Optional) Tweak commit paths, branch names or email templates for your environment.” | From sticky note “Workflow Overview” (setup step 6). |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.