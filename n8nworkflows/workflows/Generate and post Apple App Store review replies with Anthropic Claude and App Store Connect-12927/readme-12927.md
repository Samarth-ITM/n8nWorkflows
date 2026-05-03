Generate and post Apple App Store review replies with Anthropic Claude and App Store Connect

https://n8nworkflows.xyz/workflows/generate-and-post-apple-app-store-review-replies-with-anthropic-claude-and-app-store-connect-12927


# Generate and post Apple App Store review replies with Anthropic Claude and App Store Connect

## 1. Workflow Overview

This workflow automates **generating** and **posting** replies to Apple App Store reviews using **App Store Connect API**, **Anthropic Claude**, **Google Drive/Sheets (via XLSX files)**, and **Slack**. It is designed for app developers or community management teams who want AI-drafted replies with a **human approval step**, avoiding dedicated review-management platforms.

### 1.1 Daily AI-response generation (10:00)
- Pull a list of apps from an n8n Data Table.
- For each app, fetch reviews from App Store Connect.
- Filter to recent reviews (labeled “yesterday” but currently configured as last 5 days).
- Generate structured AI responses (JSON) via Claude.
- Build an XLSX spreadsheet containing reviews + proposed responses, upload it to Google Drive folder **ToReview**, and send a Slack message with the file URL.

### 1.2 Human-in-the-loop review (manual)
- A human reviews/edits the spreadsheet in **ToReview**.
- They move the spreadsheet to **ToSubmit** when ready.

### 1.3 Daily posting of approved responses (17:00)
- Search Google Drive **ToSubmit** for spreadsheets.
- Download XLSX, parse rows, and for each row post a response to App Store Connect.
- Write a success/failure log entry into an n8n Data Table.
- Move the processed spreadsheet to **Archived**.

---

## 2. Block-by-Block Analysis

### Block A — Documentation / in-canvas notes
**Overview:** Provides context, prerequisites, and explains the 3-step process (generate → human review → post).  
**Nodes involved:** Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note7, Sticky Note8, Sticky Note10, Sticky Note12, Sticky Note13, Sticky Note14, Sticky Note15, Sticky Note16, Sticky Note17, Sticky Note18, Sticky Note19.

#### Node details (all Sticky Notes)
- **Type/role:** `stickyNote` (documentation only; no runtime effect)
- **Key content captured in tables below** (Section 3 + Section 5)
- **Failure/edge cases:** none (non-executing)

---

### Block B — Scheduled trigger (10:00) + app list retrieval
**Overview:** Starts the “generation” run daily at 10:00 and retrieves the list of App Store apps to process.  
**Nodes involved:** Trigger download, Fetch list of applications, Fetch app id and name.

#### 1) Trigger download
- **Type/role:** Schedule Trigger — entry point for generation phase.
- **Config:** Runs daily at **10:00** (triggerAtHour: 10).
- **Outputs:** To **Fetch list of applications**.
- **Edge cases:** timezone differences (instance timezone), DST shifts.

#### 2) Fetch list of applications
- **Type/role:** n8n Data Table — reads configuration data (apps to monitor).
- **Config:** Operation **Get**, **Return All = true**, Data Table: **“Apple App Store apps”**.
- **Output:** Rows must contain at least `app_id`, `name`.
- **Edge cases/failures:** missing table, permissions, empty table (no apps → nothing happens).

#### 3) Fetch app id and name
- **Type/role:** Set — normalizes fields for downstream nodes.
- **Config:** sets:
  - `app_id = {{$json.app_id}}`
  - `name = {{$json.name}}`
- **Output:** Into **Loop over apps**.
- **Edge cases:** if upstream schema differs (e.g., appId vs app_id), downstream calls will break.

---

### Block C — Per-app loop + App Store Connect review retrieval
**Overview:** Iterates over each app, builds an auth token, calls App Store Connect to fetch reviews, and splits the response into individual review items.  
**Nodes involved:** Loop over apps, JWT, HTTP Request - App Store Reviews, Split Out fetched reviews, Fetch yesterday's reviews, Any reviews yesterday?, Fetch fields required by LLM.

#### 1) Loop over apps
- **Type/role:** Split In Batches — iterates over apps (batching/loop control).
- **Connections:**
  - **Output 0:** to **Get rid of empty items** (used later when the loop returns combined items)
  - **Output 1:** to **JWT** (drives fetching reviews per app)
- **Edge cases:** if batch size defaults are not tuned, large app lists can increase runtime. Also, this workflow “re-enters” the loop from later nodes, so incorrect loop wiring can cause unexpected iteration patterns.

#### 2) JWT
- **Type/role:** JWT generator — creates short-lived token claims for Apple.
- **Config highlights:**
  - Header option: `kid = K11111111`
  - Claims JSON includes:
    - `iss` (issuer)
    - `aud = appstoreconnect-v1`
    - `bid` (bundle id)
    - `exp` = now + 10 minutes (epoch seconds)
  - Uses **JWT Auth credential** (“Apple Connect JWT Auth account”)
- **Output:** to **HTTP Request - App Store Reviews**.
- **Potential issues:**
  - Incorrect `iss/kid/bid` mapping to Apple key.
  - Time drift → token “expired” or “not yet valid”.
  - Confusion: this node generates a JWT, but the HTTP nodes use **httpBearerAuth credentials**; ensure the bearer token actually matches the generated JWT strategy you intend (see notes in Node D1/E1).

#### 3) HTTP Request - App Store Reviews
- **Type/role:** HTTP Request — calls App Store Connect “List customer reviews for an app”.
- **Config:**
  - URL: `https://api.appstoreconnect.apple.com/v1/apps/{{ $('Loop over apps').item.json.app_id }}/customerReviews?sort=-createdDate`
  - Auth: **Generic Credential Type → httpBearerAuth**
  - Pagination: uses `links.next` from response; maxRequests = 5.
- **Output:** to **Split Out fetched reviews**.
- **Edge cases/failures:**
  - 401/403 (bad token, wrong issuer/key, missing role in App Store Connect).
  - Pagination cap: only first 5 pages fetched; high-volume apps may miss reviews.
  - Rate limits / 429.
  - If Apple returns unexpected schema, next-link expressions fail.

#### 4) Split Out fetched reviews
- **Type/role:** Split Out — splits API response `data[]` into items.
- **Config:** `fieldToSplitOut = data`; `onError = continueErrorOutput` (so schema issues don’t hard-fail the workflow).
- **Outputs:**
  - Main output → **Fetch yesterday’s reviews**
  - Also wired to **Loop over apps** (to continue loop on error/empty; see connections)
- **Edge cases:** If `data` is missing or not an array, node errors but continues via error output; can cause silent skipping.

#### 5) Fetch yesterday's reviews
- **Type/role:** Filter — selects reviews meeting date criteria.
- **Config (important):**
  - Condition compares `createdDate` formatted to `yyyy-MM-dd`
  - Check: **afterOrEquals** `{{$today.minus({days: 5}).format('yyyy-MM-dd')}}`
  - `onError = continueRegularOutput`, `alwaysOutputData = true`
- **Behavior note:** Despite the node name, it filters to **last 5 days**, not “yesterday”.
- **Output:** to **Any reviews yesterday?**
- **Edge cases:**
  - Date parsing depends on `createdDate` existence and ISO format.
  - Using string-formatted dates can behave unexpectedly around timezone boundaries.

#### 6) Any reviews yesterday?
- **Type/role:** IF — decides whether to process reviews or skip to next app.
- **Config:** checks whether `{{$json.data.id}}` exists.
- **Outputs:**
  - True → **Fetch fields required by LLM**
  - False → **Loop over apps** (continue to next app)
- **Edge cases:** If `alwaysOutputData` from prior filter passes items without `data`, false path should handle it (it does).

#### 7) Fetch fields required by LLM
- **Type/role:** Set — extracts required fields from each review and attaches app metadata.
- **Config sets:**
  - `reviewId = {{$json.data.id}}`
  - `title/body/rating/territory` from `data.attributes.*`
  - `app_id` and `app_name` from `$('Loop over apps').item.json.*`
- **Output:** to **Aggregate reviews for LLM use**
- **Edge cases:** missing `attributes` fields; also note it sets `rating` but later expects `starRating` in the AI output (mapping mismatch risk).

---

### Block D — Aggregate reviews, generate AI replies, normalize to items, and continue app loop
**Overview:** Aggregates all reviews for the current app, prompts Claude to generate structured JSON replies, splits replies to items, and feeds them back into the app loop and later spreadsheet creation.  
**Nodes involved:** Aggregate reviews for LLM use, LLM Response Generator, Anthropic Chat Model, Structured Output Parser, Split responses.

#### 1) Aggregate reviews for LLM use
- **Type/role:** Aggregate — combines items into one payload for the LLM.
- **Config:** `aggregateAllItemData` into field `reviews`.
- **Output:** to **LLM Response Generator**
- **Edge cases:** Large review volume can exceed LLM context limits.

#### 2) LLM Response Generator
- **Type/role:** LangChain “Chain LLM” — prompt-based generation.
- **Config highlights:**
  - Prompt instructs tone/length constraints, English-only, no quotes, <500 chars.
  - Requests JSON array output with fields: `app_id`, `app_name`, `reviewId`, `title`, `body`, `starRating`, `response`.
  - Injects reviews via `{{ $json.toJsonString() }}`
  - **retryOnFail = true**
  - **onError = continueErrorOutput**
  - Uses:
    - **Anthropic Chat Model** as `ai_languageModel`
    - **Structured Output Parser** as `ai_outputParser`
- **Outputs:**
  - Main output → **Split responses**
  - Also connected back to **Loop over apps** (to continue iteration)
- **Failure modes:**
  - Model returns invalid JSON (parser fails).
  - Output violates “no quotes” or character limit (not enforced except via instruction).
  - If parser fails, you may get empty/failed items but loop continues due to continue-on-error.

#### 3) Anthropic Chat Model
- **Type/role:** LLM provider node.
- **Config:** model = **Claude Sonnet 4.5** (as selected in node).
- **Edge cases:** credential/plan limits; model name availability; Anthropic rate limits.

#### 4) Structured Output Parser
- **Type/role:** Enforces JSON schema-like structure based on example.
- **Config:** Example JSON array with required keys.
- **Edge cases:** It is example-based; real enforcement depends on node behavior/version. Invalid outputs can stop parsing and send error downstream (but chain node continues on error output).

#### 5) Split responses
- **Type/role:** Split Out — converts the parsed JSON array (`output`) into one item per response.
- **Config:** `fieldToSplitOut = output`
- **Output:** to **Loop over apps**
- **Important design note:** This wiring implies the workflow is using the loop node as a hub to gather items before the final spreadsheet steps; ensure the loop semantics in your n8n version behave as intended (otherwise you can end up re-looping unexpectedly).

---

### Block E — Build spreadsheet, upload to ToReview, notify Slack
**Overview:** After responses exist, the workflow filters empties, sorts by rating, converts to XLSX, uploads to Google Drive ToReview, and posts the link to Slack.  
**Nodes involved:** Get rid of empty items, Sort, Convert to File, Upload file, Send to Slack.

#### 1) Get rid of empty items
- **Type/role:** Filter — removes items lacking `reviewId`.
- **Config:** condition “reviewId exists”.
- **Input:** from **Loop over apps** (output 0).
- **Output:** to **Sort**.
- **Edge cases:** If the LLM output uses a different field name (e.g., `reviewID`), items get dropped.

#### 2) Sort
- **Type/role:** Sort — sorts rows before export.
- **Config:** sort by `starRating` ascending (field name must exist on items).
- **Edge cases:** The “Set” node earlier used `rating`, but AI output uses `starRating`. Sorting will only work if the items at this stage indeed contain `starRating` (likely coming from the LLM output). If not, sort may be a no-op or error.

#### 3) Convert to File
- **Type/role:** Convert to File — exports items to **XLSX**.
- **Config:**
  - Operation: `xlsx`
  - Filename: `Apple_{{$now.format('yyyy-MM-dd-HH-mm-ss')}}_all_review_responses`
  - `headerRow = true`
- **Output:** to **Upload file**
- **Edge cases:** binary size limits; field order depends on item keys.

#### 4) Upload file
- **Type/role:** Google Drive — uploads XLSX into ToReview folder.
- **Config:**
  - Auth: **serviceAccount**
  - Folder: **ToReview**
  - Drive: “n8n drive”
- **Output:** to **Send to Slack**
- **Edge cases:** service account must have access to the shared drive/folder; “driveId” vs “folderId” mismatch causes 404.

#### 5) Send to Slack
- **Type/role:** Slack — sends file URL.
- **Config:**
  - Channel: fixed `channelId = C11111111111`
  - Text: `Hello, a new list is ready for review: {{$json.webViewLink}}`
  - `includeLinkToWorkflow = false`
- **Edge cases:** missing `webViewLink` if Drive node doesn’t return it (depends on Drive node configuration/version); Slack auth scope issues.

---

### Block F — Scheduled trigger (17:00) + find ToSubmit spreadsheets
**Overview:** Starts the “posting” run daily at 17:00 and enumerates spreadsheets awaiting submission.  
**Nodes involved:** Trigger posting responses, Search responses ready to be posted, Download file, Extract from File, Move spreadsheet to Archived folder.

#### 1) Trigger posting responses
- **Type/role:** Schedule Trigger — entry point for posting phase.
- **Config:** daily at **17:00**.
- **Output:** to **Search responses ready to be posted**
- **Edge cases:** timezone/DST.

#### 2) Search responses ready to be posted
- **Type/role:** Google Drive — searches for files in ToSubmit folder.
- **Config:**
  - Resource: fileFolder; whatToSearch = files
  - Search method: query
  - Query: `'1YIAYsqrpcCAgWnkeEfzC1jSqs1uEPplW' in parents` (ToSubmit folder id)
  - Returns fields: `name`, `id`
  - Auth: serviceAccount
- **Output:** to **Download file**
- **Edge cases:** If ToSubmit is empty, nothing proceeds; ensure folder permissions for service account.

#### 3) Download file
- **Type/role:** Google Drive — downloads each spreadsheet file.
- **Config:** operation = download; `fileId = {{$json.id}}`
- **Outputs:** to **Extract from File** and also directly to **Move spreadsheet to Archived folder**
- **Edge cases / design concern:** Moving to archive happens immediately after download (in parallel with extraction/posting). If posting fails, the sheet may still be archived prematurely.

#### 4) Extract from File
- **Type/role:** Extract From File — parses XLSX rows into JSON items.
- **Config:** operation = xlsx; `headerRow = true`
- **Output:** to **JWT1**
- **Edge cases:** column names must match expected (reviewId/response/app_id). Empty rows can create blank items.

#### 5) Move spreadsheet to Archived folder
- **Type/role:** Google Drive — moves processed spreadsheet to Archived.
- **Config:** operation = move; destination folder = **Archived**
- **Input:** from **Download file**
- **Edge cases:** As noted, this move is not gated on successful posting.

---

### Block G — Post responses to App Store Connect + execution logging
**Overview:** For each extracted row, posts a response to App Store Connect and logs success/failure in an n8n Data Table.  
**Nodes involved:** JWT1, HTTP Request - Respond to App Store reviews, Create a success log for the review, Create an errror log for the review.

#### 1) JWT1
- **Type/role:** JWT generator for posting phase.
- **Config:** similar to JWT (kid/iss/bid/exp).
- **Output:** to **HTTP Request - Respond to App Store reviews**
- **Edge cases:** same as JWT.

#### 2) HTTP Request - Respond to App Store reviews
- **Type/role:** HTTP Request — POST customer review responses.
- **Config:**
  - URL: `https://api.appstoreconnect.apple.com/v1/customerReviewResponses`
  - Method: POST
  - Raw JSON body uses expressions:
    - review id: `{{ $('Switch').item.json.reviewId }}`
    - response body: `{{ $('Switch').item.json.response }}`
  - Auth: Generic credential type → **httpBearerAuth**
  - `onError = continueErrorOutput`
- **Outputs:**
  - Success output → **Create a success log for the review**
  - Error output → **Create an errror log for the review**
- **Critical issue:** This node references **`$('Switch')`**, but **no node named “Switch” exists** in the workflow JSON. This will cause expression errors unless your actual canvas has such a node or you renamed it after exporting.
- **Other edge cases:**
  - Apple may reject responses > 500 chars, containing disallowed characters, or duplicate response for already-responded reviews.
  - 409 conflict if response already exists.
  - 422 validation errors if payload is malformed.

#### 3) Create a success log for the review
- **Type/role:** Data Table — appends a success record.
- **Config:** writes to **“Apple App Store review response logs”** with fields:
  - `app_id = {{ $('Extract from File1').item.json.app_id }}`
  - `reviewID = {{ $('Extract from File1').item.json.reviewId }}`
  - `successful = true`
- **Critical issue:** References **`Extract from File1`**, but the node is named **“Extract from File”**. Unless another node exists, this will fail.
- **Edge cases:** schema expects `app_id` as number, but earlier it’s sometimes treated as string; conversion is disabled (`attemptToConvertTypes: false`).

#### 4) Create an errror log for the review
- **Type/role:** Data Table — appends a failure record.
- **Config:** same fields as success log but `successful = false`
- **Same critical issue:** references `Extract from File1` which does not exist.
- **Edge cases:** Logging node may run even when upstream failed due to expression errors rather than API errors, unless you correct references.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow description & prerequisites | — | — | ## Generate and post Apple App Store review replies with Anthropic Claude, Google Drive and App Store Connect API … (full note text in Section 5) |
| Sticky Note1 | Sticky Note | Block label: AI generation | — | — | ## FIRST STEP: GENERATE AI RESPONSES … |
| Sticky Note2 | Sticky Note | Block label: human review | — | — | ## SECOND STEP: HUMAN IN THE LOOP … |
| Sticky Note3 | Sticky Note | Block label: posting | — | — | ## THIRD STEP: POST RESPONSES … |
| Sticky Note7 | Sticky Note | Drive search step note | — | — | Search the files in *ToSubmit* folder |
| Sticky Note8 | Sticky Note | Drive download note | — | — | Download responses sheet |
| Sticky Note10 | Sticky Note | Apple endpoint link note | — | — | Post responses using App Store Connect API: https://developer.apple.com/documentation/appstoreconnectapi/post-v1-customerreviewresponses |
| Sticky Note13 | Sticky Note | Logs table fields note | — | — | ### Data table fields … reviewID / app_id / successful |
| Trigger posting responses | Schedule Trigger | Start posting phase (17:00) | — | Search responses ready to be posted | (no sticky note) |
| JWT | JWT | Build Apple JWT for review fetch | Loop over apps | HTTP Request - App Store Reviews | Fetch reviews from App Store Connect API: https://developer.apple.com/documentation/appstoreconnectapi/get-v1-apps-_id_-customerreviews |
| Fetch app id and name | Set | Normalize app fields | Fetch list of applications | Loop over apps | ### Data table fields … app_id / name |
| HTTP Request - App Store Reviews | HTTP Request | Fetch reviews per app | JWT | Split Out fetched reviews | Fetch reviews from App Store Connect API: https://developer.apple.com/documentation/appstoreconnectapi/get-v1-apps-_id_-customerreviews |
| Sticky Note14 | Sticky Note | Apps table fields note | — | — | ### Data table fields … app_id / name |
| Sticky Note15 | Sticky Note | Spreadsheet grouping note | — | — | Collect all the reviews and responses in a spreadsheet, sorted by rating |
| Sticky Note16 | Sticky Note | Upload location note | — | — | Upload the spreadsheet to "ToReview* folder |
| Send to Slack | Slack | Notify reviewers with Drive link | Upload file | — | Send the Google Drive url of the file in a Slack message |
| Sticky Note17 | Sticky Note | Slack note | — | — | Send the Google Drive url of the file in a Slack message |
| Sticky Note18 | Sticky Note | Apple GET endpoint link | — | — | Fetch reviews from App Store Connect API: https://developer.apple.com/documentation/appstoreconnectapi/get-v1-apps-_id_-customerreviews |
| Split Out fetched reviews | Split Out | Split API `data[]` into items | HTTP Request - App Store Reviews | Fetch yesterday's reviews; Loop over apps | If there are no reviews for the app from yesterday, continue the loop with the next app |
| Sticky Note19 | Sticky Note | “no reviews” behavior note | — | — | If there are no reviews… continue the loop… |
| Fetch list of applications | Data Table | Load apps list | Trigger download | Fetch app id and name | ### Data table fields … app_id / name |
| Loop over apps | Split In Batches | Iterate apps / loop hub | Fetch app id and name; Split responses; Any reviews yesterday? (false); Split Out fetched reviews (2nd output); LLM Response Generator (2nd output) | Get rid of empty items; JWT | (no sticky note) |
| Get rid of empty items | Filter | Remove items missing reviewId | Loop over apps | Sort | Collect all the reviews and responses in a spreadsheet, sorted by rating |
| Sort | Sort | Sort by starRating | Get rid of empty items | Convert to File | Collect all the reviews and responses in a spreadsheet, sorted by rating |
| Convert to File | Convert to File | Export to XLSX | Sort | Upload file | Collect all the reviews and responses in a spreadsheet, sorted by rating |
| Upload file | Google Drive | Upload XLSX to ToReview | Convert to File | Send to Slack | Upload the spreadsheet to "ToReview* folder |
| Fetch yesterday's reviews | Filter | Keep recent reviews (configured: last 5 days) | Split Out fetched reviews | Any reviews yesterday? | (no sticky note) |
| Any reviews yesterday? | IF | Skip app if no reviews | Fetch yesterday's reviews | Fetch fields required by LLM; Loop over apps | If there are no reviews… continue the loop… |
| Fetch fields required by LLM | Set | Extract review fields for prompt | Any reviews yesterday? (true) | Aggregate reviews for LLM use | (no sticky note) |
| Aggregate reviews for LLM use | Aggregate | Combine reviews for LLM input | Fetch fields required by LLM | LLM Response Generator | (no sticky note) |
| LLM Response Generator | LangChain Chain LLM | Generate structured replies | Aggregate reviews for LLM use | Split responses; Loop over apps | (no sticky note) |
| Anthropic Chat Model | Anthropic Chat Model | Claude model provider | — | LLM Response Generator (ai_languageModel) | (no sticky note) |
| Structured Output Parser | Structured Output Parser | Parse JSON array output | — | LLM Response Generator (ai_outputParser) | (no sticky note) |
| Split responses | Split Out | Split generated JSON array into items | LLM Response Generator | Loop over apps | (no sticky note) |
| Trigger download | Schedule Trigger | Start generation phase (10:00) | — | Fetch list of applications | (no sticky note) |
| JWT1 | JWT | Build Apple JWT for posting | Extract from File | HTTP Request - Respond to App Store reviews | Post responses using App Store Connect API: https://developer.apple.com/documentation/appstoreconnectapi/post-v1-customerreviewresponses |
| HTTP Request - Respond to App Store reviews | HTTP Request | POST responses to Apple | JWT1 | Create a success log for the review; Create an errror log for the review | Post responses using App Store Connect API: https://developer.apple.com/documentation/appstoreconnectapi/post-v1-customerreviewresponses |
| Sticky Note12 | Sticky Note | Archive step note | — | — | After posting all responses, move the spreadsheet to Archived folder |
| Search responses ready to be posted | Google Drive | List files in ToSubmit | Trigger posting responses | Download file | Search the files in *ToSubmit* folder |
| Download file | Google Drive | Download XLSX | Search responses ready to be posted | Extract from File; Move spreadsheet to Archived folder | Download responses sheet |
| Extract from File | Extract From File | Parse XLSX into rows | Download file | JWT1 | (no sticky note) |
| Move spreadsheet to Archived folder | Google Drive | Move file to Archived | Download file | — | After posting all responses, move the spreadsheet to Archived folder |
| Create a success log for the review | Data Table | Write success log | HTTP Request - Respond to App Store reviews (success) | — | ### Data table fields … reviewID / app_id / successful |
| Create an errror log for the review | Data Table | Write error log | HTTP Request - Respond to App Store reviews (error) | — | ### Data table fields … reviewID / app_id / successful |

---

## 4. Reproducing the Workflow from Scratch

1) **Create supporting Data Tables (n8n)**
   1. Create Data Table **“Apple App Store apps”** with fields:
      - `app_id` (string or number; be consistent)
      - `name` (string)
   2. Create Data Table **“Apple App Store review response logs”** with fields:
      - `reviewID` (string)
      - `app_id` (number or string)
      - `successful` (boolean)

2) **Prepare Google Drive folders**
   1. Create folders: **ToReview**, **ToSubmit**, **Archived**
   2. Note each folder ID (for Drive queries/moves).
   3. Ensure the Google **Service Account** has access (shared with service account email, or configured Shared Drive access).

3) **Prepare App Store Connect API access**
   1. Create an App Store Connect API key (Issuer ID + Key ID + private key `.p8`).
   2. In n8n, create credentials:
      - **JWT Auth** credential (for signing with the private key, setting algorithm Apple expects, etc.).
      - **HTTP Bearer Auth** credential (if you intend to provide the signed JWT as bearer token dynamically, consider instead configuring the HTTP node to use the JWT output; see note below).
   3. Confirm roles/permissions allow:
      - GET customer reviews
      - POST customer review responses

4) **Prepare Anthropic credentials**
   1. Add Anthropic credential in n8n.
   2. Confirm model availability (Claude Sonnet 4.5 or adjust to your available model).

5) **Prepare Slack credentials**
   1. Add Slack API credential with permission to post messages to the target channel.
   2. Record the `channelId`.

---

### Generation phase (10:00)

6) **Add “Trigger download”**
   - Node: **Schedule Trigger**
   - Set to run daily at **10:00**
   - Connect to **Fetch list of applications**

7) **Add “Fetch list of applications”**
   - Node: **Data Table**
   - Operation: **Get**, Return All: **true**
   - Choose table: **Apple App Store apps**
   - Connect to **Fetch app id and name**

8) **Add “Fetch app id and name”**
   - Node: **Set**
   - Add fields:
     - `app_id = {{$json.app_id}}`
     - `name = {{$json.name}}`
   - Connect to **Loop over apps**

9) **Add “Loop over apps”**
   - Node: **Split In Batches**
   - Default options are acceptable initially.
   - Connect output (batch) to **JWT**
   - Also connect (later aggregation output) to **Get rid of empty items** as in the original design.

10) **Add “JWT” (for fetching reviews)**
   - Node: **JWT**
   - Header: set `kid` to your Apple Key ID
   - Claims:
     - `iss` = your Issuer ID
     - `aud` = `appstoreconnect-v1`
     - `bid` = your app bundle id (or remove if not needed for your use)
     - `exp` = now + 10 minutes (as in workflow)
   - Connect to **HTTP Request - App Store Reviews**

11) **Add “HTTP Request - App Store Reviews”**
   - Node: **HTTP Request**
   - Method: GET
   - URL: `https://api.appstoreconnect.apple.com/v1/apps/{{$('Loop over apps').item.json.app_id}}/customerReviews?sort=-createdDate`
   - Auth: Bearer token for Apple
   - Enable pagination using response `links.next` (optional but recommended).
   - Connect to **Split Out fetched reviews**

12) **Add “Split Out fetched reviews”**
   - Node: **Split Out**
   - Field: `data`
   - On error: “continue” (optional)
   - Connect to **Fetch yesterday’s reviews**

13) **Add “Fetch yesterday’s reviews”**
   - Node: **Filter**
   - Condition: createdDate >= threshold
   - If you truly want “yesterday only”, use a proper window (e.g., >= startOfYesterday and < startOfToday). The imported workflow uses last **5** days.
   - Connect to **Any reviews yesterday?**

14) **Add “Any reviews yesterday?”**
   - Node: **IF**
   - Condition: `$json.data.id` exists
   - True → **Fetch fields required by LLM**
   - False → back to **Loop over apps** (continue)

15) **Add “Fetch fields required by LLM”**
   - Node: **Set**
   - Map:
     - `reviewId`, `title`, `body`, `rating`, `territory` from `data.attributes`
     - `app_id`, `app_name` from loop item
   - Connect to **Aggregate reviews for LLM use**

16) **Add “Aggregate reviews for LLM use”**
   - Node: **Aggregate**
   - Aggregate all item data into field `reviews`
   - Connect to **LLM Response Generator**

17) **Add “Anthropic Chat Model”**
   - Node: **Anthropic Chat Model**
   - Select model (e.g., Claude Sonnet)
   - Configure credentials

18) **Add “Structured Output Parser”**
   - Node: **Structured Output Parser**
   - Provide a JSON array example matching desired output keys

19) **Add “LLM Response Generator”**
   - Node: **Chain LLM**
   - Paste the prompt (adapt as needed)
   - Attach:
     - Language model: **Anthropic Chat Model**
     - Output parser: **Structured Output Parser**
   - Connect to **Split responses**

20) **Add “Split responses”**
   - Node: **Split Out**
   - Field: `output`
   - Connect to **Loop over apps** (to keep iterating / collecting items)

21) **Add “Get rid of empty items”**
   - Node: **Filter**
   - Condition: `reviewId` exists
   - Input from **Loop over apps**
   - Output to **Sort**

22) **Add “Sort”**
   - Node: **Sort**
   - Sort by `starRating` (ensure your items contain this field; otherwise change to `rating`)
   - Output to **Convert to File**

23) **Add “Convert to File”**
   - Node: **Convert to File**
   - Operation: `xlsx`
   - Filename expression as desired
   - Header row: true
   - Output to **Upload file**

24) **Add “Upload file”**
   - Node: **Google Drive**
   - Operation: upload
   - Authentication: service account
   - Folder: **ToReview**
   - Output to **Send to Slack**

25) **Add “Send to Slack”**
   - Node: **Slack**
   - Operation: post message to channel
   - Text includes the Drive link returned by upload (e.g., `webViewLink`)
   - Configure Slack credentials + channel id

---

### Posting phase (17:00)

26) **Add “Trigger posting responses”**
   - Node: **Schedule Trigger**
   - Daily at **17:00**
   - Output to **Search responses ready to be posted**

27) **Add “Search responses ready to be posted”**
   - Node: **Google Drive**
   - Search files by query: `'<ToSubmitFolderId>' in parents`
   - Return fields: `id`, `name`
   - Output to **Download file**

28) **Add “Download file”**
   - Node: **Google Drive**
   - Operation: download
   - `fileId = {{$json.id}}`
   - Output to **Extract from File**
   - (Recommended improvement) Only move to archive *after* successful posting for that file.

29) **Add “Extract from File”**
   - Node: **Extract From File**
   - Operation: XLSX
   - Header row: true
   - Output to **JWT1**

30) **Add “JWT1”**
   - Node: **JWT**
   - Same Apple claims/header approach as in generation phase
   - Output to **HTTP Request - Respond to App Store reviews**

31) **Add “HTTP Request - Respond to App Store reviews”**
   - Node: **HTTP Request**
   - Method: POST
   - URL: `https://api.appstoreconnect.apple.com/v1/customerReviewResponses`
   - Body maps from each extracted row:
     - review id = current item `reviewId`
     - response text = current item `response`
   - **Important:** do not reference a non-existent node (replace `$('Switch')...` with `$json.reviewId` / `$json.response` or the correct node name).
   - Success output → **Create a success log for the review**
   - Error output → **Create an error log for the review**

32) **Add log nodes**
   - Node: **Data Table** (append/create)
   - Map fields using the *current* row item (avoid `Extract from File1` unless that is your real node name).
   - Success: `successful = true`
   - Error: `successful = false`

33) **Add “Move spreadsheet to Archived folder”**
   - Node: **Google Drive**
   - Operation: move
   - Source fileId = the spreadsheet file id from the search/download item
   - Destination folder = **Archived**
   - **Recommended:** place this after all rows have been posted (requires grouping per file).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Generate and post Apple App Store review replies with Anthropic Claude, Google Drive and App Store Connect API …” (includes prerequisites and 7 workflow steps) | Overall workflow sticky note (in-canvas description) |
| App Store Connect API — Fetch reviews endpoint | https://developer.apple.com/documentation/appstoreconnectapi/get-v1-apps-_id_-customerreviews |
| App Store Connect API — Post review responses endpoint | https://developer.apple.com/documentation/appstoreconnectapi/post-v1-customerreviewresponses |
| Human step: reviewer edits spreadsheet in ToReview then moves to ToSubmit | Described in sticky notes “SECOND STEP: HUMAN IN THE LOOP” |
| Important integrity issues to fix before running posting phase | The workflow references non-existent nodes/aliases: `$('Switch')` and `$('Extract from File1')` (must be corrected to real node names or `$json.*`). Also, the archive move currently occurs immediately after download, not after confirmed posting. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.