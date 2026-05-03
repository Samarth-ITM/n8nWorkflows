Generate personalized HubSpot campaign emails with Groq AI and Gmail

https://n8nworkflows.xyz/workflows/generate-personalized-hubspot-campaign-emails-with-groq-ai-and-gmail-14147


# Generate personalized HubSpot campaign emails with Groq AI and Gmail

# 1. Workflow Overview

This workflow automatically finds newly created HubSpot contacts, generates a personalized marketing campaign strategy for each contact using Groq AI, formats the generated text, and emails the result through Gmail.

Its primary use case is outbound marketing or lead nurturing: when new contacts enter HubSpot, the workflow can proactively send a customized campaign idea based on the company associated with each contact.

## 1.1 Scheduled Contact Retrieval
The workflow starts on a schedule, then queries HubSpot for contacts created within the last 24 hours.

## 1.2 Per-Contact Processing Loop
The retrieved contacts are intended to be processed one by one using a loop node so each contact can receive an individualized AI-generated output.

## 1.3 AI Campaign Generation
For each contact, an AI Agent is expected to use a Groq chat model to generate a marketing campaign strategy tailored to the contact’s company.

## 1.4 Output Formatting and Email Delivery
The AI text is converted into email-friendly HTML formatting and sent to the contact via Gmail with a personalized subject line.

## 1.5 Important Structural Observation
Although the workflow clearly includes an AI Agent and a Groq Chat Model, the JSON connections do not currently wire those nodes into the active execution path. As provided, the connected path is:

**Schedule Trigger → Search contacts → Loop Over Contacts**

and separately:

**Format AI's output → Send a message → Loop Over Contacts**

This means the workflow is structurally incomplete in its current JSON form and would not execute end-to-end as intended without adding the missing connections between the loop, AI generation, formatting, and email sending.

---

# 2. Block-by-Block Analysis

## 2.1 Workflow Documentation and In-Canvas Guidance

**Overview:**  
This block contains sticky notes that explain the workflow purpose, setup steps, and high-level behavior. These notes do not execute anything but are useful for human operators rebuilding or maintaining the workflow.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2

### Node Details

#### Sticky Note
- **Type and technical role:** Sticky Note; documentation-only canvas annotation.
- **Configuration choices:** Contains a full explanation of the workflow purpose, setup steps, and customization ideas.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Standard sticky note behavior; no special version dependency.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** Sticky Note; documentation-only annotation.
- **Configuration choices:** Labels the HubSpot retrieval section as “Fetch New Contacts”.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** Sticky Note; documentation-only annotation.
- **Configuration choices:** Labels the AI generation and email sending section.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Scheduled Retrieval of New HubSpot Contacts

**Overview:**  
This block starts the workflow automatically and searches HubSpot for contacts created in the previous 24 hours. It acts as the intake stage for all downstream processing.

**Nodes Involved:**  
- Schedule Trigger
- Search contacts

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Uses an interval-based schedule rule. The JSON shows `interval:[{}]`, which means the trigger exists but the exact interval granularity is not explicitly meaningful from the exported configuration alone. In practice, this should be reviewed in the editor after import.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - **Output:** Search contacts
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Misconfigured schedule causing the workflow to run too often or not at all  
  - Timezone misunderstandings when comparing HubSpot date fields downstream
- **Sub-workflow reference:** None.

#### Search contacts
- **Type and technical role:** `n8n-nodes-base.hubspot`; queries HubSpot CRM contacts.
- **Configuration choices:**  
  - Operation: `search`  
  - Authentication: app token  
  - Filter: contacts where `createdate|datetime` is greater than or equal to the ISO timestamp for “now minus 24 hours”
- **Key expressions or variables used:**  
  - `={{ new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString() }}`
- **Input and output connections:**  
  - **Input:** Schedule Trigger  
  - **Output:** Loop Over Contacts
- **Version-specific requirements:** Type version `2.2`; requires HubSpot credentials configured as app token authentication.
- **Edge cases or potential failure types:**  
  - Invalid or expired HubSpot private app token  
  - HubSpot API limits or temporary API unavailability  
  - Contacts missing expected properties like `email`, `company`, or website-related fields  
  - Filtering only by `createdate` may include contacts that are not ready for outreach
- **Sub-workflow reference:** None.

---

## 2.3 Per-Contact Iteration

**Overview:**  
This block is designed to process HubSpot contacts one at a time. The node is essential because downstream expressions reference the current loop item directly.

**Nodes Involved:**  
- Loop Over Contacts

### Node Details

#### Loop Over Contacts
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through incoming items in batches.
- **Configuration choices:**  
  - Uses default batch behavior with `reset: false`  
  - No explicit batch size is shown, so the effective default should be verified in n8n during rebuild
- **Key expressions or variables used:** Downstream nodes reference this node using:
  - `$('Loop Over Contacts').item.json.properties.email.value`
  - `$('Loop Over Contacts').item.json.properties.company.value`
- **Input and output connections:**  
  - **Input:** Search contacts  
  - **Intended output:** Should connect to AI Agent, then continue to formatting and email delivery  
  - **Actual connected output in JSON:** none to the AI path  
  - **Loop-back input currently present:** Send a message routes back to this node, which is the standard pattern for continuing batches
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - If no contacts are returned, nothing else runs  
  - Missing email property will break or invalidate the Gmail send step  
  - Missing company property will affect subject line personalization  
  - If the loop is not connected to a processing chain, the workflow stops after batch emission
- **Sub-workflow reference:** None.

---

## 2.4 AI Campaign Generation

**Overview:**  
This block is intended to generate a personalized campaign strategy for each contact using a Groq-hosted language model. However, the nodes exist without execution-path connections, so this part is currently inactive in the supplied JSON.

**Nodes Involved:**  
- AI Agent
- Groq Chat Model

### Node Details

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates prompt handling and LLM interaction.
- **Configuration choices:** No parameters are configured in the export. That means no visible system prompt, user prompt, tools, memory, or structured output settings are defined here.
- **Key expressions or variables used:** None present in the exported parameters.
- **Input and output connections:**  
  - **Input:** None connected in JSON  
  - **Expected input:** Loop Over Contacts  
  - **Expected output:** Format AI's output  
  - **Model dependency:** Should be connected to Groq Chat Model through the dedicated AI/model port in n8n
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain-enabled n8n version and proper model connection.
- **Edge cases or potential failure types:**  
  - No prompt configured, resulting in unusable or generic output  
  - Node not connected, so it never executes  
  - Token/context overflow if very large website or company inputs are later added  
  - Model credential/authentication failures  
  - AI output may not match the formatting assumptions in the Code node
- **Sub-workflow reference:** None.

#### Groq Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`; language model provider node for the AI Agent.
- **Configuration choices:**  
  - Model: `llama-3.3-70b-versatile`  
  - No additional options configured
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - **Input/Output:** Not connected in the JSON  
  - **Expected role:** Attached to AI Agent as its chat model
- **Version-specific requirements:** Type version `1`; requires Groq API credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - Invalid Groq credentials  
  - Model name availability changes over time  
  - Rate limiting or service errors  
  - No connection to AI Agent means the model is never used
- **Sub-workflow reference:** None.

---

## 2.5 AI Output Transformation

**Overview:**  
This block converts AI output text into a format suitable for HTML email display. It expects the AI Agent to return text in a field called `output`.

**Nodes Involved:**  
- Format AI's output

### Node Details

#### Format AI's output
- **Type and technical role:** `n8n-nodes-base.code`; transforms LLM output programmatically.
- **Configuration choices:** JavaScript code:
  - Reads the first input item from `$input.first().json.output`
  - Replaces escaped newline sequences (`\\n`) with `<br><br>`
  - Returns one item with a `campaign` field
- **Key expressions or variables used:**  
  - `$input.first().json.output`
- **Input and output connections:**  
  - **Input:** None connected in JSON  
  - **Expected input:** AI Agent  
  - **Output:** Send a message
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If `output` is missing, the code throws an error  
  - If AI returns actual newlines instead of escaped `\\n`, the replacement may not produce the intended formatting  
  - If multiple items arrive, this code only processes the first item
- **Sub-workflow reference:** None.

---

## 2.6 Personalized Email Delivery

**Overview:**  
This block sends an HTML email through Gmail using the contact’s email address and company name from the current loop item. It closes the loop by routing execution back to the batch node.

**Nodes Involved:**  
- Send a message

### Node Details

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`; sends outbound email.
- **Configuration choices:**  
  - Recipient is taken from the current item in `Loop Over Contacts`  
  - Subject includes the contact’s company name  
  - Message body is full HTML and injects the formatted campaign text via `{{$json.campaign}}`
- **Key expressions or variables used:**  
  - `={{ $('Loop Over Contacts').item.json.properties.email.value }}`
  - `=Your Personalized Marketing Campaign for {{ $('Loop Over Contacts').item.json.properties.company.value }}`
  - `{{$json.campaign}}`
- **Input and output connections:**  
  - **Input:** Format AI's output  
  - **Output:** Loop Over Contacts
- **Version-specific requirements:** Type version `2.2`; requires Gmail credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - Gmail OAuth credential failure or revoked access  
  - Missing `properties.email.value` causing send failure  
  - Missing `properties.company.value` causing a broken or blank subject line  
  - HTML formatting can render differently across email clients  
  - Sending limits or anti-spam controls in Gmail/Google Workspace
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow on a recurring schedule |  | Search contacts | # Hyper-Personalized Ad Campaign Generator<br>This workflow automatically generates AI-powered advertising campaign strategies for newly created CRM contacts and sends them via email.<br>## How it works<br>The workflow runs on a schedule and retrieves contacts created in the last 24 hours from the CRM. Each contact is processed individually through a loop. Company details such as name and website are sent to an AI agent, which analyzes the business and generates a structured advertising campaign strategy. The output is formatted into a readable format and delivered via email to the contact.<br>## Setup steps<br>1. Connect your HubSpot credentials.<br>2. Configure your AI model (Groq) credentials.<br>3. Connect your Gmail or SMTP account.<br>4. Review and customize the AI prompt if needed.<br>5. Activate the workflow.<br>## Customization tips<br>- Adjust the schedule timing based on your needs.<br>- Modify the AI prompt for different industries or campaign styles.<br>- Customize the email template for branding or personalization.<br>## Fetch New Contacts<br>Runs on schedule and retrieves contacts created in the last 24 hours from HubSpot. |
| Search contacts | n8n-nodes-base.hubspot | Searches HubSpot for newly created contacts | Schedule Trigger | Loop Over Contacts | # Hyper-Personalized Ad Campaign Generator<br>This workflow automatically generates AI-powered advertising campaign strategies for newly created CRM contacts and sends them via email.<br>## How it works<br>The workflow runs on a schedule and retrieves contacts created in the last 24 hours from the CRM. Each contact is processed individually through a loop. Company details such as name and website are sent to an AI agent, which analyzes the business and generates a structured advertising campaign strategy. The output is formatted into a readable format and delivered via email to the contact.<br>## Setup steps<br>1. Connect your HubSpot credentials.<br>2. Configure your AI model (Groq) credentials.<br>3. Connect your Gmail or SMTP account.<br>4. Review and customize the AI prompt if needed.<br>5. Activate the workflow.<br>## Customization tips<br>- Adjust the schedule timing based on your needs.<br>- Modify the AI prompt for different industries or campaign styles.<br>- Customize the email template for branding or personalization.<br>## Fetch New Contacts<br>Runs on schedule and retrieves contacts created in the last 24 hours from HubSpot. |
| Loop Over Contacts | n8n-nodes-base.splitInBatches | Processes contacts one at a time and supports loop continuation | Search contacts, Send a message |  | # Hyper-Personalized Ad Campaign Generator<br>This workflow automatically generates AI-powered advertising campaign strategies for newly created CRM contacts and sends them via email.<br>## How it works<br>The workflow runs on a schedule and retrieves contacts created in the last 24 hours from the CRM. Each contact is processed individually through a loop. Company details such as name and website are sent to an AI agent, which analyzes the business and generates a structured advertising campaign strategy. The output is formatted into a readable format and delivered via email to the contact.<br>## Setup steps<br>1. Connect your HubSpot credentials.<br>2. Configure your AI model (Groq) credentials.<br>3. Connect your Gmail or SMTP account.<br>4. Review and customize the AI prompt if needed.<br>5. Activate the workflow.<br>## Customization tips<br>- Adjust the schedule timing based on your needs.<br>- Modify the AI prompt for different industries or campaign styles.<br>- Customize the email template for branding or personalization.<br>## Generate Campaign & Send Email<br>AI creates campaign strategy → format output → send personalized email. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Intended to generate a personalized campaign strategy using an LLM |  |  | # Hyper-Personalized Ad Campaign Generator<br>This workflow automatically generates AI-powered advertising campaign strategies for newly created CRM contacts and sends them via email.<br>## How it works<br>The workflow runs on a schedule and retrieves contacts created in the last 24 hours from the CRM. Each contact is processed individually through a loop. Company details such as name and website are sent to an AI agent, which analyzes the business and generates a structured advertising campaign strategy. The output is formatted into a readable format and delivered via email to the contact.<br>## Setup steps<br>1. Connect your HubSpot credentials.<br>2. Configure your AI model (Groq) credentials.<br>3. Connect your Gmail or SMTP account.<br>4. Review and customize the AI prompt if needed.<br>5. Activate the workflow.<br>## Customization tips<br>- Adjust the schedule timing based on your needs.<br>- Modify the AI prompt for different industries or campaign styles.<br>- Customize the email template for branding or personalization.<br>## Generate Campaign & Send Email<br>AI creates campaign strategy → format output → send personalized email. |
| Groq Chat Model | @n8n/n8n-nodes-langchain.lmChatGroq | Provides the Groq-hosted LLM for the AI Agent |  |  | # Hyper-Personalized Ad Campaign Generator<br>This workflow automatically generates AI-powered advertising campaign strategies for newly created CRM contacts and sends them via email.<br>## How it works<br>The workflow runs on a schedule and retrieves contacts created in the last 24 hours from the CRM. Each contact is processed individually through a loop. Company details such as name and website are sent to an AI agent, which analyzes the business and generates a structured advertising campaign strategy. The output is formatted into a readable format and delivered via email to the contact.<br>## Setup steps<br>1. Connect your HubSpot credentials.<br>2. Configure your AI model (Groq) credentials.<br>3. Connect your Gmail or SMTP account.<br>4. Review and customize the AI prompt if needed.<br>5. Activate the workflow.<br>## Customization tips<br>- Adjust the schedule timing based on your needs.<br>- Modify the AI prompt for different industries or campaign styles.<br>- Customize the email template for branding or personalization.<br>## Generate Campaign & Send Email<br>AI creates campaign strategy → format output → send personalized email. |
| Format AI's output | n8n-nodes-base.code | Converts AI response text into HTML-friendly campaign content |  | Send a message | # Hyper-Personalized Ad Campaign Generator<br>This workflow automatically generates AI-powered advertising campaign strategies for newly created CRM contacts and sends them via email.<br>## How it works<br>The workflow runs on a schedule and retrieves contacts created in the last 24 hours from the CRM. Each contact is processed individually through a loop. Company details such as name and website are sent to an AI agent, which analyzes the business and generates a structured advertising campaign strategy. The output is formatted into a readable format and delivered via email to the contact.<br>## Setup steps<br>1. Connect your HubSpot credentials.<br>2. Configure your AI model (Groq) credentials.<br>3. Connect your Gmail or SMTP account.<br>4. Review and customize the AI prompt if needed.<br>5. Activate the workflow.<br>## Customization tips<br>- Adjust the schedule timing based on your needs.<br>- Modify the AI prompt for different industries or campaign styles.<br>- Customize the email template for branding or personalization.<br>## Generate Campaign & Send Email<br>AI creates campaign strategy → format output → send personalized email. |
| Send a message | n8n-nodes-base.gmail | Sends the personalized HTML email through Gmail | Format AI's output | Loop Over Contacts | # Hyper-Personalized Ad Campaign Generator<br>This workflow automatically generates AI-powered advertising campaign strategies for newly created CRM contacts and sends them via email.<br>## How it works<br>The workflow runs on a schedule and retrieves contacts created in the last 24 hours from the CRM. Each contact is processed individually through a loop. Company details such as name and website are sent to an AI agent, which analyzes the business and generates a structured advertising campaign strategy. The output is formatted into a readable format and delivered via email to the contact.<br>## Setup steps<br>1. Connect your HubSpot credentials.<br>2. Configure your AI model (Groq) credentials.<br>3. Connect your Gmail or SMTP account.<br>4. Review and customize the AI prompt if needed.<br>5. Activate the workflow.<br>## Customization tips<br>- Adjust the schedule timing based on your needs.<br>- Modify the AI prompt for different industries or campaign styles.<br>- Customize the email template for branding or personalization.<br>## Generate Campaign & Send Email<br>AI creates campaign strategy → format output → send personalized email. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation and setup notes |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas label for contact retrieval block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas label for AI and email block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is the practical rebuild sequence. Because the exported workflow has missing connections, this section describes how to recreate the workflow so it works as intended.

1. **Create a new workflow**
   - Name it: **Generate personalized HubSpot campaign emails with Groq AI and Gmail**.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Configure it to run at your preferred interval, such as:
     - every day
     - every hour
     - or another interval suitable for your lead volume
   - Make sure the timezone is appropriate for your business process.

3. **Add a HubSpot node named “Search contacts”**
   - Node type: **HubSpot**
   - Operation: **Search**
   - Resource: contacts, if resource selection is exposed in your n8n version
   - Authentication: **App Token**
   - Create or select HubSpot credentials:
     - use a HubSpot private app token with CRM read permissions for contacts
   - Add a filter group:
     - Property: `createdate|datetime`
     - Operator: `GTE`
     - Value expression:
       ```javascript
       {{ new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString() }}
       ```
   - Connect:
     - **Schedule Trigger → Search contacts**

4. **Add a Split In Batches node named “Loop Over Contacts”**
   - Node type: **Loop Over Items / Split In Batches**
   - Keep batch handling simple, typically one contact per iteration
   - Leave reset as `false`
   - Connect:
     - **Search contacts → Loop Over Contacts**

5. **Add an AI Agent node named “AI Agent”**
   - Node type: **AI Agent**
   - This node is present in the export but not configured. You must define the prompt.
   - Recommended prompt design:
     - Instruct the model to generate a concise personalized advertising campaign strategy
     - Use contact/company variables from the HubSpot item
     - If your HubSpot records include website or company metadata, inject them here
   - Example prompt content:
     - Company name: `{{ $('Loop Over Contacts').item.json.properties.company.value }}`
     - Contact email: `{{ $('Loop Over Contacts').item.json.properties.email.value }}`
     - If available, include website/domain property from HubSpot
   - Ask for output in clean plain text with sections such as:
     - target audience
     - ad channels
     - messaging angle
     - campaign objective
     - next steps
   - Connect:
     - **Loop Over Contacts → AI Agent**

6. **Add a Groq Chat Model node named “Groq Chat Model”**
   - Node type: **Groq Chat Model**
   - Create/select Groq credentials using your API key
   - Set model to:
     - `llama-3.3-70b-versatile`
   - Keep default options unless you want to tune temperature or response style
   - Connect the model to the **AI Agent** using the model/AI connection port, not the standard main flow line.

7. **Add a Code node named “Format AI's output”**
   - Node type: **Code**
   - Language: JavaScript
   - Paste this logic:
     ```javascript
     let text = $input.first().json.output;

     text = text.replace(/\\n/g, "<br><br>");

     return [
       {
         json: {
           campaign: text
         }
       }
     ];
     ```
   - Connect:
     - **AI Agent → Format AI's output**

8. **Add a Gmail node named “Send a message”**
   - Node type: **Gmail**
   - Operation: **Send**
   - Configure Gmail OAuth2 credentials
     - Use a Gmail or Google Workspace account authorized to send campaign-style emails
   - Set **To**:
     ```javascript
     {{ $('Loop Over Contacts').item.json.properties.email.value }}
     ```
   - Set **Subject**:
     ```javascript
     Your Personalized Marketing Campaign for {{ $('Loop Over Contacts').item.json.properties.company.value }}
     ```
   - Set the message body as HTML. Use the exported template structure:
     - Greeting
     - explanation paragraph
     - campaign content block with `{{$json.campaign}}`
     - closing section
   - Make sure the message field is treated as HTML in your node version.
   - Connect:
     - **Format AI's output → Send a message**

9. **Close the batch loop**
   - Connect:
     - **Send a message → Loop Over Contacts**
   - This allows the workflow to proceed to the next contact in the batch sequence.

10. **Optionally add safety checks before the AI or email step**
    - Recommended:
      - IF node to verify email exists
      - IF node to verify company exists
      - fallback values for missing company names
    - This is not present in the JSON but is strongly advised.

11. **Add sticky notes if you want the same visual structure**
    - Add one overall note describing the workflow purpose and setup steps
    - Add one note over the HubSpot retrieval area
    - Add one note over the AI/email area

12. **Test the HubSpot search**
    - Run the workflow manually from the Schedule Trigger or execute the Search contacts node
    - Confirm returned items contain:
      - `properties.email.value`
      - `properties.company.value`
    - If those exact property paths differ in your HubSpot node output, update downstream expressions accordingly.

13. **Test one loop iteration**
    - Ensure the AI Agent receives one contact at a time
    - Confirm it returns an `output` field
    - If the field name differs, update the Code node.

14. **Test the email formatting**
    - Verify the campaign text appears correctly in HTML
    - If the model returns actual line breaks rather than escaped `\n`, modify the code to also replace real newline characters:
      ```javascript
      text = text.replace(/\\n/g, "<br><br>").replace(/\n/g, "<br><br>");
      ```

15. **Activate the workflow**
    - Once credentials, prompts, and field mappings are validated, activate the workflow.

## Required Credentials

### HubSpot
- Use a **private app token**
- Must have permission to search/read contacts

### Groq
- Use a valid **Groq API key**
- Ensure your n8n instance supports the Groq Chat Model node

### Gmail
- Configure **OAuth2**
- Confirm the account is allowed to send to external recipients
- Watch for send quotas and compliance constraints

## Input/Output Expectations

### HubSpot Search Output
Expected item fields are assumed to resemble:
- `properties.email.value`
- `properties.company.value`

If your account returns a flatter structure such as `properties.email` instead, you must update all expressions.

### AI Agent Output
Expected output:
- `json.output` containing the generated campaign text

### Code Node Output
Returns:
- `json.campaign`

### Gmail Input
Consumes:
- `json.campaign`
- values from `Loop Over Contacts` for email and company

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Hyper-Personalized Ad Campaign Generator: This workflow automatically generates AI-powered advertising campaign strategies for newly created CRM contacts and sends them via email. | Workflow purpose |
| The workflow runs on a schedule and retrieves contacts created in the last 24 hours from the CRM. Each contact is processed individually through a loop. Company details such as name and website are sent to an AI agent, which analyzes the business and generates a structured advertising campaign strategy. The output is formatted into a readable format and delivered via email to the contact. | High-level behavior |
| Setup steps: Connect HubSpot credentials, configure Groq credentials, connect Gmail or SMTP account, review/customize the AI prompt, activate the workflow. | Build and deployment notes |
| Customization tips: Adjust schedule timing, modify the AI prompt for different industries or campaign styles, customize the email template for branding or personalization. | Adaptation guidance |
| Fetch New Contacts: Runs on schedule and retrieves contacts created in the last 24 hours from HubSpot. | Block note |
| Generate Campaign & Send Email: AI creates campaign strategy → format output → send personalized email. | Block note |

## Final Technical Assessment
The workflow concept is clear and useful, but the provided JSON is **not fully wired**. To function correctly, you must connect:

- **Loop Over Contacts → AI Agent**
- **Groq Chat Model → AI Agent** via the model connection
- **AI Agent → Format AI's output**

Without these links, no AI content is produced and no email can be sent from the contact-processing branch.