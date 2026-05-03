Automated Recruitment Process with Slack, DocuSign, Trello & Gmail Notifications

https://n8nworkflows.xyz/workflows/automated-recruitment-process-with-slack--docusign--trello---gmail-notifications-7169


# Automated Recruitment Process with Slack, DocuSign, Trello & Gmail Notifications

### 1. Workflow Overview

This workflow automates a comprehensive recruitment process integrating Google Calendar, Slack, DocuSign, Trello, Airtable, Gmail, and Google Sheets. Its main purpose is to manage candidate interviews, collect feedback, make decisions, update candidate statuses, and notify stakeholders efficiently.

Logical blocks:

- **1.1 Interview Event Detection**: Detects completion of candidate interviews via Google Calendar.
- **1.2 Feedback Collection & Processing**: Sends Slack feedback forms, waits for submissions, logs data, and calculates scores.
- **1.3 Candidate Status Update & Communication**: Updates Airtable statuses based on interview results, sends emails (offer, rejection, follow-up invitations), and manages DocuSign signing.
- **1.4 Task Management & Notifications**: Creates Trello cards for new candidates, notifies HR via Slack, and sends welcome emails.
- **1.5 Weekly Reporting**: Scheduled weekly summary involving candidate data aggregation, metric calculation, logging reports in Google Sheets, and sending Slack summaries.

---

### 2. Block-by-Block Analysis

#### 1.1 Interview Event Detection

- **Overview**: Watches Google Calendar for completed interview events to trigger the recruitment workflow.
- **Nodes Involved**:  
  - Interview Completed

- **Node Details**:

  - **Interview Completed**  
    - Type: Google Calendar Trigger  
    - Role: Triggers workflow on calendar event changes, specifically interview completions.  
    - Configuration: Default calendar trigger configured to detect interview completion events (likely filtering by event title or calendar).  
    - Inputs: External calendar events  
    - Outputs: Triggers next node (`Send Feedback Form`)  
    - Edge Cases: Missed events if calendar sync delays; unauthorized calendar access; event data format inconsistencies.

#### 1.2 Feedback Collection & Processing

- **Overview**: Sends feedback forms via Slack, waits 2 hours for submission, checks submission status, sends reminders if needed, logs feedback in Airtable, and calculates candidate scores and decisions.

- **Nodes Involved**:  
  - Send Feedback Form  
  - 2 Hours (Wait)  
  - Check for Form Submission  
  - Form Submitted? (If)  
  - Send Reminder  
  - Log Feedback  
  - Calculate Score & Decision  
  - Passed Interview? (If)

- **Node Details**:

  - **Send Feedback Form**  
    - Type: Slack  
    - Role: Sends feedback form via Slack webhook to interviewers immediately after interview completion.  
    - Config: Uses Slack webhook ID `d1ead6e6-ae65-4aa2-91be-a95dc621c07f`. Message structure likely contains form link or instructions.  
    - Inputs: Triggered by `Interview Completed`  
    - Outputs: Triggers `2 Hours` wait node  
    - Edge Cases: Slack webhook failure; message delivery delays.

  - **2 Hours**  
    - Type: Wait  
    - Role: Pauses workflow for 2 hours allowing feedback submission time.  
    - Config: 2-hour timer before proceeding.  
    - Inputs: From `Send Feedback Form`  
    - Outputs: Triggers `Check for Form Submission`  
    - Edge Cases: Workflow timeout if wait exceeds execution limits.

  - **Check for Form Submission**  
    - Type: HTTP Request  
    - Role: Checks if feedback form has been submitted by querying an endpoint or API.  
    - Config: HTTP GET or POST to feedback system endpoint; authentication likely configured.  
    - Inputs: From `2 Hours` wait  
    - Outputs: To `Form Submitted?` node  
    - Edge Cases: HTTP errors, authentication failures, JSON parsing errors.

  - **Form Submitted?**  
    - Type: If  
    - Role: Branches workflow based on whether feedback was submitted.  
    - Config: Condition evaluates HTTP response field indicating form submission.  
    - Inputs: From `Check for Form Submission`  
    - Outputs: If false → `Send Reminder`; if true → `Send Reminder` (flow seems to send reminder regardless, but likely different logic internally).  
    - Edge Cases: Incorrect condition evaluation, missing data.

  - **Send Reminder**  
    - Type: Slack  
    - Role: Sends reminder notification to interviewers to complete feedback form.  
    - Config: Slack webhook ID `6486280f-617c-47e6-bd2e-99c0c59170ea`.  
    - Inputs: From `Form Submitted?` false path or possibly always.  
    - Outputs: Triggers `Log Feedback`  
    - Edge Cases: Slack API limits, webhook errors.

  - **Log Feedback**  
    - Type: Airtable  
    - Role: Logs candidate feedback including name, overall score, and comments into Airtable database.  
    - Config: Airtable base and table configured; fields mapped to candidate name and feedback form data.  
    - Inputs: From `Send Reminder`  
    - Outputs: Triggers `Calculate Score & Decision`  
    - Edge Cases: Airtable API rate limits, authentication failures, invalid data.

  - **Calculate Score & Decision**  
    - Type: Code (JavaScript)  
    - Role: Processes logged feedback to calculate overall score and decide if candidate passed.  
    - Config: Custom JS code analyzing feedback metrics and setting a pass/fail decision.  
    - Inputs: From `Log Feedback`  
    - Outputs: Triggers `Passed Interview?`  
    - Edge Cases: Code errors, unexpected data structures, division by zero.

  - **Passed Interview?**  
    - Type: If  
    - Role: Routes workflow based on pass/fail decision.  
    - Config: Condition on calculated decision result.  
    - Inputs: From `Calculate Score & Decision`  
    - Outputs: Pass → `Update Status 'Passed'`; Fail → `Update Status 'Rejected'`  
    - Edge Cases: Incorrect branching if decision data malformed.

#### 1.3 Candidate Status Update & Communication

- **Overview**: Updates candidate status in Airtable, sends emails (offer or rejection), manages DocuSign signature process, and follow-up communications.

- **Nodes Involved**:  
  - Update Status 'Passed'  
  - Update Status 'Rejected'  
  - Send Follow-up Invitation  
  - Create Rejection Email  
  - Send Rejection Email  
  - DocuSign  
  - Signature (Wait)  
  - Update Status 'Offer Sent'

- **Node Details**:

  - **Update Status 'Passed'**  
    - Type: Airtable  
    - Role: Updates candidate record status to “Passed”.  
    - Config: Airtable fields updated accordingly.  
    - Inputs: From `Passed Interview?` (true path)  
    - Outputs: Triggers `Update Status 'Rejected'` (note: linear flow suggests further updates)  
    - Edge Cases: Airtable API errors, concurrency issues.

  - **Update Status 'Rejected'**  
    - Type: Airtable  
    - Role: Updates candidate record status to “Rejected”.  
    - Config: Maps status field to “Rejected”.  
    - Inputs: From `Update Status 'Passed'` (sequentially)  
    - Outputs: Triggers `Send Follow-up Invitation`  
    - Edge Cases: Data integrity risks if triggered incorrectly.

  - **Send Follow-up Invitation**  
    - Type: Gmail  
    - Role: Sends invitation email for further steps or re-engagement to rejected candidates.  
    - Config: Gmail OAuth2 credentials, email templates.  
    - Inputs: From `Update Status 'Rejected'`  
    - Outputs: Triggers `Create Rejection Email`  
    - Edge Cases: Email delivery failures, invalid addresses.

  - **Create Rejection Email**  
    - Type: Code (JavaScript)  
    - Role: Generates personalized rejection email content dynamically.  
    - Config: Uses candidate data and feedback to compose email body.  
    - Inputs: From `Send Follow-up Invitation`  
    - Outputs: Triggers `Send Rejection Email`  
    - Edge Cases: Code exceptions, missing candidate data.

  - **Send Rejection Email**  
    - Type: Gmail  
    - Role: Sends rejection email to candidate.  
    - Config: Gmail OAuth2, dynamic content from previous node.  
    - Inputs: From `Create Rejection Email`  
    - Outputs: Triggers `DocuSign` for possible document handling.  
    - Edge Cases: SMTP errors, mailbox full.

  - **DocuSign**  
    - Type: HTTP Request  
    - Role: Sends offer or rejection documents for signature via DocuSign API.  
    - Config: HTTP POST with authentication (API key/token), payload includes candidate info and document template.  
    - Inputs: From `Send Rejection Email`  
    - Outputs: Triggers `Signature` wait node  
    - Edge Cases: API errors, expired tokens, network failures.

  - **Signature**  
    - Type: Wait  
    - Role: Waits for candidate to complete DocuSign signature.  
    - Config: Wait period or event-based trigger.  
    - Inputs: From `DocuSign`  
    - Outputs: Triggers `Update Status 'Offer Sent'`  
    - Edge Cases: Timeout if signature not received; need for retries.

  - **Update Status 'Offer Sent'**  
    - Type: Airtable  
    - Role: Updates candidate status to “Offer Sent” in Airtable.  
    - Config: Status field set accordingly.  
    - Inputs: From `Signature`  
    - Outputs: Triggers `Create a card` in Trello  
    - Edge Cases: Airtable update conflicts.

#### 1.4 Task Management & Notifications

- **Overview**: Creates Trello cards for new candidates, notifies HR on Slack, and sends welcome emails after offer acceptance.

- **Nodes Involved**:  
  - Create a card (Trello)  
  - Notify HR (Slack)  
  - Send Welcome Email (Gmail)

- **Node Details**:

  - **Create a card**  
    - Type: Trello  
    - Role: Creates a Trello card in a specified board/list for candidate tracking.  
    - Config: Board and list IDs configured; card fields set with candidate info.  
    - Inputs: From `Update Status 'Offer Sent'`  
    - Outputs: Triggers `Notify HR`  
    - Edge Cases: API rate limits, invalid board/list IDs.

  - **Notify HR**  
    - Type: Slack  
    - Role: Sends notification message to HR Slack channel with candidate update.  
    - Config: Slack webhook ID `dd4dd863-3bb9-4dde-af0f-a387b0df1c59`.  
    - Inputs: From `Create a card`  
    - Outputs: Triggers `Send Welcome Email`  
    - Edge Cases: Slack webhook failures.

  - **Send Welcome Email**  
    - Type: Gmail  
    - Role: Sends welcome email to candidate after offer acceptance.  
    - Config: Gmail OAuth2, email template with dynamic content.  
    - Inputs: From `Notify HR`  
    - Outputs: None (end of this branch)  
    - Edge Cases: Delivery failures, invalid candidate email.

#### 1.5 Weekly Reporting

- **Overview**: Every Friday, the workflow gathers candidate data, calculates recruitment metrics, logs weekly reports in Google Sheets, and sends summary notifications on Slack.

- **Nodes Involved**:  
  - Every Friday (Schedule Trigger)  
  - Get Candidate Data (Airtable)  
  - Calculate Metrics (Code)  
  - Log Weekly Report (Google Sheets)  
  - Send Report Summary (Slack)

- **Node Details**:

  - **Every Friday**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow at a weekly interval (every Friday).  
    - Config: Weekly recurrence configured for Fridays.  
    - Inputs: External time trigger  
    - Outputs: Triggers `Get Candidate Data`  
    - Edge Cases: Missed execution on downtime.

  - **Get Candidate Data**  
    - Type: Airtable  
    - Role: Retrieves candidate data from Airtable for reporting.  
    - Config: Query filters for last week or relevant timeframe.  
    - Inputs: From `Every Friday`  
    - Outputs: Triggers `Calculate Metrics`  
    - Edge Cases: Large data sets causing timeouts.

  - **Calculate Metrics**  
    - Type: Code (JavaScript)  
    - Role: Computes metrics such as total candidates, rejection rates.  
    - Config: Custom JS code processing retrieved data.  
    - Inputs: From `Get Candidate Data`  
    - Outputs: Triggers `Log Weekly Report`  
    - Edge Cases: Data inconsistencies.

  - **Log Weekly Report**  
    - Type: Google Sheets  
    - Role: Appends calculated metrics to Google Sheets for historical tracking.  
    - Config: Sheet ID and range configured; data array includes date, totals, and rates.  
    - Inputs: From `Calculate Metrics`  
    - Outputs: Triggers `Send Report Summary`  
    - Edge Cases: API limits, sheet access permissions.

  - **Send Report Summary**  
    - Type: Slack  
    - Role: Sends a summary message of weekly recruitment metrics to Slack channel.  
    - Config: Slack webhook ID `6ad66853-3c20-450c-8848-0cf3474759ce`.  
    - Inputs: From `Log Weekly Report`  
    - Outputs: None (workflow end)  
    - Edge Cases: Slack delivery issues.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                              | Input Node(s)                | Output Node(s)                  | Sticky Note                            |
|-------------------------|----------------------------|----------------------------------------------|------------------------------|---------------------------------|--------------------------------------|
| Interview Completed      | Google Calendar Trigger    | Detects interview completion event           | -                            | Send Feedback Form              |                                      |
| Send Feedback Form       | Slack                      | Sends feedback form to interviewers via Slack| Interview Completed          | 2 Hours                        |                                      |
| 2 Hours                 | Wait                       | Waits 2 hours for feedback submission         | Send Feedback Form            | Check for Form Submission       |                                      |
| Check for Form Submission| HTTP Request               | Checks if feedback form is submitted          | 2 Hours                      | Form Submitted?                 |                                      |
| Form Submitted?          | If                         | Branches based on form submission status      | Check for Form Submission     | Send Reminder                  |                                      |
| Send Reminder            | Slack                      | Sends reminder to complete feedback form      | Form Submitted?               | Log Feedback                   |                                      |
| Log Feedback             | Airtable                   | Logs feedback data into Airtable               | Send Reminder                | Calculate Score & Decision      | Candidate Name={{ $json.candidate_name }}, Overall Score={{ $json.feedback_form.overall_score }}, Comments={{ $json.feedback_form.comments }} |
| Calculate Score & Decision| Code                      | Calculates score and interview decision        | Log Feedback                 | Passed Interview?              |                                      |
| Passed Interview?        | If                         | Branches workflow on pass/fail decision        | Calculate Score & Decision    | Update Status 'Passed', Update Status 'Rejected' |                                      |
| Update Status 'Passed'   | Airtable                   | Updates candidate status to “Passed”           | Passed Interview?             | Update Status 'Rejected'        |                                      |
| Update Status 'Rejected' | Airtable                   | Updates candidate status to “Rejected”         | Update Status 'Passed'        | Send Follow-up Invitation       |                                      |
| Send Follow-up Invitation| Gmail                      | Sends follow-up invitation email               | Update Status 'Rejected'      | Create Rejection Email          |                                      |
| Create Rejection Email   | Code                       | Creates personalized rejection email content   | Send Follow-up Invitation     | Send Rejection Email            |                                      |
| Send Rejection Email     | Gmail                      | Sends rejection email to candidate             | Create Rejection Email        | DocuSign                      |                                      |
| DocuSign                | HTTP Request               | Sends documents for signature via DocuSign API| Send Rejection Email          | Signature                     |                                      |
| Signature               | Wait                       | Waits for candidate to sign documents           | DocuSign                     | Update Status 'Offer Sent'      |                                      |
| Update Status 'Offer Sent'| Airtable                  | Updates candidate status to “Offer Sent”        | Signature                    | Create a card                  | Fields: Status=Offer Sent             |
| Create a card           | Trello                     | Creates Trello card for candidate tracking       | Update Status 'Offer Sent'    | Notify HR                     |                                      |
| Notify HR               | Slack                      | Sends Slack notification to HR team             | Create a card                | Send Welcome Email             |                                      |
| Send Welcome Email      | Gmail                      | Sends welcome email to candidate                 | Notify HR                    | -                             |                                      |
| Every Friday            | Schedule Trigger           | Weekly trigger every Friday                       | -                            | Get Candidate Data             |                                      |
| Get Candidate Data      | Airtable                   | Retrieves candidate data for report               | Every Friday                 | Calculate Metrics             |                                      |
| Calculate Metrics       | Code                       | Calculates recruitment metrics                     | Get Candidate Data           | Log Weekly Report             |                                      |
| Log Weekly Report       | Google Sheets              | Logs weekly recruitment metrics                    | Calculate Metrics            | Send Report Summary           | Values: [ [ "={{ $now }}", "={{ $json.totalCandidates }}", "={{ $json.rejectionRate }}" ] ] |
| Send Report Summary     | Slack                      | Sends weekly report summary to Slack channel      | Log Weekly Report            | -                             |                                      |
| Sticky Note             | Sticky Note                | -                                                | -                            | -                             |                                      |
| Sticky Note1            | Sticky Note                | -                                                | -                            | -                             |                                      |
| Sticky Note2            | Sticky Note                | -                                                | -                            | -                             |                                      |
| Sticky Note3            | Sticky Note                | -                                                | -                            | -                             |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger node**  
   - Name: `Interview Completed`  
   - Type: Google Calendar Trigger  
   - Configure to watch for interview event completions on the relevant calendar.

2. **Create Slack node**  
   - Name: `Send Feedback Form`  
   - Type: Slack  
   - Configure Slack webhook (ID: `d1ead6e6-ae65-4aa2-91be-a95dc621c07f`)  
   - Compose message to send feedback form link/instructions after interview.

3. **Connect `Interview Completed` → `Send Feedback Form`**

4. **Create Wait node**  
   - Name: `2 Hours`  
   - Type: Wait  
   - Set to wait for 2 hours.

5. **Connect `Send Feedback Form` → `2 Hours`**

6. **Create HTTP Request node**  
   - Name: `Check for Form Submission`  
   - Type: HTTP Request  
   - Configure method (GET/POST) and URL to check feedback form submission status.  
   - Set authentication as needed.

7. **Connect `2 Hours` → `Check for Form Submission`**

8. **Create If node**  
   - Name: `Form Submitted?`  
   - Type: If  
   - Condition based on HTTP request response indicating feedback submission.

9. **Connect `Check for Form Submission` → `Form Submitted?`**

10. **Create Slack node**  
    - Name: `Send Reminder`  
    - Type: Slack  
    - Configure Slack webhook (ID: `6486280f-617c-47e6-bd2e-99c0c59170ea`)  
    - Message reminds interviewers to submit feedback.

11. **Connect `Form Submitted?` false output → `Send Reminder`**

12. **Create Airtable node**  
    - Name: `Log Feedback`  
    - Type: Airtable  
    - Configure base and table to log candidate feedback  
    - Map fields: candidate_name, overall_score, comments from form.

13. **Connect `Send Reminder` → `Log Feedback`**

14. **Create Code node**  
    - Name: `Calculate Score & Decision`  
    - Type: Code  
    - Add JS code to compute overall score and pass/fail decision from feedback.

15. **Connect `Log Feedback` → `Calculate Score & Decision`**

16. **Create If node**  
    - Name: `Passed Interview?`  
    - Type: If  
    - Condition based on score calculation result.

17. **Connect `Calculate Score & Decision` → `Passed Interview?`**

18. **Create Airtable nodes for status updates**  
    - Name: `Update Status 'Passed'`  
    - Configure to set candidate status to “Passed”  
    - Name: `Update Status 'Rejected'`  
    - Configure to set candidate status to “Rejected”

19. **Connect `Passed Interview?` true output → `Update Status 'Passed'`**  
    Connect false output → `Update Status 'Rejected'`

20. **Create Gmail node**  
    - Name: `Send Follow-up Invitation`  
    - Configure Gmail OAuth2 credentials and email template for rejected candidates.

21. **Connect `Update Status 'Rejected'` → `Send Follow-up Invitation`**

22. **Create Code node**  
    - Name: `Create Rejection Email`  
    - JS code to generate personalized rejection email content.

23. **Connect `Send Follow-up Invitation` → `Create Rejection Email`**

24. **Create Gmail node**  
    - Name: `Send Rejection Email`  
    - Configure Gmail OAuth2, use content from previous node.

25. **Connect `Create Rejection Email` → `Send Rejection Email`**

26. **Create HTTP Request node**  
    - Name: `DocuSign`  
    - Configure HTTP POST to DocuSign API with credentials and document payload.

27. **Connect `Send Rejection Email` → `DocuSign`**

28. **Create Wait node**  
    - Name: `Signature`  
    - Configure wait for signature event/timeout.

29. **Connect `DocuSign` → `Signature`**

30. **Create Airtable node**  
    - Name: `Update Status 'Offer Sent'`  
    - Configure to update candidate status to “Offer Sent”.

31. **Connect `Signature` → `Update Status 'Offer Sent'`**

32. **Create Trello node**  
    - Name: `Create a card`  
    - Configure Trello board and list where candidate cards will be created.

33. **Connect `Update Status 'Offer Sent'` → `Create a card`**

34. **Create Slack node**  
    - Name: `Notify HR`  
    - Configure Slack webhook (ID: `dd4dd863-3bb9-4dde-af0f-a387b0df1c59`) to notify HR.

35. **Connect `Create a card` → `Notify HR`**

36. **Create Gmail node**  
    - Name: `Send Welcome Email`  
    - Configure Gmail OAuth2 and welcome email template.

37. **Connect `Notify HR` → `Send Welcome Email`**

38. **Create Schedule Trigger node**  
    - Name: `Every Friday`  
    - Configure to trigger every Friday.

39. **Create Airtable node**  
    - Name: `Get Candidate Data`  
    - Configure to query candidate data for reporting.

40. **Connect `Every Friday` → `Get Candidate Data`**

41. **Create Code node**  
    - Name: `Calculate Metrics`  
    - Add JS code to calculate total candidates, rejection rate, etc.

42. **Connect `Get Candidate Data` → `Calculate Metrics`**

43. **Create Google Sheets node**  
    - Name: `Log Weekly Report`  
    - Configure to append rows with date and calculated metrics.

44. **Connect `Calculate Metrics` → `Log Weekly Report`**

45. **Create Slack node**  
    - Name: `Send Report Summary`  
    - Configure Slack webhook (ID: `6ad66853-3c20-450c-8848-0cf3474759ce`) for report messages.

46. **Connect `Log Weekly Report` → `Send Report Summary`**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                       |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Airtable fields in `Log Feedback` include candidate_name, overall_score, comments for tracking.    | Used for feedback data logging.                       |
| Google Sheets `Log Weekly Report` stores date, total candidates, and rejection rate metrics.       | Useful for recruitment trend analysis.                |
| Slack webhook IDs are critical and must be configured correctly for messaging nodes.                | Check Slack app permissions and webhook validity.    |
| DocuSign integration requires API credentials and correct document templates for signature flows. | Refer to DocuSign API docs for setup details.         |

---

**Disclaimer:** The provided content is exclusively from an automated workflow created using n8n, an integration and automation tool. The processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.