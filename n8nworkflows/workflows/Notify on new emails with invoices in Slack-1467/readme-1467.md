Notify on new emails with invoices in Slack

https://n8nworkflows.xyz/workflows/notify-on-new-emails-with-invoices-in-slack-1467


# Notify on new emails with invoices in Slack

### 1. Workflow Overview

This workflow automates the processing of new emails in a mailbox to detect invoices and notify relevant stakeholders via Slack and email. It is designed for finance or accounting teams to quickly identify incoming invoices, extract invoice amounts, and escalate high-value payments for approval.

The workflow consists of the following logical blocks:

- **1.1 Email Retrieval:** Connects to a mailbox and reads new emails.
- **1.2 Invoice Detection:** Filters emails whose body contains the keyword “invoice.”
- **1.3 Invoice Data Extraction:** Sends invoice attachments to Mindee API for parsing the total amount.
- **1.4 Conditional Notification and Escalation:** 
  - If the invoice amount exceeds 1000, sends an email alert to the finance manager.
  - Sends a Slack notification for every invoice detected regardless of amount.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Retrieval

- **Overview:**  
  This block connects to an IMAP email server and checks for new emails in the Inbox folder, retrieving them with resolved formatting.

- **Nodes Involved:**  
  - Check for new emails

- **Node Details:**  
  **Check for new emails**  
  - Type: Email Read (IMAP)  
  - Role: Connects to the mailbox via IMAP to fetch new emails  
  - Configuration:  
    - Mailbox: Inbox  
    - Format: Resolved (HTML/text content processed)  
    - Option: Allow unauthorized SSL certificates (useful for self-signed certs)  
  - Credentials: IMAP credentials configured for Gmail account  
  - Input: None (trigger node)  
  - Output: Email messages with metadata and body content  
  - Edge Cases:  
    - Network issues or wrong credentials can cause failures  
    - Emails without expected body format could affect downstream processing  
  - Version Requirements: Standard for n8n IMAP node v1

---

#### 2.2 Invoice Detection

- **Overview:**  
  Filters emails to only pass those whose body text contains the keyword “invoice” (case-insensitive).

- **Nodes Involved:**  
  - If email body contains invoice

- **Node Details:**  
  **If email body contains invoice**  
  - Type: If node (conditional logic)  
  - Role: Checks if the email's text content contains “invoice”  
  - Configuration:  
    - Condition: string contains  
    - Expression: `{{$json["text"].toLowerCase()}}` contains `"invoice"`  
    - Combine operation: any (single condition)  
  - Input: Emails from "Check for new emails" node  
  - Output: Passes emails meeting condition to next block, others discarded  
  - Edge Cases:  
    - Emails without a “text” property or with empty body could cause expression errors  
    - Case sensitivity handled by `.toLowerCase()`  
  - Version Requirements: Standard

---

#### 2.3 Invoice Data Extraction

- **Overview:**  
  Sends the first attachment of the email to the Mindee API to extract invoice details, specifically the total amount including taxes.

- **Nodes Involved:**  
  - Extract the total amount

- **Node Details:**  
  **Extract the total amount**  
  - Type: Mindee node (invoice parsing)  
  - Role: Uses Mindee Invoice API to parse the invoice attachment  
  - Configuration:  
    - Resource: Invoice  
    - Raw data: true (returns detailed predictions)  
    - Binary property name: `attachment_0` (first attachment from email)  
  - Credentials: Mindee API credentials  
  - Input: Emails filtered by “If email body contains invoice” node  
  - Output: JSON with invoice predictions including total amount  
  - Edge Cases:  
    - Missing or corrupted attachments  
    - API authentication or quota errors  
    - Non-invoice attachments leading to parsing failures  
  - Version Requirements: Mindee node v1

---

#### 2.4 Conditional Notification and Escalation

- **Overview:**  
  Based on the extracted invoice amount, the workflow notifies the team via Slack and optionally emails the finance manager if the amount exceeds 1000.

- **Nodes Involved:**  
  - If Amount > 1000  
  - Send email to finance manager  
  - Send new invoice notification

- **Node Details:**  
  **If Amount > 1000**  
  - Type: If node  
  - Role: Checks if the invoice total (including tax) is greater than 1000  
  - Configuration:  
    - Condition: number larger than 1000  
    - Expression: `{{$json["predictions"][0]["total_incl"]["amount"]}}`  
  - Input: Invoice data from Mindee node  
  - Output:  
    - True: sends email to finance manager and then sends Slack notification  
    - False: sends only Slack notification  
  - Edge Cases:  
    - Missing total amount in API response  
    - Data type mismatches (e.g., string vs number)  
  - Version Requirements: Standard

  **Send email to finance manager**  
  - Type: Email Send node (SMTP)  
  - Role: Sends alert email about high-value invoice  
  - Configuration:  
    - To: finance-manager@company.tld  
    - From: n8n@noreply.tld  
    - Subject: New high value invoice  
    - Text: Static message alerting about invoice approval need  
    - Attachments: forwards the invoice attachment (`attachment_0`)  
  - Credentials: SMTP credentials (configured for Mailtrap)  
  - Input: True branch from “If Amount > 1000”  
  - Output: Proceeds to Slack notification  
  - Edge Cases:  
    - SMTP authentication or sending errors  
    - Attachment size or format issues  
  - Version Requirements: Standard

  **Send new invoice notification**  
  - Type: Slack node  
  - Role: Posts a message to Slack channel alerting about new invoice  
  - Configuration:  
    - Channel: team-accounts  
    - Text: ":new: There is a new invoice to pay :new:"  
    - Attachments:  
      - Color: #FFBF00 (yellow)  
      - Fields:  
        - Amount (from Mindee invoice total)  
        - From (email sender address)  
        - Subject (email subject)  
      - Footer: Date of the email  
  - Credentials: Slack API key  
  - Input: True and False branches (always runs last)  
  - Output: End of workflow  
  - Edge Cases:  
    - Slack API rate limits or credential errors  
    - Missing or malformed data in fields  
  - Version Requirements: Standard

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                       | Input Node(s)              | Output Node(s)                    | Sticky Note                                 |
|----------------------------|------------------------|------------------------------------|----------------------------|---------------------------------|---------------------------------------------|
| Check for new emails        | Email Read IMAP        | Retrieve new emails from mailbox    | -                          | If email body contains invoice   | Configure IMAP node with correct mailbox and credentials |
| If email body contains invoice | If                   | Filter emails containing "invoice" | Check for new emails        | Extract the total amount         |                                             |
| Extract the total amount    | Mindee Invoice Parser  | Extract invoice total from attachment| If email body contains invoice | If Amount > 1000                 | Configure Mindee node with your credentials |
| If Amount > 1000            | If                     | Check if invoice amount > 1000      | Extract the total amount    | Send email to finance manager, Send new invoice notification |                                             |
| Send email to finance manager | Email Send SMTP       | Email alert for high-value invoices | If Amount > 1000 (true)     | Send new invoice notification    | Configure SMTP node with mail server and recipients |
| Send new invoice notification | Slack                 | Notify team in Slack about invoice  | If Amount > 1000, Send email to finance manager | -                               | Configure Slack node with credentials and channel |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an IMAP Email Read Node**  
   - Name: `Check for new emails`  
   - Type: Email Read (IMAP)  
   - Parameters:  
     - Mailbox: Inbox  
     - Format: Resolved  
     - Options: Allow unauthorized certificates enabled  
   - Credentials: Configure with your IMAP account (e.g., Gmail OAuth2)  
   - Connect as the first node (trigger)

2. **Add If Node for Invoice Detection**  
   - Name: `If email body contains invoice`  
   - Type: If  
   - Parameters:  
     - Condition: String contains  
     - Expression: `{{$json["text"].toLowerCase()}}` contains `"invoice"`  
     - Combine operation: any  
   - Connect output of `Check for new emails` to this node’s input

3. **Add Mindee Node to Parse Invoices**  
   - Name: `Extract the total amount`  
   - Type: Mindee (Invoice)  
   - Parameters:  
     - Resource: Invoice  
     - Raw Data: true  
     - Binary Property Name: `attachment_0`  
   - Credentials: Provide Mindee API credentials (API key)  
   - Connect output of `If email body contains invoice` to this node’s input

4. **Add If Node for Amount Check**  
   - Name: `If Amount > 1000`  
   - Type: If  
   - Parameters:  
     - Condition: Number larger than 1000  
     - Expression: `{{$json["predictions"][0]["total_incl"]["amount"]}}`  
   - Connect output of `Extract the total amount` to this node’s input

5. **Add Email Send Node for Finance Manager Notification**  
   - Name: `Send email to finance manager`  
   - Type: Email Send (SMTP)  
   - Parameters:  
     - To Email: finance-manager@company.tld  
     - From Email: n8n@noreply.tld  
     - Subject: New high value invoice  
     - Text:  
       ```
       Hi,

       There is a new high value invoice to be paid that you may need to approve.

       ~ n8n workflow
       ```  
     - Attachments: `attachment_0` (pass attachment from email)  
   - Credentials: Configure SMTP server credentials (e.g., Mailtrap or your mail server)  
   - Connect True output of `If Amount > 1000` to this node’s input

6. **Add Slack Node for Invoice Notification**  
   - Name: `Send new invoice notification`  
   - Type: Slack  
   - Parameters:  
     - Channel: team-accounts (replace with your Slack channel)  
     - Text: `:new: There is a new invoice to pay :new:`  
     - Attachments:  
       - Color: #FFBF00  
       - Fields:  
         - Amount: `={{$node["If Amount > 1000"].json["predictions"][0]["total_incl"]["amount"]}}`  
         - From: `={{$node["Check for new emails"].json["from"]["value"][0]["address"]}}`  
         - Subject: `={{$node["Check for new emails"].json["subject"]}}`  
       - Footer: `=*Date:* {{$node["Check for new emails"].json["date"]}}`  
   - Credentials: Configure Slack API credentials (OAuth token)  
   - Connect:  
     - From `Send email to finance manager` (after email sent)  
     - From False output of `If Amount > 1000` (directly)  
     (Both branches converge here)

7. **Finalize Workflow**  
   - Verify all credentials are set correctly  
   - Test the flow with sample emails containing invoices and attachments  
   - Adjust mailbox and Slack channel as needed

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                               |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| To use this workflow, configure IMAP node with correct mailbox and credentials                        | Workflow description                                          |
| Mindee node requires API credentials from your Mindee Invoice account                                | https://www.mindee.com/developers/                             |
| SMTP node must be configured to use your mail server and correct recipients                          | May use services like Mailtrap for testing                     |
| Slack node requires Slack API token and channel name                                                | Slack API docs: https://api.slack.com/                         |
| The workflow filters emails by checking the presence of “invoice” in the email body (case insensitive) | Important for filtering relevant emails                        |

---

This documentation fully describes the workflow structure, logic, and configuration steps to enable advanced users or automation agents to understand, reproduce, and maintain it.