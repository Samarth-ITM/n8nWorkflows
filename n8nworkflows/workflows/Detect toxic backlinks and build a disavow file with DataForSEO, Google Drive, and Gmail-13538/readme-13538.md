Detect toxic backlinks and build a disavow file with DataForSEO, Google Drive, and Gmail

https://n8nworkflows.xyz/workflows/detect-toxic-backlinks-and-build-a-disavow-file-with-dataforseo--google-drive--and-gmail-13538


# Detect toxic backlinks and build a disavow file with DataForSEO, Google Drive, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow retrieves “toxic” backlinks for a target domain from the **DataForSEO Backlinks API** (filtered by spam score), compiles them into a Google-compliant **disavow.txt** text file, uploads it to **Google Drive**, then emails a **Gmail** recipient a link to the generated file.

**Target use cases:**
- SEO teams auditing backlinks and preparing Google Search Console disavow submissions.
- Automating repeated disavow file generation with consistent thresholds and delivery.

### 1.1 Manual Start & State Initialization
Starts on demand and initializes an accumulator array (`items`) used to store toxic backlink URLs across pages.

### 1.2 DataForSEO Retrieval & Safety Checks
Requests backlinks (1000 per page), filters by spam score > 50, checks if total results are within a practical limit (< 100k). If too many, sends an error email.

### 1.3 Pagination / Aggregation
Determines if more pages exist; merges results into the `items` array.

### 1.4 File Build, Size Validation, Drive Upload, Email Delivery
Builds newline-separated text, checks file size (<2MB), creates `disavow.txt` on Google Drive, and emails a Drive link. If file is too large, sends an error email.

---

## 2. Block-by-Block Analysis

### Block 1 — Manual trigger & initialize accumulator
**Overview:**  
Creates the initial workflow run context and initializes an empty `items` array to accumulate toxic backlink URLs.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Initialize "items" field

#### Node: When clicking ‘Execute workflow’
- **Type / role:** `Manual Trigger` — manual entry point.
- **Configuration:** No parameters.
- **Inputs/Outputs:**  
  - Output → **Initialize "items" field**
- **Failure modes:** None (only runs when executed manually).
- **Version notes:** typeVersion 1.

#### Node: Initialize "items" field
- **Type / role:** `Set` — initializes a state field.
- **Configuration choices:**  
  - Sets `items` (array) to `[]`.
- **Key variables/expressions:** `={{ [] }}`
- **Inputs/Outputs:**  
  - Input ← Manual Trigger  
  - Output → **Set "items" field**
- **Failure modes:** Minimal; expression evaluation issues are unlikely here.
- **Version notes:** Set node typeVersion 3.4.

---

### Block 2 — Prepare iteration state & call DataForSEO
**Overview:**  
Copies the running `items` array forward, then calls DataForSEO to fetch spammy backlinks page-by-page (1000 per page).

**Nodes involved:**
- Set "items" field
- Get spam backlinks

#### Node: Set "items" field
- **Type / role:** `Set` — ensures `items` is present for downstream nodes.
- **Configuration choices:**  
  - Sets `items` to the current item’s `items`.
- **Key expressions:** `={{ $json.items }}`
- **Inputs/Outputs:**  
  - Input ← Initialize "items" field **or** Has more pages? (true path loops back)  
  - Output → Get spam backlinks
- **Failure modes:** If upstream doesn’t contain `items`, `$json.items` may be `undefined` (later merges may fail).
- **Version notes:** typeVersion 3.4.

#### Node: Get spam backlinks
- **Type / role:** `DataForSEO Backlinks API` node — fetches backlinks from DataForSEO.
- **Configuration choices (interpreted):**
  - **Operation:** `get-backlinks`
  - **Target domain:** `dataforseo.com` (hardcoded; should be changed to your domain)
  - **Filter:** backlinks where `backlink_spam_score > 50`
  - **Pagination:** `limit = 1000`, `offset = $runIndex * 1000`
  - **Indirect links:** `include_indirect_links = false`
- **Key expressions:**
  - `offset: ={{ $runIndex * 1000 }}`
  - `filters: ["backlink_spam_score", ">", 50]`
- **Inputs/Outputs:**
  - Input ← Set "items" field
  - Output → Less than 100K links?
- **Credential requirements:** DataForSEO API credentials.
- **Failure modes / edge cases:**
  - Auth/permission errors (wrong login/password).
  - API quota/rate limiting.
  - Target domain format issues (domain vs URL).
  - Empty results: `tasks[0].result[0].items` may be missing or empty; downstream expressions should handle safely.
- **Version notes:** typeVersion 1.

---

### Block 3 — Volume guardrail (< 100k) & early exit
**Overview:**  
Prevents generating an excessively large disavow list by checking total backlink count before aggregation continues.

**Nodes involved:**
- Less than 100K links?
- Send a message (too many links)
- Merge "items" with DFS response

#### Node: Less than 100K links?
- **Type / role:** `IF` — validates DataForSEO response size.
- **Configuration choices:**
  - Condition: `tasks[0].result[0].total_count < 100000`
- **Key expressions:** `={{ $json.tasks[0].result[0].total_count }}`
- **Branching:**
  - **True** → Merge "items" with DFS response
  - **False** → Send a message (too many links)
- **Failure modes:**
  - If DataForSEO response shape differs or `tasks[0]...` is missing, the expression can error.
- **Version notes:** IF typeVersion 2.3.

#### Node: Send a message (too many links)
- **Type / role:** `Gmail` — sends an error email when too many links.
- **Configuration choices:**
  - To: `user@example.com` (replace)
  - Subject includes target and date
  - Plain text body
- **Key expressions:**
  - Subject: `Disavow File Error – {{ $json.tasks[0].data.target }} – {{ new Date().toDateTime().format('yyyy-MM-dd') }}`
- **Inputs/Outputs:**
  - Input ← IF (false path)
  - Output: none
- **Credential requirements:** Gmail OAuth2 credential.
- **Failure modes:** OAuth expiry, insufficient scopes, Gmail sending limits.
- **Version notes:** Gmail typeVersion 2.2.

#### Node: Merge "items" with DFS response
- **Type / role:** `Set` — merges accumulated `items` with the current DataForSEO page.
- **Intended configuration (interpreted):**
  - `items = [...previousItems, ...currentPageUrls]`
- **Actual expression present (important):**
  - `={{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items.map(item => imtem.url_from)] }}`
- **Critical issue (will break):**
  - Typo: `imtem.url_from` should be `item.url_from`.
- **Inputs/Outputs:**
  - Input ← IF (true path)
  - Output → Has more pages?
- **Failure modes / edge cases:**
  - The typo causes a runtime ReferenceError, stopping the workflow.
  - If `items` is missing on either side, spread may fail.
  - If `result[0].items` is empty/undefined, `.map()` may fail.
- **Version notes:** Set typeVersion 3.4.

---

### Block 4 — Pagination decision & final aggregation
**Overview:**  
Determines whether there are more DataForSEO pages to fetch. If yes, loops back to fetch the next page. If no, merges the final page and proceeds to file generation.

**Nodes involved:**
- Has more pages?
- Merge "items" with last response
- (Loop back) Set "items" field

#### Node: Has more pages?
- **Type / role:** `IF` — pagination controller.
- **Configuration choices:**
  - Condition: `$runIndex < (total_count / 1000 - 1)`
- **Key expressions:**
  - Left: `={{ $runIndex }}`
  - Right: `={{ $('Get spam backlinks').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
- **Branching:**
  - **True** → Set "items" field (loops, triggers next API call; offset increases via `$runIndex`)
  - **False** → Merge "items" with last response
- **Important behavior note:** `$runIndex` increments on each loop iteration (each time the node executes within the run).
- **Failure modes / edge cases:**
  - Off-by-one errors if total_count is not a multiple of 1000.
  - If `total_count` missing, expression fails.
- **Version notes:** IF typeVersion 2.3.

#### Node: Merge "items" with last response
- **Type / role:** `Set` — final merge before file creation.
- **Configuration choices:**
  - `items = [...previousItems, ...currentItems.url_from]`
- **Key expressions:**
  - `={{ [...$('Set "items" field').item.json.items, ... $('Get spam backlinks').item.json.tasks[0].result[0].items.map(item => item.url_from)]}}`
- **Inputs/Outputs:**
  - Input ← Has more pages? (false path)
  - Output → Calculate file size
- **Failure modes:** Same as other merge: missing arrays, missing `items`, missing `url_from`.
- **Version notes:** Set typeVersion 3.4.

---

### Block 5 — Build disavow text, validate file size, upload, email
**Overview:**  
Transforms the aggregated URL list into newline-separated text, checks it’s under 2MB (Google Search Console guideline), uploads it to Google Drive, and emails the file link.

**Nodes involved:**
- Calculate file size
- File size < 2MB
- Create file from text
- Send a message (success)
- Send a message (file too large)

#### Node: Calculate file size
- **Type / role:** `Code` — constructs the disavow text and computes size in bytes.
- **Configuration choices:**
  - `text = $json.items.join("\n")`
  - `fileSize = Buffer.byteLength(text, 'utf8')`
  - Outputs `{ text, fileSize }`
- **Inputs/Outputs:**
  - Input ← Merge "items" with last response
  - Output → File size < 2MB
- **Failure modes:**
  - If `$json.items` is not an array, `.join()` fails.
- **Version notes:** Code node typeVersion 2; uses Node.js `Buffer` (supported in n8n Code node).

#### Node: File size < 2MB
- **Type / role:** `IF` — validates file size.
- **Configuration choices:**
  - Condition: `$json.fileSize < 2000000`
- **Branching:**
  - **True** → Create file from text
  - **False** → Send a message (file too large)
- **Failure modes:** `fileSize` missing or non-numeric.
- **Version notes:** IF typeVersion 2.3.

#### Node: Create file from text
- **Type / role:** `Google Drive` — uploads text content as a file.
- **Configuration choices:**
  - Operation: `createFromText`
  - File name: `disavow.txt`
  - Content: `={{ $json.text }}`
  - Drive: “My Drive”
  - Folder: `root` (can be changed)
- **Inputs/Outputs:**
  - Input ← File size < 2MB (true)
  - Output → Send a message (success)
- **Credential requirements:** Google Drive OAuth2 credential (Drive file create permission).
- **Failure modes:** OAuth expiry, insufficient Drive permissions, wrong folder/drive selection.
- **Version notes:** Google Drive node typeVersion 3.

#### Node: Send a message (success)
- **Type / role:** `Gmail` — sends HTML email with Drive link.
- **Configuration choices:**
  - To: `user@example.com` (replace)
  - HTML body includes: `https://drive.google.com/file/d/{{ $json.id }}`
  - Subject includes target domain and date
- **Key expressions:**
  - Subject uses: `{{ $('Get spam backlinks').item.json.tasks[0].data.target }}`
  - Link uses Drive file id: `{{ $json.id }}`
- **Inputs/Outputs:**
  - Input ← Create file from text
  - Output: none
- **Failure modes:** Gmail OAuth, HTML formatting, Drive link permissions (recipient must have access if Drive file is not public/shared).
- **Version notes:** Gmail typeVersion 2.2.

#### Node: Send a message (file too large)
- **Type / role:** `Gmail` — sends error if file exceeds 2MB.
- **Configuration choices:**
  - To: `user@example.com`
  - Plain text body
  - Subject includes target domain and date
- **Inputs/Outputs:**
  - Input ← File size < 2MB (false)
- **Failure modes:** Gmail OAuth, rate limits.
- **Version notes:** Gmail typeVersion 2.2.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Manual entry point | — | Initialize "items" field | This workflow retrieves toxic backlinks pointing to your domain using the DataForSEO Backlinks API, automatically generates a Google-compliant disavow file, and emails you a link to the file, ready for review and upload into Google Search Console. |
| Initialize "items" field | set | Initialize accumulator array | When clicking ‘Execute workflow’ | Set "items" field | This workflow retrieves toxic backlinks pointing to your domain using the DataForSEO Backlinks API, automatically generates a Google-compliant disavow file, and emails you a link to the file, ready for review and upload into Google Search Console. |
| Set "items" field | set | Carry forward items array / loop state | Initialize "items" field; Has more pages? (true) | Get spam backlinks | Get spam backlinks with DataForSEO; Create a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed. Create a Gmail connection and set a receiver. |
| Get spam backlinks | dataForSeoBacklinksApi | Fetch backlinks filtered by spam score | Set "items" field | Less than 100K links? | Get spam backlinks with DataForSEO; Create a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed. Create a Gmail connection and set a receiver. |
| Less than 100K links? | if | Guardrail on total backlinks | Get spam backlinks | Merge "items" with DFS response (true); Send a message (too many links) (false) | Get spam backlinks with DataForSEO; Create a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed. Create a Gmail connection and set a receiver. |
| Merge "items" with DFS response | set | Merge accumulator with current page | Less than 100K links? (true) | Has more pages? | Get spam backlinks with DataForSEO; Create a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed. Create a Gmail connection and set a receiver. |
| Send a message (too many links) | gmail | Error email when total_count >= 100k | Less than 100K links? (false) | — | Get spam backlinks with DataForSEO; Create a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed. Create a Gmail connection and set a receiver. |
| Has more pages? | if | Pagination decision / loop control | Merge "items" with DFS response | Set "items" field (true); Merge "items" with last response (false) | Get spam backlinks with DataForSEO; Create a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed. Create a Gmail connection and set a receiver. |
| Merge "items" with last response | set | Final merge before file build | Has more pages? (false) | Calculate file size | Generate a disavow file and send it via email; Create a Google Drive connection and set a destination folder. Create a Gmail connection and set a receiver. |
| Calculate file size | code | Build text and compute bytes | Merge "items" with last response | File size < 2MB | Generate a disavow file and send it via email; Create a Google Drive connection and set a destination folder. Create a Gmail connection and set a receiver. |
| File size < 2MB | if | Validate disavow file size | Calculate file size | Create file from text (true); Send a message (file too large) (false) | Generate a disavow file and send it via email; Create a Google Drive connection and set a destination folder. Create a Gmail connection and set a receiver. |
| Create file from text | googleDrive | Upload disavow.txt to Drive | File size < 2MB (true) | Send a message (success) | Generate a disavow file and send it via email; Create a Google Drive connection and set a destination folder. Create a Gmail connection and set a receiver. |
| Send a message (success) | gmail | Email Drive link to disavow file | Create file from text | — | Generate a disavow file and send it via email; Create a Google Drive connection and set a destination folder. Create a Gmail connection and set a receiver. |
| Send a message (file too large) | gmail | Error email when size >= 2MB | File size < 2MB (false) | — | Generate a disavow file and send it via email; Create a Google Drive connection and set a destination folder. Create a Gmail connection and set a receiver. |
| Sticky Note | stickyNote | Comment / instructions | — | — | ## Get spam backlinks with DataForSEO\nCreate a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed.\nCreate a Gmail connection and set a receiver. |
| Sticky Note4 | stickyNote | Comment / instructions | — | — | ## Generate a disavow file and send it via email\nCreate a Google Drive connection and set a destination folder.\nCreate a Gmail connection and set a receiver. |
| Sticky Note1 | stickyNote | Comment / workflow description | — | — | This workflow retrieves toxic backlinks pointing to your domain using the DataForSEO Backlinks API, automatically generates a Google-compliant disavow file, and emails you a link to the file, ready for review and upload into Google Search Console.\n\n## How it works\n1. Triggers manually.\n2. Fetches backlinks for your domain using the DataForSEO Backlinks API.\n3. Filters backlinks by spam score above the threshold (default: >50).\n4. Extracts toxic backlinks along with their key metrics.\n5. Formats all suspicious links into a disavow.txt file aligned with Google’s rules.\n6. Uploads the file to your Google Drive.\n7. Emails you a link for checking the file.\n\n## Setup steps\n1. Create or select your DataForSEO connection (use [your API login and password](https://app.dataforseo.com/api-access)).\n2. Indicate your target domain.\n3. Connect Google Drive and choose a folder.\n4. Connect Gmail and choose who gets the message. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: *Detect toxic backlinks and build a disavow file in one click with DataForSEO*.

2. **Add node: Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add node: Set (initialize accumulator)**
   - Node: **Set**
   - Name: `Initialize "items" field`
   - Add field:
     - `items` (Type: Array) = `[]`
   - Connect: Manual Trigger → Initialize "items" field

4. **Add node: Set (carry state)**
   - Node: **Set**
   - Name: `Set "items" field`
   - Add field:
     - `items` (Array) = expression `{{$json.items}}`
   - Connect: Initialize "items" field → Set "items" field

5. **Add node: DataForSEO Backlinks API**
   - Node: **DataForSEO Backlinks API** (n8n DataForSEO integration)
   - Name: `Get spam backlinks`
   - Operation: `get-backlinks`
   - Parameters:
     - `target`: your domain (replace `dataforseo.com`)
     - `filters`: `["backlink_spam_score", ">", 50]` (adjust threshold as needed)
     - `limit`: `1000`
     - `offset`: expression `{{$runIndex * 1000}}`
     - `include_indirect_links`: false
   - Credentials:
     - Create **DataForSEO API** credential using login/password from: https://app.dataforseo.com/api-access
   - Connect: Set "items" field → Get spam backlinks

6. **Add node: IF (total_count guardrail)**
   - Node: **IF**
   - Name: `Less than 100K links?`
   - Condition (Number):
     - Left: `{{$json.tasks[0].result[0].total_count}}`
     - Operation: `is less than`
     - Right: `100000`
   - Connect: Get spam backlinks → Less than 100K links?

7. **Add node: Gmail (too many links)**
   - Node: **Gmail**
   - Name: `Send a message (too many links)`
   - Send To: your recipient email
   - Email Type: Text
   - Subject (expression):  
     `Disavow File Error – {{ $json.tasks[0].data.target }} – {{ new Date().toDateTime().format('yyyy-MM-dd') }}`
   - Message: “You have too many disavow links…”
   - Credentials: create/connect **Gmail OAuth2**
   - Connect: Less than 100K links? (false output) → this Gmail node

8. **Add node: Set (merge page items)**
   - Node: **Set**
   - Name: `Merge "items" with DFS response`
   - Add field `items` (Array) with expression (corrected):  
     `{{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items.map(item => item.url_from) ] }}`
   - Connect: Less than 100K links? (true output) → Merge "items" with DFS response  
   - **Important:** ensure `item.url_from` spelling is correct (the provided JSON contains a breaking typo).

9. **Add node: IF (pagination)**
   - Node: **IF**
   - Name: `Has more pages?`
   - Condition (Number):
     - Left: `{{$runIndex}}`
     - Operation: `is less than`
     - Right: `{{ $('Get spam backlinks').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
   - Connect: Merge "items" with DFS response → Has more pages?

10. **Loop connection**
   - Connect: Has more pages? (true output) → Set "items" field  
   This causes another call to DataForSEO with a higher offset due to `$runIndex`.

11. **Add node: Set (final merge)**
   - Node: **Set**
   - Name: `Merge "items" with last response`
   - Field `items` expression:  
     `{{ [...$('Set "items" field').item.json.items, ...$('Get spam backlinks').item.json.tasks[0].result[0].items.map(item => item.url_from)] }}`
   - Connect: Has more pages? (false output) → Merge "items" with last response

12. **Add node: Code (build text + size)**
   - Node: **Code**
   - Name: `Calculate file size`
   - JavaScript:
     - Join `items` with `\n`
     - Compute bytes via `Buffer.byteLength(text, 'utf8')`
     - Return `{ text, fileSize }`
   - Connect: Merge "items" with last response → Calculate file size

13. **Add node: IF (size check)**
   - Node: **IF**
   - Name: `File size < 2MB`
   - Condition:
     - Left: `{{$json.fileSize}}`
     - Operation: `is less than`
     - Right: `2000000`
   - Connect: Calculate file size → File size < 2MB

14. **Add node: Gmail (file too large)**
   - Node: **Gmail**
   - Name: `Send a message (file too large)`
   - To: your recipient
   - Email Type: Text
   - Subject (expression):  
     `Disavow File Error – {{ $('Get spam backlinks').item.json.tasks[0].data.target }} – {{ new Date().toDateTime().format('yyyy-MM-dd') }}`
   - Connect: File size < 2MB (false output) → this node

15. **Add node: Google Drive (create disavow.txt)**
   - Node: **Google Drive**
   - Name: `Create file from text`
   - Operation: `Create From Text`
   - File name: `disavow.txt`
   - Content: `{{$json.text}}`
   - Choose Drive: “My Drive”
   - Choose Folder: destination folder (or root)
   - Credentials: **Google Drive OAuth2**
   - Connect: File size < 2MB (true output) → Create file from text

16. **Add node: Gmail (success)**
   - Node: **Gmail**
   - Name: `Send a message (success)`
   - To: your recipient
   - Subject (expression):  
     `Disavow File Generated – {{ $('Get spam backlinks').item.json.tasks[0].data.target }} – {{ new Date().toDateTime().format('yyyy-MM-dd') }}`
   - Message (HTML) containing link:  
     `https://drive.google.com/file/d/{{ $json.id }}`
   - Connect: Create file from text → Send a message (success)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DataForSEO credentials are created using API login/password from DataForSEO account | https://app.dataforseo.com/api-access |
| Workflow intent: generate a disavow.txt, upload to Drive, email link; review before uploading to Google Search Console | From sticky note description |
| Known implementation issue: typo `imtem.url_from` in “Merge "items" with DFS response” must be corrected to `item.url_from` | Prevents runtime failure |
| Google Drive link access depends on file permissions; recipient may need Drive access | Operational consideration |

