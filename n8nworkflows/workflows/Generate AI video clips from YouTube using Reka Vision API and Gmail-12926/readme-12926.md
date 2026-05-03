Generate AI video clips from YouTube using Reka Vision API and Gmail

https://n8nworkflows.xyz/workflows/generate-ai-video-clips-from-youtube-using-reka-vision-api-and-gmail-12926


# Generate AI video clips from YouTube using Reka Vision API and Gmail

## 1. Workflow Overview

**Purpose:** Automatically detect new videos published on a YouTube channel (via RSS), request Reka Vision API to generate an AI short vertical clip (reel) from the video, poll the job status until completion (or timeout), then notify via Gmail (success or failure).

**Target use cases:** Content repurposing, social media shorts generation, automated post-production pipelines, monitoring a channel and generating highlight clips.

### 1.1 Input Reception (YouTube RSS Trigger)
Watches a YouTube channel RSS feed and triggers the workflow when a new video appears.

### 1.2 Clip Generation Job Submission (Reka Vision API)
Creates a “reel creation job” at Reka using the new video URL and a prompt/template configuration.

### 1.3 Controlled Polling Loop (Wait → Status Check → Exit Conditions)
Waits a fixed duration, checks job status, loops until `status=completed` or a maximum number of checks is reached.

### 1.4 Notifications (Gmail)
Sends an email with the clip download link on success, or a failure notice after hitting the maximum polling attempts.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (YouTube RSS Trigger)

**Overview:** Triggers once per polling schedule and emits RSS items (YouTube videos) when a new entry is detected.

**Nodes Involved:**
- **When New Video** (RSS Feed Read Trigger)

#### Node Details

**When New Video**
- **Type / role:** `rssFeedReadTrigger` — scheduled polling trigger for RSS feeds.
- **Key configuration:**
  - **Feed URL:** `https://` (placeholder; must be replaced with a valid YouTube RSS feed URL).
  - **Poll time:** at **10:00** (hour=10) daily (as configured).
- **Input / output:**
  - **Input:** none (trigger).
  - **Output:** RSS item JSON; the workflow later uses **`$json.link`** (the video URL).
- **Edge cases / failures:**
  - Invalid feed URL → no items / fetch errors.
  - YouTube feed temporarily unavailable → missed triggers depending on polling.
  - Multiple new items could trigger multiple executions depending on node behavior and n8n version/settings.

**Relevant sticky note content (applies to this area):**
- “Set the Feed URL to the YouTube channel you want to follow (ex: `https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA`)”

---

### Block 2 — Clip Generation Job Submission (Reka Vision API)

**Overview:** Submits the YouTube video URL to Reka Vision Agent “creator/reels” endpoint to create an async job that will generate a short clip.

**Nodes Involved:**
- **Create Reel Creation Job** (HTTP Request)

#### Node Details

**Create Reel Creation Job**
- **Type / role:** `httpRequest` — POST request to Reka Vision API to create a reel generation job.
- **Authentication:**
  - Uses **predefined credential** type **HTTP Bearer Auth** (`httpBearerAuth`) labeled “Reka APIs”.
  - Requires a valid **Reka API key** as Bearer token.
- **Key configuration:**
  - **Method:** `POST`
  - **URL:** `https://vision-agent.api.reka.ai/v1/creator/reels`
  - **Body (JSON):**
    - `video_urls`: `["{{ $json.link }}"]` (pulls the YouTube URL from the RSS item)
    - `prompt`: `"Create an engaging short video highlighting the best moments"`
    - `generation_config`:
      - `template`: `"moments"`
      - `num_generations`: `1`
      - `min_duration_seconds`: `0`
      - `max_duration_seconds`: `30`
    - `rendering_config`:
      - `subtitles`: `true`
      - `aspect_ratio`: `"9:16"`
- **Input / output:**
  - **Input:** from **When New Video** (RSS item).
  - **Output:** expected to include a **`job_id`** used later by the status check (e.g., `$json.job_id`).
- **Edge cases / failures:**
  - 401/403 if API key invalid.
  - 400 if body schema invalid or `video_urls` not accepted.
  - Some YouTube URLs may be restricted/unavailable; job may fail or remain pending.
  - Large/long videos can increase processing time beyond the workflow’s polling window.

**Relevant sticky note content (applies to this node):**
- Explains `video_urls`, `prompt`, `generation_config`, `rendering_config`, templates (“moments”/“compilation”), subtitles/aspect ratio, and provides a full JSON example.

---

### Block 3 — Controlled Polling Loop (Wait → Status Check → Exit Conditions)

**Overview:** Initializes a counter, then repeatedly waits and checks job status until it’s completed. A max-iteration safety stop prevents infinite looping.

**Nodes Involved:**
- **Counter Init** (Set)
- **Loop To Check** (Split In Batches)
- **Wait 10 minutes** (Wait)
- **Get Job Status** (HTTP Request)
- **Counter +1** (Set)
- **If Completed** (IF)
- **If MAX Reached** (IF)

#### Node Details

**Counter Init**
- **Type / role:** `set` — initializes loop state.
- **Key configuration:**
  - Raw JSON output: `{ "counter": 0 }`
- **Input / output:**
  - **Input:** from **Create Reel Creation Job**.
  - **Output:** emits `counter=0`. (Note: it does not explicitly preserve `job_id` unless n8n merges prior JSON automatically; in many cases, Set replaces fields unless “keep only set”/include other fields is configured. Here it uses raw JSON, so ensure `job_id` remains available downstream or is re-attached.)
- **Edge cases / failures:**
  - If `job_id` is not carried forward, later **Get Job Status** will build an invalid URL.

**Loop To Check**
- **Type / role:** `splitInBatches` — used here as a looping mechanism.
- **Key configuration:**
  - `reset: false` (keeps batch state across executions within the run).
  - Batch size is not explicitly shown; default behavior applies.
- **Connections behavior (important):**
  - **Output 0:** not used (empty).
  - **Output 1:** goes to **Wait 10 minutes**, effectively acting as the loop path.
- **Edge cases / failures:**
  - This is an unconventional use of Split In Batches; if misconfigured, it may not iterate as expected.
  - If no items are present or batching ends unexpectedly, loop may stop early.

**Wait 10 minutes**
- **Type / role:** `wait` — pauses execution before polling status.
- **Key configuration:** 10 minutes.
- **Input / output:** from **Loop To Check** → to **Get Job Status**.
- **Edge cases:**
  - n8n instance restarts can affect waits depending on execution mode/persistence settings.

**Get Job Status**
- **Type / role:** `httpRequest` — GET job status from Reka.
- **Key configuration:**
  - **URL expression:** `https://vision-agent.api.reka.ai/v1/creator/reels/{{ $json.job_id }}`
  - **onError:** `continueRegularOutput` (workflow continues even if request fails; response may contain error info instead of expected fields).
  - **Auth:** HTTP Bearer Auth “Reka APIs”.
- **Input / output:**
  - **Input:** expects `job_id` in current item JSON.
  - **Output:** job status JSON; later nodes read **`$json.status`**, and on success expect `output[0].title`, `output[0].video_url`, `output[0].caption`.
- **Edge cases / failures:**
  - If `job_id` missing → malformed URL / 404.
  - API can return transient errors; because “continue” is enabled, downstream IF checks may fail due to missing fields.

**Counter +1**
- **Type / role:** `set` — intended to increment a counter each polling attempt.
- **Key configuration (as written):**
  - Raw JSON output:
    ```js
    {
      "my_field_1": {{ $('Counter Init').item.json.counter }} + 1
    }
    ```
  - **Issue:** This expression always reads the counter from **Counter Init** (which is always 0), so it will always produce `1`, not an accumulating count.
  - **Issue:** It writes to `my_field_1` not `counter`, so later checks that read `counter` won’t reflect attempts.
- **Input / output:** from **Get Job Status** → to **If Completed**
- **Edge cases / failures:**
  - The “max reached” safety logic likely doesn’t work as intended due to counter not incrementing/persisting.

**If Completed**
- **Type / role:** `if` — branches on job completion.
- **Condition:** `{{ $json.status }} == "completed"`
- **Outputs:**
  - **True path →** **Send Clip Ready EMail**
  - **False path →** **If MAX Reached**
- **Edge cases:**
  - If `status` missing due to API error, condition evaluates false and workflow goes to max-check path.

**If MAX Reached**
- **Type / role:** `if` — failsafe loop breaker after 10 checks.
- **Condition (as written):** `{{ $('Counter Init').item.json.counter }} == "10"`
  - **Issue:** This reads from Counter Init again (always 0), and compares as a string `"10"`, so it will never be true.
  - Intended behavior (based on sticky note) is to stop after 10 iterations.
- **Outputs:**
  - **True path →** **Send Failure EMail**
  - **False path →** back to **Loop To Check** (continue polling)
- **Edge cases:**
  - As configured, this may create an infinite loop (wait/check forever).

**Relevant sticky note content (applies to this loop area):**
- Waiting note: generating a clip can take time; example “for 5–8 minute videos wait 15 minutes.”
- Max reached note: “Exit after 10 check to avoid infinite loop.”

---

### Block 4 — Notifications (Gmail)

**Overview:** Sends an email on success with the clip title/link/caption, or sends an error email if the job takes too long.

**Nodes Involved:**
- **Send Clip Ready EMail** (Gmail)
- **Send Failure EMail** (Gmail)

#### Node Details

**Send Clip Ready EMail**
- **Type / role:** `gmail` — sends success notification.
- **Key configuration:**
  - **To:** `user@example.com`
  - **Subject:** `Your Clip is ready`
  - **Body uses expressions referencing:**
    - `$('Check clip job status').item.json.output[0].title`
    - `$('Check clip job status').item.json.output[0].video_url`
    - `$('Check clip job status').item.json.output[0].caption`
  - **Issue:** There is no node named **“Check clip job status”** in this workflow. The correct node name appears to be **Get Job Status**.
- **Credentials:** Gmail OAuth2 (“Gmail account”)
- **Edge cases / failures:**
  - OAuth token expired/invalid.
  - Expression fails because node name is wrong or `output[0]` missing (e.g., job completed but output array empty).
  - Gmail send limits/quota.

**Send Failure EMail**
- **Type / role:** `gmail` — sends failure notification.
- **Key configuration:**
  - **To:** `user@example.com`
  - **Subject:** `Oops...`
  - **Body references:** `$('When New Video').item.json.link`
- **Credentials:** Gmail OAuth2 (“Gmail account”)
- **Edge cases / failures:**
  - Same Gmail auth/quota issues.
  - If trigger item not accessible in expression context, link may be blank.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When New Video | RSS Feed Read Trigger | Poll YouTube RSS and emit new video items | — | Create Reel Creation Job | ## When New Video\nSet the Feed URL to the YouTube channel you want to follow (ex: https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA) |
| Create Reel Creation Job | HTTP Request | Create Reka reel generation job | When New Video | Counter Init | ## Create Reel Creation Job\nThis API call will upload the video into your Reka's library and create a job for the clip.\n(Parameters and JSON example included in note) |
| Counter Init | Set | Initialize counter for polling loop | Create Reel Creation Job | Loop To Check |  |
| Loop To Check | Split In Batches | Loop mechanism driving wait/status check cycle | Counter Init; If MAX Reached (false path) | Wait 10 minutes (output 1) |  |
| Wait 10 minutes | Wait | Pause before polling job status | Loop To Check | Get Job Status | ## 💡 Waiting\nGenerating the clip can take time... For a 5-8 minutes videos wait 15 minutes. |
| Get Job Status | HTTP Request | Poll Reka job status by job_id | Wait 10 minutes | Counter +1 |  |
| Counter +1 | Set | Attempt to increment polling counter | Get Job Status | If Completed |  |
| If Completed | IF | Branch: completed vs not completed | Counter +1 | Send Clip Ready EMail (true); If MAX Reached (false) |  |
| If MAX Reached | IF | Failsafe: stop after 10 checks | If Completed (false) | Send Failure EMail (true); Loop To Check (false) | ## if MAX Reached\nExit after 10 check to avoid infinite loop |
| Send Clip Ready EMail | Gmail | Send success email with clip link/details | If Completed (true) | — |  |
| Send Failure EMail | Gmail | Send failure email after max attempts | If MAX Reached (true) | — |  |
| Sticky Note | Sticky Note | Comment | — | — | ## if MAX Reached\nExit after 10 check to avoid infinite loop |
| Sticky Note1 | Sticky Note | Comment | — | — | ## When New Video\nSet the Feed URL to the YouTube channel you want to follow (ex: https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA) |
| Sticky Note2 | Sticky Note | Comment / global description | — | — | Includes requirements (Reka API key, Gmail), and Discord link: https://link.reka.ai/discord |
| Sticky Note3 | Sticky Note | Comment about waiting duration | — | — | ## 💡 Waiting\nAdjust the waiting duration... Example: 5–8 minutes videos wait 15 minutes. |
| Sticky Note4 | Sticky Note | Comment documenting the Reka API call | — | — | Documents Create Reel Creation Job parameters + JSON example |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**
   1. Add node: **RSS Feed Read Trigger** named **“When New Video”**.
   2. Set **Feed URL** to your YouTube channel feed, e.g.  
      `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID`
   3. Set polling schedule (matching the template): **hour = 10** (or your preference).

2. **Create Reka Job Submission**
   1. Add node: **HTTP Request** named **“Create Reel Creation Job”**.
   2. Configure:
      - Method: **POST**
      - URL: `https://vision-agent.api.reka.ai/v1/creator/reels`
      - Body content type: **JSON**
      - JSON body:
        - `video_urls`: `[ "{{ $json.link }}" ]`
        - `prompt`: your prompt text
        - `generation_config`: set template/durations
        - `rendering_config`: subtitles + aspect ratio
   3. Credentials:
      - Create credential: **HTTP Bearer Auth**
      - Set Bearer token to your **Reka API key**
      - Select it in the node.
   4. Connect: **When New Video → Create Reel Creation Job**.

3. **Initialize Polling State**
   1. Add node: **Set** named **“Counter Init”**.
   2. Mode: “raw” / JSON output.
   3. Set JSON to:
      ```json
      { "counter": 0 }
      ```
   4. Connect: **Create Reel Creation Job → Counter Init**.
   5. Important: ensure the item still contains `job_id` from the previous node (if not, modify Set to *also include* `job_id`, e.g. `{ "counter": 0, "job_id": "{{ $json.job_id }}" }`).

4. **Build the Loop Driver**
   1. Add node: **Split In Batches** named **“Loop To Check”**.
   2. Set **Reset** to **false** (as in the workflow).
   3. Connect: **Counter Init → Loop To Check**.
   4. Use the **second output** of Split In Batches (the loop continuation) to drive the waiting step (as in the template).

5. **Wait Before Polling**
   1. Add node: **Wait** named **“Wait 10 minutes”**.
   2. Configure: Unit **minutes**, Amount **10** (adjust based on typical video length).
   3. Connect: **Loop To Check (output 1) → Wait 10 minutes**.

6. **Poll Reka Job Status**
   1. Add node: **HTTP Request** named **“Get Job Status”**.
   2. Configure:
      - Method: **GET**
      - URL (expression):  
        `https://vision-agent.api.reka.ai/v1/creator/reels/{{ $json.job_id }}`
      - Error handling: **Continue on fail** (equivalent to `onError: continueRegularOutput`) if you want the workflow to keep looping on transient errors.
   3. Select the same **HTTP Bearer Auth** credential (“Reka APIs”).
   4. Connect: **Wait 10 minutes → Get Job Status**.

7. **Increment Attempt Counter (fix recommended)**
   1. Add node: **Set** named **“Counter +1”**.
   2. Configure it to increment the *current* counter and store it back into `counter`. Example approach:
      - Output: `{ "counter": {{ $json.counter + 1 }}, "job_id": "{{ $json.job_id }}", "status": "{{ $json.status }}", "output": {{ $json.output }} }`
   3. Connect: **Get Job Status → Counter +1**.
   > Note: The provided template increments from `Counter Init` and writes `my_field_1`; to make max-iteration logic work, you must persist and increment `counter`.

8. **Check Completion**
   1. Add node: **IF** named **“If Completed”**.
   2. Condition: `{{ $json.status }}` **equals** `completed`.
   3. Connect: **Counter +1 → If Completed**.

9. **Send Success Email (fix node references)**
   1. Add node: **Gmail** named **“Send Clip Ready EMail”** (operation: Send).
   2. Configure To/Subject/Message.
   3. In the message, reference **Get Job Status** (or the current item) correctly, e.g.:
      - `{{ $json.output[0].title }}`
      - `{{ $json.output[0].video_url }}`
      - `{{ $json.output[0].caption }}`
   4. Set Gmail credentials: **Gmail OAuth2** (connect your Google account).
   5. Connect: **If Completed (true) → Send Clip Ready EMail**.

10. **Max Attempts Failsafe (fix recommended)**
   1. Add node: **IF** named **“If MAX Reached”**.
   2. Condition: `{{ $json.counter }}` **equals** `10` (number, not string), or “greater than or equal 10”.
   3. Connect: **If Completed (false) → If MAX Reached**.

11. **Send Failure Email**
   1. Add node: **Gmail** named **“Send Failure EMail”**.
   2. Include the original video link; if needed, pass it along in earlier nodes (e.g., store `video_link` from trigger).
   3. Connect: **If MAX Reached (true) → Send Failure EMail**.

12. **Loop Back**
   1. Connect: **If MAX Reached (false) → Loop To Check** to repeat wait/status checks.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This n8n template demonstrates how to use Reka API via HTTP to AI generate a clip automatically from a YouTube video and send an email notifications.” | Sticky note “Try It Out!” (overall workflow intent) |
| Requirements: Reka AI API key; Gmail account (or replace with another provider). | Sticky note “Try It Out!” |
| Discord help link | https://link.reka.ai/discord |
| YouTube RSS feed example | `https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA` |
| Operational warning: clip generation can take time; adjust wait duration (example: 5–8 min videos → wait 15 minutes). | Sticky note “Waiting” |

