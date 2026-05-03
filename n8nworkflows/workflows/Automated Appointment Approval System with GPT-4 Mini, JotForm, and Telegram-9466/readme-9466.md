Automated Appointment Approval System with GPT-4 Mini, JotForm, and Telegram

https://n8nworkflows.xyz/workflows/automated-appointment-approval-system-with-gpt-4-mini--jotform--and-telegram-9466


# Automated Appointment Approval System with GPT-4 Mini, JotForm, and Telegram

### 1. Workflow Overview

This workflow titled **"Automated Appointment Approval System with GPT-4 Mini, JotForm, and Telegram"** automates the entire appointment request and approval process for a doctor's office using form submissions and AI assistance. It captures appointment requests via JotForm, routes the details for human approval via Telegram, then uses GPT-4 Mini to generate personalized email responses confirming or declining the appointment. Upon approval, the appointment is logged in Google Sheets and a confirmation email is sent; on rejection, the submission is deleted, and a polite rescheduling email is sent.

**Target Use Cases:**  
- Medical offices or service providers needing automated appointment scheduling with human-in-the-loop approval.  
- Teams requiring seamless, professional communication with patients/clients without manual email drafting.  
- Workflows integrating form inputs, messaging platforms, AI-generated content, and record-keeping.

**Logical Blocks:**

- **1.1 Input Reception:** Capturing appointment requests from JotForm.  
- **1.2 Data Parsing:** Extracting necessary appointment and patient details from the form submission.  
- **1.3 Approval Notification:** Sending appointment details to a Telegram chat for approval or decline decision.  
- **1.4 AI Email Generation:** Using GPT-4 Mini to generate either confirmation or rescheduling emails based on approval status.  
- **1.5 Approval Routing:** Conditional routing to either approval or rejection paths.  
- **1.6 Approval Path:** Logging approved appointments to Google Sheets and sending confirmation emails.  
- **1.7 Rejection Path:** Deleting rejected submissions from JotForm and sending rejection/reschedule emails via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new appointment submissions from a specified JotForm form via webhook.

- **Nodes Involved:**  
  - Appointment Request Form Trigger

- **Node Details:**  

  - **Appointment Request Form Trigger**  
    - *Type:* JotForm Trigger  
    - *Role:* Listens for new submissions on a specified JotForm form.  
    - *Config:* Uses the form ID (`YOUR_JOTFORM_FORM_ID`) and webhook ID (placeholder) to receive submission data. Does not limit answers to only form answers (raw request included).  
    - *Expressions:* None; outputs full form submission JSON.  
    - *Input/Output:* No input; outputs raw submission data to next node.  
    - *Edge Cases:*  
      - Webhook misconfiguration leading to missed submissions.  
      - Form changes (field renaming) affecting downstream parsing.  
      - API limits or downtime on JotForm.  
    - *Credentials:* Requires JotForm API credentials configured in n8n.

---

#### 1.2 Data Parsing

- **Overview:**  
  Extracts and assigns key appointment and patient details from the raw form submission for easier downstream use.

- **Nodes Involved:**  
  - Parse: Extract Appointment Details

- **Node Details:**  

  - **Parse: Extract Appointment Details**  
    - *Type:* Set Node  
    - *Role:* Maps form fields from raw JSON to structured variables like formID, submissionID, patient's name, phone, email, visit type, and appointment date/time.  
    - *Config:* Assigns variables using expressions referencing `$json` keys for rawRequest fields (e.g., Name, Phone Number, E-mail).  
    - *Input/Output:* Input from JotForm trigger; output is enriched JSON with clear, accessible fields.  
    - *Edge Cases:*  
      - Missing fields in form submission causing undefined values.  
      - Changes in form field keys breaking expressions.  
    - *Version Requirements:* Uses Set node version supporting advanced expressions (v3.4).  

---

#### 1.3 Approval Notification

- **Overview:**  
  Sends the extracted appointment details to a Telegram chat where a decision-maker can approve or decline the request interactively.

- **Nodes Involved:**  
  - Notify for Approval or Decline

- **Node Details:**  

  - **Notify for Approval or Decline**  
    - *Type:* Telegram Node  
    - *Role:* Sends a formatted message with appointment details, waits for an approval decision (double approval type).  
    - *Config:*  
      - Chat ID set to `YOUR_TELEGRAM_CHAT_ID`.  
      - Message includes patient's full name, phone number, first-time visit status, appointment date/time, and duration using expressions.  
      - Uses `sendAndWait` operation to await approval result.  
      - Approval options configured for double confirmation type.  
    - *Input/Output:* Input from data parsing node; outputs approval status (`approved` true/false) and related data.  
    - *Edge Cases:*  
      - Telegram API failures or chat ID misconfiguration.  
      - User not responding or delays in approval.  
      - Approval response format issues.  
    - *Credentials:* Requires Telegram Bot credentials configured in n8n.  

---

#### 1.4 AI Email Generation

- **Overview:**  
  Invokes GPT-4 Mini model through Langchain to generate a professional, personalized email for appointment confirmation or rescheduling based on approval status.

- **Nodes Involved:**  
  - Generate: Appointment Response Email  
  - OpenAI Chat Model6  
  - Structured Output Parser

- **Node Details:**  

  - **Generate: Appointment Response Email**  
    - *Type:* Langchain Agent Node  
    - *Role:* Defines prompt logic for the AI agent to generate a JSON-formatted email response reflecting approval or rejection.  
    - *Config:*  
      - System message instructs the AI to extract patient/appointment details and compose either a confirmation or reschedule email in valid JSON with fields: email, subject, html_body.  
      - Input text includes approval status and raw form data serialized as JSON string.  
      - Output parser enabled to enforce structured JSON output format.  
    - *Input/Output:* Input from Telegram approval node; outputs AI-generated email JSON.  
    - *Edge Cases:*  
      - AI output not conforming to JSON schema causing parsing errors.  
      - API call failures or rate limits with OpenAI.  
      - Unexpected input data causing irrelevant outputs.  

  - **OpenAI Chat Model6**  
    - *Type:* Langchain OpenAI Chat Model  
    - *Role:* Implements GPT-4 Mini model for the Langchain Agent.  
    - *Config:* Model set to `gpt-4.1-mini`.  
    - *Input/Output:* Input from Generate node prompt; outputs AI text response.  
    - *Edge Cases:* OpenAI API quota or network issues.  

  - **Structured Output Parser**  
    - *Type:* Langchain Output Parser Structured  
    - *Role:* Parses AI output text into structured JSON using example schema for validation.  
    - *Config:* Example JSON schema provided for email, subject, and html_body.  
    - *Input/Output:* Input from AI model output; outputs JSON object.  
    - *Edge Cases:* Parsing failures if AI output deviates from expected format.

---

#### 1.5 Approval Routing

- **Overview:**  
  Routes the workflow based on the approval status flag to either approved or rejected branches.

- **Nodes Involved:**  
  - Condition: Check Approval Status

- **Node Details:**  

  - **Condition: Check Approval Status**  
    - *Type:* If Node  
    - *Role:* Checks if the approval status (`approved`) from Telegram is string `"true"`.  
    - *Config:* Condition compares value from `Notify for Approval or Decline` node's `.data.approved` field to `"true"` string.  
    - *Input/Output:* Input from AI email generation node; outputs to either approval or rejection path.  
    - *Edge Cases:*  
      - Approval status missing or malformed causing false negatives.  
      - Case sensitivity or type mismatch (string vs boolean).  
    - *Version:* v2.2 or above recommended for improved condition handling.

---

#### 1.6 Approval Path

- **Overview:**  
  For approved appointments, logs data into Google Sheets and sends a confirmation email via Gmail.

- **Nodes Involved:**  
  - Log: Record Appointment in sheets  
  - Send: Confirmation Email

- **Node Details:**  

  - **Log: Record Appointment in sheets**  
    - *Type:* Google Sheets Node  
    - *Role:* Stores appointment and patient details in a specified Google Sheet, either appending or updating based on email matching.  
    - *Config:*  
      - Maps patient's name, email, duration, phone, date/time, and first-time visit flag to sheet columns.  
      - Uses sheet name `Sheet1` (gid=0) and Google Sheet ID placeholder.  
      - Matching column set to `email` to avoid duplicates.  
    - *Input/Output:* Input from Condition Node's approved path; outputs to send email node.  
    - *Edge Cases:*  
      - Google Sheets API quota or permission errors.  
      - Sheet schema changes causing column mismatches.  
    - *Credentials:* Requires Google Sheets OAuth2 credentials.

  - **Send: Confirmation Email**  
    - *Type:* Gmail Node  
    - *Role:* Sends the AI-generated confirmation email to the patient.  
    - *Config:*  
      - Recipient email, subject, and HTML body taken from AI output node.  
      - Uses Gmail OAuth2 credentials.  
    - *Input/Output:* Input from Google Sheets logging node.  
    - *Edge Cases:*  
      - Gmail sending quota or authentication errors.  
      - Invalid email addresses causing send failures.

---

#### 1.7 Rejection Path

- **Overview:**  
  For rejected appointments, deletes the submission from JotForm and sends a rescheduling or rejection email.

- **Nodes Involved:**  
  - Delete Rejected Appointment  
  - Send: Rejection or Reschedule Email

- **Node Details:**  

  - **Delete Rejected Appointment**  
    - *Type:* HTTP Request Node  
    - *Role:* Sends a DELETE request to JotForm API to remove the rejected submission.  
    - *Config:*  
      - URL constructed dynamically with submission ID from parsed data.  
      - Query parameter includes JotForm API key placeholder.  
      - Method: DELETE  
    - *Input/Output:* Input from Condition Node's rejected path; outputs to send rejection email node.  
    - *Edge Cases:*  
      - API key invalid or missing permissions.  
      - Submission already deleted causing 404 errors.  
      - Network or API downtime.  
    - *Credentials:* Requires JotForm API key configured in node parameters.

  - **Send: Rejection or Reschedule Email**  
    - *Type:* Gmail Node  
    - *Role:* Sends AI-generated rejection/reschedule email to the patient.  
    - *Config:* Like confirmation email node, uses AI output fields for recipient, subject, and HTML body.  
    - *Input/Output:* Input from Delete node.  
    - *Edge Cases:* Same as confirmation email node.

---

### 3. Summary Table

| Node Name                          | Node Type                               | Functional Role                                   | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                      |
|-----------------------------------|---------------------------------------|--------------------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Appointment Request Form Trigger   | JotForm Trigger                       | Captures new appointment submissions             | None                             | Parse: Extract Appointment Details |                                                                                                |
| Parse: Extract Appointment Details | Set                                   | Parses and structures appointment form data      | Appointment Request Form Trigger | Notify for Approval or Decline   |                                                                                                |
| Notify for Approval or Decline     | Telegram                              | Sends appointment details for human approval     | Parse: Extract Appointment Details | Generate: Appointment Response Email |                                                                                                |
| Generate: Appointment Response Email | Langchain Agent                      | Generates confirmation or reschedule emails with AI | Notify for Approval or Decline   | Condition: Check Approval Status |                                                                                                |
| OpenAI Chat Model6                 | Langchain OpenAI Chat Model          | Provides GPT-4 Mini language model for AI generation | Generate: Appointment Response Email (prompt) | Generate: Appointment Response Email (response) |                                                                                                |
| Structured Output Parser           | Langchain Output Parser Structured   | Parses AI output into structured JSON             | OpenAI Chat Model6               | Generate: Appointment Response Email |                                                                                                |
| Condition: Check Approval Status  | If                                   | Routes workflow based on approval status          | Generate: Appointment Response Email | Log: Record Appointment in sheets, Delete Rejected Appointment |                                                                                                |
| Log: Record Appointment in sheets | Google Sheets                        | Logs approved appointment data into Google Sheets | Condition: Check Approval Status | Send: Confirmation Email          |                                                                                                |
| Send: Confirmation Email          | Gmail                                | Sends confirmation email to patient               | Log: Record Appointment in sheets | None                            |                                                                                                |
| Delete Rejected Appointment       | HTTP Request                        | Deletes rejected submissions from JotForm         | Condition: Check Approval Status | Send: Rejection or Reschedule Email |                                                                                                |
| Send: Rejection or Reschedule Email | Gmail                              | Sends rejection or reschedule email to patient    | Delete Rejected Appointment      | None                            |                                                                                                |
| Sticky Note2                      | Sticky Note                         | Workflow overview, instructions, and key benefits | None                             | None                            | # AI Powered All Purpose Appointment System via JotForm; [Get the JotForm](https://www.jotform.com/?partner=roshanramanidev) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a JotForm Trigger Node:**  
   - Type: JotForm Trigger  
   - Parameters:  
     - Form ID: Set to your actual JotForm form ID capturing appointment requests.  
     - Webhook ID: Auto-generated or set for webhook connection.  
   - Credentials: Configure with your JotForm API credentials.

2. **Add a Set Node named "Parse: Extract Appointment Details":**  
   - Purpose: Extract and map key fields from the raw JotForm submission JSON.  
   - Assign variables (using expressions referencing `$json`):  
     - formID = `{{$json.formID}}`  
     - formTitle = `{{$json.formTitle}}`  
     - submissionID = `{{$json.submissionID}}`  
     - rawRequest.Name = `{{$json.rawRequest.Name}}`  
     - rawRequest['Phone Number'] = `{{$json.rawRequest['Phone Number']}}`  
     - rawRequest['E-mail'] = `{{$json.rawRequest['E-mail']}}`  
     - rawRequest['First Time Visit?'] = `{{$json.rawRequest['First Time Visit?']}}`  
     - rawRequest['Select an Appointment Date'] = `{{$json.rawRequest['Select an Appointment Date']}}`  
   - Connect the output of the JotForm Trigger node to this node.

3. **Add a Telegram Node called "Notify for Approval or Decline":**  
   - Operation: sendAndWait  
   - Chat ID: Your Telegram chat ID where approval decisions will be made.  
   - Message: Use expressions to include patient's full name, phone, first-time visit status, date/time, and duration. Example:  
     ```
     Name : - {{ $json.rawRequest.Name.first }} {{ $json.rawRequest.Name.last }}
     number :- {{ $json.rawRequest['Phone Number'].full }}
     first time visit :- {{ $json.rawRequest['First Time Visit?'] }}
     date time :- {{ $json.rawRequest['Select an Appointment Date'].date }}
     duration :- {{ $json.rawRequest['Select an Appointment Date'].duration }}
     ```  
   - Approval Options: Double approval type.  
   - Credentials: Configure Telegram Bot credentials.  
   - Connect the output of the Parse node here.

4. **Add Langchain Nodes for AI Email Generation:**  
   a. **OpenAI Chat Model Node:**  
      - Model: gpt-4.1-mini  
      - Connect output to Langchain Agent node.  
      - Configure OpenAI credentials.  

   b. **Structured Output Parser Node:**  
      - Provide example JSON schema for parsing AI response (fields: email, subject, html_body).  
      - Connect output from OpenAI Chat Model to this node.  

   c. **Langchain Agent Node ("Generate: Appointment Response Email"):**  
      - Prompt Type: Define  
      - System Message: Instructions for the AI to generate confirmation or reschedule email in JSON.  
      - Text Input: Include approval status and rawRequest JSON string.  
      - Enable Output Parser (point to Structured Output Parser node).  
      - Connect output from Telegram approval node to this agent node.

5. **Add an If Node "Condition: Check Approval Status":**  
   - Condition: Check if `$('Notify for Approval or Decline').item.json.data.approved` equals `"true"` (string).  
   - Connect output of Langchain Agent node here.

6. **For the Approval Path (True branch):**  
   a. **Google Sheets Node "Log: Record Appointment in sheets":**  
      - Operation: Append or Update row using email as matching column.  
      - Map fields: name, email, phone no, first time visit, date and time, duration from parsed data.  
      - Sheet Name: `Sheet1` or your chosen sheet.  
      - Document ID: Your Google Sheet ID.  
      - Credentials: Google Sheets OAuth2 configured.  

   b. **Gmail Node "Send: Confirmation Email":**  
      - Send To: `{{$json.output.email}}` (from AI output)  
      - Subject: `{{$json.output.subject}}`  
      - Message (HTML): `{{$json.output.html_body}}`  
      - Credentials: Gmail OAuth2 configured.  
      - Connect Google Sheets node output to this node.

7. **For the Rejection Path (False branch):**  
   a. **HTTP Request Node "Delete Rejected Appointment":**  
      - Method: DELETE  
      - URL: `https://api.jotform.com/submission/{{$json.submissionID}}`  
      - Query Parameters: `apiKey=YOUR_JOTFORM_API_KEY`  
      - Credentials: None (API key passed as param).  

   b. **Gmail Node "Send: Rejection or Reschedule Email":**  
      - Same config as confirmation email node but sends rejection email.  
      - Connect output of HTTP Request node to this node.

8. **Create a Sticky Note (optional):**  
   - Add workflow description, instructions, and link to JotForm partner page for reference.

9. **Connect all nodes as per the described flow:**  
   - Appointment Request Form Trigger → Parse: Extract Appointment Details → Notify for Approval or Decline → Generate: Appointment Response Email → Condition: Check Approval Status → (True) Log: Record Appointment in sheets → Send: Confirmation Email  
   - Condition: Check Approval Status → (False) Delete Rejected Appointment → Send: Rejection or Reschedule Email

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates appointment approvals using AI-generated personalized emails and seamless integration of JotForm, Telegram, and Google Sheets. | Workflow Overview Sticky Note                                                                  |
| Uses GPT-4 Mini model via Langchain for email generation with strict JSON output requirement for reliability.                | AI Email Generation Block                                                                        |
| Telegram node uses double approval to ensure robust decision-making.                                                         | Telegram Approval Node                                                                           |
| Google Sheets integration uses append-or-update to avoid duplicate entries based on email matching.                         | Google Sheets Logging                                                                           |
| JotForm submission deletion on rejection keeps the form system clean.                                                        | HTTP Request Node for JotForm API deletion                                                     |
| [Get the JotForm](https://www.jotform.com/?partner=roshanramanidev)                                                         | Promotional partner link included in sticky note                                               |

---

**Disclaimer:**  
The provided workflow is entirely constructed using n8n automation respecting all content policies. It involves only legal and public data sources and does not contain any offensive or protected content.