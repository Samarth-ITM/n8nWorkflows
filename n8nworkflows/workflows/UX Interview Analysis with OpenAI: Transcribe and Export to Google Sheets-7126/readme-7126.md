UX Interview Analysis with OpenAI: Transcribe and Export to Google Sheets

https://n8nworkflows.xyz/workflows/ux-interview-analysis-with-openai--transcribe-and-export-to-google-sheets-7126


# UX Interview Analysis with OpenAI: Transcribe and Export to Google Sheets

### 1. Workflow Overview

This workflow automates the processing of UX interview recordings stored in Google Drive by transcribing the audio, analyzing the transcripts with OpenAI’s language models, and exporting structured summaries into Google Sheets. It is designed for user researchers or product teams who want to extract actionable insights from user interviews without manual transcription or analysis.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and File Retrieval:** Trigger the workflow manually and search a designated Google Drive folder for UX interview audio files (MP3 format by default).

- **1.2 Audio Download and Transcription:** Download each audio file and transcribe it into text using OpenAI’s transcription capabilities.

- **1.3 AI Transcript Analysis:** Analyze each transcript with an AI agent that summarizes key UX insights per interview persona into a structured JSON format.

- **1.4 Output Structuring and Export:** Parse the AI’s structured JSON output, split it into individual summary entries, and insert these rows into a Google Sheet for easy review and tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Retrieval

- **Overview:**  
  This block initiates the workflow manually and retrieves all user interview audio files from a specific Google Drive folder.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Search Google Drive for interview files  
  - Filter by .mp3  
  - Sticky Note (instructions for uploading files)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Config: No parameters needed.  
    - Input: None  
    - Output: To "Search Google Drive for interview files"  
    - Edge Cases: Workflow won’t start unless manually triggered.

  - **Search Google Drive for interview files**  
    - Type: Google Drive (List Files/Folders)  
    - Role: Fetches files from a specific Google Drive folder containing interview recordings.  
    - Config: Folder ID set to a fixed folder where interview recordings are stored.  
    - Input: Trigger node output  
    - Output: List of all files in the folder  
    - Credentials: Google Drive OAuth2  
    - Edge Cases: Folder ID incorrect or access denied -> no files returned or auth error.

  - **Filter by .mp3**  
    - Type: Filter node  
    - Role: Filters files to keep only those with names ending in ".mp3" (case-insensitive).  
    - Config: String condition on filename to end with ".mp3".  
    - Input: List of files from Google Drive node  
    - Output: Only MP3 files to continue  
    - Edge Cases: File names with uppercase extension or other audio formats require filter adjustment.  

  - **Sticky Note** (Positioned near this block)  
    - Content: Instructions highlighting the need to upload MP3 files to Google Drive and connect Google Drive credentials.

#### 2.2 Audio Download and Transcription

- **Overview:**  
  Downloads each MP3 file and transcribes its audio content to text using OpenAI's audio transcription feature.

- **Nodes Involved:**  
  - Download audio file  
  - Transcribe a recording

- **Node Details:**

  - **Download audio file**  
    - Type: Google Drive (Download File)  
    - Role: Downloads the MP3 audio files locally in the workflow.  
    - Config: File ID is dynamically set from filtered file list.  
    - Input: Filtered MP3 files  
    - Output: Binary audio data for transcription  
    - Credentials: Google Drive OAuth2  
    - Edge Cases: Download failures due to permissions or file not found.

  - **Transcribe a recording**  
    - Type: OpenAI (Audio Transcription)  
    - Role: Converts audio content into text transcript.  
    - Config: Uses OpenAI audio transcription operation with default options.  
    - Input: Binary data from previous node  
    - Output: JSON with transcript text  
    - Credentials: OpenAI API key  
    - Edge Cases: Audio format unsupported, transcription errors, or API rate limits.

#### 2.3 AI Transcript Analysis

- **Overview:**  
  An AI agent processes the transcript to generate a structured summary of each interview persona’s needs, pain points, and feature requests.

- **Nodes Involved:**  
  - AI Agent for creating transcript  
  - Structured Output Parser  
  - OpenAI Chat Model  
  - Sticky Note (OpenAI usage instructions)

- **Node Details:**

  - **AI Agent for creating transcript**  
    - Type: Langchain Agent (OpenAI)  
    - Role: Runs a prompt-based AI agent to analyze transcript text and produce a JSON array summarizing UX insights per persona.  
    - Config:  
      - Prompt instructs the model to produce a JSON array with keys: Persona, User needs, Pain points, New feature request.  
      - Enforces JSON-only output with no extra text.  
      - Uses the transcript text dynamically injected in prompt (`{{ $json.text }}`).  
      - Output parser enabled for structured JSON.  
    - Input: Transcript text JSON from transcription node  
    - Output: Structured JSON summary  
    - Credentials: OpenAI API key  
    - Edge Cases: Model may produce invalid JSON if prompt is altered, or if transcript is empty. API errors possible.

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Validates and parses the AI agent’s output to ensure it matches the expected JSON schema.  
    - Config: Provided example JSON schema reflecting expected summary entries.  
    - Input: AI Agent output  
    - Output: Cleaned structured JSON data  
    - Edge Cases: Parsing fails if AI output is malformed or not strictly JSON.

  - **OpenAI Chat Model**  
    - Type: Langchain Chat Model (OpenAI GPT-4)  
    - Role: Supports the AI agent’s interaction with the GPT-4.1-mini model to generate the summaries.  
    - Input/Output: Connected internally to AI Agent node  
    - Credentials: OpenAI API  
    - Edge Cases: Model unavailability, network issues.

  - **Sticky Note** (Near AI nodes)  
    - Content: Notes that OpenAI is used for summarization and can be swapped with other LLMs like Gemini or Claude; prompt customization advice.

#### 2.4 Output Structuring and Export

- **Overview:**  
  Splits the structured JSON array into individual summary items and appends them as rows into a Google Sheets document.

- **Nodes Involved:**  
  - Split Out results  
  - Insert results to Google Sheets  
  - Sticky Note (Google Sheets instructions)

- **Node Details:**

  - **Split Out results**  
    - Type: Split Out  
    - Role: Splits the array of summary objects into individual items for insertion.  
    - Config: Field to split is "output" from AI Agent.  
    - Input: Structured JSON summary array  
    - Output: Single JSON objects per item for Google Sheets insertion  
    - Edge Cases: Empty array results in no output lines.

  - **Insert results to Google Sheets**  
    - Type: Google Sheets (Append)  
    - Role: Appends each summary entry as a new row in a configured Google Sheet.  
    - Config:  
      - Columns mapped explicitly to Persona, User need, Pain points, New feature request fields.  
      - Sheet name set to "Summary".  
      - Document ID set to target Google Sheet.  
    - Input: Individual summary entries  
    - Credentials: Google Sheets OAuth2  
    - Edge Cases: Incorrect document or sheet name causes failure; API quota limits.

  - **Sticky Note** (Near Google Sheets node)  
    - Content: Instructions to create Google Sheets with specific columns to match the data.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                             | Input Node(s)                     | Output Node(s)                         | Sticky Note                                                                                                     |
|--------------------------------|--------------------------------------|---------------------------------------------|----------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger                       | Starts the workflow manually                 | None                             | Search Google Drive for interview files |                                                                                                                |
| Search Google Drive for interview files | Google Drive (List files)          | Lists files in specific Drive folder         | When clicking ‘Execute workflow’ | Filter by .mp3                        | Upload your MP3 files to Google Drive and connect Google Drive to the workflow.                                |
| Filter by .mp3                 | Filter                               | Filters to keep only .mp3 audio files        | Search Google Drive for interview files | Download audio file                   | Upload your MP3 files to Google Drive and connect Google Drive to the workflow.                                |
| Download audio file            | Google Drive (Download file)          | Downloads MP3 files for transcription        | Filter by .mp3                   | Transcribe a recording                |                                                                                                                |
| Transcribe a recording         | OpenAI Audio Transcription            | Transcribes audio to text                      | Download audio file              | AI Agent for creating transcript      |                                                                                                                |
| AI Agent for creating transcript | Langchain Agent (OpenAI)             | Summarizes transcript to structured JSON     | Transcribe a recording           | Split Out results                    | OpenAI for summarization (you can replace it with Gemini, Claude, or any other LLM). Modify the summary requirements if needed. |
| Structured Output Parser       | Langchain Structured Output Parser   | Validates/parses AI output JSON                | AI Agent for creating transcript | AI Agent for creating transcript (parser output) |                                                                                                                |
| OpenAI Chat Model              | Langchain Chat Model (OpenAI GPT-4)  | Chat model powering AI agent                   | AI Agent for creating transcript | AI Agent for creating transcript      | OpenAI for summarization (you can replace it with Gemini, Claude, or any other LLM). Modify the summary requirements if needed. |
| Split Out results              | Split Out                            | Splits JSON array into individual entries     | AI Agent for creating transcript | Insert results to Google Sheets       |                                                                                                                |
| Insert results to Google Sheets | Google Sheets (Append)                | Adds summary rows to Google Sheets             | Split Out results               | None                                | Connect Google Sheets to this node. Prior create columns as: • Persona • User Needs • Pain Points • New Feature Requests |
| Sticky Note1                  | Sticky Note                         | Documentation and overview                      | None                            | None                                | ***UX Interview Analysis with OpenAI: Transcipt, Summarize, and Export to Google Sheets!*** ... See full sticky note content. |
| Sticky Note2                  | Sticky Note                         | Advice on AI summarization model                | None                            | None                                | OpenAI for summarization (you can replace it with Gemini, Claude, or any other LLM). Modify the summary requirements if needed. |
| Sticky Note3                  | Sticky Note                         | Google Sheets setup instructions                | None                            | None                                | Connect Google Sheets to this node. Prior create columns as: • Persona • User Needs • Pain Points • New Feature Requests |
| Sticky Note                   | Sticky Note                         | Instructions for Google Drive file uploads     | None                            | None                                | Upload your MP3 files to Google Drive and connect Google Drive to the workflow.                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: *When clicking ‘Execute workflow’*  
   - Purpose: Manual start of the workflow  
   - No parameters needed.

2. **Add Google Drive node to List Files**  
   - Name: *Search Google Drive for interview files*  
   - Operation: List files/folders  
   - Set Folder ID to your specific folder containing interview recordings (e.g., "1_HlRPRZeTx48RE95HYTpaW3YVm-Tk1EG").  
   - Credentials: Set up Google Drive OAuth2 credentials.

3. **Add a Filter node**  
   - Name: *Filter by .mp3*  
   - Condition: File name ends with ".mp3" (case insensitive)  
   - Input: Connect from *Search Google Drive for interview files* node output.

4. **Add Google Drive node to Download File**  
   - Name: *Download audio file*  
   - Operation: Download file  
   - Set File ID dynamically to `{{$json["id"]}}` from filtered files.  
   - Credentials: Use same Google Drive OAuth2 credentials.  
   - Connect input from *Filter by .mp3* output.

5. **Add OpenAI node for Audio Transcription**  
   - Name: *Transcribe a recording*  
   - Resource: Audio  
   - Operation: Transcribe  
   - Input: Use binary data of the downloaded audio file.  
   - Credentials: Provide OpenAI API key with transcription permissions.

6. **Set up the Langchain AI Agent node**  
   - Name: *AI Agent for creating transcript*  
   - Prompt:  
     ```
     You are an expert UX researcher assistant.

     I will provide you with a transcript from a user interview. Analyze the transcript and return a summary for each person as a JSON array. Each object must have these keys:
     - "Persona"
     - "User needs"
     - "Pain points"
     - "New feature request"

     Follow this exact format:
     [
       {
         "Persona": "Person 1",
         "User needs": "Describe the main needs here",
         "Pain points": "Describe main pain points here",
         "New feature request": "Describe new feature requests here"
       },
       {
         "Persona": "Person 2",
         "User needs": " ... ",
         "Pain points": " ... ",
         "New feature request": " ... "
       }
     ]

     Rules:
     - Only return valid JSON.
     - No extra text, explanations, or comments.
     - Use concise summaries.

     Here is the transcript:
     {{$json["text"]}}
     ```
   - Enable structured output parser.  
   - Credentials: OpenAI API key.  
   - Input: Connect from *Transcribe a recording* node output.

7. **Add Langchain Structured Output Parser node**  
   - Name: *Structured Output Parser*  
   - JSON Schema Example: Provide sample JSON with keys Persona, User needs, Pain points, New feature request to validate output.  
   - Connect input from *AI Agent for creating transcript* node’s output parser port.

8. **Add Langchain OpenAI Chat Model node**  
   - Name: *OpenAI Chat Model*  
   - Model: Select GPT-4.1-mini or similar GPT-4 variant.  
   - Credentials: OpenAI API key.  
   - Connect as AI language model for the *AI Agent for creating transcript* node.

9. **Add Split Out node**  
   - Name: *Split Out results*  
   - Field to split: "output" (from AI Agent output)  
   - Connect input from *AI Agent for creating transcript* node.

10. **Add Google Sheets node to Append Rows**  
    - Name: *Insert results to Google Sheets*  
    - Operation: Append  
    - Document ID: ID of your target Google Sheet.  
    - Sheet Name: "Summary" (or your chosen sheet)  
    - Columns mapping:  
      - Persona → `{{$json.Persona}}`  
      - User need → `{{$json["User needs"]}}`  
      - Pain points → `{{$json["Pain points"]}}`  
      - New feature request → `{{$json["New feature request"]}}`  
    - Credentials: Google Sheets OAuth2  
    - Connect input from *Split Out results* node.

11. **Connect all nodes in the following order:**  
    Manual Trigger → Search Google Drive → Filter by .mp3 → Download audio file → Transcribe a recording → AI Agent for creating transcript → Split Out results → Insert results to Google Sheets.

12. **Add sticky notes as documentation inside the canvas:**  
    - Overview and instructions (similar to Sticky Note1).  
    - Google Drive upload instructions near Drive nodes (Sticky Note).  
    - OpenAI summarization notes near AI nodes (Sticky Note2).  
    - Google Sheets column setup near Sheets node (Sticky Note3).

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| UX Interview Analysis with OpenAI: Transcript, Summarize, and Export to Google Sheets. Set up credentials and upload MP3 files to Google Drive. | Main project description inside Sticky Note1 node.                                                          |
| OpenAI is used for both transcription and summarization but can be replaced by other LLMs like Gemini or Claude if preferred. | Sticky Note2 near AI nodes.                                                                                  |
| Create a Google Sheet with columns: Persona, User Needs, Pain Points, New Feature Requests before running the workflow.      | Sticky Note3 near Google Sheets node.                                                                        |
| Google Drive folder must be shared or accessible to the OAuth2 credentials used in the workflow for file listing and download. | Sticky Note near Google Drive nodes.                                                                         |
| Workflow relies on strict JSON output from AI agent; modifying the prompt or output parser may cause parsing errors.          | General caveat for AI Agent node.                                                                             |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.