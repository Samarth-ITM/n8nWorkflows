Schedule Facebook posts from Google Sheets with approval and Drive images

https://n8nworkflows.xyz/workflows/schedule-facebook-posts-from-google-sheets-with-approval-and-drive-images-13301


# Schedule Facebook posts from Google Sheets with approval and Drive images

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow schedules Facebook Page posts based on rows in a Google Sheets content calendar. It runs four times per day, selects rows that are **approved** and **for Facebook**, filters them to only those scheduled **for today**, then publishes either a **scheduled text post** or a **scheduled photo post** (image fetched from Google Drive). Finally, it writes the resulting Facebook URL back to the sheet and marks the row as **Published**.

**Target use cases:** Social media managers/agencies using Google Sheets as a publishing calendar with an approval step.

### Logical Blocks
**1.1 Schedule & Credentials**  
Trigger on a cron schedule; read Facebook Page ID and Page Access Token from a credentials/ENV sheet.

**1.2 Data Loading & Date Filtering**  
Read approved Facebook rows from the sheet and keep only items whose “Scheduled On” date equals today.

**1.3 Per-Post Loop + Routing**  
Iterate posts one-by-one, enforce “Platform = Facebook”, skip Story posts, then route to Text vs Photo publishing based on whether “Media URL” is empty.

**1.4 Photo Post Publishing + Tracking Update**  
Download image from Google Drive, schedule a Facebook photo post via Graph API, update sheet with URL + Published status.

**1.5 Text Post Publishing + Tracking Update + Merge**  
Schedule a Facebook text-only post via Graph API, update sheet with URL + Published status, merge all branches to end the loop cleanly.

---

## 2. Block-by-Block Analysis

### 2.1 Schedule & Credentials

**Overview:**  
Runs the workflow 4 times daily, then loads Facebook API credentials (Page ID and access token) from Google Sheets so the HTTP requests can authenticate.

**Nodes involved:**  
- Run Daily at Multiple Times  
- Load Facebook Credentials from Sheet  

#### Node: Run Daily at Multiple Times
- **Type / role:** Schedule Trigger; initiates runs on a cron expression.
- **Configuration (interpreted):** Cron expression `35 9-12 * * *` (at minute 35, hours 9–12 daily).
- **Inputs/outputs:** Entry node → outputs to “Load Facebook Credentials from Sheet”.
- **Version notes:** `typeVersion 1.2` schedule trigger supports cron expression UI.
- **Edge cases / failures:**
  - Instance timezone impacts “9–12” meaning (server timezone vs desired business timezone).
  - If you need exactly “9:35, 10:35, 11:35, 12:35 local time”, ensure n8n timezone matches.

#### Node: Load Facebook Credentials from Sheet
- **Type / role:** Google Sheets; reads values used as secrets/config.
- **Configuration (interpreted):**
  - DocumentId is set via an expression-like placeholder `=YOUR_SHEET_URL` (must be replaced with a real Sheet URL).
  - `sheetName` is left unspecified in the JSON (configured via “list” picker but currently empty), so in practice it must point to the sheet/tab that contains:
    - `Facebook Page ID`
    - `Facebook Page Access Token`
- **Key variables used downstream:**
  - `$node['Load Facebook Credentials from Sheet'].json['Facebook Page ID']`
  - `$node['Load Facebook Credentials from Sheet'].json['Facebook Page Access Token']`
- **Connections:** Trigger → this node → “Read Approved Facebook Posts”.
- **Credentials:** Google Sheets OAuth2 credential `TESTING_SHEET`.
- **Edge cases / failures:**
  - Missing tab selection or wrong tab schema → downstream expressions resolve to `undefined`, causing Facebook API auth errors.
  - Token expiry/revocation → Graph API calls fail with OAuth errors.
  - If the node returns multiple rows, downstream references to `.json[...]` will only use the current item; ensure this node returns exactly one credentials row (or add a “First Item” step).

**Sticky note (applies to this block):**  
“## Schedule & Credentials … Triggers 4x daily and loads Facebook credentials from Google Sheets for API authentication.”

---

### 2.2 Data Loading & Filtering

**Overview:**  
Fetches content-calendar rows that are approved and targeted for Facebook, then filters to keep only rows scheduled for “today” (date match only, ignoring time).

**Nodes involved:**  
- Read Approved Facebook Posts  
- Filter Posts Scheduled for Today  

#### Node: Read Approved Facebook Posts
- **Type / role:** Google Sheets; loads candidate post rows.
- **Configuration (interpreted):**
  - Reads from a specific tab (gid `98565607`) whose cached name is “Post URL”.
  - Filters:
    - `Approval Status` equals **Good**
    - `Platform` equals **Facebook**
- **Inputs/outputs:** Input from credentials node; outputs rows to “Filter Posts Scheduled for Today”.
- **Credentials:** Google Sheets OAuth2 credential `TESTING_SHEET`.
- **Edge cases / failures:**
  - Column names must match exactly (case and spacing): `Approval Status`, `Platform`.
  - If the sheet uses data validation values like “Approved” instead of “Good”, nothing will pass filters.
  - If the tab changes gid/name, the node may point to the wrong sheet.

#### Node: Filter Posts Scheduled for Today
- **Type / role:** Code node (JavaScript); filters items by date.
- **Configuration (interpreted):**
  - Computes `todayDateString = new Date().toISOString().split('T')[0]` (UTC-based “today”).
  - For each input item:
    - Reads scheduled field from `Scheduled On` OR `scheduled_on` OR `scheduledOn`
    - Extracts `YYYY-MM-DD` via regex `(\d{4}-\d{2}-\d{2})`
    - Keeps item if extracted date equals `todayDateString`
- **Inputs/outputs:** Receives all rows; returns only filtered rows → “Loop Through Each Post”.
- **Version notes:** `typeVersion 2` code node uses the current n8n code runtime.
- **Edge cases / failures:**
  - **Timezone bug risk:** `toISOString()` uses UTC; if your schedule is in local time, “today” may differ near midnight.
  - If “Scheduled On” is a native date in Sheets but is delivered as a locale-formatted string without `YYYY-MM-DD`, regex won’t match.
  - Items with missing/empty scheduled date are silently dropped.

**Sticky note (applies to this block):**  
“## Data Loading & Filtering … Reads approved Facebook posts from sheet, filters by today's date, and loops through each post for processing.”

---

### 2.3 Per-Post Loop + Routing

**Overview:**  
Processes posts one at a time. Ensures platform is Facebook (defensive check), skips Story posts, then routes to text-only flow if Media URL is empty, otherwise to photo flow.

**Nodes involved:**  
- Loop Through Each Post  
- Check if Platform is Facebook  
- Check if Not Story Post  
- Check if Text-Only or Photo Post  

#### Node: Loop Through Each Post
- **Type / role:** Split In Batches; iterates over items.
- **Configuration (interpreted):**
  - Uses default batch settings (no explicit batch size shown; n8n default is typically 1 unless changed in UI).
- **Connections (important):**
  - The node’s **second output (index 1)** is connected to “Check if Platform is Facebook”.
  - Its **first output (index 0)** is connected from “Merge All Post Types” back into the loop, enabling iteration/continuation.
- **Edge cases / failures:**
  - If batch size > 1, downstream nodes must be safe for multiple items at once; current sheet updates rely on per-item `row_number`.
  - If “Merge All Post Types” doesn’t output correctly, the loop may stall.

#### Node: Check if Platform is Facebook
- **Type / role:** Switch; verifies `Platform` equals Facebook.
- **Configuration (interpreted):**
  - Rule checks `{{$json.Platform}} === "Facebook"`.
- **Outputs:** Only the “match” output is connected onward to “Check if Not Story Post”. Non-matching cases are effectively dropped (no explicit “else” handling).
- **Edge cases:**
  - If column is named `Platform` but sheet provides `platform`, condition fails.
  - Trailing spaces (“Facebook ”) will fail strict equals.

#### Node: Check if Not Story Post
- **Type / role:** IF; blocks Story posts.
- **Configuration:** Condition `{{$json['Post Type']}} != "Story"`.
- **Outputs:**
  - **True** → “Check if Text-Only or Photo Post”
  - **False** → “Merge All Post Types” input index 2 (used as “skip branch” to keep the loop moving)
- **Edge cases:**
  - If “Post Type” is empty or uses another label (“FB Story”), it will not be excluded.

#### Node: Check if Text-Only or Photo Post
- **Type / role:** IF; detects whether to post text-only or photo.
- **Configuration:** Checks whether `Media URL` is **empty**.
  - True (Media URL empty) → text post scheduling
  - False → download + photo post scheduling
- **Edge cases:**
  - A Media URL containing whitespace may not count as empty depending on how the field is returned.
  - If Media URL is a Google Drive *share link* (not fileId), download may fail.

**Sticky note (applies to this block):**  
“## Post Type Routing … Checks platform and post type, then routes to appropriate publishing flow (text-only vs photo, skips Stories).”

---

### 2.4 Photo Post Publishing + Tracking Update

**Overview:**  
For posts with an image, downloads the file from Google Drive and schedules a Facebook photo post via Graph API `/photos`, then writes the resulting photo URL back to the tracking sheet.

**Nodes involved:**  
- Download Image from Google Drive  
- Schedule Facebook Photo Post  
- Update Sheet with Photo Post URL  

#### Node: Download Image from Google Drive
- **Type / role:** Google Drive; downloads binary content.
- **Configuration:**
  - Operation: **download**
  - File ID is taken from `{{$json['Media URL']}}` using “URL mode” (so it expects a Drive URL or ID supported by n8n’s resolver).
  - Output includes a binary property named **`data`** (required by next node).
- **Credentials:** Google Drive OAuth2 credential `Sourav_Singh`.
- **Edge cases / failures:**
  - If the Drive file is not shared to the authenticated user/service account, download fails (403).
  - If Media URL is not a Drive file URL/ID, resolver fails.
  - Large files may exceed n8n memory limits depending on instance settings.

#### Node: Schedule Facebook Photo Post
- **Type / role:** HTTP Request; calls Facebook Graph API to create a scheduled (unpublished) photo post.
- **Configuration:**
  - POST `https://graph.facebook.com/v24.0/{PAGE_ID}/photos`
  - Content-Type: **multipart/form-data**
  - Body params:
    - `access_token` from credentials sheet
    - `caption` from `{{$json.Caption}}`
    - `source` from binary field **data**
    - `published=false`
    - `scheduled_publish_time` computed as:
      - Parses `Scheduled On` string, normalizes to ISO-like, adds **15 minutes**, converts to UNIX seconds.
- **Key expressions:**
  - Page ID: `{{$node['Load Facebook Credentials from Sheet'].json['Facebook Page ID']}}`
  - Token: `{{$node['Load Facebook Credentials from Sheet'].json['Facebook Page Access Token']}}`
  - Scheduled time:
    ```js
    Math.floor((new Date($json['Scheduled On']
      .replace(' ', 'T')
      .replace(/-(\d{2})$/, ':$1')
    ).getTime() + 15 * 60 * 1000) / 1000)
    ```
- **Edge cases / failures:**
  - Facebook requires scheduled times to be sufficiently in the future (often at least 10 minutes) — the +15 minutes appears to mitigate this, but if the sheet time is “now”, it can still fail.
  - Date parsing is fragile: if “Scheduled On” format differs, `new Date(...)` may yield invalid date.
  - Token permissions: needs Page publish permissions (e.g., `pages_manage_posts`, `pages_read_engagement` and appropriate publish permissions depending on Meta’s current policy).
  - Graph API version pinned to `v24.0`; future changes may require version update.

#### Node: Update Sheet with Photo Post URL
- **Type / role:** Google Sheets; updates the originating row.
- **Configuration:**
  - Operation: **update**
  - Matching column: `row_number`
  - Sets:
    - `Post URL` = `https://www.facebook.com/photo/?fbid={{ $json.id }}`
    - `Approval Status` = `Published`
    - `row_number` = `{{ $('Check if Platform is Facebook').item.json.row_number }}`
  - Uses the same Posts sheet/tab (gid `98565607`), but documentId is `YOUR_SHEET_URL` placeholder (must be set).
- **Inputs/outputs:** From Graph response → to Merge All Post Types (input index 1).
- **Edge cases / failures:**
  - Requires `row_number` field to exist from the original Google Sheets read node. If not present, update will not match anything.
  - The photo URL format may not be the canonical permalink for all photo posts; consider using Graph API fields for permalink if needed.

**Sticky note (applies to this block):**  
“## Photo Post Publishing … Downloads image from Google Drive, schedules Facebook photo post via Graph API, updates sheet with published photo URL.”

---

### 2.5 Text Post Publishing + Tracking Update + Merge

**Overview:**  
For text-only posts, schedules a Facebook feed post via Graph API `/feed`, updates the sheet with the computed post URL, then merges into the loop continuation.

**Nodes involved:**  
- Schedule Facebook Text Post  
- Update Sheet with Text Post URL  
- Merge All Post Types  

#### Node: Schedule Facebook Text Post
- **Type / role:** HTTP Request; schedules an unpublished feed post.
- **Configuration:**
  - POST `https://graph.facebook.com/v24.0/{PAGE_ID}/feed`
  - Body params:
    - `message` = `{{$json.Caption}}`
    - `access_token` from credentials sheet
    - `published=false`
    - `scheduled_publish_time` computed by regex converting a format like `YYYY-MM-DD HH-MM` into an ISO string with a fixed `-05:00` offset, then UNIX seconds.
- **Key expression (time parsing):**
  ```js
  Math.floor(
    new Date(
      $json['Scheduled On']
        .replace(/(\d{4})-(\d{2})-(\d{2}) (\d{2})-(\d{2})/,
                 '$1-$2-$3T$4:$5:00-05:00')
    ).getTime() / 1000
  )
  ```
- **Edge cases / failures:**
  - Hard-coded timezone offset `-05:00` may be wrong seasonally (DST) or for your locale.
  - If “Scheduled On” uses `HH:MM` instead of `HH-MM`, the regex won’t match and scheduling time becomes invalid.
  - Same permission/version issues as photo posting.

#### Node: Update Sheet with Text Post URL
- **Type / role:** Google Sheets update of the originating row.
- **Configuration:**
  - Operation: update by `row_number`
  - Sets:
    - `Post URL` = `"https://www.facebook.com/" + $json.id.split('_')[0] + "/posts/" + $json.id.split('_')[1]`
    - `Approval Status` = `Published`
    - `row_number` from `$('Check if Platform is Facebook').item.json.row_number`
- **Edge cases / failures:**
  - Graph `/feed` returns an `id` in the form `{pageId}_{postId}`; if API returns a different structure, the split-based URL builder fails.
  - As with photo branch: if `row_number` missing, update won’t apply.

#### Node: Merge All Post Types
- **Type / role:** Merge; consolidates three possible branches to feed back into the batch loop.
- **Configuration:**
  - `numberInputs = 3`
  - Inputs used as:
    1) From “Update Sheet with Text Post URL”
    2) From “Update Sheet with Photo Post URL”
    3) From “Check if Not Story Post” (false branch: skipped story)
- **Connections:**
  - Output goes to “Loop Through Each Post” main input to continue the batching cycle.
- **Edge cases:**
  - Merge node behavior depends on merge mode defaults; with multi-input merging, mismatched arrival timing can matter. This workflow uses it primarily as a “join point” rather than data-combining logic.

**Sticky note (applies to this block):**  
“## Text Post Publishing … Schedules Facebook text-only post via Graph API, updates sheet with published post URL, merges with other branches.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow description & setup notes | — | — | ## Facebook Post Scheduler with Approval Workflow / How it works / Setup steps |
| Run Daily at Multiple Times | Schedule Trigger | Run workflow 4x daily | — | Load Facebook Credentials from Sheet | ## Schedule & Credentials / Triggers 4x daily and loads Facebook credentials from Google Sheets for API authentication. |
| Load Facebook Credentials from Sheet | Google Sheets | Load Page ID + access token | Run Daily at Multiple Times | Read Approved Facebook Posts | ## Schedule & Credentials / Triggers 4x daily and loads Facebook credentials from Google Sheets for API authentication. |
| Read Approved Facebook Posts | Google Sheets | Read approved FB rows | Load Facebook Credentials from Sheet | Filter Posts Scheduled for Today | ## Data Loading & Filtering / Reads approved Facebook posts from sheet, filters by today's date, and loops through each post for processing. |
| Filter Posts Scheduled for Today | Code | Keep only posts scheduled for today | Read Approved Facebook Posts | Loop Through Each Post | ## Data Loading & Filtering / Reads approved Facebook posts from sheet, filters by today's date, and loops through each post for processing. |
| Loop Through Each Post | Split In Batches | Iterate rows | Filter Posts Scheduled for Today; Merge All Post Types | Check if Platform is Facebook | ## Data Loading & Filtering / Reads approved Facebook posts from sheet, filters by today's date, and loops through each post for processing. |
| Check if Platform is Facebook | Switch | Defensive platform check | Loop Through Each Post | Check if Not Story Post | ## Post Type Routing / Checks platform and post type, then routes to appropriate publishing flow (text-only vs photo, skips Stories). |
| Check if Not Story Post | IF | Skip Stories | Check if Platform is Facebook | Check if Text-Only or Photo Post; Merge All Post Types | ## Post Type Routing / Checks platform and post type, then routes to appropriate publishing flow (text-only vs photo, skips Stories). |
| Check if Text-Only or Photo Post | IF | Branch: text vs photo | Check if Not Story Post | Schedule Facebook Text Post; Download Image from Google Drive | ## Post Type Routing / Checks platform and post type, then routes to appropriate publishing flow (text-only vs photo, skips Stories). |
| Download Image from Google Drive | Google Drive | Download image binary | Check if Text-Only or Photo Post | Schedule Facebook Photo Post | ## Photo Post Publishing / Downloads image from Google Drive, schedules Facebook photo post via Graph API, updates sheet with published photo URL. |
| Schedule Facebook Photo Post | HTTP Request | Schedule photo post via Graph API | Download Image from Google Drive | Update Sheet with Photo Post URL | ## Photo Post Publishing / Downloads image from Google Drive, schedules Facebook photo post via Graph API, updates sheet with published photo URL. |
| Update Sheet with Photo Post URL | Google Sheets | Write photo URL + Published | Schedule Facebook Photo Post | Merge All Post Types | ## Photo Post Publishing / Downloads image from Google Drive, schedules Facebook photo post via Graph API, updates sheet with published photo URL. |
| Schedule Facebook Text Post | HTTP Request | Schedule text post via Graph API | Check if Text-Only or Photo Post | Update Sheet with Text Post URL | ## Text Post Publishing / Schedules Facebook text-only post via Graph API, updates sheet with published post URL, merges with other branches. |
| Update Sheet with Text Post URL | Google Sheets | Write post URL + Published | Schedule Facebook Text Post | Merge All Post Types | ## Text Post Publishing / Schedules Facebook text-only post via Graph API, updates sheet with published post URL, merges with other branches. |
| Merge All Post Types | Merge | Join branches + feed loop | Update Sheet with Text Post URL; Update Sheet with Photo Post URL; Check if Not Story Post | Loop Through Each Post | ## Text Post Publishing / Schedules Facebook text-only post via Graph API, updates sheet with published post URL, merges with other branches. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Schedule & Credentials / Triggers 4x daily and loads Facebook credentials from Google Sheets for API authentication. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Data Loading & Filtering / Reads approved Facebook posts from sheet, filters by today's date, and loops through each post for processing. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Post Type Routing / Checks platform and post type, then routes to appropriate publishing flow (text-only vs photo, skips Stories). |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## Photo Post Publishing / Downloads image from Google Drive, schedules Facebook photo post via Graph API, updates sheet with published photo URL. |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## Text Post Publishing / Schedules Facebook text-only post via Graph API, updates sheet with published post URL, merges with other branches. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the trigger**
   1. Add node: **Schedule Trigger**
   2. Set **Cron expression** to: `35 9-12 * * *`
   3. Ensure n8n instance timezone matches your intended schedule timezone.

2) **Prepare Google Sheets structure**
   1. Create a Google Sheet (content calendar) with columns at minimum:
      - `Scheduled On`
      - `Platform`
      - `Post Type`
      - `Caption`
      - `Media URL`
      - `Approval Status`
      - `Post URL`
   2. Ensure the Google Sheets read node will include `row_number` in outputs (n8n Google Sheets node typically provides it automatically).

3) **Create a credentials/config tab**
   1. In the same spreadsheet (or another), create a tab (commonly named `.env` or similar) containing a single row with:
      - `Facebook Page ID`
      - `Facebook Page Access Token`
   2. Make sure the node that reads it returns exactly one item.

4) **Add: Load Facebook Credentials from Sheet (Google Sheets node)**
   1. Add node: **Google Sheets**
   2. Configure **Credentials:** Google Sheets OAuth2 (connect your Google account)
   3. **Document:** select your sheet by URL
   4. **Sheet/Tab:** select the credentials tab (the one with Page ID/token)
   5. Operation should be a read/get-all style (as in your build). Confirm output fields match the downstream expressions exactly.

5) **Add: Read Approved Facebook Posts (Google Sheets node)**
   1. Add node: **Google Sheets**
   2. Same Google Sheets credential
   3. Select the content calendar tab
   4. Add filters:
      - `Approval Status` = `Good`
      - `Platform` = `Facebook`

6) **Add: Filter Posts Scheduled for Today (Code node)**
   1. Add node: **Code**
   2. Paste the filtering logic (date extraction + equality check against “today”).
   3. If you need local-date comparisons, adjust to local timezone (important if your instance timezone isn’t UTC).

7) **Add: Loop Through Each Post (Split In Batches)**
   1. Add node: **Split In Batches**
   2. Batch size: set to **1** (recommended for safe row updates and per-post API calls).
   3. Connect:
      - Code node → Split In Batches (input)
      - Later you will connect Merge → Split In Batches (to continue loop)

8) **Add routing: Platform / Story / Media**
   1. Add node: **Switch** named “Check if Platform is Facebook”
      - Rule: `{{$json.Platform}}` equals `Facebook`
   2. Add node: **IF** named “Check if Not Story Post”
      - Condition: `{{$json['Post Type']}}` not equals `Story`
   3. Add node: **IF** named “Check if Text-Only or Photo Post”
      - Condition: `{{$json['Media URL']}}` is empty

9) **Text branch (Media URL empty)**
   1. Add node: **HTTP Request** named “Schedule Facebook Text Post”
      - Method: POST
      - URL: `https://graph.facebook.com/v24.0/{{ $node['Load Facebook Credentials from Sheet'].json['Facebook Page ID'] }}/feed`
      - Body params:
        - `message` = `{{$json.Caption}}`
        - `access_token` = `{{$node['Load Facebook Credentials from Sheet'].json['Facebook Page Access Token']}}`
        - `published` = `false`
        - `scheduled_publish_time` = expression that parses your “Scheduled On” and converts to UNIX seconds
   2. Add node: **Google Sheets** named “Update Sheet with Text Post URL”
      - Operation: **Update**
      - Document: your content calendar sheet
      - Sheet/tab: content calendar tab
      - Matching column: `row_number`
      - Set:
        - `row_number` = `{{ $('Check if Platform is Facebook').item.json.row_number }}`
        - `Approval Status` = `Published`
        - `Post URL` = `{{ "https://www.facebook.com/" + $json.id.split('_')[0] + "/posts/" + $json.id.split('_')[1] }}`

10) **Photo branch (Media URL not empty)**
   1. Add node: **Google Drive** named “Download Image from Google Drive”
      - Credentials: Google Drive OAuth2
      - Operation: **Download**
      - File: map from `{{$json['Media URL']}}` (ensure it is a valid Drive ID/link)
   2. Add node: **HTTP Request** named “Schedule Facebook Photo Post”
      - Method: POST
      - URL: `https://graph.facebook.com/v24.0/{{ $node['Load Facebook Credentials from Sheet'].json['Facebook Page ID'] }}/photos`
      - Content-Type: **multipart/form-data**
      - Body params:
        - `access_token` (as above)
        - `caption` = `{{$json.Caption}}`
        - `source` = Binary field (e.g., `data`) from the Drive download node
        - `published` = `false`
        - `scheduled_publish_time` = computed UNIX seconds (workflow adds +15 minutes)
   3. Add node: **Google Sheets** named “Update Sheet with Photo Post URL”
      - Operation: update by `row_number`
      - Set:
        - `row_number` = `{{ $('Check if Platform is Facebook').item.json.row_number }}`
        - `Approval Status` = `Published`
        - `Post URL` = `https://www.facebook.com/photo/?fbid={{ $json.id }}`

11) **Merge branches and continue loop**
   1. Add node: **Merge** named “Merge All Post Types”
      - Set **Number of Inputs** = 3
   2. Connect:
      - From “Update Sheet with Text Post URL” → Merge input 1
      - From “Update Sheet with Photo Post URL” → Merge input 2
      - From “Check if Not Story Post” **false** output (Story) → Merge input 3
   3. Connect Merge output → Split In Batches input (to fetch next item)
   4. Connect Split In Batches **batch output** (the output that emits items) → Switch “Check if Platform is Facebook”.

12) **Credentials checklist**
   - Google Sheets OAuth2: must have access to the spreadsheet.
   - Google Drive OAuth2: must have permission to download the media files.
   - Facebook: you are using **raw HTTP** with a Page Access Token.
     - Ensure token has the required Page permissions and the Page is correctly selected.
     - Plan for token rotation/expiry (long-lived tokens where applicable).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow sticky note includes: purpose, how it works (8 steps), and setup steps (sheet columns + .env tab + credentials + update sheet URLs). | Internal workflow notes (Sticky Note). |
| The workflow “runs daily at 9:35 AM, 10:35 AM, 11:35 AM, 12:35 PM” via cron `35 9-12 * * *`. Confirm timezone alignment to match your business hours. | Scheduling reliability / timezone. |
| Posts are only processed when `Approval Status = "Good"` and then set to `"Published"` after scheduling. | Approval + publishing tracking in Sheets. |
| Story posts are explicitly skipped (not supported by this workflow). | Platform capability constraint. |
| Graph API version is pinned to `v24.0`. | Future compatibility / maintenance. |