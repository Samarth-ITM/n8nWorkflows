Business Hours Lead Response System with Gmail, Google Sheets & Telegram Alerts

https://n8nworkflows.xyz/workflows/business-hours-lead-response-system-with-gmail--google-sheets---telegram-alerts-8282


# Business Hours Lead Response System with Gmail, Google Sheets & Telegram Alerts

### 1. Workflow Overview

This workflow automates lead response management based on business hours using Gmail, Google Sheets, and Telegram for notifications. It targets organizations that receive lead submissions via Google Forms (stored in Google Sheets) and want to send timely, context-aware email replies along with internal alerts.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Monitors Google Sheets for new or updated lead entries.
- **1.2 Processing Delay:** Waits 5 minutes before processing to ensure data completeness.
- **1.3 Business Hours Determination:** Converts timestamps and checks if the lead arrived during business hours.
- **1.4 Response Routing:** Routes the lead through two different paths based on business hours.
- **1.5 Email Response (Business Hours):** Sends a prompt email during business hours.
- **1.6 Email Response Preparation (After Hours):** Standardizes fields for after-hours email.
- **1.7 Email Response (After Hours):** Sends a delayed response outside business hours.
- **1.8 Team Notification:** Sends a Telegram message alerting the team of the new lead.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:** Listens for new or updated rows in a specified Google Sheets document containing lead form responses.
- **Nodes Involved:** Google Sheets Trigger1

**Node Details:**

- **Google Sheets Trigger1**
  - Type: Trigger node (Google Sheets Trigger)
  - Configuration:
    - Monitors sheet named "Form Responses 1" within the spreadsheet "Lead form (Responses)".
    - Event: Row update.
    - Polling frequency: every minute.
  - Key variables: Captures all columns including "First Name", "Last name", "Email address", and "Time Stamp".
  - Connections: Outputs to "Wait 5 Minutes".
  - Failure cases: Google API access errors, credential expiration, or sheet unavailability.
  - Notes: Critical as the entry point for the workflow.

#### Block 1.2: Processing Delay

- **Overview:** Adds a 5-minute delay to ensure any concurrent form edits complete before processing.
- **Nodes Involved:** Wait 5 Minutes

**Node Details:**

- **Wait 5 Minutes**
  - Type: Wait node
  - Configuration: Waits for 5 minutes before passing data forward.
  - Input: From Google Sheets Trigger1.
  - Output: To Check Business Hours.
  - Edge cases: Delay node failures are rare; ensure system clock accuracy.

#### Block 1.3: Business Hours Determination

- **Overview:** Converts the Google Sheets timestamp (Excel serial date) into a JavaScript Date object, determines the day and hour, and flags if the lead submission is within predefined business hours (Mon-Fri, 9:00-18:00).
- **Nodes Involved:** Check Business Hours

**Node Details:**

- **Check Business Hours**
  - Type: Function node (custom JavaScript)
  - Configuration:
    - Reads "Time Stamp" field as Excel serial date.
    - Converts to ISO datetime string.
    - Extracts day (0=Sun to 6=Sat) and hour.
    - Business days: Monday to Friday (1-5).
    - Business hours: 9:00 to before 18:00.
    - Outputs fields: `exactDateTime` (ISO), `onlyTime` (HH:mm:ss), and `isBusinessHours` (boolean).
  - Input: From Wait 5 Minutes.
  - Output: To IF Business Hours? node.
  - Failure cases: If "Time Stamp" is missing or malformed, conversion will fail.
  - Version: Compatible with n8n v1.x function node syntax.

#### Block 1.4: Response Routing

- **Overview:** Uses conditional logic to route the workflow based on whether the lead was received during business hours.
- **Nodes Involved:** IF Business Hours?

**Node Details:**

- **IF Business Hours?**
  - Type: IF node (conditional)
  - Configuration:
    - Condition checks if `exactDateTime` is before 2025-09-02T18:00:00 — this seems to be a placeholder and may need adjustment.
    - **Note:** This condition as configured is a placeholder and does not actually check the `isBusinessHours` boolean. The business hours flag should ideally be used here.
  - Input: From Check Business Hours.
  - Outputs:
    - True branch: Sends Gmail (Business Hours).
    - False branch: Proceeds to Edit Fields (preparing for after-hours email).
  - Failure cases: Incorrect condition logic or date references can misroute leads.

#### Block 1.5: Email Response (Business Hours)

- **Overview:** Sends a prompt, personalized reply email for leads received during business hours.
- **Nodes Involved:** Send Gmail (Business Hours)

**Node Details:**

- **Send Gmail (Business Hours)**
  - Type: Gmail node (email sending)
  - Configuration:
    - Recipient: Lead's "Email Address".
    - Subject: "Thanks for reaching out!"
    - Message: Personalized greeting with first and last name, thanking the lead.
    - Credentials: Gmail OAuth2 account configured.
  - Input: From IF Business Hours? (true branch).
  - Output: To Notify Telegram.
  - Failure cases: Gmail API errors, auth token expiration, invalid email addresses.

#### Block 1.6: Email Response Preparation (After Hours)

- **Overview:** Standardizes and maps lead data fields to ensure proper formatting for the after-hours email.
- **Nodes Involved:** Edit Fields

**Node Details:**

- **Edit Fields**
  - Type: Set node (data manipulation)
  - Configuration:
    - Assigns standardized field names "First name", "Last name", "Email address" from Google Sheets Trigger1.
    - Ensures data consistency before sending the after-hours email.
  - Input: From IF Business Hours? (false branch).
  - Output: To Send Gmail (After Hours).
  - Failure cases: Missing data fields due to inconsistent naming or data entry.

#### Block 1.7: Email Response (After Hours)

- **Overview:** Sends a tailored email acknowledging the lead was received outside business hours and sets expectation for next-day contact.
- **Nodes Involved:** Send Gmail (After Hours)

**Node Details:**

- **Send Gmail (After Hours)**
  - Type: Gmail node
  - Configuration:
    - Recipient: Standardized "Email address" field.
    - Subject: "Thanks for contacting us (After Hours)".
    - Message: Personalized greeting, thanks for after-hours contact, and promise of next business day response.
    - Credentials: Same Gmail OAuth2 account.
  - Input: From Edit Fields.
  - Output: To Notify Telegram.
  - Failure cases: Same as business hours Gmail node.

#### Block 1.8: Team Notification

- **Overview:** Sends a Telegram message to a specified chat to alert the internal team of the new lead, regardless of business hours.
- **Nodes Involved:** Notify Telegram

**Node Details:**

- **Notify Telegram**
  - Type: Telegram node (message sending)
  - Configuration:
    - Chat ID: Specified Telegram group or user ID.
    - Text: Includes lead's full name and email address, mentions next-day follow-up.
    - Credentials: Configured Telegram API token.
  - Input: From both Send Gmail (Business Hours) and Send Gmail (After Hours).
  - Output: None (end of workflow).
  - Failure cases: Invalid chat ID, Telegram API issues, message formatting errors.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role          | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                 |
|-------------------------|-------------------------|-------------------------|------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger1   | Google Sheets Trigger   | Lead Form Monitor       | -                      | Wait 5 Minutes                | This node continuously monitors a spreadsheet for new or updated lead form submissions.                      |
| Wait 5 Minutes          | Wait                    | Processing Buffer       | Google Sheets Trigger1  | Check Business Hours          | Adds a 5-minute delay to ensure data completeness before processing.                                        |
| Check Business Hours    | Function                | Time Analysis Engine    | Wait 5 Minutes          | IF Business Hours?            | Converts timestamps to datetime, determines if submission occurred during business hours.                   |
| IF Business Hours?      | IF                      | Response Path Router    | Check Business Hours    | Send Gmail (Business Hours), Edit Fields | Evaluates business hours condition and routes workflow accordingly.                                          |
| Send Gmail (Business Hours) | Gmail                  | Standard Business Response | IF Business Hours? (true) | Notify Telegram              | Sends prompt email response during business hours.                                                         |
| Edit Fields             | Set                     | Field Standardizer      | IF Business Hours? (false) | Send Gmail (After Hours)    | Standardizes lead data fields for after-hours email.                                                        |
| Send Gmail (After Hours) | Gmail                   | After-Hours Response    | Edit Fields             | Notify Telegram              | Sends tailored email response after business hours.                                                        |
| Notify Telegram         | Telegram                | Team Notification       | Send Gmail (Business Hours), Send Gmail (After Hours) | -                             | Sends Telegram alert to team with lead details.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node:**
   - Type: Google Sheets Trigger
   - Credentials: Configure OAuth2 credentials with access to Google Sheets.
   - Parameters:
     - Event: Row Update.
     - Polling frequency: Every minute.
     - Spreadsheet ID: Use your Google Sheets ID for the lead form responses.
     - Sheet Name: "Form Responses 1".
   - Purpose: Trigger workflow on new or updated lead submission.

2. **Add Wait Node:**
   - Type: Wait
   - Parameters:
     - Unit: Minutes.
     - Amount: 5.
   - Connect output of Google Sheets Trigger to this node.
   - Purpose: Delay processing to ensure data stability.

3. **Add Function Node (Check Business Hours):**
   - Type: Function
   - Code:
     ```javascript
     return items.map(item => {
       const timestamp = item.json["Time Stamp"];
       const excelEpoch = new Date(Date.UTC(1899, 11, 30));
       const jsDate = new Date(excelEpoch.getTime() + timestamp * 24 * 60 * 60 * 1000);

       const day = jsDate.getDay();
       const hour = jsDate.getHours();

       const businessDays = [1, 2, 3, 4, 5];
       const startHour = 9;
       const endHour = 18;

       let isBusinessHours = false;
       if (businessDays.includes(day) && hour >= startHour && hour < endHour) {
         isBusinessHours = true;
       }

       const onlyTime = jsDate.toTimeString().substring(0, 8);

       return {
         json: {
           ...item.json,
           exactDateTime: jsDate.toISOString(),
           onlyTime,
           isBusinessHours,
         },
       };
     });
     ```
   - Connect output of Wait node to this node.

4. **Add IF Node (IF Business Hours?):**
   - Type: IF
   - Condition:
     - Ideally configure to check `isBusinessHours` boolean field from previous node.
     - Example: Expression `{{$json.isBusinessHours}}` equals `true`.
   - Connect output of Check Business Hours to this IF node.
   - True branch: Leads received during business hours.
   - False branch: Leads received after business hours.

5. **Add Gmail Node for Business Hours Response:**
   - Type: Gmail
   - Credentials: Gmail OAuth2 account.
   - Parameters:
     - To: `={{ $json["Email Address"] }}`
     - Subject: "Thanks for reaching out!"
     - Message:
       ```
       Hii {{$json["First Name"]}} {{$json["Last name"]}}

       Thanks for connecting,
       Our team will get back to you soon
       ```
   - Connect True output of IF node to this node.

6. **Add Set Node (Edit Fields) for After Hours:**
   - Type: Set
   - Parameters:
     - Assign "First name" = `={{ $('Google Sheets Trigger1').item.json['First name'] }}`
     - Assign "Last name" = `={{ $('Google Sheets Trigger1').item.json['Last name'] }}`
     - Assign "Email address" = `={{ $('Google Sheets Trigger1').item.json['Email address'] }}`
   - Connect False output of IF node to this node.

7. **Add Gmail Node for After Hours Response:**
   - Type: Gmail
   - Credentials: Same Gmail OAuth2 account.
   - Parameters:
     - To: `={{ $json["Email address"] }}`
     - Subject: "Thanks for contacting us (After Hours)"
     - Message:
       ```
       Hii {{$json["First name"]}} {{$json["Last name"]}}

       Thanks for connecting in after hours
       Our team will get back to you tomorrow
       ```
   - Connect output of Edit Fields node to this node.

8. **Add Telegram Node (Notify Telegram):**
   - Type: Telegram
   - Credentials: Telegram API token.
   - Parameters:
     - Chat ID: Your target chat or group ID.
     - Text:
       ```
       📩 New lead received!
       Name: {{ $('Google Sheets Trigger1').item.json['First Name'] }} {{ $('Google Sheets Trigger1').item.json['Last name'] }}
       Email: {{ $('Google Sheets Trigger1').item.json['Email Address'] }}

       Connect with the lead tomorrow
       ```
   - Connect outputs of both Gmail nodes (Business Hours and After Hours) to this node.

9. **Ensure all credentials (Google Sheets, Gmail OAuth2, Telegram API) are properly configured and tested.**

10. **Activate the workflow and test by submitting leads to the connected Google Form or updating rows in Google Sheets.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The current IF node condition uses a static date check (`exactDateTime` before 2025-09-02T18:00:00), which is a placeholder. For correct business hours routing, replace this with a condition based on the `isBusinessHours` boolean. | Critical logic improvement for proper workflow routing.                                               |
| The workflow uses OAuth2 credentials for Gmail and Google Sheets access; ensure tokens are refreshed and scopes correctly assigned.                                                                                                | Credential management best practice.                                                                  |
| Telegram notifications require the chat ID of the target group or user; obtain this ID via Telegram Bot API or chat history.                                                                                                      | Telegram API documentation: https://core.telegram.org/bots/api#sendmessage                             |
| Delay node is used to avoid race conditions with form data submission; adjust delay time as needed depending on form submission speed and system performance.                                                                       | Timing considerations for reliable automation.                                                        |
| For more information on n8n Gmail node configuration, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                                                                                                | Official n8n documentation.                                                                            |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.