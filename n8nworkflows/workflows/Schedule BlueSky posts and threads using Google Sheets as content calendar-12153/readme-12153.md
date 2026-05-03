Schedule BlueSky posts and threads using Google Sheets as content calendar

https://n8nworkflows.xyz/workflows/schedule-bluesky-posts-and-threads-using-google-sheets-as-content-calendar-12153


# Schedule BlueSky posts and threads using Google Sheets as content calendar

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Schedule BlueSky posts and threads using Google Sheets as content calendar  
**Workflow name (in JSON):** BlueSky Suite: Schedule BlueSky posts and threads from Google Sheets  
**Purpose:** Periodically read a Google Sheet used as a content calendar, pick rows marked **Ready** whose **Scheduled Time** has passed (in a chosen timezone), then publish them to **BlueSky** as single posts or threaded replies (based on **Thread ID** + **Sequence**), optionally attaching an image from an external URL, and finally update the sheet row with **Posted**, **Posted At**, and a **Post Link**.

### 1.1 Input & Scheduling
- Runs on a time interval (every 15 minutes).
- Loads local configuration (BlueSky handle, app password, timezone).

### 1.2 Authentication (BlueSky session)
- Logs into BlueSky to obtain an `accessJwt` (Bearer token) and user DID.

### 1.3 Sheet Retrieval & Eligibility Filtering
- Fetches rows from a Google Sheet.
- Keeps only rows with:
  - `Status = "Ready"`
  - non-empty `Scheduled Time`
  - `Scheduled Time` <= “now” in configured timezone

### 1.4 Thread Ordering & Batch Processing
- Sorts rows by `Thread ID`, then `Sequence` to ensure proper thread order.
- Processes rows **one-by-one** (batch loop) to preserve thread state and ensure sequential replies.

### 1.5 Optional Image Handling
- If `Image URL` is present:
  - downloads the image
  - uploads it to BlueSky to obtain an image “blob”
  - merges blob back with the original row data

### 1.6 Payload Construction, Posting, Thread Memory, Sheet Update
- Builds the BlueSky post payload (text, optional embed, optional reply root/parent).
- Creates the post via BlueSky XRPC.
- Updates “thread memory” (root/parent refs) in workflow static data.
- Updates the originating Google Sheet row with posting metadata.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule + Local Configuration
**Overview:** Triggers the workflow periodically and provides the BlueSky credentials + timezone used in later nodes.  
**Nodes involved:** `Schedule Trigger`, `Configuration`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entry point timer.
- **Config (interpreted):** Runs every **15 minutes**.
- **Connections:** → `Configuration`
- **Edge cases / failures:**
  - If n8n instance is down, scheduled executions are missed.
  - High frequency can overlap executions if posting takes longer than the interval (depending on n8n concurrency settings).
- **Sticky notes (applies):**
  - **# Inputs**
  - From “How To Use”: “Turn workflow Active… run every hour (or your set interval)” (note: the actual interval here is 15 minutes).

#### Node: Configuration
- **Type / role:** `n8n-nodes-base.set` — central parameter store.
- **Config (interpreted):**
  - `bluesky_handle` (string): user handle (e.g. `steve.bsky.social`)
  - `app_password` (string): BlueSky App Password
  - `timezone` (string): defaults to `Asia/Kolkata`
- **Key expressions used later:**
  - `$('Configuration').first().json.bluesky_handle`
  - `$('Configuration').first().json.app_password`
  - `$('Configuration').first().json.timezone`
- **Connections:** receives from `Schedule Trigger` → outputs to `BlueSky Auth`
- **Edge cases / failures:**
  - Empty handle/password will cause BlueSky auth failure.
  - Invalid timezone name breaks schedule parsing/comparisons (Luxon zone errors).
- **Sticky notes (applies):**
  - **### 1- START HERE** + timezone link: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
  - **# Inputs**
  - “How To Use” Step 1.

---

### Block 2 — BlueSky Authentication (Session Creation)
**Overview:** Logs into BlueSky and retrieves an access token (`accessJwt`) and DID for subsequent API calls.  
**Nodes involved:** `BlueSky Auth`

#### Node: BlueSky Auth
- **Type / role:** `n8n-nodes-base.httpRequest` — calls BlueSky XRPC `createSession`.
- **Config (interpreted):**
  - `POST https://bsky.social/xrpc/com.atproto.server.createSession`
  - JSON body includes:
    - `identifier`: from Configuration `bluesky_handle`
    - `password`: from Configuration `app_password`
- **Key expressions:**
  - `{{$('Configuration').first().json.bluesky_handle}}`
  - `{{ $('Configuration').first().json.app_password }}`
- **Connections:** → `Get row(s) in sheet`
- **Outputs used downstream:**
  - `accessJwt` for `Authorization: Bearer ...`
  - `did` used as repo in post creation
  - `handle/identifier` used to build the public post URL later
- **Edge cases / failures:**
  - Wrong credentials → 401/403; downstream calls fail.
  - Rate limits / transient network issues.
- **Sticky notes (applies):**
  - **### 2- Get access token**
  - **# Inputs**

---

### Block 3 — Google Sheets Read + Filtering by Status and Time
**Overview:** Reads sheet rows and keeps only those ready to post and whose scheduled time has passed in the configured timezone.  
**Nodes involved:** `Get row(s) in sheet`, `Filter`

#### Node: Get row(s) in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — reads rows.
- **Config (interpreted):**
  - Uses OAuth2 credential: `googleSheetsOAuth2Api Credential`
  - Document and sheet are selected via UI list (currently blank in the exported JSON).
- **Connections:** from `BlueSky Auth` → to `Filter`
- **Edge cases / failures:**
  - Missing/invalid `documentId` or `sheetName` → runtime error.
  - OAuth token expired/revoked → auth error.
  - Sheet column names must match exactly what downstream expects.
- **Sticky notes (applies):**
  - **### 3- Google sheets rows** (includes required columns + sample sheet link)
  - **# Inputs**
  - “How To Use” Step 2/3, including:
    - Sample sheet: https://docs.google.com/spreadsheets/d/1Mg04gK1K5DBtJHrWw3ePRFc_JjkxwAp0deGjapVl2q0/edit?usp=sharing
    - Google Sheets node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/

#### Node: Filter
- **Type / role:** `n8n-nodes-base.filter` — gate rows by business rules.
- **Config (interpreted):** AND conditions:
  1. `Status` equals `"Ready"`
  2. `"Scheduled Time"` is not empty
  3. Scheduled time (parsed as `yyyy-MM-dd HH:mm` in configured timezone) is **before or equal** now (also in that timezone)
- **Key expressions:**
  - `={{ $json.Status }} == "Ready"`
  - `={{ $json["Scheduled Time"] }}` not empty
  - `={{ DateTime.fromFormat($json['Scheduled Time'], 'yyyy-MM-dd HH:mm', { zone: $('Configuration').first().json.timezone }) }}`
  - `={{ $now.setZone($('Configuration').first().json.timezone) }}`
- **Connections:** → `Sort`
- **Edge cases / failures:**
  - If `Scheduled Time` isn’t in exact format, Luxon parse may yield invalid DateTime; comparison can fail or filter incorrectly.
  - If Google Sheets auto-formats the column (not plain text), the string may not match `yyyy-MM-dd HH:mm`.
- **Sticky notes (applies):**
  - **### 4- Checks Schedule Time** (format guidance; “Plain Text” requirement)
  - **### 3- Google sheets rows**
  - **# Inputs**

---

### Block 4 — Thread Ordering + Sequential Loop
**Overview:** Ensures posts are processed in the correct order for thread creation and processes items one-by-one to maintain thread state across items.  
**Nodes involved:** `Sort`, `Loop Over Items`

#### Node: Sort
- **Type / role:** `n8n-nodes-base.sort` — orders items before posting.
- **Config (interpreted):**
  - Sort fields: `Thread ID`, then `Sequence`
- **Connections:** → `Loop Over Items`
- **Edge cases / failures:**
  - If `Sequence` is stored as text, sorting may be lexicographic (“10” before “2”) depending on n8n sort behavior and data typing. Consider forcing numeric conversion upstream if needed.
  - Missing `Thread ID` / `Sequence` leads to unstable ordering.
- **Sticky notes (applies):**
  - **### 4- The Thread Organizer**
  - **# Posting Logic**

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — batch/loop controller.
- **Config (interpreted):**
  - Batch size not explicitly set in JSON (defaults apply). The workflow wiring uses the loop’s second output to process items iteratively.
- **Connections:**
  - Main output **(done)**: not used directly.
  - Loop output → `If`
  - Receives feedback input from `Update row in sheet` to fetch the next item.
- **Edge cases / failures:**
  - If a downstream node errors hard (not “continue”), loop can stop mid-batch.
  - If batch size > 1, thread state could be impacted depending on downstream expectations; this design intends one-by-one processing.
- **Sticky notes (applies):**
  - **### 5- The Batch Processor**
  - **# Posting Logic**

---

### Block 5 — Image Branching, Download, Upload, and Data Re-assembly
**Overview:** For items with an `Image URL`, downloads the image and uploads it to BlueSky to obtain a “blob” reference, then merges that blob back into the original row item. Items without images bypass this and rejoin the stream.  
**Nodes involved:** `If`, `HTTP Download Image`, `Upload Blob`, `Attach Image Blob`, `Consolidate Streams`

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — checks if an image should be processed.
- **Config (interpreted):**
  - Condition: `$json["Image URL"]` is **not empty**
- **Connections:**
  - **True** → `HTTP Download Image` and also to `Attach Image Blob` (as the “original item” input for merging)
  - **False** → `Consolidate Streams` (text-only path)
- **Edge cases / failures:**
  - URL present but invalid/non-image still goes down true branch.
  - If column name differs (e.g., “ImageURL”), condition fails and images won’t attach.
- **Sticky notes (applies):**
  - **### 6- If post contains image**
  - **# Posting Logic**

#### Node: HTTP Download Image
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads the image file.
- **Config (interpreted):**
  - URL: from `Image URL` column
  - `onError: continueRegularOutput` (workflow keeps going if download fails)
- **Connections:** → `Upload Blob`
- **Edge cases / failures:**
  - 404/403, hotlink protection, redirects, large files, timeouts.
  - If download fails but continues, the next node may not have expected binary data.
- **Sticky notes (applies):**
  - **### 6i- Download Image**
  - **# Posting Logic**

#### Node: Upload Blob
- **Type / role:** `n8n-nodes-base.httpRequest` — uploads binary image to BlueSky to get a blob reference.
- **Config (interpreted):**
  - `POST https://bsky.social/xrpc/com.atproto.repo.uploadBlob`
  - Body: **binaryData** from input field `data`
  - Headers:
    - `Authorization: Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
    - `Content-Type: {{ $binary.data.mimeType }}`
  - `onError: continueRegularOutput`
- **Connections:** → `Attach Image Blob`
- **Edge cases / failures:**
  - If binary is missing (download failed), upload will fail or produce invalid output.
  - Token expiry mid-run → 401.
- **Sticky notes (applies):**
  - **### 6ii- Upload Image**
  - **# Posting Logic**

#### Node: Attach Image Blob
- **Type / role:** `n8n-nodes-base.merge` (mode: combine) — recombines original row data with upload response.
- **Config (interpreted):**
  - Mode: **Combine**
  - Combine by position: pairs the original item (from `If` secondary connection) with the upload result (from `Upload Blob`)
- **Connections:** → `Consolidate Streams`
- **Important data contract:**
  - Downstream `Construct Payload` expects `json.blob` to exist for image posts.
  - This merge must output an item that contains original columns plus a `blob` field (from BlueSky upload response structure).
- **Edge cases / failures:**
  - If either side produces mismatched item counts, combine-by-position can attach the wrong blob to the wrong row.
  - If upload failed but continued, blob may be missing; payload construction will skip embed.
- **Sticky notes (applies):**
  - **### 6iii- Re-assembling the Data**
  - **# Posting Logic**

#### Node: Consolidate Streams
- **Type / role:** `n8n-nodes-base.merge` — reunites “image posts” and “text-only posts”.
- **Config (interpreted):**
  - Default merge behavior (no explicit mode in JSON); used here as a stream join so both branches feed a single downstream list.
- **Connections:** → `Construct Payload`
- **Edge cases / failures:**
  - If merge mode is not appropriate for differing item counts, items could be dropped/duplicated depending on n8n defaults/version. Validate after import that it behaves as intended (“append”/“pass-through both inputs” style).
- **Sticky notes (applies):**
  - **### 7- The Reunion**
  - **# Posting Logic**

---

### Block 6 — Payload Construction, Posting, Thread State Memory, Sheet Update
**Overview:** Builds the BlueSky record (with optional embed and optional reply pointers), posts to BlueSky, updates in-memory thread pointers so subsequent sequences reply properly, then updates the corresponding Google Sheet row.  
**Nodes involved:** `Construct Payload`, `Create Post`, `Update Thread State`, `Update row in sheet`

#### Node: Construct Payload
- **Type / role:** `n8n-nodes-base.code` — builds the request body for `createRecord`.
- **Config (interpreted):**
  - Uses **workflow static data (global)** to store:
    - `currentRoot`, `currentParent`, `currentThreadId`
  - For each item:
    - Reads sheet columns: `Thread ID`, `Sequence`, `Content`, optional `Alt Text`, optional `blob`
    - Uses DID from `BlueSky Auth` (`$('BlueSky Auth').first().json.did`)
    - Builds `record` with:
      - `text`, `createdAt`, `$type: "app.bsky.feed.post"`
      - optional `embed` if `json.blob` exists
      - optional `reply` (root/parent) if sequence > 1 and same thread ID
- **Key logic/variables:**
  - `sequence = parseInt(json["Sequence"] || 1)`
  - Thread reset when `sequence === 1`
  - Embed added if `json.blob`
- **Connections:** → `Create Post`
- **Edge cases / failures:**
  - If `Sequence` is missing/non-numeric, `parseInt` may produce `NaN`; reply logic may break.
  - If rows for different threads are interleaved (sorting incorrect), replies can point to wrong parents.
  - If `Alt Text` column doesn’t exist, defaults to empty string (safe).
- **Sticky notes (applies):**
  - **### 8- Construct Payload**
  - **# Posting Logic**
  - **# Create post and update Google sheet**

#### Node: Create Post
- **Type / role:** `n8n-nodes-base.httpRequest` — creates a BlueSky record.
- **Config (interpreted):**
  - `POST https://bsky.social/xrpc/com.atproto.repo.createRecord`
  - Authorization header uses `accessJwt` from `BlueSky Auth`
  - JSON body is taken from `Construct Payload` output (intended to be the `{ repo, collection, record }` object)
- **Key expressions:**
  - `Authorization = Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
  - Body references `Construct Payload` node output (`$node["Construct Payload"].data` in the export; verify after import that it resolves to the current item JSON)
- **Connections:** → `Update Thread State`
- **Edge cases / failures:**
  - If token expired → 401.
  - If payload invalid (missing `repo`, invalid `record`), API returns error.
  - If the node’s body expression does not evaluate per-item after import, posts may be incorrect; validate by testing with 1 row.
- **Sticky notes (applies):**
  - **### 9- Create Post**
  - **# Create post and update Google sheet**

#### Node: Update Thread State
- **Type / role:** `n8n-nodes-base.code` — “memory keeper” and sheet metadata builder.
- **Config (interpreted):**
  - Reads `uri` and `cid` from the `Create Post` response.
  - Builds a public web URL:
    - Extracts rkey from the `uri` last path segment
    - Uses `handle` (from auth response) to create `https://bsky.app/profile/{handle}/post/{rkey}`
  - Updates staticData:
    - If no root yet, set root to current post ref
    - Always set parent to current post ref
  - Outputs original item data + `postUri` + `postLink`
- **Connections:** → `Update row in sheet`
- **Edge cases / failures:**
  - If `Create Post` returns unexpected structure or error body, `lastPost.uri.split('/')` can throw.
  - If `handle` is missing, link construction may be wrong (code falls back to `identifier`).
- **Sticky notes (applies):**
  - **### 10- Update Thread State**
  - **# Create post and update Google sheet**

#### Node: Update row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — writes posting results back to the calendar.
- **Config (interpreted):**
  - Operation: **Update**
  - Matching column: `row_number` (read-only field returned by Sheets node)
  - Updates columns:
    - `Status` = `"Posted"`
    - `Post Link` = `{{$json.postLink}}`
    - `Posted At` = formatted current time in configured timezone (`yyyy-MM-dd HH:mm`)
    - `row_number` taken from the loop item: `$('Loop Over Items').item.json.row_number`
  - Uses same OAuth2 credential as read node.
- **Connections:** → `Loop Over Items` (feeds loop continuation)
- **Edge cases / failures:**
  - If `row_number` missing, update will not match any row.
  - If sheet permissions change, update fails.
  - Timezone formatting depends on valid tz database name.
- **Sticky notes (applies):**
  - **### 11- Update Google Sheets**
  - **# Create post and update Google sheet**

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Time-based workflow entry | — | Configuration | # 📘 Post Scheduler - How To Use (content calendar + sample sheet link) / # Inputs |
| Configuration | set | Store handle/password/timezone | Schedule Trigger | BlueSky Auth | ### 1- START HERE (timezone link) / # Inputs |
| BlueSky Auth | httpRequest | Create BlueSky session (accessJwt, DID) | Configuration | Get row(s) in sheet | ### 2- Get access token / # Inputs |
| Get row(s) in sheet | googleSheets | Read content calendar rows | BlueSky Auth | Filter | ### 3- Google sheets rows (sample sheet link) / # Inputs |
| Filter | filter | Keep Status=Ready and Scheduled Time due | Get row(s) in sheet | Sort | ### 4- Checks Schedule Time / # Inputs |
| Sort | sort | Ensure thread order by Thread ID + Sequence | Filter | Loop Over Items | ### 4- The Thread Organizer / # Posting Logic |
| Loop Over Items | splitInBatches | Process rows sequentially | Sort; Update row in sheet | If | ### 5- The Batch Processor / # Posting Logic |
| If | if | Branch on presence of Image URL | Loop Over Items | HTTP Download Image; Attach Image Blob; Consolidate Streams | ### 6- If post contains image / # Posting Logic |
| HTTP Download Image | httpRequest | Download image from Image URL | If (true) | Upload Blob | ### 6i- Download Image / # Posting Logic |
| Upload Blob | httpRequest | Upload image binary to BlueSky (get blob) | HTTP Download Image | Attach Image Blob | ### 6ii- Upload Image / # Posting Logic |
| Attach Image Blob | merge | Combine original row + blob response | If; Upload Blob | Consolidate Streams | ### 6iii- Re-assembling the Data / # Posting Logic |
| Consolidate Streams | merge | Rejoin image and non-image posts | If (false); Attach Image Blob | Construct Payload | ### 7- The Reunion / # Posting Logic |
| Construct Payload | code | Build createRecord payload; manage thread reply pointers | Consolidate Streams | Create Post | ### 8- Construct Payload / # Create post and update Google sheet |
| Create Post | httpRequest | Create BlueSky post/reply | Construct Payload | Update Thread State | ### 9- Create Post / # Create post and update Google sheet |
| Update Thread State | code | Persist root/parent for threads; generate public link | Create Post | Update row in sheet | ### 10- Update Thread State / # Create post and update Google sheet |
| Update row in sheet | googleSheets | Mark row Posted + write link/time | Update Thread State | Loop Over Items | ### 11- Update Google Sheets / # Create post and update Google sheet |

---

## 4. Reproducing the Workflow from Scratch (Manual Build)

1. **Create a new workflow** in n8n.
2. **Add node: “Schedule Trigger”** (`Schedule Trigger`)
   - Set interval: **every 15 minutes** (or your desired cadence).
   - Connect to the next node.

3. **Add node: “Set”** and name it **Configuration**
   - Add string fields:
     - `bluesky_handle` = your handle (e.g. `steve.bsky.social`)
     - `app_password` = your BlueSky App Password
     - `timezone` = tz database name (e.g. `America/Los_Angeles`)
   - Connect `Schedule Trigger` → `Configuration`.

4. **Add node: “HTTP Request”** and name it **BlueSky Auth**
   - Method: `POST`
   - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`
   - Send body: **JSON**
   - Body:
     - `identifier`: expression `$('Configuration').first().json.bluesky_handle`
     - `password`: expression `$('Configuration').first().json.app_password`
   - Connect `Configuration` → `BlueSky Auth`.

5. **Add node: “Google Sheets”** and name it **Get row(s) in sheet**
   - Credentials: **Google Sheets OAuth2** (connect your Google account).
   - Select **Document** and **Sheet** (your content calendar).
   - Operation: read/get rows (the node label implies reading rows).
   - Connect `BlueSky Auth` → `Get row(s) in sheet`.

6. **Prepare your Google Sheet columns** (header names must match what you use in expressions):
   - `Content`, `Thread ID`, `Sequence`, `Image URL`, `Scheduled Time`, `Status`, `Posted At`, `Post Link`, and ensure `row_number` is available via the Sheets node.
   - Set **Scheduled Time column format** to **Plain Text**.
   - Use format `YYYY-MM-DD HH:mm` (e.g. `2025-12-25 14:30`).

7. **Add node: “Filter”** and name it **Filter**
   - Conditions (AND):
     1. `Status` equals `Ready`
     2. `Scheduled Time` is not empty
     3. Date/time comparison:
        - Left: `DateTime.fromFormat($json['Scheduled Time'], 'yyyy-MM-dd HH:mm', { zone: $('Configuration').first().json.timezone })`
        - Operator: **before or equals**
        - Right: `$now.setZone($('Configuration').first().json.timezone)`
   - Connect `Get row(s) in sheet` → `Filter`.

8. **Add node: “Sort”** and name it **Sort**
   - Sort by fields:
     - `Thread ID`
     - `Sequence`
   - Connect `Filter` → `Sort`.

9. **Add node: “Split In Batches”** and name it **Loop Over Items**
   - Configure to process **one item at a time** (set batch size = 1 if available in your UI/version).
   - Connect `Sort` → `Loop Over Items`.
   - You will later connect the loop “continue” by wiring the final node back into `Loop Over Items`.

10. **Add node: “If”** and name it **If**
    - Condition: `{{$json["Image URL"]}}` **is not empty**
    - Connect loop output of `Loop Over Items` → `If`.

11. **Image path (True branch):**
    1. Add **HTTP Request** named **HTTP Download Image**
       - URL: `{{$json["Image URL"]}}`
       - Configure it to download binary (depending on your n8n version, enable “Download” / “Response: File”).
       - Set **On Error**: “Continue (regular output)” if you want to mimic the provided behavior.
       - Connect `If (true)` → `HTTP Download Image`.
    2. Add **HTTP Request** named **Upload Blob**
       - Method: `POST`
       - URL: `https://bsky.social/xrpc/com.atproto.repo.uploadBlob`
       - Send body: **Binary**
       - Binary property/input field: `data`
       - Headers:
         - `Authorization`: `Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
         - `Content-Type`: `{{ $binary.data.mimeType }}`
       - Set **On Error**: “Continue (regular output)” to match export.
       - Connect `HTTP Download Image` → `Upload Blob`.
    3. Add **Merge** node named **Attach Image Blob**
       - Mode: **Combine** (by position).
       - Connect:
         - `Upload Blob` → `Attach Image Blob` (Input 1)
         - `If (true)` → `Attach Image Blob` (Input 2) to carry the original row data forward.

12. **Text-only path (False branch) and reunification:**
    - Add **Merge** node named **Consolidate Streams**
      - Configure merge behavior so it **outputs both branches into one stream** (validate after import/build; n8n merge modes vary).
      - Connect:
        - `Attach Image Blob` → `Consolidate Streams` (image items)
        - `If (false)` → `Consolidate Streams` (text-only items)

13. **Add node: “Code”** named **Construct Payload**
    - Implement logic equivalent to:
      - Build `record` with text/createdAt/$type
      - If `json.blob` exists, set `record.embed` to `app.bsky.embed.images`
      - Use workflow static data to store `currentRoot/currentParent/currentThreadId`
      - If `Sequence === 1`, reset pointers for a new thread
      - If `Sequence > 1` and same thread ID, add `record.reply = {root, parent}`
      - Output `{ repo: did, collection: "app.bsky.feed.post", record }`
    - Connect `Consolidate Streams` → `Construct Payload`.

14. **Add node: “HTTP Request”** named **Create Post**
    - Method: `POST`
    - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`
    - Headers:
      - `Authorization`: `Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
    - Body: JSON from the **current item** produced by `Construct Payload`.
    - Connect `Construct Payload` → `Create Post`.

15. **Add node: “Code”** named **Update Thread State**
    - Implement:
      - Read `uri` and `cid` from `Create Post` response
      - Extract rkey from `uri` and build `https://bsky.app/profile/{handle}/post/{rkey}`
      - Update workflow static data root/parent
      - Output original row data + `postUri` + `postLink`
    - Connect `Create Post` → `Update Thread State`.

16. **Add node: “Google Sheets”** named **Update row in sheet**
    - Credentials: same Google OAuth2
    - Operation: **Update**
    - Match by: `row_number`
    - Set fields:
      - `Status` = `Posted`
      - `Post Link` = `{{$json.postLink}}`
      - `Posted At` = `{{$now.setZone($('Configuration').first().json.timezone || 'UTC').toFormat('yyyy-MM-dd HH:mm')}}`
      - `row_number` = `{{$('Loop Over Items').item.json.row_number}}`
    - Connect `Update Thread State` → `Update row in sheet`.

17. **Close the loop**
    - Connect `Update row in sheet` → `Loop Over Items` (so it continues with the next item until exhaustion).

18. **Test with one row**
    - Put a single row with:
      - `Status = Ready`
      - valid `Scheduled Time` in the past (in your configured timezone)
      - `Thread ID` unique
      - `Sequence = 1`
    - Execute workflow manually once before activating.

19. **Activate workflow**
    - Turn on Active to run on schedule.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sample Google Sheet (structure reference) | https://docs.google.com/spreadsheets/d/1Mg04gK1K5DBtJHrWw3ePRFc_JjkxwAp0deGjapVl2q0/edit?usp=sharing |
| Google Sheets node documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/ |
| Timezone names (tz database) | https://en.wikipedia.org/wiki/List_of_tz_database_time_zones |
| Sheet requirements emphasized by notes | Must include: `Content`, `Thread ID` (always), `Sequence` (use 1 for single posts), optional `Image URL` (direct image), `Scheduled Time` (Plain Text, `YYYY-MM-DD HH:mm`), `Status` (`Ready` → `Posted`) |
| Operational intent from sticky notes | Runs periodically, posts all due “Ready” rows, updates sheet with links/timestamps |

