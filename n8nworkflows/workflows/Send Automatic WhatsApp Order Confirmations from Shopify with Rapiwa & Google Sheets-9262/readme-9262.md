Send Automatic WhatsApp Order Confirmations from Shopify with Rapiwa & Google Sheets

https://n8nworkflows.xyz/workflows/send-automatic-whatsapp-order-confirmations-from-shopify-with-rapiwa---google-sheets-9262


# Send Automatic WhatsApp Order Confirmations from Shopify with Rapiwa & Google Sheets

### 1. Workflow Overview

This workflow automates the process of sending WhatsApp order confirmation messages for new Shopify orders using the Rapiwa API and logs all interactions in Google Sheets. It is designed for e-commerce businesses seeking to streamline customer communication by sending personalized WhatsApp notifications immediately after an order is placed.

**Target Use Cases:**  
- Shopify store owners wanting automatic WhatsApp confirmations for new orders  
- Businesses needing to verify WhatsApp numbers before sending messages to avoid failed deliveries and wasted credits  
- Teams requiring an audit trail of sent and unsent messages with verification status in Google Sheets

**Logical Blocks:**

- **1.1 Input Reception:** Receives incoming Shopify order data via a Webhook and splits batch data for processing one order at a time.  
- **1.2 Data Normalization:** Cleans and restructures incoming JSON payload into a standardized format suitable for further operations.  
- **1.3 WhatsApp Number Preparation:** Cleans the customer’s WhatsApp number to remove non-numeric characters and verifies the number’s WhatsApp registration status using Rapiwa.  
- **1.4 Conditional Messaging and Logging:** Based on verification results, sends WhatsApp confirmation messages to verified numbers via Rapiwa, updates Google Sheets rows accordingly, or logs unverified numbers separately.  
- **1.5 Rate Limiting:** Implements wait periods between message sends to avoid hitting rate limits or triggering WhatsApp flags.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming HTTP POST requests containing Shopify order data and initiates per-order processing by splitting batch payloads into individual items.

- **Nodes Involved:**  
  - Webhook  
  - Loop Over Items

- **Node Details:**

  - **Webhook**  
    - *Type:* Webhook  
    - *Technical Role:* Entry point for receiving Shopify order events or similar POST requests.  
    - *Configuration:* Listens on a unique path with POST method, no authentication.  
    - *Input/Output:* No inputs, outputs raw POST payload.  
    - *Edge Cases:* Malformed requests, missing expected fields, HTTP timeouts.  
    - *Sticky Note:* Explains webhook purpose as trigger for incoming order data.

  - **Loop Over Items (SplitInBatches)**  
    - *Type:* SplitInBatches  
    - *Technical Role:* Processes each order individually from potentially batched input.  
    - *Configuration:* Default batch size, no options set.  
    - *Input/Output:* Input from Webhook, outputs each item one at a time for isolated processing.  
    - *Edge Cases:* Empty batches, payloads with unexpected structure.  
    - *Sticky Note:* Describes purpose of splitting for orderly processing.

---

#### 1.2 Data Normalization

- **Overview:** Extracts and restructures relevant fields from the raw webhook data into a clean, consistent JSON object with key customer, order, shipping, and line item details.

- **Nodes Involved:**  
  - Clean Webhooks Response Data (Code)

- **Node Details:**

  - **Clean Webhooks Response Data**  
    - *Type:* Code (JavaScript)  
    - *Technical Role:* Transform raw Shopify webhook JSON into normalized object structure with fields like customer email, phone, full name, order details, shipping and payment info.  
    - *Configuration:* Uses JS code to map input data and safely extract nested fields with fallback defaults.  
    - *Key Expressions:*  
      - Extracts nested fields like `body.customer.email`, `body.shipping_address`, `body.line_items[0]`, etc.  
      - Constructs a single JSON object per order with all relevant keys.  
    - *Input/Output:* Input from Loop Over Items, outputs cleaned JSON for next steps.  
    - *Edge Cases:* Missing nested fields, multiple line items (only first used), null values.  
    - *Sticky Note:* Details the fields extracted and why normalization is critical.

---

#### 1.3 WhatsApp Number Preparation

- **Overview:** Cleans the WhatsApp phone number by removing any non-digit characters and verifies the number’s WhatsApp registration status using the Rapiwa API.

- **Nodes Involved:**  
  - Clean WhatsApp Number (Code)  
  - Check valid whatsapp number Using Rapiwa1 (HTTP Request)

- **Node Details:**

  - **Clean WhatsApp Number**  
    - *Type:* Code (JavaScript)  
    - *Technical Role:* Sanitizes the WhatsApp number by stripping all non-digit characters to comply with WhatsApp’s required numeric format.  
    - *Configuration:* Uses regex replacement `replace(/\D/g, "")` on the `WhatsApp No` field.  
    - *Input/Output:* Input is normalized data from previous step, output is JSON with cleaned number.  
    - *Edge Cases:* Numbers missing country code remain unmodified; logic does not add country code — potential failure if country code absent.  
    - *Sticky Note:* Explains importance of cleaning for WhatsApp compatibility.

  - **Check valid whatsapp number Using Rapiwa1**  
    - *Type:* HTTP Request  
    - *Technical Role:* Sends a POST request to Rapiwa API endpoint to verify if the cleaned number is a valid WhatsApp user.  
    - *Configuration:*  
      - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
      - Auth: HTTP Bearer Token (Rapiwa Bearer Auth credential)  
      - Body Parameter: `number` set dynamically from cleaned WhatsApp No field.  
    - *Input/Output:* Input is cleaned WhatsApp number, output includes `data.exists` boolean flag indicating WhatsApp registration.  
    - *Edge Cases:* API authentication errors, rate limiting from Rapiwa, network timeouts, unexpected API response structure.  
    - *Sticky Note:* Highlights verification step importance to avoid wasted messages.

---

#### 1.4 Conditional Messaging and Logging

- **Overview:** Routes workflow based on WhatsApp number validity to either send a personalized confirmation message and log as verified or log as unverified without sending.

- **Nodes Involved:**  
  - If (conditional)  
  - Send Message Using Rapiwa (HTTP Request)  
  - Change State of Rows in Verified & Sent (Google Sheets)  
  - Change State of Rows in Unverified & Not Sent (Google Sheets)

- **Node Details:**

  - **If**  
    - *Type:* If (conditional)  
    - *Technical Role:* Branches the workflow depending on `data.exists == true` from Rapiwa verification.  
    - *Configuration:* Checks expression `={{ $json.data.exists }}` against string `"true"`.  
    - *Input/Output:* Input from WhatsApp verification node, outputs two branches: True (verified) and False (unverified).  
    - *Edge Cases:* The comparison is string-based, may require adjustment if API returns boolean.  
    - *Sticky Note:* Describes branching logic for valid vs invalid numbers.

  - **Send Message Using Rapiwa**  
    - *Type:* HTTP Request  
    - *Technical Role:* Sends templated WhatsApp message to verified customers using Rapiwa API.  
    - *Configuration:*  
      - URL: `https://app.rapiwa.com/api/send-message`  
      - Auth: HTTP Header Auth (shopify-app-auth credential)  
      - Body Parameters: `number` (customer phone), `message_type: text`, personalized `message` with order details.  
      - Message uses multiple dynamic placeholders from order data (e.g., customer name, product title, order ID, shipping, payment summary).  
    - *Input/Output:* Input is verified order data, output includes message sending response.  
    - *Edge Cases:* API errors, auth failures, message formatting issues, rate limits, network errors.  
    - *Sticky Note:* Explains message template and core function of connecting to WhatsApp via Rapiwa.

  - **Change State of Rows in Verified & Sent**  
    - *Type:* Google Sheets  
    - *Technical Role:* Updates the relevant row in Google Sheet marking message as sent and number as verified.  
    - *Configuration:* Uses `update` operation keyed by `status` (may be better with unique key for resilience). Maps multiple fields including name, email, number, order id, status, validity = "verified".  
    - *Credentials:* Google Sheets OAuth2.  
    - *Input/Output:* Input from Send Message node, output triggers waiting node.  
    - *Edge Cases:* Google Sheets API permission errors, row matching failures if sheet changes.  
    - *Sticky Note:* Notes importance of updating row status to prevent duplicate sends.

  - **Change State of Rows in Unverified & Not Sent**  
    - *Type:* Google Sheets  
    - *Technical Role:* Updates Google Sheet rows for unverified WhatsApp numbers with `validity: unverified` and `status: not sent`.  
    - *Configuration:* Similar to verified update but different fields for validity and status.  
    - *Credentials:* Google Sheets OAuth2.  
    - *Input/Output:* Input from If node false branch, output triggers wait node.  
    - *Edge Cases:* Same as above.  
    - *Sticky Note:* Explains logging unverified numbers to avoid repeated attempts.

---

#### 1.5 Rate Limiting

- **Overview:** Introduces a wait period between processing each order to prevent API rate limits or WhatsApp number blocking.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - *Type:* Wait  
    - *Technical Role:* Pauses workflow execution for a configured amount of time (default unspecified, typical 5 seconds or similar).  
    - *Configuration:* No specific parameters shown (empty). Should be set to a few seconds to throttle message sending.  
    - *Input/Output:* Input from Google Sheets row update nodes, loops back to Loop Over Items to process next batch.  
    - *Edge Cases:* Long wait times may delay processing; zero wait risks API throttling or number blocking.  
    - *Sticky Note:* Highlights importance of waiting to avoid flagged WhatsApp numbers.

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                                          | Input Node(s)                   | Output Node(s)                         | Sticky Note                                                                                  |
|-----------------------------------|-----------------------|----------------------------------------------------------|--------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| Webhook                           | Webhook               | Receives incoming Shopify order webhook POST requests    | -                              | Loop Over Items                       | Listens for incoming HTTP POST requests from Shopify or external platform.                    |
| Loop Over Items                   | SplitInBatches        | Processes each order individually                         | Webhook                        | Clean Webhooks Response Data          | Ensures each message is processed separately and in order.                                  |
| Clean Webhooks Response Data      | Code                  | Normalizes and extracts key order/customer data          | Loop Over Items                | Clean WhatsApp Number                 | Extracts customer, order, shipping, and line item details into a clean JSON structure.       |
| Clean WhatsApp Number             | Code                  | Cleans WhatsApp numbers by removing non-digits           | Clean Webhooks Response Data   | Check valid whatsapp number Using Rapiwa1 | Removes extra characters to avoid WhatsApp delivery errors.                                 |
| Check valid whatsapp number Using Rapiwa1 | HTTP Request          | Verifies if WhatsApp number is registered with WhatsApp  | Clean WhatsApp Number          | If                                  | Checks if number is active WhatsApp user to avoid sending to invalid contacts.               |
| If                               | If                    | Branches logic based on WhatsApp number verification     | Check valid whatsapp number Using Rapiwa1 | Send Message Using Rapiwa / Change State of Rows in Unverified & Not Sent | Checks `data.exists == true` to determine valid/invalid numbers.                             |
| Send Message Using Rapiwa         | HTTP Request          | Sends personalized WhatsApp order confirmation message   | If (true branch)               | Change State of Rows in Verified & Sent | Core message send node using Rapiwa API with templated order info.                           |
| Change State of Rows in Verified & Sent | Google Sheets         | Updates Google Sheet for verified and sent messages      | Send Message Using Rapiwa      | Wait                                | Updates row with status “sent” and validity “verified” to prevent duplicate sends.           |
| Change State of Rows in Unverified & Not Sent | Google Sheets         | Updates Google Sheet for unverified numbers (not sent)   | If (false branch)              | Wait                                | Logs unverified numbers with status “not sent” for reference and exclusion.                  |
| Wait                             | Wait                  | Pauses workflow between message sends to avoid limits   | Change State of Rows in Verified & Sent / Change State of Rows in Unverified & Not Sent | Loop Over Items                       | Prevents sending too many messages too quickly, avoiding flags or blocks on WhatsApp.        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: Unique path (e.g., "a9b6a936-e5f2-4d4c-9cf9-182de0a970d5")  
   - HTTP Method: POST  
   - No authentication  
   - Purpose: Receive Shopify order webhook payloads.

2. **Create SplitInBatches Node (Loop Over Items)**  
   - Connect input from Webhook node.  
   - Default batch size (1) or as required.  
   - Purpose: Process each order one by one.

3. **Create Code Node (Clean Webhooks Response Data)**  
   - Connect input from Loop Over Items.  
   - JS code: Extract customer email, phone, full name, order details, shipping address, tax, discount, payment, and line item details into a normalized JSON object.  
   - Purpose: Normalize raw payload into usable structured data.

4. **Create Code Node (Clean WhatsApp Number)**  
   - Connect input from Clean Webhooks Response Data.  
   - JS code: For each item, extract `WhatsApp No` field, convert to string if needed, remove all non-digit characters using `replace(/\D/g, "")`, update field with cleaned number.  
   - Important: Ensure numbers already include country code or add logic to prepend it if necessary.

5. **Create HTTP Request Node (Check valid whatsapp number Using Rapiwa1)**  
   - Connect input from Clean WhatsApp Number.  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Authentication: HTTP Bearer Token (create and assign credential with Rapiwa API token)  
   - Body Parameters: JSON with `number` set dynamically from cleaned WhatsApp number field.  
   - Purpose: Verify if the cleaned number is a valid WhatsApp user.

6. **Create If Node**  
   - Connect input from WhatsApp verification node.  
   - Condition: Check if `={{ $json.data.exists }}` equals string `"true"` (adjust if your API returns boolean).  
   - Output branches: True (verified), False (unverified).

7. **Create HTTP Request Node (Send Message Using Rapiwa)**  
   - Connect input from If node’s True branch.  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Authentication: HTTP Header Auth (create credential or use existing)  
   - Body Parameters:  
     - `number`: customer phone number from JSON  
     - `message_type`: "text"  
     - `message`: Multiline templated text with placeholders for customer name, order details, shipping info, payment summary, order date, etc.  
   - Purpose: Send WhatsApp confirmation message.

8. **Create Google Sheets Node (Change State of Rows in Verified & Sent)**  
   - Connect input from Send Message node.  
   - Operation: Update  
   - Document ID and Sheet: Set to your target Google Sheet  
   - Matching column: Use unique identifier (e.g., `row_number` or `order id`)  
   - Map columns: name, email, number, order id, item name, total price, validity = "verified", status = "sent", and others as needed.  
   - Credentials: Google Sheets OAuth2.  
   - Purpose: Mark message as sent in the sheet.

9. **Create Google Sheets Node (Change State of Rows in Unverified & Not Sent)**  
   - Connect input from If node’s False branch.  
   - Operation: Update  
   - Document and Sheet: Same or another sheet for unverified numbers.  
   - Matching column: same as above.  
   - Map columns: validity = "unverified", status = "not sent".  
   - Credentials: Google Sheets OAuth2.  
   - Purpose: Log unverified numbers without sending message.

10. **Create Wait Node**  
    - Connect inputs from both Google Sheets update nodes.  
    - Set wait time to a few seconds (e.g., 5 seconds) to avoid rate limiting.  
    - Connect output back to Loop Over Items node to process the next item.

11. **Final Connections**  
    - Webhook → Loop Over Items  
    - Loop Over Items → Clean Webhooks Response Data → Clean WhatsApp Number → Check valid whatsapp number Using Rapiwa1 → If  
    - If True → Send Message Using Rapiwa → Change State of Rows in Verified & Sent → Wait → Loop Over Items  
    - If False → Change State of Rows in Unverified & Not Sent → Wait → Loop Over Items

**Credentials Setup:**  
- Create HTTP Bearer Auth credential for Rapiwa API verification endpoint with your API token.  
- Create HTTP Header Auth credential for Rapiwa send-message endpoint or adapt as needed depending on your API auth method.  
- Configure Google Sheets OAuth2 credentials with appropriate permissions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires a Google Sheet structured with columns: `name`, `number`, `order id`, `item name`, `total price`, `validity`, and `status`. Sample sheet layouts are provided in the sticky notes for both verified and unverified logs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | [Verified Sheet Sample](https://docs.google.com/spreadsheets/d/1b5b9kvkfuCMtZ-7Z-w21sXOmHFmQ30N1iQPc2WMStEo/edit?usp=sharing) and [Unverified Sheet Sample](https://docs.google.com/spreadsheets/d/1H29z_8tnsu8AvCsI7o1SjiV-5LDTwiVVQk-BNr8SMk0/edit?usp=sharing) |
| The If node’s condition compares the WhatsApp verification API response string `"true"`. Adjust this comparison if the API returns boolean `true` or another format to prevent logical errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |                                                                                                                                                                                       |
| The Clean WhatsApp Number node only strips non-numeric characters but does not add country codes. Ensure all numbers already include the correct country prefix to avoid invalid number errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |                                                                                                                                                                                       |
| To avoid message sending failures or account flags, the Wait node introduces delays between each WhatsApp message send. Typical wait times are 3–10 seconds but adjust based on your volume and Rapiwa’s rate limits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |                                                                                                                                                                                       |
| Troubleshooting tips:  
- Verify the `data.exists` field in the Rapiwa verification node output to confirm number status.  
- Check phone number formats and consider enhancing the cleaning step for country code normalization.  
- Confirm Google Sheets document IDs, sheet GIDs, and OAuth2 credentials permissions if updates fail.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |                                                                                                                                                                                       |
| This workflow was created and tested with n8n version supporting SplitInBatches v3, HTTP Request v4.2, Google Sheets v4.6, and Code nodes with JS ES6 support. Verify compatibility if running on older versions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |                                                                                                                                                                                       |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.