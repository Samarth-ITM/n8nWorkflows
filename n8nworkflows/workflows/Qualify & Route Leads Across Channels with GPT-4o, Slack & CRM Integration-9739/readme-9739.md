Qualify & Route Leads Across Channels with GPT-4o, Slack & CRM Integration

https://n8nworkflows.xyz/workflows/qualify---route-leads-across-channels-with-gpt-4o--slack---crm-integration-9739


# Qualify & Route Leads Across Channels with GPT-4o, Slack & CRM Integration

---
### 1. Workflow Overview

This workflow, titled **"Qualify & Route Leads Across Channels with GPT-4o, Slack & CRM Integration"**, is designed to intelligently qualify incoming leads from multiple input channels (email and web form), enrich and score these leads using AI, and then route the qualified leads to CRM systems and notify sales teams in Slack. It also logs both successful and failed leads for audit and monitoring purposes.

The workflow is logically divided into the following blocks:

- **1.1 Intake & Configuration:** Receives lead data from email or form submissions and centralizes configuration parameters.
- **1.2 Data Validation:** Validates the incoming lead data for completeness and correctness.
- **1.3 AI Extraction:** Uses GPT-4o AI to extract and structure detailed lead information.
- **1.4 Lead Scoring:** Calculates a transparent lead score based on defined Ideal Customer Profile (ICP) criteria.
- **1.5 Routing & CRM Integration:** Routes leads to the appropriate CRM system (HubSpot or Salesforce) based on configuration.
- **1.6 Notification & Logging:** Posts notifications to Slack and logs leads (both successful and failed) to Google Sheets for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Configuration

- **Overview:**  
Receives leads from two sources: Gmail emails and web form submissions via webhook. These inputs are merged and then enriched with workflow-wide configuration parameters to be used downstream.

- **Nodes Involved:**  
  - Email Trigger  
  - Form Submission Webhook  
  - Merge Inputs  
  - Workflow Configuration  
  - Sticky Note1

- **Node Details:**

  - **Email Trigger**  
    - *Type:* Gmail Trigger  
    - *Role:* Polls Gmail inbox every minute to detect new emails that potentially contain lead data.  
    - *Configuration:* No filters applied; triggers on every new email.  
    - *Inputs:* None (trigger node)  
    - *Outputs:* To Merge Inputs node  
    - *Failures:* Connectivity issues, authentication errors with Gmail OAuth2.  
    - *Version:* 1.3

  - **Form Submission Webhook**  
    - *Type:* Webhook  
    - *Role:* Receives HTTP POST requests from web forms containing lead data.  
    - *Configuration:* Fixed webhook path; no authentication configured.  
    - *Inputs:* External HTTP POST requests  
    - *Outputs:* To Merge Inputs node  
    - *Failures:* Missing/incorrect webhook call, malformed payloads  
    - *Version:* 2.1

  - **Merge Inputs**  
    - *Type:* Merge  
    - *Role:* Combines inputs from email and webhook into a single unified stream.  
    - *Configuration:* Default merge mode (likely 'append')  
    - *Inputs:* Email Trigger, Form Submission Webhook  
    - *Outputs:* To Workflow Configuration node  
    - *Failures:* Data format inconsistencies between inputs  
    - *Version:* 3.2

  - **Workflow Configuration**  
    - *Type:* Set  
    - *Role:* Centralizes all workflow parameters such as target industries, minimum company size, buyer roles, due hours, Slack channel ID, Google Sheet URL, and CRM system choice.  
    - *Configuration:* Hardcoded values for these parameters; includes a note prompting user to customize.  
    - *Inputs:* Merge Inputs  
    - *Outputs:* To Validate Input Data node  
    - *Failures:* Misconfiguration could cause faulty lead scoring or routing.  
    - *Version:* 3.4

  - **Sticky Note1**  
    - *Role:* Documentation reminder to update configuration, add Google credentials, and prepare Google Sheets with proper headings.  
    - *Note Content:*  
      ```
      # 🟡 Intake & Configuration

      ## Update workflow configuration node: complete all fields

      ## Add Google credentials

      ## Your Google Sheet needs two headings: Leads, Failed Leads
      ```

---

#### 2.2 Data Validation

- **Overview:**  
Validates incoming lead data for presence of critical fields and proper email formatting. Routes invalid data to a failure logging branch.

- **Nodes Involved:**  
  - Validate Input Data (Code)  
  - Check Validation (If)  
  - Log Validation Failure (Set)  
  - Log Failed Leads (Google Sheets)  
  - Sticky Note2

- **Node Details:**

  - **Validate Input Data**  
    - *Type:* Code  
    - *Role:* Checks if incoming data contains either email or text content fields. Validates email format if present. Ensures text content is not empty.  
    - *Key Expressions:* Regex for email validation; checks on multiple potential text fields (`text`, `body`, `content`).  
    - *Inputs:* Workflow Configuration output (with lead data merged)  
    - *Outputs:* To Check Validation  
    - *Failures:* Incorrect data formats, missing fields cause rejection  
    - *Version:* 2

  - **Check Validation**  
    - *Type:* If  
    - *Role:* Branches flow based on `validation.isValid` boolean. If true, proceeds to AI extraction; else to failure logging.  
    - *Inputs:* Validate Input Data output  
    - *Outputs:* Two outputs — success to Extract Lead Data with AI, failure to Log Validation Failure  
    - *Version:* 2.2

  - **Log Validation Failure**  
    - *Type:* Set  
    - *Role:* Formats validation failure details including status, timestamp, and error messages for logging.  
    - *Inputs:* Check Validation failure output  
    - *Outputs:* To Log Failed Leads  
    - *Version:* 3.4

  - **Log Failed Leads**  
    - *Type:* Google Sheets  
    - *Role:* Appends or updates failed lead entries to a configured Google Sheet for auditing.  
    - *Inputs:* Log Validation Failure  
    - *Outputs:* None (terminal for failure branch)  
    - *Failures:* Google API authentication issues, sheet misconfiguration  
    - *Version:* 4.7

  - **Sticky Note2**  
    - *Role:* Highlights AI extraction & scoring requirements and notes logging of failed leads to Google Sheets.  
    - *Note Content:*  
      ```
      ## 🧠 AI Extraction & Scoring

      ## Enter Open AI credentials 

      ## AI calculates the lead score based on company, size, industry, role, problem, region, budget

      ## Lead scoring (0-100)

      ## Failed lead validations are logged to separate Google Sheet
      ```

---

#### 2.3 AI Extraction

- **Overview:**  
Uses GPT-4o model to extract structured lead details from incoming data. The output is a well-defined JSON object with comprehensive lead fields.

- **Nodes Involved:**  
  - Extract Lead Data with AI (OpenAI)  
  - Parse AI Response (Code)

- **Node Details:**

  - **Extract Lead Data with AI**  
    - *Type:* OpenAI (LangChain)  
    - *Role:* Sends lead input data to GPT-4o-mini with a prompt to extract structured lead information including fields such as firstName, lastName, email, company, jobTitle, industry, budget, timeline, and more.  
    - *Configuration:* Uses GPT-4o-mini model; prompt includes explicit required fields and input data serialized as JSON.  
    - *Inputs:* Check Validation success output  
    - *Outputs:* To Parse AI Response  
    - *Failures:* API rate limits, authentication errors, parsing errors if AI returns malformed data  
    - *Version:* 1.8

  - **Parse AI Response**  
    - *Type:* Code  
    - *Role:* Parses the AI response JSON string or object, normalizes field names (e.g., firstName vs first_name), and returns a structured lead object with fallback defaults.  
    - *Key Expressions:* JSON parsing with fallback regex extraction; normalization of various possible property names.  
    - *Inputs:* Extract Lead Data with AI output  
    - *Outputs:* To Calculate Lead Score  
    - *Failures:* Parsing failures if AI response is malformed or unexpected format  
    - *Version:* 2

---

#### 2.4 Lead Scoring

- **Overview:**  
Calculates a transparent lead scoring rubric (0-100 points) based on ICP criteria configured in the workflow. Categorizes leads into Hot, Warm, Cold, or Unqualified tiers.

- **Nodes Involved:**  
  - Calculate Lead Score (Code)

- **Node Details:**

  - **Calculate Lead Score**  
    - *Type:* Code  
    - *Role:* Scores leads using criteria: Industry Match (30 pts), Company Size (25 pts), Role Match (25 pts), Problem Statement (10 pts), Budget Signals (10 pts). Produces numeric score, tier classification, score breakdown, and reasoning notes.  
    - *Key Expressions:* Uses configuration node parameters for target industries and buyer roles; analyzes message content for problem and budget signals.  
    - *Inputs:* Parse AI Response output  
    - *Outputs:* To Route by Territory  
    - *Failures:* Missing configuration parameters could skew scoring; unexpected data formats in lead fields  
    - *Version:* 2

---

#### 2.5 Routing & CRM Integration

- **Overview:**  
Routes scored leads to either HubSpot or Salesforce CRM based on configuration. Each CRM node creates a contact or lead with enriched data including lead score and region. Afterwards, leads are posted to Slack for sales notification.

- **Nodes Involved:**  
  - Route by Territory (Switch)  
  - Create HubSpot Contact  
  - Create Salesforce Lead  
  - Post to Slack  
  - Sticky Note - Routing

- **Node Details:**

  - **Route by Territory**  
    - *Type:* Switch  
    - *Role:* Routes leads based on the configured CRM system (hubspot or salesforce). If none matches, routes to fallback output (extra).  
    - *Inputs:* Calculate Lead Score output  
    - *Outputs:* HubSpot branch, Salesforce branch, fallback  
    - *Version:* 3.3

  - **Create HubSpot Contact**  
    - *Type:* HubSpot  
    - *Role:* Creates or updates a contact in HubSpot CRM with lead details including first name, email, company, and custom properties (lead score, region).  
    - *Inputs:* Route by Territory output for HubSpot branch  
    - *Outputs:* To Post to Slack  
    - *Failures:* Authentication errors, API limits, missing required fields  
    - *Version:* 2.2

  - **Create Salesforce Lead**  
    - *Type:* Salesforce  
    - *Role:* Creates a new lead in Salesforce with fields like company name, last name (mapped to role), email, lead source, and description (problem statement).  
    - *Inputs:* Route by Territory output for Salesforce branch  
    - *Outputs:* To Post to Slack  
    - *Failures:* Authentication errors, API limits, field mapping issues  
    - *Version:* 1

  - **Post to Slack**  
    - *Type:* Slack  
    - *Role:* Posts a rich block message to a configured Slack channel summarizing the new qualified lead, score, reasoning, and next best action with an interactive button to create an intro email.  
    - *Inputs:* Outputs from both HubSpot and Salesforce nodes  
    - *Outputs:* To Log to Google Sheets  
    - *Failures:* Slack API authentication errors, channel permissions  
    - *Version:* 2.3

  - **Sticky Note - Routing**  
    - *Role:* Instruction to add credentials for HubSpot/Salesforce and Slack connections.  
    - *Note Content:*  
      ```
      ## 📨 Routing & Notifications

      ## Add credentials for HubSpot or Salesforce

      ## Add Slack credentials
      ```

---

#### 2.6 Notification & Logging

- **Overview:**  
Logs qualified leads into Google Sheets for record-keeping and auditing, enabling further analysis or reporting.

- **Nodes Involved:**  
  - Log to Google Sheets

- **Node Details:**

  - **Log to Google Sheets**  
    - *Type:* Google Sheets  
    - *Role:* Appends or updates the qualified lead data into a configured Google Sheet specified in the workflow configuration.  
    - *Inputs:* Post to Slack output  
    - *Outputs:* None (terminal node)  
    - *Failures:* Google API credentials or permissions issues, spreadsheet misconfiguration  
    - *Version:* 4.7

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                         | Input Node(s)                  | Output Node(s)                   | Sticky Note                                   |
|-------------------------|---------------------------|---------------------------------------|-------------------------------|---------------------------------|-----------------------------------------------|
| Email Trigger           | Gmail Trigger             | Lead intake via email                  | None                          | Merge Inputs                    |                                               |
| Form Submission Webhook | Webhook                   | Lead intake via form submission       | None                          | Merge Inputs                    |                                               |
| Merge Inputs            | Merge                     | Combines email and form lead inputs   | Email Trigger, Form Submission Webhook | Workflow Configuration         |                                               |
| Workflow Configuration  | Set                       | Centralized workflow parameters       | Merge Inputs                  | Validate Input Data             | # 🟡 Intake & Configuration...                 |
| Validate Input Data     | Code                      | Validates required lead fields        | Workflow Configuration        | Check Validation               | ## 🧠 AI Extraction & Scoring...                 |
| Check Validation        | If                        | Branches on validation pass/fail      | Validate Input Data           | Extract Lead Data with AI, Log Validation Failure | ## 🧠 AI Extraction & Scoring...                 |
| Log Validation Failure  | Set                       | Formats validation failure info       | Check Validation (fail)       | Log Failed Leads               | ## 🧠 AI Extraction & Scoring...                 |
| Log Failed Leads        | Google Sheets             | Logs failed leads                     | Log Validation Failure        | None                          | ## 🧠 AI Extraction & Scoring...                 |
| Extract Lead Data with AI| OpenAI (LangChain)        | AI extraction of structured lead data | Check Validation (pass)       | Parse AI Response             | ## 🧠 AI Extraction & Scoring...                 |
| Parse AI Response       | Code                      | Parses AI output to structured fields | Extract Lead Data with AI     | Calculate Lead Score          |                                               |
| Calculate Lead Score    | Code                      | Calculates lead score and tier        | Parse AI Response             | Route by Territory            |                                               |
| Route by Territory      | Switch                    | Routes leads to configured CRM        | Calculate Lead Score          | Create HubSpot Contact, Create Salesforce Lead | ## 📨 Routing & Notifications...                |
| Create HubSpot Contact  | HubSpot                   | Creates/updates contact in HubSpot    | Route by Territory (HubSpot)  | Post to Slack                 | ## 📨 Routing & Notifications...                |
| Create Salesforce Lead  | Salesforce                | Creates lead in Salesforce             | Route by Territory (Salesforce)| Post to Slack                 | ## 📨 Routing & Notifications...                |
| Post to Slack           | Slack                     | Notifies team of new qualified lead   | Create HubSpot Contact, Create Salesforce Lead | Log to Google Sheets          | ## 📨 Routing & Notifications...                |
| Log to Google Sheets    | Google Sheets             | Logs qualified leads                   | Post to Slack                | None                         |                                               |
| Sticky Note1            | Sticky Note               | Configuration and intake instructions | None                          | None                          | # 🟡 Intake & Configuration...                 |
| Sticky Note2            | Sticky Note               | AI extraction & scoring instructions  | None                          | None                          | ## 🧠 AI Extraction & Scoring...                 |
| Sticky Note - Routing   | Sticky Note               | Routing & notification instructions   | None                          | None                          | ## 📨 Routing & Notifications...                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Email Trigger Node:**  
   - Type: Gmail Trigger  
   - Configure to poll every minute without filters.  
   - Add Gmail OAuth2 credentials.  

2. **Create Form Submission Webhook:**  
   - Type: Webhook  
   - Set path to a unique identifier (e.g., "648db646-76c1-44b4-bab0-5955971721e5").  
   - No authentication configured but secure webhook endpoint recommended.  

3. **Create Merge Inputs Node:**  
   - Type: Merge  
   - Connect Email Trigger to first input, Webhook to second input.  
   - Default merge mode (append).  

4. **Create Workflow Configuration Node:**  
   - Type: Set  
   - Assign variables:  
     - targetIndustries: "technology,software,saas,fintech,healthcare,manufacturing,retail,ecommerce"  
     - minCompanySize: "100"  
     - buyerRoles: "ceo,cto,cfo,vp,director,head,manager,founder,owner"  
     - taskDueHours: "24"  
     - slackChannel: "C1234567890" (replace with your Slack channel ID)  
     - googleSheetUrl: "https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit" (replace with your Google Sheet URL)  
     - crmSystem: "hubspot" or "salesforce"  
     - note: Instruction for setup  
   - Connect Merge Inputs to this node.  

5. **Create Validate Input Data (Code) Node:**  
   - Type: Code  
   - Implement validation logic checking presence of email or text content, valid email format, and non-empty text content.  
   - Connect Workflow Configuration output here.  

6. **Create Check Validation (If) Node:**  
   - Type: If  
   - Condition: `{{$json.validation.isValid}} === true`  
   - Connect Validate Input Data output here.  

7. **Create Log Validation Failure (Set) Node:**  
   - Type: Set  
   - Assign: status="validation_failed", timestamp=current ISO time, errorMessage=validation errors, note="Formats validation failure data for logging".  
   - Connect Check Validation 'false' output here.  

8. **Create Log Failed Leads (Google Sheets) Node:**  
   - Type: Google Sheets  
   - Operation: Append or update  
   - Document ID: Use `googleSheetUrl` from configuration (extract file ID)  
   - Sheet Name: Name of the sheet for failed leads  
   - Connect Log Validation Failure output here.  
   - Add Google Sheets credentials.  

9. **Create Extract Lead Data with AI Node:**  
   - Type: OpenAI (LangChain)  
   - Model: gpt-4o-mini  
   - Prompt: Provide structured JSON extraction instructions including all required lead fields.  
   - Connect Check Validation 'true' output here.  
   - Add OpenAI credentials.  

10. **Create Parse AI Response (Code) Node:**  
    - Parse AI response string or JSON object into normalized lead data fields.  
    - Connect Extract Lead Data with AI output here.  

11. **Create Calculate Lead Score (Code) Node:**  
    - Implement lead scoring based on industry, company size, role, problem statement, and budget signals.  
    - Use parameters from Workflow Configuration for target industries and buyer roles.  
    - Connect Parse AI Response output here.  

12. **Create Route by Territory (Switch) Node:**  
    - Condition: Check `crmSystem` from Workflow Configuration node.  
    - Routes:  
      - If "hubspot", to HubSpot contact creation node.  
      - If "salesforce", to Salesforce lead creation node.  
      - Else, fallback output (optional).  
    - Connect Calculate Lead Score output here.  

13. **Create Create HubSpot Contact Node:**  
    - Type: HubSpot  
    - Map fields: email, firstName, companyName, custom properties for lead_score and region.  
    - Connect Route by Territory HubSpot output here.  
    - Add HubSpot credentials.  

14. **Create Create Salesforce Lead Node:**  
    - Type: Salesforce  
    - Map fields: company, lastname (role), email, leadSource ("Web"), description (problem statement).  
    - Connect Route by Territory Salesforce output here.  
    - Add Salesforce credentials.  

15. **Create Post to Slack Node:**  
    - Type: Slack  
    - Set channel ID from Workflow Configuration slackChannel variable.  
    - Compose block message with companyName, leadScore, score breakdown, next best action, and an interactive button labeled "Create intro email".  
    - Connect both HubSpot and Salesforce nodes outputs here.  
    - Add Slack credentials.  

16. **Create Log to Google Sheets Node:**  
    - Type: Google Sheets  
    - Operation: Append or update  
    - Document ID: Use googleSheetUrl from configuration  
    - Sheet Name: Name of sheet for qualified leads  
    - Connect Post to Slack output here.  
    - Add Google Sheets credentials.  

17. **Add Sticky Notes:**  
    - Add three sticky notes with content as per original workflow for documentation and guidance in Intake & Configuration, AI Extraction & Scoring, and Routing & Notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Setup required: Configure Google Sheets with two tabs named "Leads" and "Failed Leads".                   | Workflow data logging                              |
| Ensure Google, Slack, HubSpot, Salesforce, and OpenAI credentials are set up correctly and authorized.    | Integration requirements                           |
| Lead scoring rubric is transparent and based on configurable ICP parameters: industry, size, role, budget | Customizable lead qualification criteria          |
| Slack message includes interactive button for quick sales actions.                                        | Sales team notification and engagement             |
| Workflow designed to be extensible for multi-channel lead intake and routing.                             | Scalable lead management                           |

---

**Disclaimer:** The text above is exclusively generated from an automated n8n workflow. It complies fully with content policies, contains no illegal or protected content, and all handled data is lawful and public.