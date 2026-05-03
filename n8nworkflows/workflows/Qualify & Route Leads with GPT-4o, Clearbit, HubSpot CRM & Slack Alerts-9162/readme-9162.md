Qualify & Route Leads with GPT-4o, Clearbit, HubSpot CRM & Slack Alerts

https://n8nworkflows.xyz/workflows/qualify---route-leads-with-gpt-4o--clearbit--hubspot-crm---slack-alerts-9162


# Qualify & Route Leads with GPT-4o, Clearbit, HubSpot CRM & Slack Alerts

---

### 1. Workflow Overview

This n8n workflow automates the qualification and routing of inbound B2B leads submitted via a web form. It integrates advanced AI analysis (GPT-4o via LangChain), company enrichment data from Clearbit, CRM updates in HubSpot, logging in Airtable, and Slack alerts for sales and review teams.

The workflow is logically divided into these functional blocks:

- **1.1 Lead Intake & AI Qualification:** Receives lead submissions via webhook, invokes an AI agent to analyze and score leads, producing structured qualification data.
- **1.2 Company Enrichment (Sub-Workflow Invocation):** Extracts company domain and enriches lead data using Clearbit API to obtain critical company metrics for scoring.
- **1.3 Data Structuring & Qualification Check:** Parses AI output, structures lead data, and applies a quality threshold to determine routing.
- **1.4 CRM & Notification Routing:** Sends high-quality leads (score ≥70) to HubSpot CRM and alerts the sales Slack channel.
- **1.5 Logging & Manual Review:** Logs lower-quality leads (score <70) in Airtable and notifies a Slack review channel for manual evaluation.
- **1.6 Clearbit Enrichment Sub-Workflow:** A separate n8n workflow designed to fetch and format detailed company data from Clearbit API, critical for AI scoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Intake & AI Qualification

**Overview:**  
Receives HTTP POST lead data, triggers the AI Lead Analysis Agent that uses GPT-4o with company enrichment tools to generate a structured lead qualification JSON.

**Nodes Involved:**  
- Form Submission5 (Webhook)  
- AI Lead Analysis Agent5 (LangChain Agent)  
- OpenAI Chat Model (GPT-4o LM)  
- Clearbit Enrichment Tool (LangChain Tool)  
- Structured Output Parser (LangChain Output Parser)  
- Structure Lead Data (Code)

**Node Details:**

- **Form Submission5**  
  - Type: Webhook  
  - Role: Entry point for lead submissions via `POST /lead-intake`  
  - Config: Path=`lead-intake`, HTTP method=POST, response sent from last node in chain  
  - Inputs: External HTTP POST requests  
  - Outputs: Lead JSON body forwarded to AI Lead Analysis Agent5  
  - Edge cases: Missing fields, malformed JSON payloads, network errors  
  - Version: 2

- **AI Lead Analysis Agent5**  
  - Type: LangChain Agent node using GPT-4o  
  - Role: Analyzes lead data with instructions to enrich company info via Clearbit  
  - Config: Custom prompt with lead data interpolation; uses Clearbit enrichment tool if domain available  
  - Uses: AI system prompt detailing scoring framework and tool usage  
  - Inputs: Lead data JSON from webhook  
  - Outputs: Raw AI JSON response (string) parsed by Structured Output Parser  
  - Edge cases: AI timeout, incomplete responses, failure to call enrichment tool  
  - Version: 1.7

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat Model  
  - Role: Provides the underlying GPT-4o model for AI Lead Analysis Agent5  
  - Config: Model=gpt-4o, max tokens=1000, temperature=0.3, responseFormat=json_object  
  - Credential: OpenAI API key  
  - Inputs: Text prompt from AI Lead Analysis Agent5  
  - Outputs: AI-generated JSON object for parsing  
  - Edge cases: API quota exceeded, auth failure, network timeout  
  - Version: 1

- **Clearbit Enrichment Tool**  
  - Type: LangChain Tool Workflow node  
  - Role: Invokes the Clearbit sub-workflow to enrich company data by domain  
  - Config: Workflow ID references sub-workflow (must be replaced with actual ID)  
  - Input Schema: `{ "domain": "company.com" }`  
  - Outputs: Enriched company data used internally by AI Lead Analysis Agent  
  - Edge cases: Missing domain, sub-workflow inactive, API errors  
  - Version: 1.1  
  - Sub-workflow: Clearbit Company Enrichment Tool (documented below)  

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI’s JSON string output into structured JSON fields with validation and auto-fix  
  - Config: Manual JSON schema defining required fields and enums for score, intent, urgency, budget, pain points, action, summary  
  - Inputs: Raw AI output string from AI Lead Analysis Agent5  
  - Outputs: Parsed JSON with typed fields for downstream use  
  - Edge cases: Malformed AI output, missing fields, schema violations  
  - Version: 1.3

- **Structure Lead Data**  
  - Type: Code node (JavaScript)  
  - Role: Combines original lead input with parsed AI qualification results, prepares final data object for routing  
  - Config: Extracts domain from website or email to pass downstream; adds metadata like submission date and workflow version  
  - Inputs: Parsed AI output and original webhook body  
  - Outputs: Structured JSON with lead and qualification data  
  - Edge cases: Invalid URLs, missing fields, parsing errors  
  - Version: 2

---

#### 2.2 Company Enrichment (Clearbit API Call & Data Formatting)

**Overview:**  
Fetches detailed company info from Clearbit API using domain, validates if company found, formats data for use in AI scoring.

**Nodes Involved:**  
- When called by another workflow (Execute Workflow Trigger)  
- Clearbit API (HTTP Request)  
- Company Found? (If)  
- Format Data (Code)  
- No Data (Code)

**Node Details:**

- **When called by another workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for sub-workflow called by main workflow’s Clearbit Enrichment Tool node  
  - Inputs: domain parameter from main workflow  
  - Outputs: Triggers Clearbit API node  
  - Version: 1.1

- **Clearbit API**  
  - Type: HTTP Request  
  - Role: Calls Clearbit Company API with domain query  
  - Config: URL template `https://company.clearbit.com/v2/companies/find?domain={{ $json.domain }}`, HTTP GET  
  - Authentication: HTTP Header Auth with Clearbit API key (Bearer token)  
  - Response: Never error mode enabled to avoid node failure on 404  
  - Inputs: domain  
  - Outputs: API JSON response or empty if company not found  
  - Edge cases: 404 Not Found, API rate limits, auth errors, malformed domain  
  - Version: 4.2

- **Company Found?**  
  - Type: If  
  - Role: Checks if Clearbit API response contains company name to confirm success  
  - Condition: `name` field is not empty  
  - Inputs: Clearbit API response  
  - Outputs: True branch to Format Data, False branch to No Data  
  - Version: 2

- **Format Data**  
  - Type: Code  
  - Role: Extracts relevant company fields from Clearbit response (employees, industry, revenue, tech, location, etc.) and formats into a simpler JSON structure  
  - Inputs: Clearbit API JSON data  
  - Outputs: Structured company enrichment data object  
  - Edge cases: Missing nested fields, null values  
  - Version: 2

- **No Data**  
  - Type: Code  
  - Role: Outputs a failure JSON indicating company not found, used to inform AI agent or fallback logic  
  - Inputs: None (uses domain input from trigger)  
  - Outputs: JSON with success: false and error message  
  - Version: 2

---

#### 2.3 Data Structuring & Qualification Check

**Overview:**  
Evaluates if the lead’s qualification score meets the threshold (≥70) to route appropriately.

**Nodes Involved:**  
- Structure Lead Data (Code)  
- High Quality Lead?5 (If)

**Node Details:**

- **Structure Lead Data**  
  (Described above; prepares data for this block)

- **High Quality Lead?5**  
  - Type: If  
  - Role: Checks if `qualification_score` ≥ 70  
  - Condition: Numeric greater than or equal to 70 on qualification_score field  
  - Inputs: Structured lead data JSON  
  - Outputs: True branch for qualified leads, False branch for review logging  
  - Edge cases: Missing or non-numeric scores, boundary values at 70  
  - Version: 2

---

#### 2.4 CRM & Notification Routing (High-Quality Leads)

**Overview:**  
For leads scoring 70 or above, updates HubSpot CRM with custom properties and sends a Slack alert to the Sales team channel.

**Nodes Involved:**  
- Add to HubSpot CRM (HubSpot node)  
- Notify Sales Team (Slack node)

**Node Details:**

- **Add to HubSpot CRM**  
  - Type: HubSpot node  
  - Role: Creates or updates a contact in HubSpot CRM with lead email and populates custom properties based on AI qualification  
  - Config: Email field mapped, custom properties include qualification_score, buying_intent, urgency_level, budget_indicator, ai_summary, pain_points, recommended_action  
  - Inputs: Structured lead data JSON  
  - Credentials: HubSpot OAuth2  
  - Edge cases: Contact creation failure, API rate limits, missing email  
  - Version: 2

- **Notify Sales Team**  
  - Type: Slack node  
  - Role: Sends formatted Slack message alerting sales channel of a hot lead with detailed qualification data and summary  
  - Config: Uses Slack OAuth2 credentials, posts to predefined sales channel ID  
  - Inputs: Lead and AI qualification details for message template  
  - Edge cases: Slack API errors, invalid channel ID, message formatting errors  
  - Version: 2.2

---

#### 2.5 Logging & Manual Review (Lower-Quality Leads)

**Overview:**  
For leads scoring below 70, logs the lead data and AI analysis in Airtable and notifies a Slack review channel for manual follow-up.

**Nodes Involved:**  
- Log to Airtable (Airtable node)  
- Request Manual Review (Slack node)

**Node Details:**

- **Log to Airtable**  
  - Type: Airtable node  
  - Role: Creates new record in Airtable base and table with all lead and AI qualification fields, marking status as “Needs Review”  
  - Config: Airtable Base ID and Table ID placeholders must be replaced; columns include Name, Email, Company, Phone, Website, Qualification_Score, Buying_Intent, Urgency, Budget, AI_Summary, Pain_Points, Recommended_Action, Status, Submitted_Date  
  - Inputs: Structured lead data JSON  
  - Credentials: Airtable Token  
  - Edge cases: API limits, invalid base/table ID, missing required fields  
  - Version: 2

- **Request Manual Review**  
  - Type: Slack node  
  - Role: Sends detailed message to Slack review channel informing about low scored lead needing human assessment, including options for decision replies  
  - Config: Uses Slack OAuth2, posts to review channel ID  
  - Inputs: Lead and AI qualification data for message  
  - Edge cases: Slack API errors, invalid channel ID, message formatting issues  
  - Version: 2.2

---

#### 2.6 Clearbit Enrichment Sub-Workflow

**Overview:**  
A dedicated workflow to enrich company data via Clearbit API, used as a LangChain tool by the main workflow.

**Nodes Involved:**  
- When called by another workflow (Execute Workflow Trigger)  
- Clearbit API (HTTP Request)  
- Company Found? (If)  
- Format Data (Code)  
- No Data (Code)

**Details:**  
This sub-workflow is designed to be reusable and activated independently. It requires setting up an HTTP Header Auth credential with Clearbit API key and activating the workflow to obtain its ID, which is referenced by the main workflow’s Clearbit Enrichment Tool node.

---

### 3. Summary Table

| Node Name                | Node Type                                   | Functional Role                        | Input Node(s)             | Output Node(s)                 | Sticky Note                                                                                                                               |
|--------------------------|---------------------------------------------|-------------------------------------|---------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Form Submission5         | Webhook                                     | Lead intake entry point              | External HTTP POST        | AI Lead Analysis Agent5        |                                                                                                                                           |
| AI Lead Analysis Agent5  | LangChain Agent (GPT-4o)                     | AI qualification and enrichment      | Form Submission5          | Structure Lead Data            |                                                                                                                                           |
| OpenAI Chat Model        | LangChain LM Chat Model                      | GPT-4o language model                | AI Lead Analysis Agent5   | AI Lead Analysis Agent5        |                                                                                                                                           |
| Clearbit Enrichment Tool | LangChain Tool Workflow                      | Calls Clearbit sub-workflow for data | AI Lead Analysis Agent5   | AI Lead Analysis Agent5        |                                                                                                                                           |
| Structured Output Parser | LangChain Output Parser Structured           | Parses AI JSON output                | OpenAI Chat Model         | AI Lead Analysis Agent5        |                                                                                                                                           |
| Structure Lead Data      | Code                                         | Combine AI results with original lead | AI Lead Analysis Agent5   | High Quality Lead?5            |                                                                                                                                           |
| High Quality Lead?5      | If                                           | Qualification score threshold check | Structure Lead Data       | Add to HubSpot CRM, Log to Airtable |                                                                                                                                           |
| Add to HubSpot CRM       | HubSpot                                      | Update CRM contact with lead data   | High Quality Lead?5       | Notify Sales Team              |                                                                                                                                           |
| Notify Sales Team        | Slack                                        | Alert sales team of qualified leads | Add to HubSpot CRM        |                               |                                                                                                                                           |
| Log to Airtable          | Airtable                                     | Log unqualified leads for review    | High Quality Lead?5       | Request Manual Review          |                                                                                                                                           |
| Request Manual Review    | Slack                                        | Notify review team for manual follow-up | Log to Airtable          |                               |                                                                                                                                           |
| When called by another workflow | Execute Workflow Trigger               | Sub-workflow trigger for Clearbit   | External call             | Clearbit API                  |                                                                                                                                           |
| Clearbit API             | HTTP Request                                | Fetch company data from Clearbit    | When called by another workflow | Company Found?               |                                                                                                                                           |
| Company Found?           | If                                           | Check if company data retrieved     | Clearbit API              | Format Data, No Data           |                                                                                                                                           |
| Format Data              | Code                                         | Format Clearbit company info        | Company Found?            | Return response               |                                                                                                                                           |
| No Data                  | Code                                         | Handle missing company data          | Company Found?            | Return response               |                                                                                                                                           |
| Sticky Note1             | Sticky Note                                  | Instructions & overview for AI Lead Qualifier | —                         | —                             | 🎯 AI Lead Qualifier + CRM Intelligence Setup instructions and workflow explanation                                                        |
| Sticky Note2             | Sticky Note                                  | Clearbit Enrichment Sub-Workflow guide | —                         | —                             | 🏢 Clearbit Company Enrichment Tool setup, API key instructions, and sub-workflow notes                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Main Workflow** in n8n.

2. **Add a Webhook node** named `Form Submission5`:
   - HTTP Method: POST
   - Path: `lead-intake`
   - Response Mode: Last Node

3. **Add a LangChain Agent node** named `AI Lead Analysis Agent5`:
   - Model: GPT-4o (via LangChain)
   - Prompt: Set detailed prompt including instructions to use Clearbit enrichment tool when domain available, with JSON output schema.
   - System Message: Define scoring framework and action recommendation logic.
   - Connect input from `Form Submission5`.

4. **Add a LangChain LM Chat Model node** named `OpenAI Chat Model`:
   - Model: gpt-4o
   - Max Tokens: 1000
   - Temperature: 0.3
   - Response Format: json_object
   - Credentials: Configure OpenAI API key
   - Connect as AI language model for `AI Lead Analysis Agent5`.

5. **Add a LangChain Tool Workflow node** named `Clearbit Enrichment Tool`:
   - Set to call the Clearbit Enrichment Sub-Workflow (create separately)
   - Input Schema: `{ "domain": "company.com" }`
   - Connect as AI tool in `AI Lead Analysis Agent5`.

6. **Add a LangChain Output Parser Structured node** named `Structured Output Parser`:
   - Schema: Use manual JSON schema defining all required fields and enums (qualification_score, buying_intent, etc.)
   - Auto-fix enabled
   - Connect as AI output parser in `AI Lead Analysis Agent5`.

7. **Add a Code node** named `Structure Lead Data`:
   - JavaScript: Parse AI output JSON, combine with original webhook submission data, extract domain from website or email.
   - Add metadata fields: submission date, source, workflow version.
   - Connect output from `AI Lead Analysis Agent5`.

8. **Add an If node** named `High Quality Lead?5`:
   - Condition: qualification_score ≥ 70
   - Connect input from `Structure Lead Data`.

9. **Add a HubSpot node** named `Add to HubSpot CRM`:
   - Operation: Create/Update contact using email
   - Map custom properties: qualification_score, buying_intent, urgency_level, budget_indicator, ai_summary, pain_points (comma-separated), recommended_action
   - Credentials: HubSpot OAuth2
   - Connect True output from `High Quality Lead?5`.

10. **Add a Slack node** named `Notify Sales Team`:
    - Message: Formatted alert with lead and AI data for sales channel
    - Credentials: Slack OAuth2
    - Channel ID: Paste sales channel ID
    - Connect output from `Add to HubSpot CRM`.

11. **Add an Airtable node** named `Log to Airtable`:
    - Operation: Create record
    - Base ID and Table ID: Replace placeholders with your Airtable base/table IDs
    - Columns: Map all lead and AI fields, set Status to "Needs Review"
    - Credentials: Airtable API Token
    - Connect False output from `High Quality Lead?5`.

12. **Add a Slack node** named `Request Manual Review`:
    - Message: Notify review channel with lead details and AI analysis
    - Credentials: Slack OAuth2
    - Channel ID: Paste review channel ID
    - Connect output from `Log to Airtable`.

13. **Create the Clearbit Enrichment Sub-Workflow** separately:
    - Add an Execute Workflow Trigger node named `When called by another workflow`
    - Add HTTP Request node named `Clearbit API`:
      - URL: `https://company.clearbit.com/v2/companies/find?domain={{ $json.domain }}`
      - Authentication: HTTP Header Auth (Bearer token with Clearbit API key)
      - Set "Never error" in response settings to handle 404 gracefully
    - Add If node named `Company Found?`:
      - Condition: Check if `name` field is not empty
    - Add Code node named `Format Data`:
      - Format Clearbit response into simplified JSON (company name, employees, industry, location, revenue, tech stack, etc.)
    - Add Code node named `No Data`:
      - Return JSON with success: false and error message if company not found
    - Connect nodes accordingly:
      - Trigger → Clearbit API → Company Found? → Format Data or No Data
    - Activate this sub-workflow and note its Workflow ID

14. **Update the Clearbit Enrichment Tool node** in main workflow:
    - Paste the Clearbit Sub-Workflow ID into its configuration

15. **Add Credentials:**
    - OpenAI API key (for GPT-4o)
    - Clearbit API key (HTTP Header Auth with Authorization: Bearer token)
    - HubSpot OAuth2 credentials
    - Slack OAuth2 credentials
    - Airtable API token

16. **Replace Placeholders:**
    - Slack sales and review channel IDs in Slack nodes
    - Airtable Base and Table IDs in Airtable node

17. **Activate the Main Workflow**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| 🎯 AI Lead Qualifier + CRM Intelligence: Detailed setup instructions including Slack channels, Airtable base and table setup, HubSpot custom properties creation, and credential additions.                               | Sticky Note1 content in workflow; key for onboarding and configuration                                                               |
| 🏢 Clearbit Company Enrichment Tool: Sub-workflow setup instructions, API key creation at Clearbit, HTTP Header Auth credential setup, and activation steps.                                                                | Sticky Note2 content; foundational for accurate company data enrichment                                                               |
| Clearbit API documentation: https://clearbit.com/docs#company-api                                                                                                                                                            | For understanding the data returned and API usage                                                                                     |
| HubSpot custom properties must be created as per names and types specified to enable correct CRM updates.                                                                                                                  | HubSpot CRM setup instructions in Sticky Note1                                                                                         |
| Slack channel IDs can be obtained via right-click → View details in Slack desktop app or workspace UI.                                                                                                                      | Slack integration instructions in Sticky Note1                                                                                        |
| Ensure n8n version supports LangChain nodes with GPT-4o model and structured output parsing for this workflow to function correctly.                                                                                       | Version requirement note                                                                                                              |
| The workflow is designed to handle missing or incomplete company data gracefully by falling back to manual review logging and Slack alerts.                                                                                | Error handling and fallback strategy                                                                                                 |

---

**Disclaimer:**  
The provided text is exclusively sourced from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or offensive material. All processed data is public and lawful.

---