Send Bulk WhatsApp Messages from Google Sheets using WasenderAPI

https://n8nworkflows.xyz/workflows/send-bulk-whatsapp-messages-from-google-sheets-using-wasenderapi-6175


# Send Bulk WhatsApp Messages from Google Sheets using WasenderAPI

### 1. Workflow Overview

This n8n workflow automates sending bulk WhatsApp messages using data stored in a Google Sheet, leveraging the unofficial WasenderAPI service to avoid the higher costs associated with the official WhatsApp Business API. It is designed for users who want to send personalized WhatsApp messages from their own WhatsApp number, with the process fully automated and status updates reflected back into the Google Sheet.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Periodically triggers the workflow every 5 minutes to check for new messages to send.
- **1.2 Data Fetching:** Reads "pending" message entries from a specified Google Sheet.
- **1.3 Limiting:** Restricts the number of messages processed per execution to avoid overload.
- **1.4 Batch Processing Loop:** Processes each message individually in batches.
- **1.5 Message Sending:** Sends WhatsApp messages via WasenderAPI using HTTP requests.
- **1.6 Status Update:** Updates the Google Sheet marking messages as "sent" after successful transmission.
- **1.7 Rate Limiting / Delay:** Introduces a waiting period between message sends to prevent API rate limiting or blocking.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Initiates the workflow automatically every 5 minutes to check for pending messages.
- **Nodes Involved:** `Trigger Every 5 Minute`
- **Node Details:**

  - **Trigger Every 5 Minute**
    - Type: Schedule Trigger
    - Role: Starts the workflow on a recurring schedule every 5 minutes.
    - Configuration: Interval set to every 5 minutes.
    - Inputs: None (trigger node).
    - Outputs: Connects to `Fetch All Pending Queries for Messaging`.
    - Edge Cases: Potential delay if n8n server is down or overloaded, triggering might be missed.
    - Version Requirements: n8n version supporting scheduleTrigger v1.2 or higher.

---

#### 2.2 Data Fetching from Google Sheets

- **Overview:** Retrieves all rows from the Google Sheet where the `Status` column equals "pending", representing messages queued to be sent.
- **Nodes Involved:** `Fetch All Pending Queries for Messaging`
- **Node Details:**

  - **Fetch All Pending Queries for Messaging**
    - Type: Google Sheets
    - Role: Reads rows filtered by `Status = pending` from a specified Google Sheet and worksheet.
    - Configuration:
      - Document ID set to a specific Google Sheet containing the message queue.
      - Sheet Name corresponds to the worksheet with pending messages.
      - Filter applied to only fetch rows where `Status` column equals "pending".
    - Inputs: Triggered by `Trigger Every 5 Minute`.
    - Outputs: Passes data to the `Limit` node.
    - Credentials: Uses OAuth2 credentials for Google Sheets API.
    - Edge Cases:
      - Google API quota limits or authentication issues.
      - No rows matching filter (empty output).
      - Network errors or permission errors.
    - Version Requirements: Google Sheets node v4.6.

---

#### 2.3 Limiting Number of Messages Processed

- **Overview:** Restricts the maximum number of messages to process per execution cycle to 60 to manage load and rate limiting.
- **Nodes Involved:** `Limit`
- **Node Details:**

  - **Limit**
    - Type: Limit
    - Role: Limits the number of items passing through to downstream nodes to a maximum of 60.
    - Configuration: `maxItems` set to 60.
    - Inputs: Receives all pending rows from `Fetch All Pending Queries for Messaging`.
    - Outputs: Passes limited rows to `Loop Over Items`.
    - Edge Cases: If more than 60 pending messages exist, remainder will be processed in subsequent cycles.
    - Version Requirements: Limit node v1.

---

#### 2.4 Batch Processing Loop

- **Overview:** Processes each message individually by splitting the items into batches (effectively one by one) for sequential handling.
- **Nodes Involved:** `Loop Over Items`
- **Node Details:**

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Iterates over each message item, allowing sequential processing.
    - Configuration: Default batch size (1 item per batch) to enable per-message operations.
    - Inputs: Receives up to 60 items from `Limit`.
    - Outputs: Connects to both `Change State of Rows in Sent` and `Send Message Using HTTP Request` nodes.
    - Edge Cases:
      - If no items received, downstream nodes do not execute.
      - Potential failure if batch size is not appropriate (default is 1 here).
    - Version Requirements: SplitInBatches node v3.

---

#### 2.5 Sending WhatsApp Messages via WasenderAPI

- **Overview:** Sends each WhatsApp message using the WasenderAPI unofficial API by making an authenticated HTTP POST request with message details.
- **Nodes Involved:** `Send Message Using HTTP Request`
- **Node Details:**

  - **Send Message Using HTTP Request**
    - Type: HTTP Request
    - Role: Sends WhatsApp messages through WasenderAPI.
    - Configuration:
      - Method: POST
      - URL: `https://www.wasenderapi.com/api/send-message`
      - Authentication: HTTP Bearer token (API key) stored in WasenderAPI credential.
      - Query parameters sent:
        - `to`: phone number from Google Sheet's `WhatsApp No` field.
        - `text`: message content from Google Sheet's `Message` field.
        - `imageUrl`: optional image URL from Google Sheet's `Image URL` field.
      - Headers: Implicit Content-Type application/json.
    - Inputs: Receives one message item from `Loop Over Items`.
    - Outputs: Passes to `Wait` node after message send.
    - Credentials: Requires WasenderAPI HTTP Bearer token.
    - Edge Cases:
      - API key invalid or expired (authentication failure).
      - Network timeouts or HTTP errors.
      - Invalid phone numbers causing message rejection.
      - API rate limiting or temporary blocking.
    - Version Requirements: HTTP Request node v4.2.

---

#### 2.6 Waiting Between Sends (Rate Limiting)

- **Overview:** Introduces a delay after sending each message to avoid exceeding API rate limits and reduce the chance of being blocked.
- **Nodes Involved:** `Wait`
- **Node Details:**

  - **Wait**
    - Type: Wait
    - Role: Pauses the workflow for a configured time (default not explicitly stated, typically a few seconds).
    - Configuration: Default wait time (not explicitly shown in JSON, assumed a short delay).
    - Inputs: Receives output from `Send Message Using HTTP Request`.
    - Outputs: Loops back to `Loop Over Items` for next message batch.
    - Edge Cases:
      - If wait time is too short, risk of API throttling.
      - If wait node fails or stops, may send messages too quickly.
    - Version Requirements: Wait node v1.1.

---

#### 2.7 Updating Message Status in Google Sheets

- **Overview:** After a message is sent, updates the corresponding row in the Google Sheet changing the `Status` column from "pending" to "sent" to avoid duplicate sending.
- **Nodes Involved:** `Change State of Rows in Sent`
- **Node Details:**

  - **Change State of Rows in Sent**
    - Type: Google Sheets
    - Role: Updates the `Status` field of the processed row to "sent" based on the row number.
    - Configuration:
      - Document ID and Sheet Name same as the fetching node.
      - Operation: Update.
      - Matching column: `row_number` to identify the exact row.
      - Updates the `Status` column to "sent".
    - Inputs: Receives batch item from `Loop Over Items`.
    - Outputs: None (end of update process).
    - Credentials: Uses same Google Sheets OAuth2 credentials.
    - Edge Cases:
      - Update failure due to connectivity or permission issues.
      - Race condition if multiple workflows update same sheet simultaneously.
    - Version Requirements: Google Sheets node v4.6.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                      | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                              |
|-----------------------------------|---------------------|------------------------------------|----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------|
| Trigger Every 5 Minute             | Schedule Trigger    | Starts the workflow every 5 minutes| None                             | Fetch All Pending Queries for Messaging |                                                                                                        |
| Fetch All Pending Queries for Messaging | Google Sheets       | Fetches rows with Status = pending | Trigger Every 5 Minute           | Limit                               |                                                                                                        |
| Limit                             | Limit               | Limits number of rows to 60         | Fetch All Pending Queries for Messaging | Loop Over Items                    |                                                                                                        |
| Loop Over Items                   | SplitInBatches      | Processes messages one by one       | Limit                           | Change State of Rows in Sent, Send Message Using HTTP Request |                                                                                                        |
| Change State of Rows in Sent      | Google Sheets       | Updates Status to "sent" in sheet   | Loop Over Items                 | None                                |                                                                                                        |
| Send Message Using HTTP Request   | HTTP Request        | Sends WhatsApp message via WasenderAPI | Loop Over Items                 | Wait                                |                                                                                                        |
| Wait                             | Wait                | Delays between sending messages     | Send Message Using HTTP Request | Loop Over Items                     |                                                                                                        |
| Sticky Note1                     | Sticky Note         | Documentation and setup instructions| None                            | None                               | ## 📘 n8n Workflow: WhatsApp Bulk Messaging via Unofficial API<br>Overview, prerequisites, setup instructions, and Google Sheet format. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger Node**
   - Add a **Schedule Trigger** node.
   - Set it to trigger every 5 minutes.
   - Name it `Trigger Every 5 Minute`.

2. **Add the Google Sheets Node to Fetch Pending Messages**
   - Add a **Google Sheets** node.
   - Authenticate with your Google Sheets OAuth2 credentials.
   - Set the operation to **Read Rows**.
   - Select the target Google Sheet document by its ID.
   - Specify the worksheet (sheet name) where messages are stored.
   - Add a filter with `Status` column equal to `"pending"`.
   - Name the node `Fetch All Pending Queries for Messaging`.
   - Connect the output of `Trigger Every 5 Minute` to this node.

3. **Add a Limit Node**
   - Add a **Limit** node.
   - Set `maxItems` to 60 (to process max 60 messages per run).
   - Connect the output of `Fetch All Pending Queries for Messaging` to this node.
   - Name it `Limit`.

4. **Add a SplitInBatches Node for Looping Over Messages**
   - Add a **SplitInBatches** node.
   - Default batch size is 1 (process one message at a time).
   - Connect the output of `Limit` to this node.
   - Name it `Loop Over Items`.

5. **Add the HTTP Request Node to Send WhatsApp Messages**
   - Add an **HTTP Request** node.
   - Set method to `POST`.
   - Set URL to `https://www.wasenderapi.com/api/send-message`.
   - Authentication: Select HTTP Bearer Auth.
   - Configure Bearer Token credential with your WasenderAPI API key.
   - Set query parameters:
     - `to`: expression `{{$json["WhatsApp No"]}}`
     - `text`: expression `{{$json["Message"]}}`
     - `imageUrl`: expression `{{$json["Image URL"]}}` (optional)
   - Connect one output of `Loop Over Items` to this node.
   - Name it `Send Message Using HTTP Request`.

6. **Add the Wait Node for Rate Limiting**
   - Add a **Wait** node.
   - Configure wait time (e.g., 5 seconds recommended to avoid triggering API limits).
   - Connect the output of `Send Message Using HTTP Request` to this node.
   - Connect the output of `Wait` back to `Loop Over Items` to continue processing next batch.
   - Name it `Wait`.

7. **Add the Google Sheets Node to Update Message Status**
   - Add another **Google Sheets** node.
   - Operation: Update.
   - Use same Google Sheets OAuth2 credentials.
   - Select the same document and worksheet.
   - Map columns for update: set `Status` to `"sent"`.
   - Use `row_number` field from batch item to identify row.
   - Connect the second output of `Loop Over Items` to this node.
   - Name it `Change State of Rows in Sent`.

8. **Connect All Nodes as per Logic**
   - `Trigger Every 5 Minute` → `Fetch All Pending Queries for Messaging` → `Limit` → `Loop Over Items`.
   - `Loop Over Items` has two outputs:
     - One to `Change State of Rows in Sent`.
     - One to `Send Message Using HTTP Request` → `Wait` → back to `Loop Over Items`.

9. **Verify Credentials**
   - Ensure Google Sheets OAuth2 credentials have read/write access.
   - Set up WasenderAPI credentials with a valid API key for HTTP Bearer Authentication.

10. **Test Workflow**
    - Ensure your Google Sheet is properly formatted with columns:
      - `WhatsApp No` (with country code)
      - `Message`
      - `Image URL` (optional)
      - `Status` (initially set to `pending`)
    - Run the workflow manually or wait for scheduled trigger and monitor logs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow enables sending unlimited WhatsApp messages from your own WhatsApp number, avoiding expensive official WhatsApp Business API fees. Requires a WasenderAPI subscription (~$6/month).                                                                                                                                                                                                                                                                                                                                    | https://www.wasenderapi.com/                                                                              |
| The Google Sheet should have a column `Status` with values `pending` for unsent messages and `sent` for messages already processed.                                                                                                                                                                                                                                                                                                                                                                                              | Sample Sheet: https://docs.google.com/spreadsheets/d/1Ui4TzzI-Gq-bsEsrZELwW1Kyddw0IU9L1wxlHikktqw/edit?usp=sharing |
| Recommended to process messages in batches with a wait period (e.g., 5 seconds) to avoid API rate limiting and potential blocking.                                                                                                                                                                                                                                                                                                                                                                                                |                                                                                                         |
| Use the Google Sheets OAuth2 credential properly configured in n8n to ensure read and write access.                                                                                                                                                                                                                                                                                                                                                                                                                               |                                                                                                         |
| The workflow can be extended or modified to include media attachments (via `Image URL`), personalized messages, or error handling nodes for retries.                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                         |

---

*Disclaimer: The provided text is generated from an n8n workflow automation and complies with applicable content policies. All data handled is legal and publicly accessible.*