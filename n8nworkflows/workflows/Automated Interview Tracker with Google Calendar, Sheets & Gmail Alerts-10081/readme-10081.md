Automated Interview Tracker with Google Calendar, Sheets & Gmail Alerts

https://n8nworkflows.xyz/workflows/automated-interview-tracker-with-google-calendar--sheets---gmail-alerts-10081


# Automated Interview Tracker with Google Calendar, Sheets & Gmail Alerts

### 1. Workflow Overview

This workflow automates the tracking and notification process for candidate interviews using Google Calendar, Google Sheets, and Gmail. It supports two main functional blocks:  
- **1.1 Interview Monitoring and Reminder**: Periodically checks for upcoming interview events in Google Calendar, filters them to those starting within the next 5 minutes, logs them into a Google Sheet, and sends reminder emails to candidates.  
- **1.2 Interview Result Processing and Notification**: Listens for incoming interview results via a webhook, updates the Google Sheet accordingly, evaluates pass/fail status, and sends tailored email notifications both to the candidate and the hiring manager.

This structure ensures continuous monitoring and candidate engagement throughout the interviewing process, while also maintaining a centralized record of interviews and results.

---

### 2. Block-by-Block Analysis

#### 2.1 Interview Monitoring and Reminder

**Overview:**  
This block runs every 5 minutes to retrieve upcoming interviews from a specified Google Calendar, filters interviews starting soon, records them into a Google Sheet, and sends reminder emails to candidates.

**Nodes Involved:**  
- Check Every 5 Minutes  
- Get Calendar Events  
- Filter Upcoming Interviews  
- Add to Google Sheet  
- Send Reminder to Candidate

**Node Details:**

- **Check Every 5 Minutes**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow every 5 minutes to check for upcoming interviews.  
  - Configuration: Interval set to 5 minutes.  
  - Inputs: None (trigger node).  
  - Outputs: To "Get Calendar Events".  
  - Edge Cases: If n8n server is down or delayed, triggers may be missed, causing reminder delays.

- **Get Calendar Events**  
  - Type: Google Calendar  
  - Role: Fetches all calendar events starting from the current time to one hour ahead from a specific calendar ID.  
  - Configuration:  
    - Operation: getAll events  
    - Time Range: Now (start) to now + 1 hour (end)  
    - Calendar ID: Provided as a fixed value (masked here).  
    - Credentials: Google OAuth2 for calendar access.  
  - Inputs: From Schedule Trigger.  
  - Outputs: To "Filter Upcoming Interviews".  
  - Edge Cases:  
    - OAuth token expiration or permission issues.  
    - Empty calendar or no events in time window.

- **Filter Upcoming Interviews**  
  - Type: Filter  
  - Role: Filters events to those starting strictly between now and 5 minutes from now.  
  - Configuration:  
    - Conditions:  
      - Event start time before now + 5 minutes  
      - Event start time after now  
    - Both conditions combined with AND.  
  - Inputs: From Google Calendar node.  
  - Outputs: To "Add to Google Sheet" and "Send Reminder to Candidate".  
  - Edge Cases:  
    - Events without a valid start.dateTime property may cause evaluation errors.  
    - Timezone discrepancies might affect filtering.

- **Add to Google Sheet**  
  - Type: Google Sheets  
  - Role: Appends or updates interview event data into a specified Google Sheet to track upcoming interviews.  
  - Configuration:  
    - Operation: appendOrUpdate (auto mapping input data to columns)  
    - Sheet Name: "Sheet1"  
    - Document ID: Placeholder "YOUR_SPREADSHEET_ID" (must be set)  
    - Authentication: Service Account  
  - Inputs: From Filter node.  
  - Outputs: None (end node for this branch).  
  - Credentials: Google Sheets API with service account.  
  - Edge Cases:  
    - Invalid or missing spreadsheet ID leading to failures.  
    - Insufficient permissions on spreadsheet.  
    - Data mapping errors if input data structure changes.

- **Send Reminder to Candidate**  
  - Type: Gmail  
  - Role: Sends an email reminder to the candidate about the upcoming interview starting in 5 minutes.  
  - Configuration:  
    - Recipient: Attendee email from event JSON (`$json.attendees[0].email`)  
    - Subject: "Interview Reminder - Starting in 5 Minutes"  
    - Message: HTML formatted reminder including interview summary, start time, and Google Meet link.  
  - Inputs: From Filter node.  
  - Outputs: None.  
  - Credentials: Gmail OAuth2 credentials.  
  - Edge Cases:  
    - Missing or malformed attendee email causes send failure.  
    - Gmail API quota limits or OAuth token expiration.  
    - Event missing hangoutLink or summary fields.

---

#### 2.2 Interview Result Processing and Notification

**Overview:**  
This block handles the reception of interview results via a webhook, updates the Google Sheet with the results, evaluates if the candidate passed or failed, and sends appropriate email notifications to both the candidate and the hiring manager.

**Nodes Involved:**  
- Webhook: Submit Interview Result  
- Update Sheet with Result  
- Check if Passed  
- Email Candidate - Passed  
- Email Candidate - Failed  
- Email Manager

**Node Details:**

- **Webhook: Submit Interview Result**  
  - Type: Webhook  
  - Role: Receives POST requests containing interview result data (candidate email, result, feedback) at path "interview-result".  
  - Configuration:  
    - HTTP Method: POST  
    - Webhook Path: "interview-result"  
  - Inputs: External HTTP POST requests.  
  - Outputs: To "Update Sheet with Result".  
  - Edge Cases:  
    - Missing or malformed JSON body leads to processing errors.  
    - Security considerations: webhook is publicly accessible unless protected externally.

- **Update Sheet with Result**  
  - Type: Google Sheets  
  - Role: Appends or updates the Google Sheet with submitted interview result data.  
  - Configuration:  
    - Operation: appendOrUpdate with auto mapping  
    - Sheet Name: "Sheet1"  
    - Document ID: Placeholder "YOUR_SPREADSHEET_ID"  
    - Authentication: Service Account  
  - Inputs: From Webhook node.  
  - Outputs: To "Check if Passed".  
  - Credentials: Google Sheets API service account.  
  - Edge Cases:  
    - Spreadsheet ID must be valid.  
    - Data format mismatches.  
    - Permissions issues.

- **Check if Passed**  
  - Type: If  
  - Role: Evaluates if submitted interview result is "Pass".  
  - Configuration:  
    - Condition: `$json.result` equals string "Pass"  
  - Inputs: From Update Sheet node.  
  - Outputs:  
    - True branch: to "Email Candidate - Passed"  
    - False branch: to "Email Candidate - Failed"  
  - Edge Cases:  
    - Case sensitivity; the condition is case sensitive and strict.  
    - Missing or unexpected `$json.result` values.

- **Email Candidate - Passed**  
  - Type: Gmail  
  - Role: Sends congratulatory email to candidate who passed the interview with feedback.  
  - Configuration:  
    - Recipient: `$json.candidateEmail`  
    - Subject: "🎉 Congratulations! You Passed the Interview"  
    - Message: HTML formatted congratulation with feedback and next steps info.  
  - Inputs: True branch from If node.  
  - Outputs: To "Email Manager".  
  - Credentials: Gmail OAuth2.  
  - Edge Cases:  
    - Missing candidate email causes send failure.  
    - Gmail API issues or quota limits.

- **Email Candidate - Failed**  
  - Type: Gmail  
  - Role: Sends failure notification email to candidate with feedback.  
  - Configuration:  
    - Recipient: `$json.candidateEmail`  
    - Subject: "Interview Result"  
    - Message: HTML formatted regret email with feedback.  
  - Inputs: False branch from If node.  
  - Outputs: To "Email Manager".  
  - Credentials: Gmail OAuth2.  
  - Edge Cases: Similar to "Email Candidate - Passed".

- **Email Manager**  
  - Type: Gmail  
  - Role: Sends summary email to hiring manager about the interview result and candidate feedback.  
  - Configuration:  
    - Recipient: Fixed address "manager@company.com"  
    - Subject: "Interview Result: {{ $json.candidateEmail }}"  
    - Message: HTML formatted summary including candidate email, result, and feedback.  
  - Inputs: From both candidate email nodes.  
  - Outputs: None.  
  - Credentials: Gmail OAuth2.  
  - Edge Cases:  
    - Email delivery issues.  
    - Manager address hardcoded; needs updating if changed.

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                                | Input Node(s)              | Output Node(s)                      | Sticky Note                                                                                              |
|-----------------------------|----------------------|-----------------------------------------------|----------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------|
| Check Every 5 Minutes        | Schedule Trigger     | Periodically triggers workflow every 5 minutes | None                       | Get Calendar Events               | - **Check Every 5 Minutes**: Triggers the workflow to run every 5 minutes.                             |
| Get Calendar Events          | Google Calendar      | Retrieves upcoming interview events            | Check Every 5 Minutes       | Filter Upcoming Interviews        | - **Get Calendar Events**: Retrieves upcoming interview events from Google Calendar.                   |
| Filter Upcoming Interviews   | Filter               | Filters events starting within next 5 minutes | Get Calendar Events         | Add to Google Sheet, Send Reminder to Candidate | - **Filter Upcoming Interviews**: Filters the retrieved events to focus on upcoming interviews.          |
| Add to Google Sheet          | Google Sheets        | Logs filtered interview data to Google Sheet  | Filter Upcoming Interviews  | None                            | - **Add to Google Sheet**: Appends filtered interview details to a Google Sheet.                       |
| Send Reminder to Candidate   | Gmail                | Sends reminder email to candidate              | Filter Upcoming Interviews  | None                            | - **Send Reminder to Candidate**: Sends an email reminder to the candidate via Gmail.                  |
| Webhook: Submit Interview Result | Webhook          | Receives interview result data via POST        | External HTTP POST          | Update Sheet with Result          | - **Webhook: Submit Interview Result**: Receives interview result data via a webhook.                  |
| Update Sheet with Result     | Google Sheets        | Updates Google Sheet with interview result     | Webhook: Submit Interview Result | Check if Passed                | - **Update Sheet with Result**: Updates the Google Sheet with the submitted interview result.          |
| Check if Passed              | If                   | Checks if candidate passed interview           | Update Sheet with Result    | Email Candidate - Passed, Email Candidate - Failed | - **Check if Passed**: Evaluates whether the interview result indicates a pass or fail.                  |
| Email Candidate - Passed     | Gmail                | Sends pass notification email to candidate     | Check if Passed (true)      | Email Manager                   | - **Email Candidate - Passed**: Sends a pass notification email to the candidate via Gmail.            |
| Email Candidate - Failed     | Gmail                | Sends fail notification email to candidate     | Check if Passed (false)     | Email Manager                   | - **Email Candidate - Failed**: Sends a fail notification email to the candidate via Gmail.            |
| Email Manager               | Gmail                | Sends summary email to manager                   | Email Candidate - Passed, Email Candidate - Failed | None                     | - **Email Manager**: Sends an email update to the manager via Gmail.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Name: "Check Every 5 Minutes"  
   - Set trigger interval to every 5 minutes.

2. **Create a Google Calendar node**  
   - Name: "Get Calendar Events"  
   - Credentials: Configure with your Google Calendar OAuth2 credentials.  
   - Operation: "getAll" events.  
   - Calendar ID: Set to your interviews calendar ID.  
   - Time Min: Expression `{{$now.toISO()}}`  
   - Time Max: Expression `{{$now.plus({hours:1}).toISO()}}`  
   - Connect output of "Check Every 5 Minutes" to this node.

3. **Add a Filter node**  
   - Name: "Filter Upcoming Interviews"  
   - Conditions (AND):  
     - Left: `{{$json.start.dateTime}}` < Right: `{{$now.plus({minutes:5}).toISO()}}`  
     - Left: `{{$json.start.dateTime}}` > Right: `{{$now.toISO()}}`  
   - Connect output of "Get Calendar Events" to this node.

4. **Create a Google Sheets node**  
   - Name: "Add to Google Sheet"  
   - Credentials: Configure with Google Sheets service account credentials.  
   - Operation: appendOrUpdate  
   - Document ID: Set your spreadsheet ID containing interview tracking sheet.  
   - Sheet Name: "Sheet1"  
   - Mapping Mode: Auto map input data fields.  
   - Connect the **true** output of "Filter Upcoming Interviews" to this node.

5. **Add a Gmail node to send reminders**  
   - Name: "Send Reminder to Candidate"  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - Recipient: Expression `{{$json.attendees[0].email}}`  
   - Subject: "Interview Reminder - Starting in 5 Minutes"  
   - Message (HTML): Include candidate greeting, interview summary (`{{$json.summary}}`), start time (`{{$json.start.dateTime}}`), and Google Meet link (`{{$json.hangoutLink}}`).  
   - Connect the same output of "Filter Upcoming Interviews" to this node, parallel to "Add to Google Sheet".

6. **Create a Webhook node**  
   - Name: "Webhook: Submit Interview Result"  
   - HTTP Method: POST  
   - Path: "interview-result"  
   - This node will receive JSON payloads with fields like `candidateEmail`, `result`, and `feedback`.

7. **Create a Google Sheets node**  
   - Name: "Update Sheet with Result"  
   - Credentials: Google Sheets service account.  
   - Operation: appendOrUpdate  
   - Document ID: Same spreadsheet as above  
   - Sheet Name: "Sheet1"  
   - Connect webhook output to this node.

8. **Add an If node**  
   - Name: "Check if Passed"  
   - Condition: `$json.result` equals string "Pass" (case sensitive).  
   - Connect "Update Sheet with Result" output to this node.

9. **Create Gmail node to email passing candidates**  
   - Name: "Email Candidate - Passed"  
   - Credentials: Gmail OAuth2  
   - Recipient: `{{$json.candidateEmail}}`  
   - Subject: "🎉 Congratulations! You Passed the Interview"  
   - Message (HTML): Congratulatory message including feedback (`{{$json.feedback}}`).  
   - Connect If node **true** branch to this node.

10. **Create Gmail node to email failing candidates**  
    - Name: "Email Candidate - Failed"  
    - Credentials: Gmail OAuth2  
    - Recipient: `{{$json.candidateEmail}}`  
    - Subject: "Interview Result"  
    - Message (HTML): Polite rejection with feedback (`{{$json.feedback}}`).  
    - Connect If node **false** branch to this node.

11. **Create Gmail node to notify manager**  
    - Name: "Email Manager"  
    - Credentials: Gmail OAuth2  
    - Recipient: "manager@company.com" (update as necessary)  
    - Subject: "Interview Result: {{$json.candidateEmail}}"  
    - Message (HTML): Summary of candidate email, result, and feedback.  
    - Connect outputs of both candidate email nodes to this node.

12. **Final connections and testing**  
    - Verify all credentials are properly configured with necessary scopes.  
    - Replace placeholder spreadsheet IDs and calendar IDs with real IDs.  
    - Test schedule trigger and webhook endpoints.  
    - Ensure Gmail accounts have API access enabled and sufficient quota.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The Google Sheets nodes operate with service account authentication; ensure the service account has edit access to the spreadsheet. | Google Sheets API and Service Account setup documentation.      |
| Gmail OAuth2 credentials must have send email permissions and be authorized for the respective accounts used in the workflow.       | Gmail API OAuth2 setup guides.                                  |
| The webhook node is publicly accessible; consider adding authentication or IP filtering for security.                                | n8n Webhook security best practices.                            |
| Google Calendar event times are handled in ISO format; ensure all event times and server timezone settings align to avoid mismatches.| Google Calendar API time zone handling documentation.          |
| To update the spreadsheet ID and calendar ID placeholders, use your actual Google resources IDs.                                      | How to find Google Calendar and Sheet IDs in Google Workspace. |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a no-code automation tool. It adheres strictly to content policies and includes no illegal or protected content. All handled data is legal and public.