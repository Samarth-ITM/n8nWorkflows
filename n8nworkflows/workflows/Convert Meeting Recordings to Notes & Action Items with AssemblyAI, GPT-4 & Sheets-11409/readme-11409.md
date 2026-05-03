Convert Meeting Recordings to Notes & Action Items with AssemblyAI, GPT-4 & Sheets

https://n8nworkflows.xyz/workflows/convert-meeting-recordings-to-notes---action-items-with-assemblyai--gpt-4---sheets-11409


# Convert Meeting Recordings to Notes & Action Items with AssemblyAI, GPT-4 & Sheets

### 1. Workflow Overview

This workflow automates the conversion of meeting audio recordings into structured meeting notes and actionable task items, leveraging AssemblyAI for transcription and GPT-4 powered AI for note-taking and task extraction. The results are validated and then logged into Google Sheets for easy tracking and future reference.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception & Configuration**  
  Receives meeting recording events via webhook and loads essential configuration such as API keys and default parameters.

- **1.2 Audio Download & Transcription**  
  Downloads the meeting audio file and sends it to AssemblyAI to obtain a detailed transcript with speaker labels.

- **1.3 AI-Powered Notes & Action Items Extraction**  
  Prepares input data and uses GPT-4-based AI nodes to generate structured meeting notes and extract detailed action items, validated against strict JSON schemas.

- **1.4 Validation, Processing & Logging**  
  Validates and enriches action items with defaults, conditionally processes them (e.g., creating tasks, sending notifications), and logs meeting metadata and summaries into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration

**Overview:**  
This initial block receives a webhook call signaling that a meeting recording is ready and loads workflow-wide configuration parameters including API keys, default due dates, and admin email for notifications.

**Nodes Involved:**  
- Recording Ready Webhook  
- Workflow Configuration  
- Sticky Note1 (documentation)

**Node Details:**  

- **Recording Ready Webhook**  
  - Type: Webhook  
  - Role: Entry point receiving HTTP POST requests with meeting metadata and recording URL.  
  - Configuration: Path set to `meeting-recording-ready`, responds with output of last node.  
  - Inputs: External webhook HTTP POST call.  
  - Outputs: JSON payload including recording URL, meeting title, participants, start time, meeting ID.  
  - Edge cases: Missing or malformed webhook payload, HTTP errors, unauthorized calls if webhook exposed publicly.  
  - Sticky Note1 explains this block’s purpose.

- **Workflow Configuration**  
  - Type: Set  
  - Role: Defines and injects environment variables and constants into workflow context.  
  - Configuration: Sets AssemblyAI API key from environment variable, default due date (7 days), admin email placeholder.  
  - Inputs: From webhook node.  
  - Outputs: JSON with API key and settings for downstream nodes.  
  - Edge cases: Missing environment variable leading to API auth failure downstream.

---

#### 1.2 Audio Download & Transcription

**Overview:**  
Downloads the meeting audio file from the provided URL and sends it to AssemblyAI API to obtain a detailed transcript including speaker labels.

**Nodes Involved:**  
- Download Recording  
- AssemblyAI Transcription  
- Sticky Note2 (documentation)

**Node Details:**  

- **Download Recording**  
  - Type: HTTP Request  
  - Role: Downloads audio recording file from URL provided in webhook payload.  
  - Configuration: GET request to URL in `$json.recording_url`, response expected as file, stored in `recording` property.  
  - Inputs: From Workflow Configuration.  
  - Outputs: Binary file data of audio recording.  
  - Edge cases: Invalid or inaccessible URL, network timeouts, large file size issues, unsupported audio format.

- **AssemblyAI Transcription**  
  - Type: HTTP Request  
  - Role: Sends audio URL to AssemblyAI transcription API to start transcription job with speaker labels enabled.  
  - Configuration: POST to `https://api.assemblyai.com/v2/transcript` with JSON body containing `audio_url` and `speaker_labels: true`. Authorization header uses API key from Workflow Configuration node.  
  - Inputs: From Download Recording node (though audio file is not sent directly; URL is used).  
  - Outputs: JSON with transcription job details and eventually the transcript text.  
  - Edge cases: API auth failure, rate limits, transcription errors, long processing times.

---

#### 1.3 AI-Powered Notes & Action Items Extraction

**Overview:**  
Constructs a payload combining meeting metadata and transcript, then uses two sequential AI steps to generate structured meeting notes and extract detailed action items. Both outputs are validated against strict JSON schemas ensuring data quality.

**Nodes Involved:**  
- Prepare Meeting Notes Input  
- Generate Meeting Notes (LangChain Agent)  
- OpenAI Model - Meeting Notes (GPT-4)  
- Meeting Notes Schema (Output Parser)  
- Extract Action Items (LangChain Agent)  
- OpenAI Model - Action Items (GPT-4)  
- Action Items Schema (Output Parser)  
- Sticky Note3 (documentation)

**Node Details:**  

- **Prepare Meeting Notes Input**  
  - Type: Set  
  - Role: Builds a structured input JSON with meeting title, date, participants, and transcript text for AI processing.  
  - Configuration: Extracts fields from webhook and AssemblyAI nodes; fallback defaults for missing participants or title.  
  - Inputs: AssemblyAI Transcription and Recording Ready Webhook.  
  - Outputs: JSON payload with metadata and transcript for AI nodes.  
  - Edge cases: Missing transcript text, missing participants.

- **Generate Meeting Notes**  
  - Type: LangChain Agent  
  - Role: Defines the prompt and system message for generating comprehensive meeting notes structured in a specific format.  
  - Configuration: Prompt includes meeting metadata and transcript; system message instructs detailed note generation including agenda, decisions, next steps, and summary.  
  - Inputs: Prepare Meeting Notes Input.  
  - Outputs: AI-generated notes in raw form.  
  - Edge cases: AI output inconsistencies, parsing failures.

- **OpenAI Model - Meeting Notes**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Executes GPT-4 (gpt-4.1-mini) chat completion with the defined prompt to generate notes.  
  - Inputs: Generate Meeting Notes.  
  - Outputs: Raw AI text response.  
  - Edge cases: API limits, timeouts.

- **Meeting Notes Schema**  
  - Type: Output Parser Structured  
  - Role: Parses and validates AI output against a JSON schema defining fields like title, date, participants (array), agenda, decisions, next steps, executive summary.  
  - Inputs: Generate Meeting Notes.  
  - Outputs: Parsed and validated structured notes.  
  - Edge cases: Schema validation failures, malformed AI output.

- **Extract Action Items**  
  - Type: LangChain Agent  
  - Role: Defines prompt to extract actionable tasks from notes and transcript, specifying fields like description, owner, due date, priority, related topic.  
  - Inputs: Output from Generate Meeting Notes node and Prepare Meeting Notes Input.  
  - Outputs: Raw AI-generated action items list.  
  - Edge cases: Missing or ambiguous action items.

- **OpenAI Model - Action Items**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Runs GPT-4 model to extract action items according to prompt.  
  - Inputs: Extract Action Items.  
  - Outputs: Raw AI response with action items.  
  - Edge cases: API failures.

- **Action Items Schema**  
  - Type: Output Parser Structured  
  - Role: Validates action items output against a JSON schema requiring description, owner, priority, optional due date, and related topic.  
  - Inputs: Extract Action Items.  
  - Outputs: Validated structured action items array.  
  - Edge cases: Schema mismatch or missing required fields.

---

#### 1.4 Validation, Processing & Logging

**Overview:**  
This block normalizes and enriches action items with default values (e.g., due dates), checks if any action items exist, processes them (e.g., task creation and notification placeholders), and logs the meeting summary and counts into Google Sheets.

**Nodes Involved:**  
- Parse and Validate Action Items (Code)  
- Check If Action Items Exist (If node)  
- Process All Action Items (Code)  
- Log Meeting to Sheets (Google Sheets)  
- Sticky Note4 (documentation)

**Node Details:**  

- **Parse and Validate Action Items**  
  - Type: Code (JavaScript)  
  - Role: Fills missing due dates with a default from configuration, assigns 'Unassigned' owner if missing, and adds meeting metadata fields.  
  - Inputs: Action Items Schema node output and Workflow Configuration.  
  - Outputs: Cleaned and enriched action items array.  
  - Edge cases: Empty action items list, invalid dates.

- **Check If Action Items Exist**  
  - Type: If  
  - Role: Checks if the array of action items is not empty to conditionally process them.  
  - Inputs: Parse and Validate Action Items output.  
  - Outputs: Routes workflow to either processing or straight to logging.  
  - Edge cases: Empty action item arrays.

- **Process All Action Items**  
  - Type: Code (JavaScript)  
  - Role: Placeholder logic to process each action item (e.g., create Notion tasks, send emails); here it simulates success or error per item.  
  - Inputs: Check If Action Items Exist (true branch).  
  - Outputs: Processing results array with status flags.  
  - Edge cases: API failures to external services (Notion, email), error handling.

- **Log Meeting to Sheets**  
  - Type: Google Sheets  
  - Role: Appends or updates a row with meeting metadata, notes summary, transcript URL, participant and action item counts.  
  - Configuration: Uses service account credentials, requires Google Sheet document ID and sheet name placeholders.  
  - Inputs: Either from Process All Action Items or directly from Check If Action Items Exist (false branch).  
  - Outputs: Confirmation of successful logging.  
  - Edge cases: Google API auth errors, sheet not found, permission issues.

---

### 3. Summary Table

| Node Name                    | Node Type                       | Functional Role                                  | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                         |
|------------------------------|--------------------------------|-------------------------------------------------|----------------------------------|-------------------------------|--------------------------------------------------------------------|
| Recording Ready Webhook       | Webhook                        | Receives recording ready event                   | External webhook call             | Workflow Configuration         | ## Webhook & Config<br>Receives “recording ready” events and loads core settings... |
| Workflow Configuration        | Set                            | Loads API keys, default due dates, admin email  | Recording Ready Webhook           | Download Recording             | ## Webhook & Config                                                  |
| Download Recording            | HTTP Request                   | Downloads audio recording file                    | Workflow Configuration           | AssemblyAI Transcription       | ## Transcription<br>Downloads meeting audio and sends to AssemblyAI... |
| AssemblyAI Transcription      | HTTP Request                   | Sends audio URL to AssemblyAI for transcription | Download Recording               | Prepare Meeting Notes Input    | ## Transcription                                                    |
| Prepare Meeting Notes Input   | Set                            | Builds structured input for AI                    | AssemblyAI Transcription          | Generate Meeting Notes         | ## AI Notes & Actions<br>Builds structured payload from transcript... |
| Generate Meeting Notes        | LangChain Agent                | AI prompt for structured meeting notes          | Prepare Meeting Notes Input       | Extract Action Items           | ## AI Notes & Actions                                               |
| OpenAI Model - Meeting Notes  | LangChain OpenAI Chat Model    | Runs GPT-4 for meeting notes                      | Generate Meeting Notes            | Meeting Notes Schema           | ## AI Notes & Actions                                               |
| Meeting Notes Schema          | Output Parser Structured       | Validates notes output                            | Generate Meeting Notes            | Extract Action Items           | ## AI Notes & Actions                                               |
| Extract Action Items          | LangChain Agent                | AI prompt for extracting action items            | Generate Meeting Notes, Prepare Meeting Notes Input | Parse and Validate Action Items | ## AI Notes & Actions                                               |
| OpenAI Model - Action Items   | LangChain OpenAI Chat Model    | Runs GPT-4 for action items extraction            | Extract Action Items              | Action Items Schema            | ## AI Notes & Actions                                               |
| Action Items Schema           | Output Parser Structured       | Validates action items output                      | Extract Action Items              | Parse and Validate Action Items | ## AI Notes & Actions                                               |
| Parse and Validate Action Items | Code (JavaScript)             | Enriches and normalizes action items              | Action Items Schema               | Check If Action Items Exist    | ## Validate & Log<br>Cleans and validates action items...          |
| Check If Action Items Exist   | If                             | Checks presence of action items                    | Parse and Validate Action Items   | Process All Action Items, Log Meeting to Sheets | ## Validate & Log                                               |
| Process All Action Items      | Code (JavaScript)              | Processes each action item (task creation, email) | Check If Action Items Exist (true branch) | Log Meeting to Sheets          | ## Validate & Log                                                  |
| Log Meeting to Sheets         | Google Sheets                  | Logs meeting metadata and summary to Google Sheets | Process All Action Items, Check If Action Items Exist (false branch) | -                             | ## Validate & Log                                                  |
| Sticky Note                  | Sticky Note                    | Documentation                                    | -                                | -                             | ## How it works<br>This workflow turns meeting recordings into structured notes... |
| Sticky Note1                 | Sticky Note                    | Documentation for webhook and config             | -                                | -                             | ## Webhook & Config                                                |
| Sticky Note2                 | Sticky Note                    | Documentation for transcription                   | -                                | -                             | ## Transcription                                                  |
| Sticky Note3                 | Sticky Note                    | Documentation for AI notes and actions extraction | -                              | -                             | ## AI Notes & Actions                                             |
| Sticky Note4                 | Sticky Note                    | Documentation for validation and logging          | -                                | -                             | ## Validate & Log                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Recording Ready Webhook`  
   - Path: `meeting-recording-ready`  
   - HTTP Method: POST  
   - Response Mode: Last Node  
   - No credentials required.

2. **Create Set Node for Configuration**  
   - Type: Set  
   - Name: `Workflow Configuration`  
   - Add fields:  
     - `assemblyAiApiKey` (string): Use environment variable `{{$env.ASSEMBLYAI_KEY}}`  
     - `defaultDueDateDays` (number): `7`  
     - `adminEmail` (string): placeholder e.g. `"admin@example.com"`  
   - Connect from `Recording Ready Webhook`.

3. **Create HTTP Request Node to Download Recording**  
   - Type: HTTP Request  
   - Name: `Download Recording`  
   - Method: GET  
   - URL: `{{$json.recording_url}}` (from webhook payload)  
   - Response Format: File  
   - Output Property Name: `recording`  
   - Connect from `Workflow Configuration`.

4. **Create HTTP Request Node to Call AssemblyAI Transcription API**  
   - Type: HTTP Request  
   - Name: `AssemblyAI Transcription`  
   - URL: `https://api.assemblyai.com/v2/transcript`  
   - Method: POST  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "audio_url": "{{$json.recording_url}}",
       "speaker_labels": true
     }
     ```  
   - Authentication: HTTP Header with `Authorization` set to `{{$('Workflow Configuration').first().json.assemblyAiApiKey}}`  
   - Connect from `Download Recording`.

5. **Create Set Node to Prepare AI Input**  
   - Type: Set  
   - Name: `Prepare Meeting Notes Input`  
   - Fields:  
     - `meetingTitle`: `{{$json.meeting_title || 'Meeting'}}` (from webhook)  
     - `meetingDate`: `{{$json.start_time}}`  
     - `participants`: `{{$json.participants || 'Not provided'}}`  
     - `transcript`: `{{$('AssemblyAI Transcription').first().json.text}}`  
   - Connect from `AssemblyAI Transcription`.

6. **Create LangChain Agent Node for Meeting Notes Generation**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `Generate Meeting Notes`  
   - Text input:  
     ```
     Meeting Title: {{ $json.meetingTitle }}
     Date: {{ $json.meetingDate }}
     Participants: {{ $json.participants }}

     Transcript:
     {{ $json.transcript }}
     ```  
   - System Message (prompt): instruct detailed structured notes with fields (title, date, participants, agenda, key decisions, next steps, executive summary)  
   - Prompt Type: Define  
   - Output Parser: Enabled  
   - Connect from `Prepare Meeting Notes Input`.

7. **Create LangChain OpenAI Chat Model Node for Meeting Notes**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: `OpenAI Model - Meeting Notes`  
   - Model: `gpt-4.1-mini`  
   - Connect from `Generate Meeting Notes` (AI model input).

8. **Create Output Parser Structured Node for Meeting Notes**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Name: `Meeting Notes Schema`  
   - Schema (JSON manual): define object with required properties for notes fields as per prompt.  
   - Connect from `Generate Meeting Notes` (output parser input).

9. **Create LangChain Agent Node for Action Items Extraction**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `Extract Action Items`  
   - Text input:  
     ```
     Meeting Notes:
     {{ JSON.stringify($json.output) }}

     Transcript:
     {{ $('Prepare Meeting Notes Input').first().json.transcript }}
     ```  
   - System Message: instruct extraction of action items with description, owner, due_date, priority, related_topic  
   - Prompt Type: Define  
   - Output Parser: Enabled  
   - Connect from `Generate Meeting Notes` (main output).

10. **Create LangChain OpenAI Chat Model Node for Action Items**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Name: `OpenAI Model - Action Items`  
    - Model: `gpt-4.1-mini`  
    - Connect from `Extract Action Items` (AI model input).

11. **Create Output Parser Structured Node for Action Items**  
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - Name: `Action Items Schema`  
    - Schema (JSON manual): define object with array of action items, each with description, owner, due_date, priority (enum), related_topic  
    - Connect from `Extract Action Items` (output parser input).

12. **Create Code Node to Parse and Validate Action Items**  
    - Type: Code (JavaScript)  
    - Name: `Parse and Validate Action Items`  
    - Code:  
      ```js
      const actionItems = $input.item.json.output.action_items || [];
      const defaultDays = $('Workflow Configuration').first().json.defaultDueDateDays;
      const meetingId = $('Recording Ready Webhook').first().json.meeting_id;
      const meetingTitle = $('Recording Ready Webhook').first().json.meeting_title;

      const validated = actionItems.map(item => {
        let dueDate = item.due_date;
        if (!dueDate) {
          const future = new Date();
          future.setDate(future.getDate() + defaultDays);
          dueDate = future.toISOString().split('T')[0];
        }
        return {
          ...item,
          owner: item.owner || 'Unassigned',
          due_date: dueDate,
          meeting_id: meetingId,
          meeting_title: meetingTitle
        };
      });

      return { action_items: validated };
      ```  
    - Connect from `Action Items Schema`.

13. **Create If Node to Check Existence of Action Items**  
    - Type: If  
    - Name: `Check If Action Items Exist`  
    - Condition: Check if `{{$json.action_items}}` array is not empty (operator: notEmpty)  
    - Connect from `Parse and Validate Action Items`.

14. **Create Code Node to Process All Action Items**  
    - Type: Code (JavaScript)  
    - Name: `Process All Action Items`  
    - Code:  
      ```js
      const actionItems = $input.item.json.action_items || [];
      const results = [];

      for (const item of actionItems) {
        try {
          // Placeholder: create Notion task and send email notifications here

          results.push({
            description: item.description,
            owner: item.owner,
            due_date: item.due_date,
            priority: item.priority,
            status: 'processed',
            notion_created: true,
            email_sent: true
          });
        } catch (error) {
          results.push({
            description: item.description,
            owner: item.owner,
            status: 'error',
            error: error.message
          });
        }
      }

      return {
        json: {
          processed_count: results.length,
          action_items: results,
          meeting_id: $('Recording Ready Webhook').first().json.meeting_id,
          meeting_title: $('Recording Ready Webhook').first().json.meeting_title
        }
      };
      ```  
    - Connect from `Check If Action Items Exist` (true branch).

15. **Create Google Sheets Node to Log Meeting**  
    - Type: Google Sheets  
    - Name: `Log Meeting to Sheets`  
    - Operation: Append or Update  
    - Sheet Name: Placeholder for your sheet name (e.g., "Meetings")  
    - Document ID: Placeholder for your Google Sheets document ID  
    - Authentication: Use service account credentials configured in n8n  
    - Columns to map:  
      - `date`: `{{$json.start_time}}` from webhook  
      - `title`: `{{$json.output.title}}` from Generate Meeting Notes  
      - `meeting_id`: `{{$json.meeting_id}}` from webhook  
      - `notes_summary`: first bullet from executive_summary array  
      - `transcript_url`: `{{$json.recording_url}}` from webhook  
      - `participant_count`: count from splitting participants string by comma  
      - `action_items_count`: length of action items array after validation  
    - Connect from both:  
      - `Process All Action Items` (true branch)  
      - `Check If Action Items Exist` (false branch)

16. **Add Sticky Notes**  
    - Add sticky notes near relevant nodes explaining the workflow overview, each block’s purpose, and setup instructions as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                      | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow turns meeting recordings into structured notes and action items, then logs everything to Google Sheets. It requires AssemblyAI and Google Sheets credentials, webhook setup, and configuration of Sheet ID and default due dates. AI prompts can be customized. | Sticky Note on main workflow explaining overall function and setup.                                         |
| Webhook receives "recording ready" events and loads core settings including API keys and default parameters.                                                                                                                                                      | Sticky Note1 near webhook and configuration nodes.                                                          |
| Downloads meeting audio from URL and sends it to AssemblyAI for transcription with speaker labels enabled.                                                                                                                                                        | Sticky Note2 near download and transcription nodes.                                                         |
| AI steps generate structured meeting notes and extract action items, validated against JSON schemas for reliability.                                                                                                                                              | Sticky Note3 near AI nodes.                                                                                  |
| Validates and cleans action items, optionally processes them (task creation, notifications), then logs meeting summary and counts into Google Sheets.                                                                                                           | Sticky Note4 near validation and logging nodes.                                                             |
| For AI-driven nodes using LangChain and OpenAI GPT-4, ensure you have appropriate API keys and usage quotas.                                                                                                                                                      | General caution for AI nodes.                                                                                 |
| Google Sheets node uses service account credentials; ensure the service account has editor access to the target spreadsheet.                                                                                                                                      | Credential setup note.                                                                                        |
| The “Process All Action Items” node includes placeholders for integration with external task or email systems (e.g., Notion, email notifications). Customize this node as needed to connect with your tools.                                                     | Customization reminder for task processing code node.                                                       |

---

This completes the detailed reference document for the "Convert Meeting Recordings to Notes & Action Items with AssemblyAI, GPT-4 & Sheets" workflow. It provides a comprehensive understanding of the workflow structure, node configurations, error considerations, and a step-by-step guide for full reconstruction.