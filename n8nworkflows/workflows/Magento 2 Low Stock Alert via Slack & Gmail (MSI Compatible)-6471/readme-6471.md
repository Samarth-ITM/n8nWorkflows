Magento 2 Low Stock Alert via Slack & Gmail (MSI Compatible)

https://n8nworkflows.xyz/workflows/magento-2-low-stock-alert-via-slack---gmail--msi-compatible--6471


# Magento 2 Low Stock Alert via Slack & Gmail (MSI Compatible)

### 1. Workflow Overview

This workflow automates daily monitoring of inventory stock levels for a Magento 2 (Adobe Commerce) store supporting Multi-Source Inventory (MSI). Its primary goal is to detect products with low stock and proactively send alerts via Slack and Gmail to notify inventory managers, helping prevent stockouts and maintain sales continuity.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger and Data Retrieval**  
  Automatically triggers daily, then fetches all product SKUs from Magento 2 via API.

- **1.2 MSI Stock Status Retrieval**  
  Uses the retrieved SKUs to query Magento 2 MSI source stock data for multiple inventory sources.

- **1.3 Stock Data Processing and Filtering**  
  Processes MSI stock data to identify products with quantity below defined thresholds.

- **1.4 Alert Decision Logic**  
  Checks if any low stock alerts are needed based on processed data.

- **1.5 Alert Message Formatting**  
  Formats alert messages into both plain text (for Slack) and HTML (for Gmail) with detailed tables.

- **1.6 Notification Sending**  
  Sends formatted alerts via Slack channel and Gmail email.

Supporting this logic, multiple sticky notes provide configuration, feature descriptions, and workflow guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Data Retrieval

**Overview:**  
This block triggers the workflow daily at a specified time and retrieves all relevant product SKUs from Magento 2, excluding certain product types (configurable, virtual, downloadable).

**Nodes Involved:**  
- Daily Inventory Check  
- Get All Product Skus  
- Extract SKUs

**Node Details:**

- **Daily Inventory Check**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow daily at 8:50 AM server time via cron expression.  
  - Configuration: Cron expression `"50 8 * * *"` triggers at 08:50 daily.  
  - Inputs: None (trigger node)  
  - Outputs: To "Get All Product Skus"  
  - Edge Cases: Timezone differences may affect trigger time; ensure server timezone aligns with expected schedule.

- **Get All Product Skus**  
  - Type: HTTP Request  
  - Role: Fetches all non-configurable, non-virtual, non-downloadable product SKUs from Magento REST API.  
  - Configuration:  
    - Method: GET  
    - URL includes filters to exclude product types  
    - Authentication: HTTP Bearer token (Magento API token)  
    - Fields requested: SKU and stock item management flags  
  - Key Expression: URL dynamically constructed with static filtering parameters.  
  - Inputs: From Schedule Trigger  
  - Outputs: To "Extract SKUs"  
  - Edge Cases: API authentication failures, rate limits, or network issues can cause request failures.

- **Extract SKUs**  
  - Type: Code (JavaScript)  
  - Role: Parses the JSON response to extract an array of all SKUs.  
  - Configuration:  
    - Iterates over `items` array in API response  
    - Collects SKUs into a plain array  
  - Key Expressions: Accesses `$input.first().json.items` and extracts `sku` fields.  
  - Inputs: HTTP Request output  
  - Outputs: To "Fetch MSI Stock Status"  
  - Edge Cases: Empty or malformed responses; missing `items` field.

---

#### 1.2 MSI Stock Status Retrieval

**Overview:**  
Using the SKU list, this block queries Magento 2 MSI API to retrieve stock status and quantities for each SKU across all inventory sources.

**Nodes Involved:**  
- Fetch MSI Stock Status

**Node Details:**

- **Fetch MSI Stock Status**  
  - Type: HTTP Request  
  - Role: Requests MSI source-item stock data filtered by SKUs (comma-separated).  
  - Configuration:  
    - Method: GET  
    - URL dynamically built with SKU list from previous node, URL encoded  
    - Authentication: HTTP Bearer token (Magento API token)  
  - Key Expression: URL parameter includes encoded SKU list from `Extract SKUs` node JSON field `skus`.  
  - Inputs: From "Extract SKUs"  
  - Outputs: To "Process Stock Data and Identify Low Stock"  
  - Edge Cases:  
    - API quota limits  
    - Empty SKU list results in no data  
    - Network or auth errors  
  - Notes: Supports MSI by querying source-specific stock data.

---

#### 1.3 Stock Data Processing and Filtering

**Overview:**  
This block analyzes the MSI stock data to identify products with low inventory based on configurable thresholds.

**Nodes Involved:**  
- Process Stock Data and Identify Low Stock

**Node Details:**

- **Process Stock Data and Identify Low Stock**  
  - Type: Code (JavaScript)  
  - Role: Iterates over the MSI stock items and filters those with quantity below thresholds.  
  - Configuration:  
    - Defines `lowStockThreshold` = 10 units as main alert threshold  
    - Defines `minSourceQuantity` = 5 units for minimum per source  
    - Checks if each item's stock status is 'In Stock' (status=1) and quantity ≤ thresholds  
  - Key Expressions: Accesses `$input.first().json.items` representing stock items  
  - Outputs: JSON object with array `lowStockAlerts` listing SKU, source, quantity, and status for low stock items  
  - Inputs: From "Fetch MSI Stock Status"  
  - Outputs: To "Check if Alerts Needed"  
  - Edge Cases:  
    - No low stock items returns empty array  
    - Data inconsistency (missing fields) may cause errors  
    - Thresholds hardcoded, must be edited for customization

---

#### 1.4 Alert Decision Logic

**Overview:**  
Determines if any low stock alerts exist and conditionally routes workflow to format and send alerts only when needed.

**Nodes Involved:**  
- Check if Alerts Needed

**Node Details:**

- **Check if Alerts Needed**  
  - Type: If (Conditional)  
  - Role: Checks if the `lowStockAlerts` array length is greater than zero, i.e., if any low stock items exist.  
  - Configuration:  
    - Condition: `={{ $node["Process Stock Data and Identify Low Stock"].json["lowStockAlerts"].length > 0 }}`  
    - Loose type validation enabled for flexibility  
  - Inputs: From "Process Stock Data and Identify Low Stock"  
  - Outputs:  
    - True: Passes to "Format Message" for alert generation  
    - False: Workflow ends silently (no alert)  
  - Edge Cases:  
    - If `lowStockAlerts` is undefined or not an array, expression may fail unless loose validation handles it.

---

#### 1.5 Alert Message Formatting

**Overview:**  
Prepares alert content in both Slack-friendly plain text and Gmail-compatible HTML formats, including detailed tables of low stock items.

**Nodes Involved:**  
- Format Message

**Node Details:**

- **Format Message**  
  - Type: Code (JavaScript)  
  - Role: Builds multi-format alert messages including:  
    - Plain text with markdown-like formatting for Slack  
    - HTML table for rich email content  
  - Configuration:  
    - Reads `lowStockAlerts` array  
    - If empty, outputs "No low stock items found at this time" message  
    - Otherwise, constructs tables with SKU, Source, Quantity, Status columns  
    - Generates subject line with current date  
  - Outputs:  
    - `slackMessage`: plain text alert message  
    - `gmailHtml`: HTML formatted message for email body  
    - `subjectLine`: email subject with date  
  - Inputs: From "Check if Alerts Needed" (true branch)  
  - Outputs: To both "Send Inventory Alert" (Slack) and "Send a message" (Gmail)  
  - Edge Cases:  
    - Formatting relies on consistent data structure  
    - Date formatting uses US locale, may be adjusted for locale-specific needs.

---

#### 1.6 Notification Sending

**Overview:**  
Sends the formatted alert messages to designated Slack channel and Gmail recipient.

**Nodes Involved:**  
- Send Inventory Alert (Slack)  
- Send a message (Gmail)

**Node Details:**

- **Send Inventory Alert**  
  - Type: Slack  
  - Role: Posts the plain text alert message to a Slack channel.  
  - Configuration:  
    - Channel: `#magento-notifications` (by name)  
    - Message text: from `slackMessage` field of input JSON  
    - Includes no link to the workflow  
  - Inputs: From "Format Message"  
  - Outputs: None (terminal)  
  - Edge Cases:  
    - Slack token or webhook issues may cause failure  
    - Channel name must exist and be accessible by app/bot

- **Send a message**  
  - Type: Gmail  
  - Role: Sends an email with the low stock alert in HTML format.  
  - Configuration:  
    - Recipient: `kmyprojects@gmail.com` (hardcoded)  
    - Subject: dynamic from `subjectLine` field  
    - Message body: HTML from `gmailHtml` field  
    - Attribution disabled (no "Sent via n8n")  
  - Inputs: From "Format Message"  
  - Outputs: None (terminal)  
  - Edge Cases:  
    - Gmail OAuth2 credentials must be valid and authorized  
    - Recipient email must accept messages, otherwise bounces  
    - HTML formatting depends on email client compatibility

---

### 3. Summary Table

| Node Name                               | Node Type          | Functional Role                        | Input Node(s)                          | Output Node(s)                                  | Sticky Note                                                                                   |
|----------------------------------------|--------------------|-------------------------------------|--------------------------------------|------------------------------------------------|-----------------------------------------------------------------------------------------------|
| Daily Inventory Check                   | Schedule Trigger   | Triggers daily at 8:50 AM            | None                                 | Get All Product Skus                            | Trigs Daily 8:50am                                                                            |
| Get All Product Skus                    | HTTP Request       | Fetches product SKUs from Magento   | Daily Inventory Check                 | Extract SKUs                                    | Magento Logic                                                                                |
| Extract SKUs                           | Code               | Extracts SKU list from API response  | Get All Product Skus                  | Fetch MSI Stock Status                           | Magento Logic                                                                                |
| Fetch MSI Stock Status                  | HTTP Request       | Retrieves MSI stock data for SKUs   | Extract SKUs                         | Process Stock Data and Identify Low Stock       | Magento Logic                                                                                |
| Process Stock Data and Identify Low Stock | Code             | Identifies low stock items           | Fetch MSI Stock Status                | Check if Alerts Needed                           | Checks & Message Formatting                                                                  |
| Check if Alerts Needed                  | If                 | Conditional check for alerts needed | Process Stock Data and Identify Low Stock | Format Message                                 | Checks & Message Formatting                                                                  |
| Format Message                         | Code               | Formats alert message for Slack & Gmail | Check if Alerts Needed               | Send Inventory Alert, Send a message            | Checks & Message Formatting                                                                  |
| Send Inventory Alert                   | Slack              | Sends alert message to Slack channel | Format Message                      | None                                           | Notifications                                                                               |
| Send a message                        | Gmail              | Sends alert email                   | Format Message                      | None                                           | Notifications                                                                               |
| Workflow Info                         | Sticky Note        | Workflow purpose, setup requirements | None                                 | None                                           |                                                                                               |
| Features & Config                     | Sticky Note        | Key feature highlights              | None                                 | None                                           |                                                                                               |
| Business Value                       | Sticky Note        | Business benefits and ROI           | None                                 | None                                           |                                                                                               |
| Sticky Note                         | Sticky Note        | Workflow step summary               | None                                 | None                                           |                                                                                               |
| Sticky Note1                        | Sticky Note        | Configuration notes                 | None                                 | None                                           |                                                                                               |
| Sticky Note2                        | Sticky Note        | Schedule trigger annotation         | None                                 | None                                           | Trigs Daily 8:50am                                                                            |
| Sticky Note3                        | Sticky Note        | Magento logic grouping              | None                                 | None                                           | Magento Logic                                                                                |
| Sticky Note4                        | Sticky Note        | Notifications grouping              | None                                 | None                                           | Notifications                                                                               |
| Sticky Note5                        | Sticky Note        | Checks and message formatting grouping | None                                 | None                                           | Checks & Message Formatting                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Daily Inventory Check`  
   - Type: Schedule Trigger  
   - Set to trigger daily at 8:50 AM using cron expression `50 8 * * *`

2. **Create HTTP Request Node to Get Product SKUs**  
   - Name: `Get All Product Skus`  
   - Type: HTTP Request  
   - Method: GET  
   - URL:  
     ```
     https://magekwik.com/rest/V1/products?searchCriteria[pageSize]=0&searchCriteria[filterGroups][0][filters][0][field]=type_id&searchCriteria[filterGroups][0][filters][0][conditionType]=nin&searchCriteria[filterGroups][0][filters][0][value]=configurable,virtual,downloadable&fields=items[sku,extension_attributes[stock_item[manage_stock]]]
     ```  
   - Authentication: HTTP Bearer with Magento API token credential  
   - Connect output from `Daily Inventory Check`

3. **Create Code Node to Extract SKUs**  
   - Name: `Extract SKUs`  
   - Type: Code (JavaScript)  
   - Code:
     ```js
     const skus = [];
     for (const item of $input.first().json.items) {
       skus.push(item.sku);
     }
     return [{ json: { skus: skus } }];
     ```  
   - Connect input from `Get All Product Skus`

4. **Create HTTP Request Node to Fetch MSI Stock Status**  
   - Name: `Fetch MSI Stock Status`  
   - Type: HTTP Request  
   - Method: GET  
   - URL (Dynamic):  
     ```
     https://magekwik.com/rest/V1/inventory/source-items?searchCriteria[filterGroups][0][filters][0][field]=sku&searchCriteria[filterGroups][0][filters][0][conditionType]=in&searchCriteria[filterGroups][0][filters][0][value]={{ encodeURIComponent($node["Extract SKUs"].json["skus"].join(',')) }}
     ```  
   - Authentication: HTTP Bearer with Magento API token credential  
   - Connect input from `Extract SKUs`

5. **Create Code Node to Process Stock Data and Identify Low Stock**  
   - Name: `Process Stock Data and Identify Low Stock`  
   - Type: Code (JavaScript)  
   - Code:
     ```js
     const lowStockItems = [];
     const lowStockThreshold = 10; // Customize threshold as needed
     const minSourceQuantity = 5;

     for (const item of $input.first().json.items) {
       const sku = item.sku;
       const sourceCode = item.source_code;
       const quantity = item.quantity;
       const status = item.status;

       if (status === 1 && (quantity <= lowStockThreshold || quantity <= minSourceQuantity)) {
         lowStockItems.push({
           sku: sku,
           source: sourceCode,
           quantity: quantity,
           status: status === 1 ? 'In Stock' : 'Out of Stock'
         });
       }
     }
     return [{ json: { lowStockAlerts: lowStockItems } }];
     ```  
   - Connect input from `Fetch MSI Stock Status`

6. **Create If Node to Check if Alerts Needed**  
   - Name: `Check if Alerts Needed`  
   - Type: If  
   - Condition:  
     - Expression: `={{ $node["Process Stock Data and Identify Low Stock"].json["lowStockAlerts"].length > 0 }}`  
     - Loose type validation enabled  
   - Connect input from `Process Stock Data and Identify Low Stock`

7. **Create Code Node to Format Message**  
   - Name: `Format Message`  
   - Type: Code (JavaScript)  
   - Code:
     ```js
     const lowStockAlerts = Array.isArray($input.first().json.lowStockAlerts) ? $input.first().json.lowStockAlerts : [];

     let plainText = "*📦 Low Stock Alert for Magento 2 Multi-Store (MSI)*\n\n";
     let htmlText = "<b>📦 Low Stock Alert for Magento 2 Multi-Store (MSI)</b><br><br>";

     if (lowStockAlerts.length === 0) {
       plainText += "_No low stock items found at this time._";
       htmlText += "<i>No low stock items found at this time.</i>";
     } else {
       plainText += "*The following products have low stock:*\n\n";
       htmlText += "<b>The following products have low stock:</b><br><br>";

       plainText += "```\nSKU       Source     Quantity  Status\n";
       plainText += "--------  ---------  --------  --------\n";

       htmlText += `<table border="1" cellpadding="6" cellspacing="0" style="border-collapse: collapse;">
         <thead>
           <tr>
             <th align="left">SKU</th>
             <th align="left">Source</th>
             <th align="left">Quantity</th>
             <th align="left">Status</th>
           </tr>
         </thead>
         <tbody>
       `;

       lowStockAlerts.forEach(item => {
         const sku = item.sku || 'N/A';
         const source = item.source || 'N/A';
         const quantity = item.quantity || 0;
         const status = item.status || 'Unknown';

         plainText += `${sku.padEnd(10)}${source.padEnd(11)}${String(quantity).padEnd(9)}${status}\n`;

         htmlText += `<tr>
           <td>${sku}</td>
           <td>${source}</td>
           <td>${quantity}</td>
           <td>${status}</td>
         </tr>`;
       });

       plainText += "```\n\n🔍 Please check your Magento 2 inventory.";
       htmlText += "</tbody></table><br><br>🔍 <b>Please check your Magento 2 inventory.</b>";
     }

     return [{
       json: {
         slackMessage: plainText,
         gmailHtml: htmlText,
         subjectLine: `🔔 Magento 2 Low Stock Alert – ${new Date().toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })}`
       }
     }];
     ```
   - Connect input from `Check if Alerts Needed` (True branch)

8. **Create Slack Node to Send Inventory Alert**  
   - Name: `Send Inventory Alert`  
   - Type: Slack  
   - Authentication: Slack credentials with posting permissions  
   - Channel: `#magento-notifications` (by name)  
   - Message text: `={{ $json.slackMessage }}`  
   - Disable "include link to workflow" option  
   - Connect input from `Format Message`

9. **Create Gmail Node to Send Email**  
   - Name: `Send a message`  
   - Type: Gmail  
   - Authentication: OAuth2 Gmail credentials  
   - Recipient: `kmyprojects@gmail.com`  
   - Subject: `={{ $json.subjectLine }}`  
   - Message (HTML): `={{ $json.gmailHtml }}`  
   - Disable attribution footer  
   - Connect input from `Format Message`

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                  |
|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow author: Kanaka Kishore Kandregula                                                                          | Author credit in workflow info node                             |
| Workflow purpose: Automated Magento 2 inventory alerts with MSI support                                              | Workflow Info sticky note                                        |
| Setup requires Magento 2 API Bearer Token, Slack, and Gmail credentials                                              | Configuration Notes sticky note                                  |
| Key features include MSI support, daily automated checks, customizable thresholds, and multi-SKU reporting          | Features & Config sticky note                                    |
| Business value highlights prevention of stockouts, revenue maintenance, better inventory management, proactive alerts | Business Value sticky note                                      |
| Workflow runs daily at 8:50 AM server time via cron schedule                                                        | Sticky Note2, Daily Inventory Check node                        |
| Slack alerts post to `#magento-notifications` channel                                                                | Slack node configuration                                        |
| Email alerts send to `kmyprojects@gmail.com`                                                                         | Gmail node configuration                                        |
| Low stock thresholds are set in code node and should be customized as needed                                         | Process Stock Data and Identify Low Stock node                  |
| The date format in email subject uses US English locale, adjust if needed                                            | Format Message node, date formatting                            |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automated workflow. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.