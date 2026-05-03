Notify on Slack when new product is added in WooCommerce

https://n8nworkflows.xyz/workflows/notify-on-slack-when-new-product-is-added-in-woocommerce-1458


# Notify on Slack when new product is added in WooCommerce

### 1. Workflow Overview

This n8n workflow automatically notifies a Slack channel whenever a new product is created in a WooCommerce store. It is designed for e-commerce teams or businesses that want to keep their Slack workspace updated instantly about new product additions without manual checks.

The workflow is logically divided into two main blocks:

- **1.1 WooCommerce Trigger:** Listens for new product creation events in WooCommerce.
- **1.2 Slack Notification:** Sends a formatted message to a specified Slack channel with product details.

---

### 2. Block-by-Block Analysis

#### 1.1 WooCommerce Trigger

- **Overview:**  
  This block monitors the WooCommerce store for new product creation events and triggers the workflow execution when such an event occurs.

- **Nodes Involved:**  
  - *Product Created*

- **Node Details:**

  - **Node Name:** Product Created  
  - **Type:** WooCommerce Trigger  
  - **Technical Role:** Event trigger node that listens to WooCommerce webhook events.  
  - **Configuration Choices:**  
    - Trigger event set to `"product.created"`, meaning the node listens specifically for new products added to the WooCommerce store.  
    - Webhook ID assigned internally by n8n for this trigger.  
    - Credentials: Uses the configured WooCommerce API credentials (`WooCommerce account`), which must have appropriate permissions and API keys for the store.  
  - **Expressions / Variables:** None (this node outputs the entire product data JSON on trigger).  
  - **Input Connections:** None (it is a trigger node).  
  - **Output Connections:** Connected to the "Send to Slack" node as the main output.  
  - **Version-Specific Requirements:** Requires WooCommerce API version compatible with webhook-based product creation events. Ensure the WooCommerce store supports webhooks and API access.  
  - **Potential Failures / Edge Cases:**  
    - Authentication errors if WooCommerce credentials are invalid or expired.  
    - Webhook registration failures on WooCommerce side.  
    - Network timeouts or API limits from WooCommerce.  
    - Event filtering: Only triggers on product creation, so updates or deletions will not trigger.  
  - **Sub-workflow:** None.

#### 1.2 Slack Notification

- **Overview:**  
  This block receives product data from the WooCommerce trigger and posts a detailed message to a predefined Slack channel, informing the team about the new product.

- **Nodes Involved:**  
  - *Send to Slack*

- **Node Details:**

  - **Node Name:** Send to Slack  
  - **Type:** Slack node (message sender)  
  - **Technical Role:** Posts a message to a Slack channel using Slack API.  
  - **Configuration Choices:**  
    - Message Text: A static string with emojis `":new: A new product has been added! :new:"` to highlight the notification.  
    - Channel: Set to `"woo-commerce"` (channel name or ID where messages should be posted).  
    - Attachments: The Slack message includes an attachment with:  
      - Green color strip (`#66FF00`) indicating success or new item.  
      - Fields showing key product details extracted dynamically from the incoming JSON:  
        - Name (`{{$json["name"]}}`)  
        - Price (`{{$json["regular_price"]}}`)  
        - Sale Price (`{{$json["sale_price"]}}`)  
        - Link (`{{$json["permalink"]}}`)  
      - Footer showing the product creation date (`Added: {{$json["date_created"]}}`).  
    - Blocks UI array left empty, using attachments for formatting.  
    - Credentials: Uses Slack API credentials (`Slack Access Token`) with permission to post messages in the target channel.  
  - **Expressions / Variables:** Dynamically uses product properties from the incoming JSON to populate attachment fields and footer.  
  - **Input Connections:** Receives input from the "Product Created" node.  
  - **Output Connections:** None (terminal node).  
  - **Version-Specific Requirements:** Requires Slack API token with `chat:write` permission and the channel must be accessible by the bot/user.  
  - **Potential Failures / Edge Cases:**  
    - Authentication failures if Slack token is invalid or expired.  
    - Posting to a non-existent or restricted Slack channel.  
    - Missing product fields in the incoming data (e.g., if price or permalink is not available).  
    - Slack API rate limits or network issues.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name      | Node Type                  | Functional Role            | Input Node(s)    | Output Node(s) | Sticky Note                                                                                 |
|----------------|----------------------------|----------------------------|------------------|----------------|---------------------------------------------------------------------------------------------|
| Product Created| WooCommerce Trigger        | Detect new WooCommerce product creation | None             | Send to Slack  | To use this workflow you will need to set the credentials to use for the WooCommerce node.  |
| Send to Slack  | Slack                      | Post new product notification to Slack channel | Product Created | None           | You will also need to pick a channel to post the message to and set Slack credentials.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WooCommerce Trigger Node**  
   - Add a new node of type `WooCommerce Trigger`.  
   - Set the event parameter to `product.created` to listen for new product additions.  
   - Configure credentials by creating or selecting a WooCommerce API credential with valid API keys and permissions.  
   - Ensure WooCommerce store webhooks are enabled and accessible by n8n.

2. **Create Slack Node**  
   - Add a new node of type `Slack`.  
   - Configure credentials by creating or selecting a Slack API credential with an access token that has `chat:write` permissions.  
   - Set the message text to: `":new: A new product has been added! :new:"`.  
   - Set the target channel to `woo-commerce` (change this to your desired Slack channel name or ID).  
   - Configure the message attachments as follows:  
     - Color: `#66FF00` (green).  
     - Fields:  
       - Title: "Name", Value: `={{$json["name"]}}`, not short.  
       - Title: "Price", Value: `={{$json["regular_price"]}}`, short.  
       - Title: "Sale Price", Value: `={{$json["sale_price"]}}`, short.  
       - Title: "Link", Value: `={{$json["permalink"]}}`, not short.  
     - Footer: `=Added: {{$json["date_created"]}}`.  
   - Leave Blocks UI empty.

3. **Connect Nodes**  
   - Connect the output of the WooCommerce Trigger node (`Product Created`) to the input of the Slack node (`Send to Slack`).

4. **Activate Workflow**  
   - Save and activate the workflow.  
   - Verify that the WooCommerce webhook is properly registered and reachable by n8n.

5. **Test Workflow**  
   - Add a new product in WooCommerce.  
   - Confirm that a notification appears in the Slack channel with correct product details.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                             |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| To use this workflow, ensure you have valid credentials for both WooCommerce API and Slack API. | Workflow prerequisites                                       |
| Make sure the Slack bot/user has permission to post messages in the selected channel.           | Slack API permissions                                        |
| WooCommerce webhooks must be enabled and accessible by n8n; check webhook settings in WooCommerce. | WooCommerce webhook setup                                   |
| Message formatting uses Slack attachments for rich display; you may customize fields as needed. | Slack message formatting documentation                       |
| Workflow is inactive by default; activate it after configuration.                              | n8n workflow lifecycle                                       |