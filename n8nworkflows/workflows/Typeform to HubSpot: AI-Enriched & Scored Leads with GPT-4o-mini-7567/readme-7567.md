Typeform to HubSpot: AI-Enriched & Scored Leads with GPT-4o-mini

https://n8nworkflows.xyz/workflows/typeform-to-hubspot--ai-enriched---scored-leads-with-gpt-4o-mini-7567


# Typeform to HubSpot: AI-Enriched & Scored Leads with GPT-4o-mini

---

### 1. Workflow Overview

This workflow, titled **"Typeform to HubSpot: AI-Enriched & Scored Leads with GPT-4o-mini"** (internally named "13 - SmartScore Enrichment Engine"), automates the process of capturing leads from a Typeform survey, enriching those leads with AI-driven company information, scoring their sales potential, and syncing enriched data into multiple systems including HubSpot CRM, Airtable, and Google Sheets.

**Target Use Cases:**  
- Businesses collecting lead data via Typeform who want to enrich leads with company metadata automatically.  
- Sales and marketing teams who want an AI-powered lead scoring mechanism to prioritize follow-ups.  
- Organizations seeking centralized, consistent lead data storage and backup across Airtable, HubSpot, and Google Sheets.

**Logical Blocks:**  
- **1.1 Input Reception & Raw Data Logging:** Captures new Typeform leads via webhook; formats and logs raw data.  
- **1.2 Data Storage & Filtering:** Stores lead basics in Airtable; filters out non-business (gmail.com) emails.  
- **1.3 AI Company Enrichment:** Uses GPT-4o-mini to enrich lead company info based on email domain.  
- **1.4 Data Merging:** Combines raw lead data with AI enrichment results and adds metadata.  
- **1.5 AI Lead Scoring:** Scores the enriched lead profile for sales potential.  
- **1.6 CRM & Data Sync:** Sends enriched, scored leads to HubSpot CRM and saves enriched data in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Raw Data Logging

- **Overview:**  
Receives incoming leads from Typeform using a webhook trigger, formats the incoming JSON payload into a normalized structure, and logs the raw lead data into a Google Sheet as a backup.

- **Nodes Involved:**  
  - 📝 New Typeform Lead (Typeform Trigger)  
  - 🔧 Format Incoming Data (Code)  
  - 📄 Log Raw Lead (Google Sheets)  

- **Node Details:**

  - **📝 New Typeform Lead**  
    - Type: Typeform Trigger  
    - Role: Listens for new form submissions on a configured Typeform form via webhook.  
    - Config: Form ID must be specified; uses Typeform API credentials.  
    - Inputs: External webhook events.  
    - Outputs: Raw Typeform response JSON.  
    - Edge Cases: Webhook misconfiguration, credential expiry, missing form ID errors.

  - **🔧 Format Incoming Data**  
    - Type: Code (JavaScript)  
    - Role: Normalizes incoming payload to a standard lead object with fields: name, email, phone, message, domain, source.  
    - Key Logic: Extracts email domain; supports multiple input formats (Typeform direct object, array, Calendly payload).  
    - Inputs: Raw JSON from Typeform trigger.  
    - Outputs: Formatted lead JSON.  
    - Edge Cases: Unsupported input format detected; graceful fallback with error JSON.  
    - Version: Code node v2.

  - **📄 Log Raw Lead**  
    - Type: Google Sheets  
    - Role: Appends or updates raw lead data into a Google Sheet for archival.  
    - Config: Maps Name, Email, Message, Phone Number columns; uses Google Sheets OAuth2 credentials.  
    - Inputs: Output from Format Incoming Data.  
    - Outputs: Confirmation of append/update operation.  
    - Edge Cases: API quota limits, sheet access permissions, network errors.

---

#### 1.2 Data Storage & Filtering

- **Overview:**  
Stores basic lead information in Airtable to maintain a deduplicated lead list by email; filters out leads with gmail.com (personal) email domains to process only business leads downstream.

- **Nodes Involved:**  
  - 📦 Store Basic Info (Airtable)  
  - 📛 Filter Non-Business Emails (If)

- **Node Details:**

  - **📦 Store Basic Info (Airtable)**  
    - Type: Airtable  
    - Role: Upserts lead data keyed by Email to prevent duplicates.  
    - Config: Uses Airtable OAuth2 credentials; maps Name, Email, Message, Phone Number.  
    - Inputs: Raw lead JSON (formatted).  
    - Outputs: Airtable record upsert confirmation.  
    - Edge Cases: Credential expiration, rate limits, table/field mapping errors.

  - **📛 Filter Non-Business Emails**  
    - Type: If (Conditional filter)  
    - Role: Filters out emails with domain exactly equal to "gmail.com".  
    - Logic: Condition `domain != "gmail.com"`.  
    - Inputs: Formatted lead JSON.  
    - Outputs: Passes only leads with non-gmail.com domains to next block.  
    - Edge Cases: Missing or malformed domain field could cause unexpected filtering.

---

#### 1.3 AI Company Enrichment

- **Overview:**  
Enriches lead information by querying the GPT-4o-mini model to fetch company metadata based on the lead’s email domain.

- **Nodes Involved:**  
  - 💬 LLM (OpenAI) (Language Model node)  
  - 🏢 Enrich Company Info (AI) (Langchain Agent)

- **Node Details:**

  - **💬 LLM (OpenAI)**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4o-mini LLM instance for downstream AI agent.  
    - Config: Model set to gpt-4o-mini; uses OpenAI API credentials.  
    - Inputs: None (provides a reusable LLM resource).  
    - Outputs: LLM object for agent use.  
    - Edge Cases: API quota exhaustion, invalid credentials.

  - **🏢 Enrich Company Info (AI)**  
    - Type: Langchain Agent (AI Agent node)  
    - Role: Sends prompt to GPT-4o-mini to extract company details (name, industry, HQ, employees, website, LinkedIn, description) from domain.  
    - Prompt: Detailed instructions to return JSON with specified fields only.  
    - Inputs: Domain from filtered lead JSON.  
    - Outputs: JSON-formatted company metadata or fallback error info.  
    - Edge Cases: AI model misunderstanding or malformed JSON output; network/API failures.

---

#### 1.4 Data Merging

- **Overview:**  
Combines the original lead data with the AI-enriched company info into a single enriched lead profile, adding metadata like enrichment timestamp and workflow ID.

- **Nodes Involved:**  
  - 🪄 Merge Enriched Lead Profile (Code)

- **Node Details:**

  - **🪄 Merge Enriched Lead Profile**  
    - Type: Code (JavaScript)  
    - Role: Reads data from the filter node and AI enrichment node; parses AI JSON output carefully, merges both into a unified lead object with fallback defaults.  
    - Inputs: Output from 📛 Filter Non-Business Emails and 🏢 Enrich Company Info (AI).  
    - Outputs: Enriched lead JSON with fields including company_name, industry, description, enriched_at timestamp, and workflow ID.  
    - Edge Cases: Parsing failures of AI output; missing lead data; runtime errors in code node.

---

#### 1.5 AI Lead Scoring

- **Overview:**  
Uses GPT-4o-mini to score the enriched lead for sales potential on a scale of 1 to 10, considering company metadata and contact details.

- **Nodes Involved:**  
  - 🎯 AI Lead Scorer (Langchain Agent)  
  - OpenAI Chat Model (Langchain OpenAI Chat Model)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Supplies GPT-4o-mini model for scoring agent.  
    - Config: Uses OpenAI API credentials.  
    - Inputs: None (LLM provider).  
    - Outputs: LLM instance.  
    - Edge Cases: Same as previous LLM node.

  - **🎯 AI Lead Scorer**  
    - Type: Langchain Agent  
    - Role: Generates numeric lead score (1–10) based on enriched lead profile and company metadata.  
    - Prompt: Explicit instructions to consider company size, industry relevance, domain reputation, and return only the score.  
    - Inputs: Enriched lead JSON from merge node.  
    - Outputs: Numeric lead score as string or number.  
    - Edge Cases: Model output formatting issues, empty or incomplete input.

---

#### 1.6 CRM & Data Sync

- **Overview:**  
Sends the enriched and scored lead data to HubSpot CRM, then logs the enriched lead data including the score into a Google Sheet for reporting or further analysis.

- **Nodes Involved:**  
  - 📨 Send to HubSpot CRM (HubSpot node)  
  - 📊 Save Enriched Lead (Google Sheets)

- **Node Details:**

  - **📨 Send to HubSpot CRM**  
    - Type: HubSpot  
    - Role: Creates or updates contact record in HubSpot CRM with enriched details and lead score.  
    - Config: Uses HubSpot App Token; maps all relevant fields including custom properties (lead_score, company LinkedIn, description).  
    - Inputs: Enriched lead JSON and lead score from scoring node.  
    - Outputs: HubSpot API response.  
    - Edge Cases: API authentication failure, field mapping mismatches, rate limits.

  - **📊 Save Enriched Lead**  
    - Type: Google Sheets  
    - Role: Appends or updates enriched lead data and score into a dedicated Google Sheet for enriched leads.  
    - Config: Maps lead and company fields plus score; uses Google Sheets OAuth2 credentials.  
    - Inputs: Data from merge node and AI Lead Scorer.  
    - Outputs: Sheet append/update confirmation.  
    - Edge Cases: Sheet access issues, quota, formatting errors.

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                               | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                                                             |
|------------------------------|-------------------------------|-----------------------------------------------|------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| 📝 New Typeform Lead          | Typeform Trigger              | Capture new lead data from Typeform form     | N/A                                | 📄 Log Raw Lead, 📦 Store Basic Info, 🔧 Format Incoming Data | **Triggered on new Typeform submission. Captures: Name, Email, Phone, Message.**                                                       |
| 🔧 Format Incoming Data       | Code                         | Normalize and extract key lead fields         | 📝 New Typeform Lead                | 📛 Filter Non-Business Emails      | **Cleans data from Typeform. Extracts key fields, tags source, checks if email is business.**                                           |
| 📄 Log Raw Lead               | Google Sheets                | Backup raw lead data                           | 📝 New Typeform Lead                | N/A                               | **Saves raw lead data to Google Sheet for backup.**                                                                                      |
| 📦 Store Basic Info (Airtable) | Airtable                     | Upsert basic lead info keyed by Email         | 📝 New Typeform Lead                | N/A                               | **Upserts lead into Airtable by Email to prevent duplicates.**                                                                           |
| 📛 Filter Non-Business Emails | If                           | Filter out gmail.com personal emails           | 🔧 Format Incoming Data             | 🏢 Enrich Company Info (AI)        | **Filters out Gmail-based or personal emails. Allows only leads with non gmail.com (business) domains to proceed for enrichment.**       |
| 💬 LLM (OpenAI)              | Langchain OpenAI Chat Model  | Provide GPT-4o-mini LLM instance for AI nodes | N/A                                | 🏢 Enrich Company Info (AI)        | **Supplies the GPT-4o-mini model to the downstream agent node.**                                                                         |
| 🏢 Enrich Company Info (AI)   | Langchain Agent              | Enrich company info from domain using AI       | 📛 Filter Non-Business Emails       | 🪄 Merge Enriched Lead Profile     | **AI agent enriches company data using domain: company_name, industry, headquarters, etc.**                                              |
| 🪄 Merge Enriched Lead Profile | Code                         | Combine raw lead and AI enriched company data | 🏢 Enrich Company Info (AI), 📛 Filter Non-Business Emails | 🎯 AI Lead Scorer                  | **Merges formatted lead + enriched company data. Adds metadata: enrichment timestamp & workflow ID.**                                  |
| 🎯 AI Lead Scorer             | Langchain Agent              | Score lead potential based on enriched profile | 🪄 Merge Enriched Lead Profile      | 📨 Send to HubSpot CRM             | **Evaluates lead quality based on company metadata, industry fit, contact source, domain type. Returns a lead score (1–10).**           |
| OpenAI Chat Model             | Langchain OpenAI Chat Model  | Provide GPT-4o-mini LLM instance for scoring   | N/A                                | 🎯 AI Lead Scorer                  |                                                                                                                                         |
| 📨 Send to HubSpot CRM        | HubSpot                      | Create/update contact in HubSpot CRM           | 🎯 AI Lead Scorer                  | 📊 Save Enriched Lead              | **Creates or updates contact in HubSpot with enriched data and lead score.**                                                             |
| 📊 Save Enriched Lead         | Google Sheets                | Log enriched and scored lead into Google Sheet | 📨 Send to HubSpot CRM             | N/A                               | **Logs fully enriched and scored lead into Company's Google Spreadsheet. Columns include contact info, company info, and lead score.**  |
| Sticky Note                  | Sticky Note                  | Explanatory comments                            | N/A                                | N/A                               | See above notes in Sticky Note content for contextual explanations.                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create New Workflow** in n8n, name it "13 - SmartScore Enrichment Engine".

2. **Add Typeform Trigger Node**  
   - Type: Typeform Trigger  
   - Configure: Enter your Typeform Form ID.  
   - Credentials: Set up and select Typeform API Credential.  
   - Position: Left side (e.g., X:-500, Y:440).

3. **Add Google Sheets Node (Log Raw Lead)**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Configure columns: Name, Email, Message, Phone Number mapped from Typeform fields.  
   - Set Sheet Name and Document ID for raw leads backup sheet.  
   - Credentials: Google Sheets OAuth2 API credential.  
   - Connect: Typeform Trigger main output → Log Raw Lead input.

4. **Add Airtable Node (Store Basic Info)**  
   - Type: Airtable  
   - Operation: Upsert by Email  
   - Map fields: Name, Email, Message, Phone Number from Typeform.  
   - Configure Airtable Base ID and Table ID.  
   - Credentials: Airtable OAuth2 API credential.  
   - Connect: Typeform Trigger main output → Airtable node input.

5. **Add Code Node (Format Incoming Data)**  
   - Type: Code (JavaScript) v2  
   - Paste provided JS code to normalize input and extract domain.  
   - Connect: Typeform Trigger main output → Format Incoming Data input.

6. **Add If Node (Filter Non-Business Emails)**  
   - Type: If  
   - Condition: `{{$json.domain}} != "gmail.com"`  
   - Connect: Format Incoming Data main output → If node input.

7. **Add Langchain OpenAI Chat Model Node (LLM for Enrichment)**  
   - Type: Langchain OpenAI Chat Model  
   - Model: gpt-4o-mini  
   - Credentials: OpenAI API Credential  
   - Position near enrichment agent.

8. **Add Langchain Agent Node (Enrich Company Info)**  
   - Type: Langchain Agent  
   - Prompt: Provide domain and ask for company JSON info with fields company_name, industry, headquarters, employee_count, website, linkedin, description.  
   - Connect: If node "true" output → Enrich Company Info input.  
   - Use LLM node as AI resource input.

9. **Add Code Node (Merge Enriched Lead Profile)**  
   - Type: Code (JavaScript) v2  
   - Paste provided merging code to combine lead and AI data with metadata.  
   - Connect: Enrich Company Info output → Merge node input.  
   - Connect: If node "true" output (lead data) → Merge node input (multi-input).

10. **Add Langchain OpenAI Chat Model Node (LLM for Scoring)**  
    - Similar config as enrichment LLM node; separate instance for scoring.  
    - Credentials: OpenAI API Credential.

11. **Add Langchain Agent Node (AI Lead Scorer)**  
    - Type: Langchain Agent  
    - Prompt: Provide enriched lead profile; request numeric score 1–10.  
    - Connect: Merge Enriched Lead Profile output → AI Lead Scorer input.  
    - Use scoring LLM node as AI resource input.

12. **Add HubSpot Node (Send to HubSpot CRM)**  
    - Type: HubSpot  
    - Configure fields: Map enriched lead fields including custom properties (lead_score, linkedin, description).  
    - Credentials: HubSpot API Credential (App Token).  
    - Connect: AI Lead Scorer output → HubSpot node input.

13. **Add Google Sheets Node (Save Enriched Lead)**  
    - Type: Google Sheets  
    - Operation: Append or Update  
    - Map all enriched lead fields and score to columns.  
    - Set Sheet Name and Document ID for enriched leads sheet.  
    - Credentials: Google Sheets OAuth2 API Credential.  
    - Connect: HubSpot node output → Google Sheets enriched lead input.

14. **Add Sticky Notes**  
    - Add explanatory sticky notes near each logical block for clarity and documentation.

15. **Test the Workflow**  
    - Trigger Typeform submission to verify end-to-end functionality.  
    - Check logs and outputs in Airtable, HubSpot, and Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Filters out personal Gmail addresses to enrich only business leads, improving sales data quality.                                | Sticky Note near 📛 Filter Non-Business Emails node                                                  |
| Uses GPT-4o-mini model from OpenAI to enrich company data and score leads efficiently with AI assistance.                        | Sticky Notes near 💬 LLM (OpenAI), 🏢 Enrich Company Info, and 🎯 AI Lead Scorer nodes                |
| Lead data is stored redundantly in Airtable and Google Sheets to ensure durability and ease of access for different teams.      | Sticky Notes near Airtable and Google Sheets nodes                                                  |
| HubSpot integration sends comprehensive enriched lead profile including custom properties for advanced CRM workflows.           | Sticky Note near 📨 Send to HubSpot CRM node                                                         |
| Source code and logic for data formatting and merging is implemented in JavaScript Code nodes for flexibility and transparency.| See 🔧 Format Incoming Data and 🪄 Merge Enriched Lead Profile code nodes                            |

---

**Disclaimer:**  
The text provided is exclusively extracted and analyzed from an n8n automation workflow. It adheres strictly to current content policies and contains no illegal or offensive elements. All data handled is legal and public.

---