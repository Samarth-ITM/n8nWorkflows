Enrich Monday.com Leads & Draft Personalized Emails with Explorium MCP and GPT-4.1

https://n8nworkflows.xyz/workflows/enrich-monday-com-leads---draft-personalized-emails-with-explorium-mcp-and-gpt-4-1-7748


# Enrich Monday.com Leads & Draft Personalized Emails with Explorium MCP and GPT-4.1

### 1. Workflow Overview

This workflow titled **"Enrich Monday.com Leads & Draft Personalized Emails with Explorium MCP and GPT-4.1"** is designed to automate lead enrichment and personalized email drafting for inbound leads added to a Monday.com board. It is targeted at sales and marketing teams aiming to enhance lead data with AI-driven company research and generate tailored outreach emails, improving engagement and efficiency.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Data Extraction**: Receives new lead data from Monday.com via webhook, fetches detailed item data, and formats key lead fields.
- **1.2 Company Research Block**: Uses Explorium MCP and GPT-4.1 to perform deep company analysis based on lead information.
- **1.3 Personalized Email Drafting**: Generates a personalized inbound email using the company research and lead details.
- **1.4 CRM Enrichment**: Produces a concise CRM summary with AI solution suggestions that updates Monday.com.
- **1.5 Email Drafting and Monday.com Updates**: Creates an email draft in Gmail and posts updates to Monday.com.
- **1.6 Error Handling**: Triggers error notifications by email to the administrator if failures occur.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Extraction

**Overview:**  
This block listens for new lead additions via Monday.com webhook, retrieves the full lead item details, and extracts relevant fields for later processing.

**Nodes Involved:**  
- Webhook  
- Respond to Webhook  
- Get item (Monday.com)  
- Edit Fields (Set)

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook node  
  - Role: Entry point; receives HTTP POST requests from Monday.com when a new lead is added.  
  - Config: Path set to a unique webhook ID, listens for POST requests, responds using the "Respond to Webhook" node.  
  - Inputs: External HTTP POST from Monday.com  
  - Outputs: To Respond to Webhook node  
  - Edge cases: Webhook URL mismatch, missing payload, HTTP errors.

- **Respond to Webhook**  
  - Type: Respond to Webhook node  
  - Role: Sends back a JSON response to Monday.com acknowledging receipt.  
  - Config: Responds with a JSON object containing the "challenge" from the incoming request body (for Monday.com webhook validation).  
  - Inputs: From Webhook node  
  - Outputs: To Get item node  
  - Edge cases: Response failures could break webhook handshake.

- **Get item (Monday.com)**  
  - Type: Monday.com node  
  - Role: Fetches lead item details from Monday.com board based on the pulse (item) name provided in the webhook event.  
  - Config: Board ID and column for search specified; fetches item matching the incoming pulse name.  
  - Inputs: From Respond to Webhook  
  - Outputs: To Edit Fields node  
  - Edge cases: Item not found, API errors, rate limits.

- **Edit Fields (Set node)**  
  - Type: Set node  
  - Role: Extracts and restructures key fields from the Monday.com item JSON into standardized fields (company_name, contact_name, email_address, comments).  
  - Config: Uses JSON expressions to map fields from the item data columns.  
  - Inputs: From Get item  
  - Outputs: To Company Researcher node  
  - Edge cases: Missing or malformed data in expected columns.

---

#### 1.2 Company Research Block

**Overview:**  
Performs in-depth company and contact research using Explorium MCP APIs enhanced by GPT-4.1 to generate a structured research brief that informs email and CRM enrichment.

**Nodes Involved:**  
- Explorium MCP (MCP Client Tool)  
- ChatGPT 4.1 - 1 (Language Model)  
- Think - 1 (Think tool)  
- Company Researcher (Langchain agent)

**Node Details:**

- **Explorium MCP**  
  - Type: MCP Client Tool node  
  - Role: Connects to Explorium's Market Context Platform to fetch enriched company intelligence data.  
  - Config: SSE endpoint specified, uses header authentication with Explorium API key credential.  
  - Inputs: None directly, wired internally via Think tool to Company Researcher  
  - Outputs: To Company Researcher agent  
  - Edge cases: Authentication failures, API timeouts, SSE connection drops.

- **ChatGPT 4.1 - 1**  
  - Type: Langchain OpenAI Chat node  
  - Role: Provides GPT-4.1 language model capabilities to the Company Researcher agent.  
  - Config: Model set to "gpt-4.1", uses OpenAI credentials.  
  - Inputs: From Think - 1 tool  
  - Outputs: To Company Researcher agent  
  - Edge cases: API rate limits, token limits, network issues.

- **Think - 1**  
  - Type: Langchain Think tool  
  - Role: Facilitates internal reasoning or intermediate processing for Company Researcher.  
  - Inputs: Upstream data from Edit Fields, triggers Company Researcher node.  
  - Outputs: To Company Researcher agent  
  - Edge cases: Logic errors, expression failures.

- **Company Researcher**  
  - Type: Langchain Agent node  
  - Role: Performs analytical synthesis of input company/contact data using Explorium data and GPT-4.1 to produce a detailed research brief.  
  - Config:  
    - Input prompt includes company name, contact name, email, and salesperson comments.  
    - System message instructs detailed, structured analytical output focusing on company activities, business events, automation relevance, and sales context.  
    - Max iterations capped at 12 to control reasoning depth.  
  - Inputs: From Edit Fields and upstream Think, Explorium MCP, and ChatGPT 4.1 - 1 nodes.  
  - Outputs: To Email Writer and CRM Enrichment nodes.  
  - Edge cases: Model hallucination, incomplete data, API failures.

---

#### 1.3 Personalized Email Drafting

**Overview:**  
Generates a personalized inbound email draft using the research brief and sales context, emphasizing relevance and a call to action.

**Nodes Involved:**  
- Email Writer (Langchain Agent)  
- ChatGPT 4.1 - 2 (Language Model)  
- Think - 2 (Think tool)  
- Structured Output Parser - 2

**Node Details:**

- **Email Writer**  
  - Type: Langchain Agent node  
  - Role: Crafts a short, personalized inbound email (under 120 words) targeting marketing/growth/data contacts at the lead company.  
  - Config:  
    - Input prompt uses company research output, contact full name, and salesperson comments.  
    - System message defines B2B inbound email style, referencing 82Labs capabilities and instructs to include a call to action.  
    - Uses "thinking tool" to enhance response quality.  
  - Inputs: Receives data from Company Researcher node, Think - 2 tool, and ChatGPT 4.1 - 2 node for processing.  
  - Outputs: To Create a draft node (Gmail)  
  - Edge cases: Output parsing errors, content quality variability.

- **ChatGPT 4.1 - 2**  
  - Type: Langchain OpenAI Chat node  
  - Role: Provides GPT-4.1 model service for Email Writer agent.  
  - Inputs: From Think - 2 tool  
  - Outputs: To Email Writer  
  - Edge cases: Rate limits, network issues.

- **Think - 2**  
  - Type: Langchain Think tool  
  - Role: Intermediate processing step to enhance email generation logic.  
  - Inputs: From Company Researcher  
  - Outputs: To ChatGPT 4.1 - 2  
  - Edge cases: Tool failures.

- **Structured Output Parser - 2**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses the Email Writer agent’s response into structured JSON with fields like "email_subject" and "email_content" for downstream use.  
  - Inputs: From Email Writer  
  - Outputs: To Create a draft node  
  - Edge cases: Parsing failures if output format deviates.

---

#### 1.4 CRM Enrichment

**Overview:**  
Produces a short CRM summary with AI solution suggestions and updates the Monday.com item with this enrichment.

**Nodes Involved:**  
- CRM Enrichment (Langchain Agent)  
- ChatGPT 4.1 - 3 (Language Model)  
- Think - 3 (Think tool)  
- Structured Output Parser - 3  
- Add an update to an item - 2 (Monday.com)

**Node Details:**

- **CRM Enrichment**  
  - Type: Langchain Agent node  
  - Role: Generates a concise two-sentence summary of the company and contact, with recommendations for AI-powered solutions from 82Labs.  
  - Config: Similar input context as Email Writer, uses thinking tool, instructed to be brief and actionable.  
  - Inputs: From Company Researcher, Think - 3, and ChatGPT 4.1 - 3  
  - Outputs: To Monday.com update node  
  - Edge cases: Output parsing or content relevance.

- **ChatGPT 4.1 - 3**  
  - Type: Langchain OpenAI Chat node  
  - Role: GPT-4.1 model for CRM Enrichment agent.  
  - Inputs: From Think - 3  
  - Outputs: To CRM Enrichment  
  - Edge cases: API limits.

- **Think - 3**  
  - Type: Langchain Think tool  
  - Role: Intermediate processing for CRM Enrichment generation.  
  - Inputs: From Company Researcher  
  - Outputs: To ChatGPT 4.1 - 3  
  - Edge cases: Internal tool failures.

- **Structured Output Parser - 3**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses CRM Enrichment agent output into JSON with "crm_summary" field.  
  - Inputs: From CRM Enrichment  
  - Outputs: To Monday.com update node  
  - Edge cases: Parsing errors.

- **Add an update to an item - 2 (Monday.com)**  
  - Type: Monday.com node  
  - Role: Adds an update to the lead’s Monday.com item board with the CRM Enrichment summary text.  
  - Config: Uses item ID from "Get item" node, posts enrichment text as an update.  
  - Inputs: From Structured Output Parser - 3  
  - Outputs: None (terminal)  
  - Edge cases: API errors, permission issues.

---

#### 1.5 Email Drafting and Monday.com Updates

**Overview:**  
Creates a Gmail draft of the personalized email and adds an update to Monday.com confirming draft creation.

**Nodes Involved:**  
- Create a draft (Gmail)  
- Add an update to an item - 1 (Monday.com)

**Node Details:**

- **Create a draft (Gmail)**  
  - Type: Gmail node  
  - Role: Creates an email draft in Gmail with the personalized email content and subject generated by Email Writer.  
  - Config:  
    - Uses OAuth2 credentials for Gmail access.  
    - Drafts email to the lead’s email address.  
  - Inputs: From Email Writer agent’s structured output  
  - Outputs: To Add an update to an item - 1  
  - Edge cases: OAuth token expiration, quota limits.

- **Add an update to an item - 1 (Monday.com)**  
  - Type: Monday.com node  
  - Role: Posts confirmation update on Monday.com item that draft email was generated.  
  - Config: Uses item ID from "Get item" node.  
  - Inputs: From Create a draft node  
  - Outputs: None (terminal)  
  - Edge cases: API errors.

---

#### 1.6 Error Handling

**Overview:**  
Catches any errors occurring in the workflow and sends an email notification to the administrator.

**Nodes Involved:**  
- Error Trigger  
- Send a message (Gmail)

**Node Details:**

- **Error Trigger**  
  - Type: Error Trigger node  
  - Role: Listens for execution errors anywhere in the workflow to trigger error handling.  
  - Inputs: Monitors all nodes  
  - Outputs: To Send a message node  
  - Edge cases: Failure to detect errors.

- **Send a message (Gmail)**  
  - Type: Gmail node  
  - Role: Sends an email to the admin (elay.g@82labs.io) with error details.  
  - Config: Uses OAuth2 Gmail credentials.  
  - Inputs: From Error Trigger node  
  - Outputs: None (terminal)  
  - Edge cases: Sending failures, credential expiration.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                      | Input Node(s)              | Output Node(s)                     | Sticky Note                                                   |
|----------------------------|----------------------------------|------------------------------------|---------------------------|----------------------------------|---------------------------------------------------------------|
| Webhook                    | Webhook                          | Entry point to receive Monday.com webhook | -                         | Respond to Webhook               | ## Trigger Workflow and Organize Data Using Monday.com webhook |
| Respond to Webhook          | Respond to Webhook               | Responds to Monday.com webhook      | Webhook                   | Get item                        |                                                               |
| Get item                   | Monday.com                      | Retrieves full lead item details    | Respond to Webhook        | Edit Fields                    |                                                               |
| Edit Fields                | Set                             | Extracts key lead data fields       | Get item                  | Company Researcher              |                                                               |
| Company Researcher         | Langchain Agent                 | Performs deep company research      | Edit Fields, Explorium MCP, ChatGPT 4.1 - 1, Think - 1 | Email Writer, CRM Enrichment    | ## Research Agent This agent uses Explorium MCP and ChatGPT 4.1 to research the company and the contact |
| Explorium MCP              | MCP Client Tool                 | Fetches enriched company data       | -                         | Company Researcher             |                                                               |
| ChatGPT 4.1 - 1            | Langchain OpenAI Chat           | Provides GPT-4.1 model for research | Think - 1                 | Company Researcher             |                                                               |
| Think - 1                  | Langchain Think Tool            | Intermediate processing for research | Edit Fields               | ChatGPT 4.1 - 1               |                                                               |
| Email Writer               | Langchain Agent                 | Drafts personalized inbound email  | Company Researcher, Think - 2, ChatGPT 4.1 - 2 | Create a draft                 | ## Draft Email Agent This agent uses ChatGPT 4.1 to draft a personal inbound email |
| Think - 2                  | Langchain Think Tool            | Intermediate step for email drafting | Company Researcher        | ChatGPT 4.1 - 2               |                                                               |
| ChatGPT 4.1 - 2            | Langchain OpenAI Chat           | GPT-4.1 model for email generation  | Think - 2                 | Email Writer                  |                                                               |
| Structured Output Parser - 2 | Langchain Structured Output Parser | Parses email generation output      | Email Writer              | Create a draft                |                                                               |
| Create a draft             | Gmail                          | Creates draft email in Gmail        | Structured Output Parser - 2 | Add an update to an item - 1  |                                                               |
| Add an update to an item - 1 | Monday.com                    | Posts confirmation update on item  | Create a draft            | -                            |                                                               |
| CRM Enrichment             | Langchain Agent                 | Generates CRM summary + AI solution suggestions | Company Researcher, Think - 3, ChatGPT 4.1 - 3 | Add an update to an item - 2  | ## CRM Enrichment Agent This agent uses ChatGPT 4.1 to enrich Monday.com CRM |
| Think - 3                  | Langchain Think Tool            | Intermediate step for CRM enrichment | Company Researcher        | ChatGPT 4.1 - 3              |                                                               |
| ChatGPT 4.1 - 3            | Langchain OpenAI Chat           | GPT-4.1 model for CRM enrichment    | Think - 3                 | CRM Enrichment               |                                                               |
| Structured Output Parser - 3 | Langchain Structured Output Parser | Parses CRM enrichment output        | CRM Enrichment            | Add an update to an item - 2  |                                                               |
| Add an update to an item - 2 | Monday.com                    | Adds CRM enrichment update to item | Structured Output Parser - 3 | -                            |                                                               |
| Error Trigger              | Error Trigger                  | Catches workflow execution errors   | -                         | Send a message               | ## Error Trigger Send an email to Admin                       |
| Send a message             | Gmail                          | Sends error notification to admin   | Error Trigger             | -                            |                                                               |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Webhook to Receive Monday.com Lead Data**  
- Add a **Webhook** node.  
- Set HTTP Method to POST.  
- Set a unique webhook path (e.g., "76e1e0a6-495e-4ab6-ba56-b5746cbc4727").  
- Set Response Mode to "Respond with node."  

**Step 2: Respond to Webhook Node**  
- Add **Respond to Webhook** node connected to Webhook.  
- Configure response body as JSON: `{ "challenge": "{{ $json.body.challenge }}" }`.  

**Step 3: Get Item from Monday.com**  
- Add **Monday.com** node ("Get item").  
- Operation: Get By Column Value.  
- Board ID: Enter your Monday.com board ID with leads.  
- Column ID: Use the "name" column or appropriate column to match pulseName from webhook payload.  
- Column Value: `={{ $json.body.event.pulseName }}` (expression referencing webhook data).  
- Connect from Respond to Webhook node.  
- Add Monday.com API credentials.  

**Step 4: Edit Fields Node**  
- Add **Set** node ("Edit Fields").  
- Mode: Raw.  
- JSON Output: Map fields from Monday.com item as:  
  ```json
  {
    "company_name": "{{ $json.name }}",
    "contact_name": "{{ $json.column_values[5].text }}",
    "email_address": "{{ $json.column_values[6].text }}",
    "comments": "{{ $json.column_values[10].text }}"
  }
  ```  
- Connect from Get item node.  

**Step 5: Explorium MCP Node**  
- Add **MCP Client Tool** node ("Explorium MCP").  
- SSE Endpoint: `https://mcp.explorium.ai/sse`.  
- Authentication: Header Auth with Explorium API key credential.  
- No direct input connections; used internally by Company Researcher.  

**Step 6: Set Up GPT-4.1 Chat Nodes**  
- Add three **Langchain OpenAI Chat** nodes named ChatGPT 4.1 - 1, - 2, and - 3.  
- Set model to "gpt-4.1".  
- Add your OpenAI API credentials.  

**Step 7: Add Think Tools**  
- Add three **Langchain Think** nodes named Think - 1, - 2, - 3 to act as intermediate processors.  

**Step 8: Company Researcher Agent**  
- Add **Langchain Agent** node ("Company Researcher").  
- Prompt: Use input text with company_name, contact_name, email_address, comments from "Edit Fields".  
- System Message: Detailed analytical instructions to produce company research brief (as per original).  
- Max Iterations: 12.  
- Connect inputs: From Edit Fields, Explorium MCP (ai_tool), ChatGPT 4.1 - 1 (ai_languageModel), and Think - 1 (ai_tool).  
- Outputs: Connect to Email Writer and CRM Enrichment nodes.  

**Step 9: Email Writer Agent**  
- Add **Langchain Agent** node ("Email Writer").  
- Text input includes company research output, contact full name, and comments.  
- System Message: Compose personalized inbound email under 120 words referencing 82Labs, with a call to action, addressing contact by first name.  
- Use thinking tool.  
- Connect inputs: From Company Researcher, Think - 2, ChatGPT 4.1 - 2.  
- Add **Structured Output Parser - 2** node after Email Writer to parse email_subject and email_content fields.  

**Step 10: Create Gmail Draft**  
- Add **Gmail** node ("Create a draft").  
- Resource: Draft.  
- Message: From parsed email_content.  
- Subject: From parsed email_subject.  
- Send To: Lead email address from Edit Fields.  
- Add Gmail OAuth2 credentials.  
- Connect from Structured Output Parser - 2.  

**Step 11: Monday.com Update for Drafts**  
- Add **Monday.com** node ("Add an update to an item - 1").  
- Operation: Add Update.  
- Item ID: From Get item node.  
- Value: `"Draft email from AI Agent was generated!"`.  
- Connect from Create a draft node.  

**Step 12: CRM Enrichment Agent**  
- Add **Langchain Agent** node ("CRM Enrichment").  
- Input text similar to Email Writer but tasked to produce a two-sentence CRM summary and AI solution suggestions.  
- Use thinking tool.  
- Connect inputs: Company Researcher, Think - 3, ChatGPT 4.1 - 3.  
- Add **Structured Output Parser - 3** node to parse "crm_summary".  

**Step 13: Monday.com Update for CRM Enrichment**  
- Add **Monday.com** node ("Add an update to an item - 2").  
- Operation: Add Update.  
- Item ID: From Get item.  
- Value: Insert parsed CRM summary text.  
- Connect from Structured Output Parser - 3.  

**Step 14: Error Handling**  
- Add **Error Trigger** node connected globally to catch workflow errors.  
- Add **Gmail** node ("Send a message") connected to Error Trigger.  
- Configure to send error report email to admin (elay.g@82labs.io).  
- Use Gmail OAuth2 credentials.  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                             |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| This workflow was created by [Elay Guez](https://www.linkedin.com/in/elay-g).                                 | Author LinkedIn                                                                             |
| Uses Explorium MCP for deep company intelligence: https://www.explorium.ai/mcp                               | Explorium MCP official website                                                             |
| GPT-4.1 (OpenAI) powers natural language generation nodes.                                                   | OpenAI GPT-4.1 API                                                                         |
| Monday.com board must have columns for company name, contact name, email, and comments with matching indexes. | Monday.com board setup requirement                                                         |
| Gmail OAuth2 credentials required to create drafts and send error emails.                                     | Gmail OAuth2 setup                                                                          |
| Update system messages in all AI agent nodes to customize company context and value proposition.              | Personalization instruction in sticky note3                                                |
| Workflow triggers automatically upon new lead creation in Monday.com.                                        | Automation trigger explanation                                                             |
| Sticky notes provide additional contextual explanations but are disabled by default in the workflow.          | Internal documentation in n8n nodes                                                        |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected content. All manipulated data is legal and public.