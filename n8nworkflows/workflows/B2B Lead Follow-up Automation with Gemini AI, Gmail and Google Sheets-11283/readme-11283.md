B2B Lead Follow-up Automation with Gemini AI, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/b2b-lead-follow-up-automation-with-gemini-ai--gmail-and-google-sheets-11283


# B2B Lead Follow-up Automation with Gemini AI, Gmail and Google Sheets

### 1. Workflow Overview

This workflow automates B2B lead follow-ups by integrating Gmail, Google Sheets, and an AI assistant powered by Google Gemini AI. It targets sales and marketing teams who want to track responses to introductory emails, identify leads that have not replied within a set timeframe, and automatically generate personalized reminder emails. The logical flow is divided into the following blocks:

- **1.1 Input Reception and Lead Identification:** Triggered manually or scheduled, it fetches recent introductory emails and updates Google Sheets with lead data.
- **1.2 Lead Filtering and Timing Check:** Retrieves leads needing reminders, checks if the lead has not responded within 5 days.
- **1.3 AI-Powered Reminder Email Generation:** Generates personalized reminder emails using Gemini AI based on lead details and original query.
- **1.4 Reminder Email Dispatch and Sheet Update:** Sends the reminder email as a reply in the original thread and updates the tracking sheet with reminder status.
- **1.5 Auxiliary and Control Nodes:** Includes wait timers, manual triggers, and sticky notes providing instructions and context.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception and Lead Identification

- **Overview:**  
This block initiates the workflow, fetches recent introductory emails from Gmail matching a specific subject template, and updates the Google Sheet lead tracker accordingly.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get many messages (Gmail)  
  - Update row in sheet (Google Sheets)  
  - Get row(s) in sheet (Google Sheets)  
  - Wait (Wait node)  
  - Sticky Note (Instructional)

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - *Type:* Manual trigger to start the flow.  
     - *Config:* Default, no parameters.  
     - *Connections:* Outputs to "Get many messages", "Get row(s) in sheet", and "Wait".  
     - *Edge Cases:* None inherent; manual start only.  

  2. **Get many messages**  
     - *Type:* Gmail node to fetch emails.  
     - *Config:* Retrieves up to 10 recent emails with subject matching `<template of your introductory email>`, filtered in the "CATEGORY_PERSONAL" label.  
     - *Key Expressions:* Uses Gmail search query to filter emails by subject.  
     - *Connections:* Outputs to "Update row in sheet".  
     - *Edge Cases:* Authentication errors (OAuth2), Gmail API rate limits, incorrect query leading to empty results.  

  3. **Update row in sheet**  
     - *Type:* Google Sheets node to update lead tracker rows.  
     - *Config:* Matches on “Email ID” column, updates “Reminder 1 needed?” to “No”, extracts sender email from the email header regex.  
     - *Key Expressions:*  
       - Email extraction via regex `{{$json.headers.from.match(/<([^>]+)>/) ? $json.headers.from.match(/<([^>]+)>/)[1] : ''}}`  
     - *Connections:* Outputs to "Get row(s) in sheet".  
     - *Edge Cases:* Google Sheets API errors, matching failure if email IDs differ in format, no rows to update.  

  4. **Get row(s) in sheet**  
     - *Type:* Google Sheets node to fetch rows where "Reminder 1 needed?" is set.  
     - *Config:* Filters rows from the same sheet with a filter on “Reminder 1 needed?” column.  
     - *Connections:* Outputs to "If" node (Block 1.2).  
     - *Edge Cases:* Empty filter results, API errors.  

  5. **Wait**  
     - *Type:* Wait node to delay execution.  
     - *Config:* Default wait with no parameters (likely placeholder or to control flow).  
     - *Connections:* Outputs to "Get row(s) in sheet".  
     - *Edge Cases:* None significant; can delay flow if used in schedules.

  6. **Sticky Note**  
     - *Content:* Explains logic to exclude clients who replied and update status in tracker; instructs to update search term for the original email subject.  
     - *Role:* Instructional only.

---

#### Block 1.2: Lead Filtering and Timing Check

- **Overview:**  
Filters leads needing reminders by checking if more than 5 days have passed since the introductory email was sent.

- **Nodes Involved:**  
  - If (Conditional)  
  - Sticky Note1 (Instructional)

- **Node Details:**

  1. **If**  
     - *Type:* Conditional node for filtering leads.  
     - *Config:* Compares the "Intro email Date" parsed as date with current date minus 5 days. Passes leads where intro email date is older than 5 days.  
     - *Key Expressions:*  
       - `DateTime.fromFormat($json["Intro email Date"], "dd/MM/yyyy").toMillis()`  
       - `DateTime.now().minus({ days: 5 }).toMillis()`  
     - *Connections:* If true, proceeds to "AI Agent" node (Block 1.3).  
     - *Edge Cases:* Date format mismatch causing parse errors, missing dates, timezone considerations.  

  2. **Sticky Note1**  
     - *Content:* Explains the rationale to send reminders after 5 days of no response; notes personalization based on original message and company info.  
     - *Role:* Instructional.

---

#### Block 1.3: AI-Powered Reminder Email Generation

- **Overview:**  
Generates a personalized reminder email using Google Gemini AI based on lead data, original message, and company info.

- **Nodes Involved:**  
  - AI Agent (Langchain Agent)  
  - Google Gemini Chat Model (Language Model)  
  - Structured Output Parser  
  - Edit Fields (Set)  
  - Sticky Note3 (Instructional)

- **Node Details:**

  1. **AI Agent**  
     - *Type:* Langchain agent node to generate AI text.  
     - *Config:* Uses a prompt template to generate a short, friendly reminder email referencing the original introductory email and client message. Output structured as JSON with `ClientEmail` and `ClientEmailBody`.  
     - *Key Expressions:* References multiple JSON fields like `Email ID`, `Message`, `Intro email Date`, `Company Name`, and client names.  
     - *Connections:* Outputs to "Edit Fields".  
     - *Version Requirements:* Requires Langchain integration and Gemini Chat Model setup.  
     - *Edge Cases:* AI API failures, malformed output JSON, rate limits, prompt errors.  

  2. **Google Gemini Chat Model**  
     - *Type:* Google Gemini AI language model node.  
     - *Config:* Connected as language model backend for AI Agent.  
     - *Credentials:* Uses Google Palm API credentials.  
     - *Connections:* Connected to AI Agent as AI language model provider.  
     - *Edge Cases:* API quota limits, authentication errors.  

  3. **Structured Output Parser**  
     - *Type:* Langchain output parser node.  
     - *Config:* Validates AI output JSON structure with example schema containing `ClientEmail` and `ClientEmailBody`.  
     - *Connections:* Feeds parsed output back into AI Agent for downstream use.  
     - *Edge Cases:* Parsing failure if AI output is malformed or incomplete.  

  4. **Edit Fields**  
     - *Type:* Set node to extract and assign AI output fields for downstream use.  
     - *Config:* Assigns `ClientEmail` and `ClientEmailBody` from AI output JSON to new fields.  
     - *Connections:* Outputs to "Get many messages1" (Block 1.4).  
     - *Edge Cases:* Missing fields if AI output is incomplete.  

  5. **Sticky Note3**  
     - *Content:* Provides main workflow explanation and setup instructions for users: how lead replies are tracked, reminder emails generated, and customization tips.  
     - *Role:* Instructional.

---

#### Block 1.4: Reminder Email Dispatch and Sheet Update

- **Overview:**  
Finds the original introductory email thread in the sent mailbox, replies with the AI-generated reminder email, and updates the Google Sheet status accordingly.

- **Nodes Involved:**  
  - Get many messages1 (Gmail)  
  - Reply to a message (Gmail)  
  - Update row in sheet1 (Google Sheets)  
  - Sticky Note2 (Instructional)

- **Node Details:**

  1. **Get many messages1**  
     - *Type:* Gmail node to search sent messages.  
     - *Config:* Searches sent emails to the client (`to: {{ $json.ClientEmail }}`) with subject containing "Following up your interest" to find the original introductory email thread.  
     - *Connections:* Outputs to "Reply to a message".  
     - *Edge Cases:* No matching sent email found, Gmail API issues, incorrect or incomplete client email.  

  2. **Reply to a message**  
     - *Type:* Gmail node to reply within an email thread.  
     - *Config:* Replies to the thread ID and message ID from the found original email with the AI-generated email body. Sender name customizable.  
     - *Key Expressions:* Uses `$('Edit Fields').item.json.ClientEmailBody` for the reply content.  
     - *Connections:* Outputs to "Update row in sheet1".  
     - *Edge Cases:* Gmail API errors, invalid thread or message ID, sending limits.  

  3. **Update row in sheet1**  
     - *Type:* Google Sheets node to update lead status after sending reminder.  
     - *Config:* Updates columns like "Status" to "Reminder 1 Drafted", sets "Reminder 1 needed?" to “Yes”, and logs the reminder email date. Matches on “Email ID”.  
     - *Connections:* Terminal node.  
     - *Edge Cases:* Sheet update failures, mismatched email IDs, date formatting issues.  

  4. **Sticky Note2**  
     - *Content:* Explains sending reminders on the original email thread and updating the sheet with reminder dates.  
     - *Role:* Instructional.

---

#### Block 1.5: Auxiliary and Control Nodes

- **Overview:**  
Additional nodes used for control flow, manual execution, and providing user guidance.

- **Nodes Involved:**  
  - Wait (within Block 1.1)  
  - Sticky Note (multiple notes)  
  - Manual Trigger (When clicking ‘Execute workflow’)  

- **Notes:** Provide workflow execution pacing, instructions, and control points.

---

### 3. Summary Table

| Node Name                | Node Type                     | Functional Role                              | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                  |
|--------------------------|-------------------------------|----------------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Starts the workflow manually                  | None                         | Get many messages, Get row(s) in sheet, Wait | Main Sticky: Explains workflow logic and setup steps.                                                        |
| Get many messages         | Gmail                         | Fetches recent introductory emails            | When clicking ‘Execute workflow’ | Update row in sheet             | Check if a client replied; update status; customize email subject search.                                    |
| Update row in sheet       | Google Sheets                 | Updates lead tracker with email data           | Get many messages             | Get row(s) in sheet            | See above                                                                                                    |
| Get row(s) in sheet       | Google Sheets                 | Retrieves leads flagged for reminder           | Update row in sheet, Wait     | If                            | Identify who to send reminder to; 5 days no response check.                                                 |
| Wait                     | Wait                          | Delays execution                               | When clicking ‘Execute workflow’ | Get row(s) in sheet            |                                                                                                              |
| If                       | If                            | Filters leads with intro email older than 5 days | Get row(s) in sheet           | AI Agent                      | See above                                                                                                    |
| AI Agent                 | Langchain Agent               | Generates personalized reminder email          | If                           | Edit Fields                   | Main Sticky: AI writes reminder email based on lead details.                                                |
| Google Gemini Chat Model  | Gemini AI Language Model      | Provides LLM backend for AI Agent               | None                         | AI Agent                      |                                                                                                              |
| Structured Output Parser  | Langchain Output Parser       | Validates AI JSON output structure              | Google Gemini Chat Model      | AI Agent                      |                                                                                                              |
| Edit Fields              | Set                           | Extracts AI output fields for further use       | AI Agent                     | Get many messages1             |                                                                                                              |
| Get many messages1        | Gmail                         | Finds original intro email in sent mailbox      | Edit Fields                  | Reply to a message            | Send reminder on original email thread.                                                                     |
| Reply to a message        | Gmail                         | Sends reminder email as reply to thread          | Get many messages1            | Update row in sheet1           | See above                                                                                                    |
| Update row in sheet1      | Google Sheets                 | Updates sheet to mark reminder sent              | Reply to a message            | None                         |                                                                                                              |
| Sticky Note               | Sticky Note                   | Instructions and notes                           | None                         | None                         | Various notes providing setup guidance and workflow explanation.                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Execute workflow’` to manually start the workflow.

2. **Add a Gmail node** named `Get many messages`:  
   - Operation: `getAll` messages  
   - Limit: 10  
   - Filters: Query `subject: <template of your introductory email>`  
   - Label IDs: `CATEGORY_PERSONAL`  
   - Credentials: Configure with OAuth2 Gmail account.

3. **Add a Google Sheets node** named `Update row in sheet`:  
   - Operation: `update`  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: The sheet tab (e.g., "Form Filled")  
   - Matching Column: `Email ID`  
   - Columns to update:  
     - `Email ID`: Extract from email header using regex expression:  
       `={{ $json.headers.from.match(/<([^>]+)>/) ? $json.headers.from.match(/<([^>]+)>/)[1] : '' }}`  
     - `Reminder 1 needed?`: `"No"`  
   - Credentials: OAuth2 Google Sheets account.  

4. **Add a Google Sheets node** named `Get row(s) in sheet`:  
   - Operation: `get` or `lookup` rows where `Reminder 1 needed?` is set.  
   - Document and Sheet same as above.  
   - Filter on column `Reminder 1 needed?`.  
   - Credentials set accordingly.

5. **Add a Wait node** named `Wait` to control execution pacing (optional, default delay).

6. **Connect nodes:**  
   - Manual Trigger → Get many messages, Get row(s) in sheet, Wait  
   - Get many messages → Update row in sheet  
   - Update row in sheet → Get row(s) in sheet  
   - Wait → Get row(s) in sheet

7. **Add an If node** named `If`:  
   - Condition: Check if `Intro email Date` is older than 5 days.  
   - Use expression:  
     - Left: `DateTime.fromFormat($json["Intro email Date"], "dd/MM/yyyy").toMillis()`  
     - Operator: less than  
     - Right: `DateTime.now().minus({ days: 5 }).toMillis()`  
   - True: leads to AI Agent.

8. **Add Langchain Google Gemini Chat Model node**:  
   - Link to Google Palm API credentials.  
   - Used as language model backend.

9. **Add Langchain AI Agent node**:  
   - Prompt: Custom prompt to generate reminder email with placeholders for `ClientEmail`, `ClientMessage`, `Intro email Date`, etc.  
   - Output format JSON with fields `ClientEmail` and `ClientEmailBody`.  
   - Connect to Gemini Chat Model as AI language provider.  
   - Connect output to Structured Output Parser.

10. **Add Langchain Structured Output Parser node**:  
    - Provide JSON schema example with `ClientEmail` and `ClientEmailBody`.  
    - Connect output back to AI Agent if needed, and forward to Edit Fields.

11. **Add a Set node** named `Edit Fields`:  
    - Assign fields: `ClientEmail = {{$json.output.ClientEmail}}`, `ClientEmailBody = {{$json.output.ClientEmailBody}}`.

12. **Add Gmail node `Get many messages1`**:  
    - Operation: getAll sent messages with filter:  
      `to: {{ $json.ClientEmail }} subject: Following up your interest`  
    - Credentials: Gmail OAuth2 account.

13. **Add Gmail node `Reply to a message`**:  
    - Operation: reply to thread  
    - Thread ID and message ID from `Get many messages1` output  
    - Message body from `Edit Fields` `ClientEmailBody`  
    - Sender name customizable.  

14. **Add Google Sheets node `Update row in sheet1`**:  
    - Operation: update  
    - Matching on `Email ID`  
    - Update columns:  
      - `Status`: "Reminder 1 Drafted"  
      - `Reminder 1 needed?`: "Yes"  
      - `Reminder 1 Email Date`: Current date formatted `dd-MMM-yyyy`  
    - Credentials: Google Sheets OAuth2.

15. **Connect nodes:**  
    - If (true) → AI Agent → Edit Fields → Get many messages1 → Reply to a message → Update row in sheet1

16. **Add Sticky Notes as needed** to document process steps and instructions inside the workflow canvas.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                                        |
|------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Main workflow captures replies to introductory emails and sends reminders after 5 days if no reply is received.              | Workflow overview and setup instructions inside Sticky Note3 node.                                                    |
| Customize the Gmail search query in `Get many messages` node to match your actual introductory email subject line.          | Sticky Note near "Get many messages" node.                                                                             |
| Google Sheets document tracks lead data and email statuses; ensure correct OAuth2 credentials and sheet permissions.          | Throughout Google Sheets nodes configuration.                                                                          |
| AI prompt can be customized to match your company style; uses Google Gemini AI via Langchain integration.                     | AI Agent node prompt content.                                                                                           |
| Gmail OAuth2 credentials must have permission to read inbox and send emails on your behalf.                                   | Gmail nodes credential requirement.                                                                                    |
| Be mindful of Google and Gmail API rate limits and OAuth2 token expirations during extended workflow runs.                   | General operational consideration.                                                                                      |
| For date parsing, ensure the `Intro email Date` column in Google Sheets follows `dd/MM/yyyy` format precisely.               | Important for correct filtering in If node.                                                                             |
| Reminder emails are sent as replies in the original introductory email thread to maintain conversation context.               | Sticky Note2 content and Gmail "Reply to a message" node logic.                                                        |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or offensive material. All data handled is legal and public.