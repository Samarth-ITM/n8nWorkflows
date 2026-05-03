Email Newsletter System with SendGrid, Google Sheets & Freemium Rate Limiting

https://n8nworkflows.xyz/workflows/email-newsletter-system-with-sendgrid--google-sheets---freemium-rate-limiting-11759


# Email Newsletter System with SendGrid, Google Sheets & Freemium Rate Limiting

### 1. Workflow Overview

This workflow implements an **Email Newsletter System** integrating **SendGrid**, **Google Sheets**, and a **Freemium Rate Limiting** mechanism. Its primary purpose is to process email newsletter requests received via a webhook, differentiate between free ("demo") and pro users, enforce a daily sending limit for free users, and update user sending statistics in a Google Sheet. It also sends email notifications through SendGrid and alerts the administrator via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and User Mode Check:** Receives newsletter data via webhook and checks if the user is "demo" (free) or "pro".
- **1.2 User Existence Verification:** Reads Google Sheets to check if the user exists and retrieves their sending history.
- **1.3 Rate Limiting Check:** For demo users, verifies if the daily sending limit (5 emails/day) has been reached.
- **1.4 User Data Update and Email Sending:** Updates Google Sheets with new usage counts and sends the email via SendGrid.
- **1.5 Response and Notifications:** Sends success or limit-exceeded responses to the requester and sends Telegram notifications to the administrator.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and User Mode Check

- **Overview:**  
  Receives incoming newsletter requests from a web form, parses the payload, and routes processing based on user mode ("demo" or "pro").

- **Nodes Involved:**  
  - Webhook1  
  - Check Mode1  
  - Send Email (Pro)  
  - Success Response (Pro)1  

- **Node Details:**

  - **Webhook1**  
    - Type: Webhook  
    - Role: Entry point; listens for HTTP POST requests at path `236f9261-dd76-47e7-b09a-2667a0f315bf`.  
    - Config: Receives JSON body containing email data including mode, from, to, subject, html content.  
    - Inputs: None  
    - Outputs: To Check Mode1  
    - Edge Cases: Invalid/malformed payload, unauthorized requests (no auth configured here).  
    - Version: 1  

  - **Check Mode1**  
    - Type: If  
    - Role: Determines if the user is "demo" (free) or "pro" based on `mode` field in payload.  
    - Config: String condition checking if `mode` contains "demo".  
    - Inputs: From Webhook1  
    - Outputs: True branch (demo) -> Google Sheets Read; False branch (pro) -> Send Email (Pro)  
    - Edge Cases: Missing or unexpected mode value may misroute flow.  
    - Version: 1  

  - **Send Email (Pro)**  
    - Type: SendGrid  
    - Role: Sends the email for pro users without limit checks.  
    - Config: Uses `from`, `to`, `subject`, `html` from webhook data; sender email hardcoded; content type HTML.  
    - Inputs: False branch from Check Mode1  
    - Outputs: Success Response (Pro)1  
    - Credentials: SendGrid API required.  
    - Edge Cases: SendGrid API failures, invalid emails, quota exceeded.  
    - Version: 1  

  - **Success Response (Pro)1**  
    - Type: Respond to Webhook  
    - Role: Sends JSON success response to pro users after email is sent.  
    - Config: Response body confirms success with mode "pro".  
    - Inputs: From Send Email (Pro) node  
    - Outputs: None (ends flow)  
    - Edge Cases: Response failures if webhook is closed.  
    - Version: 1  

---

#### 2.2 User Existence Verification

- **Overview:**  
  For demo users, reads Google Sheet to find existing user data, then uses code logic to determine if user exists or not.

- **Nodes Involved:**  
  - Google Sheets Read  
  - Check User Exists2 (Code)  
  - Check User Exists1 (If)  

- **Node Details:**

  - **Google Sheets Read**  
    - Type: Google Sheets  
    - Role: Reads rows from "Email_Tracker" sheet filtering by `Email` matching webhook `from` email.  
    - Config: Filter `Email` column equals webhook `body.from`. Sheet ID and GID specified.  
    - Inputs: True branch from Check Mode1  
    - Outputs: To Check User Exists2  
    - Credentials: Google Sheets OAuth2 required.  
    - Edge Cases: API limits, missing sheet, no matching rows.  
    - Version: 4.5  

  - **Check User Exists2**  
    - Type: Code  
    - Role: Processes Google Sheet results to determine if user exists; marks existence flag.  
    - Config: Checks for non-empty Email rows; sets `exists` boolean accordingly.  
    - Inputs: From Google Sheets Read  
    - Outputs: To Check User Exists1  
    - Edge Cases: Empty result sets, case sensitivity in email.  
    - Version: 2  

  - **Check User Exists1**  
    - Type: If  
    - Role: Branches flow based on user existence flag.  
    - Config: Boolean condition on `exists` field.  
    - Inputs: From Check User Exists2  
    - Outputs: True (user exists) -> Check Limit Logic; False (new user) -> Create New User1  
    - Edge Cases: Improper boolean evaluation, missing flag.  
    - Version: 2  

---

#### 2.3 Rate Limiting Check

- **Overview:**  
  Checks if the demo user has sent fewer than 5 emails today; resets daily counter if needed.

- **Nodes Involved:**  
  - Check Limit Logic (Code)  
  - Can Send?1 (If)  
  - Limit Reached Response1  
  - Update User Count1  
  - Create New User1  

- **Node Details:**

  - **Check Limit Logic**  
    - Type: Code  
    - Role: Implements logic to check if daily email count is below 5, resets count on new day, returns updated counters and flags.  
    - Config: Uses JavaScript Date to compare today's date with last send date; increments or resets send count accordingly; flags canSend boolean.  
    - Inputs: True branch from Check User Exists1 (user exists)  
    - Outputs: To Can Send?1  
    - Edge Cases: Date format mismatches, missing fields, parsing errors.  
    - Version: 2  

  - **Can Send?1**  
    - Type: If  
    - Role: Branches flow depending on canSend boolean from previous node.  
    - Config: Boolean condition on `$json.canSend === true`  
    - Inputs: From Check Limit Logic  
    - Outputs: True -> Update User Count1; False -> Limit Reached Response1  
    - Edge Cases: Missing or malformed canSend field.  
    - Version: 2  

  - **Limit Reached Response1**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 429 error response indicating daily free email limit reached.  
    - Config: Response includes error message, limit info, and current count.  
    - Inputs: From Can Send?1 (false branch)  
    - Outputs: None (ends flow)  
    - Edge Cases: Response delivery failures.  
    - Version: 1  

  - **Update User Count1**  
    - Type: Google Sheets  
    - Role: Updates existing user row with incremented daily and total send counts, and last send date.  
    - Config: Update by matching `Email` column; increments `Total_Sent` by 1; updates `Send_Count` and `Last_Send_Date`.  
    - Inputs: From Can Send?1 (true branch)  
    - Outputs: To Send Email (Demo)1 and Telegram Notification1  
    - Credentials: Google Sheets OAuth2 required.  
    - Edge Cases: Update conflicts, concurrency issues, API errors.  
    - Version: 4  

  - **Create New User1**  
    - Type: Google Sheets  
    - Role: Appends new user row for a previously unseen demo user with initial counts set to 1.  
    - Config: Creates row with email, `Send_Count = 1`, `Total_Sent = 1`, and current date as `Last_Send_Date`.  
    - Inputs: False branch from Check User Exists1 (new user)  
    - Outputs: To Telegram Notification1 and Send Email (Demo)1  
    - Credentials: Google Sheets OAuth2 required.  
    - Edge Cases: Append errors, duplicate entries.  
    - Version: 4  

---

#### 2.4 User Data Update and Email Sending

- **Overview:**  
  Sends the demo user email using SendGrid, then sends success response and Telegram notification.

- **Nodes Involved:**  
  - Send Email (Demo)1  
  - Success Response (Demo)1  
  - Telegram Notification1  

- **Node Details:**

  - **Send Email (Demo)1**  
    - Type: SendGrid  
    - Role: Sends the email for demo users after passing the limit checks.  
    - Config: Uses email parameters from webhook; sender name is the `from` field in webhook; content type HTML.  
    - Inputs: From Update User Count1 or Create New User1  
    - Outputs: To Success Response (Demo)1  
    - Credentials: SendGrid API required.  
    - Edge Cases: API errors, invalid email data.  
    - Version: 1  

  - **Success Response (Demo)1**  
    - Type: Respond to Webhook  
    - Role: Sends JSON success response confirming email sent for demo user, includes remaining emails count for the day.  
    - Config: Response JSON includes success flag, message, mode "demo", and remaining emails (5 - newCount).  
    - Inputs: From Send Email (Demo)1  
    - Outputs: None (ends flow)  
    - Edge Cases: Response failures.  
    - Version: 1  

  - **Telegram Notification1**  
    - Type: Telegram  
    - Role: Sends notification message to administrator Telegram chat about demo email sent and remaining quota.  
    - Config: Custom text including sender, recipient, and remaining emails; uses configured chat ID.  
    - Inputs: From Update User Count1 and Create New User1 (parallel)  
    - Outputs: None  
    - Credentials: Telegram API credentials required.  
    - Edge Cases: Telegram API errors, invalid chat ID.  
    - Version: 1.2  

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                                | Input Node(s)          | Output Node(s)                            | Sticky Note                                                                                          |
|-----------------------|--------------------|-----------------------------------------------|------------------------|-------------------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook1              | Webhook            | Receives newsletter data via HTTP POST        | None                   | Check Mode1                              | ## 1. Receive data from newsletter page, validate if free or pro user. Process free user and send pro user's message |
| Check Mode1           | If                 | Checks if user mode is demo or pro             | Webhook1               | Google Sheets Read (demo), Send Email (Pro) (pro) |                                                                                                    |
| Google Sheets Read    | Google Sheets      | Reads user data from sheet based on email      | Check Mode1 (demo)     | Check User Exists2                        |                                                                                                    |
| Check User Exists2     | Code               | Determines if user exists in Google Sheet      | Google Sheets Read     | Check User Exists1                        |                                                                                                    |
| Check User Exists1     | If                 | Branches on existence of user                   | Check User Exists2     | Check Limit Logic (exists), Create New User1 (not exists) | ## 2. Check if user has used up the 5 free email limit daily. Send error message if exceeded        |
| Check Limit Logic      | Code               | Checks and updates daily send count             | Check User Exists1     | Can Send?1                              |                                                                                                    |
| Can Send?1            | If                 | Branches on if user can send email or not       | Check Limit Logic      | Update User Count1 (can send), Limit Reached Response1 (limit reached) |                                                                                                    |
| Update User Count1     | Google Sheets      | Updates user's send counts in Google Sheet      | Can Send?1 (true)      | Send Email (Demo)1, Telegram Notification1 | ## 3. Update database, send email and notification to self                                        |
| Create New User1       | Google Sheets      | Creates new user entry in Google Sheet          | Check User Exists1 (false) | Telegram Notification1, Send Email (Demo)1 |                                                                                                    |
| Send Email (Pro)       | SendGrid           | Sends email for pro users                        | Check Mode1 (pro)      | Success Response (Pro)1                   |                                                                                                    |
| Success Response (Pro)1| Respond to Webhook | Sends success JSON response to pro user         | Send Email (Pro)       | None                                     |                                                                                                    |
| Send Email (Demo)1     | SendGrid           | Sends email for demo users                       | Update User Count1/Create New User1 | Success Response (Demo)1       |                                                                                                    |
| Success Response (Demo)1| Respond to Webhook| Sends success JSON response to demo user        | Send Email (Demo)1     | None                                     |                                                                                                    |
| Limit Reached Response1| Respond to Webhook | Sends 429 error response when limit reached     | Can Send?1 (false)     | None                                     |                                                                                                    |
| Telegram Notification1 | Telegram           | Sends Telegram admin notification                | Update User Count1, Create New User1 | None                              |                                                                                                    |
| Sticky Note            | Sticky Note        | Notes for block 2 (limit check)                  | None                   | None                                     | ## 2. Check if user has used up the 5 free email limit daily. Send error message if exceeded        |
| Sticky Note1           | Sticky Note        | Notes for block 3 (update & notification)       | None                   | None                                     | ## 3. Update database, send email and notification to self                                        |
| Sticky Note2           | Sticky Note        | Notes for block 1 (input & validation)           | None                   | None                                     | ## 1. Receive data from newsletter page, validate if free or pro user. Process free user and send pro user's message |
| Sticky Note3           | Sticky Note        | Full workflow overview and setup instructions    | None                   | None                                     | ## Email Newsletter Builder with Rate Limiting and User Tracking [Get the Email Newsletter Builder Webform](https://drive.google.com/file/d/1ZipYXImNi8JbwnekzphqHoFKf5Qbhu6g/view?usp=sharing) [Get the Google Sheet Template](https://docs.google.com/spreadsheets/d/1JvsOzkCaJzJN8-x1hldFA-H6iPc0A9MH-mVYBCEWtJw/edit?usp=sharing) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Webhook1`  
   - Type: Webhook (HTTP POST)  
   - Path: `236f9261-dd76-47e7-b09a-2667a0f315bf`  
   - Response Mode: Response Node  
   - No authentication configured here (consider adding for production)  

2. **Create If Node to Check Mode:**  
   - Name: `Check Mode1`  
   - Type: If  
   - Condition: String contains  
   - Value1: `{{$json.body.mode}}`  
   - Value2: `demo`  
   - Connect `Webhook1` main output to `Check Mode1` input.  

3. **Create Google Sheets Read Node:**  
   - Name: `Google Sheets Read`  
   - Type: Google Sheets (Read)  
   - Document ID: Your Google Sheet ID (e.g., `1OLIz-oxje6O-RjhVv0NXyNOrgd3hALTYnBKjo27sj9Q`)  
   - Sheet Name: `Sheet1` (gid=0)  
   - Filters: Filter by column `Email` equals `={{ $json.body.from }}`  
   - Credentials: Configure Google Sheets OAuth2 API  
   - Connect `Check Mode1` TRUE output (demo) to this node.  

4. **Create Code Node to Check User Exists:**  
   - Name: `Check User Exists2`  
   - Type: Code  
   - Code:  
     ```javascript
     const email = $node["Webhook1"].json.body.from.toLowerCase();
     const realRows = items.filter(item => item.json.Email && item.json.Email.trim() !== '');
     if (realRows.length > 0) {
       return realRows.map(item => ({ json: {...item.json, email, exists: true} }));
     }
     return [{ json: { email, exists: false, matches: 0 } }];
     ```  
   - Connect `Google Sheets Read` to this node.  

5. **Create If Node to Branch on User Existence:**  
   - Name: `Check User Exists1`  
   - Type: If  
   - Condition: Boolean  
   - Left Value: `={{ $json.exists }}`  
   - Operation: Equals `true`  
   - Connect `Check User Exists2` output to this node.  

6. **Create Code Node for Limit Check:**  
   - Name: `Check Limit Logic`  
   - Type: Code  
   - Code:  
     ```javascript
     const today = new Date().toISOString().split('T')[0];
     const userDate = $input.item.json.Last_Send_Date;
     const sendCount = parseInt($input.item.json.Send_Count) || 0;
     let newCount1 = sendCount;
     let newCount = $input.item.json.Total_Sent;
     let canSend = false;
     if (userDate === today) {
       if (sendCount < 5) {
         canSend = true;
         newCount1 = sendCount + 1;
       }
     } else {
       canSend = true;
       newCount1 = 1;
     }
     return {
       canSend,
       Total_Sent: newCount,
       Send_Count: newCount1,
       Email: $input.item.json.Email,
       rowNumber: $input.item.json.row_number,
       Last_Send_Date: today
     };
     ```  
   - Connect TRUE output from `Check User Exists1` to this node.  

7. **Create If Node for Can Send?:**  
   - Name: `Can Send?1`  
   - Type: If  
   - Condition: Boolean equals `true` on `{{$json.canSend}}`  
   - Connect `Check Limit Logic` output to this node.  

8. **Create Google Sheets Update Node:**  
   - Name: `Update User Count1`  
   - Type: Google Sheets (Update)  
   - Document ID and Sheet Name same as before  
   - Matching Column: `Email`  
   - Columns to update:  
     - Email: `{{$json.Email}}`  
     - Send_Count: `{{$json.Send_Count}}`  
     - Total_Sent: `{{$json.Total_Sent + 1}}`  
     - Last_Send_Date: `{{$json.Last_Send_Date}}`  
     - row_number: 0 (read-only, ignored in update)  
   - Connect TRUE output of `Can Send?1` to this node.  

9. **Create Google Sheets Append Node for New Users:**  
   - Name: `Create New User1`  
   - Type: Google Sheets (Append)  
   - Columns:  
     - Email: `{{$node["Webhook1"].item.json.body.from}}`  
     - Send_Count: 1  
     - Total_Sent: 1  
     - Last_Send_Date: `{{$now.format("yyyy-MM-dd")}}`  
   - Connect FALSE output of `Check User Exists1` here.  

10. **Create SendGrid Node to Send Demo Email:**  
    - Name: `Send Email (Demo)1`  
    - Type: SendGrid  
    - Parameters:  
      - Subject: `{{$node["Webhook1"].item.json.body.subject}}`  
      - To Email: `{{$node["Webhook1"].item.json.body.to}}`  
      - From Name: `{{$node["Webhook1"].item.json.body.from}}`  
      - From Email: Your verified sender email address  
      - Content Type: `text/html`  
      - Content Value: `{{$node["Webhook1"].item.json.body.html}}`  
    - Credentials: SendGrid API credentials required  
    - Connect outputs of `Update User Count1` and `Create New User1` to this node.  

11. **Create Success Response Node for Demo:**  
    - Name: `Success Response (Demo)1`  
    - Type: Respond to Webhook  
    - Response Mode: JSON  
    - Response Body:  
      ```json
      {
        "success": true,
        "message": "Email sent successfully!",
        "mode": "demo",
        "remaining": 5 - {{$json.Send_Count}}
      }
      ```  
    - Connect output of `Send Email (Demo)1` here.  

12. **Create Limit Reached Response Node:**  
    - Name: `Limit Reached Response1`  
    - Type: Respond to Webhook  
    - HTTP Response Code: 429  
    - Response Mode: JSON  
    - Response Body:  
      ```json
      {
        "success": false,
        "error": "Daily limit reached (5 emails/day)",
        "message": "You've reached your daily limit. Upgrade to Pro for unlimited sends!",
        "current_count": {{$json.Send_Count}}
      }
      ```  
    - Connect FALSE output of `Can Send?1` here.  

13. **Create Telegram Notification Node:**  
    - Name: `Telegram Notification1`  
    - Type: Telegram  
    - Chat ID: Your Telegram chat ID  
    - Text:  
      ```
      📧 Demo email sent from: {{$node["Webhook1"].item.json.body.from}}
      ✉️ To: {{$node["Webhook1"].item.json.body.to}}
      📊 Remaining today: {{5 - $json.Send_Count}}
      ```  
    - Credentials: Telegram Bot API credentials required  
    - Connect outputs of `Update User Count1` and `Create New User1` here.  

14. **Connect Pro User Branch:**  
    - Connect FALSE output of `Check Mode1` to `Send Email (Pro)`.  
    - Configure `Send Email (Pro)` similarly to demo email node but with fixed `From Name` "Pro User".  
    - Connect `Send Email (Pro)` output to `Success Response (Pro)1` node.  

15. **Test the workflow end-to-end** ensuring correct data flows and credentials are valid.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Email Newsletter Builder with Rate Limiting and User Tracking. Setup instructions and webform & sheet templates included. | See Sticky Note3 content inside the workflow JSON.                                                          |
| Register on SendGrid and link the API key to SendGrid nodes for email sending.                                          | https://login.sendgrid.com/login/                                                                            |
| Create a Telegram bot and get chat ID for notification node credentials.                                                | https://web.telegram.org/a/                                                                                   |
| Download the Email Newsletter Builder Webform template.                                                                 | https://drive.google.com/file/d/1ZipYXImNi8JbwnekzphqHoFKf5Qbhu6g/view?usp=sharing                            |
| Download the Google Sheet Template for tracking user email sends.                                                       | https://docs.google.com/spreadsheets/d/1JvsOzkCaJzJN8-x1hldFA-H6iPc0A9MH-mVYBCEWtJw/edit?usp=sharing          |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow and adheres strictly to content policies. It contains no illegal or offensive elements. All data handled is legal and public.