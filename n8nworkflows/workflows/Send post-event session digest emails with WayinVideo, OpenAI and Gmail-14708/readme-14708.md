Send post-event session digest emails with WayinVideo, OpenAI and Gmail

https://n8nworkflows.xyz/workflows/send-post-event-session-digest-emails-with-wayinvideo--openai-and-gmail-14708


# Send post-event session digest emails with WayinVideo, OpenAI and Gmail

### 1. Workflow Overview

The **Post-Event Session Digest Email Sender** is an automated system designed to transform video recordings of event sessions into professional, summarized HTML emails. It leverages WayinVideo for AI-driven video summarization, OpenAI for high-quality email copywriting, and Gmail for distribution.

The workflow is logically divided into five functional blocks:
- **1.1 Input Reception:** A public form collects event metadata, attendee lists, and up to three video recording URLs.
- **1.2 AI Submission:** Video URLs are sent to the WayinVideo API to initiate the summarization process.
- **1.3 Processing Delay:** A mandatory wait period ensures the AI has time to process the video content.
- **1.4 Summary Retrieval:** The workflow fetches the completed summaries using the task IDs provided during submission.
- **1.5 Content Generation & Distribution:** An AI agent aggregates all summaries into a structured HTML email and sends it to the specified attendees.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Captures the necessary data to drive the workflow via a user-facing form.
- **Nodes Involved:** `1. Form — Event Details + Session URLs`
- **Node Details:**
    - **Type:** Form Trigger
    - **Configuration:** Collects Event Name, Sender Name, Attendee Emails (textarea), and Titles/URLs for up to 3 sessions.
    - **Input/Output:** Entry point $\rightarrow$ Triggers the three WayinVideo submission nodes in parallel.
    - **Edge Cases:** If the "Attendee Email Addresses" field is improperly formatted, the final Gmail node may fail.

#### 2.2 AI Submission
- **Overview:** Requests summaries for the provided video URLs from the WayinVideo API.
- **Nodes Involved:** `2. WayinVideo — Submit Session 1`, `3. WayinVideo — Submit Session 2`, `4. WayinVideo — Submit Session 3`
- **Node Details:**
    - **Type:** HTTP Request (POST)
    - **Configuration:** 
        - **URL:** `https://wayinvideo-api.wayin.ai/api/v2/summaries`
        - **Body:** JSON containing `video_url` (mapped from Form) and `target_lang` ("en").
        - **Headers:** Requires `Authorization: Bearer [TOKEN]`, `x-wayinvideo-api-version: v2`, and `Content-Type: application/json`.
    - **Edge Cases:** If Session 2 or 3 URLs are left blank in the form, the API call will likely return an error, stopping the workflow.

#### 2.3 Processing Delay
- **Overview:** Pauses the execution to allow asynchronous AI processing to complete.
- **Nodes Involved:** `5. Wait — 90 Seconds`
- **Node Details:**
    - **Type:** Wait
    - **Configuration:** Set to 90 seconds.
    - **Input/Output:** Triggered after the submission nodes $\rightarrow$ Triggers the retrieval nodes.

#### 2.4 Summary Retrieval
- **Overview:** Polls the WayinVideo API to retrieve the actual text summaries using the IDs generated in Block 2.2.
- **Nodes Involved:** `6. WayinVideo — Get Session 1 Summary`, `7. WayinVideo — Get Session 2 Summary`, `8. WayinVideo — Get Session 3 Summary`
- **Node Details:**
    - **Type:** HTTP Request (GET)
    - **Configuration:** 
        - **URL:** `.../results/{{ [Task ID from submission node] }}`
        - **Headers:** Same authentication headers as the submission nodes.
    - **Edge Cases:** If the video is very long, 90 seconds may be insufficient, resulting in empty data or an "incomplete" status.

#### 2.5 Content Generation & Distribution
- **Overview:** Aggregates data, uses an LLM to draft a professional email, and sends it via Gmail.
- **Nodes Involved:** `9. Merge — All 3 Summaries`, `10. AI Agent — Build Digest Email`, `11. OpenAI — GPT-4o-mini Model1`, `12. Output Parser — Structured HTML Email1`, `13. Gmail — Send Digest Email`
- **Node Details:**
    - **Merge Node:** Combines the three separate summary outputs into a single item for the AI Agent.
    - **AI Agent:** Uses a complex prompt to map event names, summaries, and URLs into a specific HTML template.
    - **OpenAI Model:** Configured to use `gpt-4o-mini`.
    - **Output Parser:** Forces the AI to return a JSON object with the key `html_email`.
    - **Gmail Node:** Sends the `html_email` string as the body, using the `Attendee Email Addresses` from the form as the recipient.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Form... | Form Trigger | Input Data Collection | — | 2, 3, 4 | Section 1 — Form Input |
| 2. WayinVideo... S1 | HTTP Request | Submit Video 1 | 1 | 5 | Section 2 — Submit All 3 Sessions... |
| 3. WayinVideo... S2 | HTTP Request | Submit Video 2 | 1 | 5 | Section 2 — Submit All 3 Sessions... |
| 4. WayinVideo... S3 | HTTP Request | Submit Video 3 | 1 | 5 | Section 2 — Submit All 3 Sessions... |
| 5. Wait... | Wait | Processing Delay | 2, 3, 4 | 6, 7, 8 | Section 3 — Wait 90 Seconds |
| 6. WayinVideo... G1 | HTTP Request | Fetch Summary 1 | 5 | 9 | Section 4 — Fetch All 3 Summaries |
| 7. WayinVideo... G2 | HTTP Request | Fetch Summary 2 | 5 | 9 | Section 4 — Fetch All 3 Summaries |
| 8. WayinVideo... G3 | HTTP Request | Fetch Summary 3 | 5 | 9 | Section 4 — Fetch All 3 Summaries |
| 9. Merge... | Merge | Data Aggregation | 6, 7, 8 | 10 | Section 5 — Merge, Write & Send Email |
| 10. AI Agent... | AI Agent | Email Copywriting | 9 | 13 | Section 5 — Merge, Write & Send Email |
| 11. OpenAI... | Chat Model | LLM Brain | — | 10 | (Internal Model for Agent) |
| 12. Output Parser... | Output Parser | JSON Formatting | — | 10 | (Internal Parser for Agent) |
| 13. Gmail... | Gmail | Final Distribution | 10 | — | Section 5 — Merge, Write & Send Email |

---

### 4. Reproducing the Workflow from Scratch

1.  **Setup Form Trigger:**
    - Create a **Form Trigger** node.
    - Add fields: `Event Name`, `Session 1 Recording URL`, `Session 1 Title`, `Session 2 Recording URL`, `Session 2 Title`, `Session 3 Recording URL`, `Session 3 Title`, `Attendee Email Addresses` (Textarea), and `Sender Name`.

2.  **Configure Submission Logic (WayinVideo):**
    - Create three **HTTP Request** nodes (POST).
    - URL: `https://wayinvideo-api.wayin.ai/api/v2/summaries`.
    - Header: `Authorization: Bearer YOUR_TOKEN_HERE`, `x-wayinvideo-api-version: v2`.
    - Body: JSON $\rightarrow$ Map the corresponding URL from the Form node for each of the three nodes.

3.  **Insert Wait Node:**
    - Add a **Wait** node set to 90 seconds. Connect all three submission nodes to this node.

4.  **Configure Retrieval Logic (WayinVideo):**
    - Create three **HTTP Request** nodes (GET).
    - URL: `https://wayinvideo-api.wayin.ai/api/v2/summaries/results/{{ [Submission Node ID] }}`.
    - Use the same Authorization headers as the submission nodes.

5.  **Aggregate and Generate:**
    - Add a **Merge** node with 3 inputs. Connect the three retrieval nodes.
    - Create an **AI Agent** node.
    - Attach an **OpenAI Chat Model** node (select `gpt-4o-mini`).
    - Attach a **Structured Output Parser** node with the schema: `{"html_email": "string"}`.
    - In the Agent's prompt, define the professional writer persona and the exact HTML structure provided in the original workflow JSON.

6.  **Final Email Send:**
    - Add a **Gmail** node.
    - **Recipient:** Map the `Attendee Email Addresses` field from the Form node.
    - **Subject:** `"Your Complete Session Digest — {{ Event Name }}"`.
    - **Message:** Map the `html_email` output from the AI Agent.
    - **Sender Name:** Map the `Sender Name` field from the Form node.

7.  **Credentials Setup:**
    - Configure **OpenAI API Key**.
    - Configure **Gmail OAuth2** credentials.
    - Replace `YOUR_TOKEN_HERE` in all WayinVideo HTTP nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Warning: Blank URLs** | If Session 2 or 3 are left empty, the workflow will fail. Recommendation: Add an IF node before submission. |
| **Warning: Processing Time** | 90s may not be enough for long videos. Recommendation: Add a retry loop checking if `data.summary` is present. |
| **Customization** | Upgrade to `gpt-4o` in the Model node for higher quality writing. |
| **Scaling** | To add a 4th session, duplicate a submission and retrieval node and increase Merge node inputs to 4. |