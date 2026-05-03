Schedule approved LinkedIn page posts from Google Sheets with precise timing

https://n8nworkflows.xyz/workflows/schedule-approved-linkedin-page-posts-from-google-sheets-with-precise-timing-13304


# Schedule approved LinkedIn page posts from Google Sheets with precise timing

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Schedule approved LinkedIn page posts from Google Sheets with precise timing

**Purpose:**  
Automatically publish approved LinkedIn *organization page* posts that are tracked in Google Sheets. The workflow runs on a schedule, selects posts scheduled **for today**, waits until each post’s exact scheduled time (with timezone conversion), publishes to LinkedIn (image or article), then writes the resulting LinkedIn post URL back to the sheet and updates status.

**Target use cases**
- Social media teams maintaining an editorial calendar in Google Sheets
- Scheduled LinkedIn organization posting with image assets stored in Google Drive
- Simple approval/status lifecycle tracking (“Scheduled”, “Published”)

### Logical blocks
1.1 **Trigger & Data Loading**: schedule trigger → Google Sheets reads  
1.2 **Today Filtering & Single-item Processing**: filter to “today” → take first item only  
1.3 **Routing by Post Type**: Switch routes to Image (Creative Post) vs Article  
1.4 **Creative Post Publishing (Image)**: mark scheduled → wait → download image → post → update sheet  
1.5 **Article Publishing (Link)**: mark scheduled → wait → post as article → update sheet  

---

## 2. Block-by-Block Analysis

### 1.1 Trigger & Data Loading

**Overview:**  
Starts the workflow on a cron schedule and loads data from Google Sheets. Although one node is named “Load LinkedIn Organization Credentials”, in this JSON it only reads a spreadsheet; the LinkedIn org ID is hard-coded in LinkedIn nodes.

**Nodes involved**
- Run Every Hour
- Load LinkedIn Organization Credentials
- Fetch Approved LinkedIn Posts

#### Node: Run Every Hour
- **Type / role:** Schedule Trigger — kicks off the workflow on a cron rule.
- **Configuration:** Cron expression `45 9-12 * * *` (runs at minute 45 during hours 9,10,11,12 daily).
- **Connections:** Outputs to **Load LinkedIn Organization Credentials**.
- **Version notes:** typeVersion 1.2.
- **Edge cases / failures:**
  - Timezone depends on n8n instance settings (server timezone). Cron fires based on instance timezone unless configured otherwise in n8n settings.

#### Node: Load LinkedIn Organization Credentials
- **Type / role:** Google Sheets — reads from a spreadsheet (despite node name).
- **Configuration choices:**
  - Uses `documentId` set via expression-like string: `=YOUR_SHEET_URL` (placeholder).
  - `sheetName` is set to “list” mode but has no value selected (blank).
- **Credentials:** Google Sheets OAuth2 credential `TESTING_SHEET`.
- **Connections:** Outputs to **Fetch Approved LinkedIn Posts**.
- **Version notes:** typeVersion 4.7.
- **Edge cases / failures:**
  - If `documentId` remains `=YOUR_SHEET_URL`, node will fail (invalid doc ID).
  - Blank `sheetName` will cause the node to fail or read nothing depending on n8n behavior; it must be set.
  - OAuth scope/permission issues if spreadsheet not shared with the Google account.

#### Node: Fetch Approved LinkedIn Posts
- **Type / role:** Google Sheets — intended to fetch approved posts.
- **Configuration choices:**
  - Same placeholder `documentId` `=YOUR_SHEET_URL`
  - `sheetName` blank (must be selected)
  - No explicit operation shown in JSON; in n8n this typically defaults to **Read** (but it must be verified in UI).
- **Credentials:** Google Sheets OAuth2 `TESTING_SHEET`.
- **Connections:** Outputs to **Filter Posts Scheduled for Today**.
- **Edge cases / failures:**
  - If operation is not configured to read rows, downstream code will receive no items.
  - The sticky note claims filtering by Platform=LinkedIn and Approval Status="Good", but **no such filtering node exists** in the provided JSON. If required, it must be implemented (e.g., in Sheets query, IF node, or Code node).

---

### 1.2 Today Filtering & Single-item Processing

**Overview:**  
Filters incoming sheet rows to only those whose “Scheduled On” date equals today (date-only comparison), then selects only the first item to process per run.

**Nodes involved**
- Filter Posts Scheduled for Today
- Process First Post Only

#### Node: Filter Posts Scheduled for Today
- **Type / role:** Code — filters items where scheduled date matches today.
- **Key logic:**
  - Computes `todayDateString = new Date().toISOString().split('T')[0]` (UTC date).
  - Looks for schedule field in any of:
    - `$json['Scheduled On']`
    - `$json['scheduled_on']`
    - `$json.scheduledOn`
  - Extracts `YYYY-MM-DD` from strings via regex `(\d{4}-\d{2}-\d{2})`.
  - Keeps item if extracted date equals `todayDateString`.
- **Connections:** Outputs to **Process First Post Only**.
- **Version notes:** typeVersion 2.
- **Edge cases / failures:**
  - **Timezone mismatch risk:** `toISOString()` uses UTC. If your “today” is based on local time (e.g., America/New_York), this can be off by one day around midnight UTC. Consider using Luxon with a defined zone to compute “today”.
  - If “Scheduled On” is in a non-matching format (no `YYYY-MM-DD`), item will be dropped.
  - If the sheet provides a true date object vs string, `instanceof Date` may not be true (Sheets nodes often output strings).

#### Node: Process First Post Only
- **Type / role:** Code — limits processing to one post per run.
- **Key logic:** `return [items[0]];`
- **Connections:** Outputs to **Route by Post Type**.
- **Version notes:** typeVersion 2.
- **Edge cases / failures:**
  - If there are **no items**, `items[0]` is `undefined`; returning `[undefined]` can cause downstream expressions to fail. Safer approach: return `[]` when `items.length === 0`.

---

### 1.3 Routing by Post Type

**Overview:**  
Routes the first pending post into either the image-based “Creative Post” branch or the “Article” link branch based on the `Post Type` column.

**Nodes involved**
- Route by Post Type (Switch)

#### Node: Route by Post Type
- **Type / role:** Switch — conditional branching.
- **Rules:**
  1) If `{{$json['Post Type']}}` equals `Creative Post` → output 0 (image branch)  
  2) If `{{$json['Post Type']}}` equals `Article` → output 1 (article branch)
- **Connections:**
  - Output 0 → **Mark as Scheduled in Sheet**
  - Output 1 → **Mark Article as Scheduled**
- **Version notes:** typeVersion 3.3.
- **Edge cases / failures:**
  - Exact string match; any trailing spaces or different capitalization will not route.
  - If neither matches, item is dropped (no default route configured).

---

### 1.4 Creative Post Publishing (Image)

**Overview:**  
Updates the row status to “Scheduled”, waits until the exact scheduled time, downloads the image from Google Drive, publishes the post as an organization image post, then writes back the LinkedIn post URL and sets status to “Published”.

**Nodes involved**
- Mark as Scheduled in Sheet
- Wait Until Scheduled Time
- Prepare Post Data
- Download Image from Google Drive
- Publish Creative Post to LinkedIn
- Save Post URL & Mark Published

#### Node: Mark as Scheduled in Sheet
- **Type / role:** Google Sheets — updates the row for the selected post.
- **Configuration choices:**
  - Operation: **Update**
  - Matching column: `row_number`
  - Sets:
    - `row_number = {{$('Route by Post Type').item.json.row_number}}`
    - `Approval Status = "Scheduled"`
  - Sheet tab: “Post URL” (gid 98565607)
  - Document: `=YOUR_SHEET_URL` placeholder
- **Connections:** Outputs to **Wait Until Scheduled Time**.
- **Version notes:** typeVersion 4.7.
- **Edge cases / failures:**
  - Requires `row_number` to exist from the read operation.
  - If `row_number` is missing, update may target nothing.
  - Sheet name/tab must match where rows were read from; otherwise you update a different sheet than the one you queried.

#### Node: Wait Until Scheduled Time
- **Type / role:** Wait — pauses execution until a computed datetime.
- **Configuration choices:**
  - Resume: **specificTime**
  - Datetime expression (Luxon):
    - Parses `Scheduled On` from **Route by Post Type** item using format `'yyyy-MM-dd HH-mm'`
    - Interprets in zone `'America/New_York'`
    - Converts to `'Asia/Kolkata'`
    - Outputs ISO-like `"yyyy-MM-dd'T'HH:mm:ss"`
- **Connections:** Outputs to **Prepare Post Data**.
- **Version notes:** typeVersion 1.1.
- **Edge cases / failures:**
  - **Format sensitivity:** expects `Scheduled On` like `2025-10-30 10-00` (note `HH-mm`, dash between hour and minute). If your sheet uses `HH:mm` or `HH:mm:ss`, parsing fails and Wait errors.
  - If scheduled time is in the past, n8n typically resumes immediately (behavior may vary).
  - Wait nodes require n8n to have persistence configured reliably (or executions may be lost on restarts depending on n8n mode).

#### Node: Prepare Post Data
- **Type / role:** Aggregate — groups/aggregates incoming items.
- **Configuration choices:** “fieldsToAggregate” is effectively empty (`fieldToAggregate: [{}]`).
- **Connections:** Outputs to **Download Image from Google Drive**.
- **Version notes:** typeVersion 1.
- **Practical effect:** As configured, it does not clearly transform data; it may output a single aggregated item but with default behavior. In many builds this node is unnecessary unless you intended to consolidate multiple items/media.
- **Edge cases / failures:** Could produce unexpected structure for downstream nodes; verify output shape in n8n execution data.

#### Node: Download Image from Google Drive
- **Type / role:** Google Drive — downloads media for the post.
- **Configuration choices:**
  - Operation: **download**
  - `fileId` set from `{{$node['Route by Post Type'].json['Media URL']}}`
    - Note: UI label is `fileId` but value is taken from a “Media URL” column.
- **Credentials:** Google Drive OAuth2 `Sourav_Singh`.
- **Connections:** Outputs to **Publish Creative Post to LinkedIn**.
- **Version notes:** typeVersion 3.
- **Edge cases / failures:**
  - If “Media URL” is a full Google Drive URL, it may not be a valid fileId unless n8n can parse it. Typically you must supply the actual file ID or use “Share link” parsing.
  - Permissions: the Drive credential must have access to the file.

#### Node: Publish Creative Post to LinkedIn
- **Type / role:** LinkedIn — publishes an organization post with image category.
- **Configuration choices:**
  - `postAs`: organization
  - `organization`: `56420402` (hard-coded org ID)
  - `text`: `{{$node['Route by Post Type'].json.Caption}}`
  - `shareMediaCategory`: `IMAGE`
  - **Important:** This node is connected after the Drive download, but the configuration shown does not explicitly map the downloaded binary. Depending on LinkedIn node implementation, it may require binary property mapping (often `binaryPropertyName`). This should be verified in the n8n UI.
- **Credentials:** LinkedIn OAuth2 `TESTING_LINKEDIN`.
- **Connections:** Outputs to **Save Post URL & Mark Published**.
- **Version notes:** typeVersion 1.
- **Edge cases / failures:**
  - LinkedIn API permissions: must include organization posting permissions (often “Community Management” scopes).
  - If binary isn’t passed/mapped correctly, LinkedIn publishing with image may fail.
  - Rate limits and transient API errors.

#### Node: Save Post URL & Mark Published
- **Type / role:** Google Sheets — writes result URL and final status.
- **Configuration choices:**
  - Operation: **Update**
  - Matching column: `row_number`
  - Sets:
    - `Post URL = "https://www.linkedin.com/feed/update/" + $json.urn`
    - `Approval Status = "Published"`
    - `row_number = {{$node['Route by Post Type'].json.row_number}}`
  - Sheet tab: “Post URL”
  - Document: `=YOUR_SHEET_URL`
- **Connections:** Terminal node in this branch.
- **Version notes:** typeVersion 4.7.
- **Edge cases / failures:**
  - Assumes LinkedIn node returns `urn` in the response root (`$json.urn`). If the LinkedIn node returns a different structure, URL generation fails.
  - The constructed URL format may not always match LinkedIn’s canonical URL patterns for all post types.

---

### 1.5 Article Post Publishing (Link)

**Overview:**  
Mirrors the image branch but publishes as an article/link share using the “Media URL” column as the target URL.

**Nodes involved**
- Mark Article as Scheduled
- Wait Until Article Scheduled Time
- Prepare Article Data
- Publish Article Link to LinkedIn
- Save Article URL & Mark Published

#### Node: Mark Article as Scheduled
- **Type / role:** Google Sheets — update approval status.
- **Configuration choices:**
  - Operation: **Update**, match by `row_number`
  - Sets `Approval Status = "Scheduled"`
  - Uses same “Post URL” sheet tab.
- **Connections:** Outputs to **Wait Until Article Scheduled Time**.
- **Version notes:** typeVersion 4.7.
- **Edge cases / failures:** Same as “Mark as Scheduled in Sheet”.

#### Node: Wait Until Article Scheduled Time
- **Type / role:** Wait — pauses until scheduled time.
- **Configuration:** Same Luxon expression and formatting as image branch.
- **Connections:** Outputs to **Prepare Article Data**.
- **Version notes:** typeVersion 1.1.
- **Edge cases / failures:** Same parsing/timezone risks as the other Wait node.

#### Node: Prepare Article Data
- **Type / role:** Aggregate — currently effectively empty aggregation.
- **Connections:** Outputs to **Publish Article Link to LinkedIn**.
- **Version notes:** typeVersion 1.
- **Edge cases / failures:** Same “possibly unnecessary / output shape” concerns as “Prepare Post Data”.

#### Node: Publish Article Link to LinkedIn
- **Type / role:** LinkedIn — publishes an organization article/link share.
- **Configuration choices:**
  - `postAs`: organization
  - `organization`: `56420402`
  - `text`: Caption from sheet
  - `shareMediaCategory`: `ARTICLE`
  - `additionalFields.originalUrl = {{$node['Route by Post Type'].json['Media URL']}}`
- **Credentials:** LinkedIn OAuth2 `TESTING_LINKEDIN`.
- **Connections:** Outputs to **Save Article URL & Mark Published**.
- **Version notes:** typeVersion 1.
- **Edge cases / failures:**
  - “Media URL” must be a publicly accessible URL suitable for LinkedIn scraping; private links may fail or generate poor previews.
  - Same LinkedIn permission/rate-limit risks.

#### Node: Save Article URL & Mark Published
- **Type / role:** Google Sheets — saves the LinkedIn post URL and marks Published.
- **Configuration:** Same pattern as the creative post saver:
  - `Post URL = "https://www.linkedin.com/feed/update/" + $json.urn`
  - `Approval Status = "Published"`
  - Match by `row_number`
- **Connections:** Terminal node.
- **Version notes:** typeVersion 4.7.
- **Edge cases / failures:** Same as “Save Post URL & Mark Published”.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Every Hour | Schedule Trigger | Cron-based entry point | — | Load LinkedIn Organization Credentials | # Schedule and auto-publish LinkedIn posts from Google Sheets\nAutomate LinkedIn organization page posting with precise time scheduling, Google Drive media management, and smart status tracking. Runs hourly during business hours, processes approved posts scheduled for today, waits until exact time, publishes to LinkedIn, and updates tracking sheet automatically. |
| Load LinkedIn Organization Credentials | Google Sheets | Load config / credentials sheet (named as such) | Run Every Hour | Fetch Approved LinkedIn Posts | ## Trigger & Data Loading\nRuns 4 times daily at :45 minutes (9-12 AM), loads LinkedIn credentials, fetches approved posts filtered by platform and approval status, then filters for today's scheduled posts only |
| Fetch Approved LinkedIn Posts | Google Sheets | Read candidate posts from sheet | Load LinkedIn Organization Credentials | Filter Posts Scheduled for Today | ## Trigger & Data Loading\nRuns 4 times daily at :45 minutes (9-12 AM), loads LinkedIn credentials, fetches approved posts filtered by platform and approval status, then filters for today's scheduled posts only |
| Filter Posts Scheduled for Today | Code | Filter rows to today’s scheduled date | Fetch Approved LinkedIn Posts | Process First Post Only | ## Trigger & Data Loading\nRuns 4 times daily at :45 minutes (9-12 AM), loads LinkedIn credentials, fetches approved posts filtered by platform and approval status, then filters for today's scheduled posts only |
| Process First Post Only | Code | Limit run to a single post | Filter Posts Scheduled for Today | Route by Post Type | ## Post Type Routing\nProcesses first pending post and routes to appropriate publishing branch based on post type (Creative Post with image or Article with link) |
| Route by Post Type | Switch | Branch by “Post Type” | Process First Post Only | Mark as Scheduled in Sheet; Mark Article as Scheduled | ## Post Type Routing\nProcesses first pending post and routes to appropriate publishing branch based on post type (Creative Post with image or Article with link) |
| Mark as Scheduled in Sheet | Google Sheets | Update status to Scheduled (image branch) | Route by Post Type | Wait Until Scheduled Time | ## Creative Post Publishing (Image)\nMarks as scheduled, waits until exact time, downloads image from Google Drive, publishes to LinkedIn organization page, saves live URL and updates status to Published |
| Wait Until Scheduled Time | Wait | Pause until scheduled time (image branch) | Mark as Scheduled in Sheet | Prepare Post Data | ## Creative Post Publishing (Image)\nMarks as scheduled, waits until exact time, downloads image from Google Drive, publishes to LinkedIn organization page, saves live URL and updates status to Published |
| Prepare Post Data | Aggregate | (Intended) normalize/aggregate post payload | Wait Until Scheduled Time | Download Image from Google Drive | ## Creative Post Publishing (Image)\nMarks as scheduled, waits until exact time, downloads image from Google Drive, publishes to LinkedIn organization page, saves live URL and updates status to Published |
| Download Image from Google Drive | Google Drive | Download image binary for LinkedIn | Prepare Post Data | Publish Creative Post to LinkedIn | ## Creative Post Publishing (Image)\nMarks as scheduled, waits until exact time, downloads image from Google Drive, publishes to LinkedIn organization page, saves live URL and updates status to Published |
| Publish Creative Post to LinkedIn | LinkedIn | Publish organization IMAGE post | Download Image from Google Drive | Save Post URL & Mark Published | ## Creative Post Publishing (Image)\nMarks as scheduled, waits until exact time, downloads image from Google Drive, publishes to LinkedIn organization page, saves live URL and updates status to Published |
| Save Post URL & Mark Published | Google Sheets | Save LinkedIn URL + set Published (image branch) | Publish Creative Post to LinkedIn | — | ## Creative Post Publishing (Image)\nMarks as scheduled, waits until exact time, downloads image from Google Drive, publishes to LinkedIn organization page, saves live URL and updates status to Published |
| Mark Article as Scheduled | Google Sheets | Update status to Scheduled (article branch) | Route by Post Type | Wait Until Article Scheduled Time | ## Article Post Publishing (Link)\nMarks article as scheduled, waits until exact time, publishes article link to LinkedIn organization page, saves post URL and marks as Published |
| Wait Until Article Scheduled Time | Wait | Pause until scheduled time (article branch) | Mark Article as Scheduled | Prepare Article Data | ## Article Post Publishing (Link)\nMarks article as scheduled, waits until exact time, publishes article link to LinkedIn organization page, saves post URL and marks as Published |
| Prepare Article Data | Aggregate | (Intended) normalize/aggregate article payload | Wait Until Article Scheduled Time | Publish Article Link to LinkedIn | ## Article Post Publishing (Link)\nMarks article as scheduled, waits until exact time, publishes article link to LinkedIn organization page, saves post URL and marks as Published |
| Publish Article Link to LinkedIn | LinkedIn | Publish organization ARTICLE post | Prepare Article Data | Save Article URL & Mark Published | ## Article Post Publishing (Link)\nMarks article as scheduled, waits until exact time, publishes article link to LinkedIn organization page, saves post URL and marks as Published |
| Save Article URL & Mark Published | Google Sheets | Save LinkedIn URL + set Published (article branch) | Publish Article Link to LinkedIn | — | ## Article Post Publishing (Link)\nMarks article as scheduled, waits until exact time, publishes article link to LinkedIn organization page, saves post URL and marks as Published |
| Sticky Note | Sticky Note | Global description & setup notes | — | — | # Schedule and auto-publish LinkedIn posts from Google Sheets\nAutomate LinkedIn organization page posting with precise time scheduling, Google Drive media management, and smart status tracking. Runs hourly during business hours, processes approved posts scheduled for today, waits until exact time, publishes to LinkedIn, and updates tracking sheet automatically. |
| Sticky Note1 | Sticky Note | Comment: Trigger & Data Loading | — | — | ## Trigger & Data Loading\nRuns 4 times daily at :45 minutes (9-12 AM), loads LinkedIn credentials, fetches approved posts filtered by platform and approval status, then filters for today's scheduled posts only |
| Sticky Note2 | Sticky Note | Comment: Post Type Routing | — | — | ## Post Type Routing\nProcesses first pending post and routes to appropriate publishing branch based on post type (Creative Post with image or Article with link) |
| Sticky Note3 | Sticky Note | Comment: Creative branch | — | — | ## Creative Post Publishing (Image)\nMarks as scheduled, waits until exact time, downloads image from Google Drive, publishes to LinkedIn organization page, saves live URL and updates status to Published |
| Sticky Note4 | Sticky Note | Comment: Article branch | — | — | ## Article Post Publishing (Link)\nMarks article as scheduled, waits until exact time, publishes article link to LinkedIn organization page, saves post URL and marks as Published |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Schedule Trigger**
   - Add node: **Schedule Trigger**
   - Mode: Cron
   - Cron expression: `45 9-12 * * *`
   - Name: `Run Every Hour`

2) **Add Google Sheets node: “Load LinkedIn Organization Credentials”**
   - Add node: **Google Sheets**
   - Credentials: Google Sheets OAuth2 (connect the Google account that can access the spreadsheet)
   - Set **Document**: choose your spreadsheet (paste URL in document selector)
   - Set **Sheet**: select the tab that contains config/credentials (if you actually use one)
   - Connect: `Run Every Hour` → `Load LinkedIn Organization Credentials`

3) **Add Google Sheets node: “Fetch Approved LinkedIn Posts”**
   - Add node: **Google Sheets**
   - Operation: **Read** (or “Get Many” depending on UI)
   - Document: same spreadsheet
   - Sheet: the tab containing your post queue (the one with Scheduled On, Post Type, etc.)
   - Ensure the output includes `row_number` (enable “Return All”/include row number if available in your n8n version)
   - Connect: `Load LinkedIn Organization Credentials` → `Fetch Approved LinkedIn Posts`
   - (Optional but recommended) Add filtering here or later for:
     - `Platform == "LinkedIn"`
     - `Approval Status == "Good"` (or whatever your “approved” value is)

4) **Add Code node: “Filter Posts Scheduled for Today”**
   - Add node: **Code**
   - Paste the provided JS logic (or rewrite using Luxon in a fixed timezone if needed)
   - Connect: `Fetch Approved LinkedIn Posts` → `Filter Posts Scheduled for Today`

5) **Add Code node: “Process First Post Only”**
   - Add node: **Code**
   - JS logic: return only first item (but ideally guard empty list)
   - Connect: `Filter Posts Scheduled for Today` → `Process First Post Only`

6) **Add Switch node: “Route by Post Type”**
   - Add node: **Switch**
   - Add rule 1: `{{$json["Post Type"]}}` equals `Creative Post`
   - Add rule 2: `{{$json["Post Type"]}}` equals `Article`
   - Connect: `Process First Post Only` → `Route by Post Type`

### Branch A: Creative Post (Image)

7) **Google Sheets: “Mark as Scheduled in Sheet”**
   - Operation: **Update**
   - Document: your spreadsheet
   - Sheet: your post tracking tab
   - Matching column: `row_number`
   - Values to set:
     - `row_number = {{$node["Route by Post Type"].json.row_number}}`
     - `Approval Status = Scheduled`
   - Connect: `Route by Post Type` (Creative output) → `Mark as Scheduled in Sheet`

8) **Wait node: “Wait Until Scheduled Time”**
   - Resume: **At specified time**
   - DateTime expression (Luxon):
     - Parse `Scheduled On` with the exact format you store (the workflow expects `yyyy-MM-dd HH-mm`)
     - Convert timezone as needed
   - Connect: `Mark as Scheduled in Sheet` → `Wait Until Scheduled Time`

9) **Aggregate: “Prepare Post Data”** (optional)
   - Add **Aggregate** only if you need to merge/shape fields; otherwise remove and connect Wait directly to Drive.
   - Connect: `Wait Until Scheduled Time` → `Prepare Post Data`

10) **Google Drive: “Download Image from Google Drive”**
   - Credentials: Google Drive OAuth2
   - Operation: **Download**
   - File ID: map from your sheet column
     - If you store a full Drive URL, ensure you extract the ID (you may need an extra Code node)
   - Connect: `Prepare Post Data` → `Download Image from Google Drive`

11) **LinkedIn: “Publish Creative Post to LinkedIn”**
   - Credentials: LinkedIn OAuth2 with organization posting permissions
   - Post as: **Organization**
   - Organization ID: set to your org (replace `56420402`)
   - Share type/media category: **IMAGE**
   - Text: map from `Caption`
   - Configure binary/image mapping in the node UI if required (depends on your n8n LinkedIn node behavior/version)
   - Connect: `Download Image from Google Drive` → `Publish Creative Post to LinkedIn`

12) **Google Sheets: “Save Post URL & Mark Published”**
   - Operation: **Update**
   - Match: `row_number`
   - Set:
     - `Post URL = {{"https://www.linkedin.com/feed/update/" + $json.urn}}`
     - `Approval Status = Published`
     - `row_number = {{$node["Route by Post Type"].json.row_number}}`
   - Connect: `Publish Creative Post to LinkedIn` → `Save Post URL & Mark Published`

### Branch B: Article Post (Link)

13) **Google Sheets: “Mark Article as Scheduled”**
   - Operation: **Update**, match `row_number`
   - Set `Approval Status = Scheduled`
   - Connect: `Route by Post Type` (Article output) → `Mark Article as Scheduled`

14) **Wait node: “Wait Until Article Scheduled Time”**
   - Same configuration as the image wait node
   - Connect: `Mark Article as Scheduled` → `Wait Until Article Scheduled Time`

15) **Aggregate: “Prepare Article Data”** (optional)
   - Connect: `Wait Until Article Scheduled Time` → `Prepare Article Data`

16) **LinkedIn: “Publish Article Link to LinkedIn”**
   - Post as: Organization
   - Organization ID: your org ID
   - Share media category: **ARTICLE**
   - Text: Caption
   - Original URL: map from `Media URL`
   - Connect: `Prepare Article Data` → `Publish Article Link to LinkedIn`

17) **Google Sheets: “Save Article URL & Mark Published”**
   - Operation: **Update**, match `row_number`
   - Set:
     - `Post URL = {{"https://www.linkedin.com/feed/update/" + $json.urn}}`
     - `Approval Status = Published`
   - Connect: `Publish Article Link to LinkedIn` → `Save Article URL & Mark Published`

**Credentials needed**
- Google Sheets OAuth2 (read/update the spreadsheet)
- Google Drive OAuth2 (download assets)
- LinkedIn OAuth2 (organization posting / community management scopes)

**No sub-workflows are invoked** in this JSON.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow runs at :45 during 9–12 (4 runs/day) and processes only the first eligible post each run. | Sticky Note / design intent |
| The notes claim filtering by Platform=LinkedIn and Approval Status="Good", but the provided nodes do not implement that filtering explicitly. | Verify “Fetch Approved LinkedIn Posts” configuration or add an IF/Code filter node. |
| Wait nodes parse `Scheduled On` with format `yyyy-MM-dd HH-mm` and convert `America/New_York` → `Asia/Kolkata`. Adjust to your actual sheet format and desired timezone. | Wait nodes expressions |
| Organization ID is hard-coded as `56420402` in both LinkedIn publish nodes; update it to your LinkedIn organization page ID. | Sticky Note setup step |
| Tracking sheet columns expected: Scheduled On, Platform, Post Type, Caption, Media URL, Approval Status, Post URL, row_number. | Sticky Note setup step |