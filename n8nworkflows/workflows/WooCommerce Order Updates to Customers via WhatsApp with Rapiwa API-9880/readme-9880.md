WooCommerce Order Updates to Customers via WhatsApp with Rapiwa API

https://n8nworkflows.xyz/workflows/woocommerce-order-updates-to-customers-via-whatsapp-with-rapiwa-api-9880


# WooCommerce Order Updates to Customers via WhatsApp with Rapiwa API

### 1. Workflow Overview

This workflow automates the process of sending WooCommerce order status updates to customers via WhatsApp, using the Rapiwa API for message delivery and phone number verification. It is designed for WooCommerce store owners who want to keep customers informed about their order status in real time through a popular messaging platform.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming WooCommerce order events via a webhook.
- **1.2 Data Formatting:** Transforms the raw webhook payload into a structured data object with customer and product details.
- **1.3 Batch Processing:** Splits the list of orders into batches to manage throughput.
- **1.4 Phone Number Cleaning:** Normalizes WhatsApp phone numbers by stripping non-numeric characters.
- **1.5 WhatsApp Number Verification:** Checks if the cleaned phone number is registered on WhatsApp using the Rapiwa API.
- **1.6 Conditional Routing:** Routes data depending on verification result to either send messages or log unverified numbers.
- **1.7 Message Sending:** Sends personalized WhatsApp messages via Rapiwa API to verified numbers.
- **1.8 Logging:** Records details of both sent and unsent messages in Google Sheets for audit and tracking.
- **1.9 Throttling / Looping:** Waits and loops over batches to handle rate limits and continuous processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures incoming WooCommerce order updates via HTTP POST requests to a configured webhook.
- **Nodes Involved:** `Webhook`
- **Node Details:**
  - **Type:** Webhook node, listens for HTTP POST requests.
  - **Configuration:** Path set to a unique identifier (`4746f380-002d-494d-90fe-7c963bd7a67b`), method POST.
  - **Input:** External WooCommerce order event payload.
  - **Output:** Raw JSON payload forwarded to the next node.
  - **Failure Modes:** Invalid payload structure or missing data can cause downstream errors.
  - **Version:** n8n webhook version 2.
  
#### 2.2 Data Formatting

- **Overview:** Parses and normalizes the incoming webhook payload to extract customer info, products ordered, and invoice link.
- **Nodes Involved:** `Format Webhook Response Data`
- **Node Details:**
  - **Type:** Code node executing JavaScript.
  - **Configuration:** 
    - Extracts `body` from `item.json.body`.
    - Constructs a `customer` object with billing information.
    - Maps each product in `line_items` to include status, name, size (from metadata), IDs, quantity, price, total, and image URL.
    - Returns a structured object in `json.data` with `customer`, `products`, and `invoice_link`.
  - **Key Expressions:** Uses optional chaining to safely access metadata.
  - **Input:** Raw webhook JSON.
  - **Output:** Structured data object.
  - **Failure Modes:** Missing or malformed fields in the webhook payload can cause runtime exceptions.
  - **Version:** n8n code node v2.

#### 2.3 Batch Processing

- **Overview:** Splits the array of processed orders into manageable batches for sequential processing.
- **Nodes Involved:** `Loop Over Items`
- **Node Details:**
  - **Type:** SplitInBatches node.
  - **Configuration:** Default batch size (adjustable outside current snapshot).
  - **Input:** Array of reformatted order data.
  - **Output:** Individual batches forwarded downstream.
  - **Failure Modes:** Large batch sizes might lead to rate limiting or memory issues.
  - **Version:** n8n v3.

#### 2.4 Phone Number Cleaning

- **Overview:** Ensures the WhatsApp number is sanitized by removing all non-digit characters.
- **Nodes Involved:** `Clean WhatsApp Number`
- **Node Details:**
  - **Type:** Code node.
  - **Configuration:** JavaScript code uses regex to strip non-digits from the `number` field.
  - **Input:** Batch items with `number` field.
  - **Output:** Same items with cleaned phone numbers.
  - **Failure Modes:** Missing `number` field results in empty strings; may cause verification failures downstream.
  - **Version:** n8n code node v2.

#### 2.5 WhatsApp Number Verification

- **Overview:** Uses Rapiwa API to verify if the cleaned number is registered on WhatsApp.
- **Nodes Involved:** `Check valid whatsapp number Using Rapiwa`
- **Node Details:**
  - **Type:** HTTP Request node.
  - **Configuration:** 
    - URL: `https://app.rapiwa.com/api/verify-whatsapp` (expression).
    - Method: POST.
    - Body: JSON containing the cleaned `number`.
    - Authentication: HTTP Bearer using Rapiwa credentials.
  - **Input:** Cleaned numbers.
  - **Output:** Verification response including `exists` boolean.
  - **Failure Modes:** Network errors, invalid tokens, or API limits may cause request failure.
  - **Version:** n8n HTTP Request v4.2.
  - **Credentials:** Requires valid Rapiwa Bearer token.

#### 2.6 Conditional Routing

- **Overview:** Routes each item depending on whether the number is verified as a WhatsApp user.
- **Nodes Involved:** `If`
- **Node Details:**
  - **Type:** IF node.
  - **Configuration:** Checks if `$json.data.exists` is true (boolean strict).
  - **Input:** Verification response.
  - **Output:** 
    - True path: number verified → send message.
    - False path: number unverified → log unverified.
  - **Failure Modes:** Expression errors if `exists` is undefined.
  - **Version:** n8n IF node v2.2.

#### 2.7 Message Sending

- **Overview:** Sends a personalized WhatsApp message via Rapiwa API to verified customers.
- **Nodes Involved:** `Rapiwa Sender`
- **Node Details:**
  - **Type:** HTTP Request node.
  - **Configuration:** 
    - URL: `https://app.rapiwa.com/api/send-message`.
    - Method: POST.
    - Body: JSON including `number`, message type (`text`), and a templated `message`.
    - Message template dynamically inserts customer first and last name, order status, product details (name, size, qty, price, total), shipping address, product image URL, and invoice link.
    - Authentication: HTTP Bearer with Rapiwa token.
  - **Input:** Verified numbers and structured order data.
  - **Output:** API response confirming message sent.
  - **Failure Modes:** API errors, invalid tokens, rate limits, or malformed message templates.
  - **Version:** HTTP Request v4.2.
  - **Credentials:** Rapiwa Bearer token required.

#### 2.8 Logging

- **Overview:** Logs order and message status data into Google Sheets for both verified & sent and unverified & not sent cases.
- **Nodes Involved:** `Store State of Rows in Verified & Sent`, `Store State of Rows in Unverified & Not Sent`
- **Node Details:**
  - **Type:** Google Sheets node.
  - **Configuration:**
    - Operation: Append row.
    - Document ID and Sheet: points to a specific Google Sheet with predefined schema.
    - Columns: Includes customer name, size, email, phone number, product details, invoice link, message status (`sent` or `not sent`), and validity (`verified` or `unverified`).
    - Credential: Google Sheets OAuth2.
  - **Input:** Structured order and verification data.
  - **Output:** Confirmation of row append.
  - **Failure Modes:** Authentication failure, API limits, or schema mismatch.
  - **Version:** Google Sheets v4.6.

#### 2.9 Throttling / Looping

- **Overview:** Implements a wait/delay before looping back to process the next batch, managing rate limits and flow control.
- **Nodes Involved:** `Wait`
- **Node Details:**
  - **Type:** Wait node.
  - **Configuration:** Default delay (customizable).
  - **Input:** Completed batch processing.
  - **Output:** Triggers the next batch processing cycle.
  - **Failure Modes:** Long delays might cause workflow timeouts; short delays risk hitting API rate limits.
  - **Version:** n8n Wait v1.1.

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                                  | Input Node(s)                  | Output Node(s)                        | Sticky Note                                                                                                            |
|----------------------------------|-------------------------|-------------------------------------------------|-------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Webhook                          | Webhook                 | Receive WooCommerce order events                 | -                             | Format Webhook Response Data        | ## 1. Webhook<br>- Accepts incoming HTTP POST requests.<br>- Configured with unique path.                               |
| Format Webhook Response Data     | Code                    | Normalize and structure order payload            | Webhook                       | Loop Over Items                     | ## 2. Format Webhook Response Data<br>- Transforms raw body into structured data.                                       |
| Loop Over Items                  | SplitInBatches          | Batch processing of orders                        | Format Webhook Response Data  | Clean WhatsApp Number (main)        |                                                                                                                        |
| Clean WhatsApp Number            | Code                    | Clean phone numbers by removing non-digit chars | Loop Over Items               | Check valid whatsapp number Using Rapiwa | ## 1. Clean WhatsApp Number<br>- Strips non-digit chars from `number`.<br>## 2. Check valid whatsapp number Using Rapiwa |
| Check valid whatsapp number Using Rapiwa | HTTP Request            | Verify WhatsApp registration of phone numbers    | Clean WhatsApp Number         | If                                |                                                                                                                        |
| If                              | If                      | Conditional routing on verification result       | Check valid whatsapp number   | Rapiwa Sender (true), Store State of Rows in Unverified & Not Sent (false) | ## 1. If<br>- Routes based on verification.<br>## 2. Rapiwa Sender<br>- Sends messages to verified numbers.<br>## 3. Store State of Rows in Verified & Sent<br>- Logs sent messages.<br>## 4. Store State of Rows in Unverified & Not Sent<br>- Logs unverified numbers. |
| Rapiwa Sender                   | HTTP Request            | Send WhatsApp message via Rapiwa API              | If (true)                    | Store State of Rows in Verified & Sent |                                                                                                                        |
| Store State of Rows in Verified & Sent | Google Sheets           | Log details of sent messages                       | Rapiwa Sender                | Wait                              |                                                                                                                        |
| Store State of Rows in Unverified & Not Sent | Google Sheets           | Log details of unverified and unsent messages     | If (false)                   | Wait                              |                                                                                                                        |
| Wait                            | Wait                    | Throttle and loop back for batch processing       | Store State of Rows in Verified & Sent, Store State of Rows in Unverified & Not Sent | Loop Over Items                     |                                                                                                                        |
| Sticky Note                     | Sticky Note             | Documentation and overview                         | -                             | -                                 | # Send WooCommerce Order Status Updates to Customers via WhatsApp Using Rapiwa API<br>Full workflow description and usage instructions. |
| Sticky Note1                    | Sticky Note             | Explains Webhook and Format Webhook Response Data | -                             | -                                 | ## 1. Webhook and ## 2. Format Webhook Response Data explanations.                                                    |
| Sticky Note2                    | Sticky Note             | Explains phone cleaning and WhatsApp verification | -                             | -                                 | ## 1. Clean WhatsApp Number and ## 2. Check valid whatsapp number Using Rapiwa explanations.                            |
| Sticky Note6                    | Sticky Note             | Explains IF node routing, sending messages, and logging | -                             | -                                 | ## 1. If, ## 2. Rapiwa Sender, ## 3. Store State of Rows in Verified & Sent, ## 4. Store State of Rows in Unverified & Not Sent explanations. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook
   - HTTP Method: POST
   - Path: `4746f380-002d-494d-90fe-7c963bd7a67b`
   - Purpose: Receive WooCommerce order event payloads.

2. **Add Code Node: Format Webhook Response Data**
   - Paste JavaScript code to extract `billing`, `line_items`, and `payment_url` from incoming payload.
   - Output structured `json.data` with customer details, products array and invoice link.

3. **Add SplitInBatches Node: Loop Over Items**
   - Connect from Format Webhook Response Data.
   - Set batch size to a controlled number (e.g., 1–5).
   - Purpose: Process orders in manageable batches.

4. **Add Code Node: Clean WhatsApp Number**
   - Input: Batch items with potentially messy phone numbers.
   - JavaScript: Remove all non-digit characters from the `number` field.
   - Output: Cleaned numbers in `json.number`.

5. **Add HTTP Request Node: Check valid whatsapp number Using Rapiwa**
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`
   - Method: POST
   - Body Parameters: JSON with `number` field set to cleaned phone number.
   - Authentication: HTTP Bearer with Rapiwa Bearer token.
   - Connect from Clean WhatsApp Number.
   - Purpose: Verify WhatsApp registration status.

6. **Add IF Node**
   - Condition: Check if `$json.data.exists` is `true`.
   - True Path: Verified numbers.
   - False Path: Unverified numbers.

7. **On True Path: Add HTTP Request Node Rapiwa Sender**
   - URL: `https://app.rapiwa.com/api/send-message`
   - Method: POST
   - Body Parameters: 
     - `number`: cleaned phone number.
     - `message_type`: "text".
     - `message`: Template string populated with customer first and last name, order status, product details, shipping address, product image URL, and invoice link.
   - Authentication: HTTP Bearer with Rapiwa token.
   - Connect from IF node (true output).

8. **On True Path: Add Google Sheets Node Store State of Rows in Verified & Sent**
   - Operation: Append row.
   - Document ID: Google Sheets document ID prepared for logging.
   - Sheet Name: default sheet (gid=0).
   - Columns: Map customer and product info, mark status as "sent" and validity as "verified".
   - Credential: Google Sheets OAuth2.
   - Connect from Rapiwa Sender node.

9. **On False Path: Add Google Sheets Node Store State of Rows in Unverified & Not Sent**
   - Same configuration as above but mark status as "not sent" and validity as "unverified".
   - Connect from IF node (false output).

10. **Add Wait Node**
    - Connect from both Google Sheets nodes.
    - Configure delay to control processing rate (e.g., 5-10 seconds).
    - Purpose: Throttle processing and avoid rate limits.

11. **Connect Wait Node output back to SplitInBatches node**
    - To continue processing next batch after delay.

12. **Set Credentials**
    - Add HTTP Bearer credentials for Rapiwa API (used in both verification and sending).
    - Add Google Sheets OAuth2 credentials with permission to append rows.

13. **Prepare Google Sheets**
    - Create a Google Sheet with columns matching the schema:
      - name, size, email, number, status, address1, quantity, validity, productId, totalPrice, invoiceLink, productImage, productTitle, product status
    - Share and authorize Google Sheets OAuth2 credentials.

14. **Test Workflow**
    - Send test WooCommerce order payloads to webhook.
    - Monitor logs in Google Sheets.
    - Verify WhatsApp messages are sent to verified numbers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                                                                                |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow listens for WooCommerce order events and sends WhatsApp status updates using Rapiwa API with verification and logging features.                                                                                                                                 | Main workflow purpose (Sticky Note).                                                                                                                          |
| Google Sheet example and required column structure: [Sample Sheet](https://docs.google.com/spreadsheets/d/1WKOFqFdF3tqfes5KKO1WGAp20SpO3jJl9GsE9CmtfLI/edit?usp=sharing)                                                                 | Google Sheets setup.                                                                                                                                           |
| Rapiwa API Dashboard, Documentation, and official site: [Rapiwa Dashboard](https://app.rapiwa.com/login), [Rapiwa Docs](https://docs.rapiwa.com), [Rapiwa Website](https://rapiwa.com)                                                  | Rapiwa API resources.                                                                                                                                          |
| Support channels: WhatsApp chat [Chat on WhatsApp](https://wa.me/8801322827799), Discord [SpaGreen Community](https://discord.gg/SsCChWEP), Facebook Group [SpaGreen Support](https://www.facebook.com/groups/spagreenbd)               | Support and community help.                                                                                                                                    |
| Developer portfolio and site: [Codecanyon SpaGreen](https://codecanyon.net/user/spagreen/portfolio), [SpaGreen Website](https://spagreen.net)                                                                                           | Credits and developer resources.                                                                                                                              |

---

This documentation provides an exhaustive and structured reference for understanding, reproducing, and modifying the WooCommerce to WhatsApp order update workflow using n8n and Rapiwa API integration. It addresses all nodes, their roles, configurations, and potential failure points, as well as the overall logic and data flow.