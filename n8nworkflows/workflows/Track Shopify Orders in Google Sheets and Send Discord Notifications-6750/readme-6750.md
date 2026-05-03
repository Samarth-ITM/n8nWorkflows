Track Shopify Orders in Google Sheets and Send Discord Notifications

https://n8nworkflows.xyz/workflows/track-shopify-orders-in-google-sheets-and-send-discord-notifications-6750


# Track Shopify Orders in Google Sheets and Send Discord Notifications

### 1. Workflow Overview

This workflow automates the tracking of Shopify orders by recording order details into a Google Sheet and simultaneously sending notifications to a Discord channel. It is designed for Shopify store owners who want to maintain an up-to-date orders log and receive immediate alerts for new orders in Discord.

The workflow is structured into the following logical blocks:

- **1.1 Shopify Order Reception:** Listens for new order creation events from Shopify.
- **1.2 Order Data Processing:** Extracts and formats line item details from the Shopify order.
- **1.3 Google Sheets Logging:** Appends the processed order details as a new row in a designated Google Sheet.
- **1.4 Discord Notification:** Constructs a detailed message about the order and sends it via Discord webhook.

---

### 2. Block-by-Block Analysis

#### 1.1 Shopify Order Reception

- **Overview:**  
  This block triggers the workflow whenever a new order is created in the connected Shopify store.

- **Nodes Involved:**  
  - Shopify Trigger

- **Node Details:**  
  - **Shopify Trigger**  
    - *Type & Role:* Event trigger node for Shopify, listens to order creation events.  
    - *Configuration:* Subscribed to the "orders/create" topic using OAuth2 access token authentication.  
    - *Expressions:* None, it directly receives the webhook payload from Shopify.  
    - *Input/Output:* No input; outputs raw order JSON data.  
    - *Version:* Type Version 1.  
    - *Possible Failures:* Authentication errors if the access token expires or is revoked; network issues causing webhook delivery failure.  
    - *Notes:* Requires a valid Shopify Access Token credential configured with appropriate scopes.

#### 1.2 Order Data Processing

- **Overview:**  
  Extracts line item titles and prices from the incoming Shopify order data for easier use downstream.

- **Nodes Involved:**  
  - Code (Python)

- **Node Details:**  
  - **Code (Python)**  
    - *Type & Role:* Code node running Python to parse the Shopify order's `line_items` array and concatenate titles and prices into comma-separated strings.  
    - *Configuration:* Reads `line_items` from the incoming JSON, creates two strings: one for item titles and one for prices. Returns these as new JSON fields `line_item_titles` and `line_item_prices`.  
    - *Expressions:* Uses `items[0]["json"]["line_items"]` to access data.  
    - *Input/Output:* Input from Shopify Trigger; output is JSON with two new fields for titles and prices.  
    - *Version:* Type Version 2.  
    - *Possible Failures:* Unexpected data structure changes in Shopify order JSON could cause key errors; empty or missing `line_items` array; Python runtime errors if the environment is misconfigured.

#### 1.3 Google Sheets Logging

- **Overview:**  
  Appends a new row to a specified Google Sheet with detailed order information, including the processed line item data.

- **Nodes Involved:**  
  - Append row in sheet (Google Sheets node)

- **Node Details:**  
  - **Append row in sheet**  
    - *Type & Role:* Google Sheets node configured to append rows.  
    - *Configuration:*  
      - Operation: Append  
      - Document ID: Points to a specific Google Sheets document with stored orders (sheet name by GID=0).  
      - Columns: Maps Shopify order fields and the processed line item data into predefined columns (Order Number, Customer Email, Customer Name, City, Country, Order Total, Currency, Subtotal, Tax, Financial Status, Payment Gateway, Order Date, Line Item Titles, Line Item Prices, Order Link).  
      - Mapping Mode: Define below, with explicit schema for each column.  
    - *Expressions:* Each column uses expressions to extract data from the trigger and code node outputs, e.g., `{{$('Shopify Trigger').item.json.current_total_tax}}` and `{{$json.line_item_titles}}`.  
    - *Input/Output:* Input from Code (Python) node; output passes the appended row data forward.  
    - *Version:* Type Version 4.6.  
    - *Possible Failures:* Authentication errors with Google credentials; permission issues if the sheet is deleted or access revoked; schema mismatches; rate limits on Google Sheets API.

#### 1.4 Discord Notification

- **Overview:**  
  Formats a rich order summary message and sends it to a Discord channel via webhook to notify about the new order.

- **Nodes Involved:**  
  - Code1 (JavaScript)  
  - Discord

- **Node Details:**  
  - **Code1 (JavaScript)**  
    - *Type & Role:* Code node that builds a Markdown-formatted Discord message summarizing the order details, including items, prices, customer info, and order link.  
    - *Configuration:*  
      - Parses `Line Item Titles` and `Line Item Prices` (comma-separated strings) into arrays.  
      - Constructs a multi-line message with order number, customer, location, date, ordered items (with price and currency), totals, payment status/method, and a clickable order link.  
    - *Expressions:* Uses JavaScript string template literals and array mapping.  
    - *Input/Output:* Input from Google Sheets append node; output JSON with a single field `content` for Discord message.  
    - *Version:* Type Version 2.  
    - *Possible Failures:* Parsing errors if line item fields are empty or malformed; message size limits on Discord; escaping issues in text formatting.

  - **Discord**  
    - *Type & Role:* Discord node configured to send messages via webhook authentication.  
    - *Configuration:*  
      - Content sourced directly from the previous Code1 node’s output `content` field.  
      - Authentication via Discord Webhook credential.  
    - *Input/Output:* Input from Code1 node; no output further in this workflow.  
    - *Version:* Type Version 2.  
    - *Possible Failures:* Invalid or revoked webhook URL; network errors; Discord rate limiting or API changes.

---

### 3. Summary Table

| Node Name           | Node Type              | Functional Role           | Input Node(s)     | Output Node(s)  | Sticky Note                                   |
|---------------------|------------------------|--------------------------|-------------------|-----------------|----------------------------------------------|
| Shopify Trigger     | Shopify Trigger         | Listen for new Shopify orders | None              | Code            |                                              |
| Code                | Code (Python)           | Extract line item titles and prices | Shopify Trigger   | Append row in sheet |                                              |
| Append row in sheet | Google Sheets           | Append order data to Google Sheet | Code              | Code1           |                                              |
| Code1               | Code (JavaScript)       | Format Discord order message | Append row in sheet | Discord         |                                              |
| Discord             | Discord                 | Send order notification to Discord | Code1             | None            |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Shopify Trigger Node**  
   - Create a new node of type **Shopify Trigger**.  
   - Set **Topic** to `orders/create`.  
   - Choose **Authentication** method as `accessToken`.  
   - Configure credentials with a valid **Shopify Access Token** that has permissions to read orders.  
   - Position this node as the workflow start trigger.

2. **Code Node (Python) – Extract Line Items**  
   - Add a **Code** node, set language to **Python**.  
   - Paste the following code to parse line items:

     ```python
     line_items = items[0]["json"]["line_items"]
     titles = ", ".join([item["title"] for item in line_items])
     prices = ", ".join([str(item["price"]) for item in line_items])
     return [{"json": {"line_item_titles": titles, "line_item_prices": prices}}]
     ```
   - Connect the output of **Shopify Trigger** to this node.

3. **Google Sheets Node – Append Row**  
   - Add a **Google Sheets** node.  
   - Set operation to **Append**.  
   - Specify the **Document ID** of the Google Sheet where orders will be logged.  
   - Set **Sheet Name** using the sheet's GID (e.g., `gid=0`).  
   - Define column schema with fields such as:  
     - Order Number, Customer Email, Customer Name, City, Country, Order Total, Currency, Subtotal, Tax, Financial Status, Payment Gateway, Order Date, Line Item Titles, Line Item Prices, Order Link.  
   - Map each column to expressions pulling data from the previous nodes, e.g.:  
     - Tax: `{{$('Shopify Trigger').item.json.current_total_tax}}`  
     - City: `{{$('Shopify Trigger').item.json.billing_address.city}}`  
     - Line Item Titles: `{{$json.line_item_titles}}`  
     - Line Item Prices: `{{$json.line_item_prices}}`  
   - Configure **Google Sheets OAuth2** credentials with access to this spreadsheet.  
   - Connect the **Code (Python)** node output to this node.

4. **Code Node (JavaScript) – Format Discord Message**  
   - Add a **Code** node with language set to **JavaScript**.  
   - Use the following code to build the Discord message:

     ```javascript
     const order = items[0].json;

     const prices = order["Line Item Prices"].split(', ');
     const titles = order["Line Item Titles"].split(', ');

     const message = `🛍️ **New Order Received!**

     **Order #:** ${order["Order Number"]}
     **Customer:** ${order["Customer Name"]} (${order["Customer Email"]})
     **Location:** ${order["City"]}, ${order["Country"]}
     **Date:** ${order["Order Date"]}

     🧾 **Items Ordered:**
     ${titles.map((title, i) => `- ${title} (${prices[i]} ${order["Currency"]})`).join('\n')}

     💳 **Total:** ${order["Order Total"]} ${order["Currency"]}
     **Subtotal:** ${order["Subtotal"]} | **Tax:** ${order["Tax"]} ${order["Currency"]}
     **Payment Status:** ${order["Financial Status"]}
     **Payment Method:** ${order["Payment Gateway"]}

     🔗 [View Order](${order["Order Link"]})
     `;

     return [{ json: { content: message } }];
     ```
   - Connect the output of the Google Sheets node to this node.

5. **Discord Node – Send Notification**  
   - Add a **Discord** node configured for **Webhook** authentication.  
   - Set **Content** field to `{{$json.content}}` which is the output from the previous code node.  
   - Use the Discord Webhook URL credential corresponding to the target channel.  
   - Connect the output of the JavaScript Code node to this Discord node.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                   |
|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow requires valid API credentials for Shopify, Google Sheets, and Discord Webhook.    | Credential setup in n8n for OAuth2 (Google) and API tokens (Shopify, Discord). |
| Google Sheets columns must match exactly the schema defined in the Append row node for data to map correctly. | Google Sheets document URL: https://docs.google.com/spreadsheets/d/178dfbyBMouRfyXP2OngRHylputb4qdzkwQV63Lk5Ev0 |
| Discord message formatting uses Markdown syntax for better readability in Discord channels.      | Discord Markdown guide: https://support.discord.com/hc/en-us/articles/210298617-Markdown-Text-101 |
| To avoid workflow failures, ensure Shopify webhooks are active and the Google Sheet is accessible.| Shopify webhook docs: https://shopify.dev/apps/webhooks                    |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.