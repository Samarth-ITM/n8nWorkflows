Automatic Magento 2 Product & Coupon Alerts to Telegram with Duplicate Protection

https://n8nworkflows.xyz/workflows/automatic-magento-2-product---coupon-alerts-to-telegram-with-duplicate-protection-5893


# Automatic Magento 2 Product & Coupon Alerts to Telegram with Duplicate Protection

### 1. Workflow Overview

This workflow automates alerts for new Magento 2 products and coupon vouchers, posting updates to Telegram (and optionally X/Twitter) channels. It ensures alerts are only sent once per item by maintaining a MySQL database that tracks posted products and coupons, thereby preventing duplicate notifications.

Logical blocks:

- **1.1 Auto Schedule Trigger & DB Initialization**: Periodically triggers the workflow and ensures the tracking database table exists.
- **1.2 New Voucher Processing**: Fetches the latest coupon, checks if it’s already posted, retrieves detailed coupon info, formats the message, posts it to Telegram, and marks it as posted.
- **1.3 New Product Processing**: Fetches newest products, checks duplicates, retrieves product details, formats messages, posts to Telegram (and optionally X), and marks as posted.
- **1.4 Duplication Protection**: Conditional checks based on database flags to avoid reposting the same product or coupon.
- **1.5 Message Formatting and Posting**: Prepares user-friendly alert messages and handles Telegram/X posting.
- **1.6 Database Update**: Marks items as posted after successful alerting.

---

### 2. Block-by-Block Analysis

#### 2.1 Auto Schedule Trigger & DB Initialization

**Overview:**  
This block triggers the workflow every hour and ensures the MySQL table for tracking posted items exists.

**Nodes Involved:**  
- Schedule Trigger  
- Init Database

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Runs every hour (`interval` set to hours)  
  - Inputs: None (trigger node)  
  - Outputs: To Get Latest Offer, Init Database, Fetch New Product  
  - Potential failures: Scheduling misconfiguration, n8n system downtime

- **Init Database**  
  - Type: MySQL node  
  - Configuration: Executes a SQL query to create table `posted_items` if it does not exist, with columns for item ID, type (product or coupon), value, posted flag, and timestamps  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: None (setup only)  
  - Potential failures: Database connection/authentication issues, SQL syntax errors

---

#### 2.2 New Voucher Processing

**Overview:**  
Fetches the latest coupon voucher, checks if it has already been posted, retrieves detailed coupon info, formats a Telegram message, posts the message, and updates the database to mark it posted.

**Nodes Involved:**  
- Get Latest Offer  
- New Voucher Entry  
- Voucher Duplication Protection  
- Get Rule Info  
- Coupon Status  
- Voucher Message Format  
- Post to Telegram1  
- Set Coupon as Posted

**Node Details:**

- **Get Latest Offer**  
  - Type: HTTP Request  
  - Configuration: Calls Magento 2 REST API endpoint `/rest/V1/coupons/search` to get the newest coupon, sorted by creation date descending, page size 1  
  - Authentication: Generic HTTP Header Auth (custom credentials)  
  - Inputs: Schedule Trigger  
  - Outputs: New Voucher Entry  
  - Potential failures: API endpoint downtime, auth failure, network issues

- **New Voucher Entry**  
  - Type: MySQL node  
  - Configuration: Inserts new coupon into `posted_items` table if not existing, with `posted` set to FALSE  
  - Uses SQL with conditional insert (WHERE NOT EXISTS)  
  - Inputs: Get Latest Offer  
  - Outputs: Voucher Duplication Protection  
  - Potential failures: DB connection, SQL syntax, concurrent inserts

- **Voucher Duplication Protection**  
  - Type: If node  
  - Configuration: Checks if `posted` field equals 1 (true) for the coupon entry  
  - Inputs: New Voucher Entry  
  - Outputs: If posted=1, stops further processing; else continues to Get Rule Info  
  - Edge cases: Missing `posted` field in DB response

- **Get Rule Info**  
  - Type: HTTP Request  
  - Configuration: Calls Magento 2 REST API `/rest/V1/salesRules/{{ item_id }}` with the rule ID from JSON to get detailed coupon info  
  - Authentication: Same as Get Latest Offer  
  - Inputs: Voucher Duplication Protection (false branch)  
  - Outputs: Coupon Status  
  - Potential failures: API errors, invalid rule IDs

- **Coupon Status**  
  - Type: If node  
  - Configuration: Checks if coupon is active (`is_active` field true)  
  - Inputs: Get Rule Info  
  - Outputs: Voucher Message Format  
  - Edge cases: Missing or malformed `is_active`

- **Voucher Message Format**  
  - Type: Code node (JavaScript)  
  - Configuration: Formats a multi-line coupon alert message with coupon code, usage stats, and status, using data from Get Latest Offer and Get Rule Info nodes via expressions  
  - Inputs: Coupon Status  
  - Outputs: Post to Telegram1, Voucher to X (Twitter, disabled)  
  - Potential failures: Expression errors if expected fields missing

- **Post to Telegram1**  
  - Type: Telegram node  
  - Configuration: Sends the coupon message text to configured Telegram chat ID with HTML parse mode  
  - Inputs: Voucher Message Format  
  - Outputs: Set Coupon as Posted  
  - Potential failures: Telegram API errors, invalid chat ID, network issues

- **Set Coupon as Posted**  
  - Type: MySQL node  
  - Configuration: Updates `posted` flag to 1 for the coupon in `posted_items` table using rule ID  
  - Inputs: Post to Telegram1, Voucher to X (disabled)  
  - Outputs: None  
  - Potential failures: DB connection issues, SQL errors

---

#### 2.3 New Product Processing

**Overview:**  
Fetches newest products, inserts into DB if new, skips if already posted, retrieves detailed product info, formats messages, posts alerts to Telegram and X (Twitter), and marks products as posted.

**Nodes Involved:**  
- Fetch New Product  
- New Product Entry  
- Product Duplication Protection  
- Get Product Info  
- Product Status  
- Product Message Format  
- Product Alert to Telegram  
- Product X Post  
- Set Product as Posted

**Node Details:**

- **Fetch New Product**  
  - Type: HTTP Request  
  - Configuration: Calls Magento REST API `/rest/V1/products` with sort by `created_at` descending to get newest products  
  - Authentication: Header Auth with store variables  
  - Inputs: Schedule Trigger  
  - Outputs: New Product Entry  
  - Potential failures: API errors, network, auth

- **New Product Entry**  
  - Type: MySQL node  
  - Configuration: Inserts product into `posted_items` if not existing, with `posted` FALSE  
  - Uses SKU and product ID for uniqueness  
  - Inputs: Fetch New Product  
  - Outputs: Product Duplication Protection  
  - Potential failures: DB errors

- **Product Duplication Protection**  
  - Type: If node  
  - Configuration: Checks if `posted` = 1 for product; if yes, end flow, else continue  
  - Inputs: New Product Entry  
  - Outputs: If false branch to Get Product Info  
  - Edge cases: Missing DB data

- **Get Product Info**  
  - Type: HTTP Request  
  - Configuration: Gets detailed product info by SKU from Magento REST API `/rest/default/V1/products/{{ item_value }}`  
  - Inputs: Product Duplication Protection (false branch)  
  - Outputs: Product Status  
  - Potential failures: API errors, invalid SKU

- **Product Status**  
  - Type: If node  
  - Configuration: Checks if product `status` is true (active)  
  - Inputs: Get Product Info  
  - Outputs: Product Message Format  
  - Edge cases: Missing or invalid status

- **Product Message Format**  
  - Type: Code node (JavaScript)  
  - Configuration: Constructs a message including product name, price, and link, also retrieves product image URL if available  
  - Inputs: Product Status  
  - Outputs: Product Alert to Telegram, Product X Post  
  - Potential failures: Missing fields, expression errors

- **Product Alert to Telegram**  
  - Type: Telegram node  
  - Configuration: Sends a media group message to Telegram with product image and caption (formatted message)  
  - Inputs: Product Message Format  
  - Outputs: Set Product as Posted  
  - Potential failures: Telegram API errors, invalid chat ID

- **Product X Post**  
  - Type: Twitter node (X)  
  - Configuration: Sends product image and message text to Twitter account  
  - Inputs: Product Message Format  
  - Outputs: Set Product as Posted  
  - Potential failures: Twitter API errors, disabled node by default

- **Set Product as Posted**  
  - Type: MySQL node  
  - Configuration: Updates `posted` flag to 1 for product in DB  
  - Inputs: Product Alert to Telegram, Product X Post  
  - Outputs: None  
  - Potential failures: DB errors

---

#### 2.4 Duplication Protection

**Overview:**  
If nodes that check the `posted` flag in the database to prevent reposting already alerted products or coupons.

**Nodes Involved:**  
- Voucher Duplication Protection (If node)  
- Product Duplication Protection (If node)

**Node Details:**

- Both nodes check if the `posted` field in the database equals 1 (posted). If yes, they terminate the flow branch to avoid duplicates. If no, processing continues.

---

#### 2.5 Message Formatting and Posting

**Overview:**  
Code nodes that format alert messages for coupons and products, followed by Telegram and optional Twitter posting nodes.

**Nodes Involved:**  
- Voucher Message Format  
- Product Message Format  
- Post to Telegram1  
- Product Alert to Telegram  
- Voucher to X (disabled)  
- Product X Post

**Node Details:**

- Message formats use JavaScript code to create richly formatted texts with emoji, HTML tags (coupon messages), and product links/images.  
- Telegram nodes send text or media groups with captions to Telegram chats, requiring valid chat IDs and bot credentials.  
- Twitter nodes post messages/images to X/Twitter accounts; one is disabled by default.  
- Possible failures include API limits, auth errors, invalid chat IDs, or malformed messages.

---

#### 2.6 Database Update

**Overview:**  
Updates the MySQL database marking items as posted after successful notification.

**Nodes Involved:**  
- Set Coupon as Posted  
- Set Product as Posted

**Node Details:**

- Execute UPDATE SQL queries setting `posted = 1` for the given item IDs and types.  
- Inputs from corresponding posting nodes.  
- Critical for preventing duplicate alerts.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                          | Input Node(s)              | Output Node(s)                      | Sticky Note                                                                                     |
|-----------------------------|---------------------|----------------------------------------|----------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger    | Periodic trigger to start workflow     | -                          | Get Latest Offer, Init Database, Fetch New Product | ## Auto Schedule Trigger & Initialise DB                                                        |
| Init Database              | MySQL               | Create tracking DB table if missing    | Schedule Trigger           | -                                 | ## Auto Schedule Trigger & Initialise DB                                                        |
| Get Latest Offer           | HTTP Request        | Fetch latest coupon voucher             | Schedule Trigger           | New Voucher Entry                 | ## Magento 2 - Checks for Latest Voucher - Record in Database - Validates                       |
| New Voucher Entry          | MySQL               | Insert new coupon record if new         | Get Latest Offer           | Voucher Duplication Protection    | ## Magento 2 - Checks for Latest Voucher - Record in Database - Validates                       |
| Voucher Duplication Protection | If                 | Check if coupon already posted          | New Voucher Entry          | Get Rule Info (if not posted)     | ## Magento 2 - Checks for Latest Voucher - Record in Database - Validates                       |
| Get Rule Info              | HTTP Request        | Get detailed coupon info                 | Voucher Duplication Protection | Coupon Status                 | ## Magento 2 - Checks for Latest Voucher - Record in Database - Validates                       |
| Coupon Status              | If                  | Check if coupon is active                | Get Rule Info              | Voucher Message Format            | ## Magento 2 - Checks for Latest Voucher - Record in Database - Validates                       |
| Voucher Message Format     | Code                 | Format coupon alert message              | Coupon Status              | Post to Telegram1, Voucher to X   | ## Format Messages                                                                             |
| Post to Telegram1          | Telegram             | Post coupon message to Telegram          | Voucher Message Format     | Set Coupon as Posted              | ## Post to X & Telegram                                                                        |
| Set Coupon as Posted       | MySQL                | Mark coupon as posted in DB               | Post to Telegram1, Voucher to X | -                             | ## Record as Posted                                                                           |
| Fetch New Product          | HTTP Request        | Fetch newest products                     | Schedule Trigger           | New Product Entry                 | ## Magento 2 - Checks for New Product - Record in Database - Validates                          |
| New Product Entry          | MySQL               | Insert new product record if new          | Fetch New Product          | Product Duplication Protection    | ## Magento 2 - Checks for New Product - Record in Database - Validates                          |
| Product Duplication Protection | If                 | Check if product already posted           | New Product Entry          | Get Product Info (if not posted)  | ## Magento 2 - Checks for New Product - Record in Database - Validates                          |
| Get Product Info           | HTTP Request        | Get detailed product info                  | Product Duplication Protection | Product Status                 | ## Magento 2 - Checks for New Product - Record in Database - Validates                          |
| Product Status             | If                  | Check if product is active                 | Get Product Info           | Product Message Format            | ## Magento 2 - Checks for New Product - Record in Database - Validates                          |
| Product Message Format     | Code                 | Format product alert message               | Product Status             | Product Alert to Telegram, Product X Post | ## Format Messages                                                                     |
| Product Alert to Telegram  | Telegram             | Post product alert with image to Telegram | Product Message Format     | Set Product as Posted             | ## Post to X & Telegram                                                                        |
| Product X Post             | Twitter (X)          | Post product alert to Twitter (disabled)  | Product Message Format     | Set Product as Posted             | ## Post to X & Telegram                                                                        |
| Set Product as Posted      | MySQL                | Mark product as posted in DB                | Product Alert to Telegram, Product X Post | -                         | ## Record as Posted                                                                           |
| Voucher to X               | Twitter (X)          | Post coupon alert to Twitter (disabled)    | Voucher Message Format     | Set Coupon as Posted              | ## Post to X & Telegram                                                                        |
| Coupon Status              | If                   | Check coupon active status                  | Get Rule Info              | Voucher Message Format            |                                                                                                |
| Product Status             | If                   | Check product active status                 | Get Product Info           | Product Message Format            |                                                                                                |
| Voucher Duplication Protection | If                 | Check coupon posted flag                     | New Voucher Entry          | Get Rule Info (if not posted)     |                                                                                                |
| Product Duplication Protection | If                 | Check product posted flag                    | New Product Entry          | Get Product Info (if not posted)  |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run every 1 hour (interval: hours, value: 1)  
   - Connect output to nodes: Get Latest Offer, Init Database, Fetch New Product

2. **Create Init Database node**  
   - Type: MySQL  
   - Configure credentials for MySQL connection  
   - SQL Query:  
     ```sql
     CREATE TABLE IF NOT EXISTS posted_items (
       item_id INT PRIMARY KEY,
       item_type ENUM('product', 'coupon') NOT NULL,
       item_value VARCHAR(255),
       posted BOOLEAN DEFAULT FALSE,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
     );
     ```  
   - Connect input from Schedule Trigger  
   - No outputs

3. **Create Get Latest Offer node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `{{$vars.STORE}}/rest/V1/coupons/search?searchCriteria[sortOrders][0][field]=created_at&searchCriteria[sortOrders][0][direction]=DESC&searchCriteria[pageSize]=1`  
   - Authentication: Generic HTTP Header Auth (configure credentials with Magento API token)  
   - Connect input from Schedule Trigger  
   - Connect output to New Voucher Entry

4. **Create New Voucher Entry node**  
   - Type: MySQL  
   - SQL Query:  
     ```sql
     SET @coupon_id = '{{ $json.items[0].coupon_id }}';
     SET @search_coupon = '{{ $json.items[0].code }}';

     INSERT INTO posted_items (
         item_id, 
         item_type,
         item_value,
         posted, 
         updated_at
     )
     SELECT 
         @coupon_id AS item_id,
         'coupon' AS item_type,
         @search_coupon AS item_value,
         FALSE AS posted,
         CURRENT_TIMESTAMP AS updated_at
     FROM dual
     WHERE NOT EXISTS (
         SELECT 1 
         FROM posted_items 
         WHERE item_id = @coupon_id AND item_type = 'coupon'
     );

     SELECT * 
     FROM posted_items 
     WHERE item_id = @coupon_id AND item_type = 'coupon';
     ```  
   - Connect input from Get Latest Offer  
   - Connect output to Voucher Duplication Protection

5. **Create Voucher Duplication Protection node**  
   - Type: If  
   - Condition:  
     - Check if `posted` equals 1 (number equals)  
     - Expression: `{{$json.posted}} == 1`  
   - Connect input from New Voucher Entry  
   - True branch: ends flow (no output)  
   - False branch: connects to Get Rule Info

6. **Create Get Rule Info node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://magekwik.com/rest/V1/salesRules/{{ $json.item_id }}`  
   - Authentication: Same as Get Latest Offer (Generic HTTP Header Auth)  
   - Connect input from Voucher Duplication Protection (false branch)  
   - Connect output to Coupon Status

7. **Create Coupon Status node**  
   - Type: If  
   - Condition:  
     - Check if `is_active` is true (boolean equals)  
     - Expression: `{{$json.is_active}} == true`  
   - Connect input from Get Rule Info  
   - True branch: connects to Voucher Message Format  
   - False branch: no output (skip inactive coupons)

8. **Create Voucher Message Format node**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const coupon = `🎁 New Coupon Alert! 🎁

     🎟️ <b>Coupon Code</b>: ${$('Get Latest Offer').first().json.items[0].code}
     🔄 Usage: ${$('Get Rule Info').first().json.times_used || 0}/${$('Get Rule Info').first().json.uses_per_customer || '∞'}
     ✅ Status: ${$('Get Rule Info').first().json.is_active ? 'ACTIVE ✅' : 'INACTIVE ❌'}`

     return [{ json: { coupon } }];
     ```  
   - Connect input from Coupon Status  
   - Connect outputs to Post to Telegram1 and Voucher to X (disabled by default)

9. **Create Post to Telegram1 node**  
   - Type: Telegram  
   - Credentials: Configure Telegram Bot credentials  
   - Chat ID: set to your Telegram chat ID  
   - Parameters:  
     - Text: `{{$json.coupon}}`  
     - Parse mode: HTML  
   - Connect input from Voucher Message Format  
   - Connect output to Set Coupon as Posted

10. **Create Set Coupon as Posted node**  
    - Type: MySQL  
    - SQL Query:  
      ```sql
      UPDATE posted_items set posted = 1 WHERE item_id = {{ $('Get Latest Offer').item.json.items[0].rule_id }} AND item_type = 'coupon';
      ```  
    - Connect input from Post to Telegram1 (and Voucher to X if enabled)

11. **Create Fetch New Product node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{$vars.STORE}}rest/V1/products?searchCriteria[sortOrders][0][field]=created_at&searchCriteria[sortOrders][0][direction]=DESC`  
    - Authentication: Header Auth (configure with Magento API token)  
    - Allow Unauthorized Certs: true (optional)  
    - Connect input from Schedule Trigger  
    - Connect output to New Product Entry

12. **Create New Product Entry node**  
    - Type: MySQL  
    - SQL Query:  
      ```sql
      SET @product_id = '{{ $json.items[0].id }}';
      SET @product_sku = '{{ $json.items[0].sku }}';

      INSERT INTO posted_items (
          item_id, 
          item_type,
          item_value,
          posted, 
          updated_at
      )
      SELECT 
          @product_id AS item_id,
          'product' AS item_type,
          @product_sku AS item_value,
          FALSE AS posted,
          CURRENT_TIMESTAMP AS updated_at
      FROM dual
      WHERE NOT EXISTS (
          SELECT 1 
          FROM posted_items 
          WHERE item_id = @product_id AND item_type = 'product'
      );

      SELECT * 
      FROM posted_items 
      WHERE item_id = @product_id AND item_type = 'product';
      ```  
    - Connect input from Fetch New Product  
    - Connect output to Product Duplication Protection

13. **Create Product Duplication Protection node**  
    - Type: If  
    - Condition:  
      - Check if `posted` equals 1 (number equals)  
      - Expression: `{{$json.posted}} == 1`  
    - Connect input from New Product Entry  
    - True branch: ends flow  
    - False branch: connects to Get Product Info

14. **Create Get Product Info node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://your-website.com/rest/default/V1/products/{{ $json.item_value }}`  
    - Authentication: Generic HTTP Header Auth (configure credentials)  
    - Connect input from Product Duplication Protection (false branch)  
    - Connect output to Product Status

15. **Create Product Status node**  
    - Type: If  
    - Condition:  
      - Check if `status` is true (boolean equals)  
      - Expression: `{{$json.status}} == true`  
    - Connect input from Get Product Info  
    - True branch: connects to Product Message Format  
    - False branch: ends flow

16. **Create Product Message Format node**  
    - Type: Code (JavaScript)  
    - Code:  
      ```js
      const product =  $input.first().json

      const message = `🛙 New Product Alert!\n\n*${product.name}*\nPrice: $${product.price}\n\nCheck it out now:\nhttps://magekwik.com/${product.custom_attributes[2].value}`;

      return [{
        json: {
          message,
          image: product.media_gallery_entries?.[0]?.file
            ? `https://magekwik.com/pub/media/catalog/product${product.media_gallery_entries[0].file}`
            : null,
        }
      }];
      ```  
    - Connect input from Product Status  
    - Connect outputs to Product Alert to Telegram and Product X Post

17. **Create Product Alert to Telegram node**  
    - Type: Telegram  
    - Credentials: Telegram Bot credentials  
    - Chat ID: your Telegram chat ID  
    - Operation: Send Media Group  
    - Media: image URL from expression, caption from message  
    - Connect input from Product Message Format  
    - Connect output to Set Product as Posted

18. **Create Product X Post node**  
    - Type: Twitter (X)  
    - Credentials: Twitter OAuth  
    - Text: image URL and message text from expressions  
    - Connect input from Product Message Format  
    - Connect output to Set Product as Posted  
    - Disabled by default (enable if needed)

19. **Create Set Product as Posted node**  
    - Type: MySQL  
    - SQL Query:  
      ```sql
      UPDATE posted_items set posted = 1 WHERE item_id = {{ $('Product Status').item.json.id }} AND item_type = 'product';
      ```  
    - Connect input from Product Alert to Telegram and Product X Post

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                        |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| This workflow is designed for Magento 2 (Adobe Commerce) stores to automate posting product and coupon alerts. | General project description                            |
| Duplicate prevention logic is implemented via a MySQL table recording posted items with a `posted` boolean flag.| Critical for avoiding repeated notifications          |
| Telegram bot credentials and chat IDs must be configured in Telegram nodes for posting alerts.                   | Telegram Bot API documentation                         |
| Twitter (X) posting nodes are included but one coupon posting node is disabled by default.                       | Enable and configure Twitter OAuth credentials if needed |
| Store URL and API credentials are injected via environment variables or workflow variables (`$vars.STORE`).     | Secure handling of API credentials recommended         |
| Workflow uses generic HTTP header authentication for Magento REST API calls.                                     | Magento 2 REST API docs: https://developer.adobe.com/commerce/webapi/rest/quick-reference/ |
| Includes sticky notes for visual grouping and documentation inside n8n editor.                                  | See node comments in workflow                           |