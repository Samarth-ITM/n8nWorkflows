Gumroad Clientbell: Auto-Log Sales, Ping Telegram, & Thank via Email

https://n8nworkflows.xyz/workflows/gumroad-clientbell--auto-log-sales--ping-telegram----thank-via-email-3469


# Gumroad Clientbell: Auto-Log Sales, Ping Telegram, & Thank via Email

### 1. Workflow Overview

This workflow automates the process of tracking and acknowledging sales made on Gumroad, targeting creators, solopreneurs, digital product sellers, and freelancers. It eliminates manual tasks by automatically logging sales data, notifying the user via Telegram, and optionally sending personalized thank-you emails to buyers.

The workflow is logically divided into the following blocks:

- **1.1 Gumroad Sales Trigger:** Listens for new sales events from Gumroad.
- **1.2 Data Cleaning and Extraction:** Processes raw sale data into a structured format.
- **1.3 Sales Logging:** Records the cleaned sale data into a Google Sheet for tracking.
- **1.4 Telegram Notification:** Sends a real-time Telegram message to notify about the new sale.
- **1.5 Conditional Email Sending:** Checks if an email should be sent, prepares the email content, and sends a thank-you email via Gmail (optional).
- **1.6 Workflow Notes and Disabled Nodes:** Includes setup guidance, optional upgrades, and placeholders for alternative integrations.

---

### 2. Block-by-Block Analysis

#### 2.1 Gumroad Sales Trigger

- **Overview:**  
  This block initiates the workflow by triggering on each new sale event from Gumroad.

- **Nodes Involved:**  
  - Gumroad Sales Trigger

- **Node Details:**  
  - **Type:** Gumroad Trigger Node  
  - **Role:** Listens for new sales via Gumroad webhook.  
  - **Configuration:** Connected to Gumroad API with webhook ID set; no additional parameters configured.  
  - **Input/Output:** No input; outputs raw sale JSON data.  
  - **Version:** 1  
  - **Potential Failures:**  
    - Webhook misconfiguration or invalid Gumroad API credentials.  
    - Network issues causing missed events.  
  - **Notes:** This node is the entry point of the workflow.

#### 2.2 Data Cleaning and Extraction

- **Overview:**  
  Cleans and extracts relevant buyer and sale details from the raw Gumroad data for downstream processing.

- **Nodes Involved:**  
  - This is SET to Clean & Extract

- **Node Details:**  
  - **Type:** Set Node  
  - **Role:** Maps and renames fields from the raw Gumroad sale data into a clean, structured format including buyer name, email, product, price, quantity, revenue share, receipt URL, and timestamp.  
  - **Configuration:** Uses expressions to extract and rename fields such as `buyer_name`, `buyer_email`, `product_bought`, `price_paid`, `our_revenue`, `receipt_url`, `sale_time`, and `quantity_bought`.  
  - **Input:** Raw sale data from Gumroad Sales Trigger.  
  - **Output:** Cleaned sale data JSON.  
  - **Version:** 3.4  
  - **Potential Failures:**  
    - Expression errors if expected fields are missing or renamed in Gumroad API.  
    - Null or malformed data causing incomplete extraction.

#### 2.3 Sales Logging

- **Overview:**  
  Logs the cleaned sale data into a Google Sheet for record-keeping and tracking.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Type:** Google Sheets Node  
  - **Role:** Appends a new row with sale details to a configured Google Sheet.  
  - **Configuration:** Authenticated with Google Sheets API; target spreadsheet and sheet configured to receive columns such as Name, Email, Product, Price, Quantity, Timestamp, Receipt URL, and Revenue share.  
  - **Input:** Cleaned sale data from the Set node.  
  - **Output:** Confirmation of row addition; passes data forward.  
  - **Version:** 4.5  
  - **Potential Failures:**  
    - Authentication errors with Google API.  
    - Sheet not found or incorrect sheet name.  
    - Quota limits or API rate limiting.  
  - **Notes:** This node feeds data into the Telegram notification node.

#### 2.4 Telegram Notification

- **Overview:**  
  Sends a Telegram message to notify the user of the new sale.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**  
  - **Type:** Telegram Node  
  - **Role:** Sends a message via Telegram bot to a configured chat ID with sale details.  
  - **Configuration:** Connected to Telegram Bot API with bot token and chat ID configured; message content includes buyer name, product, price, and receipt link.  
  - **Input:** Data from Google Sheets node.  
  - **Output:** Passes data to conditional email sending node.  
  - **Version:** 1.2  
  - **Potential Failures:**  
    - Telegram bot token or chat ID misconfiguration.  
    - Telegram API downtime or rate limits.  
  - **Connections:** Output feeds into the If node for email sending decision.

#### 2.5 Conditional Email Sending

- **Overview:**  
  Determines whether to send a thank-you email and, if yes, prepares and sends the email.

- **Nodes Involved:**  
  - If  
  - Prepare a manual email to replicate  
  - Gmail (disabled by default)

- **Node Details:**  
  - **If Node:**  
    - **Type:** If Node  
    - **Role:** Evaluates a condition (likely whether the buyer has opted in or email sending is enabled).  
    - **Configuration:** Condition not explicitly detailed; expected to check a boolean or flag in the data.  
    - **Input:** From Telegram node.  
    - **Output:** If true, proceeds to prepare email; if false, ends workflow.  
    - **Version:** 2.2  
    - **Potential Failures:** Expression errors or missing condition data.

  - **Prepare a manual email to replicate:**  
    - **Type:** Set Node  
    - **Role:** Constructs email fields such as recipient address, subject, reply-to, HTML body, and sender name.  
    - **Configuration:** Uses expressions to personalize email content with buyer name, product name, and receipt URL.  
    - **Input:** From If node (true branch).  
    - **Output:** Email data JSON.  
    - **Version:** 3.4  
    - **Potential Failures:** Missing or malformed email addresses or content variables.

  - **Gmail Node (disabled):**  
    - **Type:** Gmail Node  
    - **Role:** Sends the prepared thank-you email via Gmail API.  
    - **Configuration:** Requires Gmail OAuth2 credentials; currently disabled to allow optional use.  
    - **Input:** From Prepare a manual email node.  
    - **Output:** Email send confirmation.  
    - **Version:** 2.1  
    - **Potential Failures:** Authentication errors, quota limits, or invalid email parameters.  
    - **Notes:** Can be swapped with other email nodes (Outlook, SMTP, Mailgun) as per customization instructions.

#### 2.6 Workflow Notes and Disabled Nodes

- **Overview:**  
  Contains multiple sticky notes for guidance, setup instructions, and optional upgrades. Also includes disabled nodes for alternative integrations (Airtable, HubSpot, Notion, Twilio, Microsoft Outlook, AI email writer).

- **Nodes Involved:**  
  - Multiple Sticky Note nodes  
  - Disabled nodes: Airtable1, HubSpot, Notion, Twilio, Microsoft Outlook, Send Email, Email Writer Agent

- **Node Details:**  
  - **Sticky Notes:** Provide color-coded hints and setup instructions (green for main blocks, blue for setup, yellow for optional upgrades).  
  - **Disabled Nodes:** Serve as placeholders or examples for extending the workflow with other CRM, messaging, or AI email generation tools.  
  - **Potential Failures:** None, as these nodes are disabled or informational.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                         | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                      |
|--------------------------------|-------------------------|---------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Gumroad Sales Trigger           | Gumroad Trigger         | Trigger on new Gumroad sales          | -                          | This is SET to Clean & Extract |                                                                                                |
| This is SET to Clean & Extract  | Set                     | Clean and extract sale data            | Gumroad Sales Trigger       | Google Sheets                 |                                                                                                |
| Google Sheets                  | Google Sheets           | Log sale data into Google Sheet        | This is SET to Clean & Extract | Telegram                      |                                                                                                |
| Telegram                      | Telegram                | Send Telegram notification             | Google Sheets               | If                            |                                                                                                |
| If                            | If                      | Conditional check to send email        | Telegram                    | Prepare a manual email to replicate |                                                                                                |
| Prepare a manual email to replicate | Set                     | Prepare thank-you email content         | If                         | Gmail                         |                                                                                                |
| Gmail                         | Gmail                   | Send thank-you email (optional)        | Prepare a manual email to replicate | -                             | Disabled by default; can be swapped with Outlook, SMTP, Mailgun                                |
| Airtable1                     | Airtable                | Alternative CRM logging (disabled)     | -                          | HubSpot                       | Disabled node for optional CRM integration                                                     |
| HubSpot                       | HubSpot                 | Alternative CRM integration (disabled) | Airtable1                  | Notion                        | Disabled node for optional CRM integration                                                     |
| Notion                        | Notion                  | Alternative CRM integration (disabled) | HubSpot                    | -                             | Disabled node for optional CRM integration                                                     |
| Twilio                        | Twilio                  | Alternative messaging (disabled)       | -                          | -                             | Disabled node for optional SMS integration                                                     |
| Microsoft Outlook             | Microsoft Outlook       | Alternative email sending (disabled)   | -                          | Send Email                    | Disabled node for optional email sending                                                       |
| Send Email                    | EmailSend               | Alternative email sending (disabled)   | Microsoft Outlook           | -                             | Disabled node for optional email sending                                                       |
| Email Writer Agent            | Langchain Agent         | AI-generated email drafting (disabled) | -                          | -                             | Disabled node for optional AI email generation                                                 |
| Sticky Note(s)                | Sticky Note             | Setup instructions and guidance        | -                          | -                             | Multiple notes with color-coded hints and setup instructions                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gumroad Sales Trigger Node**  
   - Type: Gumroad Trigger  
   - Configure with Gumroad API credentials and webhook ID to listen for new sales.  
   - No input connections; output raw sale JSON.

2. **Add Set Node to Clean & Extract Data**  
   - Type: Set  
   - Connect input from Gumroad Sales Trigger.  
   - Map fields from raw sale data to new keys:  
     - `buyer_name` ← full_name  
     - `buyer_email` ← email  
     - `product_bought` ← product_name  
     - `price_paid` ← price (converted to dollars if needed)  
     - `our_revenue` ← calculated revenue share (e.g., 70% cut)  
     - `receipt_url` ← receipt_url  
     - `sale_time` ← created_at  
     - `quantity_bought` ← quantity  
     - `referrer` ← referrer  
   - Use expressions to extract and rename these fields.

3. **Add Google Sheets Node to Log Sales**  
   - Type: Google Sheets  
   - Connect input from Set node.  
   - Authenticate with Google Sheets API using OAuth2 credentials.  
   - Configure target spreadsheet and sheet name.  
   - Map fields to columns: Name, Email, Product, Price, Quantity, Timestamp, Receipt URL, Our Cut.  
   - Set operation to append rows.

4. **Add Telegram Node for Notifications**  
   - Type: Telegram  
   - Connect input from Google Sheets node.  
   - Authenticate with Telegram Bot API (bot token).  
   - Configure chat ID for notifications.  
   - Compose message including buyer name, product, price, and receipt link.

5. **Add If Node for Conditional Email Sending**  
   - Type: If  
   - Connect input from Telegram node.  
   - Configure condition to check if email sending is enabled or buyer consented (e.g., check a boolean flag in data).  
   - True branch proceeds to email preparation; false branch ends workflow.

6. **Add Set Node to Prepare Email Content**  
   - Type: Set  
   - Connect input from If node (true branch).  
   - Configure fields:  
     - `to`: buyer_email  
     - `subject`: e.g., "Thank you for trusting Innovatio! ✨"  
     - `reply_to`: your support email (e.g., velebit@innovatio.design)  
     - `body_html`: personalized HTML email including buyer name, product name, receipt URL, and brand signature.  
     - `from_name`: your brand name (e.g., Innovatio)

7. **Add Gmail Node to Send Email (Optional)**  
   - Type: Gmail  
   - Connect input from Set node preparing email.  
   - Authenticate with Gmail OAuth2 credentials.  
   - Map email fields accordingly.  
   - Enable node only if email sending is desired.

8. **Add Sticky Notes for Setup and Guidance**  
   - Add multiple Sticky Note nodes with color-coded content:  
     - Green for main blocks explanation  
     - Blue for setup instructions and editable fields  
     - Yellow for optional upgrades and alternative integrations

9. **(Optional) Add Disabled Nodes for Extensions**  
   - Airtable, HubSpot, Notion for CRM logging alternatives  
   - Twilio for SMS notifications  
   - Microsoft Outlook or SMTP for alternative email sending  
   - AI Email Writer Agent for dynamic email content generation

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This template is designed for Gumroad creators, solopreneurs, digital product sellers, and freelancers. | Workflow purpose and target audience.                                                               |
| Setup instructions include connecting Gumroad API, Google Sheets, Telegram bot, and optionally Gmail. | Setup guidance for credentials and node configuration.                                              |
| Color-coded sticky notes system: Green = main blocks, Blue = setup guidance, Yellow = optional upgrades. | Helps users customize and understand the workflow easily.                                           |
| Customization options include swapping Gmail with Outlook/SMTP/Mailgun, Telegram with WhatsApp/Discord/Slack, and logging to Airtable or Notion. | Flexibility and extensibility of the workflow.                                                      |
| Built by Velebit from Innovatio; support and customization contact: velebit@innovatio.design         | Creator credit and support contact.                                                                 |
| Paid version available on Gumroad with additional modules and commercial use rights.                 | Licensing and upgrade information.                                                                  |
| Video and blog resources may be linked in sticky notes for further help (not included in JSON).      | Additional learning and support resources.                                                          |

---

This documentation provides a detailed, structured understanding of the "Gumroad Clientbell" workflow, enabling users and AI agents to comprehend, reproduce, and customize the automation effectively.