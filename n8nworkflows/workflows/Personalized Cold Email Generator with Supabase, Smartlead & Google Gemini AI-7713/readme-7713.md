Personalized Cold Email Generator with Supabase, Smartlead & Google Gemini AI

https://n8nworkflows.xyz/workflows/personalized-cold-email-generator-with-supabase--smartlead---google-gemini-ai-7713


# Personalized Cold Email Generator with Supabase, Smartlead & Google Gemini AI

### 1. Workflow Overview

This workflow automates the generation of personalized cold emails using AI-driven lead qualification, profiling, and outreach content creation, integrated with Supabase for lead data management and Smartlead for campaign management. It targets sales and marketing teams aiming to scale personalized outbound email campaigns with AI-generated insights and messages.

The workflow is logically organized into these functional blocks:

- **1.1 Data Input & Scheduling:** Scheduled trigger initiates the workflow, fetching lead data from Supabase.
- **1.2 Lead Profiling (Digital Detective):** AI analyzes extensive lead and company data to create a detailed psychological and professional profile.
- **1.3 Lead Qualification & Scoring:** AI scores leads for fit and opportunity, outputting a numeric lead score and detailed justification.
- **1.4 Lead Scoring Threshold Check:** Logic checks if the lead score meets a minimum threshold to proceed.
- **1.5 Email Generation:** AI crafts ultra-personalized cold email and follow-up sequences based on profiling and qualification insights.
- **1.6 Data Updates:** Updates the Supabase lead records with profiling, scoring, and email content.
- **1.7 Campaign Creation & Lead Addition:** Creates a new Smartlead campaign and adds qualified leads with their personalized emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Input & Scheduling

- **Overview:** This block triggers the workflow on a schedule and fetches all lead records from Supabase for processing.
- **Nodes Involved:** `Schedule Trigger`, `Get many rows`, `Loop Over Items1`
- **Node Details:**

  - **Schedule Trigger**
    - *Type & Role:* Trigger node that starts the workflow at set intervals (default interval unspecified).
    - *Config:* Uses interval-based schedule with no specific time constraints.
    - *Connections:* Outputs to `Get many rows`.
    - *Edge Cases:* Workflow won't trigger if scheduling is misconfigured; large data fetches may cause delays.

  - **Get many rows**
    - *Type & Role:* Supabase node fetching all rows from the "Leads" table.
    - *Config:* Operation: getAll; returns all lead records.
    - *Credentials:* Requires Supabase API credentials.
    - *Connections:* Outputs to `Loop Over Items1`.
    - *Edge Cases:* Large datasets may cause timeouts; API errors if credentials are invalid.

  - **Loop Over Items1**
    - *Type & Role:* Splits lead data into individual items for batch processing.
    - *Config:* Default batch size; processes leads one by one downstream.
    - *Connections:* Feeds into `Aggregate7` and `Aggregate8`.
    - *Edge Cases:* Batch size can impact performance; node failure can halt batch processing.

#### 2.2 Lead Profiling (Digital Detective)

- **Overview:** Uses AI to analyze detailed lead and company data to generate a psychological and professional profile report.
- **Nodes Involved:** `Aggregate8`, `Google Gemini Chat Model3`, `Basic LLM Chain7`, `Update a row`
- **Node Details:**

  - **Aggregate8**
    - *Type & Role:* Aggregates batch items back into a single dataset.
    - *Config:* Aggregates all item data.
    - *Connections:* Outputs to `Basic LLM Chain7`.
    - *Edge Cases:* Data aggregation errors if input format is inconsistent.

  - **Google Gemini Chat Model3**
    - *Type & Role:* AI language model node powered by Google Gemini to generate lead profiling text.
    - *Config:* Model: "models/gemini-1.5-flash"; no additional options set.
    - *Credentials:* Requires Google Palm API credentials.
    - *Input:* Receives aggregated lead data.
    - *Output:* Profiling text.
    - *Edge Cases:* API rate limits, network errors, malformed input data.

  - **Basic LLM Chain7**
    - *Type & Role:* Defines prompt structure and instructions for AI to produce detailed psychological profiles.
    - *Config:* Complex prompt with detailed instructions referencing multiple lead data fields.
    - *Input:* Text from aggregated data.
    - *Output:* AI-generated profiling report text.
    - *Edge Cases:* Prompt misconfiguration can lead to irrelevant or incomplete profiles.

  - **Update a row**
    - *Type & Role:* Updates lead record in Supabase with generated profiling report.
    - *Config:* Filters leads by email; updates "LEAD PROFILING" field.
    - *Credentials:* Supabase API.
    - *Edge Cases:* Update failures if lead email not found or API errors.

#### 2.3 Lead Qualification & Scoring

- **Overview:** AI evaluates lead fit and opportunity, scoring leads numerically and providing detailed justifications.
- **Nodes Involved:** `Aggregate6`, `Google Gemini Chat Model1`, `Basic LLM Chain5`, `Information Extractor2`, `Update a row1`, `Code`, `If7`
- **Node Details:**

  - **Aggregate6**
    - Aggregates lead data for batch processing.

  - **Google Gemini Chat Model1**
    - AI model generating lead qualification text based on detailed scoring criteria.

  - **Basic LLM Chain5**
    - Prompt defining scoring rules, data sources, scoring criteria, and output format.

  - **Information Extractor2**
    - Extracts structured data (scores, verdicts, detailed breakdown) from AI's textual response.

  - **Update a row1**
    - Updates Supabase lead records with lead score, verdict, and detailed scoring breakdown.

  - **Code**
    - Custom JavaScript that checks the lead score and applies a threshold of 60 if the score is below that or missing.
    - Logs parsing results for diagnostics.

  - **If7**
    - Conditional node that checks if the computed lead score threshold is greater than 40 to continue processing.
    - Directs leads with scores above threshold to continue, others stop.

  - *Edge Cases:* Parsing errors in Code node, AI output format changes breaking extractor, API or update failures.

#### 2.4 Email Generation

- **Overview:** AI generates personalized cold email and follow-up email sequences for leads who pass the scoring threshold.
- **Nodes Involved:** `Aggregate5`, `Google Gemini Chat Model6`, `Basic LLM Chain6`, `Information Extractor3`, `Update a row2`
- **Node Details:**

  - **Aggregate5**
    - Aggregates qualified lead data for the email generation step.

  - **Google Gemini Chat Model6**
    - AI model to generate email content based on lead data and qualification context.

  - **Basic LLM Chain6**
    - Prompt with specific instructions for writing ultra-short personalized cold emails and follow-ups with strict formatting.

  - **Information Extractor3**
    - Extracts structured email content lines and subject lines from AI response.

  - **Update a row2**
    - Updates Supabase lead records with generated email subject lines and body lines for initial and follow-up emails.

  - *Edge Cases:* AI generation failures, structured extraction errors, update failures.

#### 2.5 Campaign Creation & Lead Addition (Smartlead Integration)

- **Overview:** Creates a new campaign in Smartlead and adds qualified leads with their personalized emails to the campaign.
- **Nodes Involved:** `Aggregate7`, `smatlead-create-campaign`, `Loop Over Items9`, `smartlead-add-leads-to-campaign`
- **Node Details:**

  - **Aggregate7**
    - Aggregates processed lead items before campaign creation.

  - **smatlead-create-campaign**
    - HTTP POST request to Smartlead API to create a campaign.
    - Requires API key and campaign name (hardcoded or parameterized).
    - Outputs campaign ID.

  - **Loop Over Items9**
    - Batch processing node to loop over leads for addition to campaign.

  - **smartlead-add-leads-to-campaign**
    - HTTP POST request to add leads to the created campaign using the campaign ID.
    - Sends detailed lead info including personalized email content.
    - Requires Smartlead API key.
    - Supports dynamic payload including custom fields like email lines.

  - *Edge Cases:* API authentication failures, rate limiting, payload formatting errors, campaign creation failures.

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                        | Input Node(s)              | Output Node(s)                  | Sticky Note                                           |
|-----------------------------|----------------------------------------|-------------------------------------|---------------------------|-------------------------------|------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                       | Triggers workflow on schedule        | -                         | Get many rows                 |                                                      |
| Get many rows              | Supabase                              | Fetch all leads from Supabase        | Schedule Trigger           | Loop Over Items1              |                                                      |
| Loop Over Items1           | SplitInBatches                        | Processes leads individually         | Get many rows              | Aggregate7, Aggregate8        | cold email generation and creating and adding leads to campaign |
| Aggregate7                 | Aggregate                            | Aggregate batch data for campaign creation | Loop Over Items1           | smatlead-create-campaign      | cold email generation and creating and adding leads to campaign |
| smatlead-create-campaign   | HTTP Request                         | Create campaign in Smartlead         | Aggregate7                 | Loop Over Items9              | cold email generation and creating and adding leads to campaign |
| Loop Over Items9           | SplitInBatches                       | Batch processing for adding leads    | smatlead-create-campaign   | smartlead-add-leads-to-campaign | cold email generation and creating and adding leads to campaign |
| smartlead-add-leads-to-campaign | HTTP Request                   | Add leads with emails to Smartlead   | Loop Over Items9           | Loop Over Items9 (loop)       | cold email generation and creating and adding leads to campaign |
| Aggregate8                 | Aggregate                            | Aggregate batch data for profiling   | Loop Over Items1           | Google Gemini Chat Model3     | cold email generation and creating and adding leads to campaign |
| Google Gemini Chat Model3  | AI Language Model                    | Generate detailed lead profile report | Aggregate8                | Basic LLM Chain7              |                                                      |
| Basic LLM Chain7           | AI Prompt Chain                      | Define profiling AI prompt           | Google Gemini Chat Model3  | Update a row                  |                                                      |
| Update a row               | Supabase                            | Update lead profiling in Supabase    | Basic LLM Chain7           | Aggregate6                   |                                                      |
| Aggregate6                 | Aggregate                            | Aggregate data for lead qualification | Update a row              | Google Gemini Chat Model1     |                                                      |
| Google Gemini Chat Model1  | AI Language Model                    | Generate lead qualification & scoring | Aggregate6                | Basic LLM Chain5              |                                                      |
| Basic LLM Chain5           | AI Prompt Chain                      | Define lead qualification prompt      | Google Gemini Chat Model1  | Information Extractor2        |                                                      |
| Information Extractor2     | AI Information Extractor             | Extract structured scoring data      | Basic LLM Chain5           | Update a row1                 |                                                      |
| Update a row1              | Supabase                            | Update lead scores and verdicts      | Information Extractor2     | Code                        |                                                      |
| Code                      | Code                                | Enforce minimum lead score threshold | Update a row1              | If7                         |                                                      |
| If7                       | If                                  | Conditional check on lead score      | Code                      | Aggregate5, Loop Over Items1  |                                                      |
| Aggregate5                 | Aggregate                            | Aggregate leads passing threshold    | If7                       | Basic LLM Chain6             |                                                      |
| Basic LLM Chain6           | AI Prompt Chain                      | Email generation prompt               | Aggregate5                | Information Extractor3        |                                                      |
| Information Extractor3     | AI Information Extractor             | Extract structured email content     | Basic LLM Chain6           | Update a row2                 |                                                      |
| Update a row2              | Supabase                            | Update lead records with emails      | Information Extractor3     | Loop Over Items1              |                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**
   - Type: Schedule Trigger
   - Configure interval (e.g., every day/hour as needed)
   - Connect output to Supabase "Get many rows" node.

2. **Create Supabase node to fetch leads**
   - Type: Supabase
   - Operation: Get All Rows
   - Table: "Leads"
   - Credentials: Configure with valid Supabase API credentials.
   - Connect output to SplitInBatches node (`Loop Over Items1`).

3. **Create SplitInBatches node**
   - Type: SplitInBatches
   - Default batch size (adjust if needed)
   - Connect output to two Aggregate nodes (`Aggregate7` and `Aggregate8`).

4. **Create Aggregate nodes for batching**
   - Aggregate7: Aggregate all item data for campaign creation.
   - Aggregate8: Aggregate all item data for profiling.
   - Connect `Aggregate7` to HTTP Request node for Smartlead campaign creation.
   - Connect `Aggregate8` to AI nodes for profiling.

5. **Create HTTP Request node to create Smartlead campaign (`smatlead-create-campaign`)**
   - Method: POST
   - URL: Smartlead API endpoint for campaign creation with API key query parameter.
   - Headers: Content-Type application/json.
   - Body: JSON with campaign name and optional client_id.
   - Connect output to SplitInBatches node (`Loop Over Items9`).

6. **Create SplitInBatches node (`Loop Over Items9`)**
   - Used for batch adding leads to campaign.
   - Connect output to HTTP Request node for adding leads.

7. **Create HTTP Request node to add leads to campaign (`smartlead-add-leads-to-campaign`)**
   - Method: POST
   - URL: Smartlead API endpoint with campaign ID from previous node.
   - Headers: Content-Type application/json.
   - Body: JSON payload with lead details and personalized email fields.
   - Connect output back to `Loop Over Items9` for batch processing.

8. **Create AI nodes for Lead Profiling**
   - Aggregate8 output connects to Google Gemini Chat Model3 node.
   - Configure Google Gemini Chat Model3:
     - Model Name: "models/gemini-1.5-flash"
     - Credentials: Google Palm API
   - Connect output to Basic LLM Chain7 node.
   - Set prompt with detailed detective profiling instructions referencing lead fields.
   - Connect output to Supabase Update a row node.
   - Update "LEAD PROFILING" field filtered by lead email.

9. **Create aggregation and AI nodes for Lead Qualification**
   - Aggregate6 node aggregates data post profiling update.
   - Connect to Google Gemini Chat Model1 (same model & creds).
   - Connect to Basic LLM Chain5 with detailed scoring criteria prompt.
   - Connect to Information Extractor2 to parse AI's scoring output.
   - Connect to Supabase Update a row1 to save scores and verdict.
   - Connect to Code node to enforce minimum score threshold (60).
   - Connect to If node (`If7`) that checks if scoreThreshold > 40.
   - If true, connect to Aggregate5 for leads passing threshold; else leads are skipped.

10. **Create AI nodes for Email Generation**
    - Aggregate5 output to Google Gemini Chat Model6.
    - Connect to Basic LLM Chain6 with strict prompt on cold email generation rules.
    - Connect to Information Extractor3 to extract structured email lines.
    - Connect to Supabase Update a row2 to save email subject lines and body lines.

11. **Connect Loop Over Items1 output**
    - Connect to Aggregate7 and Aggregate8 for campaign creation and profiling flows.

12. **Credential Setup**
    - Google Palm API: Set up Google Gemini API credentials.
    - Supabase API: Configure with valid project credentials.
    - Smartlead API: Store API key securely for campaign and lead addition.

13. **Workflow Connections**
    - Ensure all nodes are connected as per above.
    - Set error handling and retries especially on HTTP and Supabase update nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow automates personalized cold email generation with AI-driven lead qualification and Smartlead integration. | Project title: Personalized Cold Email Generator with Supabase, Smartlead & Google Gemini AI       |
| Campaign creation and lead addition to Smartlead API require valid API key and endpoint access.           | See nodes: smatlead-create-campaign, smartlead-add-leads-to-campaign                               |
| AI prompts include complex, multi-part instructions to ensure high-quality lead scoring and message generation. | Prompts used in Basic LLM Chain nodes and Google Gemini Chat Model nodes                            |
| Lead score threshold logic enforces minimum qualification before email generation and campaign addition. | Code and If7 nodes implement threshold logic                                                       |
| Use of Google Gemini (PaLM) AI models requires Google Palm API credentials and may be subject to usage limits. | Nodes: Google Gemini Chat Model1, 2, 3, 4, 6                                                      |
| Supabase nodes require proper API credentials and network access to update lead records reliably.          | Nodes: Get many rows, Update a row, Update a row1, Update a row2                                   |
| The email generation enforces strict deliverability and personalization rules: max 60 words, no special characters, no links, ultra-specific hooks. | Refer to Basic LLM Chain6 prompt instructions                                                    |

---

**Disclaimer:** The provided documentation is based solely on a workflow automated with n8n, respecting content policies and handling only legal and public data.