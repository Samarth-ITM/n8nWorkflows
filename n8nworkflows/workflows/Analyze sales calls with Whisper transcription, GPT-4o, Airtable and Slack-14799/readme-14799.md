Analyze sales calls with Whisper transcription, GPT-4o, Airtable and Slack

https://n8nworkflows.xyz/workflows/analyze-sales-calls-with-whisper-transcription--gpt-4o--airtable-and-slack-14799


# Analyze sales calls with Whisper transcription, GPT-4o, Airtable and Slack

### 1. Workflow Overview

The **Analyze Sales Calls with Whisper Transcription, GPT-4o, Airtable, and Slack** workflow is an end-to-end AI pipeline designed for sales managers and revenue teams. It automates the extraction of insights from sales call recordings, removing the need for manual review.

The workflow allows a sales representative to submit a call recording link via a web form. The system then downloads the audio, converts speech to text, analyzes the conversation for sentiment and deal risk, and distributes the results across a CRM (Airtable), a communication tool (Slack), and a direct feedback page for the user.

**Logical Blocks:**
- **1.1 Form Intake:** Collection and normalization of call metadata and audio links.
- **1.2 Audio Transcription:** Retrieval of the audio file and conversion to text using OpenAI Whisper.
- **1.3 AI Analysis:** Deep processing of the transcript via GPT-4o to extract structured sales insights.
- **1.4 Output and Delivery:** Distribution of results to Airtable, Slack, and the end-user.

---

### 2. Block-by-Block Analysis

#### 2.1 Form Intake
**Overview:** Captures initial data from the sales representative and ensures the data is clean and valid before processing.

- **Nodes Involved:** 
    - `1. Form — Sales Rep Submits Call`
    - `2. Code — Clean and Validate Form Data`
- **Node Details:**
    - **1. Form — Sales Rep Submits Call**: (Form Trigger) A public-facing form collecting Rep Name, Email, Prospect Company, Contact Name/Email, Deal Stage (Dropdown), Deal Value, Audio URL, Duration, and Notes.
    - **2. Code — Clean and Validate Form Data**: (Code) Normalizes text inputs (trimming/lowercase), parses numbers, and generates a unique `submissionId` (e.g., `CALL-171...`).
    - **Failure Types:** Validation errors if required fields (Rep Name, Email, Audio URL) are missing.

#### 2.2 Audio Transcription
**Overview:** Handles the technical process of fetching the remote audio file and converting it into a text transcript.

- **Nodes Involved:** 
    - `3. HTTP — Download Audio File`
    - `4. OpenAI — Transcribe with Whisper`
    - `5. Code — Merge Transcript with Call Data`
- **Node Details:**
    - **3. HTTP — Download Audio File**: (HTTP Request) Fetches the file from the `audioUrl` provided in the form. Configured to return the response as a binary file.
    - **4. OpenAI — Transcribe with Whisper**: (HTTP Request) Sends the binary audio file to the OpenAI `/v1/audio/transcriptions` endpoint using the `whisper-1` model.
    - **5. Code — Merge Transcript with Call Data**: (Code) Joins the Whisper text output back with the original form metadata and calculates the total word count.
    - **Failure Types:** 404 errors on audio URLs, OpenAI API timeouts (set to 180s), or empty transcripts.

#### 2.3 AI Analysis
**Overview:** Uses a Large Language Model (LLM) to transform raw text into a highly structured JSON analysis of the sales call.

- **Nodes Involved:** 
    - `6. AI Agent — GPT-4o Sales Analysis`
    - `7. OpenAI — Chat Model (GPT-4o)`
    - `8. Code — Parse and Combine AI Analysis`
- **Node Details:**
    - **6. AI Agent — GPT-4o Sales Analysis**: (AI Agent) Uses a complex prompt to act as a sales analyst. It extracts: summary, sentiment score, deal risk, talk ratio, objections (with quotes), buying signals, competitors, next steps, a coaching tip, a CRM note, and a follow-up email.
    - **7. OpenAI — Chat Model (GPT-4o)**: (Chat Model) The engine powering the agent, specifically using the `gpt-4o` model.
    - **8. Code — Parse and Combine AI Analysis**: (Code) Cleans the AI's response (removing markdown code fences like ```json) and parses it into a valid JSON object combined with the call metadata.
    - **Failure Types:** LLM hallucinations, JSON parsing errors if the AI returns non-JSON text, or API credit exhaustion.

#### 2.4 Output and Delivery
**Overview:** Synchronizes the analyzed data to a database, notifies the team via Slack, and returns a visual report to the user.

- **Nodes Involved:** 
    - `9. Airtable — Save Full Analysis`
    - `10. Slack — Post Analysis to Channel`
    - `11. Code — Build HTML Confirmation Page`
    - `12. Webhook — Show Results Page to Rep`
- **Node Details:**
    - **9. Airtable — Save Full Analysis**: (Airtable) Creates a record in a specified base. Maps almost every AI-extracted field to specific Airtable columns.
    - **10. Slack — Post Analysis to Channel**: (Slack) Sends a notification via Webhook to a sales channel.
    - **11. Code — Build HTML Confirmation Page**: (Code) Generates a professional HTML layout using the analysis results for immediate display.
    - **12. Webhook — Show Results Page to Rep**: (Respond to Webhook) Returns the generated HTML as the final response to the original form submission.
    - **Failure Types:** Airtable schema mismatches, Slack Webhook expiration, or network timeouts.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Form — Sales Rep Submits Call | Form Trigger | Data Entry | None | 2. Code | Form Intake |
| 2. Code — Clean and Validate Form Data | Code | Data Normalization | 1. Form | 3. HTTP | Form Intake |
| 3. HTTP — Download Audio File | HTTP Request | File Retrieval | 2. Code | 4. OpenAI | Audio Transcription |
| 4. OpenAI — Transcribe with Whisper | HTTP Request | Speech-to-Text | 3. HTTP | 5. Code | Audio Transcription / ⚠️ API Key Exposed in Plain Text |
| 5. Code — Merge Transcript with Call Data | Code | Data Aggregation | 4. OpenAI | 6. AI Agent | Audio Transcription |
| 6. AI Agent — GPT-4o Sales Analysis | AI Agent | Sales Insights Extraction | 5. Code | 8. Code | AI Analysis |
| 7. OpenAI — Chat Model (GPT-4o) | Chat Model | LLM Engine | None | 6. AI Agent | AI Analysis |
| 8. Code — Parse and Combine AI Analysis | Code | JSON Sanitization | 6. AI Agent | 9, 10, 11 | AI Analysis |
| 9. Airtable — Save Full Analysis | Airtable | Record Keeping | 8. Code | None | Output and Delivery |
| 10. Slack — Post Analysis to Channel | Slack | Team Notification | 8. Code | None | Output and Delivery |
| 11. Code — Build HTML Confirmation Page | Code | UI Generation | 8. Code | 12. Webhook | Output and Delivery |
| 12. Webhook — Show Results Page to Rep | Respond to Webhook | End-user Delivery | 11. Code | None | Output and Delivery |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Form Intake
1. Create a **Form Trigger** node. Add fields for Name, Email, Prospect Company, Contact Name, Contact Email, Deal Stage (Dropdown), Deal Value (Number), Call Recording Direct Link (URL), Duration (Number), and Notes (Textarea).
2. Connect a **Code** node. Use JavaScript to trim inputs, convert emails to lowercase, and create a `submissionId` using `Date.now()`.

#### Step 2: Audio Processing
3. Create an **HTTP Request** node. Set the URL to `{{ $json.audioUrl }}` and set the response format to `File`.
4. Create an **HTTP Request** node for OpenAI Whisper.
    - Method: `POST`
    - URL: `https://api.openai.com/v1/audio/transcriptions`
    - Body: `multipart-form-data` with parameters `model: whisper-1`, `response_format: verbose_json`, and `language: en`.
    - Header: `Authorization: Bearer YOUR_API_KEY`.
5. Connect a **Code** node to merge the `whisper.text` with the data from node #2.

#### Step 3: AI Analysis
6. Create an **AI Agent** node. Set the prompt to the "expert sales call analyst" persona. Define the strict JSON output structure (Summary, Sentiment, Risk, Momentum, Objections, etc.).
7. Attach an **OpenAI Chat Model** node to the AI Agent and select `gpt-4o`.
8. Create a **Code** node to parse the agent's string output. Use `.replace()` to remove markdown backticks and `JSON.parse()` to convert it into a usable object.

#### Step 4: Data Distribution
9. Create an **Airtable** node. Use the "Create" operation. Map the AI-parsed fields (e.g., `analysis.summary`, `analysis.deal_risk`) to your table columns.
10. Create a **Slack** node. Configure it using a Webhook URL to post the results to your channel.
11. Create a **Code** node to build a string of HTML. Use template literals to insert the AI results into a styled `<div>`.
12. Create a **Respond to Webhook** node. Set the response body to the HTML output from the previous node and set the response mode to `HTML`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Security Warning** | The OpenAI API key is hard-coded in Node 4. This is a security risk. Replace with n8n "Header Auth" credentials. |
| **Model Version** | Specifically optimized for `gpt-4o` to ensure high-quality JSON extraction. |
| **Audio Requirements** | Requires a direct download link (e.g., S3, Dropbox direct link). Links to players (YouTube, Google Drive) will fail. |