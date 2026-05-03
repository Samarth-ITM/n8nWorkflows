Generate and schedule social media posts with OpenAI, Google Sheets and Buffer

https://n8nworkflows.xyz/workflows/generate-and-schedule-social-media-posts-with-openai--google-sheets-and-buffer-13265


# Generate and schedule social media posts with OpenAI, Google Sheets and Buffer

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
Runs every day at **06:00 UTC**, fetches “pending” rows from a Google Sheet, generates platform-specific social posts using OpenAI (LinkedIn/Twitter/Facebook), writes the generated content back to the sheet, schedules posts via the **Buffer API**, then notifies via **Slack**. If the workflow errors, it logs the failure to the sheet and sends a Slack alert.

**Target use cases:**
- Content teams batching scheduled social posts from a planning sheet
- Automating multi-platform copy generation and scheduling
- Basic operational monitoring (success summary + failure alerts)

### 1.1 Trigger & Data Fetch
- Cron trigger at 06:00 UTC
- Read rows from Google Sheets where `status = "pending"`

### 1.2 Validation & Control Flow
- Ensures required fields exist before processing
- If missing/none: branch to a NoOp “No Pending Content”
- Otherwise process items sequentially using Split in Batches

### 1.3 AI Content Generation (Sequential)
- Generate LinkedIn post → Twitter post → Facebook post
- Each node uses `gpt-4o-mini` with platform-specific constraints

### 1.4 Persist Generated Content & Schedule via Buffer
- Write generated posts back to the sheet (`status = generated`)
- Create Buffer scheduled updates for LinkedIn, Twitter, Facebook
- Mark row as `scheduled`

### 1.5 Notifications & Error Handling
- Success summary to Slack after batch completion
- Global error trigger path: prepare error payload → update sheet → Slack alert

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Fetch Pending Rows
**Overview:**  
Starts daily at 06:00 UTC and retrieves all Google Sheet rows marked `pending`.

**Nodes involved:**
- Daily 6AM Trigger
- Get Pending Content

#### Node: Daily 6AM Trigger
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) – workflow entry point.
- **Config:** Cron expression `0 6 * * *` (daily at 06:00).
- **Outputs:** Sends one trigger execution to **Get Pending Content**.
- **Edge cases/failures:** None typical; time zone is effectively server/UTC as configured—post scheduling assumes UTC later.

#### Node: Get Pending Content
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) – reads data.
- **Config choices:**
- Reads from `documentId` set by variable: `{{ $vars.GOOGLE_SHEET_URL ?? '' }}`
- Sheet tab: `gid=0`
- Filter UI: `status` column equals `pending`
- **Retries:** `maxTries: 3`, `waitBetweenTries: 2000ms`, `retryOnFail: true`
- **Output:** Items representing rows that match the filter, forwarded to **Validate Required Fields**.
- **Edge cases/failures:**
  - Missing/empty `GOOGLE_SHEET_URL` variable → invalid document reference
  - OAuth permission errors / sheet not shared
  - Filter depends on column name `status` existing exactly
  - Large sheets may hit API quotas/time

---

### Block 2 — Validation & Batch Processing Control
**Overview:**  
Ensures each row contains required fields and then iterates rows one by one.

**Nodes involved:**
- Validate Required Fields
- No Pending Content
- Process Each Item

#### Node: Validate Required Fields
- **Type / role:** IF (`n8n-nodes-base.if`) – gates processing.
- **Config:** “AND” conditions (all must be not empty):
  - `content_id`
  - `topic`
  - `key_points`
  - `schedule_date`
  - `schedule_time`
- **Connections:**
  - **True** → **Process Each Item**
  - **False** → **No Pending Content**
- **Important behavior:** In n8n IF node, evaluation happens per item; items failing validation go to the false branch while valid ones go true.
- **Edge cases/failures:**
  - If some rows are valid and some invalid, the false branch will receive the invalid ones; however it currently goes to a NoOp only (no logging). You may silently drop invalid rows.
  - `schedule_date` / `schedule_time` are validated only as non-empty strings, not as valid date/time formats.

#### Node: No Pending Content
- **Type / role:** No Operation (`n8n-nodes-base.noOp`) – placeholder end.
- **Config:** None.
- **Inputs:** Receives items that fail validation (and also potentially “no rows” scenario, depending on how Sheets returns).
- **Edge cases:** This node does not notify; “no pending content” is not explicitly messaged.

#### Node: Process Each Item
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) – sequential iteration.
- **Config choices:**
  - `reset: false` (keeps state within execution; typical for looping)
- **Connections:**
  - Main output 0 → **Generate LinkedIn Post** (batch loop path)
  - Main output 1 → **Send Success Summary** (called when no items left)
  - After scheduling is done, **Update Sheet - Scheduled** connects back into **Process Each Item** to request the next batch.
- **Edge cases/failures:**
  - If an error occurs but node-level `onError: continueErrorOutput` is set downstream, the loop may continue and still end with “success” summary even if some items failed to generate/schedule.
  - The batch size is not explicitly set here; default behavior is typically 1 item per batch in the UI unless configured otherwise.

---

### Block 3 — AI Content Generation (LinkedIn → Twitter → Facebook)
**Overview:**  
For each content row, generates three platform-specific messages using OpenAI, chaining calls sequentially.

**Nodes involved:**
- Generate LinkedIn Post
- Generate Twitter Post
- Generate Facebook Post
- Collect All Platform Posts

#### Node: Generate LinkedIn Post
- **Type / role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) – text generation.
- **Config choices:**
  - Model: `gpt-4o-mini`
  - Max tokens: 500; Temperature: 0.7
  - Prompt includes formatting rules (hook, short paragraphs, hashtags, CTA, <1300 chars, 1–3 emojis)
  - Tone expression: `{{ $('Process Each Item').item.json.tone ?? 'professional' }}`
  - Topic/key points/target audience pulled from the current batch item via `$('Process Each Item').item.json...`
- **Error handling:** `onError: continueErrorOutput`, plus retries (`maxTries: 3`, wait 3000ms).
- **Connections:** Output → **Generate Twitter Post**
- **Edge cases/failures:**
  - If OpenAI returns an error, downstream Set node falls back to `GENERATION_FAILED`
  - Character-limit compliance is “best effort” via prompt only (not programmatically enforced)

#### Node: Generate Twitter Post
- **Type / role:** OpenAI generation.
- **Config choices:**
  - Model: `gpt-4o-mini`
  - Max tokens: 100; Temperature: 0.8
  - Prompt enforces 280 characters “STRICT LIMIT”
  - Uses same `tone/topic/key_points/target_audience` pattern from the batch item
- **Error handling:** `onError: continueErrorOutput`, retries enabled.
- **Connections:** Output → **Generate Facebook Post**
- **Edge cases/failures:** Same; 280-character enforcement is not validated in code.

#### Node: Generate Facebook Post
- **Type / role:** OpenAI generation.
- **Config choices:**
  - Model: `gpt-4o-mini`
  - Max tokens: 300; Temperature: 0.7
  - Prompt: friendly, question, 1–2 emojis, under 500 chars
  - Tone defaults to `casual` if missing: `{{ ...tone ?? 'casual' }}`
- **Error handling:** `onError: continueErrorOutput`, retries enabled.
- **Connections:** Output → **Collect All Platform Posts**
- **Edge cases/failures:** Same; length not programmatically checked.

#### Node: Collect All Platform Posts
- **Type / role:** Set (`n8n-nodes-base.set`) – builds a normalized object for downstream updates/scheduling.
- **Config choices (key fields):**
  - `linkedin_post`: `{{ $('Generate LinkedIn Post').first().json.text ?? 'GENERATION_FAILED' }}`
  - `twitter_post`: `{{ $('Generate Twitter Post').first().json.text ?? 'GENERATION_FAILED' }}`
  - `facebook_post`: `{{ $('Generate Facebook Post').first().json.text ?? 'GENERATION_FAILED' }}`
  - Copies from batch item: `content_id`, `topic`, `platforms`, `schedule_date`, `schedule_time`, `row_number`
  - Sets `status = "generated"`
- **Connections:** Output → **Update Sheet - Content Generated**
- **Edge cases/failures:**
  - Assumes OpenAI node output field is `json.text`. If the node version/response shape differs, `.json.text` may be undefined and trigger fallback.
  - `platforms` is collected but **not used** later to conditionally schedule; all three scheduling calls always run.

---

### Block 4 — Writeback & Buffer Scheduling
**Overview:**  
Updates the sheet with generated content, schedules three Buffer updates, then marks the row as scheduled.

**Nodes involved:**
- Update Sheet - Content Generated
- Schedule LinkedIn to Buffer
- Schedule Twitter to Buffer
- Schedule Facebook to Buffer
- Update Sheet - Scheduled

#### Node: Update Sheet - Content Generated
- **Type / role:** Google Sheets update.
- **Config choices:**
  - Operation: `update`
  - Matches row by `content_id` (matchingColumns: `content_id`)
  - Writes: `status=generated`, plus `twitter_post`, `facebook_post`, `linkedin_post`
  - Document URL from `GOOGLE_SHEET_URL`, sheet `gid=0`
- **Retries:** enabled (3 tries, 2s wait).
- **Connections:** Output → **Schedule LinkedIn to Buffer**
- **Edge cases/failures:**
  - If `content_id` is not unique, multiple rows could be updated unexpectedly (Sheets node behavior depends on implementation).
  - If the sheet column names don’t match exactly, fields won’t update.

#### Node: Schedule LinkedIn to Buffer
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) – Buffer API call.
- **Config choices:**
  - POST `https://api.bufferapp.com/1/updates/create.json`
  - Auth: Generic credential type → HTTP Header Auth (token in header)
  - Body parameters:
    - `text = {{ $json.linkedin_post }}`
    - `profile_ids[] = {{ $vars.BUFFER_LINKEDIN_PROFILE_ID ?? '' }}`
    - `scheduled_at = {{ $json.schedule_date }}T{{ $json.schedule_time }}:00Z`
  - Timeout: 30s
- **Error handling:** `onError: continueErrorOutput`, retries enabled (3 tries, 5s wait).
- **Connections:** Output → **Schedule Twitter to Buffer**
- **Edge cases/failures:**
  - Empty profile ID variable will yield Buffer API error
  - `scheduled_at` is forced to UTC (`Z`). If your intended schedule is local time, posts will shift.
  - If `schedule_time` already contains seconds or timezone, concatenation may create invalid ISO datetime.

#### Node: Schedule Twitter to Buffer
- **Type / role:** HTTP Request to Buffer API.
- **Config:** Same endpoint and auth. Uses expressions from **Collect All Platform Posts**:
  - `text = {{ $('Collect All Platform Posts').first().json.twitter_post }}`
  - `profile_ids[] = {{ $vars.BUFFER_TWITTER_PROFILE_ID ?? '' }}`
  - `scheduled_at = {{ ...schedule_date }}T{{ ...schedule_time }}:00Z`
- **Error handling:** `onError: continueErrorOutput`, retries enabled.
- **Connections:** Output → **Schedule Facebook to Buffer**
- **Edge cases:** Same as LinkedIn.

#### Node: Schedule Facebook to Buffer
- **Type / role:** HTTP Request to Buffer API.
- **Config:** Same pattern:
  - `text` from `facebook_post`
  - `profile_ids[]` from `BUFFER_FACEBOOK_PROFILE_ID`
  - `scheduled_at` same build
- **Error handling:** `onError: continueErrorOutput`, retries enabled.
- **Connections:** Output → **Update Sheet - Scheduled**
- **Edge cases:** Same as above.

#### Node: Update Sheet - Scheduled
- **Type / role:** Google Sheets update – final status.
- **Config choices:**
  - Operation: update
  - Match by `content_id` taken from **Collect All Platform Posts**
  - Set `status = scheduled`
- **Connections:** Output → **Process Each Item** (loop continues)
- **Edge cases/failures:**
  - Because scheduling nodes “continue on error”, this status may still become “scheduled” even if Buffer calls failed (unless you add checks).

---

### Block 5 — Success Notification
**Overview:**  
After all batches are processed, sends a Slack summary message.

**Nodes involved:**
- Send Success Summary

#### Node: Send Success Summary
- **Type / role:** Slack (`n8n-nodes-base.slack`) – notification.
- **Config:** Posts a static summary including:
  - Execution ID: `{{ $execution.id }}`
  - Completion timestamp: `{{ $now.toFormat('yyyy-MM-dd HH:mm') }}`
- **Connections:** None (end).
- **Triggering:** Connected to SplitInBatches **output index 1** (end-of-loop).
- **Edge cases/failures:**
  - Slack credential/channel misconfiguration
  - Message can be sent even if some items failed mid-way (due to continue-on-error)

---

### Block 6 — Global Error Handling (Trigger-based)
**Overview:**  
If the workflow execution fails (unhandled error), it prepares error details, updates the sheet, and alerts Slack.

**Nodes involved:**
- On Workflow Error
- Prepare Error Data
- Update Sheet - Failed
- Send Error Alert

#### Node: On Workflow Error
- **Type / role:** Error Trigger (`n8n-nodes-base.errorTrigger`) – separate entry point.
- **Config:** None; fires on workflow error.
- **Connections:** Output → **Prepare Error Data**
- **Edge cases:** Only catches errors that cause workflow failure. Errors swallowed by `onError: continueErrorOutput` will not reach this path.

#### Node: Prepare Error Data
- **Type / role:** Set – normalize error payload.
- **Key expressions:**
  - `error_message = {{ $json.execution?.error?.message ?? $json.error?.message ?? 'Workflow execution failed - check n8n logs' }}`
  - `content_id = {{ $json.execution?.lastNodeExecuted ?? 'unknown' }}` *(note: value is actually a node name, not a content_id)*
  - `execution_id = {{ $json.execution?.id ?? $execution.id }}`
  - `failed_node = {{ $json.execution?.lastNodeExecuted ?? 'unknown' }}`
  - `failed_at = {{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}`
- **Connections:** Output → **Update Sheet - Failed**
- **Edge cases/failures:**
  - Mis-mapping: `content_id` is set to last executed node; the sheet update that matches by `content_id` will likely not find a row to update.

#### Node: Update Sheet - Failed
- **Type / role:** Google Sheets update – logs failure.
- **Config choices:**
  - Operation: update
  - Match by `content_id`
  - Writes:
    - `status = failed`
    - `error_log = [timestamp] Node: X | Error: Y`
- **Retries:** enabled (2 tries, 2s wait).
- **Connections:** Output → **Send Error Alert**
- **Edge cases/failures:**
  - Because `content_id` is not the real row ID, update may do nothing.
  - Column `error_log` must exist.

#### Node: Send Error Alert
- **Type / role:** Slack – failure notification.
- **Config:** Posts execution id, failed node, error message, timestamp.
- **Edge cases:** Slack credential/channel issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily 6AM Trigger | Schedule Trigger | Daily cron entry point | — | Get Pending Content | **Trigger & fetch** / Daily schedule + Google Sheets read |
| Get Pending Content | Google Sheets | Fetch rows with `status=pending` | Daily 6AM Trigger | Validate Required Fields | **Trigger & fetch** / Daily schedule + Google Sheets read |
| Validate Required Fields | IF | Ensure required columns are present | Get Pending Content | Process Each Item; No Pending Content | **Validation** / Check required fields before processing |
| No Pending Content | NoOp | End branch for invalid/no items | Validate Required Fields (false) | — | **Validation** / Check required fields before processing |
| Process Each Item | Split In Batches | Loop over rows sequentially | Validate Required Fields (true); Update Sheet - Scheduled | Generate LinkedIn Post; Send Success Summary | **AI content generation** / Sequential: LinkedIn > Twitter > Facebook |
| Generate LinkedIn Post | OpenAI (LangChain) | Generate LinkedIn copy | Process Each Item | Generate Twitter Post | **AI content generation** / Sequential: LinkedIn > Twitter > Facebook |
| Generate Twitter Post | OpenAI (LangChain) | Generate Twitter/X copy | Generate LinkedIn Post | Generate Facebook Post | **AI content generation** / Sequential: LinkedIn > Twitter > Facebook |
| Generate Facebook Post | OpenAI (LangChain) | Generate Facebook copy | Generate Twitter Post | Collect All Platform Posts | **AI content generation** / Sequential: LinkedIn > Twitter > Facebook |
| Collect All Platform Posts | Set | Consolidate generated posts + metadata | Generate Facebook Post | Update Sheet - Content Generated | **AI content generation** / Sequential: LinkedIn > Twitter > Facebook |
| Update Sheet - Content Generated | Google Sheets | Write generated posts + status=generated | Collect All Platform Posts | Schedule LinkedIn to Buffer | **Scheduling & status** / Buffer API calls + Google Sheets updates |
| Schedule LinkedIn to Buffer | HTTP Request | Schedule LinkedIn update in Buffer | Update Sheet - Content Generated | Schedule Twitter to Buffer | **Scheduling & status** / Buffer API calls + Google Sheets updates |
| Schedule Twitter to Buffer | HTTP Request | Schedule Twitter update in Buffer | Schedule LinkedIn to Buffer | Schedule Facebook to Buffer | **Scheduling & status** / Buffer API calls + Google Sheets updates |
| Schedule Facebook to Buffer | HTTP Request | Schedule Facebook update in Buffer | Schedule Twitter to Buffer | Update Sheet - Scheduled | **Scheduling & status** / Buffer API calls + Google Sheets updates |
| Update Sheet - Scheduled | Google Sheets | Mark row as scheduled | Schedule Facebook to Buffer | Process Each Item | **Scheduling & status** / Buffer API calls + Google Sheets updates |
| Send Success Summary | Slack | Post completion message | Process Each Item (done output) | — |  |
| On Workflow Error | Error Trigger | Alternate entry on workflow failure | — | Prepare Error Data | **Error handling** / Log failures to sheet + Slack alert |
| Prepare Error Data | Set | Extract/format error fields | On Workflow Error | Update Sheet - Failed | **Error handling** / Log failures to sheet + Slack alert |
| Update Sheet - Failed | Google Sheets | Mark failed + write error log | Prepare Error Data | Send Error Alert | **Error handling** / Log failures to sheet + Slack alert |
| Send Error Alert | Slack | Alert on workflow failure | Update Sheet - Failed | — | **Error handling** / Log failures to sheet + Slack alert |
| Note - Main | Sticky Note | Documentation (non-executing) | — | — |  |
| Note - Trigger section | Sticky Note | Documentation (non-executing) | — | — |  |
| Note - Validation section | Sticky Note | Documentation (non-executing) | — | — |  |
| Note - AI generation section | Sticky Note | Documentation (non-executing) | — | — |  |
| Note - Buffer scheduling section | Sticky Note | Documentation (non-executing) | — | — |  |
| Note - Error handling section | Sticky Note | Documentation (non-executing) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow**
   - Name: “Generate and schedule social media posts with OpenAI, Google Sheets, and Buffer”

2. **Create required n8n Variables** (Settings → Variables)
   1) `GOOGLE_SHEET_URL` = full Google Sheet URL  
   2) `BUFFER_LINKEDIN_PROFILE_ID` = Buffer profile id for LinkedIn  
   3) `BUFFER_TWITTER_PROFILE_ID` = Buffer profile id for Twitter/X  
   4) `BUFFER_FACEBOOK_PROFILE_ID` = Buffer profile id for Facebook  

3. **Add Trigger**
   - Node: **Schedule Trigger**
   - Set cron: `0 6 * * *` (daily 06:00 UTC)
   - Connect → Google Sheets node

4. **Add Google Sheets “Get Pending Content”**
   - Node: **Google Sheets**
   - Credentials: **Google Sheets OAuth2**
   - Document: use URL expression `{{$vars.GOOGLE_SHEET_URL ?? ''}}`
   - Sheet: select tab `gid=0`
   - Add filter: column `status` equals `pending`
   - Connect → IF node

5. **Add IF “Validate Required Fields”**
   - Conditions (AND):
     - `{{$json.content_id}}` is not empty
     - `{{$json.topic}}` is not empty
     - `{{$json.key_points}}` is not empty
     - `{{$json.schedule_date}}` is not empty
     - `{{$json.schedule_time}}` is not empty
   - True output → Split In Batches
   - False output → NoOp

6. **Add “No Pending Content”**
   - Node: **NoOp**
   - No further connections

7. **Add “Process Each Item”**
   - Node: **Split In Batches**
   - Leave defaults (or set batch size explicitly to 1 if desired)
   - Output 0 → LinkedIn generation node
   - Output 1 → Slack success node (added later)

8. **Add OpenAI generation nodes (3)**
   - Credentials: **OpenAI API** credential on each node
   - Node 1: **Generate LinkedIn Post**
     - Model: `gpt-4o-mini`
     - Max tokens 500, temperature 0.7
     - Use two-message prompt (system-style rules + user content) and reference:
       - `$('Process Each Item').item.json.topic`, `key_points`, `tone`, `target_audience`
     - Enable retry (3 tries) and set **Continue On Fail** (onError continue)
     - Connect → Twitter node
   - Node 2: **Generate Twitter Post**
     - Model: `gpt-4o-mini`
     - Max tokens 100, temperature 0.8
     - Similar references
     - Continue on fail + retry
     - Connect → Facebook node
   - Node 3: **Generate Facebook Post**
     - Model: `gpt-4o-mini`
     - Max tokens 300, temperature 0.7
     - Continue on fail + retry
     - Connect → Set node

9. **Add Set “Collect All Platform Posts”**
   - Add fields:
     - `linkedin_post` = `{{$('Generate LinkedIn Post').first().json.text ?? 'GENERATION_FAILED'}}`
     - `twitter_post` = `{{$('Generate Twitter Post').first().json.text ?? 'GENERATION_FAILED'}}`
     - `facebook_post` = `{{$('Generate Facebook Post').first().json.text ?? 'GENERATION_FAILED'}}`
     - `content_id` = `{{$('Process Each Item').item.json.content_id}}`
     - `topic` = `{{$('Process Each Item').item.json.topic ?? ''}}`
     - `platforms` = `{{$('Process Each Item').item.json.platforms ?? 'linkedin,twitter,facebook'}}`
     - `schedule_date` = `{{$('Process Each Item').item.json.schedule_date ?? ''}}`
     - `schedule_time` = `{{$('Process Each Item').item.json.schedule_time ?? '09:00'}}`
     - `row_number` = `{{$('Process Each Item').item.json.row_number}}`
     - `status` = `generated`
   - Connect → Update generated content node

10. **Add Google Sheets “Update Sheet - Content Generated”**
   - Operation: **Update**
   - Match column: `content_id`
   - Values:
     - `status = generated`
     - `content_id = {{$json.content_id}}`
     - `twitter_post = {{$json.twitter_post}}`
     - `facebook_post = {{$json.facebook_post}}`
     - `linkedin_post = {{$json.linkedin_post}}`
   - Connect → Buffer LinkedIn HTTP

11. **Create Buffer credential**
   - Credentials: **HTTP Header Auth**
   - Add header like `Authorization: Bearer <BUFFER_ACCESS_TOKEN>` (or the exact header Buffer expects for your token format)
   - Use this credential in all Buffer HTTP nodes.

12. **Add HTTP “Schedule LinkedIn to Buffer”**
   - Method: POST
   - URL: `https://api.bufferapp.com/1/updates/create.json`
   - Authentication: Generic Credential → HTTP Header Auth
   - Body (form parameters):
     - `text = {{$json.linkedin_post}}`
     - `profile_ids[] = {{$vars.BUFFER_LINKEDIN_PROFILE_ID ?? ''}}`
     - `scheduled_at = {{$json.schedule_date}}T{{$json.schedule_time}}:00Z`
   - Timeout 30000ms, retries 3, continue on fail
   - Connect → Schedule Twitter node

13. **Add HTTP “Schedule Twitter to Buffer”**
   - Same endpoint/auth
   - Body:
     - `text = {{$('Collect All Platform Posts').first().json.twitter_post}}`
     - `profile_ids[] = {{$vars.BUFFER_TWITTER_PROFILE_ID ?? ''}}`
     - `scheduled_at = {{$('Collect All Platform Posts').first().json.schedule_date}}T{{$('Collect All Platform Posts').first().json.schedule_time}}:00Z`
   - Continue on fail + retries
   - Connect → Schedule Facebook node

14. **Add HTTP “Schedule Facebook to Buffer”**
   - Same endpoint/auth
   - Body:
     - `text = {{$('Collect All Platform Posts').first().json.facebook_post}}`
     - `profile_ids[] = {{$vars.BUFFER_FACEBOOK_PROFILE_ID ?? ''}}`
     - `scheduled_at = {{$('Collect All Platform Posts').first().json.schedule_date}}T{{$('Collect All Platform Posts').first().json.schedule_time}}:00Z`
   - Continue on fail + retries
   - Connect → Update scheduled node

15. **Add Google Sheets “Update Sheet - Scheduled”**
   - Operation: Update
   - Match column: `content_id`
   - Values:
     - `status = scheduled`
     - `content_id = {{$('Collect All Platform Posts').first().json.content_id}}`
   - Connect → back to **Process Each Item** (to fetch next batch)

16. **Add Slack “Send Success Summary”**
   - Node: **Slack**
   - Credentials: **Slack OAuth2**
   - Channel: select your notification channel
   - Message text uses:
     - `{{$execution.id}}`
     - `{{$now.toFormat('yyyy-MM-dd HH:mm')}}`
   - Connect this node to **Process Each Item** output **index 1** (the “done” output)

17. **Add global error path**
   1) Node: **Error Trigger** (On Workflow Error)
   2) Node: **Set** (Prepare Error Data) with fields:
      - `error_message`, `content_id`, `execution_id`, `failed_node`, `failed_at` as per expressions above
   3) Node: **Google Sheets Update** (Update Sheet - Failed)
      - Match by `content_id`
      - Set `status=failed`, write `error_log`
   4) Node: **Slack** (Send Error Alert) with error details
   - Connect: Error Trigger → Set → Sheets Update → Slack

18. **Ensure the Google Sheet columns exist**
   - Required columns referenced by this workflow:
     - `content_id`, `topic`, `key_points`, `tone`, `target_audience`, `platforms`, `schedule_date`, `schedule_time`, `status`, `linkedin_post`, `twitter_post`, `facebook_post`, `error_log`
   - Ensure `content_id` is unique if you rely on it as a key.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow runs daily at 6 AM UTC… generates three platform-specific posts… written back to the sheet, then scheduled… Slack summary… error logged + Slack alert.” | Sticky note: “Note - Main” |
| Setup steps: create variables `GOOGLE_SHEET_URL`, `BUFFER_LINKEDIN_PROFILE_ID`, `BUFFER_TWITTER_PROFILE_ID`, `BUFFER_FACEBOOK_PROFILE_ID`; connect Google Sheets OAuth2; OpenAI API credential; Buffer HTTP Header Auth; Slack OAuth2; ensure sheet columns listed. | Sticky note: “Note - Main” |
| “Trigger & fetch: Daily schedule + Google Sheets read” | Sticky note: “Note - Trigger section” |
| “Validation: Check required fields before processing” | Sticky note: “Note - Validation section” |
| “AI content generation: Sequential: LinkedIn > Twitter > Facebook” | Sticky note: “Note - AI generation section” |
| “Scheduling & status: Buffer API calls + Google Sheets updates” | Sticky note: “Note - Buffer scheduling section” |
| “Error handling: Log failures to sheet + Slack alert” | Sticky note: “Note - Error handling section” |