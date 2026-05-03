Personalized Cold Email System with Google Gemini, Telegram Approval & Google Sheets

https://n8nworkflows.xyz/workflows/personalized-cold-email-system-with-google-gemini--telegram-approval---google-sheets-9267


# Personalized Cold Email System with Google Gemini, Telegram Approval & Google Sheets

### 1. Workflow Overview

This workflow automates the generation, approval, sending, and tracking of personalized cold emails using AI (Google Gemini) with Telegram-based user approval and Google Sheets for data management. It is designed for sales or marketing teams needing to efficiently send highly tailored cold emails with human-in-the-loop validation to ensure quality and relevance.

The workflow includes these logical blocks:

- **1.1 Input Reception and Lead Fetching:** Manual trigger initiates the workflow, fetching up to 3 leads marked as "Available" from Google Sheets.
- **1.2 AI Email Generation:** For each lead, Google Gemini AI generates a personalized email subject and body.
- **1.3 Telegram Approval:** The generated email is sent to a Telegram chat for approval (approve/reject).
- **1.4 Email Sending or Rejection Handling:** If approved, the email is sent via SMTP; if rejected, feedback is collected via Telegram to regenerate the email.
- **1.5 Google Sheets Updates and Tracking:** Updates the relevant Google Sheets tabs to reflect email status and logs sent emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Lead Fetching

- **Overview:** Starts the workflow manually and retrieves up to 3 leads with status "Available" from a Google Sheet to process.
- **Nodes Involved:**  
  - When clicking 'Execute workflow' (Manual Trigger)  
  - Date & Time  
  - Get Lead from Google Sheet  
  - Limit Sheets Get Function  
  - Loop Over Items1  
  - Send a text message9  

- **Node Details:**

  - **When clicking 'Execute workflow'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually on user command.  
    - Inputs: None  
    - Outputs: Triggers Date & Time node  
    - Edge Cases: None, manual start.

  - **Date & Time**  
    - Type: Date/Time  
    - Role: Adds current timestamp contextually; can be used for logging or filtering.  
    - Inputs: Manual trigger output  
    - Outputs: Leads to "Get Lead from Google Sheet"  
    - Edge Cases: None significant.

  - **Get Lead from Google Sheet**  
    - Type: Google Sheets (Read)  
    - Role: Fetch leads from "Filtered Leads" tab where Status = "Available".  
    - Configuration: Sheet ID and tab "Filtered Leads" (gid=0); filter on "Status" column equals "Available".  
    - Inputs: Date & Time node output  
    - Outputs: Data passed to "Limit Sheets Get Function"  
    - Edge Cases: API quota limits, connectivity issues, empty result sets.

  - **Limit Sheets Get Function**  
    - Type: Limit  
    - Role: Limits the number of leads processed per run to 3.  
    - Inputs: Leads array from Google Sheets  
    - Outputs: Passes limited leads to batch processing  
    - Edge Cases: If fewer than 3 leads available, processes fewer.

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Loops through each lead individually for processing.  
    - Inputs: Limited lead batch  
    - Outputs: Parallel paths for AI processing and notification  
    - Edge Cases: Batch size must be consistent; handles 1 lead per iteration.

  - **Send a text message9**  
    - Type: Telegram  
    - Role: Sends "Batch Complete!✅" message once per batch execution to notify completion.  
    - Configuration: Fixed chat ID, no attribution appended.  
    - Inputs: First output from batch processing (executes once)  
    - Edge Cases: Telegram API rate limits, invalid chat ID.

---

#### 2.2 AI Email Generation

- **Overview:** Generates a personalized cold email (subject and body) for each lead using Google Gemini AI.
- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - Merge Data and Agent output for  

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Sends prompt to AI to generate personalized email based on lead data.  
    - Configuration: Prompt text "Generate a personalized cold email based on the lead data provided."  
    - Inputs: Each lead from Loop Over Items1  
    - Outputs: Raw AI response to be parsed  
    - Edge Cases: AI service downtime, malformed input data, prompt failure.

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini LM Chat  
    - Role: Language model that processes AI Agent's request.  
    - Inputs: AI Agent's prompt  
    - Outputs: Text completion for email generation  
    - Edge Cases: API key or quota issues.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI output into structured JSON with fields "Subject" and "Body".  
    - Configuration: JSON schema example provided for expected output.  
    - Inputs: Response from Google Gemini Chat Model  
    - Outputs: Parsed structured data for downstream usage  
    - Edge Cases: Parser errors if AI output is malformed, partial, or unexpected.

  - **Merge Data and Agent output for**  
    - Type: Merge  
    - Role: Combines original lead data with AI-generated email content for further processing.  
    - Inputs: Original lead data and structured AI output  
    - Outputs: Data forwarded to Telegram approval node  
    - Edge Cases: Mismatched data arrays, missing fields.

---

#### 2.3 Telegram Approval

- **Overview:** Sends the AI-generated email to Telegram for human approval. Waits for approval (✅) or rejection (❌).
- **Nodes Involved:**  
  - Send message and wait for response  
  - Switch  

- **Node Details:**

  - **Send message and wait for response**  
    - Type: Telegram (sendAndWait)  
    - Role: Sends the email subject and body to Telegram chat; waits for user approval response.  
    - Configuration: Message template with Subject and Body; approval type "double" (confirm/deny).  
    - Inputs: Merged data from AI output and lead info  
    - Outputs: Response with approval status boolean  
    - Edge Cases: Telegram message send failure, response timeout, invalid chat ID.

  - **Switch**  
    - Type: Switch  
    - Role: Routes workflow based on approval response boolean `data.approved`.  
    - Configuration: Two outputs — "Approved" (true) and "Rejected" (false).  
    - Inputs: Telegram approval response  
    - Outputs: Approved path triggers email sending; rejected path triggers feedback collection.  
    - Edge Cases: Unexpected response values, missing approval field.

---

#### 2.4 Email Sending or Rejection Handling

- **Overview:** Sends the approved email via SMTP or collects feedback for rejected emails to regenerate content.
- **Nodes Involved:**  
  - Send email  
  - Wait3  
  - Send message and wait for response1  
  - AI Agent3  
  - Structured Output Parser1  
  - Google Gemini Chat Model4  
  - Google Gemini Chat Model5  

- **Node Details:**

  - **Send email**  
    - Type: Email Send (SMTP)  
    - Role: Sends the approved email with subject and body to the lead’s email address.  
    - Configuration: From "connect@company.com", BCC to "connect@company.com", plain text format.  
    - Inputs: Approved email content and lead email  
    - Outputs: Triggers wait node for pacing  
    - Edge Cases: SMTP authentication errors, invalid email addresses, sending delays.

  - **Wait3**  
    - Type: Wait  
    - Role: Waits 1 second before looping back to process the next lead batch.  
    - Inputs: Post-email sending  
    - Outputs: Feeds back into batch loop node  
    - Edge Cases: None significant.

  - **Send message and wait for response1**  
    - Type: Telegram (sendAndWait)  
    - Role: When email is rejected, requests feedback via free-text from Telegram user.  
    - Configuration: Message "How to change email?"  
    - Inputs: Switch node’s rejected output  
    - Outputs: User feedback text input  
    - Edge Cases: Timeout, no response, invalid input.

  - **AI Agent3**  
    - Type: LangChain AI Agent  
    - Role: Regenerates email based on user feedback (reject feedback).  
    - Configuration: Prompt to regenerate email with feedback, expects structured output.  
    - Inputs: User feedback text  
    - Outputs: New email draft to be parsed and sent again for approval  
    - Edge Cases: AI generation errors, feedback ambiguity.

  - **Structured Output Parser1**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses regenerated AI output into structured Subject and Body fields.  
    - Inputs: AI Agent3 output  
    - Outputs: Structured email content for approval  
    - Edge Cases: Parsing errors if AI output malformed.

  - **Google Gemini Chat Model4 / Google Gemini Chat Model5**  
    - Type: LangChain Google Gemini LM Chat  
    - Role: Language model used for regenerating email and parsing structured output respectively.  
    - Inputs/Outputs: Connected in AI Agent3 regeneration flow  
    - Edge Cases: Same as earlier for AI API calls.

---

#### 2.5 Google Sheets Updates and Tracking

- **Overview:** Logs email sending activity and updates lead status in Google Sheets to track progress.
- **Nodes Involved:**  
  - AI Agent1  
  - Google Gemini Chat Model1  
  - Simple Memory1  
  - Sent Leads Append  
  - Filtered Leads Update  
  - AI Agent2  
  - Google Gemini Chat Model2  
  - Add Email Draft to Google Sheet  
  - Merge Data and Agent output for  

- **Node Details:**

  - **AI Agent1**  
    - Type: LangChain AI Agent  
    - Role: Generates update instructions to append sent leads and update lead status to "Sent".  
    - Inputs: Data merged after AI email generation  
    - Outputs: Commands to update sheets  
    - Edge Cases: AI inconsistencies.

  - **Google Gemini Chat Model1**  
    - Type: LangChain Google Gemini LM Chat  
    - Role: Language model assisting AI Agent1.  
    - Edge Cases: API issues.

  - **Simple Memory1**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains session context keyed by Lead ID for AI Agent1 to track conversation state.  
    - Inputs: Lead ID from Google Sheet node  
    - Outputs: Context for AI Agent1  
    - Edge Cases: Memory overflow or reset.

  - **Sent Leads Append**  
    - Type: Google Sheets (Append)  
    - Role: Appends a record of the sent email including Name, Email, Lead ID, and Sent Date to "Sent Leads" tab.  
    - Inputs: AI Agent1 output  
    - Edge Cases: Google Sheets API limits, data format errors.

  - **Filtered Leads Update**  
    - Type: Google Sheets (Update)  
    - Role: Updates the lead's status to "Sent" in "Filtered Leads" tab to prevent re-processing.  
    - Inputs: AI Agent1 output  
    - Edge Cases: Row matching errors.

  - **AI Agent2**  
    - Type: LangChain AI Agent  
    - Role: Archives email details to "Emails Sent" sheet for record keeping.  
    - Inputs: Merged data from AI output and lead info  
    - Edge Cases: AI or Sheets API errors.

  - **Google Gemini Chat Model2**  
    - Type: LangChain Google Gemini LM Chat  
    - Role: Supports AI Agent2 for archiving prompt generation.  
    - Edge Cases: API errors.

  - **Add Email Draft to Google Sheet**  
    - Type: Google Sheets (Append)  
    - Role: Adds the email draft (ID, Subject, Body) to a dedicated "Emails Sent" sheet tab.  
    - Inputs: AI Agent2 output  
    - Edge Cases: Google Sheets constraints.

  - **Merge Data and Agent output for**  
    - Type: Merge  
    - Role: Combines data for archival before passing to AI Agent2.  
    - Edge Cases: Data consistency.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                         | Input Node(s)                   | Output Node(s)                        | Sticky Note                                                  |
|--------------------------------|----------------------------------|---------------------------------------|--------------------------------|-------------------------------------|--------------------------------------------------------------|
| Setup Guide                    | Sticky Note                      | Workflow overview and setup guide     |                                |                                     | ## 📧 COLD EMAIL SYSTEM - SETUP ... Limit node = 3 leads per run |
| Sticky Note1                  | Sticky Note                      | Workflow start marker                  |                                |                                     | 🚀 START Manual trigger                                      |
| Sticky Note2                  | Sticky Note                      | Lead fetching configuration            |                                |                                     | 📊 FETCH Status=Available Limit: 3                            |
| Sticky Note3                  | Sticky Note                      | Loop processing marker                 |                                |                                     | 🔁 LOOP One lead at a time                                    |
| Sticky Note4                  | Sticky Note                      | AI email generation marker             |                                |                                     | 🤖 AI GENERATOR Creates Subject + Body                       |
| Sticky Note5                  | Sticky Note                      | Telegram approval marker               |                                |                                     | 📱 APPROVAL Telegram: ✅/❌                                   |
| Sticky Note6                  | Sticky Note                      | Email sending marker                   |                                |                                     | ✉️ SEND SMTP email                                           |
| Sticky Note7                  | Sticky Note                      | Rejection and regeneration marker     |                                |                                     | 🔄 REJECT Feedback → Regenerate                              |
| Sticky Note8                  | Sticky Note                      | Sheets tracking marker                 |                                |                                     | 📝 TRACKING 3 Sheets updated                                  |
| When clicking 'Execute workflow' | Manual Trigger                  | Workflow entry point                   |                                | Date & Time                        | 🚀 START Manual trigger                                      |
| Date & Time                   | Date/Time                       | Adds timestamp                        | When clicking 'Execute workflow' | Get Lead from Google Sheet           |                                                              |
| Get Lead from Google Sheet     | Google Sheets                   | Fetch leads with Status=Available     | Date & Time                   | Limit Sheets Get Function             | 📊 FETCH Status=Available Limit: 3                            |
| Limit Sheets Get Function      | Limit                          | Limits leads to 3 per run              | Get Lead from Google Sheet     | Loop Over Items1                     | 📊 FETCH Status=Available Limit: 3                            |
| Loop Over Items1               | SplitInBatches                 | Processes each lead individually       | Limit Sheets Get Function      | AI Agent, Send a text message9       | 🔁 LOOP One lead at a time                                    |
| Send a text message9           | Telegram                       | Sends batch completion notification   | Loop Over Items1               |                                     | 🔁 LOOP One lead at a time                                    |
| AI Agent                      | LangChain AI Agent             | Generates personalized cold email     | Loop Over Items1               | Merge Data and Agent output for, Send message and wait for response | 🤖 AI GENERATOR Creates Subject + Body                       |
| Google Gemini Chat Model       | LangChain LM Chat Google Gemini | AI model for email generation          | AI Agent                     | Structured Output Parser             | 🤖 AI GENERATOR Creates Subject + Body                       |
| Structured Output Parser       | LangChain Output Parser        | Parses AI output into Subject and Body | Google Gemini Chat Model      | Merge Data and Agent output for      | 🤖 AI GENERATOR Creates Subject + Body                       |
| Merge Data and Agent output for| Merge                         | Combines lead data and AI output      | AI Agent, Structured Output Parser | AI Agent2                          | 🤖 AI GENERATOR Creates Subject + Body                       |
| Send message and wait for response | Telegram                   | Sends email draft for Telegram approval | Merge Data and Agent output for| Switch                            | 📱 APPROVAL Telegram: ✅/❌                                   |
| Switch                       | Switch                        | Routes workflow based on approval      | Send message and wait for response | Send email, Send message and wait for response1 | 📱 APPROVAL Telegram: ✅/❌                                   |
| Send email                    | Email Send                    | Sends approved email via SMTP          | Switch (Approved)             | Wait3                              | ✉️ SEND SMTP email                                           |
| Wait3                        | Wait                         | Waits 1 second between sends           | Send email                   | Loop Over Items1                    | ✉️ SEND SMTP email                                           |
| Send message and wait for response1 | Telegram                 | Collects feedback on rejected emails   | Switch (Rejected)             | AI Agent3                         | 🔄 REJECT Feedback → Regenerate                              |
| AI Agent3                    | LangChain AI Agent            | Regenerates email based on feedback    | Send message and wait for response1 | Send message and wait for response | 🔄 REJECT Feedback → Regenerate                              |
| Structured Output Parser1     | LangChain Output Parser       | Parses regenerated email output        | AI Agent3                    | Send message and wait for response  | 🔄 REJECT Feedback → Regenerate                              |
| Google Gemini Chat Model4      | LangChain LM Chat Google Gemini | AI model for regeneration               | AI Agent3                    | Structured Output Parser1           | 🔄 REJECT Feedback → Regenerate                              |
| Google Gemini Chat Model5      | LangChain LM Chat Google Gemini | AI model for parsing regeneration output | Structured Output Parser1      | AI Agent3                         | 🔄 REJECT Feedback → Regenerate                              |
| AI Agent1                    | LangChain AI Agent            | Generates instructions to update sheets | Loop Over Items1, Simple Memory1 | Sent Leads Append, Filtered Leads Update | 📝 TRACKING 3 Sheets updated                                  |
| Google Gemini Chat Model1      | LangChain LM Chat Google Gemini | Supports AI Agent1                       | AI Agent1                   | Sent Leads Append, Filtered Leads Update | 📝 TRACKING 3 Sheets updated                                  |
| Simple Memory1                | LangChain Memory Buffer Window | Maintains AI session context            | Get Lead from Google Sheet    | AI Agent1                         | 📝 TRACKING 3 Sheets updated                                  |
| Sent Leads Append             | Google Sheets (Append)        | Appends sent email info to "Sent Leads" | AI Agent1                   |                                 | 📝 TRACKING 3 Sheets updated                                  |
| Filtered Leads Update          | Google Sheets (Update)        | Updates lead status to "Sent"           | AI Agent1                   |                                 | 📝 TRACKING 3 Sheets updated                                  |
| AI Agent2                    | LangChain AI Agent            | Archives email details for tracking    | Merge Data and Agent output for | Add Email Draft to Google Sheet    | 📝 TRACKING 3 Sheets updated                                  |
| Google Gemini Chat Model2      | LangChain LM Chat Google Gemini | Supports AI Agent2                       | AI Agent2                   | Add Email Draft to Google Sheet     | 📝 TRACKING 3 Sheets updated                                  |
| Add Email Draft to Google Sheet | Google Sheets (Append)        | Appends email draft to "Emails Sent" sheet | AI Agent2                   |                                 | 📝 TRACKING 3 Sheets updated                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking 'Execute workflow'`  
   - Purpose: Manual start of the workflow.

2. **Add Date & Time Node:**  
   - Connect from Manual Trigger  
   - Purpose: Add timestamp context.

3. **Add Google Sheets Read Node:**  
   - Name: `Get Lead from Google Sheet`  
   - Connect from Date & Time  
   - Configure:  
     - Document ID: Your spreadsheet containing leads.  
     - Sheet Name: "Filtered Leads" (gid=0).  
     - Filter: Status column equals "Available".

4. **Add Limit Node:**  
   - Name: `Limit Sheets Get Function`  
   - Connect from Google Sheets  
   - Set max items to 3.

5. **Add SplitInBatches Node:**  
   - Name: `Loop Over Items1`  
   - Connect from Limit Node  
   - Purpose: Loop over each lead one by one.

6. **Add Telegram Node for Batch Completion Notification:**  
   - Name: `Send a text message9`  
   - Connect as secondary output from SplitInBatches  
   - Configure: Chat ID, message "Batch Complete!✅".

7. **Add LangChain AI Agent Node:**  
   - Name: `AI Agent`  
   - Connect from SplitInBatches (main output)  
   - Prompt: "Generate a personalized cold email based on the lead data provided."  
   - Set to expect output.

8. **Add Google Gemini Chat Model Node:**  
   - Name: `Google Gemini Chat Model`  
   - Connect from AI Agent  
   - Use Google Gemini credentials.

9. **Add Structured Output Parser Node:**  
   - Name: `Structured Output Parser`  
   - Connect from Google Gemini Chat Model  
   - Configure schema expecting JSON with "Subject" and "Body".

10. **Add Merge Node:**  
    - Name: `Merge Data and Agent output for`  
    - Connect AI Agent data and Structured Output Parser outputs  
    - Purpose: Combine lead data with AI output.

11. **Add Telegram Node for Approval:**  
    - Name: `Send message and wait for response`  
    - Connect from Merge node  
    - Configure message template with email Subject and Body.  
    - Set approval type to double (approve/reject).  
    - Use your Telegram Bot credentials and chat ID.

12. **Add Switch Node:**  
    - Name: `Switch`  
    - Connect from Telegram approval node  
    - Configure two outputs:  
      - Approved: When `data.approved` is true  
      - Rejected: When `data.approved` is false.

13. **Add Email Send Node:**  
    - Name: `Send email`  
    - Connect from Switch "Approved" output  
    - Configure SMTP credentials.  
    - Set email recipient from lead’s email, subject and body from AI output.  
    - From and BCC email: "connect@company.com".

14. **Add Wait Node:**  
    - Name: `Wait3`  
    - Connect from Email Send  
    - Set wait time: 1 second.

15. **Connect Wait Node back to SplitInBatches:**  
    - To continue processing next lead.

16. **Add Telegram Node for Rejection Feedback:**  
    - Name: `Send message and wait for response1`  
    - Connect from Switch "Rejected" output  
    - Message: "How to change email?"  
    - Operation: sendAndWait for free text response.

17. **Add AI Agent for Regeneration:**  
    - Name: `AI Agent3`  
    - Connect from Telegram feedback  
    - Prompt: "Regenerate email based on user feedback."  
    - Expect structured output.

18. **Add Google Gemini Chat Model Node:**  
    - Name: `Google Gemini Chat Model4`  
    - Connect from AI Agent3.

19. **Add Structured Output Parser Node:**  
    - Name: `Structured Output Parser1`  
    - Connect from Google Gemini Chat Model4.  
    - Same schema as before.

20. **Add Google Gemini Chat Model Node:**  
    - Name: `Google Gemini Chat Model5`  
    - Connect from Structured Output Parser1.

21. **Connect Structured Output Parser1 output to Telegram Approval Node (Step 11):**  
    - To resend regenerated email for approval.

22. **Add AI Agent1 for Sheet Updates:**  
    - Connect from Merge Node output and set up LangChain memory node keyed by Lead ID for context.  
    - Prompt: "Update sheets: append to Sent Leads and update status to Sent in Filtered Leads."

23. **Add Google Gemini Chat Model1:**  
    - Connect from AI Agent1.

24. **Add Simple Memory Node:**  
    - Name: `Simple Memory1`  
    - Connect from Google Sheets lead node for session key (Lead ID).

25. **Add Google Sheets Append Node:**  
    - Name: `Sent Leads Append`  
    - Connect from AI Agent1 outputs  
    - Append Name, Email, Lead ID, Sent Date to "Sent Leads" tab.

26. **Add Google Sheets Update Node:**  
    - Name: `Filtered Leads Update`  
    - Connect from AI Agent1 outputs  
    - Update lead status to "Sent" in "Filtered Leads" tab.

27. **Add AI Agent2 for Email Archiving:**  
    - Connect from Merge Node output  
    - Prompt: "Archive email to Emails Sent sheet."

28. **Add Google Gemini Chat Model2:**  
    - Connect from AI Agent2.

29. **Add Google Sheets Append Node:**  
    - Name: `Add Email Draft to Google Sheet`  
    - Connect from AI Agent2 outputs  
    - Append Lead ID, Email Subject, Email Body to "Emails Sent" tab.

30. **Credentials Configuration:**  
    - Configure Google Sheets credentials with access to your spreadsheet.  
    - Configure Telegram Bot credentials with Bot Token and chat ID with approval rights.  
    - Configure SMTP credentials for sending emails.  
    - Configure Google Gemini API credentials for LangChain nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow limits processing to 3 leads per run to manage API usage and avoid spam.                        | Setup Guide sticky note                                                                                           |
| Telegram approval uses "double" approval type allowing explicit approve (✅) or reject (❌) responses.    | Telegram node configuration                                                                                       |
| Three Google Sheets tabs are used: "Filtered Leads", "Sent Leads", and "Emails Sent" for tracking.       | Setup Guide sticky note                                                                                           |
| Email sending uses SMTP with BCC to "connect@company.com" for internal team tracking.                    | Send email node parameters                                                                                        |
| The workflow uses LangChain AI Agents with Google Gemini Chat model for natural language processing.     | LangChain nodes                                                                                                   |
| Telegram chat ID “5471021866” is hardcoded and must be replaced with your own bot’s chat ID.             | Telegram nodes                                                                                                    |
| The workflow is designed with manual start for controlled batch runs.                                   | Manual Trigger node                                                                                               |
| This cold email system workflow was inspired by user needs for personalized, approved outreach.          | Workflow description                                                                                              |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.