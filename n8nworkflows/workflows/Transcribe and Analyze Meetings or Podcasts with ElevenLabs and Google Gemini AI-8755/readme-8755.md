Transcribe and Analyze Meetings or Podcasts with ElevenLabs and Google Gemini AI

https://n8nworkflows.xyz/workflows/transcribe-and-analyze-meetings-or-podcasts-with-elevenlabs-and-google-gemini-ai-8755


# Transcribe and Analyze Meetings or Podcasts with ElevenLabs and Google Gemini AI

---

### 1. Workflow Overview

This workflow automates the transcription and analysis of audio or video content such as meetings, podcasts, or interviews using ElevenLabs’ Speech-to-Text API combined with Google Gemini AI for advanced transcript analysis. It is designed for scenarios requiring accurate speech transcription followed by insightful summarization and extraction of key information.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception and Data Retrieval**: Fetches audio/video URLs from a Google Sheet to process.
- **1.2 Speech-to-Text Processing**: Converts audio content to text transcripts via ElevenLabs API.
- **1.3 Document Management**: Creates and updates Google Docs with transcripts.
- **1.4 Transcript Analysis**: Uses Google Gemini AI to analyze and summarize the transcript.
- **1.5 Output and Notification**: Converts analysis to HTML and sends an email summary.
- **1.6 Batch Processing and Loop Control**: Manages batch processing of multiple audio URLs.
- **1.7 Workflow Triggers**: Supports both manual and external workflow-triggered executions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Retrieval

**Overview:**  
This block initiates the workflow either manually or via another workflow and retrieves a list of audio/video URLs from a Google Sheet for processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Get audio urls (Google Sheets)  
- Loop Over Items (Split In Batches)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow via user action in n8n UI.  
  - *Connections:* Outputs to "Get audio urls".  
  - *Failure modes:* None typical; manual start.

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Allows external workflows to trigger this workflow and pass a "text" parameter for direct analysis.  
  - *Connections:* Outputs to "Transcript Analysis".  
  - *Failure modes:* Missing or malformed input parameters.

- **Get audio urls**  
  - *Type:* Google Sheets node  
  - *Role:* Reads a Google Sheet to fetch rows containing audio URLs to transcribe.  
  - *Configuration:*  
    - Filters rows by the "DONE" column (likely to exclude already processed entries).  
    - Reads from a specific Spreadsheet and sheet (gid=0).  
  - *Credentials:* Google Sheets OAuth2.  
  - *Outputs:* Passes data to "Loop Over Items".  
  - *Failures:* Credential expiration, API limits, spreadsheet access issues.

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes audio URLs one at a time to manage API rate limits and orderly execution.  
  - *Connections:* Main output to "Speech-to-Text". Returns empty batch to "Speech-to-Text" for finalization.  
  - *Failures:* Batch size misconfiguration or missing data.

---

#### 2.2 Speech-to-Text Processing

**Overview:**  
This block sends audio URLs to ElevenLabs API for transcription, handling conversion from audio/video to text.

**Nodes Involved:**  
- Speech-to-Text (HTTP Request)  
- Create Doc (Google Docs)  
- Add Transcript to Doc (Google Docs)  
- Update row (Google Sheets)  
- Call 'ElevenLabs Text-to-Speech' (Execute Workflow)

**Node Details:**

- **Speech-to-Text**  
  - *Type:* HTTP Request  
  - *Role:* Calls ElevenLabs Speech-to-Text API to transcribe audio from provided URLs.  
  - *Configuration:*  
    - POST request to `https://api.elevenlabs.io/v1/speech-to-text`.  
    - Form-urlencoded body containing: model_id (`scribe_v1`), cloud_storage_url (dynamic from current item's AUDIO URL), tag_audio_events (`true`).  
    - Authenticated by HTTP header with API key (`xi-api-key`).  
  - *Inputs:* Audio URL from "Loop Over Items".  
  - *Outputs:* JSON with transcription text.  
  - *Failures:* Invalid API key, network timeout, unsupported audio format, API limits.

- **Create Doc**  
  - *Type:* Google Docs  
  - *Role:* Creates a new Google Doc to store the transcript.  
  - *Configuration:*  
    - Title set dynamically from transcription ID.  
    - Specific Google Drive folder ID assigned.  
  - *Credentials:* Google Docs OAuth2.  
  - *Inputs:* Transcription metadata from "Speech-to-Text".  
  - *Outputs:* Document ID used downstream.  
  - *Failures:* Insufficient permissions, folder not found.

- **Add Transcript to Doc**  
  - *Type:* Google Docs  
  - *Role:* Inserts the transcript text into the newly created Google Doc.  
  - *Configuration:* Inserts text from Speech-to-Text node into the document.  
  - *Inputs:* Document ID from "Create Doc", transcript text.  
  - *Failures:* Document access errors.

- **Update row**  
  - *Type:* Google Sheets  
  - *Role:* Marks the row as processed ("DONE" = "x") and stores the created Google Doc ID.  
  - *Configuration:* Updates columns "DONE", "DOCS ID", using row_number for matching.  
  - *Inputs:* Row number from batch, Doc ID from "Create Doc".  
  - *Credentials:* Google Sheets OAuth2.  
  - *Failures:* Sheet access errors or concurrent modification.

- **Call 'ElevenLabs Text-to-Speech'**  
  - *Type:* Execute Workflow  
  - *Role:* Invokes a sub-workflow (ID: 7igUJFLIXoP9aAKl) for text-to-speech conversion of the transcript text.  
  - *Configuration:* Runs in "each" mode, waits for sub-workflow completion.  
  - *Inputs:* Text from "Speech-to-Text".  
  - *Failures:* Sub-workflow errors or misconfiguration.

---

#### 2.3 Transcript Analysis

**Overview:**  
This block processes the raw transcript text using Google Gemini AI (PaLM) to generate structured analysis and summaries.

**Nodes Involved:**  
- Transcript Analysis (Google Gemini AI)  
- Markdown (Markdown to HTML conversion)  
- Send email (Gmail)

**Node Details:**

- **Transcript Analysis**  
  - *Type:* Google Gemini (PaLM) AI node  
  - *Role:* Sends transcript text to Google Gemini model `models/gemini-2.5-pro` for in-depth analysis and summary.  
  - *Configuration:*  
    - System prompt defines detailed instructions for analyzing meetings, podcasts, interviews, extracting key points, decisions, action items, quotes, etc.  
    - Input message includes full transcript text.  
  - *Credentials:* Google Palm API (Gemini) OAuth2.  
  - *Failures:* API quota limits, malformed input, response latency.

- **Markdown**  
  - *Type:* Markdown converter  
  - *Role:* Converts Gemini AI’s markdown-formatted summary into HTML for email readability.  
  - *Inputs:* AI-generated content from "Transcript Analysis".  
  - *Failures:* Formatting errors if AI output is malformed.

- **Send email**  
  - *Type:* Gmail node  
  - *Role:* Sends the final analyzed summary to a configured email address.  
  - *Configuration:*  
    - Recipient email set via parameter `YOUR_EMAIL_ADDRESS` (to be customized).  
    - Email subject fixed as "=[N8N] Recap".  
    - Sends plain text email with HTML content from "Markdown" node.  
  - *Credentials:* Gmail OAuth2.  
  - *Failures:* Authentication errors, quota limits, invalid recipient address.

---

#### 2.4 Batch Processing and Loop Control

**Overview:**  
Manages iterative processing of multiple audio URLs to ensure orderly execution and avoid API overload.

**Nodes Involved:**  
- Loop Over Items  
- Call 'ElevenLabs Text-to-Speech' (Execute Workflow)

**Node Details:**

- **Loop Over Items**  
  - As previously described, splits input into manageable batches.  

- **Call 'ElevenLabs Text-to-Speech'**  
  - Invoked after updating the Google Sheet row to perform additional text-to-speech processing asynchronously for each transcript.

---

#### 2.5 Workflow Triggers

**Overview:**  
Allows two modes of triggering: manual execution and external workflow-based invocation.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- When Executed by Another Workflow

**Node Details:**

- *Manual Trigger* is for testing or on-demand runs.  
- *Execute Workflow Trigger* allows integration with other automation pipelines, passing transcript texts directly for analysis.

---

### 3. Summary Table

| Node Name                     | Node Type                   | Functional Role                                  | Input Node(s)                    | Output Node(s)                           | Sticky Note                                                                                                      |
|-------------------------------|-----------------------------|-------------------------------------------------|---------------------------------|-----------------------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Start workflow manually                          | —                               | Get audio urls                          | ## Transcribe and Analyze Meetings or Podcasts (audio/video) using ElevenLabs Speech-to-Text ...                 |
| Get audio urls                 | Google Sheets               | Retrieve audio URLs from Google Sheet            | When clicking ‘Execute workflow’ | Loop Over Items                         | ## STEP 1 ... Register for FREE to [Elevenlabs](https://try.elevenlabs.io/ahkbf00hocnu) ...                      |
| Loop Over Items               | Split In Batches             | Batch processing of audio URLs                    | Get audio urls                  | Speech-to-Text (main batch), Speech-to-Text (empty batch) |                                                                                                                  |
| Speech-to-Text                | HTTP Request                | Transcribe audio via ElevenLabs API               | Loop Over Items                 | Create Doc                              |                                                                                                                  |
| Create Doc                   | Google Docs                 | Create Google Doc for transcript                   | Speech-to-Text                 | Add Transcript to Doc                   |                                                                                                                  |
| Add Transcript to Doc         | Google Docs                 | Insert transcript text into Google Doc            | Create Doc                    | Update row                             |                                                                                                                  |
| Update row                   | Google Sheets               | Mark row as done and store Doc ID                  | Add Transcript to Doc          | Call 'ElevenLabs Text-to-Speech'       |                                                                                                                  |
| Call 'ElevenLabs Text-to-Speech' | Execute Workflow           | Sub-workflow call for text-to-speech conversion   | Update row                    | Loop Over Items                        |                                                                                                                  |
| When Executed by Another Workflow | Execute Workflow Trigger  | Trigger workflow externally with transcript text | —                             | Transcript Analysis                    | ## STEP 2 ... Set YOUR_EMAIL_ADDRESS in "Send email" node                                                        |
| Transcript Analysis          | Google Gemini AI             | Analyze and summarize transcript using AI         | When Executed by Another Workflow | Markdown                              |                                                                                                                  |
| Markdown                    | Markdown                    | Convert AI markdown summary to HTML                | Transcript Analysis            | Send email                            |                                                                                                                  |
| Send email                  | Gmail                       | Send analyzed summary email                         | Markdown                      | —                                     |                                                                                                                  |
| Sticky Note                 | Sticky Note                 | Informational note                                 | —                             | —                                     | ## Transcribe and Analyze Meetings or Podcasts (audio/video) using ElevenLabs Speech-to-Text ...                 |
| Sticky Note1                | Sticky Note                 | Instructions for setup and sheet cloning          | —                             | —                                     | ## STEP 1 ... Register for FREE to [Elevenlabs](https://try.elevenlabs.io/ahkbf00hocnu) ...                      |
| Sticky Note2                | Sticky Note                 | Label for transcript section                        | —                             | —                                     | ## Transcript                                                                                                    |
| Sticky Note3                | Sticky Note                 | Label for analysis section                          | —                             | —                                     | ## Analysis                                                                                                      |
| Sticky Note4                | Sticky Note                 | Reminder to set email address                       | —                             | —                                     | ## STEP 2 ... Set YOUR_EMAIL_ADDRESS in "Send email" node                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Allows manual start.

2. **Add a Google Sheets Node to Read Audio URLs**  
   - Name: `Get audio urls`  
   - Operation: Read rows from your specific Google Sheet (use your spreadsheet ID and sheet gid).  
   - Filter: Exclude rows where "DONE" is marked.  
   - Credentials: Configure Google Sheets OAuth2.

3. **Add a Split In Batches Node**  
   - Name: `Loop Over Items`  
   - Purpose: Process audio URLs sequentially or in manageable batches.

4. **Add HTTP Request Node for ElevenLabs Speech-to-Text**  
   - Name: `Speech-to-Text`  
   - Method: POST to `https://api.elevenlabs.io/v1/speech-to-text`  
   - Content-Type: `application/x-www-form-urlencoded`  
   - Body Parameters:  
     - `model_id`: `scribe_v1`  
     - `cloud_storage_url`: Set dynamically to current item’s AUDIO URL  
     - `tag_audio_events`: `true`  
   - Authentication: HTTP Header with key name `xi-api-key` and your ElevenLabs API key.

5. **Add Google Docs Node to Create Document**  
   - Name: `Create Doc`  
   - Title: Use a dynamic field (e.g., transcription ID or timestamp)  
   - Folder: Specify your Google Drive folder ID  
   - Credentials: Google Docs OAuth2.

6. **Add Google Docs Node to Insert Transcript Text**  
   - Name: `Add Transcript to Doc`  
   - Operation: Update document content by inserting the transcription text from the previous node.

7. **Add Google Sheets Node to Update the Processed Row**  
   - Name: `Update row`  
   - Operation: Mark "DONE" with "x", store the created Google Doc ID, match by row_number.  
   - Credentials: Google Sheets OAuth2.

8. **Add Execute Workflow Node to Call Sub-Workflow for Text-to-Speech**  
   - Name: `Call 'ElevenLabs Text-to-Speech'`  
   - Configuration: Pass the transcript text as input parameter `text`.  
   - Mode: Each (process items individually), wait for sub-workflow completion.  
   - Workflow ID: Use your sub-workflow's ID for text-to-speech.

9. **Establish Connections:**  
   - Connect manual trigger → Google Sheets (Get audio urls) → Split In Batches (Loop Over Items) → Speech-to-Text → Create Doc → Add Transcript to Doc → Update row → Call 'ElevenLabs Text-to-Speech' → Loop Over Items (for next batch).

10. **Add Execute Workflow Trigger Node**  
    - Name: `When Executed by Another Workflow`  
    - Accept input parameter named `text`.  
    - Connect output to `Transcript Analysis`.

11. **Add Google Gemini AI Node**  
    - Name: `Transcript Analysis`  
    - Model: `models/gemini-2.5-pro`  
    - System Message: Use the provided detailed prompt for meeting/podcast transcript analysis.  
    - Input Message: Full transcript text from input.  
    - Credentials: Google Palm API OAuth2.

12. **Add Markdown Node**  
    - Name: `Markdown`  
    - Mode: Markdown to HTML conversion.  
    - Input: Output from `Transcript Analysis`.

13. **Add Gmail Node**  
    - Name: `Send email`  
    - Recipient: Set your email address in the sendTo parameter.  
    - Subject: "=[N8N] Recap"  
    - Message: Use HTML content from Markdown node.  
    - Credentials: Gmail OAuth2.

14. **Connect `When Executed by Another Workflow` → `Transcript Analysis` → `Markdown` → `Send email`.**

15. **Add Sticky Notes for Documentation** (Optional but recommended)  
    - Add notes describing setup steps, usage instructions, and labeling sections.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow supports transcription of audio/video files up to 3 GB and 10 hours duration using ElevenLabs API. It supports multilingual recognition across 99 languages.                                                                                                                                                                                                                                                                                                                                                                                                                | ElevenLabs Speech-to-Text API official features.                                                                |
| Register for a free ElevenLabs account at [ElevenLabs](https://try.elevenlabs.io/ahkbf00hocnu). Clone the sample Google Sheet for input data at [Google Sheet Clone](https://docs.google.com/spreadsheets/d/1046e1OLfwiu1vfvnX_i2LP3yXOGL_87_BxyiXgkVmuU/edit?usp=sharing).                                                                                                                                                                                                                                                | Setup Instructions Sticky Note.                                                                                  |
| Set the HTTP header authentication in the Speech-to-Text node with header name `xi-api-key` and your ElevenLabs API key.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Credential configuration details.                                                                                |
| Customize the email recipient address in the “Send email” node to receive the transcript analysis summaries.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Reminder Sticky Note.                                                                                            |
| The Google Gemini AI system prompt is crafted to extract detailed meeting insights including decisions, action items, participants, and next steps. It uses structured markdown formatting for clarity and readability.                                                                                                                                                                                                                                                                                                                                                                                                    | AI prompt customization for transcript analysis.                                                                |

---

**Disclaimer:**  
The text and data processed by this workflow are handled strictly according to current content policies and contain no illegal or offensive elements. All data is legal and publicly accessible.

---