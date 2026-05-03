Send WhatsApp Bulk Messages from Google Sheets

https://n8nworkflows.xyz/workflows/send-whatsapp-bulk-messages-from-google-sheets-6237


# Send WhatsApp Bulk Messages from Google Sheets

---

### 1. Workflow Overview

This n8n workflow automates the process of sending bulk WhatsApp messages by reading recipient data and message content from a Google Sheet and sending personalized messages through the WhatsApp Business Cloud API. It tracks the message delivery status and updates the Google Sheet accordingly, enabling efficient mass communication for marketing, client updates, or notifications.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow every 5 minutes to process new messages.
- **1.2 Data Retrieval and Filtering**: Fetches rows from Google Sheets where the message status is empty (pending messages).
- **1.3 Limiting and Batching**: Limits the number of messages processed per run and splits them into manageable batches.
- **1.4 Data Cleaning and Preparation**: Sanitizes WhatsApp phone numbers to ensure correct format before sending.
- **1.5 Message Sending and Status Update**: Sends messages via WhatsApp Business API and updates the message status in Google Sheets to "Sent".
- **1.6 Documentation and Notes**: Provides detailed sticky note instructions and references for users.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution automatically every 5 minutes to process any new or pending messages.

- **Nodes Involved:**  
  - Trigger Every 5 Minute

- **Node Details:**  
  - **Trigger Every 5 Minute**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger at a fixed interval of every 5 minutes.  
    - Input: None (time-based trigger)  
    - Output: Triggers the workflow to start.  
    - Failure Modes: None typical, but if the scheduler fails or is disabled, the workflow won't run.  
    - Version: 1.2  

---

#### 1.2 Data Retrieval and Filtering

- **Overview:**  
  Retrieves all rows from the specified Google Sheet where the "Status" column is empty, indicating messages to be sent.

- **Nodes Involved:**  
  - Fetch All Pending Queries for Messaging  

- **Node Details:**  
  - **Fetch All Pending Queries for Messaging**  
    - Type: Google Sheets  
    - Configuration:  
      - Reads from a specific Google Sheet by document ID and sheet tab (gid=0).  
      - Filters rows where the "Status" column is empty (no status set).  
      - Uses OAuth2 credentials for Google Sheets authentication.  
    - Input: Trigger from the schedule node  
    - Output: Array of rows with pending messages, each containing fields like "WhatsApp No", "Name", "Message", "Image URL", and "row_number".  
    - Failure Modes: Authentication issues, Google Sheets API limits, invalid document ID or sheet name, empty result sets.  
    - Version: 4.6  

---

#### 1.3 Limiting and Batching

- **Overview:**  
  Limits the number of messages to process in one run to 300 and splits the dataset into batches for controlled processing.

- **Nodes Involved:**  
  - Limit  
  - Loop Over Items  

- **Node Details:**  
  - **Limit**  
    - Type: Limit  
    - Configuration: Limits the maximum number of items to 300 per workflow execution.  
    - Input: Rows fetched from Google Sheets  
    - Output: Subset of the input limited to 300 rows.  
    - Failure Modes: None typical, but if input is empty, downstream nodes receive no data.  
    - Version: 1  

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Configuration: Default settings to process items one by one or in small batches (not explicitly configured batch size).  
    - Input: Limited rows from the Limit node  
    - Output: Processes each item individually or in batches.  
    - Failure Modes: Incorrect batch size can impact rate limiting or cause timeouts.  
    - Version: 3  

---

#### 1.4 Data Cleaning and Preparation

- **Overview:**  
  Cleans the WhatsApp phone number by removing any non-digit characters to ensure the number is in the correct format for the WhatsApp API.

- **Nodes Involved:**  
  - Clean WhatsApp Number  

- **Node Details:**  
  - **Clean WhatsApp Number**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Iterates over each item and strips all non-digit characters from the "WhatsApp No" field.  
      - Ensures the phone number is a string containing only digits, handling undefined or null values gracefully.  
    - Input: Individual rows from the Loop Over Items node  
    - Output: Updated rows with cleaned phone numbers  
    - Expressions: Accesses `$json["WhatsApp No"]` and uses regex replacement `/\D/g` to remove non-digit characters.  
    - Failure Modes: If "WhatsApp No" is missing or invalid, the cleaned number may be empty, potentially causing message sending errors.  
    - Version: 2  

---

#### 1.5 Message Sending and Status Update

- **Overview:**  
  Sends the WhatsApp message using the WhatsApp Business Cloud API with dynamic content and updates the Google Sheet to mark messages as sent.

- **Nodes Involved:**  
  - Send Message to 100 Phone No1  
  - Change State of Rows in Sent1  

- **Node Details:**  
  - **Send Message to 100 Phone No1**  
    - Type: WhatsApp (WhatsApp Business Cloud API)  
    - Configuration:  
      - Uses a predefined message template called "n8n_broadcast_message" in English.  
      - Populates template components with dynamic data from each row:  
        - Header image URL from "Image URL" column  
        - Body text parameters with recipient's "Name" and the "Message" content  
      - Uses a specific phoneNumberId for the WhatsApp Business account.  
      - Recipient phone number is dynamically set from the cleaned "WhatsApp No" field.  
      - Authenticated with WhatsApp API credentials.  
    - Input: Cleaned phone number data from the previous node  
    - Output: API response indicating success or failure of message sending  
    - Failure Modes: Invalid phone numbers, API authentication failure, template not approved or mismatched, network timeouts, rate-limiting by the WhatsApp API.  
    - Version: 1  

  - **Change State of Rows in Sent1**  
    - Type: Google Sheets (Update operation)  
    - Configuration:  
      - Updates the "Status" column to "Sent" in the Google Sheet for the corresponding row number from the processed item.  
      - Uses the "row_number" field to match the correct row for updating.  
      - Authenticated with the same Google Sheets OAuth2 credentials.  
    - Input: Output from the Loop Over Items node (note: connected as the first output branch after the loop)  
    - Output: Confirmation of successful update to the Google Sheet  
    - Failure Modes: Incorrect row numbers, update conflicts, Google API rate limits or auth failures.  
    - Version: 4.6  

---

#### 1.6 Documentation and Notes

- **Overview:**  
  Provides comprehensive workflow documentation, requirements, usage instructions, and template guidelines embedded as a sticky note within the workflow.

- **Nodes Involved:**  
  - Sticky Note1  

- **Node Details:**  
  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content:  
      - Describes the workflow purpose and key features.  
      - Lists prerequisites such as WhatsApp Business Cloud API access and Google Sheets integration.  
      - Provides a sample Google Sheet template link.  
      - Details step-by-step configuration and WhatsApp template requirements.  
    - Input/Output: None (documentation only)  
    - Failure Modes: N/A  
    - Version: 1  

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                      | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                      |
|--------------------------------|-------------------------|------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Trigger Every 5 Minute          | Schedule Trigger        | Starts workflow every 5 minutes    | None                             | Fetch All Pending Queries for Messaging |                                                                                                                 |
| Fetch All Pending Queries for Messaging | Google Sheets          | Fetches pending messages from sheet | Trigger Every 5 Minute           | Limit                             |                                                                                                                 |
| Limit                         | Limit                   | Limits processing to 300 items     | Fetch All Pending Queries for Messaging | Loop Over Items                   |                                                                                                                 |
| Loop Over Items               | Split In Batches         | Processes items in batches          | Limit                           | Change State of Rows in Sent1, Clean WhatsApp Number |                                                                                                                 |
| Change State of Rows in Sent1  | Google Sheets           | Updates message status to "Sent"   | Loop Over Items (branch 1)       | None                             |                                                                                                                 |
| Clean WhatsApp Number          | Code                    | Cleans phone numbers                | Loop Over Items (branch 2)        | Send Message to 100 Phone No1      |                                                                                                                 |
| Send Message to 100 Phone No1  | WhatsApp                | Sends WhatsApp message              | Clean WhatsApp Number             | Loop Over Items                   |                                                                                                                 |
| Sticky Note1                  | Sticky Note             | Documentation and instructions     | None                            | None                            | See content in section 2.6. Contains full workflow overview, configuration steps, and sample sheet link.        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Node Type: Schedule Trigger  
   - Parameters: Set rule to trigger every 5 minutes  
   - Connects to: Google Sheets node "Fetch All Pending Queries for Messaging"

2. **Create a Google Sheets node "Fetch All Pending Queries for Messaging"**  
   - Node Type: Google Sheets  
   - Credentials: Set up Google Sheets OAuth2 credentials  
   - Operation: Read rows from a specific spreadsheet and sheet (provide Document ID and Sheet GID)  
   - Filters: Add filter to include only rows where "Status" column is empty  
   - Connects to: Limit node

3. **Create a Limit node**  
   - Node Type: Limit  
   - Parameters: Set max items to 300  
   - Connects to: Split In Batches node "Loop Over Items"

4. **Create Split In Batches node "Loop Over Items"**  
   - Node Type: Split In Batches  
   - Parameters: Use default batch size or set desired batch size (e.g., 1 for processing single items)  
   - Connects to two branches:  
     - Branch 1: Google Sheets update node "Change State of Rows in Sent1"  
     - Branch 2: Code node "Clean WhatsApp Number"

5. **Create Google Sheets node "Change State of Rows in Sent1"**  
   - Node Type: Google Sheets  
   - Credentials: Use same Google Sheets OAuth2 credentials  
   - Operation: Update row  
   - Configuration:  
     - Use "row_number" from the item to match the row to update  
     - Update the "Status" column to "Sent"  
     - Target the same spreadsheet and sheet as before  
   - Connects back to: None (end of that branch)

6. **Create Code node "Clean WhatsApp Number"**  
   - Node Type: Code (JavaScript)  
   - Parameters: Use the following code snippet to clean phone numbers:  
     ```js
     const items = $input.all();
     const updatedItems = items.map((item) => {
       const waNo = item?.json["WhatsApp No"];
       const waNoStr = typeof waNo === 'string' ? waNo : (waNo !== undefined && waNo !== null ? String(waNo) : "");
       const cleanedNumber = waNoStr.replace(/\D/g, "");
       item.json["WhatsApp No"] = cleanedNumber;
       return item;
     });
     return updatedItems;
     ```  
   - Connects to: WhatsApp node "Send Message to 100 Phone No1"

7. **Create WhatsApp node "Send Message to 100 Phone No1"**  
   - Node Type: WhatsApp  
   - Credentials: Configure WhatsApp Business Cloud API credentials  
   - Parameters:  
     - Use a pre-approved message template (e.g., "n8n_broadcast_message|en")  
     - Map template components:  
       - Header image: Use `{{ $json["Image URL"] }}`  
       - Body parameters: Use dynamic fields `{{ $json.Name }}` and `{{ $json.Message }}`  
     - Phone Number ID: Use your WhatsApp Business phone number ID  
     - Recipient Phone Number: Use `{{ String($json["WhatsApp No"]) }}`  
   - Connects back to: Loop Over Items (to continue batch processing)

8. **Create a Sticky Note node**  
   - Node Type: Sticky Note  
   - Content: Paste the detailed documentation, workflow overview, instructions, sample Google Sheet link, and WhatsApp template requirements as provided in the workflow description.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Sample Google Sheet Template for WhatsApp Bulk Messaging: Contains columns "WhatsApp No," "Message," and "Status" with instructions to keep the "Status" column empty for pending messages.                                                                                                                                                                                                                                | [Google Sheet Sample](https://docs.google.com/spreadsheets/d/1nI-AwIR3Y1FYzV0lwjr9iK6Ia4WuD5eRtgJH819rIIc/edit?usp=sharing) |
| WhatsApp message templates must be approved by Meta and use placeholders for dynamic content, matching exactly the template structure used in the workflow.                                                                                                                                                                                                                                                                | Meta Developer Portal (WhatsApp Business API Templates)                                                           |
| To prevent hitting WhatsApp API rate limits, consider adjusting batch sizes and adding delays between message sends as needed.                                                                                                                                                                                                                                                                                             |                                                                                                                   |
| Google Sheets API requires OAuth2 credentials with appropriate scopes to read and update the sheet content. Ensure credentials are up to date and have access permissions.                                                                                                                                                                                                                                                   |                                                                                                                   |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow built with n8n, a no-code/low-code automation platform. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All handled data are legal and publicly accessible.

---