Automated DHL Shipment Tracking Bot for Web Forms and Email Inquiries

https://n8nworkflows.xyz/workflows/automated-dhl-shipment-tracking-bot-for-web-forms-and-email-inquiries-9876


# Automated DHL Shipment Tracking Bot for Web Forms and Email Inquiries

---

### 1. Workflow Overview

This workflow is designed to automate responses to DHL shipment tracking inquiries originating from two distinct customer communication channels: web forms and emails. It consolidates inputs, fetches real-time shipment status from the DHL API, and returns a contextual, formatted response appropriate to the inquiry’s source.

**Target Use Cases:**  
- E-commerce or logistics businesses that receive frequent DHL shipment status questions.  
- Customer service automation for tracking inquiries submitted via website forms or directly by email.

**Logical Blocks:**  
- **1.1 Input Reception:** Collects tracking inquiries from webhooks (web forms) and Gmail email triggers.  
- **1.2 Data Normalization:** Unifies the input data format by extracting tracking numbers and customer details.  
- **1.3 DHL API Interaction:** Queries DHL’s shipment tracking API with the extracted tracking number.  
- **1.4 Response Formatting:** Constructs a personalized response message based on DHL’s API data.  
- **1.5 Response Routing:** Sends back the response either via webhook JSON response or Gmail email, depending on the origin channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens simultaneously for DHL tracking inquiries submitted via a web form (received as HTTP POST requests) and for new unread emails arriving in a monitored Gmail inbox. The data from both sources is merged into a single stream for downstream processing.

- **Nodes Involved:**  
  - Webhook Form Trigger  
  - Gmail Email Trigger  
  - Merge Triggers  
  - Trigger Setup (sticky note)

- **Node Details:**  

  - **Webhook Form Trigger**  
    - Type: Webhook  
    - Role: Receives HTTP POST requests from web forms submitting tracking inquiries.  
    - Configuration: Path set to `dhl-tracking-inquiry`, method POST, response mode set to respond via node output.  
    - Input: External HTTP POST requests.  
    - Output: Emits webhook payload to "Merge Triggers".  
    - Failure modes: Invalid or malformed requests, missing required fields.  
    - Notes: Requires URL integration in website form.  

  - **Gmail Email Trigger**  
    - Type: Gmail Trigger  
    - Role: Monitors Gmail inbox for unread emails to detect tracking inquiries.  
    - Configuration: Polls every minute, no sender filter, only unread emails.  
    - Credentials: OAuth2 Gmail credentials configured.  
    - Input: Gmail inbox events.  
    - Output: Emits new email data to "Merge Triggers".  
    - Failures: OAuth token expiration, Gmail API rate limits.  

  - **Merge Triggers**  
    - Type: Merge  
    - Role: Combines outputs of both triggers into a single stream for unified processing.  
    - Configuration: Mode “combine” merges input streams without filtering or sorting.  
    - Input: Webhook Form Trigger (index 0), Gmail Email Trigger (index 1).  
    - Output: Merged data to "Extract Tracking Number" node.  

  - **Trigger Setup** (Sticky Note)  
    - Provides instructions for configuring Gmail OAuth2 credentials and webhook URL integration.  

#### 2.2 Data Normalization

- **Overview:**  
  Standardizes the input data from both email and webhook sources by extracting the DHL tracking number, customer email, and customer name into a consistent JSON structure for later use.

- **Nodes Involved:**  
  - Extract Tracking Number (Code Node)  
  - Extraction Logic (sticky note)

- **Node Details:**  

  - **Extract Tracking Number**  
    - Type: Code (JavaScript)  
    - Role: Parses incoming data to identify and extract the tracking number and customer details from different formats.  
    - Configuration: Custom JavaScript to handle both webhook body and email text parsing. Uses a regex matching 10 or more digit numbers for DHL tracking numbers.  
    - Input: Merged data from webhook and email triggers.  
    - Output: JSON objects with fields: `trackingNumber`, `customerEmail`, `customerName`, `originalData`.  
    - Expressions: Uses `$input.all()` to process all incoming items.  
    - Failure modes: Missing or invalid tracking numbers, malformed email content, regex misses.  
    - Notes: Ensures downstream nodes receive uniform data format.  

  - **Extraction Logic** (Sticky Note)  
    - Explains the code node’s purpose in unifying data formats and fields.  

#### 2.3 DHL API Interaction

- **Overview:**  
  Uses the extracted tracking number to query DHL’s official shipment tracking API, retrieving the latest shipment status and details.

- **Nodes Involved:**  
  - Get DHL Tracking Status (HTTP Request)  
  - DHL API Configuration (sticky note)

- **Node Details:**  

  - **Get DHL Tracking Status**  
    - Type: HTTP Request  
    - Role: Sends a GET request to DHL’s tracking API endpoint with the tracking number as a query parameter.  
    - Configuration:  
      - URL: `https://api-eu.dhl.com/track/shipments`  
      - Query parameter: `trackingNumber` set dynamically from extracted data.  
      - Authentication: HTTP Header Auth with header `DHL-API-Key`.  
      - Header value: Placeholder `YOUR_DHL_API_KEY` (must be replaced with a real key).  
      - Response format: JSON expected.  
    - Credentials: HTTP Header Authentication node with API key.  
    - Input: Extracted tracking number JSON.  
    - Output: DHL API JSON response.  
    - Failures: Invalid or expired API key, API rate limits, network errors, malformed responses.  

  - **DHL API Configuration** (Sticky Note)  
    - Provides instructions to obtain and set the DHL API key via the DHL Developer Portal.  

#### 2.4 Response Formatting

- **Overview:**  
  Processes the DHL API response to create a personalized, human-readable message summarizing the shipment status, including fallback messages for errors or missing data.

- **Nodes Involved:**  
  - Format Response Message (Code Node)

- **Node Details:**  

  - **Format Response Message**  
    - Type: Code (JavaScript)  
    - Role: Reads DHL API data and customer info to build a detailed status message with shipment details, estimated delivery, and tracking link.  
    - Key expressions:  
      - Accesses `shipments[0]` from DHL data, extracts status, latest event, location, timestamps.  
      - Formats message with template literals including customer name and tracking details.  
      - Includes error handling to generate fallback messages if data is missing or API errors occur.  
      - Sets `isWebhook` flag based on original input source.  
    - Input: DHL API response and extracted customer data.  
    - Output: JSON with `customerEmail`, `customerName`, `subject`, `message`, `trackingDetails`, and `isWebhook`.  
    - Failure modes: API response format changes, missing fields, JavaScript exceptions.  

#### 2.5 Response Routing and Delivery

- **Overview:**  
  Routes the formatted response back to the customer via the channel from which the inquiry originated — either as a webhook JSON response or an email reply.

- **Nodes Involved:**  
  - Check Source (IF Node)  
  - Webhook Response  
  - Send Gmail Response  
  - Response Routing (sticky note)  
  - Email Setup (sticky note)

- **Node Details:**  

  - **Check Source**  
    - Type: IF  
    - Role: Determines the original inquiry source by checking `isWebhook` boolean flag.  
    - Configuration: Condition tests if `isWebhook` equals true.  
    - Input: Formatted response JSON.  
    - Output: True branch to webhook response, False branch to Gmail send.  
    - Failure modes: Incorrect flag setting, expression evaluation errors.  

  - **Webhook Response**  
    - Type: Respond to Webhook  
    - Role: Sends JSON shipment tracking details back as HTTP response to the original web form submitter.  
    - Configuration: HTTP 200, Content-Type application/json, body contains serialized `trackingDetails`.  
    - Input: True branch from IF node.  
    - Output: HTTP response to form submitter.  
    - Failures: Network interruptions, invalid response data.  

  - **Send Gmail Response**  
    - Type: Gmail Send  
    - Role: Sends a formatted email reply to the customer’s email address extracted earlier.  
    - Configuration:  
      - Recipient: Dynamic `customerEmail` field.  
      - Subject: Dynamic subject including tracking number.  
      - Message: Dynamic formatted message body.  
      - Reply-To: Set to `support@yourcompany.com` for customer replies.  
    - Credentials: Gmail OAuth2 credentials.  
    - Input: False branch from IF node.  
    - Output: Sends email via Gmail API.  
    - Failures: OAuth token expiration, email sending limits, invalid email addresses.  

  - **Response Routing** (Sticky Note)  
    - Explains the conditional routing logic depending on inquiry source.  

  - **Email Setup** (Sticky Note)  
    - Recommends customizing the email reply and setting the Reply-To address in the Gmail send node.  

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                                   | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                                    |
|------------------------|-----------------------|-------------------------------------------------|--------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow Overview      | Sticky Note           | Describes workflow purpose and channels handled | —                              | —                           | Multi-Channel DHL Status Bot overview including channels and process flow.                                    |
| Webhook Form Trigger   | Webhook               | Receives tracking inquiry from web form         | —                              | Merge Triggers               | Trigger Setup sticky note covers this and Gmail trigger for configuration instructions.                        |
| Gmail Email Trigger    | Gmail Trigger         | Monitors inbox for new tracking inquiry emails  | —                              | Merge Triggers               |                                                                                                               |
| Trigger Setup          | Sticky Note           | Instructions for Gmail and webhook trigger setup| —                              | —                           | Configuration guide for Gmail OAuth2 and webhook URL integration.                                             |
| Merge Triggers         | Merge                 | Combines webhook and email inputs                | Webhook Form Trigger, Gmail Trigger | Extract Tracking Number  |                                                                                                               |
| Extract Tracking Number| Code                  | Extracts and normalizes tracking data            | Merge Triggers                 | Get DHL Tracking Status      | Extraction Logic sticky note explains data normalization from both sources.                                   |
| Extraction Logic       | Sticky Note           | Explains data extraction and normalization       | —                              | —                           |                                                                                                               |
| Get DHL Tracking Status| HTTP Request          | Queries DHL API for shipment status              | Extract Tracking Number         | Format Response Message      | DHL API Configuration sticky note explains API key setup instructions.                                       |
| DHL API Configuration  | Sticky Note           | Instructions for DHL API key setup                | —                              | —                           |                                                                                                               |
| Format Response Message| Code                  | Builds personalized shipment status message      | Get DHL Tracking Status         | Check Source                |                                                                                                               |
| Check Source           | IF                    | Routes response based on inquiry source           | Format Response Message         | Webhook Response, Send Gmail Response| Response Routing sticky note explains the routing logic.                                        |
| Response Routing       | Sticky Note           | Explains conditional response routing             | —                              | —                           |                                                                                                               |
| Webhook Response       | Respond to Webhook    | Sends JSON response to web form submitter         | Check Source (true branch)      | —                           |                                                                                                               |
| Send Gmail Response    | Gmail Send            | Sends email reply to customer                      | Check Source (false branch)     | —                           | Email Setup sticky note with customization recommendations.                                                  |
| Email Setup            | Sticky Note           | Email reply customization instructions            | —                              | —                           |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Form Trigger Node**  
   - Type: Webhook  
   - Set HTTP Method: POST  
   - Path: `dhl-tracking-inquiry`  
   - Response Mode: Respond with node output  
   - Save the production webhook URL for website form integration.

2. **Create Gmail Email Trigger Node**  
   - Type: Gmail Trigger  
   - Poll Interval: Every minute  
   - Filter: Unread emails only (no sender filter)  
   - Connect Gmail OAuth2 credentials.

3. **Create Merge Node**  
   - Type: Merge  
   - Mode: Combine (merge inputs)  
   - Connect Webhook Form Trigger to input 0, Gmail Email Trigger to input 1.

4. **Create Code Node “Extract Tracking Number”**  
   - JavaScript code to:  
     - Detect source (webhook or email).  
     - Extract tracking number (from webhook JSON body fields or email text using regex for 10+ digit numbers).  
     - Extract customer email and name from respective input structures.  
     - Output uniform JSON with keys: `trackingNumber`, `customerEmail`, `customerName`, and `originalData`.  
   - Connect Merge output to this node.

5. **Create HTTP Request Node “Get DHL Tracking Status”**  
   - Method: GET  
   - URL: `https://api-eu.dhl.com/track/shipments`  
   - Query Parameter: `trackingNumber` set to `={{ $json.trackingNumber }}`  
   - Authentication: HTTP Header Auth  
   - Header: `DHL-API-Key` with your DHL API key value  
   - Response Format: JSON  
   - Connect Extract Tracking Number output to this node.

6. **Create Code Node “Format Response Message”**  
   - Use JavaScript code to:  
     - Parse DHL API response.  
     - Extract shipment status, latest event, location, timestamps, estimated delivery.  
     - Construct a personalized message using customer name and tracking info.  
     - Handle errors and missing data gracefully.  
     - Produce JSON with `customerEmail`, `customerName`, `subject`, `message`, `trackingDetails`, and `isWebhook` (true if original data includes webhook body).  
   - Connect DHL API Request output to this node.

7. **Create IF Node “Check Source”**  
   - Condition: `$json.isWebhook === true`  
   - Input: Format Response Message output.  
   - True branch for webhook responses, false for email responses.

8. **Create Respond to Webhook Node “Webhook Response”**  
   - Response Code: 200  
   - Response Headers: Content-Type: application/json  
   - Response Body: `={{ JSON.stringify($json.trackingDetails) }}`  
   - Connect True branch of IF node here.

9. **Create Gmail Node “Send Gmail Response”**  
   - Send To: `={{ $json.customerEmail }}`  
   - Subject: `={{ $json.subject }}`  
   - Message: `={{ $json.message }}`  
   - Reply-To: `support@yourcompany.com` (or your support email)  
   - Connect False branch of IF node here.  
   - Use same Gmail OAuth2 credentials as Gmail Trigger.

10. **Add Sticky Notes**  
    - Include overview and configuration instructions at appropriate places (optional but recommended for maintenance).

11. **Set Workflow Settings**  
    - Timezone: America/New_York (or adjust as needed)  
    - Execution Order: v1  
    - Enable saving of manual executions and data for success and error.

12. **Test Workflow**  
    - Submit test tracking inquiries via web form and email.  
    - Verify responses are routed correctly and data is accurate.  
    - Check for error handling and fallback messaging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                       |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| You must replace the placeholder `YOUR_DHL_API_KEY` in the DHL API Request node with your actual API key from DHL Developer Portal. | DHL Developer Portal: https://developer.dhl.com/    |
| Customize the email reply message and set a “Reply-To” address in the Gmail Send node for better customer support communication. | Email Setup sticky note in workflow                   |
| Webhook URL from the Webhook Form Trigger node must be integrated into the web form on your website (WordPress, Webflow, etc). | Trigger Setup sticky note                              |
| The workflow polls Gmail every minute; consider Gmail API quota limits to avoid throttling or delays.                         | Gmail trigger node configuration                       |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---