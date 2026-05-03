Customer Support Ticket System for SMEs with Google Sheets and Auto-Emails

https://n8nworkflows.xyz/workflows/customer-support-ticket-system-for-smes-with-google-sheets-and-auto-emails-8376


# Customer Support Ticket System for SMEs with Google Sheets and Auto-Emails

---

### 1. Workflow Overview

This workflow implements a **simple customer support ticketing system tailored for SMEs** (Small and Medium-sized Enterprises). It captures incoming support requests via a webhook, extracts and categorizes messages, saves the tickets into a Google Sheets spreadsheet for tracking, and sends automated acknowledgment emails to customers. The design focuses on lightweight, cost-effective, and transparent support management without complex CRM features.

Logical blocks:

- **1.1 Input Reception:** Capture incoming customer support requests through a webhook.
- **1.2 Message Extraction:** Extract the message content from the incoming data for processing.
- **1.3 Category Check:** Analyze the message content to identify keywords (e.g., “refund”) to categorize tickets.
- **1.4 Ticket Persistence:** Append the extracted and categorized ticket data into a Google Sheets document.
- **1.5 Customer Acknowledgment:** Send a personalized acknowledgment email to the customer confirming receipt of their ticket.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Accepts incoming HTTP requests (e.g., from forms or external systems) containing customer support data.

- **Nodes Involved:**  
  - Capture Ticket

- **Node Details:**

  - **Capture Ticket**  
    - Type: Webhook  
    - Role: Entry point for incoming customer support tickets.  
    - Configuration:  
      - Path: `customer-support` (URL endpoint to receive requests)  
      - Accepts HTTP POST requests with fields such as `name`, `email`, `message`.  
    - Input: External HTTP request  
    - Output: JSON containing submitted customer data  
    - Edge Cases:  
      - Missing or malformed input fields (e.g., no message or email)  
      - Unauthorized or unexpected source requests (no authentication configured)  
      - Webhook downtime or network issues

#### 1.2 Message Extraction

- **Overview:**  
  Extracts the `message` field from the webhook payload for focused processing.

- **Nodes Involved:**  
  - Extract Message

- **Node Details:**

  - **Extract Message**  
    - Type: Set  
    - Role: Isolates the `message` field and structures it for downstream nodes.  
    - Configuration:  
      - Sets a new variable `message` equal to the incoming JSON’s `message` property (`{{$json.message}}`).  
    - Input: Output from Capture Ticket  
    - Output: JSON with `message` field ready for categorization  
    - Edge Cases:  
      - Missing `message` field in input JSON could result in empty or undefined values  
      - Expression failures if input JSON structure changes unexpectedly

#### 1.3 Category Check

- **Overview:**  
  Inspects the message content for predefined keywords (currently “refund”) to categorize tickets and determine processing paths.

- **Nodes Involved:**  
  - Check Category

- **Node Details:**

  - **Check Category**  
    - Type: If  
    - Role: Conditional branching based on message content.  
    - Configuration:  
      - Condition: Checks if `message` contains the string `"refund"` (case-sensitive, strict type validation)  
      - Output 1 (True): Proceed to save ticket  
      - Output 2 (False): Proceed to send acknowledgment email  
    - Input: Output from Extract Message  
    - Output: Two branches (True and False)  
    - Edge Cases:  
      - Case sensitivity may miss keyword matches if variations exist (e.g., “Refund”, “REFUND”)  
      - Empty or null messages may cause false negatives  
      - Future extension needed for more categories or complex NLP

#### 1.4 Ticket Persistence

- **Overview:**  
  Stores the customer support ticket data persistently by appending it as a new row in a Google Sheets spreadsheet.

- **Nodes Involved:**  
  - Save Ticket

- **Node Details:**

  - **Save Ticket**  
    - Type: Google Sheets  
    - Role: Append ticket data to a Google Sheet for record-keeping and future reference.  
    - Configuration:  
      - Operation: Append  
      - Sheet Name: `Tickets` tab in the spreadsheet  
      - Document ID and Sheet ID: Both set to `"YOUR_SHEET_ID"` (to be replaced by actual spreadsheet IDs)  
      - Columns: Defined as `Name`, `Email`, `Message` mapped from incoming data  
    - Input: True branch of Check Category node (messages containing “refund”)  
    - Output: Confirmation of append operation  
    - Credentials: Requires Google Sheets OAuth2 credentials configured in n8n  
    - Edge Cases:  
      - Invalid or expired Google OAuth2 credentials causing auth failures  
      - Incorrect Sheet or document IDs leading to API errors  
      - Rate limits or network errors on Google Sheets API calls  
      - Data type mismatches if input data changes

#### 1.5 Customer Acknowledgment

- **Overview:**  
  Sends an email to the customer confirming receipt of the support ticket, improving communication and customer satisfaction.

- **Nodes Involved:**  
  - Send Acknowledgement

- **Node Details:**

  - **Send Acknowledgement**  
    - Type: Email Send  
    - Role: Sends an acknowledgment email with dynamic content referencing the customer’s name and message.  
    - Configuration:  
      - To: customer email (`{{$json.email}}`)  
      - From: fixed sender `support@yourcompany.com`  
      - Subject: `Support Ticket Received`  
      - Body: Personalized text including customer name and original message  
    - Input: False branch of Check Category node (messages not containing “refund”)  
    - Credentials: Requires SMTP credentials configured in n8n  
    - Edge Cases:  
      - Invalid or missing email addresses causing delivery failures  
      - SMTP connection or authentication errors  
      - Email content formatting problems  
      - Potential spam filtering by recipients  

---

### 3. Summary Table

| Node Name          | Node Type       | Functional Role                     | Input Node(s)     | Output Node(s)     | Sticky Note                                                                                                                                    |
|--------------------|-----------------|-----------------------------------|-------------------|--------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Capture Ticket     | Webhook         | Input Reception                   | -                 | Extract Message    |                                                                                                                                                 |
| Extract Message    | Set             | Message Extraction                | Capture Ticket    | Check Category     |                                                                                                                                                 |
| Check Category     | If              | Categorization of messages       | Extract Message   | Save Ticket, Send Acknowledgement |                                                                                                                                                 |
| Save Ticket        | Google Sheets   | Ticket Persistence                | Check Category    | -                  | range = "Tickets!A:C" fields = "Name,Email,Message"                                                                                            |
| Send Acknowledgement| Email Send      | Customer Acknowledgment           | Check Category    | -                  |                                                                                                                                                 |
| Sticky Note        | Sticky Note     | General annotation                | -                 | -                  | ## Flow                                                                                                                                         |
| Sticky Note1       | Sticky Note     | Project description and setup notes | -              | -                  | # 📝 Simplified Customer Support Automation for SMEs ... Full documentation embedded (see section 5 for details)                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: “Capture Ticket”**  
   - Type: Webhook  
   - Set Path to `customer-support`  
   - Accept POST requests with fields: `name`, `email`, `message`  
   - No authentication needed initially (add if required)  

2. **Create Set Node: “Extract Message”**  
   - Type: Set  
   - Add a field named `message`  
   - Value: Expression `{{$json["message"]}}`  
   - Connect input from “Capture Ticket” output  

3. **Create If Node: “Check Category”**  
   - Type: If  
   - Condition:  
     - Check if `message` contains the string `"refund"` (case-sensitive)  
   - Connect input from “Extract Message” output  
   - Configure two outputs:  
     - True (message contains “refund”)  
     - False (message does not contain “refund”)  

4. **Create Google Sheets Node: “Save Ticket”**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Replace `"YOUR_SHEET_ID"` with your actual Google Sheets document ID  
   - Sheet Name: `Tickets`  
   - Columns: Define columns for `Name`, `Email`, `Message`  
   - Map fields from input JSON accordingly  
   - Connect input from True output of “Check Category”  
   - Configure Google Sheets OAuth2 credentials with appropriate scopes (`spreadsheets`)  

5. **Create Email Send Node: “Send Acknowledgement”**  
   - Type: Email Send  
   - From Email: `support@yourcompany.com` (replace with your sending address)  
   - To Email: Expression `{{$json["email"]}}`  
   - Subject: `Support Ticket Received`  
   - Text Body:  
     ```
     Hello {{$json["name"]}},

     Your ticket has been received. Our team will get back to you shortly.

     Message: {{$json["message"]}}
     ```  
   - Connect input from False output of “Check Category”  
   - Configure SMTP credentials for your email provider  

6. **Connect Nodes**  
   - “Capture Ticket” → “Extract Message” → “Check Category”  
   - “Check Category” True → “Save Ticket”  
   - “Check Category” False → “Send Acknowledgement”  

7. **Test the Workflow**  
   - Deploy and activate the workflow  
   - Send test HTTP POST requests to the webhook URL with valid JSON payloads  
   - Verify tickets appear in Google Sheets and acknowledgment emails are sent appropriately  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                                                                                                                                                                                                                                                                                                                                                |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| # 📝 Simplified Customer Support Automation for SMEs: outlines problem, solution, target audience, scope, and setup steps. Provides a comprehensive explanation embedded as a sticky note in the workflow for easy reference during editing. This detailed project description helps understand the rationale behind node choices and workflow design.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Full content embedded in Sticky Note1 node inside the workflow JSON. A valuable internal documentation resource for maintaining or extending the workflow.                                                                                                                                                                                                                                                                                                    |
| Google Sheets node requires OAuth2 credentials with access to the target spreadsheet. Ensure that the Google account used has edit permissions on the sheet and that OAuth tokens are refreshed periodically to avoid failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Credential setup in n8n: Google Sheets OAuth2 API                                                                                                                                                                                                                                                                                                                                                                                                            |
| Email Send node requires properly configured SMTP credentials. Verify the sender email is authorized to send and not blocked by spam filters. Test sending limits and authentication settings with your email provider.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Credential setup in n8n: SMTP server credentials (e.g., Gmail, Outlook, SendGrid)                                                                                                                                                                                                                                                                                                                                                                               |
| Webhook paths should be guarded or validated in production to avoid spam or abuse. Consider adding authentication or IP whitelisting as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Security best practices for public webhooks                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Current message categorization is basic (keyword contains “refund”). For advanced classification, consider integrating AI/NLP nodes or external services.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Possible future enhancements                                                                                                                                                                                                                                                                                                                                                                                                                                    |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---