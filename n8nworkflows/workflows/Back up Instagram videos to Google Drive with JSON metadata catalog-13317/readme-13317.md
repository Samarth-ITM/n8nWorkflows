Back up Instagram videos to Google Drive with JSON metadata catalog

https://n8nworkflows.xyz/workflows/back-up-instagram-videos-to-google-drive-with-json-metadata-catalog-13317


# Back up Instagram videos to Google Drive with JSON metadata catalog

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow backs up Instagram **VIDEO** and **REELS** posts to a specified **Google Drive folder**, and maintains a searchable **JSON metadata catalog** generated from an **n8n Data Table** (“Instagram Video Backups”). It avoids re-downloading posts already backed up by checking a primary-key-like field (`postId`).

**Target use cases:**
- Personal/team archiving of Instagram video content
- Maintaining a structured catalog (JSON) for search/automation downstream
- Periodic incremental backups with deduplication

### Logical Blocks
**1.1 Entry & Scheduling**
- Manual run for testing + scheduled daily execution (midnight cron).

**1.2 Account Setup & Configuration**
- Fetches Instagram account info, then sets runtime configuration (Drive folder ID, max items, wait delay, metadata filename).

**1.3 Fetch & Filter Instagram Media**
- Fetches recent media items from Instagram Graph API, sorts them chronologically, splits into individual items, and filters to VIDEO/REELS only.

**1.4 Per-Video Dedup, Download, Upload, and Catalog Record**
- Loops over videos, checks Data Table for `postId`, downloads video binary, uploads to Google Drive, extracts metadata, and stores a record in the Data Table.

**1.5 Update JSON Catalog (Drive file + cleanup)**
- Exports the Data Table to a JSON file, uploads it to Google Drive, lists existing catalog files with the same name, and deletes older duplicates keeping only the newest.

---

## 2. Block-by-Block Analysis

### 2.1 Entry & Scheduling
**Overview:** Starts the workflow either manually (for testing) or automatically on a daily schedule.

**Nodes involved:**
- Manual Trigger
- Schedule Trigger

**Node details**
1) **Manual Trigger** (n8n-nodes-base.manualTrigger, v1)  
- **Role:** Manual entry point.  
- **Outputs:** To **Get Instagram Account Info**.  
- **Edge cases:** None (operator initiated).

2) **Schedule Trigger** (n8n-nodes-base.scheduleTrigger, v1.2)  
- **Role:** Automated entry point.  
- **Config:** Cron expression `0 0 * * *` (daily at midnight).  
- **Outputs:** To **Get Instagram Account Info**.  
- **Edge cases:** Timezone depends on n8n instance settings; missed runs if instance is down.

---

### 2.2 Account Setup & Configuration
**Overview:** Retrieves Instagram account metadata (id, username) and sets key configuration values used throughout the workflow.

**Nodes involved:**
- Get Instagram Account Info
- Configuration

**Node details**
1) **Get Instagram Account Info** (HTTP Request, v4.2)  
- **Role:** Calls Instagram Graph API to fetch `id` and `username`.  
- **Request:** `GET https://graph.instagram.com/me?fields=id,username`  
- **Auth:** Predefined credential type **httpBearerAuth** (Instagram access token).  
- **Outputs:** To **Configuration**.  
- **Failure modes:** 401/403 if token invalid/expired; 400 if permissions/scopes insufficient.

2) **Configuration** (Set, v3.4)  
- **Role:** Creates a “config object” shared by expressions downstream.  
- **Key fields set:**
  - `accountId` = `{{$json.id}}`
  - `accountUsername` = `{{$json.username}}`
  - `googleDriveFolderId` = hardcoded placeholder `YOUR_GOOGLE_DRIVE_ID` (must be replaced)
  - `maxVideosPerRun` = 100
  - `waitBetweenDownloads` = 5 (seconds)
  - `metadataFileName` = `instagram-backup-metadata.json`
- **Outputs:** To **Fetch Instagram Media**.  
- **Edge cases:** If `googleDriveFolderId` not updated, Drive operations will fail (404/permission errors).

---

### 2.3 Fetch & Filter Instagram Media
**Overview:** Fetches media items, sorts them from earliest to latest, splits them into separate items, and filters only VIDEO/REELS.

**Nodes involved:**
- Fetch Instagram Media
- Sort Earliest to Latest
- Split Out Media Items
- Filter Videos Only

**Node details**
1) **Fetch Instagram Media** (HTTP Request, v4.2)  
- **Role:** Retrieves media list from Instagram Graph API.  
- **Request:**  
  `GET https://graph.instagram.com/me/media?fields=id,media_type,media_url,permalink,caption,timestamp,thumbnail_url&limit={{ $('Configuration').item.json.maxVideosPerRun }}`
- **Auth:** httpBearerAuth.  
- **Outputs:** To **Sort Earliest to Latest**.  
- **Edge cases / failures:** Pagination is not followed (only one page); if you have more than `limit`, older items won’t be fetched. Token errors as above.

2) **Sort Earliest to Latest** (Code, v2)  
- **Role:** Sorts `response.data` by `timestamp` ascending.  
- **Input:** Single item with `json.data` array from Instagram.  
- **Output:** `{ json: { data: sortedMediaItems, paging } }`  
- **Edge cases:** Missing/invalid timestamps could produce `Invalid Date` and unstable sorting.

3) **Split Out Media Items** (Split Out, v1)  
- **Role:** Converts the `data` array into individual items (one per media post).  
- **Config:** `fieldToSplitOut = "data"`  
- **Outputs:** To **Filter Videos Only**.  
- **Edge cases:** If `data` is empty/missing, produces no items (downstream blocks won’t run).

4) **Filter Videos Only** (IF, v2)  
- **Role:** Passes through only `media_type == VIDEO` OR `media_type == REELS`.  
- **True output:** To **Loop Over Videos**.  
- **False output:** Not connected (non-video items are discarded).  
- **Edge cases:** If Instagram introduces other video-like types, they will be excluded unless added.

**Sticky note coverage (block context):**
- “## Fetch & Filter”

---

### 2.4 Per-Video Dedup, Download, Upload, and Catalog Record
**Overview:** For each video item, the workflow checks whether it is already in the Data Table. If not, it waits (rate-limit control), parses caption into structured fields, downloads the media binary, uploads to Google Drive, extracts metadata (including Drive file info), and saves a backup record to the Data Table.

**Nodes involved:**
- Loop Over Videos
- Check If Already Backed Up
- IF Not Already Backed Up
- Wait
- Parse Caption
- Download Video
- Upload to Google Drive
- Extract Metadata
- Save Backup Record
- End Loop

**Node details**
1) **Loop Over Videos** (Split In Batches, v3)  
- **Role:** Iterates through video items (batching mechanism).  
- **Connections:**
  - Main output 0 → **Get All Data Table Records** (catalog update path)  
  - Main output 1 → **Check If Already Backed Up** (per-video path)
- **Important behavior:** This is an unusual wiring pattern: the loop node is used both to feed the per-video chain and also to trigger metadata rebuild steps. In practice, metadata rebuild will run whenever the loop starts emitting on output 0.  
- **Edge cases:** If batch size/options not set, defaults apply; misuse can cause unexpected execution ordering.

2) **Check If Already Backed Up** (Data Table, v1; onError=continueRegularOutput; alwaysOutputData=true)  
- **Role:** Queries Data Table for an existing record where `postId == {{$json.id}}`.  
- **Config:** Operation `get` with filter condition on `postId`.  
- **Outputs:** To **IF Not Already Backed Up**.  
- **Failure modes:** Data Table missing/renamed; permission issues; schema mismatch. With `onError=continueRegularOutput`, failures may appear as “empty”, causing duplicate backups.

3) **IF Not Already Backed Up** (IF, v2)  
- **Role:** Determines whether to proceed with backup.  
- **Condition:** Checks if `$('Check If Already Backed Up').first().json` is “empty”.  
- **True path:** To **Wait** (proceed with backup).  
- **False path:** To **End Loop** (skip already-backed items).  
- **Edge cases:** If Data Table node returns a non-empty placeholder on error, logic may misbehave. The current condition is somewhat brittle because it evaluates the entire returned JSON object for emptiness rather than explicitly checking for a field like `postId`.

4) **Wait** (Wait, v1.1)  
- **Role:** Rate-limiting between downloads.  
- **Config:** `amount = {{ $('Configuration').item.json.waitBetweenDownloads }}` (seconds).  
- **Outputs:** To **Parse Caption**.  
- **Edge cases:** Very low wait may hit Instagram/Drive rate limits; very high wait slows runs.

5) **Parse Caption** (Code, v2)  
- **Role:** Converts `caption` into structured fields:
  - `title`: caption before first `#` (or full caption if no hashtags)
  - `description`: from first hashtag onward
  - `tagList`: all hashtags (without `#`)
  - `descriptionFull`: full caption
- **Inputs:** References `$('Filter Videos Only').item.json.caption`.  
- **Outputs:** To **Download Video**.  
- **Edge cases:** Captions with non-`\w` hashtag characters (e.g., accented letters) may not be captured by `/#\w+/g`.

6) **Download Video** (HTTP Request, v4.2)  
- **Role:** Downloads the video file from `media_url`.  
- **Config:** Response format = file (binary).  
- **URL:** `{{ $('Filter Videos Only').item.json.media_url }}`  
- **Outputs:** To **Upload to Google Drive**.  
- **Failure modes:** Media URL expired/unavailable; large file timeout; HTTP 403/404; bandwidth constraints.

7) **Upload to Google Drive** (Google Drive, v3)  
- **Role:** Uploads the downloaded binary to Google Drive folder.  
- **Name:** `={{ $json.title }}-{{ $('Loop Over Videos').item.json.id }}.mp4`  
  - Note: `$json.title` here depends on upstream binary item’s JSON; if the download node doesn’t carry `title`, the filename may become `-<id>.mp4` unless n8n merges JSON.  
- **Folder:** `={{ $('Configuration').item.json.googleDriveFolderId }}`  
- **Credentials:** Google Drive OAuth2.  
- **Outputs:** To **Extract Metadata**.  
- **Failure modes:** Wrong folder ID; insufficient permissions; upload size limits; token refresh issues.

8) **Extract Metadata** (Set, v3.4)  
- **Role:** Builds a unified metadata record combining:
  - Instagram post data (id, permalink, timestamp, media_type)
  - Parsed caption fields
  - Google Drive uploaded file info (`id`, `name`)
  - `backedUpAt = {{$now.toISO()}}`
- **Key expressions:**  
  - `postId = {{ $('Filter Videos Only').item.json.id }}`  
  - `googleDriveFileId = {{ $json.id }}` (from Drive upload result)  
  - `title/description/tagList/descriptionFull` from **Parse Caption**  
- **Outputs:** To **Save Backup Record**.  
- **Edge cases:** `tagList` is set as **array** here, but the Data Table schema expects `tagList` as **string** (see next node). This can cause coercion issues or store `[object Object]`-like output depending on n8n behavior.

9) **Save Backup Record** (Data Table, v1)  
- **Role:** Persists backup metadata to the Data Table “Instagram Video Backups”.  
- **Operation:** `get` is not used here; it writes columns via “define below” mapping.  
- **Column mapping highlights:**
  - `postId`, `accountId`, `title`, `description`, `descriptionFull`
  - `tagList` is set to `={{ $json.description }}` (this looks like a bug; it stores description into tagList)
- **Outputs:** To **End Loop**.  
- **Failure modes:** Duplicate key if `postId` is treated as unique externally; schema mismatch; table permissions.

10) **End Loop** (NoOp, v1)  
- **Role:** Loop control/termination point.  
- **Outputs:** Back to **Loop Over Videos** to fetch next batch.  
- **Edge cases:** If the split-in-batches node isn’t configured correctly, looping may stop early or repeat.

**Sticky note coverage (block context):**
- “## Process Videos”

---

### 2.5 Update JSON Catalog (Drive file + cleanup)
**Overview:** Exports the Data Table contents into a JSON file, uploads it to Google Drive as `metadataFileName`, then lists duplicates with the same name in the target folder and deletes older ones (keeps newest).

**Nodes involved:**
- Get All Data Table Records
- Build Metadata from Data Table
- Upload New Metadata File
- List All Metadata Files
- Find Old Metadata Files
- Loop Over Old Files
- Delete Old Metadata File
- End Cleanup Loop

**Node details**
1) **Get All Data Table Records** (Data Table, v1; alwaysOutputData=true)  
- **Role:** Reads all rows from “Instagram Video Backups”.  
- **Config:** Operation `get`, `returnAll=true`.  
- **Outputs:** To **Build Metadata from Data Table**.  
- **Failure modes:** Table missing; large table could impact memory/runtime.

2) **Build Metadata from Data Table** (Code, v2)  
- **Role:** Builds a structured JSON catalog and converts it to binary for upload.  
- **Metadata structure:**
  - `lastUpdated` ISO timestamp
  - `totalVideos`
  - `videos` = array of table records
- **Also attempts:** Fetch existing file id from **Search for Metadata File** node.  
  - **Issue:** There is **no node named “Search for Metadata File”** in this workflow. The `try/catch` will always fall back to “no existing metadata file found”, but it will still log and proceed.
- **Binary output:** `binary.data` with `mimeType: application/json`, `fileName` from Configuration.  
- **Outputs:** To **Upload New Metadata File**.  
- **Failure modes:** Large JSON could exceed memory; missing Data Table data yields empty catalog.

3) **Upload New Metadata File** (Google Drive, v3)  
- **Role:** Uploads the JSON binary to Google Drive folder.  
- **Name:** `={{ $('Build Metadata from Data Table').item.json.fileName }}`  
- **Folder:** from `Configuration.googleDriveFolderId`  
- **Inputs override:** Takes items directly from **Build Metadata from Data Table** (ensures binary present).  
- **Outputs:** To **List All Metadata Files**.  
- **Edge cases:** Will create multiple files over time with same name unless configured to update existing file. Because “existingFileId” isn’t used, it always uploads a new file.

4) **List All Metadata Files** (HTTP Request, v4.2; alwaysOutputData=true)  
- **Role:** Lists Drive files matching the catalog name in the chosen folder.  
- **Request:** `GET https://www.googleapis.com/drive/v3/files` with query params:
  - `q = name='{{ metadataFileName }}' and '{{ folderId }}' in parents and trashed=false`
  - `fields = files(id,name,createdTime)`
  - `orderBy = createdTime`
- **Auth:** Google Drive OAuth2 (as predefined credential).  
- **Outputs:** To **Find Old Metadata Files**.  
- **Failure modes:** Drive API permissions; query syntax errors; large result sets.

5) **Find Old Metadata Files** (Code, v2)  
- **Role:** Keeps newest file, returns a list of older ones to delete.  
- **Logic:**
  - If `<=1` file: returns `[]` (no deletion)
  - Sorts by `createdTime`
  - Returns all except newest as `{fileId, fileName, createdTime}`  
- **Outputs:** To **Loop Over Old Files**.  
- **Edge cases:** If API doesn’t return `createdTime` consistently, sorting may fail.

6) **Loop Over Old Files** (Split In Batches, v3)  
- **Role:** Iterates over files-to-delete list.  
- **Outputs:**
  - Output 0 → **End Cleanup Loop** (batch complete / stop path)
  - Output 1 → **Delete Old Metadata File** (each file)  
- **Edge cases:** Default batch sizing may delete many files quickly; consider rate limits.

7) **Delete Old Metadata File** (Google Drive, v3)  
- **Role:** Deletes a Drive file by ID.  
- **Operation:** `deleteFile`  
- **FileId:** `={{ $json.fileId }}`  
- **Outputs:** Back to **Loop Over Old Files** (continue).  
- **Failure modes:** 404 if already deleted; permissions; Drive trash vs permanent deletion behavior.

8) **End Cleanup Loop** (NoOp, v1)  
- **Role:** Cleanup loop terminator.  
- **Outputs:** None.

**Sticky note coverage (block context):**
- “## Update JSON Catalog”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Setup Instructions | Sticky Note | Workspace documentation / setup guidance | — | — | ## Instagram Video Backup to Google Drive … Daily at midnight (customizable) |
| Fetch & Filter | Sticky Note | Section header | — | — | ## Fetch & Filter |
| Process Videos | Sticky Note | Section header | — | — | ## Process Videos |
| Update JSON Catalog | Sticky Note | Section header | — | — | ## Update JSON Catalog |
| Manual Trigger | Manual Trigger | Manual entry point | — | Get Instagram Account Info |  |
| Schedule Trigger | Schedule Trigger | Scheduled entry point (daily midnight) | — | Get Instagram Account Info |  |
| Get Instagram Account Info | HTTP Request | Fetch IG account id/username | Manual Trigger; Schedule Trigger | Configuration | ## Fetch & Filter |
| Configuration | Set | Central configuration values | Get Instagram Account Info | Fetch Instagram Media | ## Fetch & Filter |
| Fetch Instagram Media | HTTP Request | Fetch IG media list | Configuration | Sort Earliest to Latest | ## Fetch & Filter |
| Sort Earliest to Latest | Code | Sort media by timestamp ascending | Fetch Instagram Media | Split Out Media Items | ## Fetch & Filter |
| Split Out Media Items | Split Out | One item per media object | Sort Earliest to Latest | Filter Videos Only | ## Fetch & Filter |
| Filter Videos Only | IF | Keep only VIDEO/REELS | Split Out Media Items | Loop Over Videos | ## Fetch & Filter |
| Loop Over Videos | Split In Batches | Iterate through video items; drives per-item branch | Filter Videos Only; End Loop | Get All Data Table Records; Check If Already Backed Up | ## Process Videos |
| Check If Already Backed Up | Data Table | Dedup check by postId | Loop Over Videos | IF Not Already Backed Up | ## Process Videos |
| IF Not Already Backed Up | IF | Branch: process new vs skip existing | Check If Already Backed Up | Wait; End Loop | ## Process Videos |
| Wait | Wait | Rate limit between downloads | IF Not Already Backed Up | Parse Caption | ## Process Videos |
| Parse Caption | Code | Extract title/description/hashtags | Wait | Download Video | ## Process Videos |
| Download Video | HTTP Request | Download media_url as binary | Parse Caption | Upload to Google Drive | ## Process Videos |
| Upload to Google Drive | Google Drive | Upload video file to Drive folder | Download Video | Extract Metadata | ## Process Videos |
| Extract Metadata | Set | Combine IG + caption + Drive file info | Upload to Google Drive | Save Backup Record | ## Process Videos |
| Save Backup Record | Data Table | Persist record into Data Table | Extract Metadata | End Loop | ## Process Videos |
| End Loop | NoOp | Loop terminator; continues batching | IF Not Already Backed Up; Save Backup Record | Loop Over Videos | ## Process Videos |
| Get All Data Table Records | Data Table | Read all records to build catalog | Loop Over Videos | Build Metadata from Data Table | ## Update JSON Catalog |
| Build Metadata from Data Table | Code | Build JSON catalog + binary payload | Get All Data Table Records | Upload New Metadata File | ## Update JSON Catalog |
| Upload New Metadata File | Google Drive | Upload JSON catalog to Drive | Build Metadata from Data Table | List All Metadata Files | ## Update JSON Catalog |
| List All Metadata Files | HTTP Request | List catalog files with same name in folder | Upload New Metadata File | Find Old Metadata Files | ## Update JSON Catalog |
| Find Old Metadata Files | Code | Determine which older catalog files to delete | List All Metadata Files | Loop Over Old Files | ## Update JSON Catalog |
| Loop Over Old Files | Split In Batches | Iterate deletions | Find Old Metadata Files; Delete Old Metadata File | End Cleanup Loop; Delete Old Metadata File | ## Update JSON Catalog |
| Delete Old Metadata File | Google Drive | Delete old JSON catalog file | Loop Over Old Files | Loop Over Old Files | ## Update JSON Catalog |
| End Cleanup Loop | NoOp | Cleanup loop end | Loop Over Old Files | — | ## Update JSON Catalog |

---

## 4. Reproducing the Workflow from Scratch

1) **Create triggers**
   1. Add **Manual Trigger** node.
   2. Add **Schedule Trigger** node:
      - Interval → Cron expression: `0 0 * * *`

2) **Add Instagram authentication**
   1. Create an **HTTP Bearer Auth** credential in n8n:
      - Token = Instagram Graph API access token with permissions to read media.

3) **Get Instagram account info**
   1. Add **HTTP Request** node named **Get Instagram Account Info**:
      - Method: GET
      - URL: `https://graph.instagram.com/me?fields=id,username`
      - Authentication: Predefined credential type → **httpBearerAuth**
   2. Connect **Manual Trigger → Get Instagram Account Info**
   3. Connect **Schedule Trigger → Get Instagram Account Info**

4) **Configuration**
   1. Add **Set** node named **Configuration** with fields:
      - `accountId` = `{{$json.id}}`
      - `accountUsername` = `{{$json.username}}`
      - `googleDriveFolderId` = *(your Drive folder ID)*
      - `maxVideosPerRun` = `100`
      - `waitBetweenDownloads` = `5`
      - `metadataFileName` = `instagram-backup-metadata.json`
   2. Connect **Get Instagram Account Info → Configuration**

5) **Fetch and prepare Instagram media**
   1. Add **HTTP Request** node **Fetch Instagram Media**:
      - GET
      - URL: `https://graph.instagram.com/me/media?fields=id,media_type,media_url,permalink,caption,timestamp,thumbnail_url&limit={{ $('Configuration').item.json.maxVideosPerRun }}`
      - Auth: httpBearerAuth credential
   2. Add **Code** node **Sort Earliest to Latest** (sort `data` by timestamp ascending).
   3. Add **Split Out** node **Split Out Media Items**:
      - Field to split out: `data`
   4. Add **IF** node **Filter Videos Only**:
      - Condition (OR):
        - `{{$json.media_type}} == "VIDEO"`
        - `{{$json.media_type}} == "REELS"`
   5. Connect:  
      **Configuration → Fetch Instagram Media → Sort Earliest to Latest → Split Out Media Items → Filter Videos Only**

6) **Create the Data Table**
   1. In n8n, create a Data Table named **Instagram Video Backups** with fields:
      - `postId` (string) *(use as primary/unique key conceptually)*
      - `accountId` (string)
      - `title` (string)
      - `description` (string)
      - `descriptionFull` (string)
      - `tagList` (string)

7) **Per-video processing (loop, dedup, download, upload, save record)**
   1. Add **Split In Batches** node **Loop Over Videos** after **Filter Videos Only**.
   2. Add **Data Table** node **Check If Already Backed Up**:
      - Operation: Get
      - Filter: `postId` equals `{{$json.id}}`
      - Enable **Always Output Data**
      - Set **On Error** to “Continue (regular output)” (optional but matches workflow)
   3. Add **IF** node **IF Not Already Backed Up**:
      - Condition: treat output of **Check If Already Backed Up** as empty (replicate current behavior)  
        *Recommended improvement:* check something explicit like `{{$('Check If Already Backed Up').first().json.postId}}` is empty.
   4. Add **Wait** node **Wait**:
      - Amount: `{{ $('Configuration').item.json.waitBetweenDownloads }}`
   5. Add **Code** node **Parse Caption** (title/description/tag extraction).
   6. Add **HTTP Request** node **Download Video**:
      - URL: `{{ $('Filter Videos Only').item.json.media_url }}`
      - Response: File (binary)
   7. Add **Google Drive** node **Upload to Google Drive**:
      - Credentials: Google Drive OAuth2
      - Operation: Upload
      - Folder ID: `{{ $('Configuration').item.json.googleDriveFolderId }}`
      - Name: `{{ $json.title }}-{{ $('Loop Over Videos').item.json.id }}.mp4`
   8. Add **Set** node **Extract Metadata**:
      - `accountId` = `{{ $('Configuration').item.json.accountId }}`
      - `postId` = `{{ $('Filter Videos Only').item.json.id }}`
      - `googleDriveFileId` = `{{ $json.id }}`
      - `googleDriveFileName` = `{{ $json.name }}`
      - `permalink` = `{{ $('Filter Videos Only').item.json.permalink }}`
      - `title`/`description`/`tagList`/`descriptionFull` = from **Parse Caption**
      - `instagramTimestamp` = `{{ $('Filter Videos Only').item.json.timestamp }}`
      - `mediaType` = `{{ $('Filter Videos Only').item.json.media_type }}`
      - `backedUpAt` = `{{ $now.toISO() }}`
   9. Add **Data Table** node **Save Backup Record**:
      - Operation: Insert/Create (map fields to columns)
      - Map: postId/accountId/title/description/descriptionFull/tagList  
      *Note:* In the provided workflow, `tagList` is incorrectly mapped to `description`. Fix by using `{{ Array.isArray($json.tagList) ? $json.tagList.join(',') : $json.tagList }}` if your table expects string.
   10. Add **NoOp** node **End Loop**.
   11. Connect the per-video chain:
      - **Filter Videos Only → Loop Over Videos**
      - **Loop Over Videos (item output) → Check If Already Backed Up → IF Not Already Backed Up**
      - **IF true → Wait → Parse Caption → Download Video → Upload to Google Drive → Extract Metadata → Save Backup Record → End Loop → Loop Over Videos**
      - **IF false → End Loop → Loop Over Videos**

8) **Build and upload JSON metadata catalog**
   1. Add **Data Table** node **Get All Data Table Records**:
      - Operation: Get
      - Return All: true
   2. Add **Code** node **Build Metadata from Data Table**:
      - Read all items from **Get All Data Table Records**
      - Produce:
        - JSON: `{ fileName: metadataFileName }`
        - Binary: JSON content as `application/json`
      - (Do not reference a missing “Search for Metadata File” node unless you add it.)
   3. Add **Google Drive** node **Upload New Metadata File**:
      - Upload binary
      - Name: `{{ $('Build Metadata from Data Table').item.json.fileName }}`
      - Folder: `{{ $('Configuration').first().json.googleDriveFolderId }}`
      - Ensure binary input is used (configure input or use “Inputs override” equivalent).
   4. Connect:
      - **Loop Over Videos (the other output / or a dedicated branch) → Get All Data Table Records → Build Metadata from Data Table → Upload New Metadata File**

9) **Cleanup old catalog duplicates**
   1. Add **HTTP Request** node **List All Metadata Files** (Google Drive API):
      - GET `https://www.googleapis.com/drive/v3/files`
      - Auth: Google Drive OAuth2
      - Query params:
        - `q`: `name='{{ $('Configuration').first().json.metadataFileName }}' and '{{ $('Configuration').first().json.googleDriveFolderId }}' in parents and trashed=false`
        - `fields`: `files(id,name,createdTime)`
        - `orderBy`: `createdTime`
   2. Add **Code** node **Find Old Metadata Files**: keep newest, return others with `fileId`.
   3. Add **Split In Batches** node **Loop Over Old Files**.
   4. Add **Google Drive** node **Delete Old Metadata File**:
      - Operation: Delete file
      - File ID: `{{ $json.fileId }}`
   5. Add **NoOp** node **End Cleanup Loop**.
   6. Connect:
      - **Upload New Metadata File → List All Metadata Files → Find Old Metadata Files → Loop Over Old Files**
      - **Loop Over Old Files → Delete Old Metadata File → Loop Over Old Files**
      - **Loop Over Old Files (done output) → End Cleanup Loop**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a dedicated Google Drive folder and copy its folder ID from the URL | Setup Instructions sticky note |
| Create Data Table “Instagram Video Backups” with fields: postId, accountId, title, description, descriptionFull, tagList | Setup Instructions sticky note |
| Credentials needed: Instagram HTTP Bearer Auth token; Google Drive OAuth2 | Setup Instructions sticky note |
| Configure `googleDriveFolderId` in the Configuration node; default `maxVideosPerRun=100` | Setup Instructions sticky note |
| Recommended testing: run Manual Trigger first, verify uploads and JSON catalog creation | Setup Instructions sticky note |
| Schedule runs daily at midnight; customizable | Setup Instructions sticky note |

### Notable implementation issues to be aware of (from the provided workflow)
- **Missing node reference:** `Build Metadata from Data Table` references **Search for Metadata File**, but that node does not exist. As written, it always behaves as “no existing file found.”
- **tagList mapping bug:** `Save Backup Record.tagList` is assigned from `{{$json.description}}` instead of the hashtag list.
- **No pagination:** Instagram media fetch does not iterate through `paging.next`, so older posts beyond the limit won’t be backed up.
- **Potential filename issue:** Upload node uses `{{$json.title}}` even though the immediate upstream item is the download result; ensure `title` exists at that point (or reference `$('Parse Caption').item.json.title` explicitly).