Send WooCommerce Cross-Sell Offers to Customers via WhatsApp Using Rapiwa API

https://n8nworkflows.xyz/workflows/send-woocommerce-cross-sell-offers-to-customers-via-whatsapp-using-rapiwa-api-10141


# Send WooCommerce Cross-Sell Offers to Customers via WhatsApp Using Rapiwa API

### 1. Workflow Overview

This n8n workflow automates sending personalized cross-sell offers to WooCommerce customers via WhatsApp using the Rapiwa API. It identifies each customer's most frequently purchased product, finds a related or latest product in that category, verifies the customer's phone number for WhatsApp availability, sends a templated promotional message, and logs the results to Google Sheets for tracking. The process is controlled with batching and waiting nodes to prevent rate limiting or spamming.

Logical blocks:

- **1.1 Scheduled Trigger & Customer Retrieval:** Starts the workflow on schedule and fetches all WooCommerce customers.
- **1.2 Filter Paying Customers:** Filters customers to those who have made purchases.
- **1.3 Customer Order & Product Analysis:** For each paying customer, fetches orders, identifies the most purchased product, cleans phone number, and retrieves detailed product info.
- **1.4 Cross-Sell Product Recommendation:** Fetches the latest product from the same category as the customer's favorite product.
- **1.5 WhatsApp Number Verification:** Checks if the customer's phone number is WhatsApp-enabled via Rapiwa API.
- **1.6 Conditional Messaging & Logging:** Sends WhatsApp promotional message if verified; logs results (sent or not sent) to Google Sheets.
- **1.7 Rate Control:** Applies waiting periods between processing customers to control request rate.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Customer Retrieval

- **Overview:** Initiates the workflow at regular intervals and retrieves all WooCommerce customers.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get many customers

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `scheduleTrigger`  
    - Role: Starts workflow execution on a time-based schedule (default interval configured).  
    - Configuration: Interval timer (default unspecified but set to repeat).  
    - Connections: Output → Get many customers.  
    - Edge cases: Trigger misconfiguration may cause no runs; timezone considerations.  

  - **Get many customers**  
    - Type: `wooCommerce` (resource: customer, operation: getAll)  
    - Role: Retrieves all customers from WooCommerce API, returning all entries.  
    - Configuration: No filters; returnAll enabled to fetch all customers without pagination limits.  
    - Credentials: WooCommerce API with read access.  
    - Connections: Output → Code (get paying_customer).  
    - Edge cases: API authentication errors, large datasets causing timeouts; WooCommerce API rate limits.

#### 1.2 Filter Paying Customers

- **Overview:** Filters the retrieved customers to those who have made purchases (paying customers).
- **Nodes Involved:**  
  - Code (get paying_customer)  

- **Node Details:**

  - **Code (get paying_customer)**  
    - Type: `code` (JavaScript)  
    - Role: Filters customers who either have `is_paying_customer` true or `total_spent` > 0.  
    - Key Logic: Iterates all input items, checks purchase-related fields, returns filtered array.  
    - Connections: Output → Loop Over Items.  
    - Edge cases: Missing or malformed fields, empty customer list, incorrect field types causing parseFloat failure.

#### 1.3 Customer Order & Product Analysis

- **Overview:** For each paying customer, retrieves order data, identifies the most purchased product, cleans the customer’s phone number, and gets detailed product info.
- **Nodes Involved:**  
  - Loop Over Items  
  - get customer data  
  - get most buy product id & Clear Number (Code)  
  - get specific product data  
  - Code (clean data)

- **Node Details:**

  - **Loop Over Items**  
    - Type: `splitInBatches`  
    - Role: Processes customers one by one or in controlled batches to avoid API overload.  
    - Configuration: Default batch options; no specific batch size defined.  
    - Connections: Output → get customer data (on batch item).  
    - Edge cases: Batch size too large causing rate limits, batch failure halting workflow.

  - **get customer data**  
    - Type: `httpRequest` (WooCommerce endpoint)  
    - Role: Fetches all orders belonging to the current customer by ID.  
    - Configuration: Uses WooCommerce API endpoint `/orders?customer={{ $json.id }}` with WooCommerce credentials.  
    - Connections: Output → get most buy product id & Clear Number.  
    - Edge cases: API call failure, empty orders, invalid customer IDs, auth errors.

  - **get most buy product id & Clear Number (Code)**  
    - Type: `code` (JavaScript)  
    - Role:  
      - Cleans phone number by removing all non-digit characters.  
      - Identifies the product with the highest quantity purchased across all orders.  
      - Extracts customer info (name, email, address, cleaned phone).  
    - Key expressions: regex replacement for phone cleaning, loops over line items to find top product by quantity.  
    - Connections: Output → get specific product data.  
    - Edge cases: Missing phone or order data, multiple products with same quantity, phone numbers in unexpected formats.

  - **get specific product data**  
    - Type: `httpRequest` (WooCommerce endpoint)  
    - Role: Retrieves detailed info about the most purchased product (product ID from previous node).  
    - Configuration: WooCommerce API call to `/products/{{ $json.product_id }}` with authentication.  
    - Connections: Output → Code (clean data).  
    - Edge cases: Product not found, API failure, variations not handled.

  - **Code (clean data)**  
    - Type: `code` (JavaScript)  
    - Role: Extracts and reformats product category information for later recommendation logic.  
    - Output: Array of objects containing product_id, product_name, and categories.  
    - Connections: Output → get specific product recommend latest product.  
    - Edge cases: Missing category data, unexpected JSON structure.

#### 1.4 Cross-Sell Product Recommendation

- **Overview:** Finds the latest product in the same category as the customer's favorite product to recommend as a cross-sell offer.
- **Nodes Involved:**  
  - get specific product recommend latest product

- **Node Details:**

  - **get specific product recommend latest product**  
    - Type: `httpRequest` (WooCommerce endpoint)  
    - Role: Fetches the newest product in the first category of the customer's favorite product, ordered by date descending.  
    - Configuration: API call to `/products?category={{ $json.categories[0].id }}&orderby=date&order=desc&per_page=1`  
    - Connections: Output → Check valid whatsapp number Using Rapiwa.  
    - Edge cases: Category ID missing, no recent products, API failure.

#### 1.5 WhatsApp Number Verification

- **Overview:** Verifies if the customer's cleaned phone number is registered on WhatsApp using the Rapiwa verification API.
- **Nodes Involved:**  
  - Check valid whatsapp number Using Rapiwa

- **Node Details:**

  - **Check valid whatsapp number Using Rapiwa**  
    - Type: `httpRequest`  
    - Role: Sends a POST request to Rapiwa’s WhatsApp verification endpoint to check number validity.  
    - Configuration: POST to `https://app.rapiwa.com/api/verify-whatsapp` with JSON body containing the phone number; HTTP Bearer authentication with Rapiwa token.  
    - Input Expression: Number passed from previous node’s cleaned phone field.  
    - Connections: Output → If (decision node).  
    - Edge cases: Network errors, invalid token, malformed phone number, API limits or downtime.

#### 1.6 Conditional Messaging & Logging

- **Overview:** Sends WhatsApp message if number verified, logs results to Google Sheets; otherwise logs unverified attempt. Adds delay before next iteration.
- **Nodes Involved:**  
  - If (decision node)  
  - Rapiwa Sender  
  - Save State of Rows in Verified & Sent (Google Sheets)  
  - Save State of Rows in Unerified & Not sent (Google Sheets)  
  - Wait

- **Node Details:**

  - **If**  
    - Type: `if`  
    - Role: Routes workflow based on Rapiwa verification response (`data.exists === true`).  
    - Connections: True → Rapiwa Sender; False → Save State of Rows in Unerified & Not sent.  
    - Edge cases: Missing `data.exists`, unexpected API response format.

  - **Rapiwa Sender**  
    - Type: `httpRequest`  
    - Role: Sends a templated WhatsApp text message with personalized cross-sell offer to verified numbers.  
    - Configuration: POST to `https://app.rapiwa.com/api/send-message`, Bearer token authentication, body includes number, message_type=text, and a templated message with customer name, product name, price, and product link.  
    - Connections: Output → Save State of Rows in Verified & Sent.  
    - Edge cases: API failure, invalid message format, rate limiting.

  - **Save State of Rows in Verified & Sent**  
    - Type: `googleSheets` (append operation)  
    - Role: Logs successful message sends with customer and product details, status "sent" and validity "verified".  
    - Configuration: Maps data fields to Google Sheet columns; uses OAuth2 credentials.  
    - Connections: Output → Wait.  
    - Edge cases: Google Sheets API errors, permission issues, rate limits.

  - **Save State of Rows in Unerified & Not sent**  
    - Type: `googleSheets` (append operation)  
    - Role: Logs unverified phone numbers with status "not sent" and validity "unverified".  
    - Configuration: Same columns as above but different status/validity values.  
    - Connections: Output → Wait.  
    - Edge cases: Same as above.

  - **Wait**  
    - Type: `wait`  
    - Role: Adds delay to prevent spamming or API rate limit violations before processing next customer.  
    - Configuration: Default wait parameters (unspecified delay).  
    - Connections: Output → Loop Over Items (to continue batch processing).  
    - Edge cases: Delay too short or too long affecting throughput.

---

### 3. Summary Table

| Node Name                           | Node Type          | Functional Role                                        | Input Node(s)                        | Output Node(s)                                | Sticky Note                                                                                             |
|-----------------------------------|--------------------|-------------------------------------------------------|------------------------------------|-----------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Trigger                  | scheduleTrigger    | Starts workflow on schedule                            | -                                  | Get many customers                            | ## Schedule Trigger Initiates workflow at intervals                                                  |
| Get many customers                | wooCommerce        | Retrieves all WooCommerce customers                    | Schedule Trigger                   | Code (get paying_customer)                     | ## Get Many Customers Retrieves all customer records                                                  |
| Code (get paying_customer)        | code               | Filters customers with purchase history                | Get many customers                 | Loop Over Items                               | ## Code (Get Paying Customer) Filters paying customers                                               |
| Loop Over Items                  | splitInBatches     | Processes customers in batches                          | Code (get paying_customer)         | get customer data                             |                                                                                                       |
| get customer data                | httpRequest        | Fetches order data for customer                         | Loop Over Items                   | get most buy product id & Clear Number        | ## Get Customer Data Fetches detailed order history                                                  |
| get most buy product id & Clear Number | code         | Identifies top purchased product and cleans phone     | get customer data                  | get specific product data                      | ## Get Most Buy Product ID & Clear Number Analyzes orders & cleans phone                              |
| get specific product data        | httpRequest        | Retrieves detailed product info                         | get most buy product id & Clear Number | Code (clean data)                        | ## Get Specific Product Data Retrieves product details                                                |
| Code (clean data)                | code               | Extracts and formats product categories                 | get specific product data          | get specific product recommend latest product | ## Code (Clean Data) Processes product categories                                                     |
| get specific product recommend latest product | httpRequest | Finds latest product in same category                  | Code (clean data)                  | Check valid whatsapp number Using Rapiwa       | ## Get Specific Product Recommend Latest Product Finds latest product in category                     |
| Check valid whatsapp number Using Rapiwa | httpRequest | Verifies WhatsApp registration of phone number         | get specific product recommend latest product | If                                      | ## Check Valid WhatsApp Number Using Rapiwa Verifies WhatsApp registration                            |
| If                               | if                 | Routes based on WhatsApp verification result            | Check valid whatsapp number Using Rapiwa | Rapiwa Sender / Save State of Rows in Unerified & Not sent | ### If (Decision Node) Routes workflow on verification status                                        |
| Rapiwa Sender                   | httpRequest        | Sends WhatsApp cross-sell message                        | If (true)                        | Save State of Rows in Verified & Sent         | ## Rapiwa Sender Sends personalized WhatsApp message                                                  |
| Save State of Rows in Verified & Sent | googleSheets | Logs successful sends to Google Sheets                  | Rapiwa Sender                   | Wait                                         | ## Save State of Rows in Verified & Sent Logs verified & sent messages                               |
| Save State of Rows in Unerified & Not sent | googleSheets | Logs unverified numbers to Google Sheets                | If (false)                      | Wait                                         | ## Save State of Rows in Unverified & Not Sent Logs unverified & not sent messages                    |
| Wait                             | wait               | Adds delay between customer processing                   | Save State of Rows in Verified & Sent / Save State of Rows in Unverified & Not sent | Loop Over Items                     | ## Wait Adds delay to control rate limiting                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: `scheduleTrigger`  
   - Configure interval as desired (e.g., daily).  

2. **Add WooCommerce Node (Get many customers):**  
   - Type: `wooCommerce`  
   - Resource: Customer  
   - Operation: getAll  
   - Return All: true  
   - Connect Schedule Trigger output to this node.  
   - Set WooCommerce API credentials with read access.

3. **Add Code Node (get paying_customer):**  
   - Type: `code`  
   - JavaScript code filters customers where `is_paying_customer === true` or `total_spent > 0`.  
   - Connect Get many customers output to this node.

4. **Add SplitInBatches Node (Loop Over Items):**  
   - Type: `splitInBatches`  
   - Default options or set batch size (e.g., 1).  
   - Connect Code node output to this node.

5. **Add HTTP Request Node (get customer data):**  
   - Type: `httpRequest`  
   - Method: GET  
   - URL: `https://your_shop_domain/wp-json/wc/v3/orders?customer={{ $json.id }}`  
   - Credentials: WooCommerce API  
   - Connect Loop Over Items output to this node.

6. **Add Code Node (get most buy product id & Clear Number):**  
   - Type: `code`  
   - JavaScript code to:  
     - Clean phone number by removing non-digit characters.  
     - Find the product with the highest quantity in orders.  
     - Extract customer name, email, address, and cleaned phone.  
   - Connect get customer data output to this node.

7. **Add HTTP Request Node (get specific product data):**  
   - Type: `httpRequest`  
   - Method: GET  
   - URL: `https://your_shop_domain/wp-json/wc/v3/products/{{ $json.product_id }}`  
   - Credentials: WooCommerce API  
   - Connect previous Code node output to this node.

8. **Add Code Node (clean data):**  
   - Type: `code`  
   - JavaScript extracts product categories and formats them into a simplified structure.  
   - Connect get specific product data output to this node.

9. **Add HTTP Request Node (get specific product recommend latest product):**  
   - Type: `httpRequest`  
   - Method: GET  
   - URL: `https://your_shop_domain/wp-json/wc/v3/products?category={{ $json.categories[0].id }}&orderby=date&order=desc&per_page=1`  
   - Credentials: WooCommerce API  
   - Connect previous Code node output to this node.

10. **Add HTTP Request Node (Check valid whatsapp number Using Rapiwa):**  
    - Type: `httpRequest`  
    - Method: POST  
    - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
    - Authentication: HTTP Bearer with Rapiwa token (set in credentials).  
    - Body Parameters: JSON object with `"number": "{{ $('get most buy product id & Clear Number').item.json.phone }}"`.  
    - Connect previous HTTP request output to this node.

11. **Add If Node:**  
    - Type: `if`  
    - Condition: Check if `$json.data.exists === true` to confirm WhatsApp number validity.  
    - Connect Check valid whatsapp number output to If node.

12. **Add HTTP Request Node (Rapiwa Sender):**  
    - Type: `httpRequest`  
    - Method: POST  
    - URL: `https://app.rapiwa.com/api/send-message`  
    - Authentication: HTTP Bearer with Rapiwa token.  
    - Body Parameters:  
      - `number`: `{{$json.data.number}}`  
      - `message_type`: `"text"`  
      - `message`: Template using customer name, product name, price, and product link from prior nodes.  
    - Connect If node True output to this node.

13. **Add Google Sheets Node (Save State of Rows in Verified & Sent):**  
    - Type: `googleSheets`  
    - Operation: Append  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: Sheet/tab name or gid=0  
    - Columns mapped for customer/product info, status "sent", validity "verified".  
    - Credentials: Google Sheets OAuth2.  
    - Connect Rapiwa Sender output to this node.

14. **Add Google Sheets Node (Save State of Rows in Unerified & Not sent):**  
    - Type: `googleSheets`  
    - Operation: Append  
    - Same document and sheet as above.  
    - Columns mapped similarly, but status "not sent", validity "unverified".  
    - Credentials: Google Sheets OAuth2.  
    - Connect If node False output to this node.

15. **Add Wait Node:**  
    - Type: `wait`  
    - Default or configured delay (e.g., few seconds).  
    - Connect both Google Sheets nodes output to this node.

16. **Connect Wait Node output back to Loop Over Items** to continue processing next batch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| This workflow enables automated cross-selling by identifying WooCommerce customers’ most purchased products, recommending latest related products, verifying WhatsApp numbers via Rapiwa, sending templated messages, and logging results in Google Sheets.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Workflow purpose overview                                                                                                         |
| Google Sheet sample template provided for logging data: [Sample Sheet](https://docs.google.com/spreadsheets/d/1yTxkAM1kq2udndvG5M2jUuQCY8SkvoLgrspQ100LvKs/edit?usp=sharing)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Google Sheets structure example                                                                                                  |
| Rapiwa API documentation and dashboard: [https://docs.rapiwa.com](https://docs.rapiwa.com/), [https://app.rapiwa.com](https://app.rapiwa.com/login)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | API and account resources                                                                                                        |
| Phone number format cleaning is critical: must produce valid E.164 formatted numbers suitable for WhatsApp verification.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Data preparation note                                                                                                            |
| Rate limiting and API quota should be monitored carefully for both WooCommerce and Rapiwa APIs to avoid temporary bans or workflow failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Operational caution                                                                                                              |
| Testing with a small number of users before full deployment is strongly recommended to validate message formatting, API responses, and Google Sheets logging.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Recommended testing practice                                                                                                    |
| Support and community links: WhatsApp chat, Discord, Facebook group, developer portfolio.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Community and support resources                                                                                                 |
| Workflow designed for n8n versions supporting OAuth2 credentials, HTTP Bearer Auth, and JavaScript code nodes as shown; tested with n8n 1.x and above.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Version compatibility note                                                                                                      |

---

**Disclaimer:** The text provided originates exclusively from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.