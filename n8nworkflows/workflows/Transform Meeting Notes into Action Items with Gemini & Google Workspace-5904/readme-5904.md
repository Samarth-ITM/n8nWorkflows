Transform Meeting Notes into Action Items with Gemini & Google Workspace

https://n8nworkflows.xyz/workflows/transform-meeting-notes-into-action-items-with-gemini---google-workspace-5904


# Transform Meeting Notes into Action Items with Gemini & Google Workspace

### 1. Workflow Overview

This workflow automates the transformation of Google Meet meeting notes into structured actionable outputs using Google Gemini AI and Google Workspace services. It is designed for teams wanting to streamline meeting follow-ups by extracting action items, key decisions, summaries, and next steps from raw notes, then automatically creating tasks, sending follow-up emails, and documenting summaries.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Validation:** Receives meeting data via webhook and validates required fields.
- **1.2 Data Extraction:** Extracts and normalizes meeting details from the input.
- **1.3 AI Processing:** Uses Google Gemini AI to analyze meeting notes and extract structured information.
- **1.4 Output Parsing:** Parses the AI response into a JSON structure.
- **1.5 Action Item Handling:** Splits action items and creates corresponding Google Tasks.
- **1.6 Follow-up Email Handling:** Splits follow-up emails and sends them via Gmail.
- **1.7 Meeting Summary Creation:** Generates a Google Docs document summarizing the meeting.
- **1.8 Final Response:** Merges all outputs and responds back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block receives incoming HTTP POST requests containing meeting notes and metadata, and verifies that essential fields (`meetingNotes` and `meetingTitle`) are present.

- **Nodes Involved:**  
  - Webhook Trigger  
  - If  
  - Error Response

- **Node Details:**

  - **Webhook Trigger**  
    - Type: Webhook Trigger (HTTP POST)  
    - Config: Listens on path `/google-meet-automation` for POST requests. Responds with output after execution.  
    - Inputs: External HTTP POST request with JSON body expected to contain meeting data.  
    - Outputs: Passes raw request JSON to next node.  
    - Failures: Invalid or malformed HTTP requests; missing required fields downstream.  

  - **If**  
    - Type: Conditional Check  
    - Config: Checks existence of `meetingTitle` and `meetingNotes` in the request body using strict validation. Both must exist and be non-empty strings.  
    - Inputs: Raw JSON from Webhook Trigger.  
    - Outputs:  
      - True branch if both fields exist → proceeds to Extract Meeting Data.  
      - False branch if missing → routes to Error Response.  
    - Edge Cases: Empty strings, null values, or missing fields cause error branch.  

  - **Error Response**  
    - Type: Respond to Webhook  
    - Config: Returns JSON error message with status "error" and message about missing required fields, including timestamp.  
    - Inputs: Triggered from False branch of If node.  
    - Outputs: Sends error HTTP response and terminates workflow execution for that request.

---

#### 2.2 Data Extraction

- **Overview:**  
  Normalizes and extracts meeting-related fields from the input JSON, supplying defaults where necessary (e.g., current date/time, default duration, empty attendees list).

- **Nodes Involved:**  
  - Extract Meeting Data

- **Node Details:**

  - **Extract Meeting Data**  
    - Type: Set Node (field assignment)  
    - Config:  
      - Extracts:  
        - `meetingNotes` from `body.meetingNotes`  
        - `meetingTitle` from `body.meetingTitle`  
        - `meetingDate` from `body.meetingDate` or sets to current ISO timestamp  
        - `attendees` from `body.attendees` or empty array  
        - `duration` from `body.duration` or defaults to "60 minutes"  
    - Inputs: True branch from If node (validated input).  
    - Outputs: JSON object with normalized meeting fields for AI Processing.  
    - Edge Cases: Missing optional fields handled by defaults.

---

#### 2.3 AI Processing

- **Overview:**  
  Feeds the meeting data and notes into Google Gemini AI to extract structured actionable insights such as tasks, decisions, summaries, follow-up emails, next meeting info, and important dates.

- **Nodes Involved:**  
  - AI Meeting Processor  
  - Google Gemini AI  
  - Structured Output Parser

- **Node Details:**

  - **AI Meeting Processor**  
    - Type: LangChain LLM Chain  
    - Config:  
      - Prompt instructs AI assistant to analyze meeting notes and output a JSON object with keys: `action_items`, `key_decisions`, `summary`, `follow_up_emails`, `next_meeting`, `important_dates`  
      - Inputs use expressions to inject extracted meeting data and notes dynamically.  
      - Output parser enabled to handle structured JSON response.  
    - Inputs: Output from Extract Meeting Data.  
    - Outputs: Sends prompt to Google Gemini AI via linked node and receives raw AI response.

  - **Google Gemini AI**  
    - Type: Google Palm API Node (Gemini)  
    - Config: Uses credentials for Google Palm API project `n8n-khmuhtadin`.  
    - Inputs: From AI Meeting Processor's LLM chain.  
    - Outputs: Raw AI-generated text response.  
    - Edge Cases: API quota limits, auth errors, network timeouts, malformed AI output.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Config: Uses a JSON schema example matching the expected AI output structure to parse and validate AI response into usable JSON.  
    - Inputs: AI raw response from Google Gemini AI.  
    - Outputs: Validated, structured JSON for downstream nodes.  
    - Edge Cases: Parsing errors if AI output deviates from schema.

---

#### 2.4 Action Item Handling

- **Overview:**  
  Processes the extracted `action_items` array by splitting it into individual tasks and creating corresponding Google Tasks entries for each.

- **Nodes Involved:**  
  - Split Action Items  
  - Create Google Tasks

- **Node Details:**

  - **Split Action Items**  
    - Type: Split Out  
    - Config: Splits the `action_items` array field into individual JSON items, emitting one item per output.  
    - Inputs: Structured JSON from AI Meeting Processor.  
    - Outputs: One item per action item.  
    - Edge Cases: Empty or missing `action_items` array results in no outputs.

  - **Create Google Tasks**  
    - Type: Google Tasks Node  
    - Config:  
      - Task list set to "My Tasks" (default Google Tasks list).  
      - Task title set to individual item’s `description`.  
      - Notes field includes meeting title, assignee, priority, due date, and creation timestamp, formatted with expressions referencing extracted meeting data and current time.  
      - Uses OAuth2 credentials for Google Tasks project `n8n-khmuhtadin`.  
    - Inputs: Single action item from Split Action Items.  
    - Outputs: Task creation response, passed to merge node.  
    - Edge Cases: OAuth token expiry, API rate limits, invalid due dates.

---

#### 2.5 Follow-up Email Handling

- **Overview:**  
  Splits the AI-extracted `follow_up_emails` array into individual emails and sends each using Gmail.

- **Nodes Involved:**  
  - Split Follow-up Emails  
  - Send Follow-up Emails

- **Node Details:**

  - **Split Follow-up Emails**  
    - Type: Split Out  
    - Config: Splits the `follow_up_emails` array field into single email items.  
    - Inputs: Structured JSON from AI Meeting Processor.  
    - Outputs: One item per follow-up email.  
    - Edge Cases: Empty or missing array leads to no emails sent.

  - **Send Follow-up Emails**  
    - Type: Gmail Node  
    - Config:  
      - Sends email to `recipient` from JSON.  
      - Subject and message body populated from each item’s subject and content with HTML formatting.  
      - Includes meeting summary, date, duration in styled email content.  
      - Uses OAuth2 Gmail credentials named `contactmuhtadin`.  
    - Inputs: Single email from Split Follow-up Emails.  
    - Outputs: Email send response, passed to merge node.  
    - Edge Cases: Invalid email address, OAuth token expiry, sending quota limits.

---

#### 2.6 Meeting Summary Creation

- **Overview:**  
  Creates a Google Docs document summarizing the meeting, using the meeting title and current date in the document title.

- **Nodes Involved:**  
  - Create Meeting Summary Document

- **Node Details:**

  - **Create Meeting Summary Document**  
    - Type: Google Docs Node  
    - Config:  
      - Document title dynamically set as "Meeting Summary - {meetingTitle} - {current date}".  
      - Saved in default folder (no custom folder specified).  
      - Uses OAuth2 credentials for Google Docs project `n8n-khmuhtadin`.  
    - Inputs: Structured JSON from AI Meeting Processor (summary content expected to be incorporated downstream or manually).  
    - Outputs: Document creation response, passed to merge node.  
    - Edge Cases: OAuth issues, folder permission errors.

---

#### 2.7 Final Response

- **Overview:**  
  Merges outputs from created Google Tasks, sent emails, and created Google Docs summary into a single response and returns it to the webhook caller.

- **Nodes Involved:**  
  - Merge  
  - Success Response

- **Node Details:**

  - **Merge**  
    - Type: Merge Node  
    - Config: Waits for three inputs (Google Tasks creation, email sending, summary doc creation) and combines them.  
    - Inputs: From Create Google Tasks, Send Follow-up Emails, and Create Meeting Summary Document nodes.  
    - Outputs: Merged combined output data.

  - **Success Response**  
    - Type: Respond to Webhook  
    - Config: Returns all merged incoming items as JSON response to the original webhook request.  
    - Inputs: Merged data from Merge node.  
    - Outputs: HTTP response to caller with full results of processing.

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                          | Input Node(s)                 | Output Node(s)               | Sticky Note                                 |
|------------------------------|----------------------------------------|----------------------------------------|------------------------------|-----------------------------|---------------------------------------------|
| Webhook Trigger              | Webhook Trigger                        | Receives incoming meeting data via HTTP POST | -                            | If                          |                                             |
| If                          | If Condition                          | Validates presence of required fields | Webhook Trigger              | Extract Meeting Data, Error Response |                                             |
| Error Response              | Respond to Webhook                     | Sends error JSON response if validation fails | If                          | -                           |                                             |
| Extract Meeting Data         | Set                                   | Normalizes input fields for AI processing | If (true branch)             | AI Meeting Processor         |                                             |
| AI Meeting Processor         | LangChain Chain (LLM)                  | Processes meeting notes with AI prompt | Extract Meeting Data          | Split Action Items, Split Follow-up Emails, Create Meeting Summary Document |                                             |
| Google Gemini AI             | Google Palm API (Gemini)               | Executes AI model call                  | AI Meeting Processor          | Structured Output Parser     |                                             |
| Structured Output Parser     | LangChain Structured Output Parser     | Parses AI response JSON structure       | Google Gemini AI              | AI Meeting Processor (parsed output) |                                             |
| Split Action Items           | Split Out                             | Splits action items array to individual tasks | AI Meeting Processor          | Create Google Tasks          |                                             |
| Create Google Tasks          | Google Tasks Node                      | Creates Google Tasks from action items | Split Action Items            | Merge                       |                                             |
| Split Follow-up Emails       | Split Out                             | Splits follow-up emails array           | AI Meeting Processor          | Send Follow-up Emails        |                                             |
| Send Follow-up Emails        | Gmail Node                           | Sends follow-up emails via Gmail        | Split Follow-up Emails        | Merge                       |                                             |
| Create Meeting Summary Document | Google Docs Node                     | Creates meeting summary document         | AI Meeting Processor          | Merge                       |                                             |
| Merge                       | Merge Node                           | Combines outputs for final response      | Create Google Tasks, Send Follow-up Emails, Create Meeting Summary Document | Success Response           |                                             |
| Success Response             | Respond to Webhook                     | Sends final success response to caller  | Merge                        | -                           |                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger**  
   - Type: Webhook Trigger  
   - HTTP Method: POST  
   - Path: `google-meet-automation`  
   - Response Mode: Response Node (delayed response)  

2. **Add If Node to Validate Input**  
   - Type: If  
   - Conditions (AND):  
     - `{{ $json.body.meetingTitle }}` exists and is a non-empty string  
     - `{{ $json.body.meetingNotes }}` exists and is a non-empty string  
   - True output: Proceed with data extraction  
   - False output: Send error response  

3. **Add Respond to Webhook Node (Error Response)**  
   - Type: Respond to Webhook  
   - Respond with: JSON  
   - Response Body:  
     ```
     {
       "status": "error",
       "message": "Missing required fields: meetingNotes and meetingTitle",
       "timestamp": "{{ $now.toISO() }}"
     }
     ```
   - Connect from If node’s false branch  

4. **Add Set Node (Extract Meeting Data)**  
   - Type: Set  
   - Assign fields:  
     - `meetingNotes`: `={{ $json.body.meetingNotes }}`  
     - `meetingTitle`: `={{ $json.body.meetingTitle }}`  
     - `meetingDate`: `={{ $json.body.meetingDate || $now.toISO() }}`  
     - `attendees`: `={{ $json.body.attendees || [] }}`  
     - `duration`: `={{ $json.body.duration || '60 minutes' }}`  
   - Connect from If node’s true branch  

5. **Add LangChain Chain Node (AI Meeting Processor)**  
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
   - Prompt: Use the multi-line prompt instructing AI to parse meeting notes into the structured JSON format (include all details from the original prompt).  
   - Enable output parser.  
   - Connect from Extract Meeting Data node.  

6. **Add Google Gemini AI Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Credentials: Google Palm API OAuth2 (project `n8n-khmuhtadin`)  
   - Connect as language model backend to AI Meeting Processor node.  

7. **Add Structured Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Provide JSON schema example matching AI output structure (as in the prompt).  
   - Connect output parser to AI Meeting Processor node’s AI raw output.  

8. **Add Split Out Node (Split Action Items)**  
   - Type: Split Out  
   - Field to split out: `action_items`  
   - Connect from AI Meeting Processor node’s main output.  

9. **Add Google Tasks Node (Create Google Tasks)**  
   - Type: Google Tasks  
   - Task list: "My Tasks"  
   - Task title: `={{ $json.description }}`  
   - Notes:  
     ```
     Meeting: {{ $('Extract Meeting Data').item.json.meetingTitle }}
     Assignee: {{ $json.assignee }}
     Priority: {{ $json.priority }}
     Due Date: {{ $json.due_date }}

     Created from Google Meet automation on {{ $now.toFormat('yyyy-MM-dd HH:mm') }}
     ```  
   - Credentials: Google Tasks OAuth2 API (`n8n-khmuhtadin`)  
   - Connect from Split Action Items node.  

10. **Add Split Out Node (Split Follow-up Emails)**  
    - Type: Split Out  
    - Field to split out: `follow_up_emails`  
    - Connect from AI Meeting Processor node’s main output.  

11. **Add Gmail Node (Send Follow-up Emails)**  
    - Type: Gmail (OAuth2)  
    - Send To: `={{ $json.recipient }}`  
    - Subject: `={{ $json.subject }}`  
    - Message (HTML):  
      ```
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h2 style="color: #2563eb;">{{ $json.subject }}</h2>
        <div style="background: #f8fafc; padding: 20px; border-radius: 8px; margin: 20px 0;">
          <h3 style="margin-top: 0; color: #374151;">Meeting Summary</h3>
          <p><strong>Meeting:</strong> {{ $('Extract Meeting Data').item.json.meetingTitle }}</p>
          <p><strong>Date:</strong> {{ $('Extract Meeting Data').item.json.meetingDate }}</p>
          <p><strong>Duration:</strong> {{ $('Extract Meeting Data').item.json.duration }}</p>
        </div>
        <div style="margin: 20px 0;">
          {{ $json.content }}
        </div>
        <hr style="border: none; border-top: 1px solid #e5e7eb; margin: 30px 0;">
        <p style="color: #6b7280; font-size: 14px;">
          This email was automatically generated from your Google Meet notes.<br>
          Generated on {{ $now.toFormat('MMMM dd, yyyy \\at HH:mm') }}
        </p>
      </div>
      ```  
    - Credentials: Gmail OAuth2 (`contactmuhtadin`)  
    - Connect from Split Follow-up Emails node.  

12. **Add Google Docs Node (Create Meeting Summary Document)**  
    - Type: Google Docs  
    - Title: `=('Meeting Summary - ' + $('Extract Meeting Data').item.json.meetingTitle + ' - ' + $now.toFormat('yyyy-MM-dd'))`  
    - Folder: Default (no custom folder)  
    - Credentials: Google Docs OAuth2 API (`n8n-khmuhtadin`)  
    - Connect from AI Meeting Processor node.  

13. **Add Merge Node**  
    - Type: Merge  
    - Number of inputs: 3  
    - Connect inputs from:  
      - Create Google Tasks  
      - Send Follow-up Emails  
      - Create Meeting Summary Document  

14. **Add Respond to Webhook Node (Success Response)**  
    - Type: Respond to Webhook  
    - Respond with: All incoming items  
    - Connect from Merge node.  

15. **Verify and test the full workflow end-to-end** ensuring credentials are valid and rate limits are respected.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                               |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| This workflow relies on Google Workspace OAuth2 credentials configured for Google Tasks, Gmail, and Google Docs. | Ensure you have set up OAuth2 credentials in n8n for these Google services with appropriate scopes.                           |
| The Google Gemini AI node requires a valid Google Palm API key/project (`n8n-khmuhtadin`).                       | Google Cloud Console → APIs & Services → Credentials.                                                                         |
| The AI prompt is crafted to enforce JSON-only output for easy parsing.                                           | Adjust prompt carefully if modifying to maintain JSON output integrity.                                                       |
| For Gmail sending, ensure the `contactmuhtadin` OAuth2 credential has "Gmail Send" scope enabled.                | Gmail API scopes: `https://www.googleapis.com/auth/gmail.send`.                                                              |
| Default task list is "My Tasks". Modify in Create Google Tasks node if you want a different task list.           | https://developers.google.com/tasks/reference/rest/v1/tasklists                                                                    |
| Current workflow uses default Google Docs folder; customize Folder ID if you want to organize documents.        | https://developers.google.com/drive/api/v3/reference/files#resource                                                          |
| Workflow is designed to respond synchronously to webhook triggers, making it suitable for HTTP integrations.    | Consider async handling or error retry strategies for production robustness.                                                  |

---

Disclaimer: The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected content. All manipulated data is legal and public.