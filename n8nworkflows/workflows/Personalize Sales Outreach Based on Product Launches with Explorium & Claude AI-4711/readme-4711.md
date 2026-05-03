Personalize Sales Outreach Based on Product Launches with Explorium & Claude AI

https://n8nworkflows.xyz/workflows/personalize-sales-outreach-based-on-product-launches-with-explorium---claude-ai-4711


# Personalize Sales Outreach Based on Product Launches with Explorium & Claude AI

### 1. Workflow Overview

This workflow automates personalized sales outreach triggered by product launch events or similar business occurrences. It integrates Explorium’s data enrichment platform and Anthropic’s Claude AI language models to research companies, enrich prospect data, and generate tailored outbound emails. The workflow handles scenarios with and without detailed prospect data, delivering relevant, concise, and human-like sales emails to targeted recipients.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives event data via a secure webhook.
- **1.2 Company Research:** Uses Explorium’s MCP and Claude AI to analyze the company event and generate a research summary.
- **1.3 Prospect Data Handling:** Fetches and processes employee/prospect data based on the event’s business ID.
- **1.4 Email Composition:** Generates personalized cold outbound emails, with two paths depending on the availability of prospect data.
- **1.5 Output & Notification:** Sends the generated emails to Slack channels or users for review or further action.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures incoming POST HTTP requests containing product launch or business event data.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook  
    - Role: Entry point for external event triggers via POST requests at path `/webhook-test/product-launch`.  
    - Configuration: Accepts POST method; secure webhook URL exposed publicly.  
    - Input: HTTP POST with JSON payload describing event data (business_id, event details).  
    - Output: Passes event data to company research node.  
    - Edge Cases: Unauthorized or malformed requests; missing required event fields.  
    - Version: n8n version supporting webhook v2.

---

#### 2.2 Company Research

- **Overview:**  
  Analyzes the incoming event data to generate a detailed research brief about the company, event significance, and business context using Explorium MCP and Claude AI.

- **Nodes Involved:**  
  - Company Researcher (LangChain Agent)  
  - Explorium MCP (external AI tool client)  
  - Anthropic llm (Claude AI model)  
  - Sticky Note (Research explanation)

- **Node Details:**

  - **Company Researcher**  
    - Type: LangChain Agent (AI research analyst)  
    - Role: Produces a structured research summary interpreting the business event and company context.  
    - Configuration: Uses prompt detailing output structure with context variables (`business_id`, event data) from webhook payload; max 12 iterations for AI reasoning.  
    - Inputs: Event data from Webhook node.  
    - Outputs: Research summary in natural language for downstream email composition.  
    - Edge Cases: AI timeout, insufficient or ambiguous input data.  
    - Sub-workflow: Integrates with Explorium MCP tool and Anthropic Claude model.

  - **Explorium MCP**  
    - Type: External AI tool client  
    - Role: Provides data enrichment and research capabilities to the Company Researcher node.  
    - Configuration: SSE endpoint with Bearer authentication.  
    - Inputs/Outputs: Connected as AI tool within LangChain.  
    - Edge Cases: Authentication failure, network issues.

  - **Anthropic llm**  
    - Type: Language model node (Claude 3.7 Sonnet)  
    - Role: AI language generation engine supporting Company Researcher analysis.  
    - Credentials: Anthropic API key required.  
    - Edge Cases: API limits, network timeouts.

  - **Sticky Note**  
    - Content: Describes this block as the research phase using Explorium MCP to analyze companies with recent events.

---

#### 2.3 Prospect Data Handling

- **Overview:**  
  Fetches up to 5 relevant employees/prospects associated with the company from Explorium’s API, enriches their contact info, and formats the data for personalized outreach.

- **Nodes Involved:**  
  - Prospect Fetcher (HTTP Request)  
  - Has Prospect data/ Does not (IF node)  
  - Formulate Prospect Data (Code)  
  - Prospect Mapping (Code)  
  - Prospect Enrichment (HTTP Request)  
  - Full Prospect Formulation (Code)  
  - Loop Over Items (Split in batches)  
  - Sticky Note3 (Employee Data explanation)

- **Node Details:**

  - **Prospect Fetcher**  
    - Type: HTTP Request  
    - Role: Queries Explorium API for marketing employees with email at the business ID from event.  
    - Configuration: POST request with filters (job level, department, has email) and pagination set to 5 prospects maximum.  
    - Credentials: Bearer auth and HTTP header auth used.  
    - Input: Business ID from Webhook event.  
    - Output: Array of prospect profiles.  
    - Edge Cases: API errors, empty response, auth failure.

  - **Has Prospect data/ Does not (IF)**  
    - Type: Conditional Split  
    - Role: Checks if prospect data was found (array length > 0) to decide email generation path.  
    - Inputs: Prospect Fetcher output.  
    - Outputs:  
      - True: proceed with personalized prospect emails.  
      - False: fallback to general company email.  
    - Edge Cases: Unexpected data types causing logic errors.

  - **Formulate Prospect Data**  
    - Type: Function Code  
    - Role: Transforms raw prospect array into simplified structured JSON with key fields (name, title, email, skills).  
    - Input: Prospect Fetcher data.  
    - Output: Array of formatted prospect JSON objects.  
    - Edge Cases: Missing fields, empty arrays.

  - **Prospect Mapping**  
    - Type: Function Code  
    - Role: Extracts prospect IDs for bulk enrichment.  
    - Input: Formulate Prospect Data output.  
    - Output: JSON with array of prospect IDs.  
    - Edge Cases: Empty list handling.

  - **Prospect Enrichment**  
    - Type: HTTP Request  
    - Role: Requests detailed contact information for prospect IDs from Explorium API.  
    - Configuration: POST with prospect_ids as JSON body.  
    - Credentials: HTTP header auth.  
    - Output: Enriched contact data.  
    - Edge Cases: API rate limits, partial data.

  - **Full Prospect Formulation**  
    - Type: Function Code  
    - Role: Merges original prospect profiles with enriched emails to select best contact email.  
    - Input: Prospect Fetcher and Prospect Enrichment outputs.  
    - Output: Full prospect profiles with enriched emails.  
    - Edge Cases: Missing emails, inconsistent data.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes each prospect individually for personalized email generation with batch size = 1.  
    - Input: Full Prospect Formulation output.  
    - Output: Single prospect data per iteration.  
    - Edge Cases: Empty inputs, batch reset behavior.

  - **Sticky Note3**  
    - Content: Indicates this block gathers data about 5 employees for email generation.

---

#### 2.4 Email Composition

- **Overview:**  
  Generates personalized cold outbound sales emails based on either detailed prospect data or general company data if no prospects found.

- **Nodes Involved:**  
  - Email Writer (NO prospect data) (LangChain Agent)  
  - Email Writer (YES prospect data) (LangChain Agent)  
  - Anthropic Chat Model1 (Claude AI)  
  - Anthropic llm1 (Claude AI)  
  - Sticky Note1 (Crossroads explanation)  
  - Sticky Note2 (Email Writer #1 explanation)  
  - Sticky Note4 (Email Writer #2 explanation)

- **Node Details:**

  - **Email Writer (NO prospect data)**  
    - Type: LangChain Agent  
    - Role: Writes a concise personalized email to a general marketing/growth/data contact without prospect details.  
    - Input: Company research summary and event data.  
    - Configuration: Prompt instructs to write under 120 words, include relevance to event, Explorium intro, and CTA to set meeting + MCP Playground link.  
    - Output: Generated email text.  
    - Edge Cases: Lack of sufficient company context, AI generation failure.

  - **Email Writer (YES prospect data)**  
    - Type: LangChain Agent  
    - Role: Writes highly personalized cold outbound email under 150 words tailored to each prospect’s role and enriched data.  
    - Input: Prospect data (email, experience, skills), company research, and event data.  
    - Configuration: Prompt requests professional email selection, relevance, human tone, and soft CTA with MCP Playground link.  
    - Output: Generated email text with selected recipient email.  
    - Edge Cases: Missing prospect email, AI errors.

  - **Anthropic Chat Model1 and Anthropic llm1**  
    - Type: Claude AI models used as language engines for the agents.  
    - Credentials: Anthropic API key.  
    - Edge Cases: API limits, network failures.

  - **Sticky Notes**  
    - Sticky Note1: Explains decision fork based on prospect data availability.  
    - Sticky Note2: Describes email writer #1 as generating emails based on company research only.  
    - Sticky Note4: Describes email writer #2 as creating highly personalized emails using both prospect and company data.

---

#### 2.5 Output & Notification

- **Overview:**  
  Sends the generated emails (or email texts) to Slack for internal notification or further processing.

- **Nodes Involved:**  
  - Slack Output (Channel message)  
  - Slack Output1 (Direct user message)

- **Node Details:**

  - **Slack Output**  
    - Type: Slack node  
    - Role: Posts personalized emails generated with prospect data to a specific Slack channel (`evergreen-outbound-campaign`).  
    - Input: Output from Email Writer (YES prospect data).  
    - Credentials: OAuth2 for Slack.  
    - Edge Cases: Slack API rate limit, auth expiration.

  - **Slack Output1**  
    - Type: Slack node  
    - Role: Sends general outbound emails (no prospect data) as direct messages to a specific Slack user (`itamar.levi`).  
    - Input: Output from Email Writer (NO prospect data).  
    - Credentials: OAuth2 for Slack.  
    - Edge Cases: User not found, message send failure.

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                            | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                     |
|---------------------------|-------------------------------------|--------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Webhook                   | HTTP Webhook                        | Receive event data from external source    | -                          | Company Researcher             |                                                                                                |
| Company Researcher        | LangChain Agent                    | Generate company and event research summary| Webhook                    | Prospect Fetcher              | ## Reaserch This agent uses the Explorium MCP to research the company that just had an event   |
| Explorium MCP             | MCP Client Tool                    | Provide AI-powered company data enrichment | -                          | Company Researcher            |                                                                                                |
| Anthropic llm             | Claude AI Language Model           | AI backend for Company Researcher           | -                          | Company Researcher            |                                                                                                |
| Prospect Fetcher          | HTTP Request                      | Fetch relevant employee prospects           | Company Researcher          | Has Prospect data/ Does not   |                                                                                                |
| Has Prospect data/ Does not| IF Condition                      | Branch logic based on prospects availability| Prospect Fetcher            | Email Writer (NO prospect data), Formulate Prospect Data | ## Crossroads if we cant find data about the company's employees, we fork up, to write a general email |
| Formulate Prospect Data   | Code                             | Structure raw prospect data                  | Has Prospect data/ Does not | Prospect Mapping             |                                                                                                |
| Prospect Mapping          | Code                             | Extract prospect IDs for enrichment          | Formulate Prospect Data     | Prospect Enrichment           |                                                                                                |
| Prospect Enrichment       | HTTP Request                     | Enrich prospect contact info                  | Prospect Mapping            | Full Prospect Formulation     |                                                                                                |
| Full Prospect Formulation | Code                             | Merge enriched emails with prospect profiles | Prospect Enrichment         | Loop Over Items              |                                                                                                |
| Loop Over Items           | Split in Batches                 | Process prospects one by one for emails       | Full Prospect Formulation   | Email Writer (YES prospect data) | ##  Employee Data Gather data about the 5 employees we will write emails to.                   |
| Email Writer (NO prospect data)| LangChain Agent               | Generate general sales email without prospect details | Has Prospect data/ Does not | Slack Output1                | ## email writer #1 Given the research, this agents writes an email                            |
| Email Writer (YES prospect data)| LangChain Agent              | Generate personalized email per prospect     | Loop Over Items             | Slack Output                 | ## Email Writer #2 Given both data about the email and the employee, the agent will write a highly personalized email to each employee |
| Anthropic Chat Model1     | Claude AI Language Model          | AI backend for Email Writer (NO prospect data)| -                          | Email Writer (NO prospect data)|                                                                                                |
| Anthropic llm1            | Claude AI Language Model          | AI backend for Email Writer (YES prospect data)| -                          | Email Writer (YES prospect data)|                                                                                                |
| Slack Output              | Slack                            | Send personalized emails to Slack channel    | Email Writer (YES prospect data) | Loop Over Items             |                                                                                                |
| Slack Output1             | Slack                            | Send general emails to Slack user             | Email Writer (NO prospect data) | -                           |                                                                                                |
| Sticky Note               | Sticky Note                      | Research explanation                          | -                          | -                            | ## Reaserch This agent uses the Explorium MCP to research the company that just had an event   |
| Sticky Note1              | Sticky Note                      | Fork explanation                              | -                          | -                            | ## Crossroads if we cant find data about the company's employees, we fork up, to write a general email |
| Sticky Note2              | Sticky Note                      | Email Writer #1 explanation                    | -                          | -                            | ## email writer #1 Given the research, this agents writes an email                            |
| Sticky Note3              | Sticky Note                      | Employee data explanation                      | -                          | -                            | ##  Employee Data Gather data about the 5 employees we will write emails to.                   |
| Sticky Note4              | Sticky Note                      | Email Writer #2 explanation                    | -                          | -                            | ## Email Writer #2 Given both data about the email and the employee, the agent will write a highly personalized email to each employee |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - Set HTTP Method to POST  
   - Set Path to `/webhook-test/product-launch`  
   - No authentication (or secure as per your environment)  

2. **Create Company Researcher Node**  
   - Type: LangChain Agent (AI research analyst)  
   - Text input: Use expression to pass `$json.body.data` from webhook  
   - Configure system prompt with provided detailed instructions interpreting event and company context  
   - Set max iterations to 12  
   - Connect Webhook output to this node’s input  

3. **Add Explorium MCP Node**  
   - Type: MCP Client Tool  
   - Set SSE Endpoint to `mcp.explorium.ai/sse`  
   - Authentication: Bearer Auth (configure with your token)  
   - Connect as AI Tool input to Company Researcher node  

4. **Add Anthropic llm Node**  
   - Type: LangChain Claude AI Model  
   - Select `claude-3-7-sonnet-20250219` model  
   - Provide Anthropic API credentials  
   - Connect as AI Language Model input to Company Researcher node  

5. **Add Prospect Fetcher Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.explorium.ai/v1/prospects`  
   - JSON Body: Include filters for marketing department, job levels, has_email = true, and business_id from webhook (`{{ $json.body.business_id }}`)  
   - Authentication: Use Bearer and Header Auth with your credentials  
   - Connect output from Company Researcher node  

6. **Add IF Node (Has Prospect data/ Does not)**  
   - Type: IF  
   - Condition: Check if `$json.data.length === 0` to branch  
   - Connect Prospect Fetcher output to this node  

7. **For NO prospect data path:**  
   - Add Email Writer (NO prospect data) node (LangChain Agent)  
   - Configure system prompt to write general outbound email with company data and event data (from Company Researcher and Webhook)  
   - Use Anthropic Chat Model1 (Claude AI) as backend with credentials  
   - Connect IF node’s false branch to Email Writer (NO prospect data)  
   - Add Slack Output1 node (Slack)  
   - Configure Slack user to receive message (your Slack user ID)  
   - Connect Email Writer (NO prospect data) output to Slack Output1  

8. **For YES prospect data path:**  
   - Add Formulate Prospect Data node (Code)  
   - Use provided JavaScript to map raw prospect data into structured JSON  
   - Connect IF node’s true branch to this node  

9. **Add Prospect Mapping node (Code)**  
   - Extract prospect IDs from Formulate Prospect Data output  
   - Connect Formulate Prospect Data output to this node  

10. **Add Prospect Enrichment node (HTTP Request)**  
    - POST to `https://api.explorium.ai/v1/prospects/contacts_information/bulk_enrich`  
    - Pass prospect_ids from Prospect Mapping node as JSON body  
    - Use HTTP Header Auth credentials  
    - Connect Prospect Mapping output to this node  

11. **Add Full Prospect Formulation node (Code)**  
    - Merge enriched contact info with prospect profiles  
    - Use provided JS code  
    - Connect Prospect Enrichment output to this node  

12. **Add Loop Over Items node (Split In Batches)**  
    - Batch size: 1  
    - Connect Full Prospect Formulation output to this node  

13. **Add Email Writer (YES prospect data) node (LangChain Agent)**  
    - Configure prompt to generate personalized outbound email using prospect data, company research, and event data  
    - Use Anthropic llm1 (Claude AI) as backend with credentials  
    - Connect Loop Over Items output to this node  

14. **Add Slack Output node (Slack)**  
    - Set Slack channel to `evergreen-outbound-campaign`  
    - OAuth2 authentication with Slack credentials  
    - Connect Email Writer (YES prospect data) output to this node  

15. **Add Sticky Notes at appropriate points for documentation and explanations.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                             | Context or Link                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| The workflow uses Explorium’s MCP Playground as a soft CTA in emails: [Explorium MCP Playground](https://www.explorium.ai/mcp-playground/)                                                              | Included in email generator prompts                         |
| The sales emails are written to feel human, confident, and relevant, avoiding generic marketing fluff and focusing on data-driven insights.                                                              | Email Writer prompt instructions                            |
| The workflow handles two outreach strategies: personalized emails with prospect data vs. general company outreach if no prospects found, ensuring flexibility.                                            | Sticky Note1 explanation                                    |
| Slack integration requires OAuth2 credentials configured for posting to channels and sending direct messages to users.                                                                                   | Slack Output nodes                                          |
| The workflow assumes access to valid Explorium API keys and Anthropic API keys for AI language model usage.                                                                                              | Credential setup steps                                      |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.