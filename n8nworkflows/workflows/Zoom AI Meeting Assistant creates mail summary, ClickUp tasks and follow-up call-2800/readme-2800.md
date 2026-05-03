Zoom AI Meeting Assistant creates mail summary, ClickUp tasks and follow-up call

https://n8nworkflows.xyz/workflows/zoom-ai-meeting-assistant-creates-mail-summary--clickup-tasks-and-follow-up-call-2800


# Zoom AI Meeting Assistant creates mail summary, ClickUp tasks and follow-up call

### 1. Workflow Overview

This workflow automates the processing of Zoom meetings from the last 24 hours by retrieving meeting data and transcripts, generating an AI-based meeting summary, emailing the summary to participants, and creating actionable tasks and follow-up calendar events. It is designed for users with Zoom Workspace Pro accounts with Cloud Recording/Transcripts enabled and integrates with ClickUp for task management and Microsoft Outlook for calendar events.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 Zoom Data Retrieval**: Fetch Zoom meetings from the last 24 hours and filter relevant recordings/transcripts.
- **1.3 Transcript Processing**: Download and extract transcript text, format it for AI consumption.
- **1.4 Meeting Summary Generation and Emailing**: Use AI to create a meeting summary, format it as HTML, and send it via email to participants.
- **1.5 Task and Follow-up Call Creation**: Use AI to parse transcript content to generate tasks and schedule follow-up meetings, integrating with ClickUp and Outlook via sub-workflows.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Starts the workflow manually. Can be replaced by scheduled or webhook triggers.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution on user command.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Connected to Zoom meeting data retrieval node.  
    - Edge Cases: None.  
    - Notes: Can be replaced with scheduled or webhook trigger for automation.

#### 2.2 Zoom Data Retrieval

- **Overview:**  
  Retrieves all scheduled Zoom meetings, filters those within the last 24 hours, and obtains recording/transcript URLs.

- **Nodes Involved:**  
  - Zoom: Get data of last meeting  
  - Filter: Last 24 hours  
  - Zoom: Get transcripts data  
  - Filter transcript URL  
  - Filter: Only 1 item  
  - No Recording/Transcript available

- **Node Details:**  
  - **Zoom: Get data of last meeting**  
    - Type: Zoom node  
    - Role: Fetches all scheduled meetings via Zoom API using OAuth2.  
    - Configuration: Operation “getAll”, filter type “scheduled”, return all meetings.  
    - Inputs: Manual trigger  
    - Outputs: Filter: Last 24 hours  
    - Edge Cases: API rate limits, OAuth token expiry, empty meeting list.

  - **Filter: Last 24 hours**  
    - Type: Filter  
    - Role: Filters meetings to only those with start_time within last 24 hours.  
    - Configuration: Conditions on start_time >= now - 24h and <= now.  
    - Inputs: Zoom meetings list  
    - Outputs: Zoom: Get transcripts data  
    - Edge Cases: No meetings found in last 24 hours.

  - **Zoom: Get transcripts data**  
    - Type: HTTP Request  
    - Role: Retrieves recordings data for a specific meeting ID.  
    - Configuration: URL constructed dynamically from meeting ID, OAuth2 authentication.  
    - Inputs: Filter: Last 24 hours (filtered meeting)  
    - Outputs: Filter transcript URL, No Recording/Transcript available (on error)  
    - Edge Cases: No recordings/transcripts available, API errors.

  - **Filter transcript URL**  
    - Type: Set  
    - Role: Extracts the transcript download URL from recording files.  
    - Configuration: Assigns `transcript_file` variable by finding recording file with file_type 'TRANSCRIPT'.  
    - Inputs: Zoom: Get transcripts data  
    - Outputs: Filter: Only 1 item  
    - Edge Cases: Transcript file missing or undefined.

  - **Filter: Only 1 item**  
    - Type: SplitInBatches  
    - Role: Ensures processing of a single transcript file at a time.  
    - Configuration: Default batch size 1.  
    - Inputs: Filter transcript URL  
    - Outputs: Zoom: Get transcript file  
    - Edge Cases: Multiple transcript files, empty batch.

  - **No Recording/Transcript available**  
    - Type: Stop and Error  
    - Role: Stops workflow with error message if no transcript is found.  
    - Configuration: Error message from Zoom API error cause.  
    - Inputs: Zoom: Get transcripts data (on error)  
    - Outputs: None  
    - Edge Cases: Triggered when no transcript is available.

#### 2.3 Transcript Processing

- **Overview:**  
  Downloads the transcript file, extracts text content, formats it for AI processing, and retrieves participant data.

- **Nodes Involved:**  
  - Zoom: Get transcript file  
  - Extract text from transcript file  
  - Format transcript text  
  - Zoom: Get participants data

- **Node Details:**  
  - **Zoom: Get transcript file**  
    - Type: HTTP Request  
    - Role: Downloads the transcript file from the URL extracted earlier.  
    - Configuration: URL from `transcript_file`, OAuth2 authentication.  
    - Inputs: Filter: Only 1 item  
    - Outputs: Extract text from transcript file  
    - Edge Cases: Download failure, invalid URL.

  - **Extract text from transcript file**  
    - Type: Extract From File  
    - Role: Extracts plain text from the downloaded transcript file.  
    - Configuration: Operation “text”.  
    - Inputs: Zoom: Get transcript file  
    - Outputs: Format transcript text  
    - Edge Cases: File format unsupported, extraction failure.

  - **Format transcript text**  
    - Type: Set  
    - Role: Processes raw transcript text to remove headers and format into a clean string for AI input.  
    - Configuration: Splits text by double line breaks, removes first line of each block, joins lines with spaces and newlines.  
    - Inputs: Extract text from transcript file  
    - Outputs: Zoom: Get participants data  
    - Edge Cases: Unexpected transcript formatting.

  - **Zoom: Get participants data**  
    - Type: HTTP Request  
    - Role: Retrieves participant details for the meeting via Zoom API.  
    - Configuration: URL uses meeting ID, OAuth2 authentication.  
    - Inputs: Format transcript text  
    - Outputs: Create meeting summary  
    - Edge Cases: API errors, empty participant list.

#### 2.4 Meeting Summary Generation and Emailing

- **Overview:**  
  Uses AI to generate a formal meeting summary from transcript and participant data, formats it as HTML, and sends it by email.

- **Nodes Involved:**  
  - Create meeting summary  
  - Sort for mail delivery  
  - Format to html  
  - Send meeting summary

- **Node Details:**  
  - **Create meeting summary**  
    - Type: LangChain Agent (AI Agent)  
    - Role: Generates meeting minutes including summary, tasks, and important dates from transcript and participant data.  
    - Configuration: System prompt includes meeting date, participants, transcript, and output format instructions.  
    - Inputs: Zoom: Get participants data, Format transcript text  
    - Outputs: Sort for mail delivery, Create tasks and follow-up call  
    - Edge Cases: AI model errors, incomplete data, prompt failures.

  - **Sort for mail delivery**  
    - Type: Set  
    - Role: Prepares email fields: subject, recipient (first participant email), and body (AI output).  
    - Configuration: Subject includes meeting topic and formatted start time; recipient is first participant email; body is AI output.  
    - Inputs: Create meeting summary  
    - Outputs: Format to html  
    - Edge Cases: Missing participant email, formatting errors.

  - **Format to html**  
    - Type: Code  
    - Role: Converts plain text meeting summary into styled HTML email content.  
    - Configuration: JavaScript code splits summary into sections (participants, summary, tasks, dates) and formats with inline CSS.  
    - Inputs: Sort for mail delivery  
    - Outputs: Send meeting summary  
    - Edge Cases: Unexpected text format, empty sections.

  - **Send meeting summary**  
    - Type: Email Send  
    - Role: Sends the formatted meeting summary email via SMTP.  
    - Configuration: Uses SMTP credentials, from email fixed, to and subject from previous node, HTML body.  
    - Inputs: Format to html  
    - Outputs: None  
    - Edge Cases: SMTP authentication failure, email delivery errors.

#### 2.5 Task and Follow-up Call Creation

- **Overview:**  
  Uses AI to extract tasks and follow-up meeting details from the transcript, then creates tasks in ClickUp and schedules follow-up calls in Outlook.

- **Nodes Involved:**  
  - Create tasks and follow-up call  
  - Create tasks  
  - ClickUp  
  - Create follow-up call  
  - Anthropic Chat Model  
  - Think  
  - Split Out (disabled)  
  - Execute Workflow Trigger (disabled)  
  - Sticky Note (sub-workflow info)

- **Node Details:**  
  - **Create tasks and follow-up call**  
    - Type: LangChain Agent (AI Agent)  
    - Role: Parses transcript to identify tasks and follow-up meeting info, formats data for downstream tools.  
    - Configuration: Detailed system prompt instructs AI to create clear tasks for the user and schedule follow-up meetings with defaults if missing info.  
    - Inputs: Create meeting summary, Anthropic Chat Model, Think (AI chain)  
    - Outputs: Create tasks, Create follow-up call  
    - Edge Cases: AI misinterpretation, missing data, formatting errors.

  - **Create tasks**  
    - Type: LangChain Tool Workflow  
    - Role: Sub-workflow node that creates tasks in ClickUp based on AI output.  
    - Configuration: Input schema expects array of tasks with name, description, due date, priority, and project name.  
    - Inputs: Create tasks and follow-up call  
    - Outputs: Create tasks and follow-up call (feedback loop)  
    - Edge Cases: Sub-workflow errors, invalid task data.

  - **ClickUp**  
    - Type: ClickUp node  
    - Role: Creates tasks in ClickUp using OAuth2 credentials.  
    - Configuration: Uses list, team, space, folder IDs; task name, content, due date from input.  
    - Inputs: Split Out (disabled in main flow)  
    - Outputs: None  
    - Edge Cases: API errors, OAuth token expiry.

  - **Create follow-up call**  
    - Type: Microsoft Outlook Tool  
    - Role: Creates calendar event for follow-up meeting in Outlook.  
    - Configuration: Uses AI-provided meeting name, start/end times, participants; calendar ID and timezone set.  
    - Inputs: Create tasks and follow-up call  
    - Outputs: None  
    - Edge Cases: OAuth errors, invalid date/time formats.

  - **Anthropic Chat Model**  
    - Type: LangChain Language Model  
    - Role: Provides AI language model backend for agents.  
    - Configuration: Uses Claude 3.7 Sonnet model with Anthropic API credentials.  
    - Inputs: Create meeting summary, Create tasks and follow-up call  
    - Outputs: AI agent nodes  
    - Edge Cases: API limits, latency.

  - **Think**  
    - Type: LangChain Tool Think  
    - Role: Intermediate AI reasoning step in chain.  
    - Configuration: Default.  
    - Inputs: Create meeting summary  
    - Outputs: Create tasks and follow-up call  
    - Edge Cases: AI processing errors.

  - **Execute Workflow Trigger** (disabled)  
    - Type: Execute Workflow Trigger  
    - Role: Placeholder for sub-workflow execution trigger.  
    - Configuration: Disabled.  
    - Inputs: None  
    - Outputs: Split Out  
    - Edge Cases: None.

  - **Split Out** (disabled)  
    - Type: Split Out  
    - Role: Splits array of tasks for individual processing.  
    - Configuration: Field to split out is `query.items`.  
    - Inputs: Execute Workflow Trigger  
    - Outputs: ClickUp  
    - Edge Cases: Disabled in main flow.

  - **Sticky Note**  
    - Content: "Sub workflow: Create Task in ClickUp"  
    - Context: Indicates that task creation is handled by a separate sub-workflow.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                              | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                                                        |
|--------------------------------|----------------------------------|----------------------------------------------|----------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                   | Starts workflow manually                      | None                             | Zoom: Get data of last meeting        |                                                                                                                                   |
| Zoom: Get data of last meeting  | Zoom                            | Retrieves all scheduled Zoom meetings        | When clicking ‘Test workflow’    | Filter: Last 24 hours                  |                                                                                                                                   |
| Filter: Last 24 hours           | Filter                          | Filters meetings within last 24 hours        | Zoom: Get data of last meeting   | Zoom: Get transcripts data            |                                                                                                                                   |
| Zoom: Get transcripts data      | HTTP Request                    | Retrieves recordings/transcripts for meeting | Filter: Last 24 hours            | Filter transcript URL, No Recording/Transcript available |                                                                                                                                   |
| Filter transcript URL           | Set                             | Extracts transcript download URL             | Zoom: Get transcripts data       | Filter: Only 1 item                    |                                                                                                                                   |
| Filter: Only 1 item             | SplitInBatches                  | Processes one transcript file at a time      | Filter transcript URL            | Zoom: Get transcript file             |                                                                                                                                   |
| Zoom: Get transcript file       | HTTP Request                    | Downloads transcript file                     | Filter: Only 1 item              | Extract text from transcript file     |                                                                                                                                   |
| Extract text from transcript file | Extract From File               | Extracts plain text from transcript file     | Zoom: Get transcript file        | Format transcript text                 |                                                                                                                                   |
| Format transcript text          | Set                             | Cleans and formats transcript text           | Extract text from transcript file | Zoom: Get participants data           |                                                                                                                                   |
| Zoom: Get participants data     | HTTP Request                    | Retrieves meeting participants                | Format transcript text           | Create meeting summary                 |                                                                                                                                   |
| Create meeting summary          | LangChain Agent                 | Generates AI meeting summary                  | Zoom: Get participants data      | Sort for mail delivery, Create tasks and follow-up call |                                                                                                                                   |
| Sort for mail delivery          | Set                             | Prepares email fields (subject, to, body)    | Create meeting summary           | Format to html                        |                                                                                                                                   |
| Format to html                 | Code                            | Converts summary text to styled HTML          | Sort for mail delivery           | Send meeting summary                  |                                                                                                                                   |
| Send meeting summary            | Email Send                     | Sends meeting summary email                    | Format to html                  | None                                 |                                                                                                                                   |
| Create tasks and follow-up call | LangChain Agent                 | Parses transcript to create tasks and meetings | Create meeting summary, Anthropic Chat Model, Think | Create tasks, Create follow-up call  |                                                                                                                                   |
| Create tasks                   | LangChain Tool Workflow         | Sub-workflow to create tasks in ClickUp       | Create tasks and follow-up call | Create tasks and follow-up call       | Sub workflow: Create Task in ClickUp                                                                                              |
| ClickUp                       | ClickUp                        | Creates tasks in ClickUp                        | Split Out (disabled)             | None                                 |                                                                                                                                   |
| Create follow-up call          | Microsoft Outlook Tool          | Creates follow-up calendar event in Outlook   | Create tasks and follow-up call | None                                 |                                                                                                                                   |
| Anthropic Chat Model           | LangChain Language Model        | AI language model backend                       | Create meeting summary, Create tasks and follow-up call | Create meeting summary, Create tasks and follow-up call |                                                                                                                                   |
| Think                         | LangChain Tool Think            | Intermediate AI reasoning step                  | Create meeting summary           | Create tasks and follow-up call       |                                                                                                                                   |
| Execute Workflow Trigger       | Execute Workflow Trigger (disabled) | Placeholder for sub-workflow trigger           | None                           | Split Out (disabled)                  |                                                                                                                                   |
| Split Out                     | Split Out (disabled)             | Splits tasks array for individual processing   | Execute Workflow Trigger (disabled) | ClickUp (disabled)                   |                                                                                                                                   |
| No Recording/Transcript available | Stop and Error                 | Stops workflow if no transcript available      | Zoom: Get transcripts data (on error) | None                             |                                                                                                                                   |
| Sticky Note                   | Sticky Note                     | Workflow description and instructions          | None                           | None                                 | Welcome to my Zoom AI Meeting Assistant Workflow! See LinkedIn contact and documentation links.                                   |
| Sticky Note                   | Sticky Note                     | Sub-workflow note                               | None                           | None                                 | Sub workflow: Create Task in ClickUp                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.  
   - No parameters needed.

2. **Add Zoom Node to Get Meetings**  
   - Type: Zoom  
   - Operation: getAll  
   - Filter: scheduled meetings only  
   - Return All: true  
   - Authentication: OAuth2 with Zoom Workspace Pro credentials.

3. **Add Filter Node to Select Meetings in Last 24 Hours**  
   - Type: Filter  
   - Condition 1: `start_time` >= current time minus 24 hours  
   - Condition 2: `start_time` <= current time

4. **Add HTTP Request Node to Get Meeting Recordings**  
   - Type: HTTP Request  
   - URL: `https://api.zoom.us/v2/meetings/{{ $json.id }}/recordings`  
   - Authentication: OAuth2 Zoom credentials.

5. **Add Set Node to Extract Transcript URL**  
   - Type: Set  
   - Assign variable `transcript_file` to the download_url of recording_files where file_type is 'TRANSCRIPT'.

6. **Add SplitInBatches Node**  
   - Type: SplitInBatches  
   - Batch Size: 1  
   - Purpose: Process one transcript file at a time.

7. **Add HTTP Request Node to Download Transcript File**  
   - Type: HTTP Request  
   - URL: `={{ $json.transcript_file }}`  
   - Authentication: OAuth2 Zoom credentials.

8. **Add Extract From File Node**  
   - Type: Extract From File  
   - Operation: text  
   - Purpose: Extract plain text from transcript file.

9. **Add Set Node to Format Transcript Text**  
   - Type: Set  
   - Use JavaScript expression to clean and join transcript text blocks, removing headers.

10. **Add HTTP Request Node to Get Participants Data**  
    - Type: HTTP Request  
    - URL: `https://api.zoom.us/v2/past_meetings/{{ $json.id }}/participants`  
    - Authentication: OAuth2 Zoom credentials.

11. **Add LangChain Agent Node to Create Meeting Summary**  
    - Type: LangChain Agent  
    - Prompt: Provide meeting date, participants, transcript; instruct AI to create formal meeting minutes with summary, tasks, and dates.  
    - AI Model: Connect to Anthropic Chat Model node.

12. **Add Set Node to Prepare Email Fields**  
    - Type: Set  
    - Assign `subject` with meeting topic and formatted date/time.  
    - Assign `to` with first participant email.  
    - Assign `body` with AI output.

13. **Add Code Node to Format Email as HTML**  
    - Type: Code  
    - JavaScript: Parse body text into sections and generate styled HTML email content.

14. **Add Email Send Node**  
    - Type: Email Send  
    - SMTP Credentials: Configure with valid SMTP account.  
    - From Email: Fixed sender email.  
    - To Email, Subject, HTML Body: From previous node.

15. **Add LangChain Agent Node to Create Tasks and Follow-up Call**  
    - Type: LangChain Agent  
    - Prompt: Instruct AI to extract tasks and follow-up meeting info from transcript, format for task creation and calendar event creation.  
    - Connect to Anthropic Chat Model node.

16. **Add LangChain Tool Workflow Node for Task Creation**  
    - Type: LangChain Tool Workflow  
    - Workflow ID: Link to sub-workflow that creates tasks in ClickUp.  
    - Input Schema: Array of tasks with name, description, due date, priority, project name.

17. **Add ClickUp Node**  
    - Type: ClickUp  
    - OAuth2 Credentials: Configure with ClickUp account.  
    - Parameters: List, team, space, folder IDs; task name, content, due date from input.

18. **Add Microsoft Outlook Tool Node to Create Follow-up Call**  
    - Type: Microsoft Outlook Tool  
    - OAuth2 Credentials: Configure with Outlook account.  
    - Parameters: Subject, start and end date/time, participants, calendar ID, timezone.

19. **Add LangChain Language Model Node (Anthropic Chat Model)**  
    - Type: LangChain Language Model  
    - Model: Claude 3.7 Sonnet or equivalent.  
    - Credentials: Anthropic API.

20. **Add LangChain Tool Think Node**  
    - Type: LangChain Tool Think  
    - Purpose: Intermediate reasoning step in AI chain.

21. **Add Stop and Error Node**  
    - Type: Stop and Error  
    - Triggered if no transcript is available.  
    - Error message from Zoom API response.

22. **Connect Nodes According to Workflow Logic**  
    - Manual Trigger → Zoom: Get data of last meeting → Filter: Last 24 hours → Zoom: Get transcripts data → Filter transcript URL → Filter: Only 1 item → Zoom: Get transcript file → Extract text from transcript file → Format transcript text → Zoom: Get participants data → Create meeting summary → Sort for mail delivery → Format to html → Send meeting summary  
    - Create meeting summary → Create tasks and follow-up call → Create tasks → ClickUp  
    - Create tasks and follow-up call → Create follow-up call  
    - Anthropic Chat Model connected as AI model for agents  
    - Think node connected between Create meeting summary and Create tasks and follow-up call.

23. **Configure Credentials**  
    - Zoom OAuth2 with Workspace Pro account and Cloud Recording enabled.  
    - SMTP credentials for email sending.  
    - ClickUp OAuth2 credentials.  
    - Microsoft Outlook OAuth2 credentials.  
    - Anthropic API credentials for AI nodes.

24. **Set Up Sub-Workflows**  
    - Create a separate workflow for task creation in ClickUp with expected input schema.  
    - Link sub-workflow in LangChain Tool Workflow node by workflow ID.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Important: Requires Zoom Workspace Pro account with Cloud Recording and Transcripts enabled.              | https://docs.n8n.io/integrations/builtin/credentials/zoom/                                              |
| Microsoft Outlook OAuth2 credentials needed for calendar event creation.                                   | https://docs.n8n.io/integrations/builtin/credentials/microsoft/                                         |
| ClickUp OAuth2 credentials needed for task creation.                                                      | https://docs.n8n.io/integrations/builtin/credentials/clickup/                                           |
| AI API access required (OpenAI, Anthropic, Google, or Ollama).                                            |                                                                                                         |
| SMTP credentials required for sending emails.                                                             |                                                                                                         |
| Sub-workflows must be created separately and linked via “Execute workflow trigger” or LangChain Tool Workflow nodes. |                                                                                                         |
| Workflow author contact: [LinkedIn - Friedemann Schuetz](https://www.linkedin.com/in/friedemann-schuetz)  |                                                                                                         |
| Workflow sequence and detailed instructions are provided in sticky notes within the workflow.             |                                                                                                         |

---

This document provides a comprehensive understanding of the Zoom AI Meeting Assistant workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.