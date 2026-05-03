Detect and move duplicate Google Drive files with Supabase and Slack

https://n8nworkflows.xyz/workflows/detect-and-move-duplicate-google-drive-files-with-supabase-and-slack-13534


# Detect and move duplicate Google Drive files with Supabase and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Detect and move duplicate Google Drive files with Supabase and Slack  
**Purpose:** Monitor a specific Google Drive folder for newly created files, compute a content-based MD5 hash from the downloaded binary, compare it against a Supabase table of known hashes, and then:
- **If duplicate (same hash, different file):** move the new file to a “Duplicates” folder and send a Slack alert.
- **If unique (hash not found):** store the hash metadata in Supabase.
- **If re-trigger of the same file (same hash and same file_id already stored):** do nothing.
- **If binary is missing:** log an error in Supabase audit log.

### Logical Blocks
**1.1 Google Drive intake & normalization**  
Trigger on file creation → normalize metadata → download file → verify binary exists.

**1.2 Hash generation & hash lookup (Supabase)**  
Generate MD5 from binary → query Supabase `file_hashes` for existing hash.

**1.3 Duplicate / re-trigger decisioning**  
If existing record matches same `file_id` → no-op.  
If existing record exists for different `file_id` → treat as duplicate.

**1.4 Duplicate handling (Drive move + Slack + audit log)**  
Move file to “Duplicates” folder → notify Slack → write audit record.

**1.5 Unique handling (insert hash + race-condition interpretation + audit log)**  
Insert new hash row → interpret insert response for conflicts/race conditions → write audit record.

**1.6 Error handling (missing binary → audit log)**  
Create structured error payload → write audit record.

---

## 2. Block-by-Block Analysis

### 2.1 Google Drive intake & normalization
**Overview:** Watches a specific Google Drive folder for new uploads, extracts a consistent set of file metadata, downloads the file content, and ensures binary content exists before hashing.

**Nodes involved:**
- Google Drive Trigger
- Prepare File Info
- Download File
- Validate Binary
- Handle Missing Binary (error branch)

#### Node: Google Drive Trigger
- **Type / role:** `Google Drive Trigger` (`n8n-nodes-base.googleDriveTrigger`) — polling trigger for new files.
- **Key configuration:**
  - Event: **fileCreated**
  - Trigger on: **specificFolder**
  - Folder to watch: **Lendium Files** (`1vleObg6ZBzUflpAVT2Joc5D7oBNYWNUw`)
  - Polling: **everyMinute**
- **Credentials:** Google Drive OAuth2 (“Google Drive account”).
- **Input/Output:**
  - Entry point of the workflow.
  - Outputs file metadata from Google Drive to **Prepare File Info**.
- **Failure/edge cases:**
  - Missed events due to polling interval and rate limits.
  - OAuth token expiry / insufficient permissions to the folder.
  - If Google sends different metadata fields (id vs fileId, name vs title), downstream normalization handles it.

#### Node: Prepare File Info
- **Type / role:** `Code` (`n8n-nodes-base.code`) — normalize trigger payload.
- **Configuration choices:**
  - Mode: **Run once for each item**
  - Extracts and normalizes:
    - `fileId` from `id` or `fileId`
    - `fileName` from `name` or `title`
    - `mimeType` from `mimeType` or `mime_type`
    - `fileSize` from `size` or `fileSize` (forced integer, default 0)
    - `createdTime` from `createdTime` or `createdDate` (fallback: now)
  - Preserves full original trigger payload under `originalData`
- **Key variables/expressions:** uses `$input.item.json` and returns `{ json: {...} }`.
- **Input/Output:**
  - Input: Google Drive Trigger output
  - Output: normalized metadata to **Download File**
- **Failure/edge cases:**
  - Missing `id`/`name` fields → defaults applied; may break later operations if `fileId` is empty.

#### Node: Download File
- **Type / role:** `Google Drive` (`n8n-nodes-base.googleDrive`) — download file binary.
- **Key configuration:**
  - Operation: **download**
  - File ID: `={{ $json.fileId }}`
- **Credentials:** Google Drive OAuth2.
- **Input/Output:**
  - Input: normalized file info
  - Output: item containing binary (typically in `$binary.data`) to **Validate Binary**
- **Failure/edge cases:**
  - Download not allowed for Google-native formats (Docs/Sheets) unless exported (this node is configured only for download, not export).
  - Large files can cause timeouts / memory issues depending on n8n binary settings.
  - Permission errors if the workflow account cannot access the file.

#### Node: Validate Binary
- **Type / role:** `IF` (`n8n-nodes-base.if`) — check presence of binary.
- **Key configuration:**
  - Condition: `={{ !!$binary && !!$binary.data }}` is **true**
- **Input/Output:**
  - True branch → **Crypto**
  - False branch → **Handle Missing Binary**
- **Failure/edge cases:**
  - If Google Drive node outputs a different binary property name than `data`, this will evaluate false.
  - Some files (shortcuts, Google Docs) may not produce binary via download.

#### Node: Handle Missing Binary
- **Type / role:** `Code` — create a structured error item for audit logging.
- **Key configuration:**
  - Builds:
    - `status: 'error'`
    - `errorType: 'missing_binary'`
    - `message` includes file name/id
    - file metadata + timestamp
- **Input/Output:**
  - Input: from Validate Binary (false)
  - Output: to **Prepare Error Log Data**
- **Failure/edge cases:**
  - Relies on `$('Prepare File Info').item.json` being available; if upstream item indexing diverges, references can fail.

---

### 2.2 Hash generation & Supabase lookup
**Overview:** Generates an MD5 hash from the downloaded binary and queries Supabase to see if the hash already exists.

**Nodes involved:**
- Crypto
- Check Hash Exists

#### Node: Crypto
- **Type / role:** `Crypto` (`n8n-nodes-base.crypto`) — hashing binary content.
- **Key configuration:**
  - `binaryData: true` (hash computed from binary input)
  - `dataPropertyName: "hash"` (hash stored in JSON at `$json.hash`)
  - (Algorithm is not explicitly shown in JSON; the node’s behavior here is intended as **MD5** per sticky note and downstream text. Ensure the Crypto node is set to MD5 in UI if required.)
- **Input/Output:**
  - Input: item with binary from Download File
  - Output: item with `hash` field → **Check Hash Exists**
- **Failure/edge cases:**
  - If algorithm mismatch (SHA vs MD5), dedup will not match historical data.
  - Missing/invalid binary → hashing fails (mitigated by Validate Binary).

#### Node: Check Hash Exists
- **Type / role:** `Supabase` (`n8n-nodes-base.supabase`) — query `file_hashes` for existing hash.
- **Key configuration:**
  - Operation: **getAll**
  - Table: `file_hashes`
  - Filters: `hash` **equals** `={{ $json.hash }}`
  - `limit: 1`
  - `alwaysOutputData: true` (prevents node from returning empty execution in some cases)
- **Credentials:** Supabase API (“Supabase account”).
- **Input/Output:**
  - Input: hashed item from Crypto
  - Output: query result → **Same File Re-triggered?**
- **Failure/edge cases:**
  - Table/column mismatch (`hash`) causes Supabase errors.
  - If multiple rows exist for same hash, `limit:1` returns one arbitrary match unless Supabase orders deterministically.
  - Auth errors or row-level security (RLS) blocking reads.

---

### 2.3 Duplicate / re-trigger decisioning
**Overview:** Distinguishes between (a) a re-trigger for the same file already recorded, (b) a true duplicate (same content but different file), and (c) a unique file (no hash match).

**Nodes involved:**
- Same File Re-triggered?
- No Operation, do nothing
- Hash Matched Different File?

#### Node: Same File Re-triggered?
- **Type / role:** `IF` — checks if the found hash belongs to the same `file_id` as the current file.
- **Key configuration (AND conditions):**
  1. `={{ $json.hash }}` **exists** (note: this refers to the incoming item; in practice this node receives Supabase output. If Supabase output doesn’t carry `hash`, this condition may behave unexpectedly.)
  2. `={{ $('Check Hash Exists').item.json.file_id }}` **equals** `={{ $('Prepare File Info').item.json.fileId }}`
- **Input/Output:**
  - True → **No Operation, do nothing**
  - False → **Hash Matched Different File?**
- **Failure/edge cases:**
  - Potential expression mismatch: when querying Supabase, returned fields typically include `file_id`, `hash`, etc. Ensure the IF checks the correct property (often `file_id` existence is the more reliable check than `$json.hash` here).
  - If Supabase returns no rows, `$('Check Hash Exists').item.json.file_id` may be undefined; the equals check will fail, routing to “Hash Matched Different File?” which then decides duplicate vs unique.

#### Node: No Operation, do nothing
- **Type / role:** `NoOp` — explicit sink for same-file retrigger.
- **Configuration:** none.
- **Input/Output:** Ends that execution path.
- **Failure/edge cases:** none (used to prevent re-processing).

#### Node: Hash Matched Different File?
- **Type / role:** `IF` — determine whether Supabase returned a record at all.
- **Key configuration:**
  - Condition: `={{ $json.file_id }}` **exists**
  - Interpreted logic:
    - If `file_id` exists in Supabase response → hash match found (and since previous IF was false, it’s a different file) → duplicate.
    - If `file_id` does not exist → no match → unique.
- **Input/Output:**
  - True → **Move to Duplicates**
  - False → **Prepare Hash Data**
- **Failure/edge cases:**
  - If Supabase node returns an array structure vs single object, `$json.file_id` may not exist. Confirm Supabase node output format in your n8n version.

---

### 2.4 Duplicate handling (Drive move + Slack + audit log)
**Overview:** For true duplicates, moves the new file into a dedicated “Duplicates” folder, sends a Slack alert, and logs the event in Supabase.

**Nodes involved:**
- Move to Duplicates
- Notify Duplicate
- Prepare Dup Log Data
- Log Duplicate Event

#### Node: Move to Duplicates
- **Type / role:** `Google Drive` — move duplicate file.
- **Key configuration:**
  - Operation: **move**
  - File ID: `={{ $('Prepare File Info').item.json.fileId }}`
  - Drive: “My Drive”
  - Destination folder: **Duplicates** (`17_l9LfnpyJas6iZy3sIMNNdfi4W7ia6o`)
- **Credentials:** Google Drive OAuth2.
- **Input/Output:** True branch from “Hash Matched Different File?” → outputs to **Notify Duplicate**
- **Failure/edge cases:**
  - If file is in a shared drive or different drive, moving may require different `driveId`/permissions.
  - Cannot move if the account lacks organizer/content manager rights.
  - If file is already in Duplicates, move may be a no-op or error depending on API behavior.

#### Node: Notify Duplicate
- **Type / role:** `Slack` — send alert to a channel.
- **Key configuration:**
  - Target: channel `estateline-ai` (channel ID `C0ABU5WAKLH`)
  - Message text includes file name/id/hash/size and timestamp.
  - Uses expressions:
    - `$('Prepare File Info').item.json.fileName`
    - `$('Prepare File Info').item.json.fileId`
    - `$('Crypto').item.json.hash`
    - `$('Prepare File Info').item.json.fileSize`
    - `$now.toISO()`
- **Credentials:** Slack API.
- **Input/Output:** Outputs to **Prepare Dup Log Data**
- **Failure/edge cases:**
  - Slack auth/token scopes (chat:write) and channel access required.
  - If `Crypto` node didn’t run in the same execution path (should, in normal flow), the hash reference would fail.

#### Node: Prepare Dup Log Data
- **Type / role:** `Code` — prepare audit log row for duplicates.
- **Key configuration:**
  - Builds row:
    - `file_id`, `file_name`, `hash`
    - `status: 'duplicate'`
    - `action_taken: 'moved_to_duplicates'`
- **Input/Output:** To **Log Duplicate Event**
- **Failure/edge cases:** Expression references to other nodes require consistent item pairing.

#### Node: Log Duplicate Event
- **Type / role:** `Supabase` — insert audit row.
- **Key configuration:**
  - Table: `dedup_audit_log`
  - Data: **autoMapInputData** (maps JSON fields directly)
- **Credentials:** Supabase API.
- **Failure/edge cases:**
  - RLS may block inserts.
  - Table schema mismatches for columns (`action_taken`, etc.).

---

### 2.5 Unique handling (insert hash + race-condition interpretation + audit log)
**Overview:** For unique content, inserts the new hash into Supabase, interprets insert results to catch conflicts/race conditions, and logs the outcome.

**Nodes involved:**
- Prepare Hash Data
- Insert New Hash
- Handle DB Error
- Prepare Unique Log Data
- Log Unique Event

#### Node: Prepare Hash Data
- **Type / role:** `Code` — map normalized + hash fields to `file_hashes` insert schema.
- **Key configuration:**
  - Produces:
    - `file_id`, `file_name`, `hash`
    - `file_size`
    - `mime_type`
- **Input/Output:** To **Insert New Hash**
- **Failure/edge cases:** Depends on `$('Crypto').item.json.hash` existing.

#### Node: Insert New Hash
- **Type / role:** `Supabase` — insert into `file_hashes`.
- **Key configuration:**
  - Table: `file_hashes`
  - Data: **autoMapInputData**
  - (No explicit upsert/onConflict configured in JSON; conflicts may surface as errors depending on DB constraints and node behavior.)
- **Credentials:** Supabase API.
- **Input/Output:** To **Handle DB Error**
- **Failure/edge cases:**
  - To make race-condition handling real, `file_hashes.hash` should be **UNIQUE** at DB level; otherwise duplicates can be inserted.
  - Insert conflicts might return an error or a response without `id` depending on Supabase settings.

#### Node: Handle DB Error
- **Type / role:** `Code` — normalize insert result into `status` values.
- **Key logic:**
  - If `item.error` exists → `status: 'duplicate_race_condition'`
  - Else if no `item.id` → `status: 'duplicate_race_condition'` (conflict inferred)
  - Else → `status: 'unique_inserted'` with `insertedId`
- **Input/Output:** To **Prepare Unique Log Data**
- **Failure/edge cases:**
  - Supabase insert responses vary; ensure the node returns `id` on insert (may require “returning” configuration depending on node/version).
  - If the table uses a different PK field name, `id` check will be wrong.

#### Node: Prepare Unique Log Data
- **Type / role:** `Code` — convert status into audit log row.
- **Key configuration:**
  - `status` taken from current item (`$json.status`)
  - `action_taken`:
    - `'stored_hash'` if `unique_inserted`
    - `'race_condition_caught'` otherwise
- **Input/Output:** To **Log Unique Event**
- **Failure/edge cases:** If status is missing/unknown, audit log may be inconsistent.

#### Node: Log Unique Event
- **Type / role:** `Supabase` — insert audit row into `dedup_audit_log`.
- **Key configuration:**
  - Table: `dedup_audit_log`
  - Data: auto-map
- **Failure/edge cases:** same as other Supabase insert nodes (RLS/schema).

---

### 2.6 Error handling (missing binary → audit log)
**Overview:** When file binary cannot be downloaded/validated, the workflow logs a structured error record into the audit table.

**Nodes involved:**
- Prepare Error Log Data
- Log Error Event

#### Node: Prepare Error Log Data
- **Type / role:** `Code` — map error to audit schema.
- **Key configuration:**
  - Inserts:
    - `file_id`, `file_name`
    - `hash: null`
    - `status: 'error'`
    - `action_taken: 'none'`
    - `error_message`
- **Input/Output:** To **Log Error Event**
- **Failure/edge cases:** expects incoming item contains `fileId`, `fileName`, `message`.

#### Node: Log Error Event
- **Type / role:** `Supabase` — insert error row into `dedup_audit_log`.
- **Key configuration:** auto-map input data.
- **Failure/edge cases:** RLS/schema.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Drive Trigger | googleDriveTrigger | Poll Drive folder for new files | — | Prepare File Info | Watches a specific Google Drive folder for new uploads, extracts metadata, downloads the file, and validates that binary data exists before hashing. |
| Prepare File Info | code | Normalize file metadata | Google Drive Trigger | Download File | Watches a specific Google Drive folder for new uploads, extracts metadata, downloads the file, and validates that binary data exists before hashing. |
| Download File | googleDrive | Download file binary | Prepare File Info | Validate Binary | Watches a specific Google Drive folder for new uploads, extracts metadata, downloads the file, and validates that binary data exists before hashing. |
| Validate Binary | if | Ensure binary exists before hashing | Download File | Crypto; Handle Missing Binary | Watches a specific Google Drive folder for new uploads, extracts metadata, downloads the file, and validates that binary data exists before hashing. |
| Crypto | crypto | Compute content hash (intended MD5) | Validate Binary (true) | Check Hash Exists | Generates an MD5 hash from the file binary and checks Supabase to determine whether the hash already exists. |
| Check Hash Exists | supabase | Query existing hash in `file_hashes` | Crypto | Same File Re-triggered? | Generates an MD5 hash from the file binary and checks Supabase to determine whether the hash already exists. |
| Same File Re-triggered? | if | Detect re-trigger of same file_id | Check Hash Exists | No Operation, do nothing; Hash Matched Different File? | If the hash exists for a different file, the workflow moves the file to a Duplicates folder and sends a Slack notification. |
| No Operation, do nothing | noOp | Stop processing for same-file retrigger | Same File Re-triggered? (true) | — | If the hash exists for a different file, the workflow moves the file to a Duplicates folder and sends a Slack notification. |
| Hash Matched Different File? | if | Decide duplicate vs unique based on hash match | Same File Re-triggered? (false) | Move to Duplicates; Prepare Hash Data | If the hash exists for a different file, the workflow moves the file to a Duplicates folder and sends a Slack notification. |
| Move to Duplicates | googleDrive | Move duplicate file to Duplicates folder | Hash Matched Different File? (true) | Notify Duplicate | If the hash exists for a different file, the workflow moves the file to a Duplicates folder and sends a Slack notification. |
| Notify Duplicate | slack | Send Slack alert for duplicates | Move to Duplicates | Prepare Dup Log Data | If the hash exists for a different file, the workflow moves the file to a Duplicates folder and sends a Slack notification. |
| Prepare Dup Log Data | code | Prepare duplicate audit log row | Notify Duplicate | Log Duplicate Event | Logs duplicates, unique inserts, race conditions, and binary errors into a Supabase audit table for traceability. |
| Log Duplicate Event | supabase | Insert duplicate event into `dedup_audit_log` | Prepare Dup Log Data | — | Logs duplicates, unique inserts, race conditions, and binary errors into a Supabase audit table for traceability. |
| Prepare Hash Data | code | Prepare `file_hashes` insert payload | Hash Matched Different File? (false) | Insert New Hash | If the hash does not exist, the workflow inserts the new hash into Supabase and confirms successful storage, including race condition handling. |
| Insert New Hash | supabase | Insert new hash into `file_hashes` | Prepare Hash Data | Handle DB Error | If the hash does not exist, the workflow inserts the new hash into Supabase and confirms successful storage, including race condition handling. |
| Handle DB Error | code | Interpret insert result (unique vs race/conflict) | Insert New Hash | Prepare Unique Log Data | If the hash does not exist, the workflow inserts the new hash into Supabase and confirms successful storage, including race condition handling. |
| Prepare Unique Log Data | code | Prepare unique/race audit log row | Handle DB Error | Log Unique Event | Logs duplicates, unique inserts, race conditions, and binary errors into a Supabase audit table for traceability. |
| Log Unique Event | supabase | Insert unique/race event into `dedup_audit_log` | Prepare Unique Log Data | — | Logs duplicates, unique inserts, race conditions, and binary errors into a Supabase audit table for traceability. |
| Handle Missing Binary | code | Create error payload when binary missing | Validate Binary (false) | Prepare Error Log Data | Watches a specific Google Drive folder for new uploads, extracts metadata, downloads the file, and validates that binary data exists before hashing. |
| Prepare Error Log Data | code | Prepare error audit log row | Handle Missing Binary | Log Error Event | Logs duplicates, unique inserts, race conditions, and binary errors into a Supabase audit table for traceability. |
| Log Error Event | supabase | Insert error event into `dedup_audit_log` | Prepare Error Log Data | — | Logs duplicates, unique inserts, race conditions, and binary errors into a Supabase audit table for traceability. |
| Sticky Note | stickyNote | Documentation / canvas note | — | — | This workflow detects and handles duplicate files uploaded to a specific Google Drive folder using content based hashing. Instead of relying on file names, it generates an MD5 hash from the actual file binary and compares it against a Supabase database. / If the same hash already exists for a different file, the new file is automatically moved to a Duplicates folder and a Slack alert is sent. If the file is unique, its hash is stored in Supabase. All outcomes, including duplicates, successful inserts, race conditions, and binary errors, are logged in an audit table. / ## How it works / 1. Watches a Google Drive folder for new files. / 2. Downloads the file and generates an MD5 hash. / 3. Checks Supabase for an existing hash match. / 4. Moves duplicate files and sends Slack alerts. / 5. Stores unique hashes and logs all events. / ## Setup steps / 1. Connect Google Drive and select the folder to monitor. / 2. Connect Supabase and create `file_hashes` and `dedup_audit_log` tables. / 3. Connect Slack and choose a notification channel. / 4. Set your Duplicates folder ID in the Move node. |
| Sticky Note1 | stickyNote | Documentation / canvas note | — | — | Watches a specific Google Drive folder for new uploads, extracts metadata, downloads the file, and validates that binary data exists before hashing. |
| Sticky Note2 | stickyNote | Documentation / canvas note | — | — | Generates an MD5 hash from the file binary and checks Supabase to determine whether the hash already exists. |
| Sticky Note3 | stickyNote | Documentation / canvas note | — | — | If the hash exists for a different file, the workflow moves the file to a Duplicates folder and sends a Slack notification. |
| Sticky Note4 | stickyNote | Documentation / canvas note | — | — | If the hash does not exist, the workflow inserts the new hash into Supabase and confirms successful storage, including race condition handling. |
| Sticky Note5 | stickyNote | Documentation / canvas note | — | — | Logs duplicates, unique inserts, race conditions, and binary errors into a Supabase audit table for traceability. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n and set the name to:  
   **“Detect and move duplicate Google Drive files with Supabase and Slack”**.

2) **Add Trigger: “Google Drive Trigger”**
   - Node type: **Google Drive Trigger**
   - Credentials: configure **Google Drive OAuth2** (connect the Google account that has access to the monitored folder).
   - Event: **File Created**
   - Trigger on: **Specific Folder**
   - Folder: select the target folder (e.g., “Lendium Files”).
   - Poll time: **Every minute**.

3) **Add Code node: “Prepare File Info”** and connect from the trigger
   - Mode: **Run once for each item**
   - Implement normalization:
     - Map `id/fileId`, `name/title`, `mimeType/mime_type`, `size/fileSize`, `createdTime/createdDate`
     - Output fields: `fileId`, `fileName`, `mimeType`, `fileSize` (int), `createdTime`, `originalData`

4) **Add Google Drive node: “Download File”**
   - Node type: **Google Drive**
   - Credentials: same Google Drive OAuth2
   - Operation: **Download**
   - File ID: expression `{{$json.fileId}}`
   - Connect **Prepare File Info → Download File**

5) **Add IF node: “Validate Binary”**
   - Condition (boolean): expression `{{ !!$binary && !!$binary.data }}`
   - Connect **Download File → Validate Binary**
   - True output will go to hashing; false output to error handling.

6) **Add Crypto node: “Crypto”** (hashing)
   - Node type: **Crypto**
   - Configure it to hash **binary data** and output into JSON field `hash`
   - Ensure the hashing algorithm is **MD5** (set in UI if applicable)
   - Connect **Validate Binary (true) → Crypto**

7) **Add Supabase node: “Check Hash Exists”**
   - Credentials: configure **Supabase API** (URL + service role key or appropriate key; ensure RLS policies allow reads/writes as needed).
   - Operation: **Get All**
   - Table: `file_hashes`
   - Filter: `hash` equals `{{$json.hash}}`
   - Limit: `1`
   - Enable “Always output data” (or equivalent) to avoid empty-output issues.
   - Connect **Crypto → Check Hash Exists**

8) **Add IF node: “Same File Re-triggered?”**
   - Conditions (AND):
     - A “row exists” check (recommended): `{{$json.file_id}} exists`
     - And equality: `{{ $('Check Hash Exists').item.json.file_id }}` equals `{{ $('Prepare File Info').item.json.fileId }}`
   - Connect **Check Hash Exists → Same File Re-triggered?**

9) **Add NoOp node: “No Operation, do nothing”**
   - Connect **Same File Re-triggered? (true) → NoOp**

10) **Add IF node: “Hash Matched Different File?”**
   - Condition: `{{$json.file_id}} exists`
   - Connect **Same File Re-triggered? (false) → Hash Matched Different File?**
   - Interpretation:
     - True = duplicate (hash exists for some other file_id)
     - False = unique (no match)

11) **Duplicate path: Move + Slack + audit**
   1. Add Google Drive node **“Move to Duplicates”**
      - Operation: **Move**
      - File ID: `{{ $('Prepare File Info').item.json.fileId }}`
      - Destination folder: select/create **Duplicates** folder and set its folder ID.
      - Connect **Hash Matched Different File? (true) → Move to Duplicates**
   2. Add Slack node **“Notify Duplicate”**
      - Credentials: configure **Slack API** (OAuth) with `chat:write` permission.
      - Resource/operation: send message to a channel
      - Channel: pick the desired channel (e.g., `estateline-ai`)
      - Message text: include file name/id/hash/size and timestamp using expressions (as in the workflow).
      - Connect **Move to Duplicates → Notify Duplicate**
   3. Add Code node **“Prepare Dup Log Data”**
      - Output JSON: `file_id`, `file_name`, `hash`, `status:'duplicate'`, `action_taken:'moved_to_duplicates'`
      - Connect **Notify Duplicate → Prepare Dup Log Data**
   4. Add Supabase node **“Log Duplicate Event”**
      - Operation: insert (auto-map input)
      - Table: `dedup_audit_log`
      - Connect **Prepare Dup Log Data → Log Duplicate Event**

12) **Unique path: Insert hash + interpret result + audit**
   1. Add Code node **“Prepare Hash Data”**
      - Output JSON for `file_hashes`: `file_id`, `file_name`, `hash`, `file_size`, `mime_type`
      - Connect **Hash Matched Different File? (false) → Prepare Hash Data**
   2. Add Supabase node **“Insert New Hash”**
      - Operation: insert (auto-map input)
      - Table: `file_hashes`
      - Connect **Prepare Hash Data → Insert New Hash**
      - Database best practice: add a **UNIQUE constraint on `hash`** to enforce dedup and enable race-condition detection.
   3. Add Code node **“Handle DB Error”**
      - If insert returns an error or no inserted id, set `status:'duplicate_race_condition'`
      - Else set `status:'unique_inserted'`
      - Connect **Insert New Hash → Handle DB Error**
   4. Add Code node **“Prepare Unique Log Data”**
      - Build audit row with `status` and `action_taken` derived from it
      - Connect **Handle DB Error → Prepare Unique Log Data**
   5. Add Supabase node **“Log Unique Event”**
      - Insert (auto-map)
      - Table: `dedup_audit_log`
      - Connect **Prepare Unique Log Data → Log Unique Event**

13) **Missing-binary error path: audit**
   1. Add Code node **“Handle Missing Binary”**
      - Build structured error with `status:'error'`, `errorType:'missing_binary'`, file metadata, timestamp
      - Connect **Validate Binary (false) → Handle Missing Binary**
   2. Add Code node **“Prepare Error Log Data”**
      - Map to audit schema: `file_id`, `file_name`, `hash:null`, `status:'error'`, `action_taken:'none'`, `error_message`
      - Connect **Handle Missing Binary → Prepare Error Log Data**
   3. Add Supabase node **“Log Error Event”**
      - Insert into `dedup_audit_log`
      - Connect **Prepare Error Log Data → Log Error Event**

14) **Supabase schema requirements (minimum)**
   - Table: `file_hashes`
     - Columns: `id` (PK, optional but expected by current “Handle DB Error”), `file_id`, `file_name`, `hash`, `file_size`, `mime_type`
     - Constraint: **UNIQUE(hash)** recommended
   - Table: `dedup_audit_log`
     - Columns: `id` (PK), `file_id`, `file_name`, `hash` (nullable), `status`, `action_taken`, `error_message` (nullable), plus optional timestamps.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow detects and handles duplicate files uploaded to a specific Google Drive folder using content based hashing (MD5 from binary), compares against Supabase, moves duplicates, sends Slack alerts, and logs all outcomes to an audit table. | From the main sticky note content |
| Setup steps: connect Google Drive, create Supabase tables `file_hashes` and `dedup_audit_log`, connect Slack channel, set Duplicates folder ID in the move node. | From the main sticky note content |
| Practical note: “Download” may not work for Google Docs/Sheets without export; if your folder contains Google-native files, consider using an export operation or excluding those MIME types. | Operational consideration based on node behavior |
| For reliable race-condition detection, enforce a UNIQUE constraint on `file_hashes.hash` and confirm Supabase insert returns an `id` field (or update “Handle DB Error” to check your actual PK field). | Required to make the “race condition” branch meaningful |