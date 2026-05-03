Send Rapiwa WhatsApp Apology & Reorder Link When Shopify Order is Cancelled

https://n8nworkflows.xyz/workflows/send-rapiwa-whatsapp-apology---reorder-link-when-shopify-order-is-cancelled-10237


# Send Rapiwa WhatsApp Apology & Reorder Link When Shopify Order is Cancelled

### 1. Workflow Overview

This workflow automates sending personalized WhatsApp apology messages with reorder links to customers whose Shopify orders have been cancelled. It ensures customers are contacted only if their phone numbers are verified as registered on WhatsApp via the **Rapiwa API**. The workflow also logs communications—both successful and unsuccessful attempts—into Google Sheets for record-keeping and monitoring purposes.

The workflow is logically divided into these blocks:

- **1.1 Shopify Trigger & Batch Processing:** Listens for order cancellation events from Shopify and processes orders in manageable batches.
- **1.2 Data Extraction & Cleaning:** Simplifies and extracts relevant customer and order data, and sanitizes the phone number to WhatsApp-compatible format.
- **1.3 WhatsApp Number Verification:** Uses Rapiwa API to verify if the customer's phone number is registered on WhatsApp.
- **1.4 Conditional Messaging & Logging:** Based on verification result, sends a personalized apology message and logs the transaction in Google Sheets or logs unverified contacts separately.
- **1.5 Rate Limiting:** Controls the workflow pace using wait nodes to comply with API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Shopify Trigger & Batch Processing

**Overview:**  
Starts the workflow for every cancelled order in Shopify and splits incoming orders into batches to avoid overload and manage API rate limits.

**Nodes Involved:**  
- Shopify Trigger (`orders/cancelled`)  
- SplitInBatches (`Loop Over Items`)

**Node Details:**

- **Shopify Trigger**  
  - *Type:* Shopify Trigger node  
  - *Role:* Listens to Shopify webhook for order cancellation events.  
  - *Configuration:* Triggers on `orders/cancelled` topic, authenticated via Shopify Access Token.  
  - *Inputs:* External webhook (Shopify)  
  - *Outputs:* Passes order cancellation data downstream.  
  - *Errors:* Possible webhook misconfiguration, authentication failure, or network issues.

- **SplitInBatches**  
  - *Type:* SplitInBatches node  
  - *Role:* Processes orders one-by-one or in small batches to control flow.  
  - *Configuration:* Default batch size (implicit), no special options set.  
  - *Inputs:* Receives batch of cancelled orders from Shopify Trigger.  
  - *Outputs:* Sends individual order items to the next node.  
  - *Errors:* Batch size misconfiguration or downstream node failures could block processing.

---

#### 1.2 Data Extraction & Cleaning

**Overview:**  
Extracts and formats key order and customer data into a simplified JSON object, then cleans the customer's billing phone number to prepare it for WhatsApp verification.

**Nodes Involved:**  
- Code (Simplify Order Data)  
- Code (Clean WhatsApp Number)

**Node Details:**

- **Code (Simplify Order Data)**  
  - *Type:* Code (JavaScript) node  
  - *Role:* Parses the Shopify order JSON, extracts customer name, email, phone, order details (totals, products), addresses, fulfillment, and refund info into a clean structure.  
  - *Key Logic:* Uses optional chaining and fallback values to avoid undefined fields, maps line items to product details.  
  - *Inputs:* Single order object from SplitInBatches node.  
  - *Outputs:* Simplified order object for downstream nodes.  
  - *Edge Cases:* Missing customer or address fields; partial or empty order data.  
  - *Notes:* Robust against missing data using null defaults.

- **Code (Clean WhatsApp Number)**  
  - *Type:* Code (JavaScript) node  
  - *Role:* Sanitizes the billing phone number by stripping all non-digit characters to produce a clean numeric string suitable for WhatsApp.  
  - *Key Logic:* Uses regex to remove spaces, dashes, parentheses, and other characters.  
  - *Inputs:* Simplified order data from prior Code node.  
  - *Outputs:* Updated data with cleaned phone number.  
  - *Edge Cases:* Null or invalid phone numbers; empty strings; international dialing codes missing or malformed.

---

#### 1.3 WhatsApp Number Verification

**Overview:**  
Verifies the cleaned phone number using the Rapiwa API to confirm if it is registered on WhatsApp.

**Nodes Involved:**  
- HTTP Request (Check valid whatsapp number Using Rapiwa)  
- IF (Check Verification Result)

**Node Details:**

- **HTTP Request (Check valid whatsapp number Using Rapiwa)**  
  - *Type:* HTTP Request node  
  - *Role:* Sends a POST request to `https://app.rapiwa.com/api/verify-whatsapp` with the cleaned phone number in the body.  
  - *Authentication:* Bearer token (Rapiwa API token).  
  - *Inputs:* Receives cleaned phone number from Code node.  
  - *Outputs:* API response indicating if the number exists on WhatsApp.  
  - *Errors:* API auth failure, network timeout, invalid API token, malformed request.

- **IF**  
  - *Type:* If node  
  - *Role:* Evaluates the Rapiwa API response to determine if the number is verified (exists === true).  
  - *Condition:* `$json.data.exists` is boolean true (strict check).  
  - *Inputs:* Response from Rapiwa verification node.  
  - *Outputs:* Routes messages to verified or unverified branches.  
  - *Edge Cases:* API response format changes; missing `data.exists` field; false negatives or positives.

---

#### 1.4 Conditional Messaging & Logging

**Overview:**  
Depending on verification, sends an apology message with reorder link via Rapiwa API to verified numbers and logs these events. For unverified numbers, logs details separately without sending messages.

**Nodes Involved:**  
- HTTP Request (Send Message Using Rapiwa)  
- Google Sheets (Save State of Rows in Verified & Sent)  
- Google Sheets (Save State of Rows in Verified & Sent1)

**Node Details:**

- **HTTP Request (Send Message Using Rapiwa)**  
  - *Type:* HTTP Request node  
  - *Role:* Sends a POST request to `https://app.rapiwa.com/api/send-message` with the recipient's phone number and a personalized apology message containing the reorder link.  
  - *Authentication:* Bearer token (same Rapiwa API token).  
  - *Message Template:* Includes customer name, apology text, reorder link, and discount mention.  
  - *Inputs:* Verified phone number and order data from IF node.  
  - *Outputs:* Confirmation of message sent, passed to logging node.  
  - *Errors:* Message quota exceeded, API errors, invalid phone number despite verification.

- **Google Sheets (Save State of Rows in Verified & Sent)**  
  - *Type:* Google Sheets node  
  - *Role:* Appends a row to a Google Sheet logging customers successfully sent WhatsApp messages.  
  - *Logged Fields:* Customer name, email, price (currency + amount), status ("sent"), product title, phone number, billing address, validity ("verified"), reorder link.  
  - *Inputs:* Data from sent message node.  
  - *Outputs:* Confirmation of append operation.  
  - *Errors:* Google API quota exceeded, invalid sheet ID, permission issues.

- **Google Sheets (Save State of Rows in Verified & Sent1)**  
  - *Type:* Google Sheets node  
  - *Role:* Appends a row logging customers whose numbers were unverified and who were not sent messages.  
  - *Logged Fields:* Similar fields with status "not sent" and validity "unverified".  
  - *Inputs:* Data from IF node branch for unverified numbers.  
  - *Outputs:* Confirmation of append operation.  
  - *Errors:* Same as above.

---

#### 1.5 Rate Limiting

**Overview:**  
Implements a wait period to avoid exceeding API rate limits or triggering spam detection by spacing out message sending.

**Nodes Involved:**  
- Wait

**Node Details:**

- **Wait**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow execution for a configured duration before processing next item.  
  - *Configuration:* Default delay (not explicitly set in JSON, assumed default or configurable).  
  - *Inputs:* After logging nodes, before looping back to batch processing.  
  - *Outputs:* Controls flow rate to next batch or message.  
  - *Edge Cases:* Too short delay may cause API throttling; too long reduces throughput.

---

### 3. Summary Table

| Node Name                           | Node Type              | Functional Role                                      | Input Node(s)                    | Output Node(s)                         | Sticky Note                                                                                                                     |
|-----------------------------------|------------------------|----------------------------------------------------|---------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Shopify Trigger                   | Shopify Trigger        | Triggers workflow on Shopify order cancellation    | External webhook                | Loop Over Items                      | ## Shopify Trigger (Order Cancelled) Purpose and details                                                                        |
| Loop Over Items                  | SplitInBatches         | Processes orders in batches                         | Shopify Trigger                | Code                                | ## Shopify Trigger & SplitInBatches overview                                                                                   |
| Code                             | Code                   | Simplifies and extracts key order/customer data    | Loop Over Items                | Clean WhatsApp Number                | ## Code (Simplify Order Data) extraction details                                                                                |
| Clean WhatsApp Number             | Code                   | Cleans and formats phone number for WhatsApp       | Code                          | Check valid whatsapp number Using Rapiwa | ## Code (Clean WhatsApp Number) formatting and HTTP Request (Rapiwa check) details                                              |
| Check valid whatsapp number Using Rapiwa | HTTP Request           | Verifies if phone number is registered on WhatsApp | Clean WhatsApp Number          | If                                  | ## Code (Clean WhatsApp Number) and Rapiwa API verification explained                                                          |
| If                               | If                     | Routes based on WhatsApp number verification status | Check valid whatsapp number Using Rapiwa | Send Message Using Rapiwa / Save State of Rows in Verified & Sent1 | ## IF node logic and subsequent messaging and logging explained                                                                 |
| Send Message Using Rapiwa         | HTTP Request           | Sends WhatsApp apology message with reorder link  | If (true branch)               | Save State of Rows in Verified & Sent | ## HTTP Request (Send WhatsApp Message) and Google Sheets (Log Verified & Sent) details                                          |
| Save State of Rows in Verified & Sent | Google Sheets          | Logs sent messages to Google Sheets                 | Send Message Using Rapiwa      | Wait                                | ## Google Sheets logging node for verified and sent messages                                                                    |
| Save State of Rows in Verified & Sent1 | Google Sheets          | Logs unverified numbers to Google Sheets            | If (false branch)              | Wait                                | ## Google Sheets logging node for unverified and not sent messages                                                              |
| Wait                             | Wait                   | Rate limiting - delays processing                    | Save State of Rows in Verified & Sent / Save State of Rows in Verified & Sent1 | Loop Over Items                      | Controls API rate limits and avoids spam detection                                                                              |
| Sticky Note                      | Sticky Note            | Documentation and overview                           | None                         | None                               | # Send WhatsApp Apology & Reorder Link When Shopify Order is Cancelled overview, links, and instructions                        |
| Sticky Note1                     | Sticky Note            | Documentation for Shopify Trigger and SplitInBatches | None                         | None                               | ## Shopify Trigger and SplitInBatches explanation                                                                               |
| Sticky Note2                     | Sticky Note            | Documentation for phone cleaning and Rapiwa verification | None                         | None                               | ## Code (Clean WhatsApp Number) and HTTP Request (Rapiwa Verify) explanation                                                    |
| Sticky Note3                     | Sticky Note            | Documentation for IF node, message sending, and logging | None                         | None                               | ## IF, HTTP Request (Send Message), and Google Sheets logging detailed explanations                                             |
| Sticky Note5                     | Sticky Note            | Usage instructions and setup steps                   | None                         | None                               | ## How to Use This Workflow instructions                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger node**  
   - Type: Shopify Trigger  
   - Set Topic: `orders/cancelled`  
   - Authenticate with Shopify Access Token credentials (API key for your store)  
   - Position: Start of workflow

2. **Add SplitInBatches node**  
   - Type: SplitInBatches  
   - Connect Shopify Trigger output to this node input  
   - Use default batch size or adjust as per rate limits

3. **Add Code node “Simplify Order Data”**  
   - Type: Code  
   - Connect SplitInBatches output to this node  
   - Paste JavaScript code to extract and format order and customer data (see Node Details 1.2)  
   - Verify fallback handling for missing fields

4. **Add Code node “Clean WhatsApp Number”**  
   - Type: Code  
   - Connect previous Code node output here  
   - Use JavaScript code to strip non-digit characters from billing phone number  
   - Ensure output retains full simplified order object with cleaned phone

5. **Add HTTP Request node “Check valid whatsapp number Using Rapiwa”**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Authentication: HTTP Bearer Auth with Rapiwa API token credentials  
   - Body Parameters: JSON with property `"number": cleaned phone number from previous node`  
   - Connect to “Clean WhatsApp Number” node output

6. **Add IF node**  
   - Type: If  
   - Condition: Check if `$json.data.exists === true` (boolean strict)  
   - Connect HTTP Request node output to IF node input  
   - True branch: verified phone number; False branch: unverified

7. **Add HTTP Request node “Send Message Using Rapiwa”** (True branch)  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Authentication: HTTP Bearer Auth (same Rapiwa token)  
   - Body Parameters: JSON including:  
     - `"number"`: cleaned billing phone number  
     - `"message_type"`: `"text"`  
     - `"message"`: personalized apology text with customer name and reorder link, e.g.:  
       ```
       Dear {{ customer.name }},
       We’re really sorry about the issue with your order. 🙏
       You can re-order using this link:👉 {{ orderUrl }}
       We’ve added a small discount for the inconvenience.
       Thanks,
       Team SpaGreen Creative
       ```  
   - Connect IF true output to this node

8. **Add Google Sheets node “Save State of Rows in Verified & Sent”** (after sending message)  
   - Type: Google Sheets Append operation  
   - Connect output from “Send Message Using Rapiwa” node  
   - Authenticate with Google Sheets OAuth2 credentials  
   - Configure sheet ID and tab (e.g., Sheet1)  
   - Map columns: customer name, email, price (currency+amount), status = "sent", product title, phone number, billing address, validity = "verified", reorder link

9. **Add Google Sheets node “Save State of Rows in Verified & Sent1”** (False branch)  
   - Type: Google Sheets Append operation  
   - Connect IF false output here  
   - Same Google Sheet as above or different tab based on your preference  
   - Map columns similarly but set status = "not sent" and validity = "unverified"

10. **Add Wait node**  
    - Type: Wait  
    - Connect outputs of both Google Sheets nodes to this node  
    - Configure delay time as needed to respect API rate limits (e.g., several seconds)

11. **Connect Wait node output back to SplitInBatches “Loop Over Items”** node to continue processing next batch.

12. **Add Sticky Notes** (optional) to document the workflow steps for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow listens for Shopify order cancellations to automate WhatsApp apologies with reorder links, verifying numbers via Rapiwa and logging all communications in Google Sheets. Respect Rapiwa API rate limits to avoid bans.                                     | Workflow overview and best practices                                                                |
| Google Sheet template required with columns: name, number, email, address1, price, title, reorder link, validity, status. Sample sheet: [Google Sheet Sample](https://docs.google.com/spreadsheets/d/12zMJod9s3Ov0RZh-7-ZoqTvJr4yMIqOeAQKTBeZlrRk/edit?usp=sharing) | Google Sheets logging setup                                                                         |
| Rapiwa API endpoints used: `/api/verify-whatsapp` to verify phone number and `/api/send-message` to send WhatsApp messages. Requires Bearer token authentication.                                                                                                   | Official Rapiwa API docs: [https://docs.rapiwa.com](https://docs.rapiwa.com/)                        |
| Ensure phone numbers are cleaned to a numeric string with country code (e.g., `88017xxxxxxx`) before verification and sending.                                                                                                                                     | Phone number formatting best practice                                                              |
| Message templates should comply with WhatsApp Business policies; avoid spam. Personalize message with customer name and order links.                                                                                                                              | Messaging compliance and personalization instructions                                               |
| For support or community help, visit: WhatsApp chat [Chat on WhatsApp](https://wa.me/8801322827799), Discord [SpaGreen Community](https://discord.gg/SsCChWEP), Facebook Group [SpaGreen Support](https://www.facebook.com/groups/spagreenbd)                      | Support links                                                                                      |
| Workflow requires Shopify Access Token, Google Sheets OAuth2, and Rapiwa API credentials properly configured in n8n.                                                                                                                                            | Credential setup instructions                                                                       |

---

**Disclaimer:**  
The provided content is extracted exclusively from an n8n automation workflow JSON. It complies with all applicable content policies and contains no illegal or offensive material. All processed data is assumed legal and public.