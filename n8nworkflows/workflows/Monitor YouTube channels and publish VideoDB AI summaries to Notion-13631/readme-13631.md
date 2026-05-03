Monitor YouTube channels and publish VideoDB AI summaries to Notion

https://n8nworkflows.xyz/workflows/monitor-youtube-channels-and-publish-videodb-ai-summaries-to-notion-13631


# Monitor YouTube channels and publish VideoDB AI summaries to Notion

# 1. Workflow Overview

This workflow automates the process of monitoring a YouTube channel, processing its latest video content through AI, and archiving a structured summary in a Notion database. It leverages **VideoDB** for heavy lifting—including video hosting, spoken word indexing, transcription, and generative AI analysis—and **Notion** as the final knowledge management repository.

The workflow is structured into three main functional phases:
1.  **Ingestion & Upload:** Monitoring a YouTube RSS feed and transferring the video to VideoDB.
2.  **Audio Processing:** Indexing spoken words and extracting a high-accuracy transcript.
3.  **AI Analysis & Archiving:** Generating a professional article-style digest using AI and creating a documented entry in Notion.

The workflow utilizes asynchronous polling loops (Wait -> HTTP Request -> If) to handle long-running video processing tasks in VideoDB.

---

### 2. Block-by-Block Analysis

#### 2.1 Video Ingestion & Upload
**Overview:** This block triggers whenever a new video is detected on a YouTube RSS feed and begins the upload process to VideoDB.
*   **Nodes Involved:** 
    *   `Watch YouTube RSS Feed`
    *   `Upload Video to VideoDB`
    *   `Wait Before Upload Status Check`
    *   `Poll Upload Status`
    *   `Is Upload Complete`
*   **Node Details:**
    *   **Watch YouTube RSS Feed (Trigger):** Monitors a specific YouTube channel URL (formatted as an RSS feed). It polls at regular intervals.
    *   **Upload Video to VideoDB:** Takes the `link` and `title` from the RSS feed and sends it to VideoDB. It returns a job URL to check for progress.
    *   **Polling Loop (Wait/Poll/If):** 
        *   The `Wait` node pauses execution (default duration).
        *   The `Poll Upload Status` (HTTP Request) queries the `output_url` provided by the upload node.
        *   The `If` node checks if `status == "complete"`. If not, it loops back to the `Wait` node.
*   **Failure Modes:** Invalid RSS URL, VideoDB API rate limits, or video size exceeding VideoDB limits.

#### 2.2 Indexing & Transcription
**Overview:** Once the video is uploaded, this block processes the audio to understand spoken content.
*   **Nodes Involved:** 
    *   `Index Spoken Words`
    *   `Wait Before Indexing Status Check`
    *   `Poll Indexing Status`
    *   `Is Indexing Complete`
    *   `Get Video Transcript`
*   **Node Details:**
    *   **Index Spoken Words:** Triggers VideoDB's speech-to-text indexing engine using the `video_id` from the previous block.
    *   **Polling Loop (Wait/Poll/If):** Similar to the upload loop, this monitors the indexing job until `status == "done"`.
    *   **Get Video Transcript:** A GET request to VideoDB to retrieve the full text of the indexed audio.
*   **Failure Modes:** Poor audio quality preventing indexing, transcription timeout for very long videos.

#### 2.3 AI Synthesis & Notion Integration
**Overview:** The transcript is analyzed by an AI model to create a professional summary, which is then saved to Notion.
*   **Nodes Involved:** 
    *   `Generate AI Digest`
    *   `Wait Before Digest Status Check`
    *   `Poll Digest Status`
    *   `Is Digest Ready`
    *   `Create Notion Page`
*   **Node Details:**
    *   **Generate AI Digest:** Sends the transcript to VideoDB’s AI (using the "ultra" model). The prompt instructs the AI to return a JSON object containing a `title` (max 80 chars) and `text` (max 2000 chars) formatted as a professional article with bolded bullet points.
    *   **Polling Loop (Wait/Poll/If):** Monitors the AI generation job until `status == "complete"`.
    *   **Create Notion Page:** Takes the JSON output from the AI and creates a new page in a designated Notion database. The `title` becomes the page title and the `text` is inserted into the page content.
*   **Failure Modes:** AI failing to follow the JSON format (handled by VideoDB's `response_type: json`), Notion database ID mismatch, or token expiration.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Watch YouTube RSS Feed | RSS Trigger | Trigger | None | Upload Video to VideoDB | Fetching the latest Youtube video from RSS feed and uploading it to VideoDB |
| Upload Video to VideoDB | VideoDB Node | Data Ingestion | Watch YouTube RSS Feed | Wait Before Upload Status Check | Fetching the latest Youtube video from RSS feed and uploading it to VideoDB |
| Wait Before Upload Status Check | Wait | Polling Pause | Upload Video to VideoDB, Is Upload Complete | Poll Upload Status | Fetching the latest Youtube video from RSS feed and uploading it to VideoDB |
| Poll Upload Status | HTTP Request | Status Check | Wait Before Upload Status Check | Is Upload Complete | Fetching the latest Youtube video from RSS feed and uploading it to VideoDB |
| Is Upload Complete | If | Logic Gate | Poll Upload Status | Index Spoken Words, Wait... | Fetching the latest Youtube video from RSS feed and uploading it to VideoDB |
| Index Spoken Words | VideoDB Node | AI Processing | Is Upload Complete | Wait Before Indexing Status Check | Indexing the spoken words in the video and fetching the transcript |
| Wait Before Indexing Status Check | Wait | Polling Pause | Index Spoken Words, Is Indexing Complete | Poll Indexing Status | Indexing the spoken words in the video and fetching the transcript |
| Poll Indexing Status | HTTP Request | Status Check | Wait Before Indexing Status Check | Is Indexing Complete | Indexing the spoken words in the video and fetching the transcript |
| Is Indexing Complete | If | Logic Gate | Poll Indexing Status | Get Video Transcript, Wait... | Indexing the spoken words in the video and fetching the transcript |
| Get Video Transcript | VideoDB Node | Data Retrieval | Is Indexing Complete | Generate AI Digest | Indexing the spoken words in the video and fetching the transcript |
| Generate AI Digest | VideoDB Node | AI Analysis | Get Video Transcript | Wait Before Digest Status Check | Generating digest notes from the transcript and sending it to Notion |
| Wait Before Digest Status Check | Wait | Polling Pause | Generate AI Digest, Is Digest Ready | Poll Digest Status | Generating digest notes from the transcript and sending it to Notion |
| Poll Digest Status | HTTP Request | Status Check | Wait Before Digest Status Check | Is Digest Ready | Generating digest notes from the transcript and sending it to Notion |
| Is Digest Ready | If | Logic Gate | Poll Digest Status | Create Notion Page, Wait... | Generating digest notes from the transcript and sending it to Notion |
| Create Notion Page | Notion | Data Storage | Is Digest Ready | None | Generating digest notes from the transcript and sending it to Notion |

---

### 4. Reproducing the Workflow from Scratch

#### Prerequisites
1.  **VideoDB Account:** Obtain an API Key.
2.  **Notion Account:** Create an internal integration and share a database with it. Obtain the Database ID.
3.  **YouTube RSS Feed:** Locate your target channel's RSS feed (usually `https://www.youtube.com/feeds/videos.xml?channel_id=CHANNEL_ID`).

#### Step-by-Step Setup
1.  **Trigger:** Add the **RSS Feed Read Trigger**. Set the `URL` to your YouTube feed URL. Set the polling frequency (e.g., every 1 hour).
2.  **Upload:** Add a **VideoDB** node. 
    *   Operation: `Upload`. 
    *   Inputs: Use expressions to map `$json.link` to URL and `$json.title` to Name.
3.  **Polling Loop (Upload):**
    *   Add a **Wait** node (set for 30–60 seconds).
    *   Add an **HTTP Request** node. Method: `GET`. URL: Expression referencing the `output_url` from the Upload node.
    *   Add an **If** node. Condition: `status` equals `complete`. 
    *   Connect the "False" path back to the **Wait** node.
4.  **Indexing:** Add a **VideoDB** node. 
    *   Operation: `Index Spoken Words`. 
    *   Inputs: `video_id` and `collection_id` from the successful Upload step.
5.  **Polling Loop (Indexing):** Repeat the Polling Loop logic (Step 3), checking for `status` equals `done`.
6.  **Transcription:** Add a **VideoDB** node.
    *   Operation: `Get Transcript`.
    *   Inputs: `video_id` from the previous successful steps.
7.  **AI Analysis:** Add a **VideoDB** node.
    *   Operation: `Generate Text`.
    *   Model: `ultra`.
    *   Response Type: `json`.
    *   Prompt: Instruct the AI to summarize the transcript into a JSON object with `title` and `text` fields.
8.  **Polling Loop (AI):** Repeat the Polling Loop logic (Step 3), checking for `status` equals `complete`.
9.  **Notion:** Add a **Notion** node.
    *   Resource: `Database Page`, Operation: `Create`.
    *   Database ID: Select your "Notes" database.
    *   Mapping: Map the AI's `title` to the page title and the `text` to a Rich Text block in the page content.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **VideoDB API Documentation** | Used for managing video indexing and AI generation. |
| **YouTube RSS Discovery** | To find a Channel ID, view the page source of a YouTube channel and search for `channelId`. |
| **Prompt Engineering** | The "Generate AI Digest" node requires specific formatting instructions to ensure the output is valid JSON for the Notion node to parse. |
| **Polling Efficiency** | If videos are long, increase the Wait node duration to reduce unnecessary API calls. |