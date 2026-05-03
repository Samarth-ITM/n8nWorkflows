Generate AI video clips from YouTube automatically with Reka Vision

https://n8nworkflows.xyz/workflows/generate-ai-video-clips-from-youtube-automatically-with-reka-vision-12829


# Generate AI video clips from YouTube automatically with Reka Vision

## 1. Workflow Overview

**Purpose:** Automatically monitor a YouTube channel via its RSS feed, submit each new video to **Reka Vision** to generate an AI-edited video clip (optionally with subtitles), then **poll** Reka’s job status until completion and finally **email** either the clip link (success) or an error message (timeout/failsafe).

**Target use cases:**
- Auto-generating short clips from newly published YouTube videos for social media.
- Automating repetitive “check status → notify me” loops for async AI rendering jobs.

### 1.1 Input Reception (YouTube RSS Trigger)
Triggered when a new item appears in a configured YouTube RSS feed.

### 1.2 Clip Generation Request (Reka Vision)
Submits the YouTube video URL to Reka Vision to start a clip-generation job.

### 1.3 Looping / Polling & Failsafe Control
Implements a timed polling loop (wait → check status → increment counter → evaluate completion/timeout) to avoid infinite loops.

### 1.4 Notifications (Gmail)
Sends a success email with clip details when completed, or a failure email after too many checks.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (YouTube RSS Trigger)

**Overview:** Watches an RSS feed (intended to be a YouTube channel feed) and triggers the workflow when a new video entry is detected.

**Nodes Involved:**
- When New Video

**Node Details**

#### 1) When New Video
- **Type / Role:** `rssFeedReadTrigger` — Trigger node that polls an RSS feed.
- **Configuration (interpreted):**
- **Feed URL:** currently `https://` (placeholder). Intended to be a YouTube channel feed, e.g. `https://www.youtube.com/feeds/videos.xml?channel_id=UC...`
- **Polling schedule:** once daily at **10:00** (server/workflow timezone).
- **Key data used later:** `{{$json.link}}` (the new RSS item’s link) becomes the input video URL for Reka.
- **Connections:**
  - **Output →** Create a clip
- **Edge cases / failure modes:**
  - Invalid/empty feed URL (as currently set) results in trigger failure or no items.
  - RSS feed throttling/network/DNS errors.
  - YouTube feed changes or rate limits may cause missed triggers.
- **Version notes:** typeVersion `1`.

---

### Block 2 — Clip Generation Request (Reka Vision)

**Overview:** Submits the YouTube video URL to Reka Vision to create a clip-generation job, enabling subtitles.

**Nodes Involved:**
- Create a clip

**Node Details**

#### 2) Create a clip
- **Type / Role:** `@reka-ai/n8n-nodes-reka.rekaVision` — Reka community node to start a vision/video job.
- **Configuration (interpreted):**
  - **Video URL(s):** `={{ $json.link }}`
  - **Additional fields:** `subtitles: true`
  - (No prompt/template/min/max duration shown in this JSON; likely defaults are used.)
- **Credentials:** `rekaApi` (Reka account / API key)
- **Output:** Expected to return a job object including an `id` used for status polling.
- **Connections:**
  - **Output →** Counter Init
- **Edge cases / failure modes:**
  - Reka credential invalid/expired → auth error.
  - Unsupported/geo-blocked/private YouTube URL.
  - Long videos may take much longer than the wait loop allows.
  - API errors/timeouts; transient failures could require retries.
- **Version notes:** typeVersion `1`.

---

### Block 3 — Looping / Polling & Failsafe Control

**Overview:** After submitting the job, the workflow initializes a counter, then loops: waits 10 minutes, checks job status, increments a counter, exits on “completed”, or fails after a maximum number of checks.

**Nodes Involved:**
- Counter Init
- Loop To Check the Status
- Wait 10 minutes
- Check clip job status
- Counter +1
- If Completed
- If MAX Reached

**Node Details**

#### 3) Counter Init
- **Type / Role:** `Set` — Initializes loop state.
- **Configuration (interpreted):**
  - Outputs fixed JSON: `{ "counter": 0 }`
- **Connections:**
  - **Output →** Loop To Check the Status
- **Edge cases / failure modes:**
  - This counter is intended to drive the max-iteration failsafe, but later nodes reference `Counter Init` again rather than the incremented value (see notes under “Counter +1” and “If MAX Reached”).
- **Version notes:** typeVersion `3.4`.

#### 4) Loop To Check the Status
- **Type / Role:** `Split In Batches` — Used here as a looping mechanism.
- **Configuration (interpreted):**
  - `reset: false` (keeps batch state between calls)
  - **Connections usage:** the workflow uses the **second output** (the “continue” path) to proceed into the loop.
- **Connections:**
  - **Output 1:** not used (empty)
  - **Output 2 →** Wait 10 minutes
- **Edge cases / failure modes:**
  - This is a non-standard but common pattern; if the node state is reset unexpectedly, looping behavior can change.
  - With no defined batch size/items, this is effectively being used as a loop gate; ensure it behaves as expected in your n8n version.
- **Version notes:** typeVersion `3`.

#### 5) Wait 10 minutes
- **Type / Role:** `Wait` — Delays before checking status again.
- **Configuration (interpreted):**
  - Waits **10 minutes** each loop iteration (sticky note text mentions 15 minutes; the node is set to 10).
- **Connections:**
  - **Output →** Check clip job status
- **Edge cases / failure modes:**
  - If Reka jobs routinely take longer than (10 minutes × number of iterations), failures will be common.
  - Wait node resumes executions; ensure your n8n instance is configured to handle waiting executions reliably.
- **Version notes:** typeVersion `1.1`.

#### 6) Check clip job status
- **Type / Role:** `@reka-ai/n8n-nodes-reka.rekaVision` — Query job status.
- **Configuration (interpreted):**
  - **Operation:** `getJobtatus` (note spelling as provided by the node/JSON)
  - **Job ID:** `={{ $('Create a clip').item.json.id }}`
- **Credentials:** `rekaApi`
- **Output:** Expected to include `status` and, when completed, an `output` array containing fields like `title`, `video_url`, `caption`.
- **Connections:**
  - **Output →** Counter +1
- **Edge cases / failure modes:**
  - If “Create a clip” returns a different structure, `id` expression fails.
  - Status fields may differ; if `status` is absent/renamed, completion check will never pass.
  - API timeouts or job not found.
- **Version notes:** typeVersion `1`.

#### 7) Counter +1
- **Type / Role:** `Set` — Intended to increment loop counter.
- **Configuration (interpreted):**
  - Outputs:  
    `my_field_1 = {{ $('Counter Init').item.json.counter }} + 1`
- **Important logic note:** This increments **the original Counter Init value** (always 0) and stores it in **`my_field_1`**, not back into `counter`. As written, it will always output `my_field_1: 1` each iteration and does **not** persist/increase across loops.
- **Connections:**
  - **Output →** If Completed
- **Edge cases / failure modes:**
  - The failsafe “10 checks max” will not work as intended unless counter state is carried forward and compared correctly.
- **Version notes:** typeVersion `3.4`.

#### 8) If Completed
- **Type / Role:** `If` — Branches on job completion.
- **Configuration (interpreted):**
  - Condition: `={{ $json.status }} == "completed"`
- **Connections:**
  - **True →** Send Clip Ready EMail
  - **False →** If MAX Reached
- **Edge cases / failure modes:**
  - If Reka returns `completed` vs `completed` (case/enum differences), the workflow may never exit successfully.
  - If `status` is nested (e.g., `$json.data.status`), expression must be updated.
- **Version notes:** typeVersion `2.2`.

#### 9) If MAX Reached
- **Type / Role:** `If` — Failsafe stop condition to prevent infinite looping.
- **Configuration (interpreted):**
  - Condition: `={{ $('Counter Init').item.json.counter }} == "10"`
- **Important logic note:** This compares the **initial counter (0)** to `"10"` (string). As written, it will never be true, so the workflow will keep looping indefinitely (or until stopped manually), unless n8n execution limits intervene.
- **Connections:**
  - **True →** Send Failure EMail
  - **False →** Loop To Check the Status (continues looping)
- **Edge cases / failure modes:**
  - Infinite loop risk (exactly what this node is meant to prevent).
- **Version notes:** typeVersion `2.2`.

---

### Block 4 — Notifications (Gmail)

**Overview:** Sends an email on success containing clip metadata and download URL, or a failure email if the loop exceeds the max checks (intended behavior).

**Nodes Involved:**
- Send Clip Ready EMail
- Send Failure EMail

**Node Details**

#### 10) Send Clip Ready EMail
- **Type / Role:** `gmail` — Sends success notification email.
- **Configuration (interpreted):**
  - **To:** `user@example.com` (placeholder)
  - **Subject:** `Your Clip is ready`
  - **Body:** Uses fields from `Check clip job status`:
    - Title: `{{ $('Check clip job status').item.json.output[0].title }}`
    - Download URL: `{{ $('Check clip job status').item.json.output[0].video_url }}`
    - Description: `{{ $('Check clip job status').item.json.output[0].caption }}`
- **Credentials:** `gmailOAuth2` (Gmail account)
- **Connections:** No outgoing connections.
- **Edge cases / failure modes:**
  - If `output` is empty or not an array, indexing `[0]` will error.
  - Gmail OAuth token expiration / missing scopes.
  - Email deliverability issues if sending limits are hit.
- **Version notes:** typeVersion `2.1`.

#### 11) Send Failure EMail
- **Type / Role:** `gmail` — Sends failure/timeout notification email.
- **Configuration (interpreted):**
  - **To:** `user@example.com` (placeholder)
  - **Subject:** `Oops...`
  - **Body:** Includes the source video link: `{{ $('When New Video').item.json.link }}`
  - Mentions “your 8n8 automation” (likely meant “n8n”).
- **Credentials:** `gmailOAuth2`
- **Connections:** No outgoing connections.
- **Edge cases / failure modes:**
  - Same Gmail OAuth/delivery risks as above.
- **Version notes:** typeVersion `2.1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When New Video | rssFeedReadTrigger | Poll YouTube RSS feed and trigger on new videos | — | Create a clip | ## Customize - **When New Video**: Set the Feed URL to the YouTube channel you want to follow (ex: https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA) - **Create a clip**: Change the prompt and adjust the setting to your preferences - **Send ___ EMail**: Personalize the email the way you like. |
| Create a clip | @reka-ai/n8n-nodes-reka.rekaVision | Submit clip-generation job to Reka Vision | When New Video | Counter Init | ## Customize - **When New Video**: Set the Feed URL to the YouTube channel you want to follow (ex: https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA) - **Create a clip**: Change the prompt and adjust the setting to your preferences - **Send ___ EMail**: Personalize the email the way you like. |
| Counter Init | n8n-nodes-base.set | Initialize loop counter | Create a clip | Loop To Check the Status |  |
| Loop To Check the Status | n8n-nodes-base.splitInBatches | Loop gate for repeated wait/status checks | Counter Init, If MAX Reached (false branch) | Wait 10 minutes (via output 2) | ## Looping - After 15 minutes check if the job status is equal to "completed" - Exit after 10 checks to avoid an infinite loop |
| Wait 10 minutes | n8n-nodes-base.wait | Delay between polling attempts | Loop To Check the Status | Check clip job status | ## 💡 Waiting Generating the clip can take time. The video needs to be downloaded to your Reka’s library, analyzed, edited, and then rendered. Adjust the waiting duration according to the duration of your video. #### Example For 5-8 minutes, videos wait 15 minutes. |
| Check clip job status | @reka-ai/n8n-nodes-reka.rekaVision | Poll Reka job status | Wait 10 minutes | Counter +1 | ## Looping - After 15 minutes check if the job status is equal to "completed" - Exit after 10 checks to avoid an infinite loop |
| Counter +1 | n8n-nodes-base.set | Increment counter (intended) | Check clip job status | If Completed | ## Looping - After 15 minutes check if the job status is equal to "completed" - Exit after 10 checks to avoid an infinite loop |
| If Completed | n8n-nodes-base.if | Branch on job completion | Counter +1 | Send Clip Ready EMail (true), If MAX Reached (false) | ## Looping - After 15 minutes check if the job status is equal to "completed" - Exit after 10 checks to avoid an infinite loop |
| If MAX Reached | n8n-nodes-base.if | Stop looping after max checks (intended) | If Completed (false) | Send Failure EMail (true), Loop To Check the Status (false) | ## Looping - After 15 minutes check if the job status is equal to "completed" - Exit after 10 checks to avoid an infinite loop |
| Send Clip Ready EMail | n8n-nodes-base.gmail | Email clip details on success | If Completed (true) | — | ## Customize - **When New Video**: Set the Feed URL to the YouTube channel you want to follow (ex: https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA) - **Create a clip**: Change the prompt and adjust the setting to your preferences - **Send ___ EMail**: Personalize the email the way you like. |
| Send Failure EMail | n8n-nodes-base.gmail | Email failure/timeout notice | If MAX Reached (true) | — | ## Customize - **When New Video**: Set the Feed URL to the YouTube channel you want to follow (ex: https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA) - **Create a clip**: Change the prompt and adjust the setting to your preferences - **Send ___ EMail**: Personalize the email the way you like. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation: looping behavior | — | — | ## Looping - After 15 minutes check if the job status is equal to "completed" - Exit after 10 checks to avoid an infinite loop |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation: customization points | — | — | ## Customize - **When New Video**: Set the Feed URL to the YouTube channel you want to follow (ex: https://www.youtube.com/feeds/videos.xml?channel_id=UCAr20GBQayL-nFPWFnUHNAA) - **Create a clip**: Change the prompt and adjust the setting to your preferences - **Send ___ EMail**: Personalize the email the way you like. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation: template overview/requirements/links | — | — | ## Try It Out! **This n8n template demonstrates how to use Reka Vision to automatically generate a customized video clip from a YouTube video using AI. The workflow keeps you informed by sending an email when the work is done, ready to be published on social!** ### How it works - Looking at the RSS feed of a YouTube channel, the flow will be triggered when there is a new video published. - Using Reka's Community Node we will submit a request to create a clip using AI. You can customize: - The template used - The prompt to generate the clip - If you want captions or not - The minimum or maximum duration of the clip - Wait a little while, the magic happens. - Check if the job status is **completed** - If **yes**, it send an success **email** - If **no**, it will **loop** - As a failsafe, after 10 iterations in the loop, it will send an error email ### Getting Started Edit those nodes with your settings and credentials. - **When New Video**: Set the Feed URL to the YouTube channel you want to follow (ex:  - **Create a clip**: Add your credential. Change the prompt and adjust the setting to your preferences - **Send ___ EMail**: Personalize the email the way you like. ### Requirements - Reka AI API key (it's free! Get yours from [here from Reka](https://link.reka.ai/free)) - Gmail account (feel free to change it to another email provider) ### Need Help? Join the [Discord](https://link.reka.ai/discord) Happy clipping! |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation: waiting guidance | — | — | ## 💡 Waiting Generating the clip can take time. The video needs to be downloaded to your Reka’s library, analyzed, edited, and then rendered. Adjust the waiting duration according to the duration of your video. #### Example For 5-8 minutes, videos wait 15 minutes. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger Node**
   - Add **RSS Feed Read Trigger** node named **When New Video**
   - Set **Feed URL** to your YouTube channel feed, e.g.  
     `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID`
   - Set polling time (e.g., daily at **10:00**).

2) **Install and configure Reka community node**
   - Ensure the **Reka Vision** community node package is installed in your n8n environment (`@reka-ai/n8n-nodes-reka`).
   - Create **Reka API** credentials (API key) in n8n.

3) **Create “Create a clip” node**
   - Add **Reka Vision** node named **Create a clip**
   - Configure it to **create a clip** from a video URL:
     - Video URL field: use expression `{{$json.link}}`
     - Enable **subtitles/captions** (set `subtitles = true`)
     - Attach **Reka API credentials**
   - Connect: **When New Video → Create a clip**

4) **Initialize loop counter**
   - Add **Set** node named **Counter Init**
   - Mode: **Raw JSON**
   - JSON output:
     - `{"counter": 0}`
   - Connect: **Create a clip → Counter Init**

5) **Add loop gate**
   - Add **Split In Batches** node named **Loop To Check the Status**
   - Set **Options → Reset** to **false**
   - Connect: **Counter Init → Loop To Check the Status**
   - Use the node’s looping output (commonly the **second output**) to continue the loop.

6) **Add waiting**
   - Add **Wait** node named **Wait 10 minutes**
   - Configure: **Unit = minutes**, **Amount = 10**
   - Connect: **Loop To Check the Status (output 2) → Wait 10 minutes**

7) **Add job status polling**
   - Add **Reka Vision** node named **Check clip job status**
   - Operation: job status lookup (as exposed by your node version)
   - Job ID expression: `{{ $('Create a clip').item.json.id }}`
   - Attach **Reka API credentials**
   - Connect: **Wait 10 minutes → Check clip job status**

8) **Add counter increment**
   - Add **Set** node named **Counter +1**
   - Increment a counter value (important: store it consistently, e.g. keep the field name `counter`)
   - Connect: **Check clip job status → Counter +1**

   **Note:** The provided workflow’s counter logic does not persist correctly. To reproduce the *intended* behavior, ensure the increment reads the previous counter value from the current item and writes back to `counter`.

9) **Add completion check**
   - Add **If** node named **If Completed**
   - Condition: `{{$json.status}}` equals `completed`
   - Connect: **Counter +1 → If Completed**

10) **Add max-iterations failsafe**
   - Add **If** node named **If MAX Reached**
   - Condition: your counter field (e.g. `{{$json.counter}}`) equals `10` (number) or `"10"` (string), consistently with how you store it
   - Connect:
     - **If Completed (false) → If MAX Reached**
     - **If MAX Reached (false) → Loop To Check the Status** (continue looping)

11) **Configure Gmail credentials**
   - Create **Gmail OAuth2** credentials in n8n (Google Cloud project + OAuth consent + scopes as required by n8n).
   - Alternatively replace Gmail nodes with another email provider node.

12) **Success email**
   - Add **Gmail** node named **Send Clip Ready EMail**
   - To: your email address
   - Subject: “Your Clip is ready”
   - Body: reference the polling node output fields, e.g.:
     - `output[0].title`, `output[0].video_url`, `output[0].caption`
   - Connect: **If Completed (true) → Send Clip Ready EMail**

13) **Failure email**
   - Add **Gmail** node named **Send Failure EMail**
   - To: your email address
   - Subject: “Oops...”
   - Body: include the source video link, e.g. `{{ $('When New Video').item.json.link }}`
   - Connect: **If MAX Reached (true) → Send Failure EMail**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Reka AI API key is required (noted as free). | https://link.reka.ai/free |
| Community support link. | https://link.reka.ai/discord |
| Suggested YouTube RSS format: `https://www.youtube.com/feeds/videos.xml?channel_id=...` | Mentioned in sticky notes (example provided). |
| Waiting guidance: clip generation can take time; adjust wait duration based on video length. | Captured in “Waiting” sticky note. |
| **Implementation warning:** As provided, the counter/timeout logic is inconsistent (counter never truly increments; max check compares against the initial counter). | Impacts “Exit after 10 checks” behavior. |

