Automated Meeting Minutes from Audio using OpenAI Transcription and Google Docs

https://n8nworkflows.xyz/workflows/automated-meeting-minutes-from-audio-using-openai-transcription-and-google-docs-9644


# Automated Meeting Minutes from Audio using OpenAI Transcription and Google Docs

### 1. Workflow Overview

This workflow automates the generation of meeting minutes from audio recordings using OpenAI transcription and summarization, then documents the results in Google Docs. It targets use cases where users record meetings and want concise, structured summaries including key points, next actions, and concerns, without manual note-taking.

The logic is grouped into the following blocks:

- **1.1 Input Reception:** Collect meeting audio and metadata via a form trigger.
- **1.2 Audio Transcription:** Use OpenAI Whisper model to transcribe the audio recording into text.
- **1.3 AI Meeting Minutes Generation:** Summarize the transcript into structured meeting minutes with action points via OpenAI GPT.
- **1.4 Google Docs Management:** Create a new Google Doc and insert the generated meeting minutes content.
- **1.5 Documentation & Guidance:** Sticky notes providing explanations, setup tips, and usage notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block collects the raw input data for the workflow: the meeting audio file and metadata such as Manager, Partner, and meeting Situation. It uses a form trigger node for user-friendly input.

**Nodes Involved:**  
- Meeting Intake

**Node Details:**

- **Meeting Intake**  
  - Type: Form Trigger (n8n-nodes-base.formTrigger)  
  - Role: Entry point; collects audio file and metadata via a form.  
  - Configuration:  
    - Form titled "Meeting-Minutes Assistant"  
    - Fields:  
      - Audio File (required, accepts m4a, mp3, wav, webm, m4b, mpeg)  
      - Manager (required text)  
      - Partner (required text)  
      - Situation (dropdown with options: First meeting, Estimate/Proposal, Support)  
  - Inputs: Webhook trigger from form submission  
  - Outputs: Passes binary audio and JSON metadata downstream  
  - Version: 2.3  
  - Edge Cases:  
    - Large audio files (>50MB) may cause slow processing or failures  
    - Unsupported file formats rejected by the form  
    - Missing required fields prevent submission  
  - Notes: Binary data is normalized and passed to transcription node  
  - Sticky Note: Explains purpose and usage, suggests keeping audio < 50MB  

#### 1.2 Audio Transcription

**Overview:**  
Converts the uploaded meeting audio file into a text transcript using OpenAI's Whisper transcription model.

**Nodes Involved:**  
- Transcribe recording

**Node Details:**

- **Transcribe recording**  
  - Type: OpenAI Langchain node (audio transcription)  
  - Role: Transcribes audio binary input to text  
  - Configuration:  
    - Resource: audio  
    - Operation: transcribe  
    - Binary property name dynamically set to the uploaded audio file key  
  - Inputs: Binary audio file from "Meeting Intake"  
  - Outputs: JSON including transcript text under property `text`  
  - Credentials: OpenAI API (using free API credits)  
  - Version: 1.8  
  - Edge Cases:  
    - Audio quality issues or unsupported codec may reduce transcription accuracy  
    - API rate limits or network errors  
    - Missing binary data leads to failure  
  - Sticky Note: None specific, but general setup notes apply  

#### 1.3 AI Meeting Minutes Generation

**Overview:**  
Summarizes the transcribed text into concise, structured meeting minutes using a GPT-4 variant. The output includes key points, next actions with owners and deadlines, and concerns from the other party.

**Nodes Involved:**  
- Generate Meeting Minutes

**Node Details:**

- **Generate Meeting Minutes**  
  - Type: OpenAI Langchain node (chat completion)  
  - Role: Generate structured meeting minutes from transcript and metadata  
  - Configuration:  
    - Model: GPT-4O-MINI (a GPT-4 variant optimized for cost/performance)  
    - Prompt: System message instructs to produce:  
      1. Key points (3–6 lines)  
      2. Next actions (owner, deadline)  
      3. Other party’s concerns/requests  
    - Input variables embedded in prompt:  
      - Transcript text (`{{ $json.text }}` from transcription node)  
      - Manager, Partner, and Situation values from form trigger  
    - Output length targeted: 300–600 characters, concise bullet points in English  
  - Inputs: JSON transcript plus form metadata  
  - Outputs: JSON with a `message.content` string containing the meeting minutes  
  - Credentials: OpenAI API  
  - Version: 1.8  
  - Edge Cases:  
    - Prompt failures due to malformed expressions  
    - API quota exceeded or network errors  
    - Very short or poor transcript quality leading to incomplete summaries  
  - Sticky Note: Highlights purpose, structure and input; suggests prompt tuning  

#### 1.4 Google Docs Management

**Overview:**  
Creates a new Google Doc named after the Partner and Situation, then inserts the AI-generated meeting minutes content into the document.

**Nodes Involved:**  
- Create Minutes Doc  
- Insert Minutes Content

**Node Details:**

- **Create Minutes Doc**  
  - Type: Google Docs node  
  - Role: Creates a new Google Document in a specified folder  
  - Configuration:  
    - Document title constructed as `{Partner}_{Situation}` from form input  
    - Destination folder ID statically set to a Google Drive folder  
  - Inputs: From Generate Meeting Minutes  
  - Outputs: JSON with new document ID and URL  
  - Version: 2  
  - Credentials: Google OAuth2 (configured externally)  
  - Edge Cases:  
    - Permission issues for Google Drive folder  
    - Rate limits or API errors from Google Docs API  

- **Insert Minutes Content**  
  - Type: Google Docs node  
  - Role: Updates the created document by inserting meeting minutes text  
  - Configuration:  
    - Operation: update  
    - Action: insert text from `Generate Meeting Minutes` output at document start  
    - Document URL dynamically set from `Create Minutes Doc` output document ID  
  - Inputs: From Create Minutes Doc  
  - Outputs: Final updated document info  
  - Version: 2  
  - Credentials: Google OAuth2  
  - Edge Cases:  
    - Document ID mismatch or access revoked  
    - Insertion errors due to text formatting or length  
  - Sticky Note: Mentions template with timestamp and structured sections  

#### 1.5 Documentation & Guidance

**Overview:**  
Sticky notes provide users with workflow overview, setup tips, input requirements, and node descriptions.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note2  
- Sticky Note4  
- Sticky Note7

**Node Details:**

- **Sticky Note** (positioned top-left)  
  - Content: Overall workflow summary, setup instructions, tips about audio length, credentials setup, and troubleshooting references.  
- **Sticky Note2** (near Meeting Intake)  
  - Content: Details about the Meeting Intake form trigger fields, file size limits, and timezone normalization.  
- **Sticky Note4** (near Generate Meeting Minutes)  
  - Content: Purpose and structure of the AI meeting minutes generation step, input expectations, and length guidelines.  
- **Sticky Note7** (near Insert Minutes Content)  
  - Content: Describes the Google Docs insertion template, mentioning timestamp and sections included.  

---

### 3. Summary Table

| Node Name             | Node Type                     | Functional Role                          | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                  |
|-----------------------|-------------------------------|----------------------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Meeting Intake        | Form Trigger                  | Collect audio and metadata input       | (Trigger)             | Transcribe recording      | ## Meeting Intake (Trigger) Purpose: Collect audio + meta. Fields: Audio (m4a/mp3/wav), Manager, Partner, Situation. Notes: Keep file <50MB; pass binary to next node; normalize timezone. |
| Transcribe recording  | OpenAI Langchain (audio)      | Transcribe audio to text                | Meeting Intake        | Generate Meeting Minutes  |                                                                                              |
| Generate Meeting Minutes | OpenAI Langchain (chat)       | Summarize transcript into meeting minutes | Transcribe recording  | Create Minutes Doc        | ## Generate Meeting Minutes Purpose: Summarize transcript → action-oriented minutes. Structure: Key Points / Next Actions (OWNER, DUE) / Concerns. Input: {{ $json.text }} + form fields. Keep ~300–600 chars (edit as needed). |
| Create Minutes Doc    | Google Docs                   | Create Google Doc for meeting minutes  | Generate Meeting Minutes | Insert Minutes Content    |                                                                                              |
| Insert Minutes Content | Google Docs                   | Insert meeting minutes text into Doc   | Create Minutes Doc     | (End)                    | ## Insert Minutes Content Template: Timestamp + sections (Key Points / Next Actions / Concerns). |
| Sticky Note           | Sticky Note                   | Workflow overview and setup tips       | —                     | —                        | ## Meeting Minutes Assistant — Overview Form → Transcribe (OpenAI) → Summarize → Google Docs (Create/Append) Result: Clean minutes + Doc URL. Setup: Connect OpenAI & Google (OAuth2). No hardcoded keys. Tip: Test with <2 min audio; then tune the prompt. Next: See Description page for full setup & troubleshooting. |
| Sticky Note2          | Sticky Note                   | Explains Meeting Intake node            | —                     | —                        | ## Meeting Intake (Trigger) Purpose: Collect audio + meta. Fields: Audio (m4a/mp3/wav), Manager, Partner, Situation. Notes: Keep file <50MB; pass binary to next node; normalize timezone. |
| Sticky Note4          | Sticky Note                   | Explains Generate Meeting Minutes node  | —                     | —                        | ## Generate Meeting Minutes Purpose: Summarize transcript → action-oriented minutes. Structure: Key Points / Next Actions (OWNER, DUE) / Concerns. Input: {{ $json.text }} + form fields. Keep ~300–600 chars (edit as needed). |
| Sticky Note7          | Sticky Note                   | Explains Insert Minutes Content node    | —                     | —                        | ## Insert Minutes Content Template: Timestamp + sections (Key Points / Next Actions / Concerns). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named "Meeting Intake":**  
   - Set form title to "Meeting-Minutes Assistant".  
   - Add fields:  
     - File upload field named "Audio File", required, accept types: m4a, mp3, wav, webm, m4b, mpeg.  
     - Text fields "Manager" (required) and "Partner" (required).  
     - Dropdown "Situation" (required) with options: "First meeting", "Estimate/Proposal", "Support".  
   - Ensure webhook is active for receiving form submissions.

2. **Add an OpenAI Langchain node named "Transcribe recording":**  
   - Set resource to "audio" and operation to "transcribe".  
   - Set "binaryPropertyName" to dynamically reference the uploaded audio file key: `={{ Object.keys($binary)[0] }}`.  
   - Connect input from "Meeting Intake".  
   - Configure OpenAI API credentials with valid API key.

3. **Add an OpenAI Langchain Chat node named "Generate Meeting Minutes":**  
   - Choose GPT-4O-MINI model (or equivalent GPT-4 variant).  
   - Create a system message prompt:  
     ```
     You are a meeting-minutes assistant. From the following audio transcript, please produce:

     1. Key points (3–6 lines)
     2. Next actions (clearly specify owner and deadline)
     3. The other party’s concerns/requests

     Write concise bullet points in English, keeping the total length around 300–600 characters.

     # Input Data:
      - Transcript: {{ $json.text }}
      - Manager: {{ $('Meeting Intake').item.json.Manager }}
      - Partner: {{ $('Meeting Intake').item.json.Partner }}
      - Situation: {{ $('Meeting Intake').item.json.Situation }}
     ```  
   - Connect input from "Transcribe recording".  
   - Use same OpenAI API credential.

4. **Add a Google Docs node named "Create Minutes Doc":**  
   - Set operation to create a new document.  
   - Set title to `={{ $('Meeting Intake').item.json.Partner }}_{{ $('Meeting Intake').item.json.Situation }}`.  
   - Specify a Google Drive folder ID (create a dedicated folder beforehand).  
   - Connect input from "Generate Meeting Minutes".  
   - Configure Google OAuth2 credentials with write access to Google Docs and Drive.

5. **Add a Google Docs node named "Insert Minutes Content":**  
   - Set operation to "update".  
   - Under actions, add an insert text action with text: `={{ $('Generate Meeting Minutes').item.json.message.content }}`.  
   - Set document URL dynamically to `={{ $json.id }}` from the "Create Minutes Doc" output.  
   - Connect input from "Create Minutes Doc".  
   - Use same Google OAuth2 credentials.

6. **Add Sticky Notes for documentation:**  
   - Add a sticky note summarizing overall workflow, setup tips, and usage notes near the start.  
   - Add sticky notes near each major node to explain purpose, inputs, and constraints, especially for "Meeting Intake", "Generate Meeting Minutes", and "Insert Minutes Content".

7. **Activate the workflow and test:**  
   - Upload short (<2 minutes) audio files to test transcription quality and summarization.  
   - Adjust prompt if summaries are too long or incomplete.  
   - Monitor for errors like API quota exhaustion or file size issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup requires configuring OpenAI API credentials and Google OAuth2 credentials with Drive and Docs scopes.            | Workflow credential configuration                                                                   |
| Keep audio files under 50MB for optimal processing speed and reliability.                                              | Sticky Note2                                                                                        |
| Test with short audio inputs initially (<2 minutes) to verify transcription and summarization quality.                 | Sticky Note                                                                                         |
| The workflow uses GPT-4O-MINI model to balance cost and performance. Adjust model choice as needed.                     | Generate Meeting Minutes node description                                                           |
| Google Drive folder ID must be pre-created and accessible by the Google OAuth2 credentials used in the workflow.       | Create Minutes Doc node setup                                                                        |
| Prompt is designed to produce concise bullet points; modify prompt text to customize output format or language if needed. | Generate Meeting Minutes prompt                                                                      |
| For troubleshooting and detailed setup instructions, refer to the original workflow description page (not included).  | Sticky Note                                                                                         |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow built with n8n, an integration and automation tool. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.