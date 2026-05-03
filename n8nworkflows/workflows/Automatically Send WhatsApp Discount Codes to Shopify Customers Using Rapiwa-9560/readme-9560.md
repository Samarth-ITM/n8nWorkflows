Automatically Send WhatsApp Discount Codes to Shopify Customers Using Rapiwa

https://n8nworkflows.xyz/workflows/automatically-send-whatsapp-discount-codes-to-shopify-customers-using-rapiwa-9560


# Automatically Send WhatsApp Discount Codes to Shopify Customers Using Rapiwa

### 1. Workflow Overview

This workflow is designed to **automatically send WhatsApp discount codes to selected Shopify customers** when a new discount code is created in the Shopify store. It targets merchants who want to engage high-value customers via WhatsApp with promotional discount codes verified through Rapiwa, a WhatsApp verification and messaging API.

The key logical blocks are:

- **1.1 Input Reception:** Receive Shopify webhook notification about newly created discount codes.
- **1.2 Discount Data Cleanup:** Extract and clean essential discount code data from the webhook.
- **1.3 Customer Data Retrieval and Filtering:** Fetch all Shopify customers and filter for high-spending customers.
- **1.4 Customer Data Processing Loop:** Process each filtered customer individually.
- **1.5 WhatsApp Number Cleaning and Verification:** Clean customer phone numbers and verify WhatsApp validity using Rapiwa.
- **1.6 Conditional Messaging and Logging:** If verified, send WhatsApp discount message and log as sent; if unverified, log as not sent.
- **1.7 Throttling:** Introduce wait time between iterations to respect API limits.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception

- **Overview:**  
Receives the webhook POST request triggered by Shopify when a new discount code is created.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point for the workflow, listens for HTTP POST requests at a unique path.  
    - Config: HTTP method POST, unique webhook path set.  
    - Inputs: External Shopify webhook POST requests.  
    - Outputs: Passes received webhook data downstream.  
    - Edge cases: Invalid or malformed webhook requests might cause failure or no trigger; webhook path must be correctly configured in Shopify.

---

#### Block 1.2: Discount Data Cleanup

- **Overview:**  
Extracts relevant discount details from the Shopify webhook payload and normalizes them for later use.

- **Nodes Involved:**  
  - Clean Webhooks Response Data (Code)

- **Node Details:**  
  - **Clean Webhooks Response Data**  
    - Type: Code  
    - Role: Parses incoming webhook JSON to extract discount ID, title, status, timestamps, and shop domain.  
    - Config: JavaScript code accessing `item.json.body` and headers to retrieve fields.  
    - Inputs: Output from Webhook node.  
    - Outputs: Simplified JSON object with selected discount details.  
    - Edge cases: Missing fields or unexpected payload structure could cause undefined values or errors.

---

#### Block 1.3: Customer Data Retrieval and Filtering

- **Overview:**  
Fetches all customers from Shopify store via API, then filters for customers with total spent over 5000, extracting relevant details.

- **Nodes Involved:**  
  - Get All Customer Data In Shopify Store (HTTP Request)  
  - Clean Customer Data In Shopify Store (Code)

- **Node Details:**  
  - **Get All Customer Data In Shopify Store**  
    - Type: HTTP Request  
    - Role: GET customers list from Shopify REST API.  
    - Config: URL with Shopify API endpoint for customers, includes X-Shopify-Access-Token header for authentication.  
    - Inputs: Output from discount data cleanup node.  
    - Outputs: Raw customers list JSON.  
    - Edge cases: API rate limits, invalid or expired access tokens, network issues.  
    - Requirements: Proper Shopify access token with read_customers scope.

  - **Clean Customer Data In Shopify Store**  
    - Type: Code  
    - Role: Filters customers with total spent > 5000, formats each customer with name, email, phone, address, and order count.  
    - Config: JavaScript code parsing customers array, checking `total_spent`, and mapping fields.  
    - Inputs: Output from HTTP Request node.  
    - Outputs: Array of filtered customer objects.  
    - Edge cases: Empty or malformed customer data, missing phone numbers.

---

#### Block 1.4: Customer Data Processing Loop

- **Overview:**  
Splits filtered customer list into individual or batch items for sequential processing.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)

- **Node Details:**  
  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over customer list one by one or in batches for controlled processing.  
    - Config: Default batch size (not explicitly set, defaults to 1).  
    - Inputs: Filtered customer list.  
    - Outputs: Single customer records downstream.  
    - Edge cases: Empty customer list halts looping; large batches may risk API limits.

---

#### Block 1.5: WhatsApp Number Cleaning and Verification

- **Overview:**  
Cleans the customer’s WhatsApp number by removing non-digit characters, then verifies if the number is valid on WhatsApp using Rapiwa API.

- **Nodes Involved:**  
  - Clean WhatsApp Number (Code)  
  - Check valid whatsapp number Using Rapiwa (HTTP Request)

- **Node Details:**  
  - **Clean WhatsApp Number**  
    - Type: Code  
    - Role: Sanitizes phone number by stripping non-digit characters to ensure correct format for API calls.  
    - Config: JavaScript replacing all non-digits with empty string.  
    - Inputs: Single customer object from loop.  
    - Outputs: Customer object with cleaned number.  
    - Edge cases: Missing or null phone number fields; cleaning may result in empty strings.

  - **Check valid whatsapp number Using Rapiwa**  
    - Type: HTTP Request  
    - Role: Calls Rapiwa’s WhatsApp verification endpoint with cleaned number.  
    - Config: POST request to `https://app.rapiwa.com/api/verify-whatsapp` with bearer token authentication.  
    - Inputs: Cleaned phone number.  
    - Outputs: JSON response including `data.exists` boolean indicating WhatsApp validity.  
    - Edge cases: Invalid bearer token, network errors, API quota exceeded.  
    - Requirements: Rapiwa API Bearer token credential configured.

---

#### Block 1.6: Conditional Messaging and Logging

- **Overview:**  
Branches logic based on WhatsApp verification result: if verified, send WhatsApp message and log success; if not, log failure.

- **Nodes Involved:**  
  - If (Boolean check)  
  - Send Message Using Rapiwa (HTTP Request)  
  - Save data in Sheet Verified & Sent (Google Sheets)  
  - Save data Sheet Unverified & Not sent (Google Sheets)

- **Node Details:**  
  - **If**  
    - Type: If condition  
    - Role: Checks if phone number is verified on WhatsApp (`data.exists === true`).  
    - Config: Expression comparing `$json.data.exists` to true (boolean).  
    - Inputs: Output of Rapiwa verification HTTP Request.  
    - Outputs: Two branches: true (verified) and false (unverified).  
    - Edge cases: Missing or malformed API response; expression evaluation errors.

  - **Send Message Using Rapiwa**  
    - Type: HTTP Request  
    - Role: Sends personalized WhatsApp text message with discount code to verified number via Rapiwa API.  
    - Config: POST request to `https://app.rapiwa.com/api/send-message`, bearer token authentication, message built with customer name and discount title interpolated.  
    - Inputs: Verified branch from If node; uses cleaned customer info and discount code data.  
    - Outputs: Confirmation or error response from Rapiwa message API.  
    - Edge cases: Message send failures, API limits, invalid number despite verification, auth errors.

  - **Save data in Sheet Verified & Sent**  
    - Type: Google Sheets (Append)  
    - Role: Logs details of customers with verified WhatsApp numbers and successfully sent messages.  
    - Config: Appends row with fields: customer name, discount title, phone number, status “sent”, verify “verified”, discount metadata, timestamps.  
    - Inputs: Output of Send Message node.  
    - Outputs: Confirmation of append operation.  
    - Edge cases: Google Sheets API errors, insufficient permissions, quota exceeded.  
    - Requirements: Google Sheets OAuth2 credentials configured with sheet access.

  - **Save data Sheet Unverified & Not sent**  
    - Type: Google Sheets (Append)  
    - Role: Logs details of customers with unverified WhatsApp numbers, marks status as “not sent”.  
    - Config: Similar to above but with verify “unverified” and status “not sent”.  
    - Inputs: Unverified branch from If node.  
    - Outputs: Confirmation of append operation.  
    - Edge cases: Same as above.

---

#### Block 1.7: Throttling

- **Overview:**  
Implements a wait/delay between iterations to prevent API rate limits and Google Sheets quota issues.

- **Nodes Involved:**  
  - Wait

- **Node Details:**  
  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for a short time between customer iterations.  
    - Config: Default delay (not explicitly set; typically configurable).  
    - Inputs: After append logging nodes.  
    - Outputs: Loops back to next customer processing iteration.  
    - Edge cases: Long delays may slow workflow; too short may risk API limits.

---

### 3. Summary Table

| Node Name                         | Node Type             | Functional Role                                      | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|----------------------------------|-----------------------|-----------------------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                          | Webhook               | Receives Shopify webhook on discount creation       |                             | Clean Webhooks Response Data   | ## 1. Webhook  Receives Shopify Webhook (discount creation) via HTTP POST request. This is triggered when a discount is created in your Shopify store.                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Clean Webhooks Response Data      | Code                  | Extracts relevant discount code data from webhook   | Webhook                     | Get All Customer Data In Shopify Store | ## 2. Clean Webhooks Response Data Extracts useful fields (`title`, `status`, `created_at`, `shop_domain`, etc.) from the incoming Shopify webhook and formats them for further use.                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Get All Customer Data In Shopify Store | HTTP Request          | Fetches all customers from Shopify API              | Clean Webhooks Response Data | Clean Customer Data In Shopify Store | ## 1. Get All Customer Data In Store Fetches all customer data from the Shopify store using your API credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Clean Customer Data In Shopify Store | Code                  | Filters high-spending customers and formats data    | Get All Customer Data In Shopify Store | Loop Over Items                | ## 2. Clean HTTP Request Data (Filter & Format Customers) Filters customers whose total spent is greater than 5000, and extracts customer details.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Loop Over Items                  | SplitInBatches        | Processes customers one at a time                    | Clean Customer Data In Shopify Store | Clean WhatsApp Number          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Clean WhatsApp Number            | Code                  | Cleans phone numbers by removing non-digit chars    | Loop Over Items             | Check valid whatsapp number Using Rapiwa | ## 1. Clean WhatsApp Number Cleans the WhatsApp number by converting to string and removing non-digit characters.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Check valid whatsapp number Using Rapiwa | HTTP Request          | Verifies WhatsApp number via Rapiwa API             | Clean WhatsApp Number       | If                            | ## 2. Check Valid WhatsApp Number Using Rapiwa API endpoint: `https://app.rapiwa.com/api/verify-whatsapp` Uses the Rapiwa API to verify whether the cleaned number is associated with WhatsApp.                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| If                              | If                    | Checks if WhatsApp number is verified                | Check valid whatsapp number Using Rapiwa | Send Message Using Rapiwa (true branch), Save data Sheet Unverified & Not sent (false branch) | ## 1. Node: If Checks if the phone number is verified as a WhatsApp number.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Send Message Using Rapiwa        | HTTP Request          | Sends WhatsApp message with discount code            | If (true branch)            | Save data in Sheet Verified & Sent | ## 2. Node: Send Message Using Rapiwa Sends a personalized discount message via WhatsApp to verified customers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Save data in Sheet Verified & Sent | Google Sheets          | Logs verified & sent message customers               | Send Message Using Rapiwa   | Wait                          | ## 3. Node: Append Rows in Sheet Verified & Sent Logs details of customers with verified WhatsApp numbers and sent messages in Google Sheet.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Save data Sheet Unverified & Not sent | Google Sheets          | Logs unverified & not sent message customers         | If (false branch)           | Wait                          | ## 4. Node: Append Rows in Sheet Unverified & Not sent Logs customers with unverified WhatsApp numbers as “not sent” in Google Sheet.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Wait                            | Wait                  | Adds delay between iterations to respect limits      | Save data in Sheet Verified & Sent, Save data Sheet Unverified & Not sent | Loop Over Items                |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Sticky Note                     | StickyNote             | Overview and instructions                            |                             |                               | # Automatically Send WhatsApp Discount Codes to Shopify Customers Using Rapiwa ... [Full detailed overview and support links]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Sticky Note1                    | StickyNote             | Explains Webhook and Discount Data Cleanup           |                             |                               | ## 1. Webhook & 2. Clean Webhooks Response Data                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Sticky Note2                    | StickyNote             | Explains customer data retrieval and filtering       |                             |                               | ## 1. Get All Customer Data In Store & 2. Clean HTTP Request Data (Filter & Format Customers)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Sticky Note4                    | StickyNote             | Explains WhatsApp number cleaning and verification   |                             |                               | ## 1. Clean WhatsApp Number & 2. Check Valid WhatsApp Number Using Rapiwa                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Sticky Note5                    | StickyNote             | Explains conditional sending and logging logic       |                             |                               | ## 1. Node: If & 2. Node: Send Message Using Rapiwa & 3. Node: Append Rows in Sheet Verified & Sent & 4. Node: Append Rows in Sheet Unverified & Not sent                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `a9b6a936-e5f2-4d4c-9cf9-182de0a970d5`)  
   - Purpose: Receive Shopify discount creation webhook.  
   - Save.

2. **Create Code Node: Clean Webhooks Response Data**  
   - Connect from Webhook node.  
   - Paste JS code to extract fields from webhook JSON, including discount ID, title, status, created_at, updated_at, and shop domain from headers.  
   - Output simplified JSON for downstream nodes.

3. **Create HTTP Request Node: Get All Customer Data In Shopify Store**  
   - Connect from Clean Webhooks Response Data node.  
   - Method: GET  
   - URL: `https://your_domain_/admin/api/2025-07/customers.json` (replace `your_domain_` with actual Shopify domain)  
   - Headers: Add `X-Shopify-Access-Token` with your Shopify Admin API access token.  
   - Save.

4. **Create Code Node: Clean Customer Data In Shopify Store**  
   - Connect from Get All Customer Data In Shopify Store node.  
   - JS code to filter customers with `total_spent > 5000` and map customer fields: id, full name, email, phone, totalSpent, ordersCount, address components.  
   - Output array of customer objects.

5. **Create SplitInBatches Node: Loop Over Items**  
   - Connect from Clean Customer Data In Shopify Store node.  
   - Configure batch size as needed (default to 1).  
   - Purpose: Process customers individually.

6. **Create Code Node: Clean WhatsApp Number**  
   - Connect from Loop Over Items node.  
   - JS code to clean phone number: convert to string if needed, remove all non-digit characters.  
   - Store cleaned number back to JSON.

7. **Create HTTP Request Node: Check valid whatsapp number Using Rapiwa**  
   - Connect from Clean WhatsApp Number node.  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Body Parameters: Pass `number` as cleaned phone number.  
   - Authentication: HTTP Bearer token with Rapiwa API key.  
   - Save.

8. **Create If Node**  
   - Connect from Check valid whatsapp number Using Rapiwa node.  
   - Condition: `$json.data.exists === true` (boolean true)  
   - Purpose: Branch workflow based on WhatsApp verification.

9. **Create HTTP Request Node: Send Message Using Rapiwa**  
   - Connect from If node’s "true" branch.  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Body Parameters:  
     - `number`: cleaned WhatsApp number  
     - `message_type`: "text"  
     - `message`: personalized message including customer name and discount title using expressions referencing previous nodes.  
   - Authentication: HTTP Bearer token with Rapiwa API key.  
   - Save.

10. **Create Google Sheets Node: Save data in Sheet Verified & Sent**  
    - Connect from Send Message Using Rapiwa node.  
    - Operation: Append  
    - Document ID: your Google Sheet ID  
    - Sheet Name: specify sheet by gid or name  
    - Columns: map customer name, discount title, number, status "sent", verify "verified", discount metadata, timestamps using expressions.  
    - Credentials: Google Sheets OAuth2 API credentials with write access.  
    - Save.

11. **Create Google Sheets Node: Save data Sheet Unverified & Not sent**  
    - Connect from If node’s "false" branch.  
    - Same configuration as above but with status "not sent" and verify "unverified".  
    - Save.

12. **Create Wait Node**  
    - Connect from both Google Sheets nodes (Verified & Unverified).  
    - Configure delay time (e.g., 2000 ms or 2 seconds).  
    - Purpose: Throttle requests to avoid API and quota limits.

13. **Connect Wait Node back to Loop Over Items node**  
    - This creates a loop allowing batch processing with delay between customers.

14. **Final checks:**  
    - Ensure all credentials (Shopify API token, Rapiwa Bearer token, Google Sheets OAuth2) are properly configured.  
    - Test webhook URL externally and verify Shopify discount creation triggers workflow.  
    - Adjust wait time and batch size if necessary to respect API limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                      | Context or Link                                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| This workflow triggers on Shopify discount code creation and targets high-value customers (total spent > 5000) for WhatsApp promotion.                                                                                                                         | Branding and workflow purpose                                                                                                 |
| Rapiwa API is used for WhatsApp number verification and message sending to ensure messages are sent only to valid WhatsApp users.                                                                                                                               | https://rapiwa.com                                                                                                            |
| Google Sheets is used to log all message sending attempts for auditing and tracking purposes.                                                                                                                                                                    | Sample Sheet: https://docs.google.com/spreadsheets/d/1Zx_WXQW29NsITFPJ-SnjHgOlouvzG_sBNGzSA_B8cSA/edit?usp=sharing             |
| Support channels include WhatsApp chat, Discord community, Facebook group, and SpaGreen developer portfolio for assistance and updates.                                                                                                                        | WhatsApp: https://wa.me/8801322827799; Discord: https://discord.gg/SsCChWEP; Facebook: https://www.facebook.com/groups/spagreenbd; Codecanyon: https://codecanyon.net/user/spagreen/portfolio |
| Ensure API tokens and credentials are securely stored and have appropriate permissions (Shopify Admin API with customers scope, Rapiwa API keys, Google Sheets API with edit permissions).                                                                          | Security best practices                                                                                                       |
| Adjust batch size and wait duration to optimize throughput without hitting API rate limits or quota.                                                                                                                                                            | Performance tuning                                                                                                            |
| This workflow requires n8n version supporting HTTP Request v4.2+, Google Sheets v4.6+, and Code node v2 for compatibility with expressions and JavaScript features.                                                                                              | Version compatibility                                                                                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow built with n8n integration and automation tool. The processing fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.