Extract Meeting To-Do Lists from Audio with Google Gemini and Send to Slack

https://n8nworkflows.xyz/workflows/extract-meeting-to-do-lists-from-audio-with-google-gemini-and-send-to-slack-9808


# Extract Meeting To-Do Lists from Audio with Google Gemini and Send to Slack

### 1. Workflow Overview

This workflow automates the extraction of actionable to-do items from meeting audio files stored in Google Drive and then sends the structured task list to a Slack channel. It is tailored for teams and project managers who want to streamline post-meeting follow-ups by automatically transcribing audio recordings, analyzing the transcript for tasks, and notifying team members via Slack.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Handling**: Watches a specific Google Drive folder for new or updated audio files and downloads them.
- **1.2 AI Transcription and Analysis**: Uses Google Gemini (PaLM) AI models to transcribe audio recordings into text and then extract actionable to-do items from the transcription.
- **1.3 Formatting and Notification**: Adds a timestamp to the extracted tasks and sends the formatted to-do list message to a designated Slack channel.
- **1.4 Supporting Documentation**: Sticky Notes provide meta-information about the workflow purpose and node groups for clarity.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Handling

- **Overview:**  
  This block listens for new or updated audio files in a specified Google Drive folder and downloads the detected audio file for further processing.

- **Nodes Involved:**  
  - Looking for uploading file (Google Drive Trigger)  
  - Download file (Google Drive)

- **Node Details:**

  - **Looking for uploading file**  
    - *Type:* Google Drive Trigger  
    - *Role:* Watches a specific Google Drive folder for file updates (uploads or modifications).  
    - *Configuration:*  
      - Event triggered on file update of any file type within the folder ID `1LfNfyCnJ-XVCevq32rSULZfH0Zi6KgH8`.  
      - Polling interval: every minute.  
    - *Input:* None (trigger node).  
    - *Output:* Emits metadata of updated file including its Google Drive file ID.  
    - *Edge cases:* Permissions errors on Google Drive OAuth, folder ID changes or deletions, network timeouts.  
    - *Credentials:* Google Drive OAuth2 (named "Google Drive account 3").  

  - **Download file**  
    - *Type:* Google Drive (File download)  
    - *Role:* Downloads the file identified by the trigger node for subsequent transcription.  
    - *Configuration:*  
      - Downloads the file using the dynamic file ID from the trigger node (`{{$json.id}}`).  
    - *Input:* File metadata from trigger node.  
    - *Output:* Binary data of the audio file.  
    - *Edge cases:* File access denied, download failures, unsupported file formats.  
    - *Credentials:* Same Google Drive OAuth2 as trigger node.

---

#### 2.2 AI Transcription and Analysis

- **Overview:**  
  This block converts the downloaded audio into text using Google Gemini’s audio transcription model, then analyzes the transcript to extract actionable to-do items in JSON format.

- **Nodes Involved:**  
  - Transcribe a recording1 (Google Gemini - audio transcription)  
  - Analyze document (Google Gemini - text analysis)

- **Node Details:**

  - **Transcribe a recording1**  
    - *Type:* Google Gemini AI (Langchain integration)  
    - *Role:* Transcribes the audio file binary into a text transcript.  
    - *Configuration:*  
      - Uses model `models/gemini-2.5-pro`.  
      - Input resource is audio, taking the binary data from the previous node.  
    - *Input:* Audio binary from "Download file" node.  
    - *Output:* Text transcript of the meeting audio.  
    - *Credentials:* Google Gemini API (named "Google Gemini(PaLM) Api account 4").  
    - *Edge cases:* Transcription errors, audio format incompatibility, API rate limits or authentication errors.

  - **Analyze document**  
    - *Type:* Google Gemini AI (Langchain integration)  
    - *Role:* Processes the transcript text to extract structured action items as JSON.  
    - *Configuration:*  
      - Uses model `models/gemini-2.5-flash`.  
      - The prompt instructs the AI to extract a JSON array of action items including `task_description`, `assigned_to`, `deadline`, and `priority`.  
      - Enforces strict output rules: JSON array only, empty array if no tasks, no summaries or extraneous text.  
      - Input text is dynamically set to the transcription result from "Transcribe a recording1".  
    - *Input:* Transcript text from transcription node.  
    - *Output:* JSON array of extracted action items.  
    - *Credentials:* Same Google Gemini API as above.  
    - *Edge cases:* AI output format errors, prompt misinterpretation, API errors, empty transcripts.

---

#### 2.3 Formatting and Notification

- **Overview:**  
  This block timestamps the extracted to-do list and sends it as a message to a specific Slack channel for team visibility.

- **Nodes Involved:**  
  - Get date (Date/Time)  
  - Format date (Date/Time)  
  - Send a message (Slack)

- **Node Details:**

  - **Get date**  
    - *Type:* Date/Time node  
    - *Role:* Captures the current date and time when the to-do list is generated.  
    - *Configuration:* Uses default settings, output field named `Date`.  
    - *Input:* JSON content from "Analyze document".  
    - *Output:* JSON including current datetime.  
    - *Edge cases:* System clock issues or timezone misconfigurations.

  - **Format date**  
    - *Type:* Date/Time node  
    - *Role:* Formats the captured date/time into a readable string for inclusion in messages.  
    - *Configuration:* Formats the date from the `Date` field in input JSON.  
    - *Input:* Output from "Get date".  
    - *Output:* Formatted date string.  
    - *Edge cases:* Date format errors if input missing or malformed.

  - **Send a message**  
    - *Type:* Slack node  
    - *Role:* Sends the extracted to-do list text as a message to a designated Slack channel.  
    - *Configuration:*  
      - Sends message text from the first part of the AI's response (`$('Analyze document').item.json.content.parts[0].text`).  
      - Targets Slack channel ID `C09LK8LDW79` (named "議事録ーtodoリスト").  
      - Uses OAuth2 authentication.  
    - *Input:* Formatted to-do list text, linked from "Format date".  
    - *Output:* None (end node).  
    - *Credentials:* Slack OAuth2 (named "Slack account 7").  
    - *Edge cases:* Slack API rate limits, wrong channel ID, OAuth token expiration or permission issues.

---

#### 2.4 Supporting Documentation (Sticky Notes)

- **Overview:**  
  These nodes provide contextual documentation and guidance within the workflow editor for easier understanding and maintenance.

- **Nodes Involved:**  
  - Sticky Note (near file download nodes)  
  - Sticky Note2 (near transcription nodes)  
  - Sticky Note1 (near date/Slack nodes)  
  - Sticky Note4 (overview and instructions)

- **Content Summary:**  
  - Explains node groups and their responsibilities (e.g., downloading files, generating summaries, formatting, and sending messages).  
  - Provides detailed textual overview of the workflow purpose, use cases, setup instructions, and customization tips.  
  - Includes workflow branding and instructions for credential setup.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                      | Input Node(s)               | Output Node(s)             | Sticky Note                                                   |
|-------------------------|----------------------------------|------------------------------------|----------------------------|----------------------------|---------------------------------------------------------------|
| Looking for uploading file | Google Drive Trigger             | Watches Google Drive folder for new/updated files | None                       | Download file              | ## Download the file These two nodes are responsible for looking and downloading the uploaded file |
| Download file            | Google Drive                     | Downloads updated audio file        | Looking for uploading file  | Transcribe a recording1    | ## Download the file These two nodes are responsible for looking and downloading the uploaded file |
| Transcribe a recording1  | Google Gemini AI (audio)         | Transcribes audio to text           | Download file              | Analyze document           | ## Generate Summary These two nodes are responsible for looking and downloading the uploaded file |
| Analyze document         | Google Gemini AI (text)          | Extracts action items from transcript | Transcribe a recording1    | Get date                   | ## Generate Summary These two nodes are responsible for looking and downloading the uploaded file |
| Get date                 | Date/Time                       | Captures current datetime           | Analyze document           | Format date                | ## Format and Send Message These three nodes are responsible for timestamping the result and sending it to your Slack channel. - **Get & Format Date:** Gets the current date and time to record when the to-do list was created. - **Send a message:** Sends the final to-do list extracted by the AI to your designated Slack channel. |
| Format date              | Date/Time                       | Formats captured date/time          | Get date                   | Send a message             | ## Format and Send Message These three nodes are responsible for timestamping the result and sending it to your Slack channel. - **Get & Format Date:** Gets the current date and time to record when the to-do list was created. - **Send a message:** Sends the final to-do list extracted by the AI to your designated Slack channel. |
| Send a message           | Slack                          | Sends to-do list message to Slack  | Format date                | None                      | ## Format and Send Message These three nodes are responsible for timestamping the result and sending it to your Slack channel. - **Get & Format Date:** Gets the current date and time to record when the to-do list was created. - **Send a message:** Sends the final to-do list extracted by the AI to your designated Slack channel. |
| Sticky Note              | Sticky Note                    | Documentation                      | None                       | None                      | ## Download the file These two nodes are responsible for looking and downloading the uploaded file |
| Sticky Note2             | Sticky Note                    | Documentation                      | None                       | None                      | ## Generate Summary These two nodes are responsible for looking and downloading the uploaded file |
| Sticky Note1             | Sticky Note                    | Documentation                      | None                       | None                      | ## Format and Send Message These three nodes are responsible for timestamping the result and sending it to your Slack channel. - **Get & Format Date:** Gets the current date and time to record when the to-do list was created. - **Send a message:** Sends the final to-do list extracted by the AI to your designated Slack channel. |
| Sticky Note4             | Sticky Note                    | Documentation                      | None                       | None                      | Generate meeting to-do lists from audio files in Google Drive and send to Slack This workflow automates the process of converting audio meeting recordings into a structured to-do list. It listens for new audio files in a Google Drive folder, transcribes them, extracts action items using AI, and sends a formatted list to a designated Slack channel. Who’s it for This template is perfect for project managers, teams, and anyone who wants to save time on post-meeting administrative tasks. If you record your meetings and use Google Drive for storage and Slack for team communication, this workflow will streamline your follow-up process and ensure no action item is missed. What it does This workflow automates the entire process of turning spoken words from a meeting into actionable tasks for your team. Trigger on New Audio: The workflow starts automatically when you upload a new audio file (e.g., MP3, M4A, WAV) to a specific folder in your Google Drive. Transcribe Audio: It takes the audio file and uses Google Gemini to generate a full text transcript of the recording. Extract To-Do Items: The transcript is then passed to another Google Gemini node with a specialized prompt. This prompt instructs the AI to carefully analyze the text and extract all action items. Format Output: The AI formats the extracted tasks into a clean JSON array. Each task includes a description, the assigned person, a deadline, and its priority. Send to Slack: Finally, the workflow sends the structured to-do list as a message to your specified Slack channel, making it easy for the whole team to see and act upon. How to set up Configure Credentials: Ensure you have configured your credentials for Google Drive, Google Gemini, and Slack in n8n. Set Google Drive Folder: In the "Looking for uploading file" node, select the Google Drive folder you want the workflow to monitor. Set Slack Channel: In the "Send a message" node, choose the correct Slack account and select the channel where you want the to-do list to be posted. Activate Workflow: Save your changes and activate the workflow using the toggle at the top right. Test It: Upload a meeting recording to the designated Google Drive folder to see the magic happen! How to customize the workflow Change AI Model: You can easily swap the Google Gemini nodes for other AI models like OpenAI or Anthropic to handle transcription and analysis based on your preference. Modify the AI Prompt: Adjust the prompt in the "Analyze document" node to change the output format. For example, you could ask for a meeting summary in addition to the to-do list. Change Notification Service: Replace the Slack node with another notification service like Discord, Microsoft Teams, or an email node. Archive Results: Add a node (e.g., Google Sheets, Notion, Airtable) after the "Analyze document" node to save a history of all meeting transcripts and their corresponding action items. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node ("Looking for uploading file")**  
   - Type: Google Drive Trigger  
   - Set event to `fileUpdated` for all file types.  
   - Configure to watch a specific folder by its ID (e.g., `1LfNfyCnJ-XVCevq32rSULZfH0Zi6KgH8`).  
   - Poll every minute.  
   - Assign Google Drive OAuth2 credentials.

2. **Create Google Drive node ("Download file")**  
   - Type: Google Drive (File download)  
   - Operation: Download  
   - File ID: Use expression `{{$json["id"]}}` to dynamically reference the file triggered.  
   - Assign same Google Drive OAuth2 credentials as trigger.  
   - Connect the output of "Looking for uploading file" to this node.

3. **Create Google Gemini node for transcription ("Transcribe a recording1")**  
   - Type: Google Gemini (Langchain integration)  
   - Model ID: Set to `models/gemini-2.5-pro` (audio transcription model).  
   - Resource: Audio  
   - Input Type: Binary (connects to downloaded audio binary)  
   - Assign Google Gemini API credentials.  
   - Connect "Download file" output to this node.

4. **Create Google Gemini node for task extraction ("Analyze document")**  
   - Type: Google Gemini (Langchain integration)  
   - Model ID: `models/gemini-2.5-flash` (text analysis model).  
   - Resource: Document (text input)  
   - Input Text: Use expression to insert transcript text from previous node:  
     ```
     =What's in this document🧠 System Prompt: Action Item Extractor (JSON Output)

     You are a highly specialized AI assistant focused on task extraction. Your sole responsibility is to analyze the provided meeting transcript and extract all actionable tasks (To-Do items).

     Your output MUST be a valid JSON array of objects. Each object in the array represents a single action item and must contain the following keys:
     - "task_description": A clear and concise description of the task.
     - "assigned_to": The name of the person responsible. If not mentioned, use null.
     - "deadline": The due date for the task. If not mentioned, use null. Try to format it as YYYY-MM-DD.
     - "priority": The priority of the task ("High", "Medium", "Low"). Infer this from the context. If it's unclear, default to "Medium".

     CRITICAL RULES:
     - Only output the JSON array. Do not include any explanatory text, introductory sentences, or markdown formatting like ```json.
     - If no action items are found in the transcript, output an empty array: 対象議事録なし.
     - Do not include summaries, discussion points, or any information that is not a specific, actionable task.?

     {{ $('Transcribe a recording1').item.json.text }}
     ```
   - Assign Google Gemini API credentials.  
   - Connect "Transcribe a recording1" output to this node.

5. **Create Date/Time node ("Get date")**  
   - Type: Date/Time  
   - Operation: None (default, just outputs current datetime)  
   - Output field name: `Date`  
   - Connect "Analyze document" output to this node.

6. **Create Date/Time node ("Format date")**  
   - Type: Date/Time  
   - Operation: Format date  
   - Date input: Use expression `{{$json["Date"]}}` from previous node output.  
   - Connect "Get date" output to this node.

7. **Create Slack node ("Send a message")**  
   - Type: Slack  
   - Authentication: OAuth2 (Slack OAuth2 credentials)  
   - Channel: Select or input the target Slack channel ID (e.g., `C09LK8LDW79`).  
   - Message text: Use expression to extract AI output text:  
     ```
     ={{ $('Analyze document').item.json.content.parts[0].text }}
     ```  
   - Connect "Format date" output to this node.

8. **Add Sticky Notes (Optional but recommended for clarity)**  
   - Add notes near logical node groups describing their purpose: file download, AI processing, formatting and sending.

9. **Credential Setup**  
   - Google Drive OAuth2 credentials with access to the watched folder.  
   - Google Gemini API credentials with permission to use selected models.  
   - Slack OAuth2 credentials with permissions to post messages to the target channel.

10. **Activate the workflow and test**  
    - Save and activate the workflow.  
    - Upload an audio meeting recording (MP3, M4A, WAV) to the specified Google Drive folder.  
    - Monitor Slack for the generated to-do list message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow automates converting meeting audio files into actionable to-do lists and sending them to Slack, saving time on post-meeting tasks. It is ideal for project managers and teams using Google Drive and Slack.                                                                                                                                                                                     | Workflow purpose and target audience                          |
| To customize, you can swap Google Gemini nodes with other AI providers like OpenAI or Anthropic, modify the AI prompt for different outputs (e.g., summaries), or replace Slack with other notification services (Discord, Teams, email). You can also add nodes to archive results in Google Sheets, Notion, or Airtable.                                                                                     | Customization guidance                                        |
| Ensure you configure all required credentials (Google Drive, Google Gemini, Slack) in n8n before running the workflow.                                                                                                                                                                                                                                                                                     | Credential requirements                                       |
| Official Google Drive folder URL for the watched folder: https://drive.google.com/drive/folders/1LfNfyCnJ-XVCevq32rSULZfH0Zi6KgH8                                                                                                                                                                                                                                                                              | Google Drive folder reference                                 |
| Slack channel name referenced: 議事録ーtodoリスト (means “Meeting minutes - to-do list” in Japanese)                                                                                                                                                                                                                                                                                                          | Slack channel naming                                          |
| The prompt used in the "Analyze document" node enforces strict JSON output for task extraction, preventing extraneous text and ensuring machine-readable structured results.                                                                                                                                                                                                                                  | AI prompt design                                              |

---

**Disclaimer:**  
The provided text and workflow are generated from an automated n8n workflow. All data handled is legal and public, compliant with content policies. No raw JSON is included here except references for clarity.