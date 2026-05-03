Automated Shopify Abandoned Cart Recovery with WhatsApp Messages & Google Sheets

https://n8nworkflows.xyz/workflows/automated-shopify-abandoned-cart-recovery-with-whatsapp-messages---google-sheets-11129


# Automated Shopify Abandoned Cart Recovery with WhatsApp Messages & Google Sheets

### 1. Workflow Overview

This workflow automates the recovery of abandoned Shopify carts by sending WhatsApp messages to customers and logging interactions in Google Sheets. It periodically retrieves abandoned checkout data from Shopify, verifies customer phone numbers via the Rapiwa API, sends notification messages through WhatsApp, and tracks the status of communications.

**Target Use Cases:**  
- Shopify store owners aiming to recover sales from abandoned carts.  
- Automated customer engagement via WhatsApp messaging.  
- Centralized tracking of contact attempts in Google Sheets.

**Logical Blocks:**  
- **1.1 Scheduling & Data Retrieval:** Periodically trigger the workflow and fetch abandoned checkout data from Shopify.  
- **1.2 Data Preparation & Customer Info Retrieval:** Process retrieved data batch-wise and enrich it with customer details.  
- **1.3 Phone Number Verification:** Verify customer phone numbers using the Rapiwa API.  
- **1.4 Messaging & Status Handling:** Conditionally send WhatsApp messages based on verification results and update Google Sheets accordingly.  
- **1.5 Wait State:** Introduce delay after processing to manage workflow pacing.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Data Retrieval

**Overview:**  
This block triggers the workflow on a schedule and fetches abandoned checkout orders from Shopify. It retrieves both initial and all abandoned checkouts to ensure comprehensive data.

**Nodes Involved:**  
- Schedule Trigger  
- Get Initial Abandoned Checkout  
- Get All Abandoned checkouts  
- Split Out1  
- Split Out

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow periodically (default schedule parameters)  
  - Inputs: None  
  - Outputs: Triggers "Get Initial Abandoned Checkout" node  
  - Failure Modes: Misconfiguration of schedule, trigger not firing  
  - Version: 1.2  

- **Get Initial Abandoned Checkout**  
  - Type: HTTP Request  
  - Role: Fetches initial batch of abandoned checkouts from Shopify API  
  - Config: Uses Shopify API credentials, likely GET request to abandoned checkouts endpoint  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Feeds "Split Out1"  
  - Failure Modes: API rate limits, authentication errors, network timeouts  
  - Version: 4.2  

- **Split Out1**  
  - Type: Split Out  
  - Role: Splits the initial abandoned checkout data to individual items for batch processing  
  - Inputs: From "Get Initial Abandoned Checkout"  
  - Outputs: Passes to "Loop Over Items1"  
  - Failure Modes: Empty data input causing no downstream processing  
  - Version: 1  

- **Get All Abandoned checkouts**  
  - Type: HTTP Request  
  - Role: Fetches all abandoned checkout data from Shopify, possibly for a separate or additional dataset  
  - Inputs: Not directly triggered here, suggests parallel or subsequent use (node is defined but no direct input connection in JSON)  
  - Outputs: Feeds "Split Out"  
  - Failure Modes: Similar to "Get Initial Abandoned Checkout"  
  - Version: 4.2  

- **Split Out**  
  - Type: Split Out  
  - Role: Splits all abandoned checkout data into individual items for batch processing  
  - Inputs: From "Get All Abandoned checkouts"  
  - Outputs: Feeds "Loop Over Items"  
  - Failure Modes: Empty input data  
  - Version: 1  

#### 1.2 Data Preparation & Customer Info Retrieval

**Overview:**  
This block processes each abandoned checkout item in batches and enriches them by retrieving detailed customer information.

**Nodes Involved:**  
- Loop Over Items  
- Get customer info  
- Loop Over Items1  
- Code in JavaScript

**Node Details:**  

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes batches of abandoned checkout items from "Split Out"  
  - Inputs: From "Split Out"  
  - Outputs: Two outputs: one empty (no usage), one to "Get customer info"  
  - Failure Modes: Batch size misconfiguration causing incomplete processing  
  - Version: 3  

- **Get customer info**  
  - Type: HTTP Request  
  - Role: Retrieves detailed customer information based on checkout data (e.g., customer profile, contact details)  
  - Config: Calls Shopify API with customer ID or token  
  - Inputs: From "Loop Over Items" (second output)  
  - Outputs: None connected downstream (possibly final data enrichment)  
  - Failure Modes: API errors, missing customer data  
  - Version: 4.2  

- **Loop Over Items1**  
  - Type: Split In Batches  
  - Role: Processes batches from "Split Out1" (initial abandoned checkouts)  
  - Inputs: From "Split Out1"  
  - Outputs: Two outputs: one empty, one to "Code in JavaScript"  
  - Failure Modes: Similar to "Loop Over Items"  
  - Version: 3  

- **Code in JavaScript**  
  - Type: Code  
  - Role: Custom JavaScript processing (likely formatting or filtering phone numbers and checkout data)  
  - Inputs: From "Loop Over Items1"  
  - Outputs: Feeds "Rapiwa (verify number)" for phone verification  
  - Key Expressions: Custom code manipulating data items, exact logic not specified  
  - Failure Modes: Code errors, null/undefined data causing exceptions  
  - Version: 2  

#### 1.3 Phone Number Verification

**Overview:**  
This block verifies customer phone numbers using the Rapiwa API before sending WhatsApp messages.

**Nodes Involved:**  
- Rapiwa (verify number)  
- If

**Node Details:**  

- **Rapiwa (verify number)**  
  - Type: Rapiwa Node (3rd-party WhatsApp-related API)  
  - Role: Verifies if the phone number is valid and reachable via WhatsApp  
  - Inputs: From "Code in JavaScript"  
  - Outputs: Feeds "If" node with verification result  
  - Failure Modes: API errors, phone number formatting issues, rate limiting  
  - Version: 1  

- **If**  
  - Type: If  
  - Role: Branches workflow based on verification result (e.g., verified vs unverified)  
  - Inputs: From "Rapiwa (verify number)"  
  - Outputs:  
    - True branch: "Rapiwa (sent message)" (send WhatsApp message)  
    - False branch: "Store State of Rows in Unverified & Not Sent" (logging unverified numbers)  
  - Failure Modes: Expression evaluation failure if verification result missing or malformed  
  - Version: 2.2  

#### 1.4 Messaging & Status Handling

**Overview:**  
Sends WhatsApp messages to verified phone numbers and logs the status of both sent and unverified contacts in Google Sheets.

**Nodes Involved:**  
- Rapiwa (sent message)  
- Store State of Rows in Verified & Sent  
- Store State of Rows in Unverified & Not Sent

**Node Details:**  

- **Rapiwa (sent message)**  
  - Type: Rapiwa Node  
  - Role: Sends the WhatsApp message to the verified phone number  
  - Inputs: From "If" node (true branch)  
  - Outputs: Feeds "Store State of Rows in Verified & Sent"  
  - Failure Modes: Message sending errors, API limits, invalid message template  
  - Version: 1  

- **Store State of Rows in Verified & Sent**  
  - Type: Google Sheets  
  - Role: Logs data for verified and contacted customers (e.g., timestamp, phone number, message status)  
  - Inputs: From "Rapiwa (sent message)"  
  - Outputs: Feeds "Wait" node  
  - Failure Modes: Google Sheets API quota, authentication errors, write conflicts  
  - Version: 4.6  

- **Store State of Rows in Unverified & Not Sent**  
  - Type: Google Sheets  
  - Role: Logs data for customers whose phone numbers couldn't be verified or messages were not sent  
  - Inputs: From "If" node (false branch)  
  - Outputs: Feeds "Wait" node  
  - Failure Modes: Same as above for Google Sheets  
  - Version: 4.6  

#### 1.5 Wait State

**Overview:**  
Introduces a wait/delay after processing message sending and logging, likely to prevent API rate limits or pacing issues.

**Nodes Involved:**  
- Wait

**Node Details:**  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for a configured duration (default or custom)  
  - Inputs: From both Google Sheets nodes ("Store State of Rows in Verified & Sent" and "Store State of Rows in Unverified & Not Sent")  
  - Outputs: None (ends workflow cycle)  
  - Failure Modes: None generally, unless misconfigured duration or system resource issues  
  - Webhook ID configured (possibly for external trigger or monitoring)  
  - Version: 1.1  

---

### 3. Summary Table

| Node Name                          | Node Type               | Functional Role                               | Input Node(s)                        | Output Node(s)                                   | Sticky Note                                                     |
|-----------------------------------|-------------------------|-----------------------------------------------|------------------------------------|-------------------------------------------------|----------------------------------------------------------------|
| Schedule Trigger                  | Schedule Trigger        | Periodic workflow initiation                   | None                               | Get Initial Abandoned Checkout                   |                                                                |
| Get Initial Abandoned Checkout    | HTTP Request            | Fetch initial abandoned Shopify checkouts     | Schedule Trigger                   | Split Out1                                       |                                                                |
| Split Out1                       | Split Out               | Split initial abandoned checkout data         | Get Initial Abandoned Checkout     | Loop Over Items1                                 |                                                                |
| Loop Over Items1                 | Split In Batches        | Batch processing for initial abandoned carts  | Split Out1                        | Code in JavaScript                               |                                                                |
| Code in JavaScript               | Code                    | Custom data processing/preparation             | Loop Over Items1                  | Rapiwa (verify number)                           |                                                                |
| Rapiwa (verify number)           | Rapiwa                  | Phone number verification via Rapiwa API      | Code in JavaScript                | If                                              |                                                                |
| If                              | If                      | Conditional branching on verification result  | Rapiwa (verify number)             | Rapiwa (sent message), Store State of Rows in Unverified & Not Sent |                                                                |
| Rapiwa (sent message)            | Rapiwa                  | Send WhatsApp message                           | If (true branch)                  | Store State of Rows in Verified & Sent           |                                                                |
| Store State of Rows in Verified & Sent | Google Sheets           | Log message sent status                         | Rapiwa (sent message)             | Wait                                            |                                                                |
| Store State of Rows in Unverified & Not Sent | Google Sheets           | Log unverified numbers                          | If (false branch)                 | Wait                                            |                                                                |
| Wait                            | Wait                    | Delay/pacing between workflow runs             | Store State of Rows in Verified & Sent, Store State of Rows in Unverified & Not Sent | None                                            |                                                                |
| Get All Abandoned checkouts      | HTTP Request            | Fetch all abandoned Shopify checkouts          | Not connected in main flow        | Split Out                                        |                                                                |
| Split Out                       | Split Out               | Split all abandoned checkout data               | Get All Abandoned checkouts       | Loop Over Items                                  |                                                                |
| Loop Over Items                 | Split In Batches        | Batch processing for all abandoned carts        | Split Out                        | Get customer info                                |                                                                |
| Get customer info               | HTTP Request            | Retrieve detailed customer info                  | Loop Over Items                  | None                                            |                                                                |
| Sticky Note                     | Sticky Note             | Annotation placeholder                           | None                             | None                                            |                                                                |
| Sticky Note2                    | Sticky Note             | Annotation placeholder                           | None                             | None                                            |                                                                |
| Sticky Note3                    | Sticky Note             | Annotation placeholder                           | None                             | None                                            |                                                                |
| Sticky Note4                    | Sticky Note             | Annotation placeholder                           | None                             | None                                            |                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" Node:**  
   - Type: Schedule Trigger  
   - Configure the desired interval (e.g., every hour or daily) to start the workflow automatically.

2. **Create "Get Initial Abandoned Checkout" Node:**  
   - Type: HTTP Request  
   - Configure with Shopify API credentials (OAuth or API key).  
   - Set method to GET and endpoint to Shopify's abandoned checkouts API for initial batch.  
   - Connect input from "Schedule Trigger".

3. **Create "Split Out1" Node:**  
   - Type: Split Out  
   - Connect input from "Get Initial Abandoned Checkout" to split the data into individual items.

4. **Create "Loop Over Items1" Node:**  
   - Type: Split In Batches  
   - Connect input from "Split Out1".  
   - Configure batch size (default or as needed).

5. **Create "Code in JavaScript" Node:**  
   - Type: Code  
   - Connect input from "Loop Over Items1".  
   - Add custom JavaScript code to preprocess each batch item, e.g., extract phone numbers, format data for verification.

6. **Create "Rapiwa (verify number)" Node:**  
   - Type: Rapiwa  
   - Configure with Rapiwa API credentials.  
   - Connect input from "Code in JavaScript".  
   - Set operation to verify phone number.

7. **Create "If" Node:**  
   - Type: If  
   - Connect input from "Rapiwa (verify number)".  
   - Configure condition to check if phone number is verified (e.g., if verification result is true).

8. **Create "Rapiwa (sent message)" Node:**  
   - Type: Rapiwa  
   - Configure with Rapiwa API credentials.  
   - Connect input from "If" node’s true output.  
   - Set operation to send WhatsApp message with pre-defined template or content.

9. **Create "Store State of Rows in Verified & Sent" Node:**  
   - Type: Google Sheets  
   - Configure Google Sheets credentials and target spreadsheet/tab for verified contacts.  
   - Map data such as phone number, timestamp, message status from "Rapiwa (sent message)".  
   - Connect input from "Rapiwa (sent message)".

10. **Create "Store State of Rows in Unverified & Not Sent" Node:**  
    - Type: Google Sheets  
    - Configure Google Sheets credentials and target spreadsheet/tab for unverified contacts.  
    - Map data from "If" node’s false output.  
    - Connect input from "If" node’s false output.

11. **Create "Wait" Node:**  
    - Type: Wait  
    - Connect inputs from both Google Sheets nodes.  
    - Configure delay duration as needed (e.g., 30 seconds) to pace workflow execution.

12. **(Optional) Create "Get All Abandoned checkouts" Node:**  
    - Type: HTTP Request  
    - Configure Shopify API credentials.  
    - Set method to GET and endpoint to fetch all abandoned checkouts.  
    - Connect output to "Split Out".

13. **Create "Split Out" Node:**  
    - Type: Split Out  
    - Connect input from "Get All Abandoned checkouts".

14. **Create "Loop Over Items" Node:**  
    - Type: Split In Batches  
    - Connect input from "Split Out".  
    - Configure batch size.

15. **Create "Get customer info" Node:**  
    - Type: HTTP Request  
    - Configure Shopify API credentials.  
    - Use customer ID from batch items to fetch detailed customer info.  
    - Connect input from "Loop Over Items".

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                      |
|--------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow uses the Rapiwa node for WhatsApp phone number verification and messaging.          | https://www.rapiwa.com/ (API provider)              |
| Google Sheets nodes require OAuth2 credentials with write access to the target spreadsheet.       | Google Sheets API documentation                      |
| Shopify API requests require appropriate API keys or OAuth credentials with "read_checkouts" scope.| Shopify Admin API docs: https://shopify.dev/api/admin-rest |
| The Wait node helps prevent API rate limits and pacing issues during message sending.             | n8n Wait node documentation                          |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. All data handled is legal and public, respecting applicable content policies.