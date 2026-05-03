Publish videos to 9 social platforms from Google Sheets with Blotato

https://n8nworkflows.xyz/workflows/publish-videos-to-9-social-platforms-from-google-sheets-with-blotato-12836


# Publish videos to 9 social platforms from Google Sheets with Blotato

## 1. Workflow Overview

**Workflow name:** Auto Publish Videos to 9 Social Media Platforms  
**Purpose:** Automatically publish a video (plus title/caption) from a Google Sheets “content queue” to multiple social platforms via **Blotato**, then update the sheet to reflect **Posted** or **Errored** status.

**Primary use case:** A creator/marketer maintains a Google Sheet of video posts; rows marked *Ready To Post* are picked up on a schedule and posted in parallel to selected platforms (Instagram, TikTok, Facebook, LinkedIn, Pinterest, Threads, X, YouTube, Bluesky). After each platform attempt, the workflow updates the same sheet.

### Logical blocks
**1.1 Scheduling & Queue Intake**
- Triggers every minute.
- Reads the first Google Sheets row whose `Status` equals `Ready To Post`.

**1.2 Parallel Publishing via Blotato (9 platforms)**
- Sends the same content to multiple Blotato “post create” actions (one per platform).
- Most nodes build `postContentText` from `Title` + `Caption` and attach a video via `Media URL`.

**1.3 Post-run Status Update (Success / Error)**
- Each platform node routes **success output** to “Mark as Published”.
- Each platform node routes **error output** to “Log Error” (nodes use `onError: continueErrorOutput` on some platforms).

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Queue Intake

**Overview:** Starts the workflow on a time interval and pulls one pending row from Google Sheets to publish.

**Nodes involved:**
- Schedule Content Publishing
- Read Pending Posts (Google Sheets)

#### Node: Schedule Content Publishing
- **Type / role:** `Schedule Trigger` — entry point.
- **Configuration (interpreted):** Runs every **1 minute**.
- **Inputs/Outputs:** No input. Outputs to **Read Pending Posts (Google Sheets)**.
- **Version notes:** typeVersion **1.2**.
- **Failure/edge cases:**
  - If the n8n instance is paused/down, scheduled runs are missed (depends on n8n execution mode).
  - High frequency (every minute) can cause overlapping executions if publishing is slow and concurrency is enabled.

#### Node: Read Pending Posts (Google Sheets)
- **Type / role:** `Google Sheets` — reads content queue row.
- **Configuration (interpreted):**
  - Reads from spreadsheet **“Plan Blotato Posting”** and sheet **Sheet1 (gid=0)**.
  - Filter: `Status` **equals** `Ready To Post`.
  - Option: **Return First Match** enabled (so only one row is taken per run).
- **Key fields expected in the row (based on expressions used later):**
  - `Title`
  - `Caption`
  - ` Media URL` (note the **leading space** in the column name in several nodes)
  - `row_number` (used by the error logger)
  - `Status`
- **Outputs/Connections:** Fan-out to 9 Blotato nodes in parallel: Instagram, TikTok, Facebook, LinkedIn, Pinterest, Threads, X, YouTube, Bluesky.
- **Version notes:** typeVersion **4.6**.
- **Failure/edge cases:**
  - If no row matches, the node returns no items; downstream nodes won’t run.
  - Column name mismatch is a major risk here: the workflow references both `Media URL` and ` Media URL` (with a leading space) across nodes.
  - If `Return First Match` is used without a deterministic sort in Sheets, the “first” matched row may not be the intended one.

---

### 2.2 Parallel Publishing via Blotato (9 platforms)

**Overview:** Publishes the selected row’s video and text to each platform using Blotato. Each node runs independently; failures shouldn’t stop the entire workflow.

**Nodes involved:**
- Instagram
- TikTok
- Facebook
- LinkedIn
- Pinterest
- Threads
- X
- YouTube
- Bluesky

> **Important:** Several nodes reference the Google Sheets row using `$('Read Pending Posts (Google Sheets)').item.json...` rather than `$json...`. This tightly couples them to that upstream node and assumes exactly one item is in context. Since you “Return First Match”, that assumption generally holds.

#### Node: Instagram
- **Type / role:** `Blotato` node (`@blotato/n8n-nodes-blotato.blotato`) — publish to Instagram.
- **Configuration (interpreted):**
  - Uses Blotato account **giangxai.aff** (accountId 25299).
  - Text: `Title` + blank line + `Caption` from Google Sheets.
  - Media URL: `$('Read Pending Posts...').item.json[' Media URL']` (leading space in key).
  - **onError:** `continueErrorOutput` (sends failures to the error output channel).
- **Inputs/Outputs:**
  - Input: from **Read Pending Posts (Google Sheets)**.
  - Output 0 (success): **Update Sheet – Mark as Published**
  - Output 1 (error): **Update Sheet – Log Error**
- **Version notes:** typeVersion **2**.
- **Failure/edge cases:**
  - Invalid/expired Blotato credentials.
  - Instagram media constraints (format, size, aspect ratio) can cause API rejection.
  - If the sheet column is actually `Media URL` (no leading space), the expression will resolve to `undefined` and posting likely fails.

#### Node: TikTok
- **Type / role:** Blotato — publish to TikTok.
- **Configuration:**
  - Platform explicitly set to `tiktok`.
  - Account: **kytagiang** (accountId 23867).
  - Text uses `$json.Title` and `$json.Caption`.
  - Media URL uses `$json[' Media URL']` (leading space).
- **Connections:** Success → Mark as Published; Error → Log Error.
- **Version notes:** typeVersion **2**.
- **Failure/edge cases:**
  - TikTok upload limitations and regional/account restrictions.
  - If `$json` does not contain the expected keys (due to upstream change), post fails.

#### Node: Facebook
- **Type / role:** Blotato — publish to Facebook Page.
- **Configuration:**
  - Platform: `facebook`
  - Account: **Giang VT** (accountId 15884)
  - Facebook Page selected: id **688227101036478**
  - Text: Title + Caption (from Google Sheets node reference)
  - Media URL: `$json[' Media URL']`
  - **onError:** `continueErrorOutput`
- **Connections:** Success → Mark as Published; Error → Log Error.
- **Version notes:** typeVersion **2**.
- **Failure/edge cases:**
  - Page permissions/token expiration.
  - Facebook may require specific video encoding; API may reject unsupported URLs or blocked domains.

#### Node: LinkedIn
- **Type / role:** Blotato — publish to LinkedIn.
- **Configuration:**
  - Platform: `linkedin`
  - Account: **Giang Vuong Thi** (accountId 10190)
  - Text: Title + Caption (from Google Sheets node reference)
  - Media URL: `$json[' Media URL']`
- **Connections:** This node is *not explicitly connected* to the update nodes in the provided connections map (only the 4 nodes with onError shown are connected plus TikTok/YouTube/Facebook/Instagram). However, it is connected from the Google Sheets reader.  
  - **Result:** LinkedIn will publish, but **its success/error will not update the sheet** unless connections are added.
- **Failure/edge cases:** OAuth expiration, LinkedIn upload limits, unsupported media URLs.

#### Node: Pinterest
- **Type / role:** Blotato — publish to Pinterest.
- **Configuration:**
  - Platform: `pinterest`
  - Account: **the_lawofattraction_tip111** (accountId 3601)
  - Board ID: `1116681738796146206`
  - Title option: from `Title`
  - Text: uses only `Caption`
  - Media URL: `$json[' Media URL']`
- **Connections:** Same issue as LinkedIn: no downstream update connections shown.
- **Failure/edge cases:** Board permission issues, Pinterest media/metadata requirements.

#### Node: Threads
- **Type / role:** Blotato — publish to Threads.
- **Configuration:**
  - Platform: `threads`
  - Account: **giangxai.aff** (accountId 4049)
  - Text: Title + Caption
  - Media URL: `$json[' Media URL']`
- **Connections:** Not shown connected to update nodes.
- **Failure/edge cases:** Account restrictions, media constraints.

#### Node: X
- **Type / role:** Blotato — publish to X (Twitter).
- **Configuration:**
  - Platform: `twitter`
  - Account: **GiangKyta** (accountId 10665)
  - Text: Title + Caption
  - Media URL: `$json[' Media URL']`
- **Connections:** Not shown connected to update nodes.
- **Failure/edge cases:** X API limits, media upload constraints, rate limiting.

#### Node: YouTube
- **Type / role:** Blotato — publish to YouTube.
- **Configuration:**
  - Platform: `youtube`
  - Account: **xAI Giang ( GiangVT)** (accountId 22282)
  - Video title option: from `Title`
  - Description/text: `Caption`
  - Notify subscribers: **false**
  - Media URL: `$json[' Media URL']`
  - **onError:** `continueErrorOutput`
- **Connections:** Success → Mark as Published; Error → Log Error.
- **Failure/edge cases:** YouTube upload quotas, copyright checks, invalid source URL, processing delays.

#### Node: Bluesky
- **Type / role:** Blotato — publish to Bluesky.
- **Configuration:**
  - Platform: `bluesky`
  - **AccountId is empty** (not configured).
  - Text: Title + Caption (from Google Sheets node reference)
  - Media URL: `{{ $json.url }}` (does not match the Google Sheets “Media URL” fields used elsewhere).
- **Connections:** Not shown connected to update nodes.
- **Failure/edge cases:**
  - Will likely fail immediately due to missing account selection/credentials.
  - Media URL expression likely wrong (`$json.url` vs `$json[' Media URL']`), causing undefined media.

---

### 2.3 Post-run Status Update (Success / Error)

**Overview:** Writes back to Google Sheets to mark the row as Posted or Errored. In the current wiring, only Instagram/TikTok/Facebook/YouTube feed these nodes.

**Nodes involved:**
- Update Sheet – Mark as Published
- Update Sheet – Log Error

#### Node: Update Sheet – Mark as Published
- **Type / role:** Google Sheets `update` — marks a row as posted by matching on Title.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Match column(s): `Title`
  - Values set:
    - `Title` = Title from the read row
    - `Status` = `Posted`
- **Inputs/Outputs:** Receives success output from some platform nodes; no outgoing connections.
- **Version notes:** typeVersion **4.6**.
- **Failure/edge cases:**
  - Matching by `Title` is risky: duplicates can update the wrong row (or multiple rows, depending on connector behavior).
  - If any platform publishes successfully and triggers this, the row is marked Posted even if other platforms failed or never ran.
  - If the Title contains extra spaces/case differences vs the sheet, it might not match.

#### Node: Update Sheet – Log Error
- **Type / role:** Google Sheets `update` — marks a row as errored by matching on row number.
- **Configuration:**
  - Operation: **Update**
  - Match column(s): `row_number`
  - Values set:
    - `Status` = `Errored`
    - `row_number` = `$('Read Pending Posts (Google Sheets)').item.json.row_number`
- **Inputs/Outputs:** Receives error output from some platform nodes; no outgoing connections.
- **Version notes:** typeVersion **4.6**.
- **Failure/edge cases:**
  - This node relies on `row_number` being present in the read output. If the Google Sheets node doesn’t return `row_number` (or it’s renamed/disabled), error logging will fail.
  - Mixed success: one platform error will mark the row Errored even if others succeeded.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Content Publishing | Schedule Trigger | Time-based entry point | — | Read Pending Posts (Google Sheets) | ## Schedule & Load Content from Google Sheets<br><br>Trigger workflow on schedule and fetch pending video content |
| Read Pending Posts (Google Sheets) | Google Sheets | Fetch first row with `Status = Ready To Post` | Schedule Content Publishing | Instagram, TikTok, Facebook, LinkedIn, Pinterest, Threads, X, YouTube, Bluesky | ## Schedule & Load Content from Google Sheets<br><br>Trigger workflow on schedule and fetch pending video content |
| Instagram | Blotato | Publish video to Instagram | Read Pending Posts (Google Sheets) | Update Sheet – Mark as Published; Update Sheet – Log Error | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| TikTok | Blotato | Publish video to TikTok | Read Pending Posts (Google Sheets) | Update Sheet – Mark as Published; Update Sheet – Log Error | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| YouTube | Blotato | Publish video to YouTube | Read Pending Posts (Google Sheets) | Update Sheet – Mark as Published; Update Sheet – Log Error | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| Facebook | Blotato | Publish video to Facebook Page | Read Pending Posts (Google Sheets) | Update Sheet – Mark as Published; Update Sheet – Log Error | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| X | Blotato | Publish video to X/Twitter | Read Pending Posts (Google Sheets) | (none wired) | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| Pinterest | Blotato | Publish video to Pinterest board | Read Pending Posts (Google Sheets) | (none wired) | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| Threads | Blotato | Publish video to Threads | Read Pending Posts (Google Sheets) | (none wired) | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| LinkedIn | Blotato | Publish video to LinkedIn | Read Pending Posts (Google Sheets) | (none wired) | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| Bluesky | Blotato | Publish video to Bluesky | Read Pending Posts (Google Sheets) | (none wired) | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |
| Update Sheet – Mark as Published | Google Sheets | Set Status=Posted (match by Title) | Instagram, TikTok, Facebook, YouTube (success paths) | — | ## Mark Post as Published (Success) |
| Update Sheet – Log Error | Google Sheets | Set Status=Errored (match by row_number) | Instagram, TikTok, Facebook, YouTube (error paths) | — | ## Log Publishing Errors |
| Sticky Note | Sticky Note | Documentation/overview | — | — | ## Overview<br>**Author:** [GiangxAI](https://www.youtube.com/@giangxai.official)<br><br>This n8n template helps you automatically publish videos to multiple social media platforms using Google Sheets as the content control center.<br>The workflow runs on a schedule, reads pending video posts from Google Sheets, and publishes them in parallel to 9 social media platforms. After publishing, it automatically updates the post status and logs any errors, so you always know what has been published and what needs attention.<br>This template is designed to save time, reduce manual uploads, and ensure consistent video distribution across all platforms.<br><br><br>## Quick Setup<br><br>1. **Prepare Google Sheets**<br>   - Create a sheet with video URLs, captions, and a `status` column (e.g. `pending`).<br>   - Only rows marked as `pending` will be published.<br><br>2. **Connect Google Sheets**<br>   - Authenticate your Google Sheets account in n8n.<br>   - Select the correct spreadsheet in the read and update nodes.<br><br>3. **Set Up Platform Credentials**<br>   - Add API credentials or OAuth access for each social platform you want to use.<br>   - Disable any platform nodes you don’t need.<br><br>4. **Configure the Schedule**<br>   - Open the **Schedule Content Publishing** trigger.<br>   - Set how often the workflow should run (daily, hourly, or custom).<br><br>5. **Run & Monitor**<br>   - Activate the workflow.<br>   - Published posts are marked as successful, and errors are logged back to Google Sheets. |
| Sticky Note1 | Sticky Note | Section header | — | — | ## Schedule & Load Content from Google Sheets<br><br>Trigger workflow on schedule and fetch pending video content |
| Sticky Note2 | Sticky Note | Section header | — | — | ## Mark Post as Published (Success) |
| Sticky Note3 | Sticky Note | Section header | — | — | ## Log Publishing Errors |
| Sticky Note4 | Sticky Note | Section header | — | — | ## Auto Publish Videos to 9 Social Media Platforms<br>Post videos in parallel based on selected platforms |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: *Auto Publish Videos to 9 Social Media Platforms* (or your preferred name).

2) **Add trigger: “Schedule Content Publishing”**
   - Node type: **Schedule Trigger**
   - Interval: every **1 minute** (or adjust as desired).

3) **Add Google Sheets read node: “Read Pending Posts (Google Sheets)”**
   - Node type: **Google Sheets**
   - Credentials: connect a Google account with access to the spreadsheet.
   - Operation: **Read / Get Many (with filters)** (exact UI varies by n8n version)
   - Spreadsheet: select your sheet (e.g., “Plan Blotato Posting”)
   - Tab: select `Sheet1`
   - Filter:
     - Lookup column: `Status`
     - Lookup value: `Ready To Post`
   - Option: enable **Return First Match**.
   - Connect: **Schedule Content Publishing → Read Pending Posts (Google Sheets)**

4) **Prepare your Google Sheet columns**
   - Ensure you have consistent column names used by expressions:
     - `Title`
     - `Caption`
     - `Media URL` **or** ` Media URL` (choose one; avoid leading spaces)
     - `Status`
   - Ensure your read node returns `row_number` if you plan to match by row number for error updates.

5) **Add 9 Blotato nodes (one per platform)**
   - Node type: `@blotato/n8n-nodes-blotato.blotato`
   - Credentials: configure/connect your Blotato credential in n8n (as required by the Blotato community node).
   - For each node, set:
     - **Platform** (if required): instagram, tiktok, facebook, linkedin, pinterest, threads, twitter, youtube, bluesky
     - **Account**: select the correct Blotato account for that platform
     - **Text** (typical):
       - `{{$json.Title}}` + newline + `{{$json.Caption}}`
       - or reference the read node explicitly: `{{ $('Read Pending Posts (Google Sheets)').item.json.Title }}`
     - **Media URL(s)**: map to your sheet column, e.g. `{{$json["Media URL"]}}`
     - Platform-specific options:
       - Facebook: select **Page**
       - Pinterest: select **Board**, map **Title** option
       - YouTube: map **Title** option; set notify subscribers as desired
   - Connect: **Read Pending Posts (Google Sheets) → each Blotato node** (parallel fan-out)

6) **Add Google Sheets update node: “Update Sheet – Mark as Published”**
   - Node type: **Google Sheets**
   - Operation: **Update**
   - Spreadsheet/tab: same as read node
   - Matching strategy (recommended):
     - Prefer matching by `row_number` (most reliable) if available.
   - Set fields:
     - `Status` = `Posted`
     - (Optionally keep `Title` unchanged)
   - Connect each platform node’s **success** output to this node.
     - If you want “Posted” only when *all* platforms succeed, you must add a merge/aggregation step (not present in current workflow).

7) **Add Google Sheets update node: “Update Sheet – Log Error”**
   - Node type: **Google Sheets**
   - Operation: **Update**
   - Match by: `row_number`
   - Set fields:
     - `Status` = `Errored`
   - Connect each platform node’s **error** output to this node.
   - On each Blotato node, set error handling to allow routing (where supported):
     - Enable/choose **Continue on Fail** or `onError: continueErrorOutput` behavior so the workflow can proceed and log errors.

8) **(Optional but strongly recommended) Fix wiring gaps**
   - In the provided workflow, LinkedIn/Pinterest/Threads/X/Bluesky do not route to the update nodes.
   - Connect both success/error outputs for those nodes too, if you want consistent status logging.

9) **Activate the workflow**
   - Test with one row marked `Ready To Post`.
   - Verify status changes to `Posted` or `Errored`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Author:** GiangxAI | https://www.youtube.com/@giangxai.official |
| Uses Google Sheets as a “content control center” and publishes in parallel to 9 platforms via Blotato; updates sheet status and logs errors. | From the workflow overview sticky note |
| Quick setup guidance: prepare sheet with URLs/captions/status; connect Google Sheets; configure platform credentials; configure schedule; activate and monitor. | From the workflow overview sticky note |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.