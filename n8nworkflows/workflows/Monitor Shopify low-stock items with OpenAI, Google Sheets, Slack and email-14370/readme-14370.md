Monitor Shopify low-stock items with OpenAI, Google Sheets, Slack and email

https://n8nworkflows.xyz/workflows/monitor-shopify-low-stock-items-with-openai--google-sheets--slack-and-email-14370


# Monitor Shopify low-stock items with OpenAI, Google Sheets, Slack and email

# 1. Workflow Overview

This workflow monitors Shopify inventory on a recurring schedule, identifies low-stock products, generates alert copy with OpenAI, logs or updates alert records in Google Sheets, sends Slack notifications based on urgency, and emails the team for newly detected low-stock items.

Its main use case is retail inventory monitoring for replenishment teams. It is designed to reduce manual stock checking, centralize alert tracking, and prevent duplicate alert records.

## 1.1 Scheduled Inventory Retrieval
The workflow starts every 5 hours, pulls product data from Shopify, and normalizes the fields needed downstream.

## 1.2 Low-Stock Detection and AI Message Generation
Products are processed in small batches. Each product is checked against a stock threshold, and low-stock items are sent through an OpenAI-backed LLM chain to generate a professional alert message.

## 1.3 Alert Formatting and Data Enrichment
The generated AI response is parsed into subject/body format and merged back with product metadata such as SKU, product name, variant ID, and timestamp.

## 1.4 Alert Tracking in Google Sheets
The workflow checks whether the SKU already exists in a Google Sheet. New items are appended; existing ones are updated.

## 1.5 Priority Classification and Slack Notification
Each low-stock item is classified into High, Medium, or Low priority based on exact stock quantity values, then sent to Slack.

## 1.6 Loop Continuation
After Slack notification or record update, the workflow loops back to the batch processor to continue with the next product.

## 1.7 Email Notification for New Records
When a low-stock product is newly added to the sheet, an email is sent using Gmail.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Inventory Retrieval

### Overview
This block triggers the workflow every 5 hours, retrieves all Shopify products, and reshapes the product payload into a simpler structure for stock evaluation and alerting.

### Nodes Involved
- Trigger in Schedule time
- Fetch Products
- Prepare Product data

### Node Details

#### Trigger in Schedule time
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Runs on an interval of every 5 hours.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Fetch Products**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Workflow must be active for the trigger to fire.
  - Time-based execution depends on the n8n instance clock/timezone.
- **Sub-workflow reference:** None.

#### Fetch Products
- **Type and role:** `n8n-nodes-base.shopify`; fetches Shopify product records.
- **Configuration choices:**
  - Resource: `product`
  - Operation: `getAll`
  - Authentication: access token
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Trigger in Schedule time**; output to **Prepare Product data**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Invalid or expired Shopify access token.
  - Store API permission issues.
  - Large stores may return many items; execution time can increase.
  - Variant assumptions downstream may fail if a product has no variants.
- **Sub-workflow reference:** None.

#### Prepare Product data
- **Type and role:** `n8n-nodes-base.set`; extracts and standardizes key fields from Shopify output.
- **Configuration choices:** Creates these fields:
  - `varient_id` from `{{$json.id}}`
  - `product_id` from `{{$json.variants[0].product_id}}`
  - `sku` from `{{$json.variants[0].sku}}`
  - `inventory_item_id` from `{{$json.variants[0].inventory_item_id}}`
  - `price` from `{{$json.variants[0].price}}`
  - `product_name` from `{{$json.title}}`
  - `vendor` from `{{$json.vendor}}`
  - `inventory_quantity` from `{{$json.variants[0].inventory_quantity}}`
- **Key expressions or variables used:**
  - Direct use of `variants[0]`
- **Input and output connections:** Input from **Fetch Products**; output to **Process each product**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If `variants` is empty or missing, expressions like `variants[0].sku` will fail.
  - Typo in field name `varient_id` is preserved and reused downstream.
  - Only the first variant is considered; multi-variant products are not fully covered.
- **Sub-workflow reference:** None.

---

## 2.2 Low-Stock Detection and AI Message Generation

### Overview
This block iterates through products in batches, filters to only low-stock items, and uses an LLM chain with an OpenAI chat model to generate human-readable alert text.

### Nodes Involved
- Process each product
- Check low stock
- AI text genrator
- Genrate Alert Message

### Node Details

#### Process each product
- **Type and role:** `n8n-nodes-base.splitInBatches`; controls iteration and throttles processing.
- **Configuration choices:** Batch size is `5`.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from **Prepare Product data**
  - Loop return inputs from **Send slack alert** and **Update existing alert in sheet**
  - Second output goes to **Check low stock**
- **Version-specific requirements:** Type version `3`.
- **Edge cases / failures:**
  - Loop behavior depends on proper return connection.
  - Because multiple branches return to this node, execution order should be verified carefully during maintenance.
- **Sub-workflow reference:** None.

#### Check low stock
- **Type and role:** `n8n-nodes-base.if`; filters products whose inventory is low.
- **Configuration choices:** Passes only items where `inventory_quantity <= 10`.
- **Key expressions or variables used:**
  - `={{ $json.inventory_quantity }}`
- **Input and output connections:** Input from **Process each product**; true output goes to **Genrate Alert Message**. No false-path node is connected.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Strict numeric validation means null/non-numeric values may fail or not match as expected.
  - Products above 10 units are silently skipped.
- **Sub-workflow reference:** None.

#### AI text genrator
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model provider node.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Used as the model input for the downstream LLM chain
- **Key expressions or variables used:** None directly.
- **Input and output connections:** AI model connection to **Genrate Alert Message**.
- **Version-specific requirements:** Type version `1.2`; requires LangChain-compatible n8n nodes.
- **Edge cases / failures:**
  - OpenAI credential errors
  - Model access restrictions
  - API latency or rate limits
- **Sub-workflow reference:** None.

#### Genrate Alert Message
- **Type and role:** `@n8n/n8n-nodes-langchain.chainLlm`; prompts the LLM to write an inventory alert.
- **Configuration choices:**
  - Prompt type: defined directly in node
  - System-style message: inventory management assistant producing concise purchasing alerts
  - User prompt includes:
    - product name
    - SKU
    - current stock
    - reorder threshold
    - price
    - vendor
  - Requests a professional alert suitable for Email or Slack
- **Key expressions or variables used:**
  - `{{$json.product_name || "Unknown"}}`
  - `{{$json.sku || "N/A"}}`
  - `{{$json.inventory_quantity}}`
  - `{{$json.reorder_level || 10}}`
  - `{{$json.price}}`
  - `{{$json.vendor || "Unknown"}}`
- **Input and output connections:** Main input from **Check low stock**; AI model input from **AI text genrator**; output to **Formate Alert Message**.
- **Version-specific requirements:** Type version `1.7`.
- **Edge cases / failures:**
  - If model node is disconnected or unavailable, chain execution fails.
  - Output format is not guaranteed unless prompt strongly enforces it.
  - Misspelled node name has no functional impact but can confuse maintainers.
- **Sub-workflow reference:** None.

---

## 2.3 Alert Formatting and Data Enrichment

### Overview
This block parses the LLM text into structured fields and restores product metadata so downstream Google Sheets, Slack, and email nodes can use a consistent payload.

### Nodes Involved
- Formate Alert Message

### Node Details

#### Formate Alert Message
- **Type and role:** `n8n-nodes-base.code`; converts AI text into structured JSON.
- **Configuration choices:**
  - Reads all items from **Check low stock**
  - For each incoming AI result:
    - extracts `text`
    - checks whether first line starts with `Subject:`
    - sets fallback subject `Low Stock Alert`
    - creates `body`
    - merges in Shopify fields from the corresponding low-stock item
    - adds `time_stemp` as current ISO timestamp
- **Key expressions or variables used:**
  - `$items('Check low stock')`
  - `item.json.text`
  - `new Date().toISOString()`
  - maps `variant_id` from `shopifyItem.varient_id`
- **Input and output connections:** Input from **Genrate Alert Message**; outputs to both **Check existing records** and **Set stock priority**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Index-based merging assumes AI outputs remain aligned with IF node items.
  - If AI output does not include `text`, body becomes empty.
  - Typo `time_stemp` is preserved downstream.
  - If LLM returns unexpected formatting, subject/body parsing may be imperfect.
- **Sub-workflow reference:** None.

---

## 2.4 Alert Tracking in Google Sheets

### Overview
This block checks whether a SKU already exists in the tracking sheet and either appends a new alert row or updates an existing one.

### Nodes Involved
- Check existing records
- New or existing records??
- Add product alert to sheet
- Update existing alert in sheet
- Notify team

### Node Details

#### Check existing records
- **Type and role:** `n8n-nodes-base.googleSheets`; searches for existing alert rows by SKU.
- **Configuration choices:**
  - Uses Google Sheets document `New Order Track`
  - Sheet/tab: `order_restock`
  - Filter: `sku` equals `$('Formate Alert Message').item.json.sku`
- **Key expressions or variables used:**
  - `={{ $('Formate Alert Message').item.json.sku }}`
- **Input and output connections:** Input from **Formate Alert Message**; output to **New or existing records??**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Google OAuth authentication issues
  - Sheet not found / schema changed
  - SKU column naming mismatch
  - If multiple rows match same SKU, downstream update logic may become ambiguous
- **Sub-workflow reference:** None.

#### New or existing records??
- **Type and role:** `n8n-nodes-base.if`; decides whether the SKU is new.
- **Configuration choices:** Compares:
  - `($items('Check existing records').length === 0).toString()`
  - against `=`
- **Key expressions or variables used:**
  - `={{ ($items('Check existing records').length === 0).toString() }}`
- **Input and output connections:** Input from **Check existing records**; first output to **Add product alert to sheet**, second output to **Update existing alert in sheet**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - This condition appears misconfigured. Comparing the stringified boolean to `"="` will not correctly test true/false.
  - There is also a trailing newline in the expression value.
  - In practice, this node may not classify records correctly without correction.
- **Sub-workflow reference:** None.

#### Add product alert to sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; appends a new low-stock alert row.
- **Configuration choices:**
  - Operation: `append`
  - Writes:
    - `sku`
    - `message` from `subject`
    - `varient_id` from `variant_id`
    - `product_name`
    - `inventory_quntity`
  - Does not populate `time_stemp` even though the schema includes it
- **Key expressions or variables used:**
  - `={{ $json.sku }}`
  - `={{ $json.subject }}`
  - `={{ $json.variant_id }}`
  - `={{ $json.product_name }}`
  - `={{ $json.inventory_quantity }}`
- **Input and output connections:** Input from **New or existing records??**; output to **Notify team**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Column names contain typos: `varient_id`, `inventory_quntity`.
  - If sheet schema differs from node schema, append can fail or map incorrectly.
  - Omits timestamp even though downstream reporting may expect it.
- **Sub-workflow reference:** None.

#### Update existing alert in sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; updates an existing row by row number.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Writes:
    - `sku`
    - `message`
    - `row_number`
    - `time_stemp`
    - `varient_id`
    - `product_name`
    - `inventory_quntity`
- **Key expressions or variables used:**
  - `={{ $json.row_number }}`
  - `={{ $json.message }}`
  - `={{ $json.time_stemp }}`
- **Input and output connections:** Input from **New or existing records??**; output back to **Process each product**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Upstream payload from **Formate Alert Message** does not include `message`, `row_number`, `varient_id`, or `inventory_quntity` in the same shape expected here.
  - Unless **Check existing records** output is merged properly, this node likely lacks the row number needed for update.
  - Risk of failed updates or blank field overwrites.
- **Sub-workflow reference:** None.

#### Notify team
- **Type and role:** `n8n-nodes-base.gmail`; sends an email notification for newly logged alerts.
- **Configuration choices:**
  - To: `{{email_here}}`
  - Subject from formatted AI message subject
  - Body from formatted AI message body
  - Attribution disabled
- **Key expressions or variables used:**
  - `={{ $('Formate Alert Message').item.json.body }}`
  - `={{ $('Formate Alert Message').item.json.subject }}`
- **Input and output connections:** Input from **Add product alert to sheet**; no downstream connection.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases / failures:**
  - Placeholder `{{email_here}}` must be replaced with a real address or expression.
  - Gmail OAuth scope and sending permissions required.
  - Because the node uses cross-node item lookup rather than current item fields, item alignment should be verified.
- **Sub-workflow reference:** None.

---

## 2.5 Priority Classification and Slack Notification

### Overview
This block assigns human-readable urgency messages based on exact stock values and posts a Slack alert.

### Nodes Involved
- Set stock priority
- High Priority Alert
- Medium Priority Alert
- Low Priority Alert
- Send slack alert

### Node Details

#### Set stock priority
- **Type and role:** `n8n-nodes-base.switch`; routes low-stock items into priority bands.
- **Configuration choices:**
  - Route 1 if `inventory_quantity == 2`
  - Route 2 if `inventory_quantity == 6`
  - Route 3 if `inventory_quantity == 10`
- **Key expressions or variables used:**
  - `={{ $json.inventory_quantity }}`
- **Input and output connections:** Input from **Formate Alert Message**; outputs to **High Priority Alert**, **Medium Priority Alert**, and **Low Priority Alert**.
- **Version-specific requirements:** Type version `3.3`.
- **Edge cases / failures:**
  - Only exact values 2, 6, and 10 are handled.
  - Inventory values 0, 1, 3, 4, 5, 7, 8, 9 also count as low stock but will not go anywhere.
  - Consider replacing with range-based logic for real-world use.
- **Sub-workflow reference:** None.

#### High Priority Alert
- **Type and role:** `n8n-nodes-base.set`; assigns high-priority wording.
- **Configuration choices:**
  - `priority`: “HIGH PRIORITY: Stock is critically low...”
  - carries forward `product_name`, `sku`, `inventory_quantity`
- **Key expressions or variables used:** direct field copies from current JSON.
- **Input and output connections:** Input from **Set stock priority**; output to **Send slack alert**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - `inventory_quantity` is stored as string here, unlike other branches.
- **Sub-workflow reference:** None.

#### Medium Priority Alert
- **Type and role:** `n8n-nodes-base.set`; assigns medium-priority wording.
- **Configuration choices:**
  - field name is configured as `=priority` instead of `priority`
  - carries `product_name`, `sku`, `inventory_quantity`
- **Key expressions or variables used:** direct field copies.
- **Input and output connections:** Input from **Set stock priority**; output to **Send slack alert**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - The field name typo `=priority` likely means Slack expression `{{$json.priority}}` will be empty for this branch.
- **Sub-workflow reference:** None.

#### Low Priority Alert
- **Type and role:** `n8n-nodes-base.set`; assigns low-priority wording.
- **Configuration choices:**
  - field name is configured as `=priority` instead of `priority`
  - carries `product_name`, `sku`, `inventory_quantity`
- **Key expressions or variables used:** direct field copies.
- **Input and output connections:** Input from **Set stock priority**; output to **Send slack alert**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Same field name typo as medium branch; Slack priority text may be blank.
- **Sub-workflow reference:** None.

#### Send slack alert
- **Type and role:** `n8n-nodes-base.slack`; posts alert details to a Slack channel.
- **Configuration choices:**
  - Sends to channel `n8n`
  - Message body includes:
    - priority
    - product name
    - SKU
    - inventory quantity
  - Workflow link disabled
- **Key expressions or variables used:**
  - `{{ $json.priority }}`
  - `{{ $json.product_name }}`
  - `{{ $json.sku }}`
  - `{{ $json.inventory_quantity }}`
- **Input and output connections:** Input from all three priority set nodes; output back to **Process each product**.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - Slack auth or channel permission errors
  - For medium/low branches, `priority` may be empty due to upstream field naming typo
  - Some low-stock items never reach Slack because switch handles only exact quantities
- **Sub-workflow reference:** None.

---

## 2.6 Documentation / Annotation Nodes

### Overview
These nodes are visual annotations only. They do not affect workflow execution but document setup, business purpose, and logical grouping.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; overall workflow description and setup guidance.
- **Configuration choices:** Large note explaining the process and setup requirements.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None; non-executable.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and role:** sticky note for product fetch/preparation area.
- **Configuration choices:** Describes Shopify fetch and data prep stage.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and role:** sticky note for low-stock AI processing area.
- **Configuration choices:** Describes batch processing, low-stock filtering, AI text generation, formatting.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and role:** sticky note for priority/slack area.
- **Configuration choices:** Describes urgency assignment and Slack alerting.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and role:** sticky note for Google Sheets tracking area.
- **Configuration choices:** Describes create/update behavior in Sheets.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger in Schedule time | Schedule Trigger | Starts workflow every 5 hours |  | Fetch Products | ## Fetch Products & Prepare data Every 5 hours, the system automatically checks your Shopify store. It collects product details like stock levels and prepares the data for further actions such as alerts and reporting. |
| Fetch Products | Shopify | Retrieves all Shopify products | Trigger in Schedule time | Prepare Product data | ## Fetch Products & Prepare data Every 5 hours, the system automatically checks your Shopify store. It collects product details like stock levels and prepares the data for further actions such as alerts and reporting. |
| Prepare Product data | Set | Extracts relevant fields from Shopify product payload | Fetch Products | Process each product | ## Fetch Products & Prepare data Every 5 hours, the system automatically checks your Shopify store. It collects product details like stock levels and prepares the data for further actions such as alerts and reporting. |
| Process each product | Split In Batches | Iterates through products in batches of 5 | Prepare Product data; Send slack alert; Update existing alert in sheet | Check low stock | ## Low-Stock Alert Processing The system goes through each product one by one and checks whether the stock is running low. For low-stock items, it uses AI to create a clear and professional alert message, then formats the message with all important product details so it’s ready to be sent by email or Slack. |
| Check low stock | If | Filters products with inventory quantity 10 or below | Process each product | Genrate Alert Message | ## Low-Stock Alert Processing The system goes through each product one by one and checks whether the stock is running low. For low-stock items, it uses AI to create a clear and professional alert message, then formats the message with all important product details so it’s ready to be sent by email or Slack. |
| AI text genrator | OpenAI Chat Model (LangChain) | Supplies LLM model to the chain node |  | Genrate Alert Message (AI model connection) | ## Low-Stock Alert Processing The system goes through each product one by one and checks whether the stock is running low. For low-stock items, it uses AI to create a clear and professional alert message, then formats the message with all important product details so it’s ready to be sent by email or Slack. |
| Genrate Alert Message | LLM Chain | Generates low-stock alert copy | Check low stock; AI text genrator | Formate Alert Message | ## Low-Stock Alert Processing The system goes through each product one by one and checks whether the stock is running low. For low-stock items, it uses AI to create a clear and professional alert message, then formats the message with all important product details so it’s ready to be sent by email or Slack. |
| Formate Alert Message | Code | Parses AI text into subject/body and enriches item data | Genrate Alert Message | Check existing records; Set stock priority | ## Low-Stock Alert Processing The system goes through each product one by one and checks whether the stock is running low. For low-stock items, it uses AI to create a clear and professional alert message, then formats the message with all important product details so it’s ready to be sent by email or Slack. |
| Check existing records | Google Sheets | Looks up existing sheet rows by SKU | Formate Alert Message | New or existing records?? | ## Alert Tracking & Record Management The system checks your Google Sheet to see if a low-stock alert for the product already exists. If it’s a new product, it adds a fresh entry to the sheet; if it’s already there, it updates the existing record with the latest stock and alert details so everything stays up to date. |
| New or existing records?? | If | Determines whether to append or update sheet row | Check existing records | Add product alert to sheet; Update existing alert in sheet | ## Alert Tracking & Record Management The system checks your Google Sheet to see if a low-stock alert for the product already exists. If it’s a new product, it adds a fresh entry to the sheet; if it’s already there, it updates the existing record with the latest stock and alert details so everything stays up to date. |
| Add product alert to sheet | Google Sheets | Appends new alert row | New or existing records?? | Notify team | ## Alert Tracking & Record Management The system checks your Google Sheet to see if a low-stock alert for the product already exists. If it’s a new product, it adds a fresh entry to the sheet; if it’s already there, it updates the existing record with the latest stock and alert details so everything stays up to date. |
| Update existing alert in sheet | Google Sheets | Updates existing alert row | New or existing records?? | Process each product | ## Alert Tracking & Record Management The system checks your Google Sheet to see if a low-stock alert for the product already exists. If it’s a new product, it adds a fresh entry to the sheet; if it’s already there, it updates the existing record with the latest stock and alert details so everything stays up to date. |
| Notify team | Gmail | Emails the team about newly added alert records | Add product alert to sheet |  | ## Alert Tracking & Record Management The system checks your Google Sheet to see if a low-stock alert for the product already exists. If it’s a new product, it adds a fresh entry to the sheet; if it’s already there, it updates the existing record with the latest stock and alert details so everything stays up to date. |
| Set stock priority | Switch | Routes items into priority levels based on exact stock values | Formate Alert Message | High Priority Alert; Medium Priority Alert; Low Priority Alert | ## Priority-Based Alerts & Team Notification Based on how low the stock is, the system assigns a priority level (high, medium, or low) and prepares the right message for each case. It then sends a clear alert to your Slack channel so the team knows exactly which products need urgent attention and which can be restocked later. |
| High Priority Alert | Set | Creates high-priority Slack payload | Set stock priority | Send slack alert | ## Priority-Based Alerts & Team Notification Based on how low the stock is, the system assigns a priority level (high, medium, or low) and prepares the right message for each case. It then sends a clear alert to your Slack channel so the team knows exactly which products need urgent attention and which can be restocked later. |
| Medium Priority Alert | Set | Creates medium-priority Slack payload | Set stock priority | Send slack alert | ## Priority-Based Alerts & Team Notification Based on how low the stock is, the system assigns a priority level (high, medium, or low) and prepares the right message for each case. It then sends a clear alert to your Slack channel so the team knows exactly which products need urgent attention and which can be restocked later. |
| Low Priority Alert | Set | Creates low-priority Slack payload | Set stock priority | Send slack alert | ## Priority-Based Alerts & Team Notification Based on how low the stock is, the system assigns a priority level (high, medium, or low) and prepares the right message for each case. It then sends a clear alert to your Slack channel so the team knows exactly which products need urgent attention and which can be restocked later. |
| Send slack alert | Slack | Sends priority-based team notification | High Priority Alert; Medium Priority Alert; Low Priority Alert | Process each product | ## Priority-Based Alerts & Team Notification Based on how low the stock is, the system assigns a priority level (high, medium, or low) and prepares the right message for each case. It then sends a clear alert to your Slack channel so the team knows exactly which products need urgent attention and which can be restocked later. |
| Sticky Note | Sticky Note | Documents workflow purpose and setup |  |  | ## How it works This workflow automatically monitors your Shopify inventory and alerts your team when products are running low. Every 5 hours, it checks all products in your Shopify store and reviews their stock levels. Products with low inventory (10 units or less) are identified and processed one by one to avoid overload. For each low-stock product, the system creates a clear alert message using AI. The message includes product details such as name, SKU, and current stock, and is formatted for email and Slack. To avoid duplicate records, the workflow checks a Google Sheet to see if the product has already been logged. If it’s a new product, a new row is added; if it already exists, the row is updated with the latest stock and alert details. Finally, products are categorized by priority (High, Medium, or Low) based on stock level, and a notification is sent to Slack so the team knows what action to take. ## Setup steps **1.** Connect your **Shopify account** to fetch product and inventory data. **2.** Connect **OpenAI** to generate alert messages. **3.** Connect **Gmail** (optional) and **Slack** for notifications. **4.** Connect a **Google Sheet** to store and track low-stock alerts. **5.** Adjust stock thresholds or schedule timing if needed. **6.** Turn on the workflow and let it run automatically. |
| Sticky Note1 | Sticky Note | Documents Shopify fetch and prep stage |  |  |  |
| Sticky Note2 | Sticky Note | Documents low-stock AI processing stage |  |  |  |
| Sticky Note3 | Sticky Note | Documents priority/slack stage |  |  |  |
| Sticky Note4 | Sticky Note | Documents Google Sheets tracking stage |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a complete rebuild sequence in n8n.

## 1. Create the schedule trigger
1. Add a **Schedule Trigger** node.
2. Name it **Trigger in Schedule time**.
3. Set the rule to run every **5 hours**.
4. This will be the workflow entry point.

## 2. Add Shopify product retrieval
5. Add a **Shopify** node and name it **Fetch Products**.
6. Connect **Trigger in Schedule time → Fetch Products**.
7. Configure:
   - **Resource:** Product
   - **Operation:** Get All
   - **Authentication:** Access Token
8. Create or select Shopify credentials:
   - Shopify store URL
   - Admin/API access token with product read permissions

## 3. Normalize the Shopify payload
9. Add a **Set** node named **Prepare Product data**.
10. Connect **Fetch Products → Prepare Product data**.
11. Add these fields:
   - `varient_id` → `{{$json.id}}`
   - `product_id` → `{{$json.variants[0].product_id}}`
   - `sku` → `{{$json.variants[0].sku}}`
   - `inventory_item_id` → `{{$json.variants[0].inventory_item_id}}`
   - `price` → `{{$json.variants[0].price}}`
   - `product_name` → `{{$json.title}}`
   - `vendor` → `{{$json.vendor}}`
   - `inventory_quantity` → `{{$json.variants[0].inventory_quantity}}`
12. Keep in mind this structure assumes the first variant exists.

## 4. Process products in batches
13. Add a **Split In Batches** node named **Process each product**.
14. Connect **Prepare Product data → Process each product**.
15. Set **Batch Size = 5**.

## 5. Filter low-stock items
16. Add an **If** node named **Check low stock**.
17. Connect the **loop output** of **Process each product** to **Check low stock**.
18. Configure condition:
   - `inventory_quantity`
   - Number
   - Less than or equal to
   - `10`

## 6. Add the OpenAI chat model
19. Add an **OpenAI Chat Model** node from the LangChain nodes.
20. Name it **AI text genrator**.
21. Select model **gpt-4.1-mini**.
22. Create/select OpenAI credentials with API key access.

## 7. Add the LLM chain
23. Add an **LLM Chain** node and name it **Genrate Alert Message**.
24. Connect **Check low stock (true) → Genrate Alert Message**.
25. Connect **AI text genrator** to the LLM chain through the **AI language model** connector.
26. Set prompt mode to define the prompt directly.
27. Add a system/instruction message similar to:
   - “You are an inventory management assistant. Create a concise alert message for the purchasing team.”
28. Set the main prompt text to include:
   - Product name
   - SKU
   - Current stock
   - Reorder threshold
   - Price
   - Vendor
   - Request for clear professional output for email or Slack
29. Example expression pattern:
   - `Product Name: {{$json.product_name || "Unknown"}}`
   - `SKU: {{$json.sku || "N/A"}}`
   - `Current Stock: {{$json.inventory_quantity}}`
   - `Reorder Threshold: {{$json.reorder_level || 10}}`
   - `Price: {{$json.price}}`
   - `Vendor: {{$json.vendor || "Unknown"}}`

## 8. Parse the AI response
30. Add a **Code** node named **Formate Alert Message**.
31. Connect **Genrate Alert Message → Formate Alert Message**.
32. Paste this logic in equivalent form:
   - Read AI output text from each item
   - Split by line
   - If first line starts with `Subject:`, store it as subject
   - Use remainder as body
   - Default subject to `Low Stock Alert`
   - Attach `sku`, `inventory_quantity`, `product_name`, `variant_id`, `time_stemp`
33. The code should also reference the low-stock items from **Check low stock** so product context is preserved.

## 9. Add Google Sheets record lookup
34. Add a **Google Sheets** node named **Check existing records**.
35. Connect **Formate Alert Message → Check existing records**.
36. Configure:
   - Choose your spreadsheet
   - Choose your sheet/tab, e.g. `order_restock`
   - Add filter: column `sku` equals the formatted SKU
37. Use Google Sheets OAuth2 credentials with spreadsheet access.

## 10. Add new/existing record decision
38. Add an **If** node named **New or existing records??**.
39. Connect **Check existing records → New or existing records??**.
40. Configure it to test whether the lookup returned zero rows.
41. Recommended correct logic:
   - Expression: `{{ $items('Check existing records').length === 0 }}`
   - Compare as boolean to `true`
42. Do not replicate the original comparison to `"="`; that appears broken.

## 11. Append new records to Google Sheets
43. Add a **Google Sheets** node named **Add product alert to sheet**.
44. Connect the **true/new** output of **New or existing records??** to this node.
45. Set:
   - Operation: **Append**
   - Spreadsheet: same file as above
   - Sheet: same tab
46. Map columns:
   - `sku` ← `{{$json.sku}}`
   - `message` ← `{{$json.subject}}`
   - `varient_id` ← `{{$json.variant_id}}`
   - `product_name` ← `{{$json.product_name}}`
   - `inventory_quntity` ← `{{$json.inventory_quantity}}`
   - optionally also add `time_stemp` ← `{{$json.time_stemp}}`
47. Ensure your sheet actually contains those exact column names if you keep the original spellings.

## 12. Update existing records
48. Add another **Google Sheets** node named **Update existing alert in sheet**.
49. Connect the **false/existing** output of **New or existing records??** to this node.
50. Set:
   - Operation: **Update**
   - Match on `row_number`
51. To make this work correctly, ensure the lookup node returns `row_number` and that you merge that value into the update payload.
52. Map:
   - `row_number`
   - `sku`
   - `message`
   - `time_stemp`
   - `varient_id`
   - `product_name`
   - `inventory_quntity`
53. Important: the original workflow is missing a proper merge step between lookup output and formatted alert output. Add a **Merge** node if needed so row metadata and formatted message data are available together.

## 13. Add email notification for new records
54. Add a **Gmail** node named **Notify team**.
55. Connect **Add product alert to sheet → Notify team**.
56. Configure Gmail OAuth2 credentials.
57. Set:
   - **To:** a real email address or expression
   - **Subject:** `{{$('Formate Alert Message').item.json.subject}}`
   - **Message:** `{{$('Formate Alert Message').item.json.body}}`
   - Disable attribution if desired
58. Replace the placeholder `{{email_here}}` with an actual recipient.

## 14. Add priority routing
59. Add a **Switch** node named **Set stock priority**.
60. Connect **Formate Alert Message → Set stock priority**.
61. Add 3 rules. The original workflow uses exact matches:
   - Route 1: `inventory_quantity == 2`
   - Route 2: `inventory_quantity == 6`
   - Route 3: `inventory_quantity == 10`
62. Recommended improvement: use ranges instead of exact values.

## 15. Add priority payload nodes
63. Add a **Set** node named **High Priority Alert**.
64. Connect output 1 of **Set stock priority** to it.
65. Add fields:
   - `priority` = “HIGH PRIORITY: Stock is critically low. Please restock this item immediately to avoid stockout.”
   - `product_name` = `{{$json.product_name}}`
   - `sku` = `{{$json.sku}}`
   - `inventory_quantity` = `{{$json.inventory_quantity}}`

66. Add a **Set** node named **Medium Priority Alert**.
67. Connect output 2 of **Set stock priority** to it.
68. Add fields:
   - `priority` = “MEDIUM PRIORITY: Stock is running low. Please plan restocking soon.”
   - `product_name`
   - `sku`
   - `inventory_quantity`

69. Add a **Set** node named **Low Priority Alert**.
70. Connect output 3 of **Set stock priority** to it.
71. Add fields:
   - `priority` = “LOW PRIORITY: Stock has reached reorder level. Restock when convenient”
   - `product_name`
   - `sku`
   - `inventory_quantity`

72. Important: use field name `priority`, not `=priority`.

## 16. Add Slack notification
73. Add a **Slack** node named **Send slack alert**.
74. Connect all three priority nodes into **Send slack alert**.
75. Configure Slack credentials.
76. Choose **channel** as destination and select your target channel.
77. Set the message text similar to:
   - `Order Restock Notification:`
   - `Priority: {{$json.priority}}`
   - `Product Name: {{$json.product_name}}`
   - `SKU: {{$json.sku}}`
   - `Inventory_Quantity: {{$json.inventory_quantity}}`
78. Optionally disable link-to-workflow inclusion.

## 17. Close the batch loop
79. Connect **Send slack alert → Process each product** to continue the next batch item.
80. Connect **Update existing alert in sheet → Process each product** as well, matching the original design.

## 18. Add optional annotation notes
81. Add sticky notes to document:
   - Overall workflow purpose and setup
   - Shopify fetch/prep area
   - AI processing area
   - Google Sheets tracking area
   - Slack priority area

## 19. Final credential checklist
82. Configure these credentials before activation:
   - **Shopify Access Token**
   - **OpenAI API**
   - **Google Sheets OAuth2**
   - **Slack API**
   - **Gmail OAuth2**
83. Confirm scopes/permissions:
   - Shopify product read access
   - OpenAI model access
   - Google Sheets read/write
   - Slack post message permission
   - Gmail send email permission

## 20. Recommended corrections before production use
84. Fix `New or existing records??` logic to correctly detect zero results.
85. Add a merge step for update-path row metadata if `row_number` is needed.
86. Replace exact priority checks with ranges.
87. Handle products with no variants safely.
88. Decide whether to keep or rename typo fields:
   - `varient_id`
   - `inventory_quntity`
   - `time_stemp`
   If renaming, update all dependent expressions and sheet columns.
89. Replace placeholder recipient email.
90. Test with sample products at stock 2, 6, 10, and other low values like 1, 5, 9.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Overall purpose: automatically monitor Shopify inventory, generate AI-written low-stock alerts, store/update records in Google Sheets, and notify the team by Slack and email. | Workflow-level |
| Schedule behavior: runs every 5 hours. | Trigger configuration |
| Setup guidance embedded in workflow notes mentions Shopify, OpenAI, Gmail, Slack, and Google Sheets connections. | Internal workflow annotation |
| Gmail is described as optional in the note, but the current workflow includes a Gmail node on the new-record branch. | Design note |
| Several labels and field names contain typos (`Genrate`, `Formate`, `varient_id`, `inventory_quntity`, `time_stemp`). They are functional only if used consistently. | Maintenance note |
| The current switch logic does not cover all low-stock values; only exact quantities 2, 6, and 10 trigger Slack alerts. | Important implementation note |
| The current new/existing record IF condition appears malformed and should be corrected before relying on append/update behavior. | Important implementation note |

If you want, I can also provide:
1. a **corrected production-safe version** of this workflow logic, or  
2. a **node-by-node validation checklist** for testing it in n8n.