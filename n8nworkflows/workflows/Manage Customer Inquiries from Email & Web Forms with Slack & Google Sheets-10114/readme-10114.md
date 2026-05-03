Manage Customer Inquiries from Email & Web Forms with Slack & Google Sheets

https://n8nworkflows.xyz/workflows/manage-customer-inquiries-from-email---web-forms-with-slack---google-sheets-10114


# Manage Customer Inquiries from Email & Web Forms with Slack & Google Sheets

### 1. Workflow Overview

This workflow, titled **"Multi-Channel Customer Support Inquiry Management and Tracking System"**, is designed to automate the handling of customer inquiries arriving via two main channels: email and web form submissions. It targets customer support teams, marketing and sales teams, SMBs, and individuals who require efficient, centralized management of incoming customer questions and requests.

The workflow logically divides into the following blocks:

- **1.1 Multi-Channel Input**  
  Listens for incoming customer inquiries from two sources: an IMAP email inbox and a webhook endpoint for web form submissions.

- **1.2 Data Extraction & Parsing**  
  Extracts and normalizes key inquiry data (customer name, email, subject, message, source, timestamp) and intelligently classifies the inquiry type into categories such as "urgent," "billing," or "general."

- **1.3 Merge Inquiries**  
  Combines parsed data streams from email and webform inputs into a unified flow for subsequent processing.

- **1.4 Route by Inquiry Type**  
  Routes inquiries based on their classified type to different notification and processing paths.

- **1.5 Notifications & Logging**  
  Sends notifications to dedicated Slack channels based on inquiry type, logs all inquiries into a Google Sheets document, and sends an auto-reply email to the customer.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Multi-Channel Input

**Overview:**  
This block captures customer inquiries from two sources: email (via IMAP) and web form submissions (via webhook). It acts as the entry point of the workflow.

**Nodes Involved:**  
- Email Trigger  
- Webhook - Web Form

**Node Details:**  

- **Email Trigger**  
  - Type: Email Read (IMAP) Trigger  
  - Role: Watches an IMAP email inbox for new messages.  
  - Configuration: Default IMAP options; no specific folders or filters configured here (assumes default inbox).  
  - Inputs: None (trigger node)  
  - Outputs: Raw email data including metadata, HTML content, sender info, subject, and timestamp.  
  - Edge Cases: Possible IMAP connection failures, authentication errors, or no new emails.  
  - Notes: Requires proper IMAP credentials setup.

- **Webhook - Web Form**  
  - Type: Webhook (HTTP POST) Trigger  
  - Role: Listens for incoming HTTP POST requests, expected from a web form submitting JSON data.  
  - Configuration: Path set to "customer-inquiry", HTTP method POST, response mode set to last node.  
  - Inputs: None (trigger node)  
  - Outputs: JSON body containing form data fields: name, email, subject, message, type.  
  - Edge Cases: Invalid JSON payloads, missing expected fields, or unauthorized requests.  
  - Notes: Requires external web form integration to post data to this webhook URL.

---

#### 2.2 Data Extraction & Parsing

**Overview:**  
Extracts relevant data fields from raw inputs and standardizes the inquiry data structure, while classifying the inquiry type based on keywords.

**Nodes Involved:**  
- Extract Email Content  
- Parse Email Data  
- Parse Webhook Data

**Node Details:**  

- **Extract Email Content**  
  - Type: HTML Extract  
  - Role: Parses the HTML content of incoming emails to extract the message body text.  
  - Configuration: Extracts content from the `<body>` tag, trims and cleans whitespace.  
  - Inputs: Raw email data from Email Trigger node, specifically the "html" property.  
  - Outputs: JSON with extracted text under the key "text".  
  - Edge Cases: Emails with malformed or missing HTML bodies, extraction failure.  
  - Version: 1.2.

- **Parse Email Data**  
  - Type: Set Node  
  - Role: Maps raw email data fields to normalized inquiry fields: customerName, customerEmail, subject, message, source, receivedAt, and inquiryType.  
  - Configuration:  
    - customerName: Extracted from email sender's name.  
    - customerEmail: Extracted from sender's email address.  
    - subject: Email subject line.  
    - message: Extracted plain text from email body.  
    - source: Hardcoded string "email".  
    - receivedAt: Email received timestamp.  
    - inquiryType: Determined by checking if subject contains keywords for "urgent" (English or Japanese), "billing" (English or Japanese), else defaults to "general".  
  - Inputs: Output from Extract Email Content node.  
  - Outputs: Structured JSON with standardized inquiry data.  
  - Edge Cases: Missing or malformed sender fields, subject line absent, case sensitivity in keyword search.  
  - Version: 3.4.

- **Parse Webhook Data**  
  - Type: Set Node  
  - Role: Maps incoming webhook JSON fields to the same normalized inquiry structure as emails.  
  - Configuration:  
    - customerName, customerEmail, subject, message: Directly mapped from webhook body.  
    - source: Hardcoded string "webform".  
    - receivedAt: Current timestamp at processing time.  
    - inquiryType: Uses webhook-provided "type" field or defaults to "general" if missing.  
  - Inputs: Output from Webhook - Web Form node.  
  - Outputs: Structured JSON with standardized inquiry data.  
  - Edge Cases: Missing fields in webhook JSON, invalid data formats.  
  - Version: 3.4.

---

#### 2.3 Merge Inquiries

**Overview:**  
Combines the two inbound inquiry streams (email-derived and webform-derived) into a single unified flow for downstream processing.

**Nodes Involved:**  
- Merge Inquiries

**Node Details:**  

- **Merge Inquiries**  
  - Type: Merge Node  
  - Role: Combines two input streams by position into one output stream.  
  - Configuration: Mode set to "combine", combining inputs by their position (i.e., pairs of items from each input).  
  - Inputs: Main output from Parse Email Data (index 0) and Parse Webhook Data (index 1).  
  - Outputs: Unified stream of inquiries.  
  - Edge Cases: Unequal number of items on either input can result in missing pairs; empty inputs.  
  - Version: 3.2.

---

#### 2.4 Route by Inquiry Type

**Overview:**  
Routes the combined inquiries based on their classified inquiryType to specific downstream processing nodes.

**Nodes Involved:**  
- Route by Inquiry Type

**Node Details:**  

- **Route by Inquiry Type**  
  - Type: Switch Node  
  - Role: Directs each inquiry to one of three outputs based on inquiryType field.  
  - Configuration:  
    - Output "General Inquiries" if inquiryType equals "general".  
    - Output "Urgent Inquiries" if inquiryType equals "urgent".  
    - Output "Billing Inquiries" if inquiryType equals "billing".  
  - Inputs: Output from Merge Inquiries node.  
  - Outputs: Three separate outputs, each connected to different notification and logging nodes.  
  - Edge Cases: inquiryType with unexpected or missing values will not match any case and may be dropped or require default handling (not defined here).  
  - Version: 3.3.

---

#### 2.5 Notifications & Logging

**Overview:**  
Sends notifications to Slack channels appropriate to inquiry urgency, logs all inquiries to Google Sheets for record-keeping, and sends an automatic confirmation email to the customer.

**Nodes Involved:**  
- Save to Google Sheets  
- Notify Urgent - Slack  
- Notify General - Slack  
- Send Auto-Reply Email

**Node Details:**  

- **Save to Google Sheets**  
  - Type: Google Sheets node  
  - Role: Appends or updates inquiry records in a Google Sheets document to maintain a centralized log.  
  - Configuration:  
    - Operation: Append or update by matching on customerEmail column.  
    - Columns mapped: customerName, customerEmail, subject, message, source, receivedAt, inquiryType.  
    - Requires Google Sheets Document ID and Sheet Name to be set.  
  - Inputs: Output from Route by Inquiry Type (all inquiries regardless of type).  
  - Outputs: None connected downstream.  
  - Credentials: Google Sheets OAuth2 credentials required.  
  - Edge Cases: API rate limits, authorization failures, spreadsheet not found, column mismatch.  
  - Version: 4.7.

- **Notify Urgent - Slack**  
  - Type: Slack node  
  - Role: Posts formatted alert messages about urgent inquiries to a dedicated Slack channel.  
  - Configuration:  
    - Message text includes customer name, email, subject, message, receivedAt, and source.  
    - Posting to a configured Slack channel ID for urgent alerts.  
    - Uses OAuth2 authentication.  
  - Inputs: Output from Route by Inquiry Type for "Urgent Inquiries".  
  - Outputs: None connected downstream.  
  - Credentials: Slack OAuth2 credentials required.  
  - Edge Cases: Slack API rate limits, authentication errors, invalid channel IDs.  
  - Version: 2.3.

- **Notify General - Slack**  
  - Type: Slack node  
  - Role: Posts formatted notifications for general inquiries to a separate Slack channel.  
  - Configuration: Similar message template as urgent but for general inquiries.  
    - Posts to a configured Slack channel ID for general inquiries.  
    - Uses OAuth2 authentication.  
  - Inputs: Output from Route by Inquiry Type for "General Inquiries".  
  - Outputs: None connected downstream.  
  - Credentials: Slack OAuth2 credentials required.  
  - Edge Cases: Same as Notify Urgent.  
  - Version: 2.3.

- **Send Auto-Reply Email**  
  - Type: Gmail node  
  - Role: Sends an automated reply email to acknowledge receipt of the inquiry.  
  - Configuration:  
    - Sends email to customerEmail.  
    - Subject prefixed with "Re: " plus original subject.  
    - Message body is a polite, templated Japanese-language acknowledgment.  
  - Inputs: Output from Merge Inquiries (all inquiries).  
  - Outputs: None connected downstream.  
  - Credentials: Gmail OAuth2 credentials required.  
  - Edge Cases: Email sending failures, invalid email addresses.  
  - Version: 2.1.

---

### 3. Summary Table

| Node Name             | Node Type                | Functional Role                                | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                                              |
|-----------------------|--------------------------|-----------------------------------------------|------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------|
| Email Trigger         | Email Read IMAP Trigger   | Captures incoming customer emails             | None                         | Extract Email Content              | ## Multi-Channel Input \n**Listens for inquiries from incoming emails (IMAP) and web form submissions (Webhook).** |
| Webhook - Web Form    | Webhook Trigger          | Receives web form inquiry submissions         | None                         | Parse Webhook Data                | ## Multi-Channel Input \n**Listens for inquiries from incoming emails (IMAP) and web form submissions (Webhook).** |
| Extract Email Content | HTML Extract             | Extracts message body text from email HTML    | Email Trigger                | Parse Email Data                  | ## Data Extraction & Parsing \n**Extracts details like customer name, email, subject, message, source, received timestamp, and intelligently determines inquiryType (urgent, billing, general) from both email content and web form data.** |
| Parse Email Data      | Set Node                 | Normalizes and classifies email inquiry fields| Extract Email Content        | Merge Inquiries                  | ## Data Extraction & Parsing \n**Extracts details like customer name, email, subject, message, source, received timestamp, and intelligently determines inquiryType (urgent, billing, general) from both email content and web form data.** |
| Parse Webhook Data    | Set Node                 | Normalizes and classifies web form inquiry fields | Webhook - Web Form           | Merge Inquiries                  | ## Data Extraction & Parsing \n**Extracts details like customer name, email, subject, message, source, received timestamp, and intelligently determines inquiryType (urgent, billing, general) from both email content and web form data.** |
| Merge Inquiries       | Merge Node               | Combines email and web form inquiries into one stream | Parse Email Data, Parse Webhook Data | Route by Inquiry Type, Send Auto-Reply Email |                                                                                                        |
| Route by Inquiry Type | Switch Node              | Routes inquiries by inquiryType (urgent, billing, general) | Merge Inquiries              | Save to Google Sheets, Notify Urgent - Slack, Notify General - Slack |                                                                                                        |
| Save to Google Sheets | Google Sheets            | Logs inquiries into Google Sheets              | Route by Inquiry Type        | None                            |                                                                                                        |
| Notify Urgent - Slack | Slack                    | Posts urgent inquiry notifications to Slack   | Route by Inquiry Type        | None                            |                                                                                                        |
| Notify General - Slack| Slack                    | Posts general inquiry notifications to Slack  | Route by Inquiry Type        | None                            |                                                                                                        |
| Send Auto-Reply Email | Gmail                    | Sends auto-reply emails to customers           | Merge Inquiries              | None                            | ## Send Auto-Reply\n                                                                                     |
| Sticky Note           | Sticky Note              | Overview and instructions                      | None                         | None                            | ## Multi-Channel Customer Support Inquiry Management and Tracking System \n\n**Who's it for?**\nCustomer support teams, marketing & sales teams, SMBs, individuals needing efficient inquiry management. [...] |
| Sticky Note1          | Sticky Note              | Describes Multi-Channel Input block            | None                         | None                            | ## Multi-Channel Input \n**Listens for inquiries from incoming emails (IMAP) and web form submissions (Webhook).** |
| Sticky Note2          | Sticky Note              | Describes Send Auto-Reply block                 | None                         | None                            | ## Send Auto-Reply\n                                                                                     |
| Sticky Note3          | Sticky Note              | Describes Data Extraction & Parsing block     | None                         | None                            | ## Data Extraction & Parsing \n**Extracts details like customer name, email, subject, message, source, received timestamp, and intelligently determines inquiryType (urgent, billing, general) from both email content and web form data.** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Email Trigger" node**  
   - Type: Email Read IMAP Trigger  
   - Configure IMAP credentials with your email server details.  
   - Leave default options for mailbox polling.  
   - Position this node as the starting trigger for email input.

2. **Create "Webhook - Web Form" node**  
   - Type: Webhook Trigger  
   - Set HTTP Method to POST.  
   - Set Path to "customer-inquiry".  
   - Leave response mode as "last node".  
   - Position as the starting trigger for web form input.

3. **Create "Extract Email Content" node**  
   - Type: HTML Extract  
   - Set Data Property Name to `{{$json.html}}` (extract HTML body from email).  
   - Extraction rule: CSS Selector `body`, key as "text".  
   - Enable options to trim and clean extracted text.  
   - Connect input from "Email Trigger" node.

4. **Create "Parse Email Data" node**  
   - Type: Set Node  
   - Map the following fields using expressions:  
     - customerName: `{{$json.from.name}}`  
     - customerEmail: `{{$json.from.address}}`  
     - subject: `{{$json.subject}}`  
     - message: `{{$json.text}}` (from Extract Email Content)  
     - source: "email" (static string)  
     - receivedAt: `{{$json.date}}`  
     - inquiryType: Use conditional expression to check if subject includes keywords "urgent" or "緊急" → "urgent", else if includes "billing" or "請求" → "billing", else "general".  
   - Connect input from "Extract Email Content".

5. **Create "Parse Webhook Data" node**  
   - Type: Set Node  
   - Map fields from webhook JSON body:  
     - customerName: `{{$json.body.name}}`  
     - customerEmail: `{{$json.body.email}}`  
     - subject: `{{$json.body.subject}}`  
     - message: `{{$json.body.message}}`  
     - source: "webform" (static string)  
     - receivedAt: current time `{{$now}}`  
     - inquiryType: `{{$json.body.type || 'general'}}` (defaulting to general)  
   - Connect input from "Webhook - Web Form".

6. **Create "Merge Inquiries" node**  
   - Type: Merge Node  
   - Mode: Combine  
   - Combine By: Position  
   - Connect inputs from "Parse Email Data" (input 1) and "Parse Webhook Data" (input 2).

7. **Create "Route by Inquiry Type" node**  
   - Type: Switch Node  
   - Rule 1: If `inquiryType` equals "general" → output "General Inquiries"  
   - Rule 2: If `inquiryType` equals "urgent" → output "Urgent Inquiries"  
   - Rule 3: If `inquiryType` equals "billing" → output "Billing Inquiries"  
   - Connect input from "Merge Inquiries" output 0.

8. **Create "Save to Google Sheets" node**  
   - Type: Google Sheets node  
   - Operation: Append or update  
   - Map columns: customerName, customerEmail, subject, message, source, receivedAt, inquiryType  
   - Matching column: customerEmail  
   - Set Spreadsheet ID and Sheet Name accordingly (create spreadsheet with these columns beforehand).  
   - Connect input from "Route by Inquiry Type" output 0 (all inquiries).  
   - Authenticate with Google Sheets OAuth2 credentials.

9. **Create "Notify Urgent - Slack" node**  
   - Type: Slack node  
   - Text: Formatted message with customerName, customerEmail, subject, message, receivedAt, and source.  
   - Channel: Slack channel ID for urgent inquiries.  
   - Authentication: OAuth2 Slack credentials.  
   - Connect input from "Route by Inquiry Type" output 1 (urgent inquiries).

10. **Create "Notify General - Slack" node**  
    - Type: Slack node  
    - Text: Similar format for general inquiries.  
    - Channel: Slack channel ID for general inquiries.  
    - Authentication: OAuth2 Slack credentials.  
    - Connect input from "Route by Inquiry Type" output 2 (general inquiries).

11. **Create "Send Auto-Reply Email" node**  
    - Type: Gmail node  
    - Send To: `{{$json.customerEmail}}`  
    - Subject: "Re: {{$json.subject}}"  
    - Message: Template Japanese text acknowledging receipt of inquiry.  
    - Authenticate with Gmail OAuth2 credentials.  
    - Connect input from "Merge Inquiries" node (all inquiries).

12. **Add Sticky Notes** for documentation and instructions as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow is designed for customer support teams to automate inquiry management from emails and web forms, routing notifications via Slack, logging inquiries in Google Sheets, and sending auto-replies via Gmail.                                                                                                                                                     | Workflow description.                                                                                         |
| Setup requires proper OAuth2 credentials for Google Sheets, Slack, and Gmail, plus IMAP email access and a webhook-enabled web form.                                                                                                                                                                                                                                  | Credential setup instructions.                                                                                |
| Slack Channel IDs and Google Sheets Document ID and Sheet Name need to be configured specifically for your workspace and documents.                                                                                                                                                                                                                                   | Configuration note.                                                                                           |
| Inquiry types can be expanded or modified in the "Route by Inquiry Type" node to support more categories or different routing logic.                                                                                                                                                                                                                                  | Customization guidance.                                                                                       |
| Consider adding CRM integration nodes or AI sentiment analysis for advanced prioritization.                                                                                                                                                                                                                                                                           | Suggested future enhancements.                                                                                |
| The Japanese text in the auto-reply and Slack notifications is part of the original workflow and can be localized as needed.                                                                                                                                                                                                                                          | Localization note.                                                                                            |
| Slack OAuth2 authentication requires scopes for posting messages to channels. Ensure these are granted in your Slack app configuration.                                                                                                                                                                                                                                | Slack OAuth2 requirements.                                                                                    |
| Google Sheets API quotas may limit throughput; ensure appropriate handling of API limits and errors.                                                                                                                                                                                                                                                                   | Google Sheets API usage note.                                                                                 |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow and adheres strictly to content policies. It contains no illegal, offensive, or protected material. All processed data is legal and public.