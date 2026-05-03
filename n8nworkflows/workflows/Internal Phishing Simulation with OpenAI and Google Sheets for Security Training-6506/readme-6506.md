Internal Phishing Simulation with OpenAI and Google Sheets for Security Training

https://n8nworkflows.xyz/workflows/internal-phishing-simulation-with-openai-and-google-sheets-for-security-training-6506


# Internal Phishing Simulation with OpenAI and Google Sheets for Security Training

### 1. Workflow Overview

This workflow, titled **"Internal Phishing Simulation with OpenAI and Google Sheets for Security Training"**, is designed to automate the generation and delivery of phishing simulation emails to a list of internal targets. It integrates AI-generated content via OpenAI, email sending through Gmail, and tracking of recipient interactions using webhooks and Google Sheets. The purpose is to conduct phishing awareness training by simulating phishing attacks and logging both email sends and user clicks.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger and acquire the list of target recipients from Google Sheets.
- **1.2 AI Processing:** Generate customized phishing email content using OpenAI models.
- **1.3 Email Preparation and Sending:** Format the generated email and send it via Gmail.
- **1.4 Logging Sent Emails:** Record details of sent emails into Google Sheets for tracking.
- **1.5 Click Tracking and Logging:** Capture recipient clicks on phishing links via a webhook and log these events into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and retrieves the list of phishing targets from a Google Sheets spreadsheet.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- GetTargetList

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow on demand.  
  - Configuration: No parameters; triggers workflow execution when manually activated.  
  - Connections: Outputs to GetTargetList node.  
  - Edge Cases: None significant; manual execution may limit automation.  
  
- **GetTargetList**  
  - Type: Google Sheets (Read)  
  - Role: Reads the phishing target list (email addresses and possibly other metadata) from a specified spreadsheet.  
  - Configuration: Reads from a configured Google Sheets document and sheet; expects credentials with Google Sheets API access.  
  - Inputs: Receives trigger from manual start node.  
  - Outputs: Feeds list data to AI processing node.  
  - Edge Cases: Authentication failures, empty or malformed sheets, API rate limits.

---

#### 1.2 AI Processing

**Overview:**  
Generates phishing email content tailored to the target list using an OpenAI Chat Model through a Langchain agent node.

**Nodes Involved:**  
- OpenAI Chat Model  
- 🤖 Generate Phishing Email

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides AI language processing capabilities to generate phishing email text.  
  - Configuration: Uses OpenAI credentials with proper API key and model selection (e.g., GPT-4 or GPT-3.5).  
  - Inputs: Feeds AI model into the Langchain agent node.  
  - Outputs: Provides language model interface for phishing email generation.  
  - Edge Cases: API key invalid/expired, rate limits, response delays, model unavailability.  
  - Version: 1.2  

- **🤖 Generate Phishing Email**  
  - Type: Langchain Agent Node  
  - Role: Coordinates AI prompt execution, crafts phishing email content based on input target list and model output.  
  - Configuration: Uses OpenAI Chat Model as the language model backend. Prompts likely include phishing scenarios and personalization.  
  - Inputs: Receives target list from GetTargetList; uses OpenAI Chat Model node as AI languageModel input.  
  - Outputs: Provides generated email content for formatting.  
  - Edge Cases: Prompt failures, unexpected AI output, connection issues with AI model.  
  - Version: 2  

---

#### 1.3 Email Preparation and Sending

**Overview:**  
Prepares the output of AI-generated email content into the proper format and sends it via Gmail.

**Nodes Involved:**  
- ✉️ Format Email for Sending  
- Gmail

**Node Details:**

- **✉️ Format Email for Sending**  
  - Type: Set Node  
  - Role: Maps and structures the AI output into email fields required by Gmail node (e.g., recipient, subject, body, attachments).  
  - Configuration: Uses expressions to set email parameters like "To", "Subject", "HTML Body".  
  - Inputs: Receives AI-generated phishing email content.  
  - Outputs: Passes formatted email data to Gmail node.  
  - Edge Cases: Incorrect formatting leading to email sending errors, missing fields.  
  - Version: 3.4  

- **Gmail**  
  - Type: Gmail Node  
  - Role: Sends the formatted phishing email to each target.  
  - Configuration: Uses OAuth2 credentials for Gmail access with "send email" scope. Supports batch sending if configured.  
  - Inputs: Takes formatted email data from previous node.  
  - Outputs: Passes sending result to logging node.  
  - Edge Cases: Authentication errors, quota limits, invalid recipient addresses, network failures.  
  - Version: 2.1  

---

#### 1.4 Logging Sent Emails

**Overview:**  
Logs details of each sent phishing email into a Google Sheets document for record-keeping and analysis.

**Nodes Involved:**  
- 🧾 Log Sent Email

**Node Details:**

- **🧾 Log Sent Email**  
  - Type: Google Sheets (Append/Update)  
  - Role: Records metadata such as recipient email, timestamp, email content snippet, or message ID.  
  - Configuration: Points to a Google Sheets document and sheet dedicated to log storage.  
  - Inputs: Receives Gmail send results.  
  - Outputs: None or further downstream processing if configured.  
  - Edge Cases: Sheet access issues, quota limits, data format mismatches.  
  - Version: 4.6  

---

#### 1.5 Click Tracking and Logging

**Overview:**  
Tracks user clicks on phishing email links via an HTTP webhook, records click data, and logs it into Google Sheets.

**Nodes Involved:**  
- 🎯 Click Tracker  
- 🗂️ Record Click (Set)  
- 🧾 Record Click (Google Sheets)

**Node Details:**

- **🎯 Click Tracker**  
  - Type: Webhook  
  - Role: Listens for HTTP requests triggered when a recipient clicks a phishing link embedded in the email.  
  - Configuration: Configured with a unique webhook URL and method (likely GET or POST).  
  - Inputs: External HTTP requests from user clicks.  
  - Outputs: Sends data to Record Click (Set) node.  
  - Edge Cases: Webhook URL exposure, replay attacks, missing parameters, request timeouts.  
  - Version: 2  

- **🗂️ Record Click (Set)**  
  - Type: Set Node  
  - Role: Organizes and formats click event data (e.g., timestamp, user ID, IP address) for logging.  
  - Configuration: Uses expressions to extract and map webhook data fields.  
  - Inputs: Receives data from Click Tracker webhook.  
  - Outputs: Provides structured click data to Google Sheets logging node.  
  - Edge Cases: Missing data fields, malformed requests.  
  - Version: 3.4  

- **🧾 Record Click (Google Sheets)**  
  - Type: Google Sheets (Append)  
  - Role: Logs click event details into a Google Sheets document for tracking and reporting.  
  - Configuration: Points to a dedicated sheet for click records.  
  - Inputs: Receives formatted click data.  
  - Outputs: None.  
  - Edge Cases: Authentication issues, sheet write conflicts, quota limits.  
  - Version: 4.6  

---

### 3. Summary Table

| Node Name                   | Node Type                               | Functional Role                     | Input Node(s)                      | Output Node(s)                    | Sticky Note |
|-----------------------------|---------------------------------------|-----------------------------------|----------------------------------|---------------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger                        | Workflow manual start trigger      |                                  | GetTargetList                   |             |
| GetTargetList               | Google Sheets                         | Retrieve phishing target list      | When clicking ‘Execute workflow’ | 🤖 Generate Phishing Email       |             |
| OpenAI Chat Model           | Langchain OpenAI Chat Model           | Provide AI language model          |                                  | 🤖 Generate Phishing Email (ai_languageModel) |             |
| 🤖 Generate Phishing Email   | Langchain Agent Node                  | Generate phishing email content    | GetTargetList, OpenAI Chat Model | ✉️ Format Email for Sending      |             |
| ✉️ Format Email for Sending   | Set Node                             | Format email for Gmail send        | 🤖 Generate Phishing Email       | Gmail                          |             |
| Gmail                      | Gmail Node                           | Send phishing email                | ✉️ Format Email for Sending       | 🧾 Log Sent Email               |             |
| 🧾 Log Sent Email            | Google Sheets                        | Log sent email details             | Gmail                            |                                 |             |
| 🎯 Click Tracker             | Webhook                             | Capture phishing link clicks       |                                  | 🗂️ Record Click                |             |
| 🗂️ Record Click             | Set Node                             | Format click event data            | 🎯 Click Tracker                 | 🧾 Record Click                |             |
| 🧾 Record Click              | Google Sheets                        | Log click event details            | 🗂️ Record Click                 |                                 |             |
| Sticky Note                 | Sticky Note                         | Comments/notes                    |                                  |                                 |             |
| Sticky Note1                | Sticky Note                         | Comments/notes                    |                                  |                                 |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed. This node will start the workflow manually.

2. **Add Google Sheets Node to Read Target List**  
   - Name: `GetTargetList`  
   - Type: Google Sheets (Read)  
   - Configure credentials with Google Sheets OAuth2.  
   - Set Spreadsheet ID and Sheet name containing target emails.  
   - Connect output of Manual Trigger to this node.

3. **Add OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: Langchain OpenAI Chat Model  
   - Configure with OpenAI API key credentials.  
   - Select appropriate model (e.g., GPT-4).  
   - No input connections (used as ai_languageModel input).

4. **Add Langchain Agent Node for Email Generation**  
   - Name: `🤖 Generate Phishing Email`  
   - Type: Langchain Agent Node  
   - Set `ai_languageModel` input to `OpenAI Chat Model`.  
   - Use the output of `GetTargetList` as input data for dynamic email generation.  
   - Configure prompt templates for phishing email content.  
   - Connect outputs to formatting node.

5. **Add Set Node to Format Email**  
   - Name: `✉️ Format Email for Sending`  
   - Type: Set Node  
   - Map generated email text to fields: To, Subject, HTML Body, etc.  
   - Use expressions to extract data from previous node output.  
   - Connect output from email generation node to this node.

6. **Add Gmail Node to Send Email**  
   - Name: `Gmail`  
   - Type: Gmail Node  
   - Configure OAuth2 credentials with Gmail API access for sending emails.  
   - Connect input from formatting node.  
   - Connect output to logging node.

7. **Add Google Sheets Node to Log Sent Emails**  
   - Name: `🧾 Log Sent Email`  
   - Type: Google Sheets (Append)  
   - Configure with appropriate sheet for sent email logs.  
   - Connect input from Gmail node.

8. **Add Webhook Node for Click Tracking**  
   - Name: `🎯 Click Tracker`  
   - Type: Webhook  
   - Configure unique webhook URL and method (GET/POST).  
   - No input connections (receives external HTTP requests).  
   - Output connected to click data processing node.

9. **Add Set Node to Format Click Data**  
   - Name: `🗂️ Record Click`  
   - Type: Set Node  
   - Map webhook request data (e.g., timestamp, user ID, IP) to structured fields.  
   - Connect input from webhook node.

10. **Add Google Sheets Node to Log Clicks**  
    - Name: `🧾 Record Click`  
    - Type: Google Sheets (Append)  
    - Configure to append to click log spreadsheet.  
    - Connect input from previous Set node.

11. **Add Sticky Notes (Optional)**  
    - Add sticky notes with any comments or instructions for documentation or team use.  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is part of a RedOps training module focusing on internal phishing simulations.     | Project context                                                                                     |
| Uses Langchain nodes for AI integration, leveraging OpenAI Chat models for content generation.   | n8n Langchain integration documentation: https://docs.n8n.io/integrations/ai/langchain/            |
| Gmail node requires OAuth2 credentials with 'send email' scopes enabled.                         | Gmail API Setup: https://developers.google.com/gmail/api/quickstart/js                              |
| Webhook node exposes an endpoint for click tracking; secure URL distribution is essential.       | Best practices for webhook security: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/             |
| Google Sheets nodes require proper OAuth2 scopes for reading and writing.                        | Google Sheets API docs: https://developers.google.com/sheets/api                                    |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.