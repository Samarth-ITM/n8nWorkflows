Generate and post Google Play review replies with Anthropic Claude and Google Drive

https://n8nworkflows.xyz/workflows/generate-and-post-google-play-review-replies-with-anthropic-claude-and-google-drive-12591


# Generate and post Google Play review replies with Anthropic Claude and Google Drive

## 1. Workflow Overview

**Title:** Generate and post Google Play review replies with Anthropic Claude and Google Drive  
**Purpose:** This workflow automates the end-to-end process of responding to Google Play Store reviews for one or more Android apps. It (1) fetches yesterday’s reviews, (2) uses an Anthropic Claude chat model to generate draft replies in a structured JSON format, (3) exports the results into an XLSX spreadsheet stored in Google Drive for human review, then (4) later downloads reviewed spreadsheets and posts the approved replies back to Google Play via the Android Publisher API, while logging successes/failures and archiving processed files.

**Primary use cases**
- Community management teams replying daily to new Google Play reviews across multiple apps.
- Human-in-the-loop moderation: AI drafts → human edits → automated posting.
- Cost reduction vs. third-party review management tools by using n8n + Google APIs + LLM.

### 1.1 Logical blocks (by dependencies)

1. **Scheduled AI Draft Generation (10:00)**
   - Trigger → load app list → loop apps → fetch reviews → filter to yesterday → generate AI responses → create spreadsheet → upload to *ToReview*.

2. **Human Review Step (out-of-band)**
   - Community manager edits the spreadsheet and moves it from *ToReview* to *ToSubmit* (manual action outside n8n).

3. **Scheduled Posting (17:00)**
   - Trigger → find spreadsheets in *ToSubmit* → download → extract rows → post replies → log success/failure → move spreadsheet to *Archived*.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Daily trigger and app inventory (10:00)

**Overview:** Starts the “draft generation” phase every day at 10 AM, loading the list of apps (bundle IDs + names) from an n8n Data Table, then preparing items for looping.

**Nodes involved**
- Trigger download
- Fetch list of applications
- Fetch app bundle id and name
- Loop over apps

#### Node: **Trigger download**
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point for morning run.
- **Config choices:** Runs daily at **10:00** (`triggerAtHour: 10`). Has node note “daily at 10am”.
- **Outputs:** Main → *Fetch list of applications*.
- **Edge cases:** Timezone depends on n8n instance timezone; confirm desired TZ for “yesterday” filtering later.

#### Node: **Fetch list of applications**
- **Type / role:** Data Table (`n8n-nodes-base.dataTable`) — reads the inventory of apps to process.
- **Config choices:** Operation **Get**, `returnAll: true`, Data Table: **Google Play apps**.
- **Expected schema (from Sticky Note12):**
  - `bundle_id` (e.g., from Play URL `https://play.google.com/store/apps/details?id=com.anthropic.claude`)
  - `name` (app name)
- **Outputs:** Main → *Fetch app bundle id and name*.
- **Failure modes:** Data Table missing, permission issues in project, empty table → no apps processed.

#### Node: **Fetch app bundle id and name**
- **Type / role:** Set (`n8n-nodes-base.set`) — normalizes field names for downstream use.
- **Config choices:** Creates:
  - `bundle_id` = `{{$json.bundle_id}}`
  - `name` = `{{$json.name}}`
- **Outputs:** Main → *Loop over apps*.
- **Edge cases:** If incoming rows don’t contain `bundle_id`/`name`, expressions resolve to `null`.

#### Node: **Loop over apps**
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates apps in batches (default batch size if not set).
- **Config choices:** No explicit batch size provided (uses n8n default).
- **Connections:**
  - Output 0 → *Get rid of empty items* (this is reached later via loopback)
  - Output 1 → *Fetch reviews from Google Play* (main app processing path)
- **Edge cases:** If not carefully looped back, may stop early. Here, the loopback is driven after responses are split (see Block 2.3).

**Sticky note coverage**
- “FIRST STEP: GENERATE AI RESPONSES…” (Sticky Note1) conceptually applies to this entire morning branch.

---

### Block 2.2 — Fetch Google Play reviews and filter to yesterday

**Overview:** For each app, fetches reviews from Google Play Android Publisher API with pagination, splits them into individual review items, keeps only those modified “yesterday”, and skips apps with none.

**Nodes involved**
- Fetch reviews from Google Play
- Split out fetched reviews
- Fetch yesterday's reviews
- Any reviews yesterday?

#### Node: **Fetch reviews from Google Play**
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Android Publisher API.
- **Config choices:**
  - URL: `https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{{ $json.bundle_id }}/reviews`
  - Auth: **Predefined credential type → Google API** using **Google Service Account 1**
  - Pagination: uses `nextPageToken` via parameter named `token`, stops when `nextPageToken == null`, max 10 requests, 10 ms (?) request interval set to 10 (n8n interprets as ms in many nodes; verify your node’s unit).
  - `alwaysOutputData: true`
- **Outputs:** Main → *Split out fetched reviews*.
- **Failure modes / edge cases:**
  - Service account not authorized in Play Console (most common). Must grant access to the app(s) in Google Play Console and use correct scope.
  - API returns quota or 403/401 errors.
  - Pagination expression references `$response.body.tokenPagination.nextPageToken` — if API response shape differs, pagination may fail.

#### Node: **Split out fetched reviews**
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — explodes `reviews[]` into one item per review.
- **Config choices:** Field to split: `reviews`.
- **Outputs:** Main → *Fetch yesterday’s reviews*.
- **Edge cases:** If `reviews` is missing or not an array, node may output zero items.

#### Node: **Fetch yesterday's reviews**
- **Type / role:** Filter (`n8n-nodes-base.filter`) — keeps only reviews last modified yesterday.
- **Config choices:**
  - Condition compares:
    - Left: `{{ $json.comments[0].userComment.lastModified.seconds.toDateTime('s').toISODate() }}`
    - Right: `{{ $today.minus({days: 1}) }}`
  - `onError: continueRegularOutput`
  - `alwaysOutputData: true`
- **Outputs:** Main → *Any reviews yesterday?*
- **Failure modes / edge cases:**
  - If `comments[0]...lastModified.seconds` is missing, expression can fail; `onError: continueRegularOutput` reduces hard-fail risk but may let through unexpected items depending on n8n behavior.
  - Timezone mismatch: “yesterday” uses n8n server timezone; Play timestamps are epoch seconds (UTC). This can cause off-by-one-day around midnight.

#### Node: **Any reviews yesterday?**
- **Type / role:** IF (`n8n-nodes-base.if`) — branches based on whether a review exists.
- **Config choices:** “exists” check on `{{$json.reviewId}}`.
- **Outputs:**
  - True → *Fetch fields required by LLM*
  - False → *Loop over apps* (skip current app)
- **Sticky note coverage:** Sticky Note4 (“If there are no reviews… continue the loop…”) applies here.
- **Edge cases:** If the filter outputs items without `reviewId`, those will route to “no reviews” branch.

**Sticky note coverage**
- Sticky Note9 (“Fetch reviews from Google Play API”) applies to the API fetch node.

---

### Block 2.3 — Prepare review payload, generate LLM responses, and split results

**Overview:** Normalizes the fields required by the LLM, aggregates reviews into a single payload, calls Anthropic Claude with a structured-output parser, then splits the resulting JSON list into individual items for export.

**Nodes involved**
- Fetch fields required by LLM
- Aggregate reviews for LLM use
- Anthropic Chat Model
- Structured Output Parser
- LLM Response Generator
- Split responses

#### Node: **Fetch fields required by LLM**
- **Type / role:** Set — reshapes review data into a compact schema.
- **Config choices:** Creates:
  - `reviewId` = review ID
  - `userComment` = `comments[0].userComment.text`
  - `starRating` = `comments[0].userComment.starRating` (number)
  - `reviewerLanguage` = `comments[0].userComment.reviewerLanguage`
  - `bundleID` = `$('Fetch app bundle id and name').item.json.bundle_id` (note: field name here is `bundleID`, not `bundleId`)
  - `appName` = `$('Loop over apps').item.json.name`
- **Outputs:** Main → *Aggregate reviews for LLM use*.
- **Edge cases:**
  - Google Play review structure may differ; `comments[0]` may not exist.
  - Inconsistent naming: later prompt requests `bundleId`, but this node emits `bundleID`.

#### Node: **Aggregate reviews for LLM use**
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — combines items.
- **Config choices:** “aggregateAllItemData” into destination field `reviews`.
- **Outputs:** Main → *LLM Response Generator*.
- **Edge cases:** Large volume may exceed LLM context window; consider chunking by app or max items.

#### Node: **Anthropic Chat Model**
- **Type / role:** LangChain Anthropic chat model (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) — the LLM backend.
- **Config choices:** Model: **claude-sonnet-4-5-20250929**.
- **Connections:** Provides `ai_languageModel` input to *LLM Response Generator*.
- **Failure modes:** Missing Anthropic credentials, model not available, rate limits.

#### Node: **Structured Output Parser**
- **Type / role:** LangChain structured parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces JSON schema.
- **Config choices:** Expects an **array** of objects with:
  - `bundleId`, `reviewId`, `userComment`, `starRating`, `response`
- **Connections:** Provides `ai_outputParser` to *LLM Response Generator*.
- **Edge cases:** If LLM outputs invalid JSON or mismatched keys, parser can error and fail the chain.

#### Node: **LLM Response Generator**
- **Type / role:** LangChain LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`) — prompt + model + parser.
- **Config choices:**
  - Prompt enforces:
    - Always English responses
    - Thanking first
    - Tone/length rules: negative ≤ 5 sentences, 5-star positive ≤ 2 sentences, < 500 characters
    - No quotation marks
    - Must output JSON list matching schema
  - Injects data with: `{{ $json.toJsonString() }}`
  - `hasOutputParser: true` (uses structured parser)
- **Outputs:** Main → *Split responses*.
- **Failure modes / edge cases:**
  - JSON formatting issues (most common).
  - The prompt asks for `bundleId`, but upstream may provide `bundleID`; the LLM might copy the wrong key.
  - If review text contains quotes, the “no quotation marks” constraint can be hard to satisfy consistently.

#### Node: **Split responses**
- **Type / role:** Split Out — turns the LLM JSON array into individual items.
- **Config choices:** Field to split: `output` (the structured parser output field).
- **Connections:**
  - Main → *Loop over apps* (loopback to continue processing next app after generating responses for current batch)
- **Edge cases:** If `output` missing/empty due to parser failure, nothing continues.

---

### Block 2.4 — Build and upload spreadsheet to Google Drive (*ToReview*)

**Overview:** Filters out empty rows, converts the responses to an XLSX file, and uploads it to a Google Drive folder for human review.

**Nodes involved**
- Get rid of empty items
- Create the spreadsheet
- Upload spreadsheet

#### Node: **Get rid of empty items**
- **Type / role:** Filter — removes items without `reviewId` before export.
- **Config choices:** Condition “exists” on `{{$json.reviewId}}`.
- **Input source:** From *Loop over apps* (after some loopback path).
- **Outputs:** Main → *Create the spreadsheet*.
- **Sticky note coverage:** Sticky Note5 (“Collect all the reviews and responses in a spreadsheet”) applies here conceptually.

#### Node: **Create the spreadsheet**
- **Type / role:** Convert to File (`n8n-nodes-base.convertToFile`) — creates XLSX.
- **Config choices:** Operation **xlsx**, filename:
  - `{{$now.format('yyyy-MM-dd-HH-mm-ss') }}_all_review_responses`
- **Outputs:** Main → *Upload spreadsheet*.
- **Edge cases:** Very large datasets can produce large XLSX and slow executions.

#### Node: **Upload spreadsheet**
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) — uploads the XLSX to Drive.
- **Config choices:**
  - Auth: service account (Google Service Account 1)
  - Drive: “n8n drive”
  - Folder: **ToReview** (folder ID `1BFjKINf7etQH5Z4E-EW4Inem3sXEzo6-`)
- **Outputs:** No downstream nodes in provided JSON.
- **Failure modes:** Service account not shared on the Drive/folder; insufficient permissions; upload size limits.

**Sticky note coverage**
- Sticky Note6 (“Upload the spreadsheet to ToReview folder”) applies to *Upload spreadsheet*.

---

### Block 2.5 — Human review (manual step)

**Overview:** A human edits AI-generated responses and moves the spreadsheet from *ToReview* to *ToSubmit*. This is not automated inside the workflow.

**Nodes involved:** None (manual).

**Sticky note coverage**
- Sticky Note2 describes this step.

**Operational considerations**
- Spreadsheet column names must remain compatible with downstream posting step (notably `bundle_id` vs `bundleId` vs `bundleID`, and `reviewId`, `response`).
- If editors change headers, posting step may break.

---

### Block 2.6 — Daily trigger and retrieval of reviewed spreadsheets (17:00)

**Overview:** Starts the “posting” phase daily at 5 PM, searches Drive for files in the *ToSubmit* folder, then downloads each spreadsheet.

**Nodes involved**
- Trigger posting responses
- Search responses ready to be posted
- Download responses sheet

#### Node: **Trigger posting responses**
- **Type / role:** Schedule Trigger — entry point for afternoon run.
- **Config choices:** Runs daily at **17:00** (`triggerAtHour: 17`). Note “daily at 5pm”.
- **Outputs:** Main → *Search responses ready to be posted*.

#### Node: **Search responses ready to be posted**
- **Type / role:** Google Drive — lists files to process.
- **Config choices:**
  - Auth: service account
  - Search method: query
  - Query: `'1YIAYsqrpcCAgWnkeEfzC1jSqs1uEPplW' in parents` (this is the *ToSubmit* folder ID)
  - Returns fields: `name`, `id`
  - `returnAll: true`
- **Outputs:** Main → *Download responses sheet*.
- **Failure modes:** Folder not shared with service account, wrong folder ID, too many files.

#### Node: **Download responses sheet**
- **Type / role:** Google Drive — downloads each spreadsheet file.
- **Config choices:** Operation **download**, `fileId: {{$json.id}}`.
- **Outputs:** Main → *Extract responses* and → *Move spreadsheet to Archived folder* (parallel paths).
- **Sticky note coverage:** Sticky Note8 (“Download files in the folder”) applies here.
- **Edge cases:** Non-XLSX files in folder will break extraction.

**Sticky note coverage**
- Sticky Note7 (“Search the files in ToSubmit folder”) applies to the search node.

---

### Block 2.7 — Extract rows, post replies, log results, and archive

**Overview:** Extracts rows from XLSX, posts each reply to Google Play, logs success/failure to a Data Table, and moves the processed spreadsheet to an Archived folder.

**Nodes involved**
- Extract responses
- Post responses
- Create a success log for the review
- Create an error log for the review
- Move spreadsheet to Archived folder

#### Node: **Extract responses**
- **Type / role:** Extract From File (`n8n-nodes-base.extractFromFile`) — parses XLSX into JSON rows.
- **Config choices:** Operation **xlsx**.
- **Outputs:** Main → *Post responses*.
- **Failure modes:** Unexpected spreadsheet structure, missing headers, multiple sheets ambiguity.

#### Node: **Post responses**
- **Type / role:** HTTP Request — calls Android Publisher “reply to review” endpoint.
- **Config choices:**
  - POST URL: `https://www.googleapis.com/androidpublisher/v3/applications/{{ $json.bundle_id }}/reviews/{{$json.reviewId}}:reply`
  - Body raw JSON: `{"replyText": "{{ $json.response }}" }`
  - Header: `Content-Type: application/json`
  - Auth: Google API predefined credential (service account)
  - `onError: continueErrorOutput` (so failures go to the error output)
- **Outputs:**
  - Success output → *Create a success log for the review*
  - Error output → *Create an error log for the review*
- **Edge cases / integration risks:**
  - **Field mismatch:** URL uses `bundle_id`, but earlier generation used `bundleID` / parser expects `bundleId`. The spreadsheet must contain a `bundle_id` column for posting to succeed as configured.
  - Reply text length limits and restricted characters per Play policy.
  - 401/403 if service account lacks access to reply to reviews.

#### Node: **Create a success log for the review**
- **Type / role:** Data Table — writes a success row to logs table.
- **Config choices:**
  - Table: **Google Play review response logs**
  - Columns written:
    - `bundleID` = `{{ $('Extract responses').item.json.bundle_id }}`
    - `reviewID` = `{{ $('Extract responses').item.json.reviewId }}`
    - `successful` = `true`
- **Outputs:** None.
- **Edge cases:** Expression relies on `Extract responses` item context; if batching/concurrency changes, item linking can misalign.

#### Node: **Create an error log for the review**
- **Type / role:** Data Table — writes a failure row to logs table.
- **Config choices:**
  - Table: **Google Play review response logs**
  - Columns written:
    - `reviewID` = `{{ $('Extract responses').item.json.reviewId }}`
    - `successful` = `false`
  - Note: `bundleID` exists in schema but is not populated in this node’s configured value.
- **Outputs:** None.
- **Edge cases:** Same item-linking risk as above; additionally missing `bundleID` reduces traceability.

#### Node: **Move spreadsheet to Archived folder**
- **Type / role:** Google Drive — moves processed spreadsheet file to archive.
- **Config choices:**
  - Operation **move**
  - `fileId: {{$json.id}}` (from search/list item)
  - Destination folder: **Archived** (`1_a0VKmCKx8K4VWSh8EJ5B86NhN9PPhxu`)
  - Auth: service account
- **Inputs:** Triggered in parallel from *Download responses sheet* (not “after all responses posted” in a strict dependency sense).
- **Important behavior note:** Despite Sticky Note11’s intent (“After posting all responses…”), this node is connected directly from the download step, so the move can occur **before** all posting/logging finishes.
- **Mitigation:** Gate archiving behind completion of posting (e.g., using a Merge/Wait/Execute Workflow pattern).

**Sticky note coverage**
- Sticky Note10 (“Post responses using Google Play API”) applies to *Post responses*.
- Sticky Note13 (log table fields) applies to both logging nodes.
- Sticky Note11 applies to the archival intent, but current wiring archives immediately after download.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger download | Schedule Trigger | Start morning draft-generation run | — | Fetch list of applications | ## FIRST STEP: GENERATE AI RESPONSES<br>At 10 am, download the previous day's reviews from Google Play Store and generate AI responses in a spreadsheet in *ToReview* folder |
| Fetch list of applications | Data Table | Load app inventory (bundle_id, name) | Trigger download | Fetch app bundle id and name | ### Data table fields<br>* bundle_id: Google Play Store bundle id for each app. Bundle ids can be found in the Google Play Store url of the app. For example the bundle id of the app with url https://play.google.com/store/apps/details?id=com.anthropic.claude is *com.anthropic.claude*<br>* name: The application’s name (eg. Claude by Anthropic) |
| Fetch app bundle id and name | Set | Normalize app fields for processing | Fetch list of applications | Loop over apps | ### Data table fields<br>* bundle_id: Google Play Store bundle id for each app. Bundle ids can be found in the Google Play Store url of the app. For example the bundle id of the app with url https://play.google.com/store/apps/details?id=com.anthropic.claude is *com.anthropic.claude*<br>* name: The application’s name (eg. Claude by Anthropic) |
| Loop over apps | Split In Batches | Iterate over apps | Fetch app bundle id and name; Split responses; Any reviews yesterday? (false branch) | Fetch reviews from Google Play; Get rid of empty items | ## FIRST STEP: GENERATE AI RESPONSES<br>At 10 am, download the previous day's reviews from Google Play Store and generate AI responses in a spreadsheet in *ToReview* folder |
| Fetch reviews from Google Play | HTTP Request | Fetch reviews for one app from Android Publisher API | Loop over apps | Split out fetched reviews | Fetch reviews from Google Play API |
| Split out fetched reviews | Split Out | Split `reviews[]` into items | Fetch reviews from Google Play | Fetch yesterday's reviews | Fetch reviews from Google Play API |
| Fetch yesterday's reviews | Filter | Keep only reviews modified yesterday | Split out fetched reviews | Any reviews yesterday? | Fetch reviews from Google Play API |
| Any reviews yesterday? | IF | Skip app if no reviews yesterday | Fetch yesterday's reviews | Fetch fields required by LLM (true); Loop over apps (false) | If there are no reviews for the app from yesterday, continue the loop with the next app |
| Fetch fields required by LLM | Set | Prepare compact review fields | Any reviews yesterday? (true) | Aggregate reviews for LLM use |  |
| Aggregate reviews for LLM use | Aggregate | Combine reviews for LLM call | Fetch fields required by LLM | LLM Response Generator |  |
| Anthropic Chat Model | LangChain Anthropic Chat Model | Provides Claude model | — | LLM Response Generator (ai_languageModel) |  |
| Structured Output Parser | LangChain Structured Output Parser | Enforce JSON array schema | — | LLM Response Generator (ai_outputParser) |  |
| LLM Response Generator | LangChain LLM Chain | Generate structured responses | Aggregate reviews for LLM use | Split responses |  |
| Split responses | Split Out | Split LLM JSON list into items | LLM Response Generator | Loop over apps |  |
| Get rid of empty items | Filter | Remove items lacking reviewId before export | Loop over apps | Create the spreadsheet | Collect all the reviews and responses in a spreadsheet |
| Create the spreadsheet | Convert to File | Create XLSX spreadsheet | Get rid of empty items | Upload spreadsheet | Collect all the reviews and responses in a spreadsheet |
| Upload spreadsheet | Google Drive | Upload XLSX to *ToReview* folder | Create the spreadsheet | — | Upload the spreadsheet to "ToReview* folder |
| Trigger posting responses | Schedule Trigger | Start afternoon posting run | — | Search responses ready to be posted | ## THIRD STEP: POST RESPONSES<br>At 5pm, fetch the spreadsheets in *ToSubmit* folder, post the responses using the Google Play Store API, create execution logs and move the processed spreadsheets to a different folder called *Archived* |
| Search responses ready to be posted | Google Drive | List files in *ToSubmit* folder | Trigger posting responses | Download responses sheet | Search the files in *ToSubmit* folder |
| Download responses sheet | Google Drive | Download each spreadsheet | Search responses ready to be posted | Extract responses; Move spreadsheet to Archived folder | Download files in the folder |
| Extract responses | Extract From File | Parse XLSX rows to JSON | Download responses sheet | Post responses |  |
| Post responses | HTTP Request | Post replies to Google Play reviews | Extract responses | Create a success log for the review (success); Create an error log for the review (error) | Post responses using Google Play API |
| Create a success log for the review | Data Table | Log successful posts | Post responses (success) | — | ### Data table fields<br>* reviewID: Google Play Store Identifier for each review.<br>* bundleID: Corresponding application bundle identifier.<br>* successful: Boolean status indicating the posting success. |
| Create an error log for the review | Data Table | Log failed posts | Post responses (error) | — | ### Data table fields<br>* reviewID: Google Play Store Identifier for each review.<br>* bundleID: Corresponding application bundle identifier.<br>* successful: Boolean status indicating the posting success. |
| Move spreadsheet to Archived folder | Google Drive | Archive processed spreadsheet | Download responses sheet | — | After posting all responses, move the spreadsheet to Archived folder |
| Sticky Note | Sticky Note | Workflow description & prerequisites | — | — |  |
| Sticky Note1 | Sticky Note | Morning step description | — | — |  |
| Sticky Note2 | Sticky Note | Human review step description | — | — |  |
| Sticky Note3 | Sticky Note | Afternoon step description | — | — |  |
| Sticky Note4 | Sticky Note | Skip loop hint | — | — |  |
| Sticky Note5 | Sticky Note | Spreadsheet aggregation hint | — | — |  |
| Sticky Note6 | Sticky Note | Upload hint | — | — |  |
| Sticky Note7 | Sticky Note | Search *ToSubmit* hint | — | — |  |
| Sticky Note8 | Sticky Note | Download hint | — | — |  |
| Sticky Note9 | Sticky Note | Fetch reviews hint | — | — |  |
| Sticky Note10 | Sticky Note | Posting hint | — | — |  |
| Sticky Note11 | Sticky Note | Archive intent | — | — |  |
| Sticky Note12 | Sticky Note | App table schema | — | — |  |
| Sticky Note13 | Sticky Note | Log table schema | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1) **Google API (Service Account)** credential in n8n  
      - Use a Google Cloud service account JSON key.  
      - Ensure the service account has access to:
        - Google Drive folders (*ToReview*, *ToSubmit*, *Archived*)
        - Google Play Android Publisher API for the relevant apps (grant access in Play Console).
   2) **Anthropic** credentials (for LangChain Anthropic Chat Model).

2. **Create Data Tables in n8n**
   1) Data Table: **Google Play apps**  
      - Columns: `bundle_id` (string), `name` (string).  
      - Add one row per app.
   2) Data Table: **Google Play review response logs**  
      - Columns: `reviewID` (string), `bundleID` (string), `successful` (boolean).

3. **Morning branch (10:00) — draft generation**
   1) Add node **Schedule Trigger** named “Trigger download”  
      - Set daily trigger at **10:00**.
   2) Add **Data Table** node “Fetch list of applications”  
      - Operation: **Get**, Return All: **true**, select table “Google Play apps”.
   3) Add **Set** node “Fetch app bundle id and name”  
      - Add fields:
        - `bundle_id = {{$json.bundle_id}}`
        - `name = {{$json.name}}`
   4) Add **Split In Batches** node “Loop over apps”.
   5) Add **HTTP Request** node “Fetch reviews from Google Play”  
      - Method: GET  
      - URL: `https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{{ $json.bundle_id }}/reviews`  
      - Auth: Predefined credential type → **Google API** (service account credential)  
      - Enable pagination using `nextPageToken`:
        - Request parameter name: `token`
        - Value from previous response: nextPageToken expression.
        - Stop when nextPageToken is null.
   6) Add **Split Out** node “Split out fetched reviews”  
      - Field to split: `reviews`.
   7) Add **Filter** node “Fetch yesterday's reviews”  
      - Condition: review lastModified date equals `{{$today.minus({days: 1})}}` using the review timestamp field.
      - Configure `onError` to continue if desired.
   8) Add **IF** node “Any reviews yesterday?”  
      - Condition: `reviewId` exists.
      - False branch should connect back to “Loop over apps” to continue.
   9) Add **Set** node “Fetch fields required by LLM”  
      - Map: `reviewId`, `userComment`, `starRating`, `reviewerLanguage`, plus app identifiers/name using item references.
   10) Add **Aggregate** node “Aggregate reviews for LLM use”  
       - Aggregate all item data → destination field `reviews`.
   11) Add **LangChain Anthropic Chat Model** node “Anthropic Chat Model”  
       - Select model (as desired) and set Anthropic credentials.
   12) Add **LangChain Structured Output Parser** node “Structured Output Parser”  
       - Define schema example as an array of objects containing: `bundleId`, `reviewId`, `userComment`, `starRating`, `response`.
   13) Add **LangChain LLM Chain** node “LLM Response Generator”  
       - Paste your community-manager prompt and include `{{ $json.toJsonString() }}` for input.
       - Attach “Anthropic Chat Model” to `ai_languageModel`.
       - Attach “Structured Output Parser” to `ai_outputParser`.
   14) Add **Split Out** node “Split responses”  
       - Field to split: `output`.
       - Connect its output back to “Loop over apps” (this is what advances the loop).
   15) Add **Filter** node “Get rid of empty items”  
       - Keep only items where `reviewId` exists.
   16) Add **Convert to File** node “Create the spreadsheet”  
       - Operation: `xlsx`  
       - Filename: `{{$now.format('yyyy-MM-dd-HH-mm-ss')}}_all_review_responses`
   17) Add **Google Drive** node “Upload spreadsheet”  
       - Operation: upload (default upload behavior)  
       - Drive: your target shared drive (or My Drive)  
       - Folder: **ToReview**  
       - Auth: service account.

4. **Human step**
   1) Ensure reviewers know to **edit responses** and **move the file** from *ToReview* to *ToSubmit*.
   2) Require that columns needed for posting remain present (at minimum: `bundle_id`, `reviewId`, `response` as configured in the posting node).

5. **Afternoon branch (17:00) — posting**
   1) Add node **Schedule Trigger** named “Trigger posting responses” at **17:00**.
   2) Add **Google Drive** node “Search responses ready to be posted”  
      - Search method: query  
      - Query: `'<ToSubmitFolderId>' in parents`  
      - Return: `id`, `name`.
   3) Add **Google Drive** node “Download responses sheet”  
      - Operation: download  
      - File ID: `{{$json.id}}`.
   4) Add **Extract From File** node “Extract responses”  
      - Operation: `xlsx`.
   5) Add **HTTP Request** node “Post responses”  
      - Method: POST  
      - URL: `https://www.googleapis.com/androidpublisher/v3/applications/{{ $json.bundle_id }}/reviews/{{$json.reviewId}}:reply`  
      - Body (raw JSON): `{"replyText":"{{ $json.response }}"}`  
      - Header: `Content-Type: application/json`  
      - Auth: Google API service account  
      - Enable “Continue on fail” behavior via error output if desired.
   6) Add **Data Table** node “Create a success log for the review”  
      - Write `reviewID`, `bundleID`, `successful=true` to “Google Play review response logs”.
   7) Add **Data Table** node “Create an error log for the review”  
      - Write `reviewID`, optionally `bundleID`, `successful=false`.
   8) Add **Google Drive** node “Move spreadsheet to Archived folder”  
      - Operation: move  
      - File ID: `{{$json.id}}`  
      - Destination: **Archived** folder.

   **Important:** To ensure the file is moved only after all rows are posted, connect archiving after posting completion using an explicit synchronization (Merge/Wait) rather than directly from the download node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Generate responses for Google Play Store reviews using Anthropic Claude, Google Drive and Google Play Store API … eliminates the need for costly third-party platforms like AppFollow or Appbot …” | Sticky Note (workflow description) |
| Pre-requisites: Google Drive/Sheets access, Google Play Developer account/service account, LLM credentials (Anthropic) | Sticky Note (pre-requisites) |
| Bundle ID example: `https://play.google.com/store/apps/details?id=com.anthropic.claude` → `com.anthropic.claude` | Sticky Note12 |
| Process design: 10am generate → human review move to *ToSubmit* → 5pm post → archive | Sticky Note + Sticky Note1/2/3 |
| Disclaimer: *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | Provided by user (disclaimer) |