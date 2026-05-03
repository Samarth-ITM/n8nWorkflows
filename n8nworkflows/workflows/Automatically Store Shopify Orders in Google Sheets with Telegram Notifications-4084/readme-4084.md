Automatically Store Shopify Orders in Google Sheets with Telegram Notifications

https://n8nworkflows.xyz/workflows/automatically-store-shopify-orders-in-google-sheets-with-telegram-notifications-4084


# Automatically Store Shopify Orders in Google Sheets with Telegram Notifications

### 1. Workflow Overview

This workflow automates the process of capturing new orders from a Shopify store, storing the order details in a Google Sheets spreadsheet, and sending real-time notifications via Telegram about the order status. It is designed for Shopify store owners or managers who want an automated, centralized order tracking system combined with instant alerts.

**Target Use Cases:**  
- Automatically logging Shopify orders into a Google Sheet for record-keeping and analysis.  
- Receiving immediate Telegram notifications on new orders or errors during processing.  
- Maintaining a standardized order data format for consistency across records.

**Logical Blocks:**  
- **1.1 Input Reception:** Capturing Shopify order data via webhook.  
- **1.2 Data Transformation:** Standardizing raw Shopify order data into a simplified, structured format.  
- **1.3 Data Storage:** Saving the transformed order data to Google Sheets.  
- **1.4 Outcome Evaluation & Notification:** Checking success or failure of the storage operation and sending appropriate Telegram notifications.  
- **1.5 Variables Configuration:** Managing essential variables such as spreadsheet ID and Telegram chat ID for reuse throughout the workflow.  
- **1.6 Documentation & Notes:** Sticky notes providing workflow description and error handling instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming Shopify order webhook POST requests, serving as the workflow’s entry point.

- **Nodes Involved:**  
  - Receive New Shopify Order

- **Node Details:**  
  - **Receive New Shopify Order**  
    - Type: Webhook  
    - Role: Receives POST webhook calls from Shopify containing new order JSON payloads.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `shopify-webhook` (must be configured in Shopify as the webhook URL endpoint)  
      - No authentication or header filters configured here.  
    - Input: HTTP POST request from Shopify containing order data.  
    - Output: JSON object with raw Shopify order payload under `body`.  
    - Edge Cases:  
      - Failure if Shopify webhook is misconfigured or network issues block delivery.  
      - Payload format change in Shopify API can cause parsing errors downstream.  
    - Version: n8n Webhook node v2.

---

#### 1.2 Data Transformation

- **Overview:**  
  This block extracts relevant fields from the complex Shopify order data and converts them into a simplified, consistent structure suitable for Google Sheets storage.

- **Nodes Involved:**  
  - Transform Order Data to Standard Format

- **Node Details:**  
  - **Transform Order Data to Standard Format**  
    - Type: Function  
    - Role: Code-based data parsing and restructuring. Converts raw Shopify order JSON into a flat, standardized object.  
    - Configuration:  
      - Custom JavaScript function extracts order id, number, timestamps, customer details (name, email, phone), shipping address, order line items with key attributes, total price, and currency.  
      - Handles missing fields gracefully by fallback logic (e.g., customer name derived from shipping or billing address or customer object).  
      - Converts nested objects (customer, shippingAddress, lineItems) to JSON strings for Google Sheets compatibility.  
      - Defaults `processed` field to false and leaves `processed_at` empty, indicating further processing state can be tracked later.  
    - Input: Raw JSON order data from webhook node.  
    - Output: Single JSON object with standardized fields ready for Google Sheets.  
    - Key Expressions: Uses conditional checks and mapping functions to build structured data.  
    - Edge Cases:  
      - Missing or malformed fields in Shopify order data.  
      - Unexpected data types (e.g., line_items not an array).  
      - JSON.stringify failure if circular references exist (unlikely in Shopify data).  
    - Version: Function node v1.

---

#### 1.3 Data Storage

- **Overview:**  
  This block appends the standardized order data as a new row in the configured Google Sheets document, enabling order tracking and reporting.

- **Nodes Involved:**  
  - Save Order to Google Sheets

- **Node Details:**  
  - **Save Order to Google Sheets**  
    - Type: Google Sheets  
    - Role: Append operation to add order data to a specific worksheet.  
    - Configuration:  
      - Operation: Append row  
      - Document ID: Google Sheet ID (configured via variables or directly)  
      - Sheet Name: The target sheet/tab name or ID where orders are recorded.  
      - Mapping Mode: Automatic mapping of incoming JSON fields to columns (autoMapInputData).  
      - No type conversion or forced string conversion enabled, preserving original data types.  
    - Input: Standardized order JSON object from the previous function node.  
    - Output: Success or error response from Google Sheets API.  
    - Credentials: Google Sheets OAuth2 authentication configured with authorized account.  
    - Edge Cases:  
      - Authentication errors if OAuth token expires or is invalid.  
      - Google Sheets API rate limits or quota exceeded errors.  
      - Incorrect Sheet ID or sheet name causing append failure.  
    - Version: Google Sheets node v4.2.

---

#### 1.4 Outcome Evaluation & Notification

- **Overview:**  
  This block evaluates if the data append was successful and sends either a success or error notification to a Telegram chat accordingly.

- **Nodes Involved:**  
  - Success?  
  - Send Success Notification1  
  - Send Error Notification

- **Node Details:**  
  - **Success?**  
    - Type: If  
    - Role: Conditional branching to check if the Google Sheets append operation succeeded.  
    - Configuration:  
      - Condition: Boolean check, hardcoded as `true == true` in the JSON but in practice this node is likely used as a placeholder or should be connected to actual success criteria (e.g., presence of output data).  
    - Input: Output from Google Sheets node.  
    - Output: Two branches: success and failure.

  - **Send Success Notification1**  
    - Type: Telegram  
    - Role: Sends a formatted message to Telegram chat notifying about the newly recorded order.  
    - Configuration:  
      - Message Text: Includes order number, customer name (parsed from JSON), total price, currency, and order date.  
      - Chat ID: Hardcoded or set from variables (needs to be configured).  
    - Credentials: Telegram Bot API credentials for authenticated sending.  
    - Input: Triggered from success condition branch.  
    - Output: Telegram API response.  
    - Edge Cases:  
      - Invalid chat ID or bot token causing send failure.  
      - Telegram API rate limits.  

  - **Send Error Notification**  
    - Type: Telegram  
    - Role: Sends an error alert message via Telegram if orders fail to save.  
    - Configuration:  
      - Message Text: Includes error details and timestamp.  
      - Chat ID: Set from variables node.  
    - Credentials: Telegram Bot API credentials.  
    - Input: Triggered from failure condition branch.  
    - Output: Telegram API response.  
    - Edge Cases: Same as success notification plus error object may be undefined if upstream node fails silently.

---

#### 1.5 Variables Configuration

- **Overview:**  
  This block centralizes configurable settings such as Google Sheets document ID, sheet name, and Telegram chat ID, enabling easier maintenance and reuse.

- **Nodes Involved:**  
  - Variables

- **Node Details:**  
  - **Variables**  
    - Type: Set  
    - Role: Defines key-value pairs for reusable configuration variables.  
    - Configuration:  
      - `spreadsheetId`: Google Sheet ID string placeholder.  
      - `sheetName`: Name of the sheet/tab (e.g., "orders").  
      - `telegramChatId`: Telegram chat ID string placeholder.  
    - Input: None (initializes static values).  
    - Output: Variables accessible throughout the workflow via expressions.  
    - Edge Cases:  
      - Missing or incorrect values will cause failures in Google Sheets or Telegram nodes.

---

#### 1.6 Documentation & Notes

- **Overview:**  
  Provides human-readable explanations, setup instructions, and error handling tips via sticky notes for workflow maintainers.

- **Nodes Involved:**  
  - Workflow Overview (sticky note)  
  - Error Handling Note (sticky note)

- **Node Details:**  
  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Describes workflow purpose, setup, and step-by-step logic.  
    - Content: Summary of functionality, required configurations, and process steps.

  - **Error Handling Note**  
    - Type: Sticky Note  
    - Role: Describes error branch logic and importance of Telegram bot/chat ID configuration.  
    - Content: Instructions on configuring error notification via Telegram.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                                  |
|-------------------------------|---------------------|----------------------------------------|--------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow Overview             | Sticky Note         | Workflow description and setup guide   | None                           | None                            | ## Shopify to Google Sheets Order Tracking with Telegram Notifications<br>Details on setup and workflow steps |
| Receive New Shopify Order     | Webhook             | Capture incoming Shopify order data    | None                           | Transform Order Data to Standard Format |                                                                                                              |
| Transform Order Data to Standard Format | Function            | Standardize Shopify order data format  | Receive New Shopify Order      | Save Order to Google Sheets      |                                                                                                              |
| Save Order to Google Sheets   | Google Sheets       | Append standardized order to spreadsheet | Transform Order Data to Standard Format | Success?                       |                                                                                                              |
| Success?                     | If                  | Check if order saving succeeded        | Save Order to Google Sheets    | Send Success Notification1, Send Error Notification |                                                                                                              |
| Send Success Notification1    | Telegram            | Notify successful order save via Telegram | Success? (success branch)      | None                            |                                                                                                              |
| Send Error Notification       | Telegram            | Notify failure in order processing via Telegram | Success? (failure branch)      | None                            |                                                                                                              |
| Variables                    | Set                 | Holds configurable workflow variables  | None                           | None                            |                                                                                                              |
| Error Handling Note           | Sticky Note         | Explains error handling and Telegram setup | None                           | None                            | ## Error Handling<br>Configure Telegram bot and chat ID for notifications                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it:**  
   "Automatically Store Shopify Orders in Google Sheets with Telegram Notifications"

2. **Add a Sticky Note node:**  
   - Name: "Workflow Overview"  
   - Content: Copy the detailed explanation about purpose, setup, and workflow steps as given in the Workflow Overview node.

3. **Add a Webhook node:**  
   - Name: "Receive New Shopify Order"  
   - HTTP Method: POST  
   - Path: `shopify-webhook`  
   - No authentication required.  
   - This node will serve as the entry point for Shopify webhook calls.

4. **Add a Function node:**  
   - Name: "Transform Order Data to Standard Format"  
   - Paste the provided JavaScript code that extracts and maps Shopify order data into a standardized flat structure.  
   - This node consumes webhook JSON input and outputs formatted JSON.

5. **Add a Google Sheets node:**  
   - Name: "Save Order to Google Sheets"  
   - Operation: Append row  
   - Document ID: Use your Google Sheets document ID (create and prepare the sheet beforehand).  
   - Sheet Name: Name or ID of the sheet tab (e.g., "orders").  
   - Mapping Mode: Auto map input data fields to columns.  
   - Credentials: Add and select OAuth2 Google Sheets credentials with access to your spreadsheet.

6. **Add an If node:**  
   - Name: "Success?"  
   - Condition: Configure to check if the Google Sheets append operation succeeded (e.g., check for presence of output data or status code).  
   - Connect Google Sheets node output as input.  
   - Setup two outputs: success and failure branches.

7. **Add a Telegram node:**  
   - Name: "Send Success Notification1"  
   - Text: Compose a message template including order number, customer name, total price, currency, and date. Use expressions referencing the "Transform Order Data to Standard Format" node.  
   - Chat ID: Your Telegram chat ID where notifications should be sent.  
   - Credentials: Add Telegram Bot API credentials.  
   - Connect from "Success?" node success output.

8. **Add another Telegram node:**  
   - Name: "Send Error Notification"  
   - Text: Compose an error message template including the error details and timestamp using expressions.  
   - Chat ID: Use Telegram chat ID variable or direct value.  
   - Credentials: Use the same Telegram Bot API credentials.  
   - Connect from "Success?" node failure output.

9. **Add a Set node:**  
   - Name: "Variables"  
   - Add string values for:  
     - `spreadsheetId` (your Google Sheet ID)  
     - `sheetName` (your sheet/tab name)  
     - `telegramChatId` (your Telegram chat ID)  
   - This node can be used to centralize variable values used in expressions in other nodes.

10. **Connect the nodes:**  
    - Webhook node → Function node → Google Sheets node → If node  
    - If node success output → Send Success Notification  
    - If node failure output → Send Error Notification  
    - Variables node is independent but referenced in expressions for IDs.

11. **Test the workflow:**  
    - Configure a Shopify webhook to point to your n8n instance URL + `/webhook/shopify-webhook`  
    - Place an order in Shopify or send a test webhook payload.  
    - Verify data appears in Google Sheets and Telegram notifications are received.

12. **Optional:**  
    - Add sticky notes for error handling and overview documentation.  
    - Adjust error condition in If node to properly detect Google Sheets API success/failure.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow requires setting up Shopify webhook pointing to n8n webhook URL with path `/shopify-webhook`.                                                                                                                               | Shopify Webhooks setup documentation                                                                   |
| Google Sheets spreadsheet must be pre-created with appropriate columns matching the mapped JSON fields.                                                                                                                             | Google Sheets API and spreadsheet preparation                                                         |
| Telegram notifications require a Telegram bot token and chat ID; instructions for creating bots and retrieving chat IDs are available on Telegram's official documentation.                                                         | Telegram Bot API docs                                                                                   |
| Sticky note "Workflow Overview" provides a concise summary and setup instructions for users.                                                                                                                                          | Inline with the workflow for quick reference                                                          |
| Sticky note "Error Handling Note" emphasizes importance of Telegram bot configuration for error alerts.                                                                                                                               | Inline with the workflow for quick reference                                                          |
| The workflow’s 'Success?' node is currently hardcoded to always true; users should adapt the condition to properly detect Google Sheets operation success or failure for accurate notification routing.                              | Customization tip for robust error handling                                                           |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected content. All manipulated data is legal and public.