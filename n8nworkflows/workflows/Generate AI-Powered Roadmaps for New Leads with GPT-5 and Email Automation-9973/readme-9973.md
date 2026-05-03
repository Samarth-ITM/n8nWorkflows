Generate AI-Powered Roadmaps for New Leads with GPT-5 and Email Automation

https://n8nworkflows.xyz/workflows/generate-ai-powered-roadmaps-for-new-leads-with-gpt-5-and-email-automation-9973


# Generate AI-Powered Roadmaps for New Leads with GPT-5 and Email Automation

### 1. Workflow Overview

This workflow automates the generation and delivery of personalized AI-powered roadmaps for new leads submitted via a web form. It captures lead information, processes their business challenges with GPT-5 AI agents to produce a strategic automation roadmap, formats it into a professional HTML email, and sends it automatically to the lead. Internal notifications are optionally sent via email and Telegram to alert the team of new leads.

Logical blocks:

- **1.1 Input Reception and Data Storage:** Receives lead data via webhook and stores it in a data table.
- **1.2 AI Processing:** Uses GPT-5 models to analyze challenges and generate a personalized automation roadmap.
- **1.3 HTML Styling:** Formats the AI-generated roadmap into professional, email-ready HTML.
- **1.4 Email Delivery:** Sends the styled roadmap to the customer and sends internal notifications.
- **1.5 Post-Processing Update:** Updates the data table to mark the lead as contacted.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Storage

- **Overview:** Captures lead submission data from an external form/webhook and inserts or updates the lead record in a database table for tracking.
- **Nodes Involved:** Webhook, Add Customer Data

##### Node: Webhook
- Type: Webhook (HTTP POST receiver)
- Configuration:  
  - Path: `lead-magnet`  
  - HTTP Method: POST  
- Inputs: External HTTP POST request with JSON payload containing lead info: name, email, company type, URL, primary goal/challenges, timestamp.
- Outputs: JSON object with lead data passed to next node.
- Edge cases:  
  - Missing or malformed payload may cause failure.  
  - Ensure webhook URL is correctly configured and accessible.  
- Notes: Entry point of the workflow.

##### Node: Add Customer Data
- Type: Data Table Upsert (database table interaction)
- Configuration:  
  - Target table: “Lead Magnet” data table (must be pre-created with specific columns).  
  - Columns set from webhook data: Name, Email, CompanyType, CompanyURL, Challenges, Tag (default "New"), EmailSent (default false).  
  - Operation: Upsert based on Email (to avoid duplicates).  
- Inputs: JSON data from webhook.  
- Outputs: JSON data passed to AI Consultant node.  
- Failure cases: Database connection issues, duplicate keys if Email is not unique.  
- Setup Requirements: Manual creation of a Data Table with specified columns (Name, Email, CompanyType, CompanyURL, Challenges, EmailSent, Tag).

---

#### 1.2 AI Processing

- **Overview:** Analyzes the customer’s specified challenges using GPT-5 AI models to produce a clear, actionable automation roadmap.
- **Nodes Involved:** AI Consultant, GPT 5 Nano, GPT 5 Mini

##### Node: AI Consultant
- Type: Langchain Agent with OpenAI GPT-5 Nano model  
- Configuration:  
  - Input prompt: Customer’s "Challenges" field from data table.  
  - System message: Detailed prompt instructing the AI to act as an expert automation consultant and generate a structured roadmap with phases, ROI, and next steps.  
- Inputs: Customer challenges text from Add Customer Data node.  
- Outputs: Raw text roadmap output, passed to Style Agent.  
- Edge cases: AI timeout, API quota limits, incomplete challenge descriptions.  
- Credentials: OpenAI API key required.  
- Notes: Core AI logic for roadmap generation.

##### Node: GPT 5 Nano
- Type: OpenAI GPT-5 Nano model (alternative AI model, not directly connected in this workflow; possibly legacy or optional).  
- Configuration: GPT-5 Nano model selected, no additional prompt.  
- Inputs/Outputs: Not connected downstream in this workflow.  
- Notes: Present but unused in current flow.

##### Node: GPT 5 Mini
- Type: OpenAI GPT-5 Mini model  
- Configuration: GPT-5 Mini with 60s timeout.  
- Inputs: Not directly connected in main flow but linked to Style Agent.  
- Notes: May be used for another AI step or fallback.

---

#### 1.3 HTML Styling

- **Overview:** Converts the raw AI roadmap text into a professional, fully formatted HTML email document with inline CSS.
- **Nodes Involved:** Style Agent

##### Node: Style Agent
- Type: Langchain Agent (custom AI agent for HTML styling)  
- Configuration:  
  - Input: Raw roadmap text output from AI Consultant.  
  - System prompt: Strong instructions to produce complete, self-contained, email-ready HTML with inline CSS, semantic structure, and specific branding placeholders.  
  - Output: Complete HTML document ready for email insertion.  
- Inputs: Raw text roadmap from AI Consultant.  
- Outputs: Formatted HTML to email sending node.  
- Edge cases: AI formatting errors, incomplete HTML output, potential branding placeholder omissions.  
- Notes: Requires customization with your logo URL in prompt.

---

#### 1.4 Email Delivery and Notifications

- **Overview:** Sends the generated roadmap to the customer via email, updates the data table, and optionally sends internal notifications via email and Telegram.
- **Nodes Involved:** Generated Roadmap to Customer, Update row(s), Send email, Send a text message

##### Node: Generated Roadmap to Customer
- Type: Email Send  
- Configuration:  
  - To: Customer email from "Add Customer Data" node.  
  - From: Company email address.  
  - Subject: "Your roadmap is ready!"  
  - Content: Styled HTML output from Style Agent node.  
- Inputs: HTML formatted roadmap.  
- Outputs: Triggers Update row(s) node.  
- Edge cases: SMTP authentication failure, invalid email address, spam filtering.  
- Credentials: SMTP credentials required.

##### Node: Update row(s)
- Type: Data Table Update  
- Configuration:  
  - Updates current lead row: set EmailSent=true, Tag="Delivered".  
  - Uses lead record ID from Add Customer Data.  
- Inputs: After email sent.  
- Outputs: Triggers internal notifications.  
- Edge cases: Database write failure.

##### Node: Send email
- Type: Email Send (Internal notification)  
- Configuration:  
  - To: Internal email address (fixed).  
  - Subject: "New Client!"  
  - Content: Summary of client data and styled report.  
- Inputs: After row update.  
- Credentials: SMTP.  
- Edge cases: Same as other email node.

##### Node: Send a text message
- Type: Telegram Message Send (Internal notification)  
- Configuration:  
  - Chat ID: Your Telegram chat ID (must be updated)  
  - Text: Summary of lead data using HTML formatting.  
- Inputs: After row update.  
- Credentials: Telegram API token.  
- Edge cases: Telegram API limits, invalid chat ID.

---

#### 1.5 Post-Processing Update

- This is part of the Update row(s) node, marking the lead as contacted, ensuring accurate tracking.

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                             | Input Node(s)          | Output Node(s)                    | Sticky Note                                                                                                                                    |
|-------------------------|-----------------------------------|--------------------------------------------|------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                 | Webhook                           | Entry point, receives lead data             | External HTTP POST     | Add Customer Data                | ## Webhook Entrypoint, Payload example, URL formats                                                                                           |
| Add Customer Data       | Data Table Upsert                 | Stores lead data in database                 | Webhook                | AI Consultant                   | ## Add customer data, Data table schema instructions                                                                                           |
| AI Consultant           | Langchain AI Agent (GPT-5 Nano)  | Generates automation roadmap from challenges| Add Customer Data       | Style Agent                    | ## AI Consultant, prompt customization instructions                                                                                            |
| GPT 5 Nano              | OpenAI GPT-5 Nano model           | Alternate AI model (unused in flow)          | -                      | -                                |                                                                                                                                               |
| GPT 5 Mini              | OpenAI GPT-5 Mini model           | AI model sometimes used for styling          | -                      | Style Agent                    |                                                                                                                                               |
| Style Agent             | Langchain AI Agent                | Formats roadmap as professional HTML email  | AI Consultant, GPT 5 Mini | Generated Roadmap to Customer | ## Style Agent, HTML email generation instructions                                                                                            |
| Generated Roadmap to Customer | Email Send                   | Sends styled roadmap email to customer       | Style Agent             | Update row(s)                   | ## Send Roadmap to Customer, email sender and recipient customization                                                                         |
| Update row(s)           | Data Table Update                 | Marks lead as contacted after email sent     | Generated Roadmap to Customer | Send email, Send a text message | ## Updates rows, sets EmailSent=true, Tag=Delivered                                                                                           |
| Send email              | Email Send                       | Sends internal notification email            | Update row(s)           | -                              | ## Notification email setup instructions                                                                                                     |
| Send a text message     | Telegram Message                 | Sends internal notification via Telegram     | Update row(s)           | -                              | ## Notification Telegram setup instructions                                                                                                  |
| Sticky Note             | Sticky Note                     | Documentation and workflow explanation        | -                      | -                              | Covers multiple nodes; explains workflow purpose and setup                                                                                   |
| Sticky Note1            | Sticky Note                     | AI Consultant node instructions                | -                      | -                              | AI Consultant customization notes                                                                                                           |
| Sticky Note2            | Sticky Note                     | Style Agent instructions                        | -                      | -                              | Style Agent customization notes                                                                                                             |
| Sticky Note3            | Sticky Note                     | Generated Roadmap to Customer instructions     | -                      | -                              | Email node customization notes                                                                                                              |
| Sticky Note4            | Sticky Note                     | Notifications setup instructions                | -                      | -                              | Email and Telegram notification setup notes                                                                                                 |
| Sticky Note5            | Sticky Note                     | Add Customer Data setup instructions            | -                      | -                              | Data Table creation and schema notes                                                                                                        |
| Sticky Note6            | Sticky Note                     | Update row(s) setup instructions                 | -                      | -                              | Update row configuration notes                                                                                                              |
| Sticky Note7            | Sticky Note                     | Webhook setup and payload example                | -                      | -                              | Webhook usage and payload example                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `lead-magnet`  
   - Purpose: Entry point to receive lead form submissions.

2. **Create a Data Table for Leads**  
   - Name: `Lead Magnet`  
   - Columns:  
     - Name (String)  
     - Email (String)  
     - CompanyType (String)  
     - CompanyURL (String)  
     - Challenges (String)  
     - EmailSent (Boolean)  
     - Tag (String)  
   - Purpose: Store leads and track email delivery status.

3. **Add a Data Table Upsert Node ("Add Customer Data")**  
   - Operation: Upsert by Email  
   - Map webhook data fields to table columns:  
     - Name ← `{{$json.body.name}}`  
     - Email ← `{{$json.body.email}}`  
     - CompanyType ← `{{$json.body.companyType}}`  
     - CompanyURL ← `{{$json.body.companyUrl}}`  
     - Challenges ← `{{$json.body.primaryGoal}}`  
     - Tag ← "New" (default)  
     - EmailSent ← false (default)  
   - Connect Webhook → Add Customer Data.

4. **Create AI Agent Node ("AI Consultant")**  
   - Type: Langchain Agent with OpenAI GPT-5 Nano model  
   - Credentials: Set your OpenAI API key  
   - Set input to `{{$json.Challenges}}` from Add Customer Data node  
   - Paste system message prompt that instructs the AI to generate a business automation roadmap with phases, ROI, and next steps.  
   - Connect Add Customer Data → AI Consultant.

5. **Create AI Agent Node ("Style Agent")**  
   - Type: Langchain Agent for HTML styling  
   - Credentials: OpenAI API key  
   - Input: raw text output from AI Consultant (`{{$json.output}}`)  
   - System prompt: Complete, self-contained HTML email generation instructions including inline CSS and branding placeholders. Replace `[Your Logo Link]` with your logo URL.  
   - Connect AI Consultant → Style Agent.

6. **Create Email Send Node ("Generated Roadmap to Customer")**  
   - SMTP credentials: configure your company SMTP account  
   - To Email: `{{$('Add Customer Data').item.json.Email}}`  
   - From Email: `YourCompanyName <your@email.com>` (replace accordingly)  
   - Subject: "Your roadmap is ready!"  
   - Content: HTML from Style Agent output (`{{$json.output}}`)  
   - Connect Style Agent → Generated Roadmap to Customer.

7. **Create Data Table Update Node ("Update row(s)")**  
   - Operation: Update lead in `Lead Magnet` table by ID from Add Customer Data node  
   - Set `EmailSent` to true  
   - Set `Tag` to "Delivered"  
   - Connect Generated Roadmap to Customer → Update row(s).

8. **Create Internal Notification Email Node ("Send email")**  
   - SMTP credentials: same SMTP account  
   - To Email: your internal email (e.g., your@email.com)  
   - From Email: `Lead Magnet <your@email.com>`  
   - Subject: "New Client!"  
   - Content: Text summary with client info and styled report from Style Agent output  
   - Connect Update row(s) → Send email.

9. **Create Telegram Message Node ("Send a text message")**  
   - Credentials: Telegram API token configured  
   - Chat ID: replace with your Telegram chat ID  
   - Message text: formatted with client details using template expressions  
   - Connect Update row(s) → Send a text message.

10. **Test the entire workflow**  
    - Post sample JSON payload to the webhook URL (e.g., via Postman or your form).  
    - Confirm data is stored, AI roadmap generated and styled, emails sent, and notifications received.

11. **Activate the workflow** for production use.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow transforms manual lead engagement into a fully automated, AI-driven experience delivering instant, personalized value to every new contact.                                                                          | Sticky Note overview explaining the workflow purpose.           |
| AI Consultant prompt requires you to replace **[Your Company Name]** and **[Your Calendly Link]** with your actual company name and scheduling link.                                                                              | Sticky Note1                                                     |
| Style Agent prompt requires replacing **[Your Logo Link]** with your company logo URL to brand the HTML emails correctly.                                                                                                          | Sticky Note2                                                     |
| Email node parameters must be configured with your real email addresses for sender and recipient in both customer and internal notifications.                                                                                      | Sticky Note3                                                     |
| For Telegram notifications, you must insert your Telegram Chat ID and ensure the Telegram API credential is set up with a valid bot token.                                                                                        | Sticky Note4                                                     |
| The "Lead Magnet" data table schema must be created manually before running the workflow, matching specified column names and types.                                                                                              | Sticky Note5                                                     |
| The data table update node marks leads as contacted, setting EmailSent to true and Tag to Delivered to prevent duplicates and enable tracking.                                                                                     | Sticky Note6                                                     |
| Webhook URL for testing and production must be properly substituted with your n8n instance URL. Example payload format is provided in the notes for quick integration with external forms or APIs.                                  | Sticky Note7                                                     |

---

**Disclaimer:** The provided text and workflow solely originate from an automated n8n workflow and comply with content policies. All data processed is legal and public.