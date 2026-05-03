Food Menu Update Notifier

https://n8nworkflows.xyz/workflows/food-menu-update-notifier-5877


# Food Menu Update Notifier

---

### 1. Workflow Overview

This workflow, titled **Food Menu Update Notifier via WhatsApp, Email & Twilio SMS**, automates the process of detecting updates in a restaurant's special menu stored in Google Sheets and notifies customers through their preferred communication channels: WhatsApp, Email, or SMS (via Twilio). It ensures timely dissemination of new, updated, or removed menu items to customers, providing real-time updates to enhance customer engagement and satisfaction.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Menu Data Retrieval**: Periodically fetches the latest special menu data from Google Sheets.
- **1.2 Menu Change Detection**: Compares the newly fetched menu data with the previous snapshot to identify changes.
- **1.3 Notification Message Generation**: Creates formatted messages summarizing menu changes for different communication platforms.
- **1.4 Customer Data Retrieval and Merge**: Fetches customer contact information and merges it with generated message data.
- **1.5 Notification Dispatch**: Splits the merged data by customer preference and sends notifications via WhatsApp, Email, or SMS.
- **1.6 Logging**: Logs each notification’s status into Google Sheets for auditing and tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Menu Data Retrieval

- **Overview**: This block triggers the workflow on a schedule (every 30 minutes) and retrieves the current special menu data from a Google Sheets document.
- **Nodes Involved**:
  - Daily Menu Update Scheduler
  - Fetch Special Menu Data
  - Detect Menu Changes

**Node Details**:

- **Daily Menu Update Scheduler**
  - **Type**: Schedule Trigger
  - **Role**: Triggers workflow every 30 minutes.
  - **Configuration**: Interval set to 30 minutes.
  - **Inputs**: None (trigger node).
  - **Outputs**: Connects to "Fetch Special Menu Data".
  - **Edge Cases**: Scheduler downtime or misconfiguration may halt the entire workflow execution.

- **Fetch Special Menu Data**
  - **Type**: Google Sheets
  - **Role**: Retrieves special menu data from the Google Sheet with document ID `185-CpBG7aVkoiq25NXjTW5gnNJC75n7w3TOBH9HpzhQ` and sheet `gid=0`.
  - **Configuration**: Uses OAuth2 credentials for Google Sheets.
  - **Inputs**: Trigger from scheduler.
  - **Outputs**: Passes data to "Detect Menu Changes".
  - **Edge Cases**: Google API quota limits, authentication errors, or sheet structure changes.

- **Detect Menu Changes**
  - **Type**: Code (JavaScript)
  - **Role**: Compares current menu data with previous snapshot stored in workflow static data to detect new, updated, or removed items.
  - **Configuration**: Uses workflow static data to store last menu snapshot.
  - **Key Expressions**: Uses JSON.stringify for deep comparison; extracts new, updated, and removed items arrays.
  - **Inputs**: Current menu data.
  - **Outputs**: Boolean flag `hasChanged` and arrays of changes.
  - **Edge Cases**: First run treats all items as new; large data sets might impact performance; improper static data handling could cause false positives.

---

#### 1.2 Notification Message Generation

- **Overview**: Formats the detected menu changes into human-readable messages tailored for WhatsApp, Email, and SMS.
- **Nodes Involved**:
  - Generate Menu Alert Message

**Node Details**:

- **Generate Menu Alert Message**
  - **Type**: Code (JavaScript)
  - **Role**: Builds notification messages with emojis and markdown for WhatsApp; plain text for email and SMS (SMS truncated to 160 characters).
  - **Configuration**: Processes arrays of new, updated, and removed items; creates separate message formats for each channel.
  - **Key Expressions**: Uses array mapping and string manipulation; removes emojis for SMS and Email.
  - **Inputs**: Output from "Detect Menu Changes".
  - **Outputs**: JSON object containing `whatsappMessage`, `emailMessage`, `smsMessage`, and flags indicating types of changes.
  - **Edge Cases**: Message length limits for SMS; emoji stripping might remove essential info; empty changes result in minimal messages.

---

#### 1.3 Customer Data Retrieval and Merge

- **Overview**: Fetches customer contact details and their notification preferences, then merges this data with the generated menu messages in preparation for sending.
- **Nodes Involved**:
  - Fetch Customer Contact List
  - Wait For All data
  - Merge Menu with Customer Data
  - Split by Notification Preference

**Node Details**:

- **Fetch Customer Contact List**
  - **Type**: Google Sheets
  - **Role**: Reads customer contact info and notification preferences from Google Sheet (`gid=1419004309`).
  - **Configuration**: Uses the same Google Sheets OAuth2 credentials.
  - **Inputs**: Output from "Generate Menu Alert Message".
  - **Outputs**: Passes data to "Wait For All data".
  - **Edge Cases**: Sheet structure changes, missing data, or API errors.

- **Wait For All data**
  - **Type**: Wait
  - **Role**: Synchronizes data flow, ensuring both menu messages and customer data are ready before merging.
  - **Inputs**: From "Fetch Customer Contact List".
  - **Outputs**: Connects to "Merge Menu with Customer Data".
  - **Edge Cases**: Potential delays if upstream nodes are slow or fail.

- **Merge Menu with Customer Data**
  - **Type**: Set
  - **Role**: Combines customer array and message data array into a single JSON object.
  - **Configuration**:
    - Assigns `customers` to all customer data.
    - Assigns `messageData` to the first item of generated messages.
  - **Inputs**: From "Wait For All data".
  - **Outputs**: Connects to "Split by Notification Preference".
  - **Edge Cases**: Large datasets could cause performance issues; data misalignment risks.

- **Split by Notification Preference**
  - **Type**: SplitInBatches
  - **Role**: Splits the merged data array into individual customer records for conditional processing.
  - **Inputs**: From "Merge Menu with Customer Data".
  - **Outputs**: Branches to filtering nodes for WhatsApp, SMS, and Email.
  - **Edge Cases**: Batch size defaults (not explicitly set) could affect throughput; large customer lists might cause delays.

---

#### 1.4 Notification Dispatch

- **Overview**: Routes customers based on their preferred notification channel and sends the formatted menu update via WhatsApp, Email, or SMS.
- **Nodes Involved**:
  - Filter WhatsApp Users
  - Send WhatsApp Notification
  - Log WhatsApp Status
  - Filter Email Users
  - Send Menu Email
  - Log Email Status1
  - Filter SMS Users
  - Send Twilio SMS Alert
  - Log SMS Status

**Node Details**:

- **Filter WhatsApp Users**
  - **Type**: If
  - **Role**: Checks if customer opted for WhatsApp notifications (`WhatsApp Notifications` field equals "Yes").
  - **Inputs**: From "Split by Notification Preference".
  - **Outputs**: True branch to "Send WhatsApp Notification".
  - **Edge Cases**: Field missing or misspelled could cause false negatives.

- **Send WhatsApp Notification**
  - **Type**: HTTP Request
  - **Role**: Sends POST request to WhatsApp API with the customer's phone number and the preformatted WhatsApp message.
  - **Configuration**:
    - URL: `https://api.whatsapp.com/send`
    - Uses Bearer token for authorization (requires setting `YOUR_WHATSAPP_API_TOKEN`).
    - Body contains `to` and `message`.
  - **Inputs**: Filtered WhatsApp users.
  - **Outputs**: Connects to "Log WhatsApp Status".
  - **Edge Cases**: API token invalid/expired, network issues, message formatting errors.

- **Log WhatsApp Status**
  - **Type**: Google Sheets Append
  - **Role**: Logs notification status with timestamp, customer name, message snippet, and notification type.
  - **Inputs**: From "Send WhatsApp Notification".
  - **Outputs**: None (end node).
  - **Edge Cases**: Google Sheets API limits or authentication errors.

- **Filter Email Users**
  - **Type**: If
  - **Role**: Checks if customer opted for Email notifications (`Email Notifications` field equals "Yes").
  - **Inputs**: From "Split by Notification Preference".
  - **Outputs**: True branch to "Send Menu Email".
  - **Edge Cases**: Similar to WhatsApp filter node.

- **Send Menu Email**
  - **Type**: Email Send
  - **Role**: Sends the email containing the menu update to all customers who opted in.
  - **Configuration**:
    - Subject: "🍽️ Special Menu Update - Your Favorite Restaurant"
    - From: `vrushti@itoneclick.com`
    - To: All customer emails concatenated.
    - Format: Plain text.
    - Uses SMTP credentials.
  - **Inputs**: Filtered email users.
  - **Outputs**: Connects to "Log Email Status1".
  - **Edge Cases**: Email server errors, invalid email addresses, SMTP credential issues.

- **Log Email Status1**
  - **Type**: Google Sheets Append
  - **Role**: Similar logging as WhatsApp but for email notifications.
  - **Inputs**: From "Send Menu Email".
  - **Outputs**: None.
  - **Edge Cases**: Google Sheets API errors.

- **Filter SMS Users**
  - **Type**: If
  - **Role**: Checks if customer opted for SMS notifications (`SMS Notifications` field equals "Yes").
  - **Inputs**: From "Split by Notification Preference".
  - **Outputs**: True branch to "Send Twilio SMS Alert".
  - **Edge Cases**: Same as other filter nodes.

- **Send Twilio SMS Alert**
  - **Type**: HTTP Request
  - **Role**: Sends SMS via Twilio REST API.
  - **Configuration**:
    - URL: Twilio Messages API endpoint with account SID.
    - Authentication: HTTP Basic Auth using Twilio credentials.
    - Body: From phone, To customer phone, Body SMS message.
  - **Inputs**: Filtered SMS users.
  - **Outputs**: Connects to "Log SMS Status".
  - **Edge Cases**: Twilio API limits, invalid credentials, phone number formatting issues.

- **Log SMS Status**
  - **Type**: Google Sheets Append
  - **Role**: Logs SMS notification status similarly.
  - **Inputs**: From "Send Twilio SMS Alert".
  - **Outputs**: None.
  - **Edge Cases**: Same as other logging nodes.

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                              | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                                         |
|----------------------------|----------------------|----------------------------------------------|----------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Daily Menu Update Scheduler | Schedule Trigger     | Triggers workflow periodically               | None                             | Fetch Special Menu Data        |                                                                                                                     |
| Fetch Special Menu Data     | Google Sheets        | Retrieves special menu data                   | Daily Menu Update Scheduler      | Detect Menu Changes            |                                                                                                                     |
| Detect Menu Changes         | Code                 | Detects changes in menu data since last check| Fetch Special Menu Data          | Generate Menu Alert Message    |                                                                                                                     |
| Generate Menu Alert Message | Code                 | Formats menu changes into notification messages| Detect Menu Changes              | Fetch Customer Contact List    |                                                                                                                     |
| Fetch Customer Contact List | Google Sheets        | Retrieves customer contact and preference data| Generate Menu Alert Message      | Wait For All data              |                                                                                                                     |
| Wait For All data           | Wait                 | Synchronizes data flow                         | Fetch Customer Contact List      | Merge Menu with Customer Data  |                                                                                                                     |
| Merge Menu with Customer Data| Set                  | Combines customer data and message data       | Wait For All data                | Split by Notification Preference|                                                                                                                     |
| Split by Notification Preference| SplitInBatches   | Splits merged data for per-customer processing| Merge Menu with Customer Data    | Filter WhatsApp Users, Filter SMS Users, Filter Email Users |                                                                                                                     |
| Filter WhatsApp Users       | If                   | Filters customers wanting WhatsApp notifications| Split by Notification Preference| Send WhatsApp Notification     |                                                                                                                     |
| Send WhatsApp Notification  | HTTP Request         | Sends WhatsApp notifications                   | Filter WhatsApp Users            | Log WhatsApp Status            |                                                                                                                     |
| Log WhatsApp Status         | Google Sheets Append | Logs WhatsApp notification status              | Send WhatsApp Notification       | None                          |                                                                                                                     |
| Filter Email Users          | If                   | Filters customers wanting Email notifications  | Split by Notification Preference| Send Menu Email                |                                                                                                                     |
| Send Menu Email             | Email Send           | Sends menu update emails                        | Filter Email Users               | Log Email Status1              |                                                                                                                     |
| Log Email Status1           | Google Sheets Append | Logs Email notification status                  | Send Menu Email                 | None                          |                                                                                                                     |
| Filter SMS Users            | If                   | Filters customers wanting SMS notifications    | Split by Notification Preference| Send Twilio SMS Alert          |                                                                                                                     |
| Send Twilio SMS Alert       | HTTP Request         | Sends SMS via Twilio                            | Filter SMS Users                | Log SMS Status                |                                                                                                                     |
| Log SMS Status              | Google Sheets Append | Logs SMS notification status                    | Send Twilio SMS Alert           | None                          |                                                                                                                     |
| Sticky Note                 | Sticky Note          | Description of workflow purpose and overview   | None                           | None                          | ## Automatically detects changes in the restaurant's special menu from Google Sheets and notifies customers via their preferred channel – WhatsApp, Email, or SMS (Twilio).\n\n## This ensures real-time updates to all customers about new or updated food offers. |
| Sticky Note1                | Sticky Note          | Notes on scheduled menu checking                | None                           | None                          | ## Automatically checks the special menu sheet on a schedule (daily/hourly) and detects any updates or changes in menu items. |
| Sticky Note2                | Sticky Note          | Notes on message generation                      | None                           | None                          | ## Creates a custom message (with updated menu info) to be sent to customers in a readable and attractive format.    |
| Sticky Note3                | Sticky Note          | Notes on customer data retrieval and merging     | None                           | None                          | ## Reads customer contact info and communication preferences (WhatsApp, Email, or SMS), then prepares data for each user. |
| Sticky Note4                | Sticky Note          | Notes on notification sending and logging        | None                           | None                          | ## Sends the menu update to customers via their selected channel and logs the status for confirmation and audit.\n\n |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add **Schedule Trigger** node named "Daily Menu Update Scheduler".
   - Set interval to every 30 minutes.

2. **Fetch Special Menu Data:**
   - Add **Google Sheets** node named "Fetch Special Menu Data".
   - Configure with OAuth2 credentials for Google Sheets.
   - Set document ID to `185-CpBG7aVkoiq25NXjTW5gnNJC75n7w3TOBH9HpzhQ`.
   - Set sheet to `gid=0` (special menu sheet).
   - Connect from "Daily Menu Update Scheduler".

3. **Detect Menu Changes:**
   - Add **Code** node named "Detect Menu Changes".
   - Use JavaScript to compare current menu data with stored snapshot in workflow static data.
   - Store current snapshot for future comparisons.
   - Connect from "Fetch Special Menu Data".

4. **Generate Menu Alert Message:**
   - Add **Code** node named "Generate Menu Alert Message".
   - Format message for WhatsApp (with emojis and markdown), Email (plain text), and SMS (plain text, max 160 chars).
   - Connect from "Detect Menu Changes".

5. **Fetch Customer Contact List:**
   - Add **Google Sheets** node named "Fetch Customer Contact List".
   - Use same Google Sheets OAuth2 credentials.
   - Set document ID to same as menu sheet.
   - Set sheet to `gid=1419004309` (customers sheet).
   - Connect from "Generate Menu Alert Message".

6. **Wait For All data:**
   - Add **Wait** node named "Wait For All data".
   - Connect from "Fetch Customer Contact List".

7. **Merge Menu with Customer Data:**
   - Add **Set** node named "Merge Menu with Customer Data".
   - Assign `customers` to all items from "Fetch Customer Contact List".
   - Assign `messageData` to first item from "Generate Menu Alert Message".
   - Connect from "Wait For All data".

8. **Split by Notification Preference:**
   - Add **SplitInBatches** node named "Split by Notification Preference".
   - Connect from "Merge Menu with Customer Data".

9. **Filter WhatsApp Users:**
   - Add **If** node named "Filter WhatsApp Users".
   - Condition: `WhatsApp Notifications` field equals "Yes".
   - Connect from "Split by Notification Preference".

10. **Send WhatsApp Notification:**
    - Add **HTTP Request** node named "Send WhatsApp Notification".
    - POST to `https://api.whatsapp.com/send`.
    - Headers include `Authorization: Bearer YOUR_WHATSAPP_API_TOKEN` and `Content-Type: application/json`.
    - Body parameters: `to` from customer WhatsApp number, `message` from merged message data.
    - Connect true branch from "Filter WhatsApp Users".

11. **Log WhatsApp Status:**
    - Add **Google Sheets Append** node named "Log WhatsApp Status".
    - Append to notification logs sheet (`gid=1731532192`).
    - Columns: Status="Sent", Message snippet, Timestamp, Customer Name, Notification Type="WhatsApp".
    - Connect from "Send WhatsApp Notification".

12. **Filter Email Users:**
    - Add **If** node named "Filter Email Users".
    - Condition: `Email Notifications` field equals "Yes".
    - Connect from "Split by Notification Preference".

13. **Send Menu Email:**
    - Add **Email Send** node named "Send Menu Email".
    - Subject: "🍽️ Special Menu Update - Your Favorite Restaurant".
    - From: `vrushti@itoneclick.com`.
    - To: concatenate emails of all customers filtered.
    - Text content from generated email message.
    - Use SMTP credentials.
    - Connect true branch from "Filter Email Users".

14. **Log Email Status1:**
    - Add **Google Sheets Append** node named "Log Email Status1".
    - Append similar log entries as WhatsApp but for email.
    - Connect from "Send Menu Email".

15. **Filter SMS Users:**
    - Add **If** node named "Filter SMS Users".
    - Condition: `SMS Notifications` field equals "Yes".
    - Connect from "Split by Notification Preference".

16. **Send Twilio SMS Alert:**
    - Add **HTTP Request** node named "Send Twilio SMS Alert".
    - POST to Twilio Messages API (replace placeholders with your Account SID and phone number).
    - Use HTTP Basic Auth with Twilio credentials.
    - Body parameters: From (your Twilio number), To (customer phone), Body (SMS message).
    - Connect true branch from "Filter SMS Users".

17. **Log SMS Status:**
    - Add **Google Sheets Append** node named "Log SMS Status".
    - Append notification logs for SMS.
    - Connect from "Send Twilio SMS Alert".

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                                     |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow automatically detects menu changes and notifies customers on preferred channels.| Sticky Note at workflow start describing purpose.                                                                  |
| Scheduled checks ensure restaurant menu updates are timely communicated.                       | Sticky Note1: Scheduled data retrieval explanation.                                                                |
| Custom messages are tailored for each notification channel to enhance readability and engagement.| Sticky Note2: Message formatting explanation.                                                                       |
| Customer preferences are respected by filtering notifications accordingly.                     | Sticky Note3: Customer data and preference handling.                                                                |
| Notification status logging in Google Sheets enables audit trails and confirmation.            | Sticky Note4: Notification sending and logging notes.                                                               |
| WhatsApp API requires valid Bearer token; Twilio requires Account SID and Auth credentials.   | Credentials must be securely configured for these third-party services.                                            |
| SMS messages are truncated to 160 characters to comply with SMS length limits.                 | Managed in message generation node.                                                                                  |
| Google Sheets document and sheet IDs are hardcoded and must be consistent with actual documents.| Update accordingly if used in different environments or documents.                                                 |
| Email sender address is `vrushti@itoneclick.com`; change to valid sender as needed.            | SMTP credentials must correspond with sender domain.                                                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting current content policies and containing no illegal, offensive, or protected content. All data handled is legal and public.

---