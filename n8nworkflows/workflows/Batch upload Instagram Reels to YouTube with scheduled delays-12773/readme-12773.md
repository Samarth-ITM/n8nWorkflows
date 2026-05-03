Batch upload Instagram Reels to YouTube with scheduled delays

https://n8nworkflows.xyz/workflows/batch-upload-instagram-reels-to-youtube-with-scheduled-delays-12773


# Batch upload Instagram Reels to YouTube with scheduled delays

## 1. Workflow Overview

**Purpose:**  
This workflow runs on a schedule, fetches the latest Instagram media posts, filters to videos/Reels only, deduplicates against a persistent Data Table, then uploads each new video to YouTube with a configurable delay between uploads. It also auto-generates YouTube-friendly titles, descriptions, and tags from the Instagram caption.

**Primary use cases:**
- Cross-posting Instagram Reels/videos to YouTube automatically
- Maintaining a lightweight “already uploaded” registry (no spreadsheet queue)
- Batch uploading while rate-limiting uploads via delays

### 1.1 Scheduled Start & Central Configuration
Runs daily via cron, then loads all runtime settings from a single Set node.

### 1.2 Fetch & Filter Instagram Media
Calls Instagram Graph API to retrieve recent media, splits the response list into items, and keeps only VIDEO/REELS.

### 1.3 Batch Loop + Deduplication
Processes items sequentially. Each item is checked against a Data Table (postId registry) to avoid re-uploading.

### 1.4 Delay, Transform Metadata, Download, Upload, Persist Record
Waits between uploads, derives title/tags/description, downloads the video file, uploads to YouTube, and stores the mapping (Instagram postId → YouTube videoId).

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled Start & Central Configuration

**Overview:**  
Triggers once per day and defines the workflow’s behavior (delay, title length, YouTube category/privacy, etc.) in one place.

**Nodes involved:**
- Schedule Trigger
- Configuration

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — starts the workflow on a cron schedule.
- **Configuration choices:** Cron expression `0 0 * * *` (daily at 00:00).
- **Connections:** Outputs to **Configuration**.
- **Version notes:** typeVersion `1.2`.
- **Failure modes / edge cases:**  
  - Instance timezone affects “midnight”; ensure n8n timezone matches intent.

#### Node: Configuration
- **Type / role:** `Set` — provides a single JSON config object used by later nodes.
- **Configuration choices (interpreted):**
  - `includeSourceLink` (boolean): append Instagram permalink to YouTube description
  - `waitTimeoutSeconds` (number): delay between uploads (in seconds)
  - `maxTitleLength` (number): title truncation limit
  - `categoryId` (string): YouTube category (e.g., `"22"`)
  - `privacyStatus` (string): `public | private | unlisted`
  - `notifySubscribers` (boolean)
  - `defaultLanguage` (string): e.g., `en`
  - `ageRestricted` (boolean): present but **not currently applied** to the YouTube upload node in this workflow
- **Key expressions/variables:** later referenced as `$('Configuration').item.json.<field>` or `$('Configuration').first().json`.
- **Connections:** Outputs to **Fetch Instagram Posts**.
- **Version notes:** typeVersion `3.4`.
- **Failure modes / edge cases:**
  - Invalid `categoryId`/`privacyStatus` values can cause YouTube upload errors.
  - `waitTimeoutSeconds` should be numeric; otherwise Wait node expression can fail.

---

### 2.2 Fetch & Filter Instagram Media

**Overview:**  
Fetches up to 25 recent Instagram media entries and filters down to video formats suitable for YouTube upload.

**Nodes involved:**
- Fetch Instagram Posts
- Split Out
- Filter Videos Only

#### Node: Fetch Instagram Posts
- **Type / role:** `HTTP Request` — calls Instagram Graph API.
- **Configuration choices (interpreted):**
  - **Method:** GET (implicit)
  - **URL:** `https://graph.instagram.com/me/media`
  - **Query (JSON mode):**
    - `fields`: `id,caption,media_url,media_type,timestamp,permalink`
    - `limit`: `25`
  - **Auth:** Predefined credential type `httpBearerAuth` (Bearer token).
  - **alwaysOutputData:** enabled (workflow continues even if node yields no items or errors depending on n8n settings).
- **Inputs/outputs:**  
  - Input from **Configuration**  
  - Output to **Split Out**
- **Version notes:** typeVersion `4.2`.
- **Failure modes / edge cases:**
  - Expired/invalid access token → 401/400 errors.
  - Missing permissions/scopes for Instagram Graph API.
  - Pagination is not handled (only first page, limit 25).
  - Some media may lack `caption` (handled later with fallback to empty string).
  - `media_url` for some IG media types may be unavailable/expired depending on API/token.

#### Node: Split Out
- **Type / role:** `Split Out` — turns an array field into one item per element.
- **Configuration choices:** splits the `data` field (`fieldToSplitOut: "data"`).
- **Connections:** Outputs to **Filter Videos Only**.
- **Version notes:** typeVersion `1`.
- **Failure modes / edge cases:**
  - If Instagram response doesn’t include `data` (error payload), this can output zero items.

#### Node: Filter Videos Only
- **Type / role:** `Filter` — keeps only posts where `media_type` is `VIDEO` or `REELS`.
- **Configuration choices:**
  - OR combinator:
    - `$json.media_type == "VIDEO"`
    - `$json.media_type == "REELS"`
- **Connections:** Outputs to **Loop Over Items**.
- **Version notes:** typeVersion `2.2`.
- **Failure modes / edge cases:**
  - Any other media type (IMAGE, CAROUSEL_ALBUM) is skipped.
  - Strict validation: if `media_type` missing or non-string, may evaluate false and be filtered out.

---

### 2.3 Batch Loop + Deduplication

**Overview:**  
Processes media items in sequence and checks whether each Instagram post was previously uploaded (using Data Table as a registry).

**Nodes involved:**
- Loop Over Items
- Check If Already Uploaded
- If Not Uploaded Yet
- Done

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — batch/loop controller.
- **Configuration choices:** default options (batch size defaults to 1 unless configured in UI).
- **Connections:**
  - First output goes to **Check If Already Uploaded** (process current item)
  - Also connected to **Done** (see edge case note below)
  - Receives loop-back input from **Save Upload Record** and from **If Not Uploaded Yet** (false branch) to continue with next items.
- **Version notes:** typeVersion `3`.
- **Failure modes / edge cases:**
  - **Wiring note:** The connection to **Done** from Loop Over Items means Done may execute on each batch emission, not only at the end. In many patterns, “Done” is connected from the second output/“No Items Left” branch; here it is connected from the main output list, so it may not represent true completion.

#### Node: Check If Already Uploaded
- **Type / role:** `Data Table` — lookup operation.
- **Configuration choices:**
  - Operation: `get`
  - Filter condition: `postId == {{$json.id}}` (using current Instagram item id)
  - Data Table: “Instagram To YouTube”
  - alwaysOutputData: enabled
- **Connections:** Outputs to **If Not Uploaded Yet**.
- **Version notes:** typeVersion `1`.
- **Failure modes / edge cases:**
  - Data Table missing / permissions / wrong table ID → node failure.
  - If the `get` operation returns an empty result, downstream logic relies on how fields are represented (see next node).

#### Node: If Not Uploaded Yet
- **Type / role:** `If` — branch based on whether a prior upload record exists.
- **Configuration choices:**
  - Condition: `postId` **notExists** (`leftValue: {{$json.postId}}`)
  - True branch (not uploaded) → **Wait**
  - False branch (already uploaded) → **Loop Over Items** (continue)
- **Connections:** As above.
- **Version notes:** typeVersion `2.2`.
- **Failure modes / edge cases:**
  - This assumes the incoming JSON from the Data Table “get” includes `postId` when a record exists. If the Data Table node returns a different shape (e.g., array of matches, or empty item with metadata), the `notExists` test may behave unexpectedly and cause re-uploads or skipping. Validate actual Data Table “get” output in your n8n version.

#### Node: Done
- **Type / role:** `Set` — emits a status message.
- **Configuration choices:** sets `status = "Workflow completed"`.
- **Connections:** none.
- **Version notes:** typeVersion `3.4`.
- **Failure modes / edge cases:**
  - As noted above, it may execute per item depending on loop wiring rather than only at the end.

---

### 2.4 Delay, Transform Metadata, Download, Upload, Persist Record

**Overview:**  
For each not-yet-uploaded Instagram video, waits a configured duration, generates YouTube metadata, downloads the video file, uploads it to YouTube, then stores an upload record in the Data Table and continues the loop.

**Nodes involved:**
- Wait
- Process Title and Tags
- Download Video
- Upload to YouTube
- Save Upload Record

#### Node: Wait
- **Type / role:** `Wait` — rate-limits uploads with a pause between items.
- **Configuration choices:** amount = `$('Configuration').item.json.waitTimeoutSeconds`
- **Connections:** Outputs to **Process Title and Tags**.
- **Version notes:** typeVersion `1.1`.
- **Failure modes / edge cases:**
  - Very large waits keep executions open longer; consider queue mode/workers if high volume.
  - If `waitTimeoutSeconds` is missing or not a number, expression evaluation can fail.

#### Node: Process Title and Tags
- **Type / role:** `Code` — transforms caption into title/description/tags and passes through media URL.
- **Configuration choices (logic):**
  - Reads `config` from `$('Configuration').first().json`
  - Reads Instagram post data from `$('Loop Over Items').item.json`:
    - `caption`, `id`, `media_url`, `permalink`
  - Extracts hashtags via regex `/#[\w\u00C0-\u024F]+/gu`
  - Title rules:
    1. Use caption text before first hashtag
    2. Else first hashtag (without `#`)
    3. Else “Instagram Video”
    4. Truncate intelligently to `maxTitleLength` and append `...`
  - Description:
    - Starts as full caption
    - If `includeSourceLink`, appends “Originally posted on Instagram: <permalink>”
  - Tags:
    - From hashtags, remove `#`, filter length 2..30, take first 10
  - Outputs JSON: `postId,title,description,tags,mediaUrl,originalCaption,hashtagCount,sourceLinkIncluded`
- **Connections:** Outputs to **Download Video**.
- **Version notes:** typeVersion `2`.
- **Failure modes / edge cases:**
  - If `Loop Over Items` item is not in scope (changed wiring), `$()` references can break.
  - Captions with unusual Unicode hashtags may not match regex (only certain ranges).
  - Very long captions increase description length; YouTube has limits (description max is large, but not infinite).

#### Node: Download Video
- **Type / role:** `HTTP Request` — downloads the Instagram media file as binary.
- **Configuration choices:**
  - URL: `{{$json.mediaUrl}}` (from previous Code node output)
  - Response format: `file` (binary)
- **Connections:** Outputs to **Upload to YouTube**.
- **Version notes:** typeVersion `4.3`.
- **Failure modes / edge cases:**
  - `mediaUrl` may be expired or require headers; download can 403/404.
  - Large files may hit n8n memory/disk limits depending on binary data handling settings.
  - Network timeouts.

#### Node: Upload to YouTube
- **Type / role:** `YouTube` — uploads video content.
- **Configuration choices (interpreted):**
  - Operation: `video.upload`
  - Title: from `Process Title and Tags`
  - Description: from `Process Title and Tags`
  - Tags: joined with commas from `tags` array (`tags.join(',')`)
  - CategoryId: from Configuration (`categoryId`)
  - Privacy/status/language/notifySubscribers: from Configuration
  - Other options:
    - license: `youtube`
    - embeddable: true
    - publicStatsViewable: true
    - selfDeclaredMadeForKids: false (COPPA-related flag)
  - Region code: `US`
  - Credentials: YouTube OAuth2
- **Connections:** Outputs to **Save Upload Record**.
- **Version notes:** typeVersion `1`.
- **Failure modes / edge cases:**
  - OAuth token expired/revoked; channel permission issues.
  - YouTube API quota limits.
  - Tag formatting: the node is given a comma-separated string; confirm your n8n YouTube node expects this format for your version.
  - **Config gap:** `ageRestricted` exists in Configuration but is not applied here; if you need age restriction, add the appropriate YouTube option (if supported by the node/version).

#### Node: Save Upload Record
- **Type / role:** `Data Table` — upsert-like write to register upload.
- **Configuration choices:**
  - Operation: (Data Table “insert/update” style via column mapping; configured to match on `postId`)
  - Writes:
    - `postId` = from Process Title and Tags
    - `youtubeId` = `{{$json.id}}` (YouTube upload response video ID)
  - Matching columns: `postId` (prevents duplicates)
  - Data Table: “Instagram To YouTube”
- **Connections:** Outputs back to **Loop Over Items** to continue processing next item.
- **Version notes:** typeVersion `1`.
- **Failure modes / edge cases:**
  - If YouTube node output doesn’t include `id`, `youtubeId` would be blank and dedupe breaks.
  - Data Table write failures cause re-uploads on next run.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Daily cron entry point | — | Configuration | ## Instagram to YouTube Uploader Automatically upload Instagram videos to YouTube with scheduled intervals. … (full instructions note applies to workflow) |
| Configuration | Set | Central runtime config | Schedule Trigger | Fetch Instagram Posts | ## Configuration |
| Fetch Instagram Posts | HTTP Request | Pull recent IG media list via Graph API | Configuration | Split Out | ## Fetch & Filter Videos |
| Split Out | Split Out | Convert `data[]` into per-item entries | Fetch Instagram Posts | Filter Videos Only | ## Fetch & Filter Videos |
| Filter Videos Only | Filter | Keep only VIDEO/REELS | Split Out | Loop Over Items | ## Fetch & Filter Videos |
| Loop Over Items | Split In Batches | Sequential loop over posts | Filter Videos Only; Save Upload Record; If Not Uploaded Yet (false) | Check If Already Uploaded; Done |  |
| Check If Already Uploaded | Data Table | Lookup by postId for dedupe | Loop Over Items | If Not Uploaded Yet | ## Deduplication |
| If Not Uploaded Yet | If | Branch: upload only if not found in table | Check If Already Uploaded | Wait (true); Loop Over Items (false) | ## Deduplication |
| Wait | Wait | Delay between uploads | If Not Uploaded Yet (true) | Process Title and Tags |  |
| Process Title and Tags | Code | Build title/description/tags from caption | Wait | Download Video | ## Process & Upload |
| Download Video | HTTP Request | Download IG video as binary | Process Title and Tags | Upload to YouTube | ## Process & Upload |
| Upload to YouTube | YouTube | Upload video to YouTube | Download Video | Save Upload Record | ## Process & Upload |
| Save Upload Record | Data Table | Persist mapping postId → youtubeId | Upload to YouTube | Loop Over Items | ## Process & Upload |
| Done | Set | Emit completion status | Loop Over Items | — |  |

Sticky note content (applies broadly; preserved):  
- **Instagram to YouTube Uploader**: includes configuration JSON example, setup steps, category ID table, and notes (includeSourceLink, waitTimeoutSeconds, ageRestricted, etc.).
- **Configuration**, **Fetch & Filter Videos**, **Deduplication**, **Process & Upload** headings as block labels.

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Schedule Trigger” (Schedule Trigger node)**
   - Set **Cron** to: `0 0 * * *` (daily at midnight).

2. **Create “Configuration” (Set node)**
   - Mode: **Raw JSON**
   - Paste a config object like:
     - `includeSourceLink` (boolean)
     - `waitTimeoutSeconds` (number, e.g., 900 for 15 minutes)
     - `maxTitleLength` (number, e.g., 100)
     - `categoryId` (string, e.g., `"22"`)
     - `privacyStatus` (`public/private/unlisted`)
     - `notifySubscribers` (boolean)
     - `defaultLanguage` (string, e.g., `en`)
     - `ageRestricted` (boolean; optional—note this workflow does not apply it by default)
   - Connect: **Schedule Trigger → Configuration**

3. **Create “Fetch Instagram Posts” (HTTP Request node)**
   - Method: **GET**
   - URL: `https://graph.instagram.com/me/media`
   - Enable **Send Query Parameters**
   - Use **JSON** query parameters:
     - `fields`: `id,caption,media_url,media_type,timestamp,permalink`
     - `limit`: `25`
   - Authentication: **Bearer Auth** (predefined credential)
     - Create credential containing a valid Instagram Graph API access token.
   - Connect: **Configuration → Fetch Instagram Posts**

4. **Create “Split Out” (Split Out node)**
   - Field to split out: `data`
   - Connect: **Fetch Instagram Posts → Split Out**

5. **Create “Filter Videos Only” (Filter node)**
   - Add condition group with **OR**:
     - `{{$json.media_type}}` equals `VIDEO`
     - `{{$json.media_type}}` equals `REELS`
   - Connect: **Split Out → Filter Videos Only**

6. **Create “Loop Over Items” (Split In Batches node)**
   - Use defaults (batch size effectively 1 unless changed).
   - Connect: **Filter Videos Only → Loop Over Items**

7. **Create a Data Table in n8n**
   - Name: e.g. **Instagram To YouTube**
   - Columns:
     - `postId` (string)
     - `youtubeId` (string)

8. **Create “Check If Already Uploaded” (Data Table node)**
   - Operation: **Get**
   - Data Table: **Instagram To YouTube**
   - Filter: `postId` equals `{{$json.id}}`
   - Connect: **Loop Over Items → Check If Already Uploaded**

9. **Create “If Not Uploaded Yet” (If node)**
   - Condition: **postId** “not exists”
     - Left value: `{{$json.postId}}`
   - Connect:
     - **Check If Already Uploaded → If Not Uploaded Yet**
     - True branch → **Wait**
     - False branch → **Loop Over Items** (to continue to next item)

10. **Create “Wait” (Wait node)**
    - Amount expression: `{{$('Configuration').item.json.waitTimeoutSeconds}}`
    - Connect: **If Not Uploaded Yet (true) → Wait**

11. **Create “Process Title and Tags” (Code node)**
    - Paste logic to:
      - Read config from `$('Configuration').first().json`
      - Read the current IG item from `$('Loop Over Items').item.json`
      - Generate `title`, `description`, `tags`, and output `mediaUrl` + `postId`
    - Connect: **Wait → Process Title and Tags**

12. **Create “Download Video” (HTTP Request node)**
    - Method: GET
    - URL: `{{$json.mediaUrl}}`
    - Response: **File** (binary)
    - Connect: **Process Title and Tags → Download Video**

13. **Create “Upload to YouTube” (YouTube node)**
    - Resource: **Video**
    - Operation: **Upload**
    - Credentials: **YouTube OAuth2**
      - Connect your Google account/channel with permissions to upload.
    - Map fields:
      - Title: `{{$('Process Title and Tags').item.json.title}}`
      - Description: `{{$('Process Title and Tags').item.json.description}}`
      - Tags: `{{$('Process Title and Tags').item.json.tags.join(',')}}`
      - Category ID: `{{$('Configuration').item.json.categoryId}}`
      - Privacy status: `{{$('Configuration').item.json.privacyStatus}}`
      - Default language: `{{$('Configuration').item.json.defaultLanguage}}`
      - Notify subscribers: `{{$('Configuration').item.json.notifySubscribers}}`
      - Set other options as desired (license, embeddable, made-for-kids flag, etc.)
    - Connect: **Download Video → Upload to YouTube**

14. **Create “Save Upload Record” (Data Table node)**
    - Data Table: **Instagram To YouTube**
    - Configure column mapping:
      - `postId` = `{{$('Process Title and Tags').item.json.postId}}`
      - `youtubeId` = `{{$json.id}}` (from YouTube response)
    - Set matching column(s): `postId` (to prevent duplicates)
    - Connect: **Upload to YouTube → Save Upload Record**
    - Connect loop-back: **Save Upload Record → Loop Over Items**

15. **Create “Done” (Set node) [optional]**
    - Set field `status` to “Workflow completed”.
    - If you want true “end-of-loop” behavior, connect it to the loop’s “no items left” output (implementation depends on how you configure Split In Batches). In this provided workflow it’s connected from Loop Over Items’ main output, which may not reflect completion.
    - Connect as desired.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically upload Instagram videos to YouTube with scheduled intervals; configuration centralized in one node; uses delays to progressively upload remaining content. | Sticky Note “Instagram to YouTube Uploader” |
| Configuration JSON (includeSourceLink, waitTimeoutSeconds, maxTitleLength, categoryId, privacyStatus, notifySubscribers, defaultLanguage, ageRestricted). | Sticky Note “Instagram to YouTube Uploader” |
| Key settings: includeSourceLink may need to be false for unverified accounts; waitTimeoutSeconds controls gap; ageRestricted true for 18+ content. | Sticky Note “Instagram to YouTube Uploader” |
| Features: video-only filter, hashtag→tags extraction, Data Table deduplication, COPPA options. | Sticky Note “Instagram to YouTube Uploader” |
| Setup checklist: set Bearer Auth on IG fetch; create Data Table (postId, youtubeId); set YouTube OAuth2. | Sticky Note “Instagram to YouTube Uploader” |
| Category IDs reference: 22 People & Blogs; 24 Entertainment; 20 Gaming; 10 Music. | Sticky Note “Instagram to YouTube Uploader” |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.