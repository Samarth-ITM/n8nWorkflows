Generate AI UGC videos with HeyGen and post to Instagram and Facebook daily

https://n8nworkflows.xyz/workflows/generate-ai-ugc-videos-with-heygen-and-post-to-instagram-and-facebook-daily-14266


# Generate AI UGC videos with HeyGen and post to Instagram and Facebook daily

# 1. Workflow Overview

This workflow generates one short AI-driven UGC-style vertical video per day from a Google Sheets content library, renders it in HeyGen, then publishes it to Instagram and optionally Facebook. It also maintains production status in a Google Sheets log and records failures if video generation fails or times out.

The workflow is designed for recurring social content production with minimal manual effort. It rotates through eligible workbook ideas, uses OpenAI to write a short spoken script, sends that script to HeyGen for avatar-based video generation, polls HeyGen until the render is complete, and then posts the resulting video through the upload-post.com API.

## 1.1 Daily Trigger and Content Rotation
The workflow starts on a daily schedule, calculates the current day-of-year, loads all eligible content rows from Google Sheets, and selects one row deterministically using modulo arithmetic.

## 1.2 Avatar Selection and Script Generation
After choosing the daily workbook section, the workflow randomly assigns one avatar ID from a fixed list, sends the content to OpenAI, and parses the returned JSON script.

## 1.3 Production Log Initialization
Before video generation begins, the workflow appends a new row to a “Production Logs” sheet with the generated script and a status of `Generating`.

## 1.4 HeyGen Video Generation and Polling Loop
The workflow prepares a HeyGen request, submits the video generation job, stores the returned HeyGen video ID, then enters a polling loop. It repeatedly waits and checks the HeyGen render status until the job is complete, fails, or reaches a timeout threshold.

## 1.5 Video Finalization and Social Publishing
When the HeyGen render is complete, the workflow extracts the rendered video URL, stores it in the production log, marks the row complete, and posts the video to Instagram. A Facebook posting node is present but currently disabled.

## 1.6 Failure Handling
If HeyGen reports a failed/canceled status or the workflow exceeds its allowed polling attempts, the failure reason is written back to the production log. The workflow settings also reference a separate global error workflow for unhandled failures.

---

# 2. Block-by-Block Analysis

## 2.1 Daily Trigger and Content Selection

### Overview
This block launches the workflow each day and determines which content item should be used. It ensures that only workbook rows marked as `Idea` are considered, and it rotates through them based on the day of the year.

### Nodes Involved
- Schedule: Daily 9am
- Code: Calculate Content Index
- Google Sheets: Get Workbook Content
- Code: Select Today's Section

### Node Details

#### Schedule: Daily 9am
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Configured with an interval rule using `triggerAtHour: 12`. Despite the node name saying “Daily 9am”, the actual configured trigger hour in the JSON is 12. The effective runtime depends on the n8n instance timezone.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry node; outputs to `Code: Calculate Content Index`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Misleading node name vs actual configured hour.
  - Timezone differences can shift the real posting time.
  - If the workflow is inactive, no trigger occurs.
- **Sub-workflow reference:** None.

#### Code: Calculate Content Index
- **Type and role:** `Code` node; computes current day-of-year and formatted date.
- **Configuration choices:** JavaScript calculates:
  - `dayOfYear`
  - `date` as `YYYY-MM-DD`
- **Key expressions or variables used:**
  - `new Date()`
  - `now.toISOString().split('T')[0]`
- **Input and output connections:** Input from `Schedule: Daily 9am`; output to `Google Sheets: Get Workbook Content`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Uses server time, so timezone consistency matters.
  - Leap years will slightly alter annual rotation behavior.
- **Sub-workflow reference:** None.

#### Google Sheets: Get Workbook Content
- **Type and role:** `Google Sheets` node; fetches workbook content candidates.
- **Configuration choices:**
  - Reads from spreadsheet `AI Avatar - PinkMatcha`
  - Sheet: `Workbook Content` (`gid=0`)
  - Filter: only rows where `Status = Idea`
- **Key expressions or variables used:** None in the filter definition beyond static value `Idea`.
- **Input and output connections:** Input from `Code: Calculate Content Index`; output to `Code: Select Today's Section`.
- **Version-specific requirements:** Type version `4.4`; requires Google Sheets credentials with read access.
- **Edge cases / failures:**
  - Authentication/permission errors.
  - Empty result set if no rows have `Status = Idea`.
  - Column naming mismatches will prevent correct filtering.
- **Sub-workflow reference:** None.

#### Code: Select Today's Section
- **Type and role:** `Code` node; picks the current workbook row based on modulo rotation.
- **Configuration choices:**
  - Pulls `dayOfYear` and `date` from `Code: Calculate Content Index`
  - Reads all rows from `Google Sheets: Get Workbook Content`
  - Computes `sectionIndex = (dayOfYear - 1) % totalSections`
  - Returns:
    - `rowId`
    - `sectionTitle`
    - `workbookContent`
    - `keyMessage`
- **Key expressions or variables used:**
  - `$('Code: Calculate Content Index').first().json.dayOfYear`
  - `$('Google Sheets: Get Workbook Content').all()`
  - `selectedSection.row_id || selectedSection.row_number`
- **Input and output connections:** Input from `Google Sheets: Get Workbook Content`; output to `Code: Select Random Avatar & Combine Data`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Explicitly throws if the workbook content is empty.
  - Assumes sheet columns exist with names such as `section_title`, `workbook_content`, `key_message`.
  - `row_id` vs `row_number` inconsistency may cause later log update mismatches if source schema changes.
- **Sub-workflow reference:** None.

---

## 2.2 Avatar Selection and Script Generation

### Overview
This block enriches the selected content with a randomly chosen HeyGen avatar ID, asks OpenAI to write a short spoken script, and normalizes the AI output into a strict JSON structure.

### Nodes Involved
- Code: Select Random Avatar & Combine Data
- Generate Script
- Code: Parse AI Response

### Node Details

#### Code: Select Random Avatar & Combine Data
- **Type and role:** `Code` node; combines selected workbook data with one random avatar ID.
- **Configuration choices:**
  - Uses a hardcoded array of 14 avatar IDs
  - Randomly chooses one avatar for visual variation
  - Returns date, row ID, section title, workbook content, key message, avatar ID, and avatar index
- **Key expressions or variables used:**
  - `$('Code: Select Today\\'s Section').first().json`
  - `Math.floor(Math.random() * avatarIds.length)`
- **Input and output connections:** Input from `Code: Select Today's Section`; output to `Generate Script`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - If the avatar IDs become invalid in HeyGen, generation will later fail.
  - Random selection means identical content can produce different visuals on different days.
- **Sub-workflow reference:** None.

#### Generate Script
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi`; uses OpenAI to create the spoken script.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Sends:
    - `Section Title`
    - `Workbook Content`
    - `Key Message`
  - System prompt defines a spiritual guide persona: “The Shepherd”
  - Requires:
    - 75–90 words
    - roughly 30 seconds spoken
    - must end with: `Visit the link in bio to learn more.`
    - output must be exact JSON:
      `{ "script": "..." }`
- **Key expressions or variables used:**
  - `{{ $('Code: Select Random Avatar & Combine Data').first().json.sectionTitle }}`
  - `{{ $('Code: Select Random Avatar & Combine Data').first().json.workbookContent }}`
  - `{{ $('Code: Select Random Avatar & Combine Data').first().json.keyMessage }}`
- **Input and output connections:** Input from `Code: Select Random Avatar & Combine Data`; output to `Code: Parse AI Response`.
- **Version-specific requirements:** Type version `2.1`; requires OpenAI credentials and a model available to that account.
- **Edge cases / failures:**
  - OpenAI auth issues, quota limits, rate limiting.
  - Model may return markdown-wrapped JSON or non-JSON despite prompt constraints.
  - The script may exceed desired duration if model drifts.
- **Sub-workflow reference:** None.

#### Code: Parse AI Response
- **Type and role:** `Code` node; extracts and parses JSON containing the `script` field from potentially inconsistent AI responses.
- **Configuration choices:**
  - Tries multiple response shapes:
    - `response.message.content`
    - `response.choices[0].message.content`
    - `response.content[0].text`
    - recursive string search through nested objects
  - Attempts direct `JSON.parse`
  - Falls back to regex extraction of JSON-looking content
  - Validates presence of `parsed.script`
  - Combines parsed script with date, row ID, section title, and avatar ID
- **Key expressions or variables used:**
  - `$('Generate Script').first().json`
  - recursive `findJsonRecursive()`
  - `$('Code: Select Random Avatar & Combine Data').first().json`
- **Input and output connections:** Input from `Generate Script`; output to `Google Sheets: Create Production Log`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Throws if no string containing `"script"` is found.
  - Throws if JSON parsing fails.
  - Throws if parsed JSON has no `script` property.
  - Very malformed AI outputs may still fail despite fallback logic.
- **Sub-workflow reference:** None.

---

## 2.3 Production Log Initialization

### Overview
This block creates a production log row before video rendering begins and prepares the core fields needed for the HeyGen request.

### Nodes Involved
- Google Sheets: Create Production Log
- Code: Prepare HeyGen Request

### Node Details

#### Google Sheets: Create Production Log
- **Type and role:** `Google Sheets` node; appends a new production-tracking row.
- **Configuration choices:**
  - Spreadsheet: `AI Avatar - PinkMatcha`
  - Sheet: `Production Logs`
  - Operation: append
  - Sets:
    - `Date = $json.date`
    - `Section Title = $json.sectionTitle`
    - `Script Text = $json.script`
    - `Status = Generating`
- **Key expressions or variables used:**
  - `={{ $json.date }}`
  - `={{ $json.script }}`
  - `={{ $json.sectionTitle }}`
- **Input and output connections:** Input from `Code: Parse AI Response`; output to `Code: Prepare HeyGen Request`.
- **Version-specific requirements:** Type version `4.4`; requires Google Sheets write access.
- **Edge cases / failures:**
  - Append can fail if credentials are invalid or sheet structure changes.
  - If row numbering is not returned as expected, downstream updates may fail.
- **Sub-workflow reference:** None.

#### Code: Prepare HeyGen Request
- **Type and role:** `Code` node; packages the generated script and avatar for HeyGen submission.
- **Configuration choices:**
  - Reads row number from the newly appended log record
  - Returns:
    - `logRowNumber`
    - `avatarId`
    - `script`
    - `sectionTitle`
- **Key expressions or variables used:**
  - `$('Google Sheets: Create Production Log').first().json`
  - `$('Code: Parse AI Response').first().json`
  - `logRecord.row_number || logRecord.rowNumber`
- **Input and output connections:** Input from `Google Sheets: Create Production Log`; output to `HTTP: HeyGen Generate Video`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - If append response doesn’t expose a row number, later updates to the log may not work.
  - Relies on successful completion of both earlier nodes.
- **Sub-workflow reference:** None.

---

## 2.4 HeyGen Video Generation and Polling Loop

### Overview
This is the core asynchronous rendering block. It submits the video creation request to HeyGen, stores the HeyGen video ID, then waits and polls until the video is complete, failed, or timed out.

### Nodes Involved
- HTTP: HeyGen Generate Video
- Code: Extract HeyGen Video ID
- Google Sheets: Save HeyGen Video ID
- Wait: HeyGen Processing
- HTTP: Poll HeyGen Status
- Code: Check HeyGen Status
- Switch

### Node Details

#### HTTP: HeyGen Generate Video
- **Type and role:** `HTTP Request` node; creates a new HeyGen video generation job.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.heygen.com/v2/video/generate`
  - Body type: JSON
  - Timeout: 60 seconds
  - Uses generic header auth credentials
  - Payload includes:
    - `character.type = talking_photo`
    - `character.talking_photo_id = $json.avatarId`
    - `voice.type = text`
    - `voice.input_text = $json.script`
    - fixed `voice_id = ca320fd62b784352af74d06a16a6ef3d`
    - vertical format `720x1280`
    - `test = false`
    - `caption = false`
- **Key expressions or variables used:**
  - `$json.avatarId`
  - `$json.script`
  - JSON body assembled with `JSON.stringify(...)`
- **Input and output connections:** Input from `Code: Prepare HeyGen Request`; output to `Code: Extract HeyGen Video ID`.
- **Version-specific requirements:** Type version `4.2`; requires HTTP Header Auth credential containing valid HeyGen API authentication.
- **Edge cases / failures:**
  - Auth failure from HeyGen.
  - Invalid avatar ID or voice ID.
  - Request timeout or malformed body.
  - Script too long for the selected voice/video constraints.
- **Sub-workflow reference:** None.

#### Code: Extract HeyGen Video ID
- **Type and role:** `Code` node; extracts `video_id` from the HeyGen generation response and initializes poll tracking state.
- **Configuration choices:**
  - Reads HeyGen response
  - Reads prep data
  - Returns:
    - `logRowNumber`
    - `heygenVideoId`
    - `pollCount = 0`
    - `maxPolls = 20`
    - `pollIntervalSec = 30`
- **Key expressions or variables used:**
  - `$('HTTP: HeyGen Generate Video').first().json`
  - `$('Code: Prepare HeyGen Request').first().json`
  - `heygenResponse.data?.video_id || heygenResponse.video_id`
- **Input and output connections:** Input from `HTTP: HeyGen Generate Video`; output to `Google Sheets: Save HeyGen Video ID`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Throws if `video_id` is missing.
  - Note a data mapping inconsistency appears later: downstream polling URL expects `HeyGen Video ID`, but this node returns `heygenVideoId`.
- **Sub-workflow reference:** None.

#### Google Sheets: Save HeyGen Video ID
- **Type and role:** `Google Sheets` node; updates the production log row with the HeyGen video ID and marks status `Processing`.
- **Configuration choices:**
  - Operation: update
  - Match column: `row_number`
  - Writes:
    - `Status = Processing`
    - `row_number = $('Code: Parse AI Response').item.json.rowId`
    - `HeyGen Video ID = $json.heygenVideoId`
- **Key expressions or variables used:**
  - `={{ $('Code: Parse AI Response').item.json.rowId }}`
  - `={{ $json.heygenVideoId }}`
- **Input and output connections:** Input from `Code: Extract HeyGen Video ID`; output to `Wait: HeyGen Processing`.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Important logical risk: it uses `rowId` from the selected content sheet row, not `logRowNumber` from the appended production log row. If those are not the same row number in `Production Logs`, the wrong log row may be updated or the update may fail.
  - Sheet schema mismatch can break updates.
- **Sub-workflow reference:** None.

#### Wait: HeyGen Processing
- **Type and role:** `Wait` node; pauses before each poll.
- **Configuration choices:**
  - Wait amount: `50`
  - No unit is explicitly visible in the JSON snippet, but sticky note indicates polling every 50 seconds.
  - Has a generated webhook ID for resume handling.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - First reached from `Google Sheets: Save HeyGen Video ID`
  - Also reached from `Switch` retry branch
  - Outputs to `HTTP: Poll HeyGen Status`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Long waits increase total workflow runtime.
  - If execution data retention or wait handling is misconfigured in n8n, resumed executions may fail.
- **Sub-workflow reference:** None.

#### HTTP: Poll HeyGen Status
- **Type and role:** `HTTP Request` node; queries HeyGen for render progress.
- **Configuration choices:**
  - URL:
    `https://api.heygen.com/v1/video_status.get?video_id={{ $json['HeyGen Video ID'] }}`
  - Timeout: 30 seconds
  - Uses generic header auth
- **Key expressions or variables used:**
  - `{{ $json['HeyGen Video ID'] }}`
- **Input and output connections:** Input from `Wait: HeyGen Processing`; output to `Code: Check HeyGen Status`.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Critical mapping risk: current upstream data appears to carry `heygenVideoId`, not `HeyGen Video ID`. Unless the Google Sheets update node returns a field named exactly `HeyGen Video ID`, this request may call HeyGen with an empty video ID.
  - Auth errors and timeout risk.
  - HeyGen API version mismatch between generate endpoint (`v2`) and status endpoint (`v1`) may require validation against current API docs.
- **Sub-workflow reference:** None.

#### Code: Check HeyGen Status
- **Type and role:** `Code` node; interprets HeyGen status and decides whether to retry, continue, or fail.
- **Configuration choices:**
  - Reads poll response and retained state
  - Normalizes status to lowercase
  - Increments `pollCount`
  - Branches:
    - `completed` → `heygenRoute = completed`
    - `failed/error/canceled/cancelled` → `heygenRoute = failed`
    - `pollCount >= maxPolls` → timeout failure
    - otherwise → `heygenRoute = retry`
- **Key expressions or variables used:**
  - `$('HTTP: Poll HeyGen Status').first().json`
  - `$('Wait: HeyGen Processing').first().json || $('Code: Extract HeyGen Video ID').first().json`
  - `pollResponse.data?.status || pollResponse.status || 'unknown'`
- **Input and output connections:** Input from `HTTP: Poll HeyGen Status`; output to `Switch`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - If state is not preserved as expected, poll count logic can reset or behave unexpectedly.
  - Unknown statuses route to retry until timeout.
- **Sub-workflow reference:** None.

#### Switch
- **Type and role:** `Switch` node; routes polling outcomes.
- **Configuration choices:**
  - Branch 1: `heygenRoute = retry` → back to `Wait: HeyGen Processing`
  - Branch 2: `heygenRoute = completed` → `Code: Extract HeyGen Video URL`
  - Branch 3: `heygenRoute = failed` → `Google Sheets: Log Failure`
- **Key expressions or variables used:**
  - `={{ $json.heygenRoute }}`
- **Input and output connections:** Input from `Code: Check HeyGen Status`; outputs to:
  - `Wait: HeyGen Processing`
  - `Code: Extract HeyGen Video URL`
  - `Google Sheets: Log Failure`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If `heygenRoute` is missing, no branch may match.
  - Strict comparison is enabled.
- **Sub-workflow reference:** None.

---

## 2.5 Video Finalization and Log Completion

### Overview
Once HeyGen reports success, this block extracts the final video URL, stores intermediate and final URL data, and marks the production log row as complete.

### Nodes Involved
- Code: Extract HeyGen Video URL
- Google Sheets: Save Raw Video URL
- Code: Set Final Video URL
- Google Sheets: Update Complete

### Node Details

#### Code: Extract HeyGen Video URL
- **Type and role:** `Code` node; extracts the rendered video URL from HeyGen status output.
- **Configuration choices:**
  - Reads poll response and current input
  - Returns:
    - `logRowNumber`
    - `heygenVideoId`
    - `rawVideoUrl`
- **Key expressions or variables used:**
  - `$('HTTP: Poll HeyGen Status').first().json`
  - `$input.first().json`
  - `pollResponse.data?.video_url || pollResponse.video_url`
- **Input and output connections:** Input from `Switch` completed branch; output to `Google Sheets: Save Raw Video URL`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Throws if no `video_url` is present.
  - Depends on poll response structure matching expected schema.
- **Sub-workflow reference:** None.

#### Google Sheets: Save Raw Video URL
- **Type and role:** `Google Sheets` node; updates the production log with the HeyGen output URL and marks status `Finalizing`.
- **Configuration choices:**
  - Operation: update
  - Match column: `row_number`
  - Writes:
    - `Status = Finalizing`
    - `row_number = $('Code: Parse AI Response').item.json.rowId`
    - `Raw Video URL = $json.rawVideoUrl`
- **Key expressions or variables used:**
  - `={{ $('Code: Parse AI Response').item.json.rowId }}`
  - `={{ $json.rawVideoUrl }}`
- **Input and output connections:** Input from `Code: Extract HeyGen Video URL`; output to `Code: Set Final Video URL`.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Same row-number mismatch risk as earlier update nodes: it references source content row ID instead of appended production log row number.
- **Sub-workflow reference:** None.

#### Code: Set Final Video URL
- **Type and role:** `Code` node; currently sets the final video URL equal to the raw HeyGen URL.
- **Configuration choices:**
  - Reads from `Code: Extract HeyGen Video URL`
  - Returns:
    - `logRowNumber`
    - `finalVideoUrl = rawVideoUrl`
- **Key expressions or variables used:**
  - `$('Code: Extract HeyGen Video URL').first().json`
- **Input and output connections:** Input from `Google Sheets: Save Raw Video URL`; output to `Google Sheets: Update Complete`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - No transformation is applied; this is effectively a placeholder for future post-processing/CDN storage.
- **Sub-workflow reference:** None.

#### Google Sheets: Update Complete
- **Type and role:** `Google Sheets` node; writes the final video URL and marks the production row `Complete`.
- **Configuration choices:**
  - Operation: update
  - Match column: `row_number`
  - Writes:
    - `Status = Complete`
    - `row_number = $('Code: Parse AI Response').item.json.rowId`
    - `Final Video URL = $json.finalVideoUrl`
- **Key expressions or variables used:**
  - `={{ $('Code: Parse AI Response').item.json.rowId }}`
  - `={{ $json.finalVideoUrl }}`
- **Input and output connections:** Input from `Code: Set Final Video URL`; outputs to both `facebook` and `instagram`.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Same row mapping risk as above.
  - If social posting fails later, row remains `Complete` until `Google Sheets: Mark Posted` is reached.
- **Sub-workflow reference:** None.

---

## 2.6 Social Publishing

### Overview
This block submits the final rendered video to social posting endpoints. Instagram is active, Facebook is configured but disabled. A merge node waits for both branches before marking the log as posted.

### Nodes Involved
- facebook
- instagram
- Merge
- Google Sheets: Mark Posted

### Node Details

#### facebook
- **Type and role:** `HTTP Request` node; posts the final video to upload-post.com for Facebook publishing.
- **Configuration choices:**
  - Disabled: `true`
  - Method: `POST`
  - URL: `https://api.upload-post.com/api/upload`
  - Content type: `multipart-form-data`
  - Body fields:
    - `user = pink-matcha`
    - `platform[] = facebook`
    - `video = finalVideoUrl`
    - `title = sectionTitle substring + script substring + "... Link in Bio"`
  - Uses generic header auth
- **Key expressions or variables used:**
  - `$('Code: Set Final Video URL').first().json.finalVideoUrl`
  - `$('Code: Parse AI Response').first().json.sectionTitle.substring(0, 120)`
  - `$('Code: Parse AI Response').first().json.script.substring(0, 100)`
- **Input and output connections:** Input from `Google Sheets: Update Complete`; output to `Merge`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases / failures:**
  - Currently disabled, so the branch will not execute.
  - Depending on merge behavior, a disabled branch may affect whether downstream merge receives both expected inputs.
  - upload-post.com API auth or payload validation may fail.
- **Sub-workflow reference:** None.

#### instagram
- **Type and role:** `HTTP Request` node; posts the final video to upload-post.com for Instagram publishing.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.upload-post.com/api/upload`
  - Content type: `multipart-form-data`
  - Body fields:
    - `user = pink-matcha`
    - `platform[] = instagram`
    - `video = finalVideoUrl`
    - `title = sectionTitle substring + script substring + "... Link in Bio"`
  - Uses generic header auth
- **Key expressions or variables used:** Same caption logic as Facebook.
- **Input and output connections:** Input from `Google Sheets: Update Complete`; output to `Merge`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases / failures:**
  - Social API may reject direct remote URLs depending on service requirements.
  - Caption length/platform-specific formatting may need validation.
  - If upload-post.com expects actual file upload instead of URL, this can fail.
- **Sub-workflow reference:** None.

#### Merge
- **Type and role:** `Merge` node; combines social posting outputs before final status update.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input 1 from `facebook`
  - Input 2 from `instagram`
  - Output to `Google Sheets: Mark Posted`
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases / failures:**
  - Because `facebook` is disabled, the merge may not behave as intended depending on execution behavior and whether both inputs are required.
  - If one social branch fails, `Mark Posted` may never run.
- **Sub-workflow reference:** None.

#### Google Sheets: Mark Posted
- **Type and role:** `Google Sheets` node; marks the production log entry as `Posted`.
- **Configuration choices:**
  - Operation: update
  - Match column: `row_number`
  - Writes:
    - `Status = Posted`
    - `row_number = $('Code: Parse AI Response').item.json.rowId`
- **Key expressions or variables used:**
  - `={{ $('Code: Parse AI Response').item.json.rowId }}`
- **Input and output connections:** Input from `Merge`; no downstream outputs.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Same row-number mismatch risk.
  - If merge never fires due to disabled Facebook branch, this status may never be written.
- **Sub-workflow reference:** None.

---

## 2.7 Failure Handling

### Overview
This block captures HeyGen failures and timeout conditions and writes them into the production log. It is specifically tied to the polling loop outcome.

### Nodes Involved
- Google Sheets: Log Failure

### Node Details

#### Google Sheets: Log Failure
- **Type and role:** `Google Sheets` node; writes failure state back to the production log row.
- **Configuration choices:**
  - Operation: update
  - Match column: `row_number`
  - Writes:
    - `Status = $json.failReason || $json.failStatus`
    - `row_number = $json.row_number`
- **Key expressions or variables used:**
  - `={{ $json.failReason || $json.failStatus }}`
  - `={{ $json.row_number }}`
- **Input and output connections:** Input from `Switch` failed branch; no downstream outputs.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Likely data mapping problem: `Code: Check HeyGen Status` returns fields like `logRowNumber`, `failStatus`, `failReason`, but not clearly `row_number`. Unless a previous node output includes `row_number`, this update may not match any row.
  - Failure logging may silently miss the intended record if row mapping is wrong.
- **Sub-workflow reference:** None.

---

## 2.8 Documentation / Annotation Nodes

### Overview
These nodes are visual annotations inside the workflow canvas. They do not execute but provide operational context.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and role:** `Sticky Note`; top-level overview of the workflow.
- **Configuration choices:** Contains summary of end-to-end process and logging.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and role:** `Sticky Note`; explains content selection logic.
- **Configuration choices:** Notes modulo rotation and `Status = "Idea"` filter.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and role:** `Sticky Note`; explains script generation design and parsing tolerance.
- **Configuration choices:** Notes GPT-4.1-mini persona and JSON output expectation.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and role:** `Sticky Note`; explains HeyGen async polling loop.
- **Configuration choices:** Describes retry/completed/failed routing and timeout behavior.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and role:** `Sticky Note`; explains social publishing behavior.
- **Configuration choices:** Notes auto-generated caption and disabled Facebook node.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and role:** `Sticky Note`; explains failure logging and external error workflow.
- **Configuration choices:** Notes logging to sheet and existence of a configured error workflow.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** References global workflow setting `errorWorkflow = kohRSFWsjOdaD1r7`, but not as a callable sub-workflow node.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule: Daily 9am | Schedule Trigger | Starts the workflow daily |  | Code: Calculate Content Index | # Overall Workflow Overview  This workflow runs daily, picks a content section from Google Sheets, generates a 30-second spoken script using GPT-4.1-mini, renders a talking-head video via HeyGen, then publishes it to Facebook and Instagram. All steps are logged in the "Production Logs" sheet. |
| Code: Calculate Content Index | Code | Calculates day-of-year and current date | Schedule: Daily 9am | Google Sheets: Get Workbook Content | ## 1. Content Selection\nPicks today's content using day-of-year modulo total sections, so content rotates automatically without repeating. Only rows with Status = "Idea" are eligible. Add new rows to the "Workbook Content" sheet to expand the content pool. |
| Google Sheets: Get Workbook Content | Google Sheets | Reads eligible workbook ideas | Code: Calculate Content Index | Code: Select Today's Section | ## 1. Content Selection\nPicks today's content using day-of-year modulo total sections, so content rotates automatically without repeating. Only rows with Status = "Idea" are eligible. Add new rows to the "Workbook Content" sheet to expand the content pool. |
| Code: Select Today's Section | Code | Selects the current day’s workbook row | Google Sheets: Get Workbook Content | Code: Select Random Avatar & Combine Data | ## 1. Content Selection\nPicks today's content using day-of-year modulo total sections, so content rotates automatically without repeating. Only rows with Status = "Idea" are eligible. Add new rows to the "Workbook Content" sheet to expand the content pool. |
| Code: Select Random Avatar & Combine Data | Code | Adds a random HeyGen avatar to selected content | Code: Select Today's Section | Generate Script | ## 1. Content Selection\nPicks today's content using day-of-year modulo total sections, so content rotates automatically without repeating. Only rows with Status = "Idea" are eligible. Add new rows to the "Workbook Content" sheet to expand the content pool. |
| Generate Script | OpenAI | Generates short spoken script from workbook content | Code: Select Random Avatar & Combine Data | Code: Parse AI Response | ## 2. Script Generation\nUses GPT-4.1-mini with "The Shepherd" persona. Target: 75-90 words (~30 seconds spoken). Must end with "Visit the link in bio to learn more." Output is strict JSON { "script": "..." } — the parse node handles markdown-wrapped or malformed responses. |
| Code: Parse AI Response | Code | Extracts and validates the `script` field from AI output | Generate Script | Google Sheets: Create Production Log | ## 2. Script Generation\nUses GPT-4.1-mini with "The Shepherd" persona. Target: 75-90 words (~30 seconds spoken). Must end with "Visit the link in bio to learn more." Output is strict JSON { "script": "..." } — the parse node handles markdown-wrapped or malformed responses. |
| Google Sheets: Create Production Log | Google Sheets | Appends a new production log row | Code: Parse AI Response | Code: Prepare HeyGen Request | ## 2. Script Generation\nUses GPT-4.1-mini with "The Shepherd" persona. Target: 75-90 words (~30 seconds spoken). Must end with "Visit the link in bio to learn more." Output is strict JSON { "script": "..." } — the parse node handles markdown-wrapped or malformed responses. |
| Code: Prepare HeyGen Request | Code | Prepares data for HeyGen generation request | Google Sheets: Create Production Log | HTTP: HeyGen Generate Video | ## 2. Script Generation\nUses GPT-4.1-mini with "The Shepherd" persona. Target: 75-90 words (~30 seconds spoken). Must end with "Visit the link in bio to learn more." Output is strict JSON { "script": "..." } — the parse node handles markdown-wrapped or malformed responses. |
| HTTP: HeyGen Generate Video | HTTP Request | Submits HeyGen video generation job | Code: Prepare HeyGen Request | Code: Extract HeyGen Video ID | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Code: Extract HeyGen Video ID | Code | Extracts HeyGen video ID and initializes polling state | HTTP: HeyGen Generate Video | Google Sheets: Save HeyGen Video ID | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Google Sheets: Save HeyGen Video ID | Google Sheets | Updates log with HeyGen video ID and processing status | Code: Extract HeyGen Video ID | Wait: HeyGen Processing | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Wait: HeyGen Processing | Wait | Delays before the next HeyGen status poll | Google Sheets: Save HeyGen Video ID; Switch | HTTP: Poll HeyGen Status | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| HTTP: Poll HeyGen Status | HTTP Request | Queries HeyGen for render status | Wait: HeyGen Processing | Code: Check HeyGen Status | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Code: Check HeyGen Status | Code | Decides retry/completed/failed route | HTTP: Poll HeyGen Status | Switch | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Switch | Switch | Routes HeyGen polling outcomes | Code: Check HeyGen Status | Wait: HeyGen Processing; Code: Extract HeyGen Video URL; Google Sheets: Log Failure | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Code: Extract HeyGen Video URL | Code | Extracts rendered video URL from HeyGen response | Switch | Google Sheets: Save Raw Video URL | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Google Sheets: Save Raw Video URL | Google Sheets | Updates log with raw video URL | Code: Extract HeyGen Video URL | Code: Set Final Video URL | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Code: Set Final Video URL | Code | Sets final video URL from raw HeyGen URL | Google Sheets: Save Raw Video URL | Google Sheets: Update Complete | ## 3. HeyGen Polling Loop\nHeyGen video generation is async. This loop polls every 50 seconds, up to 20 times (~16 min max). Switch routes to: retry (still processing), completed (extract URL), or failed (log error). If it times out, it also routes to failed. |
| Google Sheets: Update Complete | Google Sheets | Marks log complete and saves final video URL | Code: Set Final Video URL | facebook; instagram | ## 4. Social Publishing\nPosts to Facebook and Instagram via upload-post.com API. The caption is auto-generated from the section title + first 100 chars of script + "... Link in Bio". Note: Facebook node is currently disabled — enable it when ready to go live on Facebook. |
| facebook | HTTP Request | Sends video to upload-post.com for Facebook posting | Google Sheets: Update Complete | Merge | ## 4. Social Publishing\nPosts to Facebook and Instagram via upload-post.com API. The caption is auto-generated from the section title + first 100 chars of script + "... Link in Bio". Note: Facebook node is currently disabled — enable it when ready to go live on Facebook. |
| instagram | HTTP Request | Sends video to upload-post.com for Instagram posting | Google Sheets: Update Complete | Merge | ## 4. Social Publishing\nPosts to Facebook and Instagram via upload-post.com API. The caption is auto-generated from the section title + first 100 chars of script + "... Link in Bio". Note: Facebook node is currently disabled — enable it when ready to go live on Facebook. |
| Merge | Merge | Waits for social posting branch outputs | facebook; instagram | Google Sheets: Mark Posted | ## 4. Social Publishing\nPosts to Facebook and Instagram via upload-post.com API. The caption is auto-generated from the section title + first 100 chars of script + "... Link in Bio". Note: Facebook node is currently disabled — enable it when ready to go live on Facebook. |
| Google Sheets: Mark Posted | Google Sheets | Marks production log as posted | Merge |  | ## 4. Social Publishing\nPosts to Facebook and Instagram via upload-post.com API. The caption is auto-generated from the section title + first 100 chars of script + "... Link in Bio". Note: Facebook node is currently disabled — enable it when ready to go live on Facebook. |
| Google Sheets: Log Failure | Google Sheets | Writes HeyGen failure or timeout status to log | Switch |  | ## 5. Error Handling\nIf HeyGen fails or times out, the failure reason and status are written back to the Production Logs sheet. The workflow also has an error workflow configured for unexpected crashes. |
| Sticky Note | Sticky Note | Canvas annotation: overall workflow description |  |  |  |
| Sticky Note1 | Sticky Note | Canvas annotation: content selection notes |  |  |  |
| Sticky Note2 | Sticky Note | Canvas annotation: script generation notes |  |  |  |
| Sticky Note3 | Sticky Note | Canvas annotation: HeyGen polling notes |  |  |  |
| Sticky Note4 | Sticky Note | Canvas annotation: social publishing notes |  |  |  |
| Sticky Note5 | Sticky Note | Canvas annotation: error handling notes |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Heygen Viral UGC generation`.
   - Activate it only after all credentials and mappings are verified.

2. **Add a Schedule Trigger node**
   - Type: `Schedule Trigger`
   - Name: `Schedule: Daily 9am`
   - Configure a daily schedule.
   - Important: the provided workflow actually uses hour `12`, even though the name says 9am. Decide your intended hour and match both the node label and config.
   - Set your n8n instance timezone correctly.

3. **Add a Code node to calculate the content index**
   - Name: `Code: Calculate Content Index`
   - Paste logic that:
     - calculates `dayOfYear`
     - outputs `date` as `YYYY-MM-DD`
   - Connect `Schedule: Daily 9am -> Code: Calculate Content Index`.

4. **Add a Google Sheets node to fetch workbook content**
   - Name: `Google Sheets: Get Workbook Content`
   - Type: `Google Sheets`
   - Credential: Google Sheets OAuth2 or service account with access to the spreadsheet.
   - Spreadsheet: `AI Avatar - PinkMatcha`
   - Sheet: `Workbook Content`
   - Operation: read/get rows
   - Add a filter where `Status = Idea`
   - Connect `Code: Calculate Content Index -> Google Sheets: Get Workbook Content`.

5. **Add a Code node to select today’s section**
   - Name: `Code: Select Today's Section`
   - Logic:
     - read `dayOfYear` and `date` from previous code node
     - read all eligible rows from Google Sheets
     - throw an error if there are no rows
     - compute `(dayOfYear - 1) % totalSections`
     - output:
       - `date`
       - `dayOfYear`
       - `sectionIndex`
       - `totalSections`
       - `rowId`
       - `sectionTitle`
       - `workbookContent`
       - `keyMessage`
   - Connect `Google Sheets: Get Workbook Content -> Code: Select Today's Section`.

6. **Add a Code node for avatar randomization**
   - Name: `Code: Select Random Avatar & Combine Data`
   - Paste a fixed array of HeyGen talking photo IDs.
   - Randomly choose one using `Math.random()`.
   - Output:
     - `date`
     - `rowId`
     - `sectionTitle`
     - `workbookContent`
     - `keyMessage`
     - `avatarId`
     - `avatarIndex`
   - Connect `Code: Select Today's Section -> Code: Select Random Avatar & Combine Data`.

7. **Add an OpenAI node for script generation**
   - Name: `Generate Script`
   - Type: OpenAI / LangChain OpenAI node
   - Credential: OpenAI API credential
   - Model: `gpt-4.1-mini`
   - Configure messages:
     - **User/content message** includes section title, workbook content, and key message from the previous node.
     - **System message** defines the “The Shepherd” persona and requires:
       - 75–90 words
       - spoken-only copy
       - no stage directions
       - must end with `Visit the link in bio to learn more.`
       - strict JSON output:
         `{ "script": "..." }`
   - Connect `Code: Select Random Avatar & Combine Data -> Generate Script`.

8. **Add a Code node to parse the AI response**
   - Name: `Code: Parse AI Response`
   - Implement robust parsing logic:
     - try common OpenAI response locations
     - recursively search nested properties for a string containing `"script"`
     - parse direct JSON or extract JSON substring with regex
     - validate presence of `script`
   - Output:
     - `date`
     - `rowId`
     - `sectionTitle`
     - `avatarId`
     - `script`
   - Connect `Generate Script -> Code: Parse AI Response`.

9. **Add a Google Sheets node to append a production log entry**
   - Name: `Google Sheets: Create Production Log`
   - Spreadsheet: `AI Avatar - PinkMatcha`
   - Sheet: `Production Logs`
   - Operation: append row
   - Map columns:
     - `Date = {{$json.date}}`
     - `Section Title = {{$json.sectionTitle}}`
     - `Script Text = {{$json.script}}`
     - `Status = Generating`
   - Connect `Code: Parse AI Response -> Google Sheets: Create Production Log`.

10. **Add a Code node to prepare the HeyGen request**
    - Name: `Code: Prepare HeyGen Request`
    - Read:
      - row number from `Google Sheets: Create Production Log`
      - `avatarId`, `script`, `sectionTitle` from `Code: Parse AI Response`
    - Output:
      - `logRowNumber`
      - `avatarId`
      - `script`
      - `sectionTitle`
    - Connect `Google Sheets: Create Production Log -> Code: Prepare HeyGen Request`.

11. **Add an HTTP Request node for HeyGen video generation**
    - Name: `HTTP: HeyGen Generate Video`
    - Credential: HTTP Header Auth for HeyGen API
    - Method: `POST`
    - URL: `https://api.heygen.com/v2/video/generate`
    - Timeout: `60000`
    - Send JSON body:
      - one `video_inputs` item
      - `character.type = talking_photo`
      - `character.talking_photo_id = {{$json.avatarId}}`
      - `voice.type = text`
      - `voice.input_text = {{$json.script}}`
      - use fixed `voice_id = ca320fd62b784352af74d06a16a6ef3d`
      - `dimension.width = 720`
      - `dimension.height = 1280`
      - `test = false`
      - `caption = false`
    - Connect `Code: Prepare HeyGen Request -> HTTP: HeyGen Generate Video`.

12. **Add a Code node to extract the HeyGen video ID**
    - Name: `Code: Extract HeyGen Video ID`
    - Parse `video_id` from the response.
    - Throw an error if missing.
    - Output:
      - `logRowNumber`
      - `heygenVideoId`
      - `pollCount = 0`
      - `maxPolls = 20`
      - `pollIntervalSec = 30`
    - Connect `HTTP: HeyGen Generate Video -> Code: Extract HeyGen Video ID`.

13. **Add a Google Sheets node to save the HeyGen video ID**
    - Name: `Google Sheets: Save HeyGen Video ID`
    - Sheet: `Production Logs`
    - Operation: update
    - Match column: `row_number`
    - Recommended mapping:
      - `row_number = {{$json.logRowNumber}}`
      - `HeyGen Video ID = {{$json.heygenVideoId}}`
      - `Status = Processing`
    - Note: the provided workflow uses `rowId` from the source sheet, which is risky. Use `logRowNumber` instead for a reliable rebuild.
    - Connect `Code: Extract HeyGen Video ID -> Google Sheets: Save HeyGen Video ID`.

14. **Add a Wait node**
    - Name: `Wait: HeyGen Processing`
    - Configure a delay of about 50 seconds.
    - Connect `Google Sheets: Save HeyGen Video ID -> Wait: HeyGen Processing`.

15. **Add an HTTP Request node to poll HeyGen status**
    - Name: `HTTP: Poll HeyGen Status`
    - Credential: same HeyGen HTTP Header Auth
    - Method: `GET`
    - URL should use the HeyGen video ID from incoming JSON.
    - Recommended expression:
      - `https://api.heygen.com/v1/video_status.get?video_id={{ $json.heygenVideoId }}`
    - Timeout: `30000`
    - Note: the original workflow references `$json['HeyGen Video ID']`, which can break if that exact field is not present. Prefer `heygenVideoId`.
    - Connect `Wait: HeyGen Processing -> HTTP: Poll HeyGen Status`.

16. **Add a Code node to interpret HeyGen status**
    - Name: `Code: Check HeyGen Status`
    - Logic:
      - read poll response
      - recover prior state from wait input or initial extraction node
      - normalize the returned status
      - increment `pollCount`
      - if complete, set `heygenRoute = completed`
      - if failed/canceled/error, set `heygenRoute = failed` plus `failStatus` and `failReason`
      - if max polls exceeded, set timeout failure
      - else set `heygenRoute = retry`
    - Connect `HTTP: Poll HeyGen Status -> Code: Check HeyGen Status`.

17. **Add a Switch node**
    - Name: `Switch`
    - Create 3 rules on `{{$json.heygenRoute}}`:
      1. equals `retry`
      2. equals `completed`
      3. equals `failed`
    - Connect:
      - retry output -> `Wait: HeyGen Processing`
      - completed output -> `Code: Extract HeyGen Video URL`
      - failed output -> `Google Sheets: Log Failure`

18. **Create the polling loop**
    - Connect `Switch` retry branch back to `Wait: HeyGen Processing`.

19. **Add a Code node to extract the final HeyGen video URL**
    - Name: `Code: Extract HeyGen Video URL`
    - Read `video_url` from the poll response.
    - Throw an error if missing.
    - Output:
      - `logRowNumber`
      - `heygenVideoId`
      - `rawVideoUrl`
    - Connect `Switch` completed branch -> `Code: Extract HeyGen Video URL`.

20. **Add a Google Sheets node to save the raw video URL**
    - Name: `Google Sheets: Save Raw Video URL`
    - Sheet: `Production Logs`
    - Operation: update
    - Match column: `row_number`
    - Recommended mapping:
      - `row_number = {{$json.logRowNumber}}`
      - `Raw Video URL = {{$json.rawVideoUrl}}`
      - `Status = Finalizing`
    - Connect `Code: Extract HeyGen Video URL -> Google Sheets: Save Raw Video URL`.

21. **Add a Code node to set the final video URL**
    - Name: `Code: Set Final Video URL`
    - For now, simply set:
      - `finalVideoUrl = rawVideoUrl`
    - Output:
      - `logRowNumber`
      - `finalVideoUrl`
    - Connect `Google Sheets: Save Raw Video URL -> Code: Set Final Video URL`.

22. **Add a Google Sheets node to mark the video complete**
    - Name: `Google Sheets: Update Complete`
    - Sheet: `Production Logs`
    - Operation: update
    - Match column: `row_number`
    - Recommended mapping:
      - `row_number = {{$json.logRowNumber}}`
      - `Final Video URL = {{$json.finalVideoUrl}}`
      - `Status = Complete`
    - Connect `Code: Set Final Video URL -> Google Sheets: Update Complete`.

23. **Add the Instagram publishing node**
    - Name: `instagram`
    - Type: `HTTP Request`
    - Credential: HTTP Header Auth for upload-post.com
    - Method: `POST`
    - URL: `https://api.upload-post.com/api/upload`
    - Body type: `multipart/form-data`
    - Body parameters:
      - `user = pink-matcha`
      - `platform[] = instagram`
      - `video = {{ $('Code: Set Final Video URL').first().json.finalVideoUrl }}`
      - `title = {{ $('Code: Parse AI Response').first().json.sectionTitle.substring(0, 120) + ' - ' + $('Code: Parse AI Response').first().json.script.substring(0, 100) + '... Link in Bio' }}`
    - Connect `Google Sheets: Update Complete -> instagram`.

24. **Add the Facebook publishing node**
    - Name: `facebook`
    - Same configuration as Instagram except:
      - `platform[] = facebook`
    - Keep it disabled if you do not want live Facebook publishing yet.
    - Connect `Google Sheets: Update Complete -> facebook`.

25. **Add a Merge node**
    - Name: `Merge`
    - Mode: `combine`
    - Combine by: `position`
    - Connect:
      - `facebook -> Merge` input 1
      - `instagram -> Merge` input 2
    - Important: if Facebook remains disabled, consider replacing the merge with a simpler path or configure branch logic that does not require both inputs.

26. **Add a Google Sheets node to mark posting complete**
    - Name: `Google Sheets: Mark Posted`
    - Sheet: `Production Logs`
    - Operation: update
    - Match column: `row_number`
    - Recommended mapping:
      - `row_number = {{$json.logRowNumber}}` or carry it forward explicitly
      - `Status = Posted`
    - Connect `Merge -> Google Sheets: Mark Posted`.

27. **Add the failure logging node**
    - Name: `Google Sheets: Log Failure`
    - Sheet: `Production Logs`
    - Operation: update
    - Match column: `row_number`
    - Recommended mapping:
      - `row_number = {{$json.logRowNumber}}`
      - `Status = {{$json.failReason || $json.failStatus}}`
    - Connect `Switch` failed branch -> `Google Sheets: Log Failure`.

28. **Create the Google Sheets spreadsheet structure**
    - Spreadsheet: `AI Avatar - PinkMatcha`
    - Sheet 1: `Workbook Content`
      - Suggested columns:
        - `section_title`
        - `workbook_content`
        - `key_message`
        - `Status`
        - optionally `row_id`
    - Sheet 2: `Production Logs`
      - Columns used by the workflow:
        - `Date`
        - `Section Title`
        - `Script Text`
        - `HeyGen Video ID`
        - `Raw Video URL`
        - `Final Video URL`
        - `Status`
        - `row_number` if returned/maintained by the node

29. **Configure credentials**
    - **Google Sheets credential**
      - Must allow read/write access to the spreadsheet.
    - **OpenAI credential**
      - Must permit access to `gpt-4.1-mini`.
    - **HeyGen credential**
      - Configure as HTTP Header Auth.
      - Usually send API key in the header required by HeyGen.
    - **upload-post.com credential**
      - Configure as HTTP Header Auth according to the service’s API spec.

30. **Add canvas notes if desired**
    - Create sticky notes for:
      - overall workflow purpose
      - content selection logic
      - script generation rules
      - HeyGen polling loop
      - social publishing notes
      - error handling notes

31. **Validate the workflow with test data**
    - Test the Google Sheets fetch first.
    - Test OpenAI output shape and parsing.
    - Test HeyGen generation and confirm `video_id` and `video_url` fields.
    - Test a single social platform before enabling both.
    - Verify log updates hit the correct `Production Logs` row.

32. **Recommended fixes before production**
    - Use `logRowNumber` consistently for all updates to `Production Logs`.
    - Use `heygenVideoId` consistently in polling.
    - Revisit the merge design if Facebook remains disabled.
    - Align actual schedule time with node label.
    - Optionally add explicit success checks on upload-post.com responses before marking `Posted`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Overall purpose: runs daily, selects a content section from Google Sheets, generates a short script with GPT-4.1-mini, renders a talking-head video with HeyGen, and logs all steps in the Production Logs sheet. | Workflow canvas note |
| Content selection rotates automatically using day-of-year modulo number of eligible sections. Only rows with `Status = "Idea"` are used. | Workflow canvas note |
| Script style uses “The Shepherd” persona and targets 75–90 words, around 30 seconds spoken, ending with “Visit the link in bio to learn more.” | Workflow canvas note |
| HeyGen generation is asynchronous and is polled in a loop until retry, completion, or failure. | Workflow canvas note |
| Social publishing uses upload-post.com. Facebook posting is configured but currently disabled. | Workflow canvas note |
| Failure status and reason are written back to the `Production Logs` sheet. | Workflow canvas note |
| The workflow references a separate global error workflow for unexpected crashes. | Workflow setting: `errorWorkflow = kohRSFWsjOdaD1r7` |
| Workflow description: “Generate viral videos using Heygen and autopublish them directly to facebook and instagram” | Workflow metadata |

## Important implementation observations
- The current JSON contains likely field-mapping inconsistencies between:
  - `rowId` vs `logRowNumber`
  - `heygenVideoId` vs `HeyGen Video ID`
- The current merge design may block `Google Sheets: Mark Posted` if Facebook stays disabled.
- The schedule node label and actual configured hour do not match.