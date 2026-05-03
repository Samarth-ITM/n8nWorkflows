Monitor Shopify Inventory & Send Low Stock Alerts to Slack

https://n8nworkflows.xyz/workflows/monitor-shopify-inventory---send-low-stock-alerts-to-slack-6110


# Monitor Shopify Inventory & Send Low Stock Alerts to Slack

### 1. Workflow Overview

This workflow, titled **"Monitor Shopify Inventory & Send Low Stock Alerts to Slack"**, is designed to automate daily inventory monitoring for a Shopify store and notify a Slack channel when product variants fall below a specified stock threshold. It targets e-commerce managers or store operators who want to proactively manage inventory to prevent stockouts and maintain sales.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Inventory Retrieval:** Automatically triggers daily to fetch all products and their inventory details from Shopify.
- **1.2 Low Stock Filtering:** Processes retrieved products to identify variants with inventory below a configurable threshold.
- **1.3 Alert Decision Making:** Determines if any low stock items exist to decide whether to proceed with alerting.
- **1.4 Alert Message Formatting:** Constructs a user-friendly message listing low stock items for Slack.
- **1.5 Alert Dispatch:** Sends the alert message to a specified Slack channel using OAuth2 authentication.

Supporting this core logic are sticky notes providing metadata, feature descriptions, configuration guidance, and business value context.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Inventory Retrieval

**Overview:**  
This block initiates the workflow once per day at 9 AM, triggering a Shopify API call to retrieve all products and their variants.

**Nodes Involved:**  
- Daily Inventory Check  
- Get All Products  

**Node Details:**

- **Daily Inventory Check**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution daily at a fixed time.  
  - Configuration: Cron expression set to "0 9 * * *" (9 AM every day).  
  - Inputs: None (trigger node).  
  - Outputs: Connected to "Get All Products".  
  - Potential Failures: Scheduler misconfiguration, n8n downtime preventing trigger.  
  - Version: 1.2.

- **Get All Products**  
  - Type: Shopify API node  
  - Role: Retrieves complete product list with variants from Shopify store.  
  - Configuration: Resource = "product", Operation = "getAll", Return all results.  
  - Inputs: Triggered by "Daily Inventory Check".  
  - Outputs: Passes all product data to "Filter Low Stock Items".  
  - Potential Failures: Shopify API authentication errors, rate limits, network issues.  
  - Version: 1.  
  - Credentials: Requires valid Shopify API credentials with read access to products.

---

#### 2.2 Low Stock Filtering

**Overview:**  
Processes the full product list to filter variants whose inventory quantity is below or equal to a defined threshold but greater than zero.

**Nodes Involved:**  
- Filter Low Stock Items  

**Node Details:**

- **Filter Low Stock Items**  
  - Type: Code (JavaScript) node  
  - Role: Custom logic to iterate over products and variants, selecting those with low stock.  
  - Configuration:  
    - Threshold variable `lowStockThreshold` set to 10 (modifiable).  
    - Logic checks each variant’s `inventory_quantity`.  
    - Collects product title, variant title, SKU, current stock, product and variant IDs.  
    - Returns array of low stock items as individual JSON objects.  
  - Inputs: Receives full product data array from "Get All Products".  
  - Outputs: Low stock variants array sent to "Check if Alerts Needed".  
  - Potential Failures: Code execution errors if product data format changes; empty input arrays; undefined variant fields.  
  - Version: 2.

---

#### 2.3 Alert Decision Making

**Overview:**  
Evaluates whether any low stock items were found to determine if an alert should be sent.

**Nodes Involved:**  
- Check if Alerts Needed  

**Node Details:**

- **Check if Alerts Needed**  
  - Type: If node  
  - Role: Conditional branching based on the length of low stock items array.  
  - Configuration:  
    - Condition checks if count of items from "Filter Low Stock Items" is greater than 0.  
  - Inputs: Low stock items from "Filter Low Stock Items".  
  - Outputs:  
    - True branch: proceeds to "Format Alert Message" to create notification.  
    - False branch: workflow ends (no alert needed).  
  - Potential Failures: Expression evaluation errors if input data missing; logical errors if condition misconfigured.  
  - Version: 2.

---

#### 2.4 Alert Message Formatting

**Overview:**  
Constructs a formatted alert message listing all low stock items with details, preparing the payload for Slack.

**Nodes Involved:**  
- Format Alert Message  

**Node Details:**

- **Format Alert Message**  
  - Type: Set node  
  - Role: Uses an expression to generate a markdown-formatted alert message summarizing low stock variants.  
  - Configuration:  
    - JSON output contains two fields: `alert_message` and `products_count`.  
    - Message uses Slack markdown with bullet points including product and variant titles and current stock counts.  
  - Inputs: True branch of "Check if Alerts Needed".  
  - Outputs: Passes formatted message JSON to "Send Inventory Alert".  
  - Potential Failures: Expression syntax errors; empty arrays leading to empty messages (though blocked by prior condition).  
  - Version: 3.4.

---

#### 2.5 Alert Dispatch

**Overview:**  
Sends the formatted low stock alert message to a designated Slack channel using OAuth2 authentication.

**Nodes Involved:**  
- Send Inventory Alert  

**Node Details:**

- **Send Inventory Alert**  
  - Type: Slack node  
  - Role: Posts message to Slack channel specified by channel ID.  
  - Configuration:  
    - Text content dynamically set from `alert_message` field of previous node.  
    - Channel ID: "C1234567890" (placeholder; must be replaced with actual channel ID).  
    - Authentication: OAuth2 (Slack App with chat:write permission).  
  - Inputs: Receives alert message from "Format Alert Message".  
  - Outputs: Workflow ends after dispatch.  
  - Potential Failures: Slack API auth failures, invalid channel ID, message formatting errors, network issues.  
  - Version: 2.3.

---

#### 2.6 Documentation and Configuration Notes

**Overview:**  
Sticky notes providing workflow metadata, feature summary, threshold customization advice, and business value.

**Nodes Involved:**  
- Workflow Info (sticky note)  
- Features & Config (sticky note)  
- Business Value (sticky note)  
- Threshold Guide (sticky note)  

**Node Details:**

- These nodes do not affect workflow logic but provide essential documentation for users.

---

### 3. Summary Table

| Node Name              | Node Type                  | Functional Role                         | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                                      |
|------------------------|----------------------------|---------------------------------------|----------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Daily Inventory Check   | Schedule Trigger           | Triggers workflow daily at 9 AM       | -                          | Get All Products          | Part of main daily trigger flow.                                                                                                |
| Get All Products        | Shopify                    | Retrieves all products and variants   | Daily Inventory Check       | Filter Low Stock Items    | Requires Shopify API credentials.                                                                                               |
| Filter Low Stock Items  | Code                       | Filters variants below stock threshold| Get All Products            | Check if Alerts Needed    | Threshold customizable in code (`lowStockThreshold`).                                                                          |
| Check if Alerts Needed  | If                         | Checks if any low stock items exist   | Filter Low Stock Items      | Format Alert Message      | Proceeds only if low stock variants found.                                                                                      |
| Format Alert Message    | Set                        | Formats alert message for Slack       | Check if Alerts Needed      | Send Inventory Alert      | Uses Slack markdown formatting.                                                                                                |
| Send Inventory Alert    | Slack                      | Sends alert message to Slack channel  | Format Alert Message        | -                        | Requires Slack OAuth2 credentials; channel ID must be configured.                                                               |
| Workflow Info           | Sticky Note                | Documentation of workflow purpose     | -                          | -                        | "Automatically monitor Shopify inventory levels and alert when products are running low..."                                    |
| Features & Config       | Sticky Note                | Lists key features and configuration  | -                          | -                        | Highlights customization options and features.                                                                                 |
| Business Value          | Sticky Note                | Explains business benefits            | -                          | -                        | Emphasizes preventing stockouts and maintaining revenue.                                                                       |
| Threshold Guide         | Sticky Note                | Guides on configuring low stock level | -                          | -                        | Explains how to adjust `lowStockThreshold` variable.                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Inventory Check" node**  
   - Type: Schedule Trigger  
   - Set Cron expression to `0 9 * * *` (daily at 9 AM).  
   - No credentials needed.

2. **Create "Get All Products" node**  
   - Type: Shopify  
   - Operation: Get All Products (`resource=product`, `operation=getAll`).  
   - Enable "Return All" to fetch all products.  
   - Connect input from "Daily Inventory Check".  
   - Configure with valid Shopify API credentials (API key, password, store URL, etc.).

3. **Create "Filter Low Stock Items" node**  
   - Type: Code (JavaScript)  
   - Paste the following code, adjusting threshold if desired:  
     ```javascript
     const lowStockThreshold = 10; // Modify threshold here
     const lowStockProducts = [];

     for (const product of $input.all()) {
       for (const variant of product.json.variants) {
         if (variant.inventory_quantity <= lowStockThreshold && variant.inventory_quantity > 0) {
           lowStockProducts.push({
             product_title: product.json.title,
             variant_title: variant.title,
             sku: variant.sku,
             current_stock: variant.inventory_quantity,
             product_id: product.json.id,
             variant_id: variant.id
           });
         }
       }
     }

     return lowStockProducts.map(item => ({ json: item }));
     ```  
   - Connect input from "Get All Products".

4. **Create "Check if Alerts Needed" node**  
   - Type: If  
   - Set condition: `{{ $('Filter Low Stock Items').all().length }} > 0`  
   - Connect input from "Filter Low Stock Items".

5. **Create "Format Alert Message" node**  
   - Type: Set  
   - Configure JSON output mode with this expression:  
     ```json
     {
       "alert_message": "🚨 *Low Stock Alert*\n\nThe following items are running low:\n\n{{ $('Filter Low Stock Items').all().map(item => `• ${item.json.product_title} (${item.json.variant_title}): ${item.json.current_stock} left`).join('\n') }}",
       "products_count": "{{ $('Filter Low Stock Items').all().length }}"
     }
     ```  
   - Connect input from the "True" output of "Check if Alerts Needed".

6. **Create "Send Inventory Alert" node**  
   - Type: Slack  
   - Select "Send Message" operation.  
   - Set the message text to `{{ $json.alert_message }}`.  
   - Set channel ID to your Slack target channel (replace `"C1234567890"` with your actual channel ID).  
   - Authenticate with Slack OAuth2 credentials with permissions to post messages.  
   - Connect input from "Format Alert Message".

7. **(Optional) Add Sticky Notes**  
   - Add notes for workflow info, features & config, business value, and threshold guide as per original workflow content, to aid maintainers and users.

8. **Activate the workflow**  
   - Test the workflow with a manual run or wait for the scheduled trigger.  
   - Monitor for errors, especially related to API authentication or Slack message delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow author: DAVID OLUSOLA                                                               | Author credit                                                                                           |
| Workflow purpose: Automatically monitor Shopify inventory levels and alert when low          | Summary from "Workflow Info" sticky note                                                               |
| Key Features: Daily automated checks, customizable thresholds, multi-variant support         | From "Features & Config" sticky note                                                                    |
| Threshold customization: Adjust `lowStockThreshold` in the Code node                         | From "Threshold Guide" sticky note                                                                      |
| Business Value: Prevent stockouts, maintain revenue, better inventory management              | From "Business Value" sticky note                                                                        |
| Slack channel ID must be replaced with your actual Slack channel ID                          | Important configuration note for Slack node                                                            |
| Shopify API credentials must have permissions to read products                               | Critical for "Get All Products" node                                                                    |
| Slack OAuth2 credentials need chat:write scope                                              | Required for posting messages                                                                            |
| Timezone considerations: Cron triggers run server timezone; adjust if necessary             | To ensure daily check occurs at intended time                                                          |
| Potential expansions: Add email alerts or SMS notifications                                 | Mentioned as possible workflow enhancements                                                            |
| Link to n8n Slack node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/ | For configuring Slack node in detail                                                                     |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow and complies fully with current content policies. All data processed is legal and publicly accessible.