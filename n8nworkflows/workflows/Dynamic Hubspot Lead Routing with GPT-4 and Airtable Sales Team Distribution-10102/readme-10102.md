Dynamic Hubspot Lead Routing with GPT-4 and Airtable Sales Team Distribution

https://n8nworkflows.xyz/workflows/dynamic-hubspot-lead-routing-with-gpt-4-and-airtable-sales-team-distribution-10102


# Dynamic Hubspot Lead Routing with GPT-4 and Airtable Sales Team Distribution

### 1. Workflow Overview

This workflow automates the dynamic routing of new sales leads captured in HubSpot by leveraging AI (GPT-4) and Airtable to intelligently assign leads to the most appropriate sales team. It handles lead qualification, data cleansing, team matching based on multiple criteria, and notification delivery, ensuring fast, accurate, and scalable lead distribution.

Logical blocks:

- **1.1 Lead Capture and Data Retrieval:** Triggered by new leads in HubSpot, fetches detailed contact and company info.
- **1.2 Data Cleansing and Standardization:** Processes and structures lead data for downstream use.
- **1.3 Lead Logging in Airtable:** Records the cleaned lead into Airtable for tracking and reference.
- **1.4 AI-Based Lead Qualification and Team Assignment:** Uses GPT-4 with access to Airtable team data to select the best sales team based on industry, region, team size, and capacity.
- **1.5 Notifications and Updates:** Updates Airtable and HubSpot with assignment info; sends notifications via Slack and Gmail.
- **1.6 Auxiliary Nodes and Utilities:** Memory buffer for AI context, structured output parser, no-op node, and sticky notes for documentation.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Capture and Data Retrieval

**Overview:**  
This block triggers on new HubSpot lead events and retrieves detailed contact information, including associated company data, to provide comprehensive lead profiles for processing.

**Nodes Involved:**  
- HubSpot Trigger  
- Get a contact

**Node Details:**  

- **HubSpot Trigger**  
  - Type: HubSpot Trigger node  
  - Role: Listens for new lead/contact events in HubSpot.  
  - Config: Uses developer API credentials; triggers on all events (empty eventValues array).  
  - Input: External webhook events from HubSpot  
  - Output: Raw lead data including contact and associated company fields.  
  - Edge cases: Webhook connectivity issues, event filtering misconfigurations, HubSpot API rate limits.

- **Get a contact**  
  - Type: HubSpot node (operation: get contact)  
  - Role: Fetch full contact details by contact ID with OAuth2 authentication.  
  - Config: Retrieves multiple properties (firstname, lastname, email, company, website, descriptions, phone, jobtitle, team size, industry, createdate).  
  - Input: Contact ID from trigger node (dynamically set).  
  - Output: Detailed JSON of contact and associated company properties.  
  - Edge cases: OAuth token expiration, missing contact IDs, API errors, partial data availability.

---

#### 1.2 Data Cleansing and Standardization

**Overview:**  
Transforms the complex nested HubSpot data structure into a clean, flat object with fallback values and standardized keys for consistent downstream processing.

**Nodes Involved:**  
- Code in JavaScript

**Node Details:**  

- **Code in JavaScript**  
  - Type: Code node (JavaScript)  
  - Role: Extracts, normalizes, and sanitizes key lead fields from raw HubSpot data.  
  - Config: Uses safe path access; collects first/last name, email, phone, job title, team size, industry (with multiple fallbacks), company website, company name.  
  - Output: Cleaned lead object with "N/A" defaults for missing values.  
  - Input: JSON from "Get a contact" node.  
  - Edge cases: Unexpected data schemas, missing nested properties, null or undefined values.

---

#### 1.3 Lead Logging in Airtable

**Overview:**  
Creates a new record in the Airtable "Leads" table to log the cleaned lead data for tracking and reference in the sales process.

**Nodes Involved:**  
- Create a record (Airtable)

**Node Details:**  

- **Create a record**  
  - Type: Airtable node (create operation)  
  - Role: Inserts the cleaned lead data into a specified Airtable base and table.  
  - Config: Maps fields explicitly (Email, Job Title, Last Name, Team Size, Industry, First Name, Company Name, Description, Phone Number, Record ID, Company Website).  
  - Authentication: OAuth2 with Airtable personal access token.  
  - Input: Clean lead JSON from the JavaScript Code node.  
  - Output: Airtable record metadata (record ID, fields).  
  - Edge cases: API limits, invalid field mapping, auth expiration.

---

#### 1.4 AI-Based Lead Qualification and Team Assignment

**Overview:**  
This critical block uses an AI Agent powered by GPT-4 to analyze lead details and match the lead to the best-fit sales team by querying an Airtable "TeamDatabase". It applies strict logic based on industry, region, team size, and team capacity, and formats a structured JSON output with assignment details and notifications.

**Nodes Involved:**  
- AI Agent (LangChain Agent)  
- TeamDatabase (Airtable Tool)  
- Simple Memory (LangChain Memory Buffer)  
- OpenAI Chat Model  
- Google Gemini Chat Model  
- Structured Output Parser

**Node Details:**  

- **TeamDatabase**  
  - Type: Airtable Tool node  
  - Role: Provides read/search access to the Airtable "Growify Sales Teams" table containing team data (Team Name, Focus Industries, Region, Contact Person, Email, Max Leads, Slack Channel).  
  - Auth: Airtable OAuth2 token.  
  - Input: AI Agent invokes the tool to retrieve team records for matching.  
  - Edge cases: Airtable API failures, no matching teams found.

- **Simple Memory**  
  - Type: LangChain memory buffer  
  - Role: Maintains conversational context up to 10 entries keyed by "data" to support AI reasoning continuity.  
  - Input: Lead data and team info context.  
  - Edge cases: Memory overflow, session key mismanagement.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat model (GPT-4 variant)  
  - Role: Processes AI prompts for lead routing logic with system message instructing strict matching criteria and output format.  
  - Auth: OpenAI API key (free credits).  
  - Edge cases: API latency, rate limits, prompt parsing errors.

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat model (PaLM)  
  - Role: Alternative LLM for language processing, used alongside OpenAI for robust AI handling.  
  - Auth: Google PaLM API credentials.  
  - Edge cases: API availability, quota limits.

- **Structured Output Parser**  
  - Type: LangChain output parser (structured JSON)  
  - Role: Validates and auto-corrects AI output to ensure it matches the required JSON schema for lead persona, recommended team, and notifications.  
  - Config: Uses predefined JSON example schema for parsing.  
  - Edge cases: Malformed AI output, parsing failures.

- **AI Agent**  
  - Type: LangChain Agent combining tools and LLMs  
  - Role: Orchestrates AI lead qualification by querying TeamDatabase, applying routing logic, generating notifications, and returning structured JSON output.  
  - Input: Clean lead data, team database tool, memory, chat models.  
  - Output: Final JSON with lead persona, recommended team, Slack and email notification details.  
  - Edge cases: No suitable team found, API errors, logic inconsistencies.  
  - Notes: Contains explicit system prompt with strict lead routing logic and fallback response if no match found.

---

#### 1.5 Notifications and Updates

**Overview:**  
This block updates Airtable and HubSpot with assignment results, and notifies the assigned sales representative via Slack and Gmail with detailed messages about the new lead.

**Nodes Involved:**  
- Update record (Airtable)  
- Send a message (Slack)  
- Send a message1 (Gmail)  
- Create or update a contact (HubSpot)  
- No Operation, do nothing (noop)

**Node Details:**  

- **Update record**  
  - Type: Airtable update operation  
  - Role: Updates the lead record with assigned team, lead status, and AI persona summary.  
  - Config: Matches by Email field; updates Assigned to, Lead Status, Persona (AI) fields.  
  - Auth: Airtable OAuth2 token.  
  - Input: AI Agent output JSON and initial lead data.  
  - Edge cases: Record not found, concurrency conflicts.

- **Send a message** (Slack)  
  - Type: Slack node  
  - Role: Sends a notification message to the assigned team’s Slack channel, pinging the contact person.  
  - Config: Constructs message text and channel dynamically from AI Agent output; uses OAuth2 Slack credentials.  
  - Edge cases: Slack API errors, invalid channel or user handle.

- **Send a message1** (Gmail)  
  - Type: Gmail node  
  - Role: Sends an internal email notification to the assigned team contact with lead details.  
  - Config: Uses Gmail OAuth2 credentials; fills recipient, subject, and body from AI Agent output.  
  - Edge cases: Email delivery issues, quota limits.

- **Create or update a contact** (HubSpot)  
  - Type: HubSpot node (create or update contact)  
  - Role: Updates HubSpot contact record with assignment status, AI persona summary, and message to sales team.  
  - Config: Uses OAuth2 credentials; updates status, message, persona fields dynamically.  
  - Edge cases: HubSpot API limits, contact not found.

- **No Operation, do nothing**  
  - Type: NoOp node  
  - Role: Terminal node after HubSpot update, effectively ends workflow branch.  
  - Edge cases: None.

---

#### 1.6 Auxiliary Nodes and Documentation

**Overview:**  
Contains sticky notes for user guidance, author info, and a no-op node to conclude the workflow cleanly.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note5  
- No Operation, do nothing (covered above)

**Node Details:**  

- **Sticky Notes** contain detailed workflow overview, use cases, user guide, benefits, and author contact information, providing context and documentation directly within the workflow editor.  
- **No Operation** node is a clean end to the workflow chain.

---

### 3. Summary Table

| Node Name              | Node Type                        | Functional Role                           | Input Node(s)               | Output Node(s)             | Sticky Note                                               |
|------------------------|---------------------------------|-----------------------------------------|----------------------------|----------------------------|-----------------------------------------------------------|
| HubSpot Trigger        | HubSpot Trigger                 | Triggers workflow on new HubSpot lead  | -                          | Get a contact              |                                                           |
| Get a contact          | HubSpot                        | Fetches detailed contact info           | HubSpot Trigger            | Code in JavaScript         |                                                           |
| Code in JavaScript     | Code                           | Cleans and standardizes lead data       | Get a contact              | Create a record            |                                                           |
| Create a record        | Airtable                       | Logs cleaned lead in Airtable            | Code in JavaScript         | AI Agent                  |                                                           |
| AI Agent               | LangChain Agent                | AI lead qualification & team assignment | Create a record, TeamDatabase, Simple Memory, OpenAI Chat Model, Gemini Chat Model, Structured Output Parser | Update record             |                                                           |
| TeamDatabase           | Airtable Tool                  | Provides sales team data for AI          | -                          | AI Agent (tool input)      |                                                           |
| Simple Memory          | LangChain Memory Buffer        | Maintains AI session context             | -                          | AI Agent (memory input)    |                                                           |
| OpenAI Chat Model      | LangChain LLM Chat Model       | LLM for AI processing                     | -                          | AI Agent (languageModel input) |                                                           |
| Google Gemini Chat Model| LangChain LLM Chat Model       | Alternative LLM for AI                    | -                          | Structured Output Parser   |                                                           |
| Structured Output Parser| LangChain Output Parser       | Parses AI output into strict JSON format | Google Gemini Chat Model    | AI Agent                  |                                                           |
| Update record          | Airtable                       | Updates Airtable lead record with assignment | AI Agent                   | Send a message            |                                                           |
| Send a message         | Slack                         | Sends Slack notification to assigned rep | Update record              | Send a message1           |                                                           |
| Send a message1        | Gmail                         | Sends email notification to assigned rep | Send a message             | Create or update a contact |                                                           |
| Create or update a contact | HubSpot                    | Updates HubSpot contact with assignment info | Send a message1            | No Operation, do nothing  |                                                           |
| No Operation, do nothing| NoOp                          | Ends workflow branch                     | Create or update a contact | -                          |                                                           |
| Sticky Note            | Sticky Note                   | Documentation and workflow overview      | -                          | -                          |                                                           |
| Sticky Note1           | Sticky Note                   | Detailed user guide and benefits         | -                          | -                          |                                                           |
| Sticky Note5           | Sticky Note                   | Author details and contact info          | -                          | -                          |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create HubSpot Trigger Node**  
   - Type: HubSpot Trigger  
   - Credentials: HubSpot Developer API credentials  
   - Configure to trigger on all contact-related events (default eventValues)  
   - Position: Start of workflow  

2. **Add Get a contact Node**  
   - Type: HubSpot node (operation: get contact)  
   - Credentials: HubSpot OAuth2 account  
   - Set contactId to be dynamic from the trigger (`{{$json['vid']}}` or equivalent)  
   - Select properties to retrieve: firstname, lastname, email, company, website, descriptions, mobilephone, jobtitle, team_size, industry, createdate  
   - Connect output from HubSpot Trigger to this node  

3. **Add Code in JavaScript Node**  
   - Extract and normalize lead fields from HubSpot data  
   - Implement safe path accessing with multiple fallbacks for industry and company info  
   - Provide default "N/A" for missing fields  
   - Connect output from Get a contact to this node  

4. **Add Airtable Create a record Node**  
   - Credentials: Airtable OAuth2 Personal Access Token  
   - Configure base and table for "Growify" and "Leads" respectively  
   - Map fields explicitly from cleaned lead JSON: Email, Job Title, Last Name, Team Size, Industry, First Name, Company Name, Description, Phone Number, Record ID (HS), Company Website  
   - Connect output from Code in JavaScript to this node  

5. **Add TeamDatabase Airtable Tool Node**  
   - Credentials: Airtable OAuth2 Personal Access Token  
   - Base: Growify base  
   - Table: Growify Sales Teams  
   - This node will be used as a tool by AI Agent to search for sales teams  

6. **Add Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Set sessionKey to "data"  
   - Set contextWindowLength to 10  
   - This node stores AI conversational context  

7. **Add OpenAI Chat Model Node**  
   - Credentials: OpenAI API key  
   - Model: gpt-4.1-mini (or GPT-4 equivalent)  
   - This node provides LLM capabilities for AI Agent  

8. **Add Google Gemini Chat Model Node**  
   - Credentials: Google PaLM API  
   - This node acts as an alternative LLM for AI Agent  

9. **Add Structured Output Parser Node**  
   - Configure with a JSON schema example for expected AI output format (lead_persona, recommended_team, slack_notification, email_notification)  
   - Enable autoFix to correct minor format issues  

10. **Add AI Agent Node**  
    - Type: LangChain Agent  
    - Input parameters:  
      - Text: JSON-encoded cleaned lead fields (First Name, Last Name, Job Title, Team Size, Industry, Region, Description)  
      - System Message: Detailed prompt specifying strict lead routing logic based on industry, region, capacity, and required output JSON format  
    - Connect AI tools:  
      - ai_tool: TeamDatabase node  
      - ai_memory: Simple Memory node  
      - ai_languageModel: OpenAI Chat Model node and Google Gemini Chat Model node  
      - ai_outputParser: Structured Output Parser node  
    - Connect output from Create a record node to AI Agent as input data  
    - Position this node after Airtable create and AI tools  

11. **Add Airtable Update record Node**  
    - Credentials: Airtable OAuth2  
    - Base/Table: Same as Create a record node (Growify Leads)  
    - Matching column: Email  
    - Update Assigned to, Lead Status, Persona (AI) fields from AI Agent output  
    - Connect AI Agent output to this node  

12. **Add Slack Send a message Node**  
    - Credentials: Slack OAuth2  
    - Channel: Dynamic from AI Agent output recommended_team.slack_channel  
    - Message: Ping contact person and include AI Agent slack_notification.message  
    - Connect output from Update record node  

13. **Add Gmail Send a message1 Node**  
    - Credentials: Gmail OAuth2  
    - To: AI Agent recommended_team.email  
    - Subject and Message: From AI Agent email_notification.subject and email_notification.body  
    - Connect output from Slack Send a message node  

14. **Add HubSpot Create or update a contact Node**  
    - Credentials: HubSpot OAuth2  
    - Email: From Airtable created record Email field  
    - Status: "active"  
    - Additional fields: status with assigned team info, message, persona from AI Agent output  
    - Connect output from Gmail Send a message1 node  

15. **Add No Operation Node**  
    - Connect output from HubSpot create/update contact to No Operation node to finalize the workflow  

16. **Add Sticky Notes**  
    - Create nodes for overview, user guide, benefits, and author info as per documentation needs, placing them conveniently for user reference  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| AI-Powered Lead Routing and Sales Team Distribution automates lead qualification and assignment using HubSpot, Airtable, and GPT-4, improving lead conversion speed and accuracy.                                                                                                                                                                   | Workflow overview sticky note                                        |
| Author: Manish Kumar — Expert Designer & Automation Engineer for Custom Shopify Apps. Contact: manipritraj@gmail.com, +91-9334888899                                                                                                                                                                                                                | Author info sticky note with image                                   |
| Workflow user guide in 5 steps: Capture, Clean, Qualify, Distribute, Sync for fast, scalable lead processing.                                                                                                                                                                                                                                         | User guide sticky note                                               |
| Lead routing logic emphasizes Industry, Region, Team Size, and Team Capacity with strict JSON output format for AI. If no suitable team found, fallback JSON response is returned.                                                                                                                                                                    | System message in AI Agent node                                      |
| Airtable bases: "Growify" for leads and "Growify Sales Teams" for sales team data. Integration via OAuth2 credentials for robust authentication. HubSpot OAuth2 and Slack OAuth2 credentials required for respective APIs. Gmail OAuth2 used for email notifications.                                                                                | Credentials and integrations notes                                  |
| Slack notifications ping assigned contact person in the team’s channel for urgent awareness. Gmail messages deliver detailed lead info to assigned rep. HubSpot is updated to reflect AI assignment and lead status.                                                                                                                                  | Notification flow detailed in node connections                      |
| See Airtable URLs for base and table references: https://airtable.com/appcEbIlJr6Vce1R7 and tables for Leads and Growify Sales Teams.                                                                                                                                                                                                               | Airtable links in node configurations                               |

---

**Disclaimer:**  
The provided content is extracted from a fully automated n8n workflow. It complies strictly with content policies, contains no illegal or offensive material, and handles only legal and public data.