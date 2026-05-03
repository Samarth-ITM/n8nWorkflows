Capture Website Leads with Slack Notifications, Gmail Responses & Sheets Archiving

https://n8nworkflows.xyz/workflows/capture-website-leads-with-slack-notifications--gmail-responses---sheets-archiving-7219


# Capture Website Leads with Slack Notifications, Gmail Responses & Sheets Archiving

### 1. Workflow Overview

This workflow is designed to capture new leads submitted via a website contact form and streamline the sales team's response process. It automates the reception of form submissions, immediately notifies the sales team via Slack, archives lead details into a Google Sheet for record-keeping, and sends an automatic personalized confirmation email to the lead.

**Target Use Cases:**  
- Small businesses, freelancers, e-commerce stores, and marketing professionals who require fast lead capture and response.  
- Scenarios where website form submissions must be instantly shared with sales teams and stored for tracking.  
- Environments where immediate confirmation emails to leads enhance customer experience.

**Logical Blocks:**  
- **1.1 Input Reception:** Captures incoming HTTP POST requests from the website form.  
- **1.2 Sales Team Notification:** Sends real-time Slack messages notifying the sales team about new leads.  
- **1.3 Data Archiving:** Appends lead information to a Google Sheets spreadsheet for logging.  
- **1.4 Lead Confirmation Email:** Sends an automatic thank-you email to the new lead using Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block serves as the entry point of the workflow, receiving website form submissions via an HTTP webhook configured to listen for POST requests.

**Nodes Involved:**  
- Receive Website Form Submissions

**Node Details:**

- **Receive Website Form Submissions**  
  - **Type:** Webhook Trigger  
  - **Technical Role:** Captures incoming HTTP POST data from the website form on the path `/new-lead`.  
  - **Configuration:**  
    - HTTP Method: POST  
    - Webhook path: `new-lead`  
    - No additional options configured (default behavior).  
  - **Input/Output:**  
    - Input: External HTTP POST request from the website form.  
    - Output: JSON payload containing lead data fields (e.g., name, email, company, message).  
  - **Version Requirements:** n8n version supporting Webhook node v2 or later.  
  - **Potential Failures:**  
    - Improper webhook URL setup on website form side.  
    - Missing or malformed POST data.  
    - Network connectivity issues.  
  - **Sub-workflow:** None  

---

#### 1.2 Sales Team Notification

**Overview:**  
After receiving the lead data, this block formats and sends a Slack message to a specified sales channel, ensuring immediate team awareness.

**Nodes Involved:**  
- Notify Sales Team

**Node Details:**

- **Notify Sales Team**  
  - **Type:** Slack node  
  - **Technical Role:** Sends a message to Slack channel with lead details.  
  - **Configuration:**  
    - Channel selected by ID (must be replaced with actual Slack channel ID).  
    - Message text dynamically generated using expressions to include lead name, company, email, and message content.  
      Example expression:  
      ```
      New Website Lead! - Name: {{ $json.name }}
      Company: {{ $json.company }}
      Email: {{ $json.email }}
      Message: {{ $json.message }}
      ```  
  - **Input/Output:**  
    - Input: Lead JSON data from webhook.  
    - Output: Slack API response (not used further).  
  - **Credentials:** Slack API OAuth token configured under credentials.  
  - **Version Requirements:** Slack node v2.3 or higher recommended for expression support.  
  - **Potential Failures:**  
    - Slack API authentication errors (invalid/missing token).  
    - Incorrect Slack channel ID or inadequate permissions.  
    - Network or API rate limits.  
  - **Sub-workflow:** None  

---

#### 1.3 Data Archiving

**Overview:**  
This block appends each lead’s data into a row of a specified Google Sheet, providing a persistent, organized archive of all leads.

**Nodes Involved:**  
- Archive Lead Data

**Node Details:**

- **Archive Lead Data**  
  - **Type:** Google Sheets node  
  - **Technical Role:** Appends lead information as a new row in a Google Sheets document.  
  - **Configuration:**  
    - Operation: Append  
    - Spreadsheet ID: Must be replaced with the actual Google Sheets document ID.  
    - Sheet Name: "Leads" (default, can be changed).  
    - Data mapped using expressions:  
      - Name: `{{ $json.name }}`  
      - Email: `{{ $json.email }}`  
      - Date: `{{ $now }}` (current timestamp)  
  - **Input/Output:**  
    - Input: Lead data JSON from Slack notification node.  
    - Output: Google Sheets API response (not used further).  
  - **Credentials:** OAuth2 for Google Sheets configured.  
  - **Version Requirements:** Google Sheets node v4.6 or higher.  
  - **Potential Failures:**  
    - Authentication errors with Google API.  
    - Incorrect Spreadsheet ID or sheet name.  
    - API quotas or network problems.  
  - **Sub-workflow:** None  

---

#### 1.4 Lead Confirmation Email

**Overview:**  
Sends an automatic, personalized confirmation email to the lead acknowledging receipt of their message and setting expectations.

**Nodes Involved:**  
- Send Automatic Confirmation Email

**Node Details:**

- **Send Automatic Confirmation Email**  
  - **Type:** Gmail node  
  - **Technical Role:** Sends an email to the lead’s email address.  
  - **Configuration:**  
    - To: `{{ $json.email }}` (lead’s email from input data)  
    - Subject: "Thanks for contacting us!"  
    - Message body:  
      ```
      Hi {{ $json.name }}, thanks for reaching out. We've received your message and will get back to you shortly.
      ```  
  - **Input/Output:**  
    - Input: Lead data JSON from Google Sheets node.  
    - Output: Gmail API response (not further used).  
  - **Credentials:** Gmail OAuth2 credentials configured.  
  - **Version Requirements:** Gmail node v2.1 or later.  
  - **Potential Failures:**  
    - Gmail authentication errors.  
    - Invalid email address format.  
    - Gmail API rate limits or quota exceeded.  
  - **Sub-workflow:** None  

---

### 3. Summary Table

| Node Name                     | Node Type       | Functional Role                  | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                   |
|-------------------------------|-----------------|---------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------|
| Receive Website Form Submissions | Webhook         | Entry point, captures form data | None                          | Notify Sales Team              |                                                                                                               |
| Notify Sales Team              | Slack           | Sends Slack notification        | Receive Website Form Submissions | Archive Lead Data              |                                                                                                               |
| Archive Lead Data             | Google Sheets   | Archives lead data in spreadsheet | Notify Sales Team              | Send Automatic Confirmation Email | Data: Name, Email, Date                                                                                       |
| Send Automatic Confirmation Email | Gmail           | Sends confirmation email         | Archive Lead Data              | None                           |                                                                                                               |
| Sticky Note                   | Sticky Note     | Visual note "## Flow"            | None                          | None                           |                                                                                                               |
| Sticky Note1                  | Sticky Note     | Detailed workflow note and setup | None                          | None                           | # Workflow Note: Website Lead Notification System\n\n---\n\n### **Problem**\nDelayed responses to new website leads can lead to a significant loss of business. When form submissions are only sent to a general email inbox, it often takes too long for the sales team to notice and act on them. This slow response time results in a poor prospect experience and allows competitors to get ahead.\n\n### **Solution**\nThis is a simple but highly effective n8n workflow designed to eliminate response delays. By connecting your website's contact form directly to your team's communication channels, the system ensures that every new lead is instantly captured and a real-time notification is sent. This allows your sales team to act immediately, significantly improving lead-to-conversion rates.\n\n### **For Whom**\nThis workflow is ideal for **small businesses, e-commerce stores, freelancers, and marketing professionals** who need to respond quickly to new business inquiries. It is perfect for anyone who wants to ensure that no lead falls through the cracks and that their sales pipeline remains robust.\n\n### **Scope**\n* **What it includes:**\n    * An instant trigger for every form submission from your website.\n    * Real-time notifications delivered directly to a specified Slack channel.\n    * Optional automatic archiving of lead data into a Google Sheet.\n    * Optional immediate and personalized confirmation emails sent to the new lead.\n\n* **What it excludes:**\n    * Advanced lead scoring or qualification.\n    * Integration with a complex CRM system (though it can be easily extended to do so).\n    * Automated follow-up sequences.\n\n### **How to Set Up**\n\n1.  **Prerequisites:** You will need an n8n instance and accounts for **Slack**, **Google Sheets**, and **Gmail**. Your website's contact form must be able to send data via an HTTP `POST` request to a specific URL.\n2.  **Workflow Import:** Import the workflow's `.json` file into your n8n instance.\n3.  **Credential Configuration:**\n    * Set up credentials for **Slack**, **Google Sheets**, and **Gmail** within n8n.\n4.  **Node-Specific Configuration:**\n    * **`Webhook Trigger`:** Click the node and copy the **Webhook URL**. You must paste this URL into your website's form settings as the submission endpoint.\n    * **`Slack` Node:** Enter the `Channel ID` of the Slack channel where you want notifications to be sent.\n    * **`Google Sheets` Node (Optional):** Enter your `Spreadsheet ID` and `Sheet Name`. Map the form data fields (e.g., `{{ $json.name }}`) to the corresponding columns in your spreadsheet.\n    * **`Gmail` Node (Optional):** Enter the `To` field using the lead's email variable (`{{ $json.email }}`) and customize the subject and body of the confirmation email.\n5.  **Activation:** After all configurations are complete, click **\"Save\"** and then **\"Active\"**. The workflow is now live! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a **Webhook** trigger node named "Receive Website Form Submissions".  
   - Configure HTTP Method to `POST`.  
   - Set the webhook path to `new-lead`.  
   - Save and copy the generated webhook URL for use in your website form.  

2. **Create Slack Node**  
   - Add a **Slack** node named "Notify Sales Team".  
   - Connect the output of "Receive Website Form Submissions" to this node.  
   - Configure the node to send a message to a Slack channel by ID: replace `YOUR_SALES_CHANNEL_ID` with your actual channel ID.  
   - Message text:  
     ```
     New Website Lead! - Name: {{ $json.name }}
     Company: {{ $json.company }}
     Email: {{ $json.email }}
     Message: {{ $json.message }}
     ```  
   - Set up Slack API credentials (OAuth token with chat:write permissions).  

3. **Create Google Sheets Node**  
   - Add a **Google Sheets** node named "Archive Lead Data".  
   - Connect the output of "Notify Sales Team" to this node.  
   - Set operation to `Append`.  
   - Enter your Google Spreadsheet ID in the Document ID field.  
   - Enter the target sheet name, e.g., `Leads`.  
   - Map columns with expressions:  
     - Name: `{{ $json.name }}`  
     - Email: `{{ $json.email }}`  
     - Date: `{{ $now }}` (current timestamp)  
   - Set up Google Sheets OAuth2 credentials with appropriate access scopes.  

4. **Create Gmail Node**  
   - Add a **Gmail** node named "Send Automatic Confirmation Email".  
   - Connect the output of "Archive Lead Data" to this node.  
   - Configure email parameters:  
     - Send To: `{{ $json.email }}`  
     - Subject: "Thanks for contacting us!"  
     - Message:  
       ```
       Hi {{ $json.name }}, thanks for reaching out. We've received your message and will get back to you shortly.
       ```  
   - Configure Gmail OAuth2 credentials to allow sending emails.  

5. **Deployment and Activation**  
   - Ensure all node credentials are valid and tested.  
   - Save the workflow.  
   - Activate the workflow in n8n.  
   - Configure your website’s form to send POST requests to the webhook URL provided by the "Receive Website Form Submissions" node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow addresses the common issue of delayed lead response by integrating instant notifications and data archiving. It is designed to be simple yet effective, facilitating quick sales follow-up and better lead management without requiring complex CRM integrations. It is easily customizable and extendable for additional features such as lead scoring or CRM syncing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow purpose and scope                                                                                      |
| For detailed setup instructions and best practices, refer to the sticky note within the workflow node titled "Sticky Note1". It covers problem definition, solution overview, target users, and setup steps for Slack, Google Sheets, and Gmail credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Embedded sticky note in workflow                                                                                 |
| Remember to replace placeholder IDs such as `YOUR_SALES_CHANNEL_ID` and `YOUR_SPREADSHEET_ID` with actual values from your Slack workspace and Google Sheets document respectively. Credential setup is mandatory for Slack, Google Sheets, and Gmail nodes to function properly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Credential and configuration reminders                                                                          |
| Slack API documentation: https://api.slack.com/messaging/sending                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Slack messaging official docs                                                                                   |
| Google Sheets API docs: https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets.values/append                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Google Sheets API reference                                                                                      |
| Gmail API docs for sending emails: https://developers.google.com/gmail/api/guides/sending                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Gmail API sending guide                                                                                          |

---

*Disclaimer: The provided text is exclusively derived from an automated n8n workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.*