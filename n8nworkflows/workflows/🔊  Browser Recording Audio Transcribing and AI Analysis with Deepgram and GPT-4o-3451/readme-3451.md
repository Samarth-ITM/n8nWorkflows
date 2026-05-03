🔊  Browser Recording Audio Transcribing and AI Analysis with Deepgram and GPT-4o

https://n8nworkflows.xyz/workflows/----browser-recording-audio-transcribing-and-ai-analysis-with-deepgram-and-gpt-4o-3451


# 🔊  Browser Recording Audio Transcribing and AI Analysis with Deepgram and GPT-4o

### 1. Workflow Overview

The **Transcript Evalu8r V2** workflow is a comprehensive, browser-based audio transcription and AI-powered analysis tool. It enables users to record or upload audio directly in their browser, transcribe it using Deepgram’s speech-to-text API, and perform advanced AI analysis with GPT-4o through LangChain nodes integrated in n8n. The workflow stores and organizes files on Google Drive, generates detailed AI-driven summaries and action items in Google Docs, and provides interactive transcript exploration.

**Target Use Cases:** Researchers, podcasters, legal teams, customer support, market analysts, and any professionals needing rich audio transcript analysis with sentiment, topic extraction, and speaker insights.

**Logical Blocks in the Workflow:**

- **1.1 Input Reception & Storage**
  - Handles browser audio recording uploads and Google Drive folder monitoring.
- **1.2 Audio Processing & Transcription**
  - Downloads audio from Google Drive and sends to Deepgram API.
- **1.3 Transcript JSON Handling & Preparation**
  - Processes Deepgram JSON output, structures data for AI analysis.
- **1.4 AI Analysis & Summarization**
  - Runs the transcript through OpenAI Chat Model (GPT-4o) using LangChain agent, parses output.
- **1.5 Output Generation & File Management**
  - Creates Google Docs with AI insights; moves and organizes files in Google Drive.
- **1.6 Webhook and UI Interaction**
  - Manages webhook endpoints for browser interactions including recording uploads and transcript retrieval.
- **1.7 Setup Nodes**
  - Folder and credential pre-configuration nodes triggered manually to prepare Google Drive structure before usage.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Storage  
**Overview:**  
This block collects audio files from browser uploads via webhooks and monitors relevant Google Drive folders where the recorded audio is stored. It initiates the workflow as soon as new audio files are detected.

**Nodes involved:**  
- Webhook4  
- Google Drive uploadAudio  
- Respond to Webhook4  
- Google Drive downloadUploaded  
- Google Drive Trigger  
- Google Drive Download Audio  

**Node Details:**  

- **Webhook4**  
  - *Type:* Webhook (Trigger node)  
  - *Role:* Receives POST requests from browser audio recording or uploads.  
  - *Config:* Unique webhook ID, no additional auth.  
  - *Input:* Incoming HTTP request from browser UI.  
  - *Output:* Passes data to Google Drive uploadAudio.  
  - *Failures:* Network issues or webhook not called correctly delay audio processing.  
- **Google Drive uploadAudio**  
  - *Type:* Google Drive - Upload  
  - *Role:* Saves uploaded audio file to a designated Google Drive folder.  
  - *Config:* Folder ID set for the audio upload destination.  
  - *Input:* Audio file binary from webhook.  
  - *Output:* Two outputs: confirmation response to webhook, and file metadata to Google Drive downloadUploaded.  
  - *Failures:* Auth failures with Google API, quota limits, upload errors.  
- **Respond to Webhook4**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends confirmation response to the browser that upload succeeded.  
  - *Input:* From uploadAudio node.  
  - *Output:* HTTP 200 response to client.  
- **Google Drive downloadUploaded**  
  - *Type:* Google Drive - Download  
  - *Role:* Retrieves uploaded audio file binary for transcription.  
  - *Config:* Uses file ID from uploadAudio.  
  - *Output:* Binary file to DeepGram.  
- **Google Drive Trigger**  
  - *Type:* Google Drive Trigger (Start node)  
  - *Role:* Watches the “Completed Audio” folder for any new audio files uploaded manually or otherwise.  
  - *Output:* File metadata to Google Drive Download Audio.  
  - *Failures:* Incorrect folder config could cause missed triggers.  
- **Google Drive Download Audio**  
  - *Type:* Google Drive - Download  
  - *Role:* Downloads audio binary for Deepgram transcription.  
  - *Input:* From Google Drive Trigger.  

---

#### 1.2 Audio Processing & Transcription  
**Overview:**  
This block sends audio files to Deepgram for speech-to-text transcription and initial processing.

**Nodes involved:**  
- DeepGram  
- ProcessJSON  
- Merge  

**Node Details:**  

- **DeepGram**  
  - *Type:* HTTP Request  
  - *Role:* Sends audio binary to Deepgram API using the Deepgram API key for transcription and AI analysis.  
  - *Config:* POST request to Deepgram endpoint, including proper headers and parameters for detailed transcripts with speaker detection.  
  - *Input:* Audio binary from Google Drive Download Audio or downloadUploaded.  
  - *Output:* JSON transcript data to ProcessJSON and Merge node (merge for later integration).  
  - *Failures:* API key invalid, download issue, network timeouts, Deepgram quota exceeded.  
- **ProcessJSON**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Processes and formats Deepgram JSON transcript into a cleaner structure, extracting relevant transcript details for later use.  
  - *Input:* JSON from DeepGram.  
  - *Output:* Structured JSON used to create inputs for AI agent.  
  - *Failures:* Malformed JSON, unexpected data structure, missing fields.  
- **Merge**  
  - *Type:* Merge node  
  - *Role:* Combines Deepgram transcript info with processed data for downstream consumption (e.g., AI Agent and audio file renaming).  
  - *Input:* JSON transcript and processed JSON.  
  - *Output:* Passes merged data to subsequent nodes.  

---

#### 1.3 Transcript JSON Handling & Preparation  
**Overview:**  
This block refines transcript JSON, sets fields needed for AI analysis, and prepares data flow to trigger LangChain AI operations.

**Nodes involved:**  
- SetJSONFields  
- AI Agent  

**Node Details:**  

- **SetJSONFields**  
  - *Type:* Set node  
  - *Role:* Defines or modifies keys in the JSON object to structure input for AI Agent. For example, setting prompt templates, metadata, or controlling AI parameters.  
  - *Input:* Processed JSON from ProcessJSON / Merge.  
  - *Output:* Cleaned and appropriately structured data for AI consumption.  
- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Executes AI prompt workflows using OpenAI GPT-4o model to analyze transcript - generates summaries, sentiment, key points, and action items.  
  - *Config:* Connects under the hood with OpenAI Chat Model (GPT-4o) node and Structured Output Parser.  
  - *Input:* Prepared JSON with transcript and parameters.  
  - *Output:* AI-generated structured insights for Google Docs creation and further processing.  
  - *Failures:* API quota exceeded, invalid prompt, timeouts, parsing errors from output parser.  

---

#### 1.4 AI Analysis & Summarization  
**Overview:**  
Handles detailed AI processing and parsing of the language model’s output and captures speaker insights.

**Nodes involved:**  
- OpenAI Chat Model  
- Structured Output Parser  
- GetSpeakers  
- Convert to File1  
- Upload Json  

**Node Details:**  

- **OpenAI Chat Model**  
  - *Type:* LangChain LM Chat OpenAI  
  - *Role:* Calls OpenAI API with GPT-4o model using conversation context and prompts set by AI Agent.  
  - *Input:* Chat request from AI Agent.  
  - *Output:* Model response sent to Structured Output Parser.  
- **Structured Output Parser**  
  - *Type:* LangChain Output Parser  
  - *Role:* Parses AI output into structured JSON with defined sections: summary, sentiment, key points, etc.  
  - *Input:* Raw AI output.  
  - *Output:* Structured JSON for downstream use in Docs and data storage.  
- **GetSpeakers**  
  - *Type:* Code node  
  - *Role:* Extracts speaker names and metadata from transcript JSON (e.g., speaker mapping).  
  - *Output:* Speaker info passed for file conversion.  
- **Convert to File1**  
  - *Type:* Convert to File  
  - *Role:* Converts structured JSON AI output into a file format suitable for Google Drive upload.  
- **Upload Json**  
  - *Type:* Google Drive Upload  
  - *Role:* Saves AI output JSON file in Google Drive summaries folder for archival.  

---

#### 1.5 Output Generation & File Management  
**Overview:**  
Creates Google Docs with AI insights and transcript, organizes processed files into appropriate folders on Google Drive.

**Nodes involved:**  
- RenameAudio  
- MoveAudio  
- Create Google Docs  
- Google Docs1  
- MoveTranscriptAnalysis  
- Merge1  
- Gmail  

**Node Details:**  

- **RenameAudio**  
  - *Type:* Google Drive  
  - *Role:* Renames the original audio file, e.g., appending status or date stamp for clarity.  
  - *Input:* Audio metadata from Merge.  
- **MoveAudio**  
  - *Type:* Google Drive  
  - *Role:* Moves the renamed audio file to a “Completed Audio” folder signaling workflow completion.  
  - *Output:* Triggers Google Docs creation.  
- **Create Google Docs**  
  - *Type:* Google Docs  
  - *Role:* Creates a Google Document containing AI-generated transcript summaries, key points, and full transcript.  
  - *Input:* AI processed text and transcript data.  
  - *Output:* Passes doc metadata to Google Docs1 for advanced operations.  
- **Google Docs1**  
  - *Type:* Google Docs  
  - *Role:* Additional processing or revision of created Google Doc, e.g., setting permissions or formatting.  
  - *Output:* Triggers MoveTranscriptAnalysis.  
- **MoveTranscriptAnalysis**  
  - *Type:* Google Drive  
  - *Role:* Moves final Google Doc to “Transcript Analysis” folder.  
  - *Output:* Feeds merge to trigger Gmail notification.  
- **Merge1**  
  - *Type:* Merge  
  - *Role:* Joins Google Drive final folder file and JSON upload nodes for notification.  
- **Gmail**  
  - *Type:* Gmail Node  
  - *Role:* Sends notification emails containing links to transcript and analyses.  

---

#### 1.6 Webhook and UI Interaction  
**Overview:**  
Manages multiple webhook endpoints servicing browser front-end: receiving uploads, retrieving transcripts/JSONs, and responding with status or streaming content.

**Nodes involved:**  
- Webhook1  
- HTML  
- Respond to Webhook2  
- Webhook2  
- GetJSONs  
- Respond to Webhook1  
- Webhook3  
- GetJSON  
- ExtractJSON  
- Respond to Webhook  

**Node Details:**  

- **Webhook1** → **HTML** → **Respond to Webhook2**  
  - Serves initial HTML pages or UI components for the web app.  
- **Webhook2** → **GetJSONs** → **Respond to Webhook1**  
  - Fetches list of available JSON transcript files on Google Drive, sends to frontend.  
- **Webhook3** → **GetJSON** → **ExtractJSON** → **Respond to Webhook**  
  - Retrieves specific JSON transcript file content and returns as JSON to the requesting client.  

---

#### 1.7 Setup Nodes  
**Overview:**  
Manual trigger nodes used for initial setup or testing, creating necessary Google Drive folders and folder references for workflow to use.

**Nodes involved:**  
- When clicking ‘Test workflow’  
- Setup - Create Transcript-Evalu8r Folder  
- SETUP - TE Summaries  
- SETUP - TE Completed Audio  
- SETUP - TE Transcript_json  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts folder creation process.  
- **Setup - Create Transcript-Evalu8r Folder**  
  - *Type:* Google Drive  
  - *Role:* Creates main working folder structure in Google Drive.  
- **SETUP - TE Summaries**, **TE Completed Audio**, **TE Transcript_json**  
  - *Type:* Google Drive nodes  
  - *Role:* Create sub-folders for storing summaries, finalized audio, and JSON files respectively.

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                          | Input Node(s)                       | Output Node(s)                | Sticky Note                     |
|-------------------------------|--------------------------------|----------------------------------------|-----------------------------------|------------------------------|--------------------------------|
| Google Drive Trigger           | Google Drive Trigger            | Initiate workflow on new audio files   | None                              | Google Drive Download Audio   |                                |
| Google Drive Download Audio    | Google Drive                   | Download audio file for transcription  | Google Drive Trigger              | DeepGram                     |                                |
| Webhook4                      | Webhook                        | Receives browser audio upload           | HTTP Request                     | Google Drive uploadAudio      |                                |
| Google Drive uploadAudio       | Google Drive                   | Uploads audio file to Drive folder      | Webhook4                        | Respond to Webhook4, Google Drive downloadUploaded |                                |
| Respond to Webhook4            | Respond to Webhook             | Confirms audio upload success           | Google Drive uploadAudio         | None                        |                                |
| Google Drive downloadUploaded  | Google Drive                   | Downloads uploaded audio for processing | Google Drive uploadAudio          | DeepGram                     |                                |
| DeepGram                      | HTTP Request                   | Sends audio for transcription           | Google Drive Download Audio / downloadUploaded | ProcessJSON, Merge           |                                |
| ProcessJSON                   | Code                          | Process & clean DeepGram JSON           | DeepGram                        | SetJSONFields                |                                |
| Merge                        | Merge                         | Combine transcript and processed JSON   | DeepGram, ProcessJSON            | AI Agent, RenameAudio        |                                |
| SetJSONFields                | Set                           | Set/prepare JSON fields for AI input    | ProcessJSON                  | AI Agent                    |                                |
| AI Agent                     | LangChain Agent               | Runs GPT-4o AI analysis & summarization | SetJSONFields                | Merge, RenameAudio           |                                |
| RenameAudio                  | Google Drive                  | Rename original audio file after processing | Merge                         | MoveAudio                   |                                |
| MoveAudio                    | Google Drive                  | Move audio file to Completed Audio folder | RenameAudio                   | Create Google Docs           |                                |
| Create Google Docs           | Google Docs                   | Create Google Doc with AI transcript & insights | MoveAudio                   | Google Docs1                |                                |
| Google Docs1                 | Google Docs                   | Further process or finalize Google Doc  | Create Google Docs              | MoveTranscriptAnalysis       |                                |
| MoveTranscriptAnalysis       | Google Drive                  | Organize Google Doc into Analysis folder | Google Docs1                 | Merge1                      |                                |
| Merge1                      | Merge                         | Combine final file outputs for notification | Upload Json, MoveTranscriptAnalysis | Gmail                    |                                |
| Gmail                       | Gmail                         | Notify users via email about transcript | Merge1                        | None                        |                                |
| Convert to File1             | Convert to File               | Convert AI JSON output to file           | GetSpeakers                    | Upload Json                 |                                |
| Upload Json                 | Google Drive                  | Upload JSON analysis file                 | Convert to File1              | Merge1                      |                                |
| GetSpeakers                 | Code                          | Extract speaker info from transcript JSON | Merge                      | Convert to File1            |                                |
| Webhook1                    | Webhook                       | Serve UI HTML for app front-end           | HTTP request                   | HTML                        |                                |
| HTML                        | HTML                          | Serve HTML page for transcript app         | Webhook1                     | Respond to Webhook2          |                                |
| Respond to Webhook2          | Respond to Webhook             | Deliver HTML response to browser          | HTML                        | None                        |                                |
| Webhook2                    | Webhook                       | Request list of JSON transcript files      | HTTP Request                 | GetJSONs                    |                                |
| GetJSONs                    | Google Drive                  | Get list of JSON transcripts               | Webhook2                     | Respond to Webhook1          |                                |
| Respond to Webhook1          | Respond to Webhook             | Send JSON list response for UI             | GetJSONs                     | None                        |                                |
| Webhook3                    | Webhook                       | Request specific JSON transcript content   | HTTP Request                 | GetJSON                     |                                |
| GetJSON                     | Google Drive                  | Download specific JSON transcript file      | Webhook3                     | ExtractJSON                 |                                |
| ExtractJSON                 | Extract from File             | Extract JSON data from downloaded file      | GetJSON                      | Respond to Webhook           |                                |
| Respond to Webhook           | Respond to Webhook             | Return extracted JSON data to client        | ExtractJSON                  | None                        |                                |
| When clicking ‘Test workflow’| Manual Trigger               | Manually start setup of Drive folders        | None                        | Setup - Create Transcript-Evalu8r Folder |                                |
| Setup - Create Transcript-Evalu8r Folder | Google Drive    | Create root folder structure for workflow    | When clicking ‘Test workflow’ | SETUP - TE Summaries         |                                |
| SETUP - TE Summaries        | Google Drive                  | Create summaries subfolder                      | Setup - Create Transcript-Evalu8r Folder | SETUP - TE Completed Audio |                                |
| SETUP - TE Completed Audio | Google Drive                  | Create completed audio folder                   | SETUP - TE Summaries         | SETUP - TE Transcript_json   |                                |
| SETUP - TE Transcript_json | Google Drive                   | Create JSON transcript storage folder            | SETUP - TE Completed Audio  | None                        |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Setup Google Drive Credentials** in n8n for all Google Drive nodes. Also, setup OpenAI credentials for Chat Models and HTTP credential for Deepgram API.

2. **Create Setup Folders:**
   - Add a **Manual Trigger** node named *When clicking ‘Test workflow’*.
   - Connect to a **Google Drive** node named *Setup - Create Transcript-Evalu8r Folder* to create the root folder.
   - Chain sequential Google Drive nodes to create:
     - *SETUP - TE Summaries* folder (subfolder for summaries),
     - *SETUP - TE Completed Audio* (subfolder for finalized audio),
     - *SETUP - TE Transcript_json* (subfolder for transcript JSON files).

3. **Configure Input Reception Webhook:**
   - Add **Webhook** node *Webhook4* with a unique webhook URL for browser audio uploads.
   - Connect to **Google Drive - Upload** node *Google Drive uploadAudio*, set folder to upload audio files.
     - Pass binary data in file upload.
   - Connect *Google Drive uploadAudio* to **Respond to Webhook** *Respond to Webhook4* to send upload confirmation.
   - Also connect *Google Drive uploadAudio* to **Google Drive - Download** node *Google Drive downloadUploaded*, configured to download using the uploaded file ID.

4. **Set Google Drive Trigger:**
   - Add **Google Drive Trigger** node *Google Drive Trigger*, configured to watch the “Completed Audio” folder for new files.
   - Connect to **Google Drive - Download** node *Google Drive Download Audio* to fetch new audio files.

5. **Deepgram Transcription Setup:**
   - Add **HTTP Request** node *DeepGram*.
   - Configure to POST audio binary to Deepgram API endpoints with authorization header and proper parameters for speaker diarization, punctuation, etc.
   - Inputs come from both *Google Drive Download Audio* and *Google Drive downloadUploaded* nodes.
   - Output sends transcript JSON to next steps.

6. **Process Transcript JSON:**
   - Add **Code** node *ProcessJSON* to clean and structure Deepgram transcript JSON to expected format.
   - Connect *DeepGram* output to *ProcessJSON*.
   - Connect *ProcessJSON* output to **Set** node *SetJSONFields*, which sets fields such as prompts, metadata, or conversation context for AI processing.

7. **AI Analysis with LangChain:**
   - Add **LangChain LM Chat OpenAI** node *OpenAI Chat Model*, configured with GPT-4o.
   - Add **LangChain Output Parser Structured** node *Structured Output Parser*, defining schemas for summary, sentiment, key points etc.
   - Add **LangChain Agent** node *AI Agent*, connect *SetJSONFields* output and internal connections:
     - AI Agent uses *OpenAI Chat Model* as language model node.
     - AI Agent uses *Structured Output Parser* as output parser.
   - AI Agent outputs to *Merge* node and triggers subsequent audio renaming.

8. **File Management and Google Docs Creation:**
   - Connect AI Agent output to **Google Drive** node *RenameAudio* to update audio filename.
   - Connect *RenameAudio* to **Google Drive** node *MoveAudio* moving file to completed audio folder.
   - Connect *MoveAudio* to **Google Docs** node *Create Google Docs* to generate document with transcript and AI-generated content.
   - Connect *Create Google Docs* to **Google Docs** node *Google Docs1* for any additional formatting or permissions.
   - Connect *Google Docs1* to **Google Drive** node *MoveTranscriptAnalysis* to place documents into analysis folder.
   - Connect *MoveTranscriptAnalysis* and *Upload Json* (uploads AI JSON file) to a **Merge** node *Merge1*.
   - Connect *Merge1* to **Gmail** node for email notification with document and transcript links.

9. **JSON Transcript Serving Webhooks:**
   - Add **Webhook** nodes *Webhook1*, *Webhook2*, and *Webhook3* for:
     - Serving HTML UI,
     - Listing JSON files through Google Drive *GetJSONs*,
     - Downloading specific JSON files via *GetJSON* and *ExtractJSON*.
   - Connect appropriate **Respond to Webhook** nodes to respond with HTML or JSON data.

10. **Additional Functional Nodes:**
    - Add **Code** node *GetSpeakers* to extract speaker information from transcript JSON.
    - Connect speaker data to Convert to File node *Convert to File1* and then to *Upload Json* Google Drive upload node.

11. **Final Testing:**
    - Use *When clicking ‘Test workflow’* manual trigger to create folder structure.
    - Test audio recording upload via *Webhook4* endpoint.
    - Verify transcription, AI analysis, Google Docs output, and notification flow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                            |
|----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| Transcript Evalu8r V2 introduces in-browser audio recording and device selection capabilities for immediate workflow input.     | Overview section                                                          |
| Use Deepgram API for highly accurate speech-to-text with speaker diarization and punctuation.                                    | https://deepgram.com                                                      |
| GPT-4o model via LangChain agent provides advanced AI summarization, sentiment analysis, and insight extraction.                 | https://openai.com/                                                        |
| Google Drive integration manages file storage including audio, JSON transcripts, and AI summaries in Docs format.                | n8n native Google Drive nodes                                              |
| Webhooks expose REST endpoints for browser communication, supporting seamless upload, playback, and transcript retrieval.       | Nodes: Webhook1-4; architecture enables UI interactions                    |
| Folder setup nodes allow easy initialization of Drive folder hierarchy, essential for organized file management.                  | Setup block: triggered manually at workflow start                         |
| Workflow error handling is managed through a configured error workflow (referenced in settings), recommended for production use.| Workflow settings                                                         |
| Transcript Evalu8r V2 supports export to multiple formats and integration with downstream tools like Slack, Notion, or Todoist.  | Expandable workflow - can be customized after existing nodes              |
| UI and player controls are handled outside of this workflow but interact via webhooks and REST API calls to n8n endpoints.       | Front-end application complementary to workflow                          |

---

This document encompasses all essential details for understanding, modifying, and redeploying the Transcript Evalu8r V2 workflow in n8n, including insight into integration points, AI processing, error scenarios, and file management strategies.