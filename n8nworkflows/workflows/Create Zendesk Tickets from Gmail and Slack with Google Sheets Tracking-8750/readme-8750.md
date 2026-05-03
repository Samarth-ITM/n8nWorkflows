Create Zendesk Tickets from Gmail and Slack with Google Sheets Tracking

https://n8nworkflows.xyz/workflows/create-zendesk-tickets-from-gmail-and-slack-with-google-sheets-tracking-8750


# Create Zendesk Tickets from Gmail and Slack with Google Sheets Tracking

### 1. Workflow Overview

This workflow automates the creation of Zendesk support tickets by capturing input from two sources: Gmail emails and Slack messages. It consolidates and normalizes data from these channels, creates Zendesk tickets accordingly, and logs the ticket details into a Google Sheet for tracking purposes. It also provides notifications in Slack regarding the success or failure of ticket creation.

Logical blocks:

- **1.1 Input Reception:** Trigger nodes for Gmail and Slack inputs.
- **1.2 Data Normalization:** Separate normalization for Gmail and Slack data to unify structure.
- **1.3 Data Merging:** Merging normalized data streams into a single flow.
- **1.4 Zendesk Ticket Creation:** Creating tickets in Zendesk based on merged data.
- **1.5 Google Sheets Logging:** Formatting and logging ticket data into Google Sheets.
- **1.6 Error Handling and Notifications:** Checking for errors and sending Slack notifications for success or failure.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for new emails in Gmail and new messages in Slack to serve as triggers for ticket creation.
- **Nodes Involved:**  
  - Gmail Trigger  
  - Slack Trigger

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node listening for incoming Gmail messages.  
    - Configuration: Default Gmail account set up with OAuth2 credentials; triggers on new email arrival.  
    - Inputs: None (trigger node).  
    - Outputs: Raw Gmail message data forwarded to "Normalize Gmail Data".  
    - Failure modes: Authentication failure, Gmail API quota exceeded, connectivity issues.  
    - Sticky note: Present but empty.

  - **Slack Trigger**  
    - Type: Trigger node for Slack, listening to a configured webhook.  
    - Configuration: Uses a Slack app with webhook ID "slack-support-webhook". Triggers on new Slack messages/events.  
    - Inputs: None (trigger node).  
    - Outputs: Raw Slack event data forwarded to "Normalize Slack Data".  
    - Failure modes: Slack app revoked, webhook invalid, connectivity issues.  
    - Sticky note: Present but empty.

#### 1.2 Data Normalization

- **Overview:** Processes raw data from Gmail and Slack to normalize into a uniform format for downstream processing.
- **Nodes Involved:**  
  - Normalize Gmail Data  
  - Normalize Slack Data

- **Node Details:**

  - **Normalize Gmail Data**  
    - Type: Code node (JavaScript) to parse Gmail message fields.  
    - Configuration: Extracts relevant data such as sender, subject, body, date; restructures into a common schema.  
    - Inputs: Data from "Gmail Trigger".  
    - Outputs: Normalized Gmail data forwarded to "Merge Channels".  
    - Expressions: Uses JSON parsing and field extraction.  
    - Failure modes: Unexpected email structure, missing fields, code errors.  
    - Sticky note: Present but empty.

  - **Normalize Slack Data**  
    - Type: Code node (JavaScript) to parse Slack message payload.  
    - Configuration: Extracts user info, message text, timestamp, channel, and formats into the common schema.  
    - Inputs: Data from "Slack Trigger".  
    - Outputs: Normalized Slack data forwarded to "Merge Channels".  
    - Expressions: JSON parsing and field mapping.  
    - Failure modes: Slack payload changes, missing fields, code errors.  
    - Sticky note: Present but empty.

#### 1.3 Data Merging

- **Overview:** Combines normalized inputs from Gmail and Slack into a single unified flow for ticket creation.
- **Nodes Involved:**  
  - Merge Channels

- **Node Details:**

  - **Merge Channels**  
    - Type: Merge node with default settings (likely "Wait for all" or "Merge by index").  
    - Configuration: Merges two input streams — one from Gmail normalization, one from Slack normalization.  
    - Inputs: Two inputs — "Normalize Gmail Data" (input 0), "Normalize Slack Data" (input 1).  
    - Outputs: Merged data sent to "Create Zendesk Ticket".  
    - Failure modes: If one input stream fails or delays, merge may stall; data inconsistency if inputs arrive at different rates.  
    - Sticky note: Present but empty.

#### 1.4 Zendesk Ticket Creation

- **Overview:** Uses the merged normalized data to create tickets in Zendesk.
- **Nodes Involved:**  
  - Create Zendesk Ticket

- **Node Details:**

  - **Create Zendesk Ticket**  
    - Type: Zendesk node configured to create a new ticket.  
    - Configuration: Uses Zendesk credentials (API token or OAuth2). Ticket fields mapped from merged data (e.g., requester email, subject, description).  
    - Inputs: From "Merge Channels".  
    - Outputs: Zendesk API response forwarded to "Format Sheet Data".  
    - Failure modes: Authentication errors, API rate limits, invalid ticket data, network errors.  
    - Sticky note: Present but empty.

#### 1.5 Google Sheets Logging

- **Overview:** Formats the Zendesk ticket data and logs it into a Google Sheet to track created tickets.
- **Nodes Involved:**  
  - Format Sheet Data  
  - Log to Google Sheets

- **Node Details:**

  - **Format Sheet Data**  
    - Type: Code node to transform Zendesk ticket data into a row format suitable for Google Sheets.  
    - Configuration: Extracts ticket ID, status, requester, timestamp, and other relevant info.  
    - Inputs: From "Create Zendesk Ticket".  
    - Outputs: Formatted data to "Log to Google Sheets".  
    - Failure modes: Data transformation errors, missing fields in Zendesk response.  
    - Sticky note: Present but empty.

  - **Log to Google Sheets**  
    - Type: Google Sheets node to append a row.  
    - Configuration: Uses Google Sheets OAuth2 credentials; targets a specific spreadsheet and worksheet for ticket logging. Appends formatted data as a new row.  
    - Inputs: From "Format Sheet Data".  
    - Outputs: Response sent to "Check for Errors".  
    - Failure modes: Permission errors, quota limits, sheet not found, network issues.  
    - Sticky note: Present but empty.

#### 1.6 Error Handling and Notifications

- **Overview:** Checks for errors after logging and sends Slack notifications indicating success or failure.
- **Nodes Involved:**  
  - Check for Errors  
  - Send Error Notification  
  - Send Success Notification

- **Node Details:**

  - **Check for Errors**  
    - Type: If node evaluating success or failure conditions based on previous node outputs.  
    - Configuration: Checks for error flags or status codes in "Log to Google Sheets" response or prior steps.  
    - Inputs: From "Log to Google Sheets".  
    - Outputs:  
      - First output (true branch): to "Send Error Notification"  
      - Second output (false branch): to "Send Success Notification"  
    - Failure modes: Incorrect error condition logic, unhandled exceptions.  
    - Sticky note: Present but empty.

  - **Send Error Notification**  
    - Type: Slack node posting to a Slack webhook.  
    - Configuration: Uses Slack OAuth2 or webhook URL; sends a message to a designated channel detailing the error encountered.  
    - Inputs: From "Check for Errors" (true branch).  
    - Outputs: None (terminal node).  
    - Failure modes: Slack webhook invalid, message formatting issues.  
    - Sticky note: Present but empty.

  - **Send Success Notification**  
    - Type: Slack node posting a success message.  
    - Configuration: Similar to error notification but sends confirmation of ticket creation and logging.  
    - Inputs: From "Check for Errors" (false branch).  
    - Outputs: None (terminal node).  
    - Failure modes: Slack webhook invalid, message formatting issues.  
    - Sticky note: Present but empty.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)                | Output Node(s)             | Sticky Note |
|-------------------------|---------------------|----------------------------------------|-----------------------------|----------------------------|-------------|
| Gmail Trigger           | gmailTrigger        | Trigger on new Gmail emails             | None                        | Normalize Gmail Data        |             |
| Slack Trigger           | slackTrigger        | Trigger on new Slack messages            | None                        | Normalize Slack Data        |             |
| Normalize Gmail Data    | code                | Normalize Gmail email data               | Gmail Trigger               | Merge Channels             |             |
| Normalize Slack Data    | code                | Normalize Slack message data             | Slack Trigger               | Merge Channels             |             |
| Merge Channels          | merge               | Merge normalized data streams            | Normalize Gmail Data, Normalize Slack Data | Create Zendesk Ticket      |             |
| Create Zendesk Ticket   | zendesk             | Create ticket in Zendesk from merged data | Merge Channels              | Format Sheet Data          |             |
| Format Sheet Data       | code                | Format Zendesk data for Google Sheets    | Create Zendesk Ticket       | Log to Google Sheets       |             |
| Log to Google Sheets    | googleSheets        | Log ticket data in Google Sheets          | Format Sheet Data           | Check for Errors           |             |
| Check for Errors        | if                  | Evaluate if any error occurred            | Log to Google Sheets        | Send Error Notification, Send Success Notification |             |
| Send Error Notification | slack               | Send error message to Slack channel       | Check for Errors (true)     | None                      |             |
| Send Success Notification| slack              | Send success message to Slack channel     | Check for Errors (false)    | None                      |             |
| Sticky Note - Gmail Trigger | stickyNote       | -                                        | -                           | -                          |             |
| Sticky Note - Slack Trigger | stickyNote       | -                                        | -                           | -                          |             |
| Sticky Note - Gmail Normalize | stickyNote     | -                                        | -                           | -                          |             |
| Sticky Note - Slack Normalize | stickyNote     | -                                        | -                           | -                          |             |
| Sticky Note - Merge     | stickyNote          | -                                        | -                           | -                          |             |
| Sticky Note - Zendesk   | stickyNote          | -                                        | -                           | -                          |             |
| Sticky Note - Format    | stickyNote          | -                                        | -                           | -                          |             |
| Sticky Note - Sheets    | stickyNote          | -                                        | -                           | -                          |             |
| Sticky Note - Error Check| stickyNote         | -                                        | -                           | -                          |             |
| Sticky Note - Error Notify| stickyNote        | -                                        | -                           | -                          |             |
| Sticky Note - Success Notify| stickyNote       | -                                        | -                           | -                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Configure with Gmail OAuth2 credentials  
   - Set to trigger on new email reception.

2. **Create the Slack Trigger node:**  
   - Type: Slack Trigger  
   - Configure with Slack app webhook ID "slack-support-webhook"  
   - Set to trigger on new Slack messages.

3. **Create Normalize Gmail Data node:**  
   - Type: Code (JavaScript)  
   - Input: Connect from Gmail Trigger  
   - Code logic: Extract sender email, subject, body text, timestamp from Gmail data; reformat into unified JSON structure.  
   - Output: Normalized data to "Merge Channels".

4. **Create Normalize Slack Data node:**  
   - Type: Code (JavaScript)  
   - Input: Connect from Slack Trigger  
   - Code logic: Extract user, message text, timestamp, and channel from Slack payload; reformat into unified JSON structure.  
   - Output: Normalized data to "Merge Channels".

5. **Create Merge Channels node:**  
   - Type: Merge  
   - Inputs: Connect normalized Gmail data (input 0) and normalized Slack data (input 1)  
   - Configuration: Use default merge mode (e.g., "Merge by index" or "Wait for all inputs").  
   - Output: Forward merged data to "Create Zendesk Ticket".

6. **Create Create Zendesk Ticket node:**  
   - Type: Zendesk node  
   - Configure Zendesk credentials (API token or OAuth2)  
   - Map fields from merged input to Zendesk ticket fields: requester email, subject, description, etc.  
   - Output: Send Zendesk response to "Format Sheet Data".

7. **Create Format Sheet Data node:**  
   - Type: Code (JavaScript)  
   - Input: Connect from "Create Zendesk Ticket"  
   - Code logic: Extract ticket ID, status, requester, timestamps; format as array or object suitable for Google Sheets row append.  
   - Output: Forward formatted data to "Log to Google Sheets".

8. **Create Log to Google Sheets node:**  
   - Type: Google Sheets  
   - Configure Google Sheets OAuth2 credentials  
   - Select target spreadsheet and worksheet  
   - Operation: Append row  
   - Input: Connect from "Format Sheet Data"  
   - Output: Forward response to "Check for Errors".

9. **Create Check for Errors node:**  
   - Type: If node  
   - Input: Connect from "Log to Google Sheets"  
   - Condition: Check if the previous node's output contains errors or unsuccessful status (e.g., status code not 200 or error flag present)  
   - Output:  
     - If error (true) → "Send Error Notification"  
     - If no error (false) → "Send Success Notification"

10. **Create Send Error Notification node:**  
    - Type: Slack  
    - Configure Slack webhook or OAuth2 credentials  
    - Message: Compose an error notification including relevant details from the workflow run  
    - Input: Connect from "Check for Errors" (true branch)

11. **Create Send Success Notification node:**  
    - Type: Slack  
    - Configure Slack webhook or OAuth2 credentials  
    - Message: Compose a success notification confirming ticket creation and logging  
    - Input: Connect from "Check for Errors" (false branch)

12. **Connect all nodes in the order described.**

13. **Optionally add Sticky Notes** for documentation near each node for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                               |
|----------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow requires valid OAuth2 credentials for Gmail, Slack, Zendesk, and Google Sheets.| Credential setup in n8n for each service.    |
| Ensure Slack app has appropriate permissions and the webhook is correctly configured.        | Slack app & Webhook setup documentation.     |
| Google Sheets API must be enabled in the Google Cloud Console for the associated project.    | Google Sheets API setup guide.                 |
| Zendesk API token or OAuth2 must have permissions to create tickets in the target Zendesk account.| Zendesk developer portal.                    |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.