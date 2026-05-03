Qualify & Auto-Reply to Leads with OpenAI, Airtable, and Gmail

https://n8nworkflows.xyz/workflows/qualify---auto-reply-to-leads-with-openai--airtable--and-gmail-10928


# Qualify & Auto-Reply to Leads with OpenAI, Airtable, and Gmail

---
### 1. Workflow Overview

This workflow automates the process of capturing new leads from a website form, qualifying them using AI, storing the enriched lead data in Airtable CRM, and auto-generating email notifications and draft replies for high-priority leads. It is designed for agencies or sales teams that want to efficiently prioritize inbound inquiries and accelerate follow-up communications with personalized AI-generated content.

Logical blocks:

- **1.1 Input Reception & Normalization:** Capture form submissions and prepare lead data for AI analysis.
- **1.2 AI Lead Qualification:** Use OpenAI-powered agents to analyze lead info, returning structured qualification data such as lead score, priority, business type, budget, and timeline.
- **1.3 Data Persistence:** Save the enriched lead data to Airtable CRM for tracking and review.
- **1.4 Lead Quality Filtering:** Determine if a lead is high quality (score ≥ 7) to trigger notifications and auto-replies.
- **1.5 Notification & Alerting:** Notify the internal sales team of high-priority leads via email (Slack and WhatsApp nodes are present but disabled).
- **1.6 Drafting AI Reply for Lead:** Generate an AI-crafted, context-aware email draft reply to the lead, created as a Gmail draft for manual review and sending.
- **1.7 Incoming Email Trigger and Reply Drafting:** Listen for incoming emails matching high-priority lead alerts and generate reply drafts accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization

**Overview:**  
This block triggers the workflow on new form submissions, normalizes and prepares the lead data for AI processing.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Role: Captures lead data instantly from a website form submission via webhook.  
  - Config: Form titled "Leads Form" with required fields for Name, Email, optional Company, Website URL, and a message field for "What do you need help with?"  
  - Inputs: HTTP webhook input triggered by form submission  
  - Outputs: JSON object containing all submitted form fields and metadata (e.g., submission timestamp).  
  - Edge cases: Missing required fields prevented by form validation; malformed inputs unlikely due to form constraints.  
  - Version: 2.3

---

#### 2.2 AI Lead Qualification

**Overview:**  
Analyze and classify the lead data with an AI agent, producing structured output including lead score, priority, business type, estimated budget, timeline, and internal notes.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**  
- **OpenAI Chat Model**  
  - Type: Language Model (GPT-4.1-mini)  
  - Role: Provides the core language model for the AI Agent.  
  - Config: Model set to "gpt-4.1-mini", no special options enabled.  
  - Inputs: Text prompt from AI Agent node.  
  - Outputs: Raw AI completion response.  
  - Credentials: OpenAI API key configured.  
  - Edge cases: API timeout or quota exceeded; model version compatibility.  
  - Version: 1.3  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Wraps the prompt and coordinates interaction with the OpenAI Chat Model and output parser.  
  - Config: The prompt includes detailed instructions to analyze lead info fields and produce a JSON object matching a fixed JSON schema. It enforces strict output formatting and inference rules for missing data.  
  - Inputs: Data from "On form submission" node (normalized lead info) and OpenAI Chat Model response.  
  - Outputs: Parsed JSON object with lead_score, priority, business_type, use_case_summary, budget_normalized_usd, timeline, and ai_notes.  
  - Edge cases: Parsing errors if AI response is malformed; schema validation failures; incomplete or vague lead info requiring inference.  
  - Version: 3  

- **Structured Output Parser**  
  - Type: Output Parser (JSON Schema)  
  - Role: Validates and extracts the structured JSON output from the AI response according to a predefined schema.  
  - Config: JSON schema defines required fields and types, including enumerated values for priority and timeline.  
  - Inputs: Raw AI completion text from OpenAI Chat Model via AI Agent.  
  - Outputs: Strictly validated JSON with AI lead qualification data.  
  - Edge cases: Schema mismatch or invalid JSON causing workflow failures.  
  - Version: 1.3

---

#### 2.3 Data Persistence

**Overview:**  
Save the combined raw and AI-enriched lead data into an Airtable base for CRM tracking.

**Nodes Involved:**  
- Create a record

**Node Details:**  
- **Create a record**  
  - Type: Airtable Node  
  - Role: Writes lead data and AI qualification results into a specified Airtable table.  
  - Config:  
    - Base: AI Lead Qualifier Airtable base  
    - Table: Leads table  
    - Fields mapped with expressions combining form submission fields and AI output fields (e.g., Name, Email, Priority, Lead Score, AI Notes, etc.)  
    - Typecasting enabled to ensure correct data types.  
  - Inputs: JSON output from AI Agent node with combined raw and AI data.  
  - Outputs: Confirmation of created Airtable record.  
  - Credentials: Airtable Personal Access Token configured.  
  - Edge cases: Airtable API rate limits, invalid field mappings, network errors.  
  - Version: 2.1

---

#### 2.4 Lead Quality Filtering

**Overview:**  
Determine if the AI lead score is high (≥7) to trigger notifications and auto-replies.

**Nodes Involved:**  
- Quality Leads Based On Score (If Node)

**Node Details:**  
- **Quality Leads Based On Score**  
  - Type: If (Conditional) Node  
  - Role: Checks if AI lead_score ≥ 7 to classify lead as high quality.  
  - Config: Condition compares lead_score from AI Agent output to threshold 7.  
  - Inputs: AI Agent output after Airtable record creation.  
  - Outputs: True branch if lead_score ≥ 7, else false branch.  
  - Edge cases: Missing or invalid lead_score value could cause condition to fail or misclassify.  
  - Version: 2.2

---

#### 2.5 Notification & Alerting

**Overview:**  
Notify the internal sales team by email when a high-priority lead is detected. Slack and WhatsApp notifications are included but disabled.

**Nodes Involved:**  
- Send a message1 (Gmail)  
- Send a message (Slack, disabled)  
- Send message (WhatsApp, disabled)

**Node Details:**  
- **Send a message1**  
  - Type: Gmail Node (Send Email)  
  - Role: Sends an email notification summarizing the high-priority lead details to the sales team.  
  - Config:  
    - Recipient: growthriseagency@gmail.com  
    - Subject: Includes lead score and lead name dynamically.  
    - Body: Includes lead contact info and AI qualification fields formatted clearly.  
  - Inputs: True branch from Quality Leads Based On Score node.  
  - Credentials: Gmail OAuth2 configured.  
  - Edge cases: Gmail API limits, invalid recipient address, or authentication errors.  
  - Version: 2.1  

- **Send a message (Slack)** and **Send message (WhatsApp)**  
  - Disabled nodes, indicating optional integration channels for notifications.  
  - Could be enabled for multi-channel alerts.  
  - Edge cases: API credentials, webhook URLs, rate limits when enabled.

---

#### 2.6 Drafting AI Reply for Lead

**Overview:**  
Automatically generate a context-aware, friendly draft email reply to the lead inquiry using AI, then create the draft in Gmail for manual review and sending.

**Nodes Involved:**  
- Gmail Trigger  
- If  
- Basic LLM Chain  
- OpenAI Chat Model1  
- Create a draft

**Node Details:**  
- **Gmail Trigger**  
  - Type: Gmail Trigger  
  - Role: Polls Gmail inbox every minute for new emails.  
  - Config: No filters; triggers on any new email.  
  - Credentials: Same Gmail OAuth2 as notification node.  
  - Edge cases: Polling frequency limits, handling large inboxes or spam.  
  - Version: 1.3  

- **If**  
  - Type: Conditional Node  
  - Role: Filters emails whose subject contains "🔥 New High-Priority Lead" to trigger draft reply generation.  
  - Inputs: Gmail Trigger output.  
  - Outputs: True branch proceeds to AI draft generation, false branch discards.  
  - Version: 2.2  

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain  
  - Role: Generates a concise, friendly reply email body based on the lead’s original email snippet.  
  - Config:  
    - Prompt instructs AI to acknowledge inquiry, explain help offered, ask focused questions, and offer booking call link.  
    - Word limit ~180 words, professional tone.  
  - Inputs: Snippet of original lead email from Gmail Trigger.  
  - Outputs: Generated email body text.  
  - Version: 1.7  

- **OpenAI Chat Model1**  
  - Type: Language Model (GPT-4.1-mini)  
  - Role: Provides the underlying model for Basic LLM Chain.  
  - Credentials: Same OpenAI API key as AI Agent.  
  - Version: 1.3  

- **Create a draft**  
  - Type: Gmail Node (Create Draft)  
  - Role: Creates a draft email in Gmail with subject prefixed "Re:" and body from AI-generated text.  
  - Config:  
    - Subject copies original email subject dynamically.  
    - Draft message body from Basic LLM Chain output.  
  - Credentials: Gmail OAuth2 configured.  
  - Edge cases: Gmail API limits, draft creation failures, formatting issues.  
  - Version: 2.1

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                                 | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                      |
|---------------------------|--------------------------------|------------------------------------------------|-------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| On form submission        | Form Trigger                   | Capture lead form submissions                   | (Webhook)               | AI Agent                   | ## 1. Capture the leads form the Form - Leads form submission triggers workflow start           |
| AI Agent                 | LangChain Agent               | Analyze and qualify leads with AI               | On form submission, OpenAI Chat Model, Structured Output Parser | Create a record            |                                                                                                 |
| OpenAI Chat Model         | Language Model (GPT-4.1-mini) | Provides AI model for lead qualification         | AI Agent                | AI Agent                   |                                                                                                 |
| Structured Output Parser  | Output Parser (JSON Schema)   | Validate and extract structured AI output       | OpenAI Chat Model       | AI Agent                   |                                                                                                 |
| Create a record           | Airtable                      | Save lead + AI qualification data                | AI Agent                | Quality Leads Based On Score |                                                                                                 |
| Quality Leads Based On Score | If (Conditional)             | Filter leads by score ≥7                           | Create a record         | Send a message1            | ## 2. Qualify a Lead - Check if lead is high quality (score ≥ 7)                                |
| Send a message1           | Gmail (Send Email)            | Notify sales team of high-priority lead           | Quality Leads Based On Score |                          | ## 3. Send Email To Team - Notifies sales on high quality leads                               |
| Send a message            | Slack (Disabled)              | Optional Slack notification (disabled)            |                         |                            | ## Send Notification ?? (Optional) - Can notify via Slack or WhatsApp (disabled)               |
| Send message              | WhatsApp (Disabled)           | Optional WhatsApp notification (disabled)         |                         |                            |                                                                                                 |
| Gmail Trigger             | Gmail Trigger                 | Poll Gmail inbox for new emails                    |                         | If                         |                                                                                                 |
| If                       | If (Conditional)              | Filter incoming emails for high-priority lead alerts | Gmail Trigger          | Basic LLM Chain            |                                                                                                 |
| Basic LLM Chain           | LangChain LLM Chain           | Generate AI draft reply email                       | OpenAI Chat Model1, If  | Create a draft             | ## 4 . Draft a Reply Email AI Agent - Generates context-aware draft reply for review           |
| OpenAI Chat Model1        | Language Model (GPT-4.1-mini) | Provides AI model for reply generation             | Basic LLM Chain         | Basic LLM Chain            |                                                                                                 |
| Create a draft            | Gmail (Create Draft)          | Create Gmail draft email from AI-generated reply  | Basic LLM Chain         |                            |                                                                                                 |
| Sticky Note               | Sticky Note                   | Documentation and explanations                      |                         |                            | ## 1. Capture the leads form the Form - Workflow explanation                                   |
| Sticky Note1              | Sticky Note                   | Documentation                                         |                         |                            | ## 2. Qualify a Lead - Explanation of lead scoring check                                       |
| Sticky Note2              | Sticky Note                   | Documentation                                         |                         |                            | ## Send Notification ?? (Optional) - Explains optional notification channels                    |
| Sticky Note3              | Sticky Note                   | Documentation                                         |                         |                            | ## 3. Send Email To Team - Explains email notification node                                    |
| Sticky Note4              | Sticky Note                   | Documentation                                         |                         |                            | ## 4 . Draft a Reply Email AI Agent - Explains automatic draft reply generation                |
| Sticky Note5              | Sticky Note                   | Documentation                                         |                         |                            | # 🧠 How This Lead Qualifier + Draft Reply Agent Works - Full workflow step summary            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node (Form Trigger):**  
   - Type: Form Trigger  
   - Configure form titled "Leads Form" with fields:  
     - Name (text, required)  
     - Email (email, required)  
     - Company / Business Name (text)  
     - Website (URL)  
     - What do you need help with? (text)  
   - Description: "Work with Us to get more work done."  
   - Position: Left top area of editor.  

2. **Add "OpenAI Chat Model" node:**  
   - Type: LangChain OpenAI Chat Model  
   - Model: gpt-4.1-mini  
   - Connect no direct input; will be linked from AI Agent node as languageModel input.  
   - Configure OpenAI API credentials.  

3. **Add "Structured Output Parser" node:**  
   - Type: Output Parser Structured (JSON Schema)  
   - Paste the JSON schema exactly as provided in the workflow for lead qualification output.  

4. **Add "AI Agent" node:**  
   - Type: LangChain Agent  
   - Prompt: Use the detailed prompt that instructs the agent to analyze lead info and return structured JSON matching schema.  
   - Connect inputs:  
     - From "On form submission" node (lead data)  
     - From "OpenAI Chat Model" node (languageModel)  
     - From "Structured Output Parser" node (outputParser)  
   - Configure to parse outputs strictly with the given schema.  

5. **Add "Create a record" node:**  
   - Type: Airtable  
   - Configure Airtable credentials (Personal Access Token).  
   - Select base "AI Lead Qualifier" and table "Leads".  
   - Map fields:  
     - Name, Email, Phone (set "NA"), Stage ("New"), Source ("Website"), AI Notes, Priority, Timeline, Use Case, Lead Score, Assigned To ("Salesperson 1"), Raw Message, Business Type, Budget (Normalized)  
     - Use expressions to pull values from form submission and AI Agent output accordingly.  
   - Enable typecasting.  

6. **Add "Quality Leads Based On Score" node (If):**  
   - Type: If  
   - Condition: Check if `lead_score` from AI Agent output is greater than or equal to 7.  
   - Connect input from "Create a record" node.  

7. **Add "Send a message1" node (Gmail Send Email):**  
   - Type: Gmail  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient to sales team email (growthriseagency@gmail.com).  
   - Subject: Use expression to include lead score and lead name dynamically.  
   - Message: Include detailed lead info and AI qualification fields formatted in plain text.  
   - Connect input from true branch of "Quality Leads Based On Score" node.  

8. **Optional: Add Slack and WhatsApp message nodes:**  
   - Add Slack and WhatsApp nodes for notifications (disabled by default).  
   - Configure credentials and webhook URLs if used.  

9. **Add "Gmail Trigger" node:**  
   - Type: Gmail Trigger  
   - Configure Gmail OAuth2 credentials.  
   - Poll interval: Every minute.  

10. **Add "If" node to filter Gmail Trigger emails:**  
    - Type: If  
    - Condition: Email subject contains "🔥 New High-Priority Lead".  
    - Connect input from "Gmail Trigger" node.  

11. **Add "OpenAI Chat Model1" node:**  
    - Duplicate of OpenAI Chat Model, or use a new node with same GPT model and credentials.  

12. **Add "Basic LLM Chain" node:**  
    - Type: LangChain Chain LLM  
    - Prompt: Instruct AI to write a concise, friendly reply email under 180 words, acknowledging the lead inquiry, explaining help, asking clarifying questions, and offering a booking link. Sign off as "Growth Rise Agency".  
    - Connect languageModel input from "OpenAI Chat Model1".  
    - Connect input from true branch of "If" node filtering Gmail Trigger emails.  

13. **Add "Create a draft" node (Gmail):**  
    - Type: Gmail  
    - Configure Gmail OAuth2 credentials.  
    - Subject: Prefix "Re: " to original email subject dynamically.  
    - Message: Use output from "Basic LLM Chain" as draft body.  
    - Connect input from "Basic LLM Chain" output.  

14. **Connect all nodes as per logical flow:**  
    - Form submission → AI Agent → Create Airtable record → Quality Leads If node → Email Notification  
    - Gmail Trigger → If (filter) → Basic LLM Chain → Create Draft  

15. **Add Sticky Notes:**  
    - Add notes for documentation at each logical block describing purpose and usage, replicating contents as in the original workflow for clarity.  

16. **Test the workflow:**  
    - Perform test form submissions.  
    - Validate AI qualification outputs.  
    - Confirm Airtable records creation.  
    - Check email notifications and draft creation.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow uses OpenAI GPT-4.1-mini model for both lead qualification and reply generation, requiring appropriate API keys. | OpenAI API documentation: https://platform.openai.com/docs/api-reference/introduction                      |
| Airtable Personal Access Token must have write access to the specified base and table.                                         | Airtable API docs: https://airtable.com/api                                                               |
| Gmail OAuth2 credentials require enabling Gmail API and proper scopes for reading, sending, and creating drafts.               | Gmail API: https://developers.google.com/gmail/api                                                          |
| Disabled Slack and WhatsApp notification nodes indicate optional multi-channel alerting capabilities.                         | Slack API: https://api.slack.com/ , WhatsApp Business API: https://developers.facebook.com/docs/whatsapp    |
| The AI prompt enforces strict JSON schema adherence to avoid parser errors and ensure data integrity in CRM.                  | JSON Schema standards: https://json-schema.org/                                                             |
| The workflow balances automation speed and control by generating drafts instead of sending auto-replies directly.             | Allows manual review and edits before sending emails.                                                       |

---

**Disclaimer:** The provided text is based exclusively on an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.