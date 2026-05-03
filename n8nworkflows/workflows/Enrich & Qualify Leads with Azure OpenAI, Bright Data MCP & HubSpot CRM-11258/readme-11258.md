Enrich & Qualify Leads with Azure OpenAI, Bright Data MCP & HubSpot CRM

https://n8nworkflows.xyz/workflows/enrich---qualify-leads-with-azure-openai--bright-data-mcp---hubspot-crm-11258


# Enrich & Qualify Leads with Azure OpenAI, Bright Data MCP & HubSpot CRM

### 1. Workflow Overview

This workflow, titled **"Enrich & Qualify Leads with Azure OpenAI, Bright Data MCP & HubSpot CRM"**, automates the process of capturing raw lead data from a form submission, enriching it with verified business intelligence, scoring the lead’s fit quality, and updating the CRM accordingly. It targets B2B SaaS, tech, and RevOps agencies aiming to streamline lead qualification and CRM data hygiene.

The workflow is organized into four main logical blocks:

- **1.1 Input Reception:** Captures lead data (Name and Email) from a web form.
- **1.2 Lead Enrichment:** Uses Azure OpenAI and Bright Data MCP (a web search tool) to enrich contact and company information with verified data.
- **1.3 Lead Qualification (Scoring):** Applies AI-driven scoring logic on the enriched data to generate a numeric fit score (0–100).
- **1.4 CRM Update & Notification:** Depending on the score, updates HubSpot CRM records and optionally sends Slack notifications to the sales team for qualified leads.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Collects initial lead data via a web form trigger capturing essential fields (Name and Email). This is the entry point for the workflow.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Starts workflow on form data submission.  
  - Configuration: Listens for submissions on a form titled “Lead Form” with required fields “Name” (text) and “Email” (email type).  
  - Inputs: External form submission via webhook.  
  - Outputs: JSON object containing submitted Name and Email.  
  - Edge Cases: Missing required fields prevented by form validation; webhook failures possible.  
  - Notes: Positioned as the workflow’s entry node.

---

#### 2.2 Lead Enrichment

**Overview:**  
Enriches the raw lead data with business details extracted or verified using Azure OpenAI and Bright Data MCP. The enrichment involves extracting job titles, LinkedIn profiles, company details such as industry, size, revenue, location, and funding. Uses a multi-step AI agent workflow with structured JSON input/output and external web data search.

**Nodes Involved:**  
- Azure OpenAI Chat Model  
- MCP Client  
- Structured Output Parser  
- Lead Enricher Agent

**Node Details:**

- **Azure OpenAI Chat Model**  
  - Type: Azure OpenAI Language Model (Chat)  
  - Role: Provides foundational AI processing for the enrichment agent.  
  - Configuration: Uses Azure OpenAI API credentials, model set as “openai” with default options.  
  - Inputs: Calls from Lead Enricher Agent.  
  - Outputs: Chat completions to support enrichment logic.  
  - Edge Cases: API key/auth errors, rate limiting, response latency.  
  - Version: 1

- **MCP Client**  
  - Type: Bright Data MCP Client Tool  
  - Role: Searches the public web for verified, reliable company/contact info when AI confidence is low or data missing.  
  - Configuration: Endpoint URL includes authentication token; no additional options specified.  
  - Inputs: Called by Lead Enricher Agent as a tool.  
  - Outputs: Raw search results feeding back into the enrichment logic.  
  - Edge Cases: Connectivity issues, invalid token, unexpected data format.  
  - Version: 1.2

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI agent output into a strictly formatted JSON schema containing contact and company fields.  
  - Configuration: JSON schema example with fields for job title, LinkedIn, country, company name, description, industry, employees, revenue, LinkedIn page, HQ location, funding.  
  - Inputs: Response from AI enrichment agent.  
  - Outputs: Structured JSON object for downstream use.  
  - Edge Cases: Parsing failures if AI output deviates from schema.  
  - Version: 1.3

- **Lead Enricher Agent**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates the enrichment logic combining Azure OpenAI and Bright Data MCP; decides when to call external search tool; applies business rules to avoid hallucination, ensures verifiable data only.  
  - Configuration:  
    - Text prompt includes contact and company info from form data, with instructions to call Bright Data MCP if fields missing or confidence low.  
    - System message enforces strict data enrichment rules with no hallucination or guessing.  
    - Uses Structured Output Parser to enforce JSON output.  
  - Inputs: Form submission data, Azure OpenAI, MCP Client.  
  - Outputs: Enriched contact and company JSON object.  
  - Edge Cases: AI hallucination risk mitigated by strict instructions, tool call failure fallback.  
  - Version: 3

---

#### 2.3 Lead Qualification (Scoring)

**Overview:**  
Scores the enriched lead data to generate a numeric Fit Score (0–100) that reflects the lead’s alignment with the company’s Ideal Customer Profile (ICP). The score considers industry relevance, company size, job title seniority, and funding status, using AI logic with strict input constraints.

**Nodes Involved:**  
- Azure OpenAI Chat Model1  
- Structured Output Parser1  
- Lead Scoring Agent  
- If (conditional)

**Node Details:**

- **Azure OpenAI Chat Model1**  
  - Type: Azure OpenAI Language Model (Chat)  
  - Role: Provides AI processing for lead scoring.  
  - Configuration: Azure OpenAI API with “openai” model, default options.  
  - Inputs: Lead Scoring Agent calls it.  
  - Outputs: Chat completions for scoring logic.  
  - Edge Cases: Similar to original Azure OpenAI node.

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the lead scoring agent’s output into JSON with a single numeric “fit_score” field.  
  - Configuration: Example schema with a single fit_score integer.  
  - Inputs: Output from Lead Scoring Agent.  
  - Outputs: Parsed fit_score for routing.  
  - Edge Cases: Parsing failure if AI output format deviates.  

- **Lead Scoring Agent**  
  - Type: LangChain Agent Node  
  - Role: Calculates a numeric Fit Score (0–100) solely from enriched data, strictly forbidding enrichment or data alteration.  
  - Configuration:  
    - Text prompt provides schema and sample JSON, instructions to only use provided data and no narrative.  
    - Uses Structured Output Parser1 for output validation.  
  - Inputs: Output from Lead Enricher Agent.  
  - Outputs: JSON with fit_score field.  
  - Edge Cases: AI model misinterpretation, missing data leading to conservative scoring.  
  - Version: 3

- **If**  
  - Type: Conditional Node  
  - Role: Routes workflow based on fit_score value.  
  - Configuration: Condition tests if fit_score > 70.  
  - Inputs: From Lead Scoring Agent.  
  - Outputs: Two branches — “true” for qualified leads, “false” for not qualified.  
  - Edge Cases: Missing or malformed fit_score could disrupt routing.  
  - Version: 2.2

---

#### 2.4 CRM Update & Notification

**Overview:**  
Updates HubSpot CRM with enriched contact and company data. For qualified leads (fit_score > 70), it also sends a Slack notification to alert the sales team. For unqualified leads, it updates HubSpot silently without notifications.

This block includes company search and update nodes to ensure company records are correctly linked and enriched.

**Nodes Involved:**  
- Create or update a contact  
- Search for a company by Domain  
- Update a company2  
- Send a message (Slack)  
- Create or update a contact1  
- Search for a company by Domain1  
- Update a company  
- Sticky Notes (for contextual explanation)

**Node Details:**

- **Create or update a contact**  
  - Type: HubSpot Node (contact)  
  - Role: Creates or updates contact record in HubSpot with enriched details (job title, LinkedIn, country, company name).  
  - Configuration: Uses email from form submission as unique key, updates additional fields from Lead Enricher Agent output.  
  - Inputs: From "If" node (qualified branch).  
  - Outputs: Passes data to “Search for a company by Domain” for company linkage.  
  - Credentials: HubSpot App Token.  
  - Edge Cases: API errors, duplicate contacts, missing email.

- **Search for a company by Domain**  
  - Type: HubSpot Node (company)  
  - Role: Searches HubSpot companies by email domain to find matching company record.  
  - Configuration: Uses domain extracted from contact email.  
  - Inputs: From “Create or update a contact”.  
  - Outputs: Company search result for updating company record.  
  - Credentials: HubSpot App Token.  
  - Edge Cases: No company found, multiple matches.

- **Update a company2**  
  - Type: HubSpot Node (company)  
  - Role: Updates company record with enriched info (city, description, revenue, country, funding, size, industry, LinkedIn page).  
  - Configuration: Uses company ID from previous search node, fields populated from Lead Enricher Agent output.  
  - Inputs: From “Search for a company by Domain”.  
  - Outputs: Leads to Slack notification node.  
  - Credentials: HubSpot App Token.  
  - Edge Cases: Invalid company ID, API errors.

- **Send a message**  
  - Type: Slack Node  
  - Role: Sends a Slack notification message to a specific user ID with lead details for qualified leads.  
  - Configuration: Message includes Name, Email, Job Title, LinkedIn profile, Country from Lead Enricher Agent and form submission.  
  - Inputs: From “Update a company2”.  
  - Outputs: Terminal node.  
  - Credentials: Slack API OAuth2 credentials.  
  - Edge Cases: Slack rate limits, user ID invalid.

- **Create or update a contact1**  
  - Type: HubSpot Node (contact)  
  - Role: Similar to “Create or update a contact” but for unqualified leads branch (fit_score ≤ 70).  
  - Configuration: Same as above, updates contact silently.  
  - Inputs: From "If" node (not qualified branch).  
  - Outputs: Passes to “Search for a company by Domain1”.  
  - Credentials: HubSpot App Token.

- **Search for a company by Domain1**  
  - Type: HubSpot Node (company)  
  - Role: Same as “Search for a company by Domain” but for unqualified branch.  
  - Inputs: From “Create or update a contact1”.  
  - Outputs: Passes to “Update a company”.  
  - Credentials: HubSpot App Token.

- **Update a company**  
  - Type: HubSpot Node (company)  
  - Role: Updates company record for unqualified leads with enriched info (similar fields as above).  
  - Inputs: From “Search for a company by Domain1”.  
  - Outputs: Terminal node.  
  - Credentials: HubSpot App Token.

- **Sticky Notes**  
  - Provide high-level explanations for each logical block and overall workflow purpose.  
  - Content highlights:  
    - Lead enrichment with Bright Data MCP  
    - Lead qualification rules  
    - CRM update and Slack notification logic  
    - Summary and goals of the workflow (authored by Sparsh from Automation Jinn)  

---

### 3. Summary Table

| Node Name                   | Node Type                              | Functional Role                         | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                   |
|-----------------------------|--------------------------------------|---------------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| On form submission           | Form Trigger                         | Capture lead input (Name, Email)      | External webhook             | Lead Enricher Agent           | ## Form Trigger                                                                             |
| Azure OpenAI Chat Model      | Azure OpenAI LM Chat                 | Provide AI completions for enrichment | Lead Enricher Agent (ai_languageModel) | Lead Enricher Agent           |                                                                                              |
| MCP Client                  | Bright Data MCP Client Tool           | Search public web for info            | Lead Enricher Agent (ai_tool) | Lead Enricher Agent           | ## Lead Enrichment using Bright Data MCP                                                   |
| Structured Output Parser    | LangChain Structured Output Parser   | Parse enrichment AI output             | Lead Enricher Agent (ai_outputParser) | Lead Enricher Agent           |                                                                                              |
| Lead Enricher Agent         | LangChain Agent                      | Orchestrate data enrichment            | On form submission, Azure OpenAI Chat Model, MCP Client | Lead Scoring Agent           | ## Lead Enrichment using Bright Data MCP                                                   |
| Azure OpenAI Chat Model1    | Azure OpenAI LM Chat                 | Provide AI completions for scoring     | Lead Scoring Agent (ai_languageModel) | Lead Scoring Agent            | ## Lead Qualifier on Predefined rules                                                     |
| Structured Output Parser1   | LangChain Structured Output Parser   | Parse scoring AI output                | Lead Scoring Agent (ai_outputParser) | Lead Scoring Agent            |                                                                                              |
| Lead Scoring Agent          | LangChain Agent                      | Calculate lead fit score (0–100)       | Lead Enricher Agent          | If                          | ## Lead Qualifier on Predefined rules                                                     |
| If                         | Conditional                        | Route based on fit_score > 70          | Lead Scoring Agent           | Create/Update Contact, Create/Update Contact1 | ## Notifying team and Updating CRM if qualified                                            |
| Create or update a contact  | HubSpot Contact Node                 | Update/create contact for qualified leads | If (true branch)             | Search for a company by Domain | ## Notifying team and Updating CRM if qualified                                            |
| Search for a company by Domain | HubSpot Company Search             | Find company by email domain            | Create or update a contact   | Update a company2             | ## Notifying team and Updating CRM if qualified                                            |
| Update a company2           | HubSpot Company Update               | Update company data for qualified leads | Search for a company by Domain | Send a message               | ## Notifying team and Updating CRM if qualified                                            |
| Send a message              | Slack Message                       | Notify sales team about qualified lead | Update a company2            | None                         | ## Notifying team and Updating CRM if qualified                                            |
| Create or update a contact1 | HubSpot Contact Node                 | Update/create contact for unqualified leads | If (false branch)            | Search for a company by Domain1 | ## Only updating CRM if not qualified                                                      |
| Search for a company by Domain1 | HubSpot Company Search             | Find company by email domain            | Create or update a contact1  | Update a company             | ## Only updating CRM if not qualified                                                      |
| Update a company            | HubSpot Company Update               | Update company data for unqualified leads | Search for a company by Domain1 | None                       | ## Only updating CRM if not qualified                                                      |
| Sticky Note                 | Sticky Note                         | Explanation - Lead Enrichment          | None                        | None                         | ## Lead Enrichment using Bright Data MCP                                                   |
| Sticky Note1                | Sticky Note                         | Explanation - Lead Qualification       | None                        | None                         | ## Lead Qualifier on Predefined rules                                                     |
| Sticky Note2                | Sticky Note                         | Explanation - CRM Update & Notifications | None                        | None                         | ## Notifying team and Updating CRM if qualified                                            |
| Sticky Note3                | Sticky Note                         | Explanation - CRM update for unqualified leads | None                     | None                         | ## Only updating CRM if not qualified                                                      |
| Sticky Note4                | Sticky Note                         | Workflow overview and credits          | None                        | None                         | ### 🚀 AI Lead Enricher & Qualifier using Bright Data MCP and Hubspot( By Sparsh from Automation Jinn automationjinn.com) |
| Sticky Note5                | Sticky Note                         | Explanation - Form Trigger              | None                        | None                         | ## Form Trigger                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" Node**  
   - Type: Form Trigger  
   - Configure webhook with form title “Lead Form”  
   - Add required fields: “Name” (text), “Email” (email)  
   - Position as entry node.

2. **Add "Azure OpenAI Chat Model" Node**  
   - Type: Azure OpenAI Language Model (Chat)  
   - Set model to “openai”  
   - Attach Azure OpenAI credentials (Azure Open AI account)  
   - No special options.

3. **Add "MCP Client" Node**  
   - Type: Bright Data MCP Client Tool  
   - Set endpoint URL with valid Bright Data MCP token  
   - No extra options needed.

4. **Add "Structured Output Parser" Node**  
   - Type: LangChain Structured Output Parser  
   - Provide JSON schema example with contact and company fields as per specification.

5. **Add "Lead Enricher Agent" Node**  
   - Type: LangChain Agent  
   - Configure prompt text with contact and company info placeholders (Name, Email, company from email domain)  
   - Add system message enforcing no hallucination, verified data only, use MCP tool for missing fields  
   - Set prompt type: define  
   - Enable output parser, link to "Structured Output Parser"  
   - Connect inputs: Form submission to this node; Azure OpenAI Chat Model and MCP Client as AI tool and language model.  

6. **Add "Azure OpenAI Chat Model1" Node**  
   - Same as step 2, separate instance for scoring.

7. **Add "Structured Output Parser1" Node**  
   - Similar to step 4 but JSON schema only includes { "fit_score": 0 }.

8. **Add "Lead Scoring Agent" Node**  
   - Type: LangChain Agent  
   - Prompt text instructs to calculate numeric fit score from enriched JSON only, no enrichment allowed  
   - System message restricts output to JSON with “fit_score” number  
   - Use “Structured Output Parser1”  
   - Connect input from "Lead Enricher Agent" output  
   - Connect AI language model to "Azure OpenAI Chat Model1".

9. **Add "If" Node**  
   - Conditional: if fit_score > 70  
   - Input from Lead Scoring Agent.

10. **Qualified Branch:**

    a. **Add "Create or update a contact" Node**  
       - HubSpot contact resource, operation create/update  
       - Use Email from form submission  
       - Additional fields mapped from Lead Enricher Agent output (job title, LinkedIn, country, company name)  
       - Use HubSpot App Token credentials.

    b. **Add "Search for a company by Domain" Node**  
       - HubSpot company search by domain  
       - Domain extracted from contact email domain (hs_email_domain property)  
       - Credentials same as above.

    c. **Add "Update a company2" Node**  
       - HubSpot company update operation  
       - Use company ID from search result  
       - Update fields: city, description, revenue, country, funding, size, industry, LinkedIn page from Lead Enricher Agent output  
       - Credentials: HubSpot App Token.

    d. **Add "Send a message" Node**  
       - Slack message node  
       - Message text includes lead fields (Name, Email, Job Title, LinkedIn, Country)  
       - User: specify Slack user ID to notify  
       - Use Slack API OAuth2 credentials.

11. **Not Qualified Branch:**

    a. **Add "Create or update a contact1" Node**  
       - Same as qualified contact node but for unqualified branch  
       - HubSpot App Token credentials.

    b. **Add "Search for a company by Domain1" Node**  
       - Same as above, separate instance.

    c. **Add "Update a company" Node**  
       - Same as "Update a company2" but for unqualified branch  
       - HubSpot App Token credentials.

12. **Connect Nodes:**  
    - Form submission → Lead Enricher Agent  
    - Lead Enricher Agent → Lead Scoring Agent  
    - Lead Scoring Agent → If node  
    - If (true) → Create or update a contact → Search for a company by Domain → Update a company2 → Send a message  
    - If (false) → Create or update a contact1 → Search for a company by Domain1 → Update a company

13. **Add Sticky Notes for clarity** at each block as per original workflow for documentation and internal reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                               | Context or Link                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| 🚀 AI Lead Enricher & Qualifier using Bright Data MCP and Hubspot (By Sparsh from Automation Jinn automationjinn.com) This workflow converts **raw form submissions → enriched CRM profiles → qualified lead alerts** with zero manual effort.                            | https://automationjinn.com                                                                  |
| Workflow designed to ensure **only high-quality leads reach the sales team** while maintaining a **clean and consistently enriched CRM** automatically.                                                                                                                    | Internal project goal                                                                       |
| The Bright Data MCP tool enables real-time, verifiable web data lookup integrated with AI to avoid hallucination and guesswork.                                                                                                                                           | https://brightdata.com                                                                       |
| Slack notifications are targeted to a specific user ID for immediate sales team alerting on qualified leads.                                                                                                                                                               | Slack API integration details                                                               |
| HubSpot integration uses App Token authentication for seamless CRM updates of contacts and companies, ensuring data consistency across CRM records.                                                                                                                         | HubSpot API documentation                                                                   |
| The workflow uses LangChain agents with strict output parsers to guarantee structured JSON outputs, critical for reliable routing and database updates.                                                                                                                    | https://docs.langchain.com/                                                                 |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow built with n8n, a no-code automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.