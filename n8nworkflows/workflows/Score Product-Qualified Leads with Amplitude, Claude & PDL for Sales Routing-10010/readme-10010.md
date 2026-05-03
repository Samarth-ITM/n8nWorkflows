Score Product-Qualified Leads with Amplitude, Claude & PDL for Sales Routing

https://n8nworkflows.xyz/workflows/score-product-qualified-leads-with-amplitude--claude---pdl-for-sales-routing-10010


# Score Product-Qualified Leads with Amplitude, Claude & PDL for Sales Routing

---

### 1. Workflow Overview

This workflow automates the scoring of Product-Qualified Leads (PQLs) that enter a specific Amplitude cohort. It integrates multiple data sources and AI services to enrich lead data, analyze company context, and produce a detailed PQL score. The output enables targeted sales routing and personalized sales alerts through Slack.

**Target Use Cases:**  
- Real-time lead scoring based on product usage and company attributes  
- Enriching leads with third-party data for better qualification  
- Automated sales notifications and routing based on AI-driven scoring  
- Teams using Amplitude cohorts, People Data Labs enrichment, Perplexity AI research, Anthropic Claude AI, Google Docs criteria, and Slack alerts

**Logical Blocks:**  
- **1.1 Input Reception:** Receive webhook from Amplitude when users enter a PQL cohort  
- **1.2 Data Parsing & Filtering:** Extract and validate user data; filter only users entering the cohort  
- **1.3 Lead Enrichment:** Enrich lead data with People Data Labs API and Perplexity AI company research  
- **1.4 Data Merging:** Consolidate enriched data and usage metrics into a unified lead profile  
- **1.5 AI Scoring:** Use Anthropic Claude AI agent with Google Docs criteria to score and route leads  
- **1.6 Slack Alert Formatting & Delivery:** Format hot lead alerts and send to Slack sales channel  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives real-time webhook notifications from Amplitude when a user enters a defined PQL cohort.

- **Nodes Involved:**  
  - Amplitude Cohort Webhook  
  - Sticky Note (Amplitude Setup instructions)

- **Node Details:**  

  - **Amplitude Cohort Webhook**  
    - *Type:* Webhook  
    - *Role:* Entry point triggered by Amplitude cohort webhook POST requests  
    - *Configuration:* Path `amplitude-pql-cohort`, HTTP POST method  
    - *Input:* HTTP requests from Amplitude  
    - *Output:* Raw webhook JSON payload containing user and cohort info  
    - *Edge Cases:* Missing payload, malformed requests, webhook authorization issues (not enforced here)  
    - *Sticky Note:* Explains Amplitude cohort webhook setup  

---

#### 2.2 Data Parsing & Filtering

- **Overview:**  
Parses the webhook payload to extract user and usage details; validates email format; filters to process only users entering the cohort.

- **Nodes Involved:**  
  - Parse Webhook Data (Code node)  
  - Filter: Entering Cohort Only  
  - Sticky Notes (Webhook Data Parsing info)

- **Node Details:**  

  - **Parse Webhook Data**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts user information and usage metrics from webhook payload  
    - *Configuration:*  
      - Processes the first user in the payload batch  
      - Validates email format against regex  
      - Extracts fields: email, name, userId, domain, free email flag, cohort info, usage data (sessions, plan, features, etc.)  
      - Throws errors if no users or invalid email  
    - *Input:* Webhook JSON payload  
    - *Output:* Structured JSON with parsed and validated user info  
    - *Edge Cases:* Empty user list, missing email, invalid email format, missing usage properties  

  - **Filter: Entering Cohort Only**  
    - *Type:* Filter  
    - *Role:* Passes data only if user is entering the cohort (`inCohort` true)  
    - *Input:* Output from parser  
    - *Output:* Passes or blocks data depending on cohort entry status  
    - *Edge Cases:* Missing or false `inCohort` field  

  - **Sticky Note1:** Instructions on Amplitude setup for webhook  
  - **Sticky Note (near Parse Webhook Data):** Explains webhook data extraction  

---

#### 2.3 Lead Enrichment

- **Overview:**  
Enriches lead data with detailed person and company information using People Data Labs (PDL) and performs AI-driven company research with Perplexity.

- **Nodes Involved:**  
  - PDL Enrich (HTTP Request)  
  - Company Research (Perplexity AI node)  
  - Sticky Notes for PDL and Perplexity API setup

- **Node Details:**  

  - **PDL Enrich**  
    - *Type:* HTTP Request  
    - *Role:* Calls People Data Labs Person Enrich API to retrieve job title, company size, industry, seniority, LinkedIn URL, location  
    - *Configuration:*  
      - URL: `https://api.peopledatalabs.com/v5/person/enrich`  
      - Query parameters: email from parsed data  
      - Authentication: HTTP Header Auth with API key (`X-Api-Key`)  
      - Continues on fail to avoid blocking workflow on enrichment errors  
    - *Input:* Filtered user data with email  
    - *Output:* Enrichment JSON from PDL API or error info  
    - *Edge Cases:* API failures, invalid email, missing data, rate limits  

  - **Company Research**  
    - *Type:* Perplexity AI node  
    - *Role:* Uses AI to research company context relevant to product-led growth  
    - *Configuration:*  
      - Prompt dynamically built using enriched company name  
      - Focus on growth stage, funding, tech stack, expansion signals, budget indicators  
      - Output is text with specific sections expected  
      - Continues on fail to tolerate AI errors  
    - *Input:* Output from PDL Enrich (company name)  
    - *Output:* AI research text summary  
    - *Edge Cases:* AI API failures, incomplete or unstructured response  

  - **Sticky Note2:** PDL API key setup instructions  
  - **Sticky Note3:** Perplexity API key and usage instructions  

---

#### 2.4 Data Merging

- **Overview:**  
Combines Amplitude usage data, PDL enrichment, and Perplexity research into one unified data object representing the enriched PQL.

- **Nodes Involved:**  
  - Merge All Sources (Merge node)  
  - Merge PQL Data (Code node)

- **Node Details:**  

  - **Merge All Sources**  
    - *Type:* Merge  
    - *Role:* Combines outputs from Company Research, PDL Enrich, and filtered Amplitude data into an array for processing  
    - *Input:* Outputs from enrichment and research nodes  
    - *Output:* Array of inputs for code merging  
    - *Edge Cases:* Missing or failed inputs  

  - **Merge PQL Data**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses merged inputs to create a structured enriched lead profile  
    - *Configuration:*  
      - Extracts specific fields from each input index (company research, PDL data, Amplitude usage)  
      - Parses AI research text into structured sections (company stage, tech sophistication, etc.)  
      - Calculates a preliminary quality score based on presence of key data  
      - Tracks sources successful or failed  
    - *Input:* Merged array of enrichment and usage data  
    - *Output:* Single enriched PQL JSON object with metadata and quality score  
    - *Edge Cases:* Partial data availability, missing sections in AI text, malformed JSON from API  

---

#### 2.5 AI Scoring

- **Overview:**  
Uses an Anthropic Claude AI agent combined with Google Docs ICP criteria to score leads on usage, fit, intent, and timing, producing routing recommendations.

- **Nodes Involved:**  
  - ICP Guidelines (Google Docs Tool)  
  - PQL Scoring Agent (LangChain Agent node)  
  - Anthropic Chat Model (AI language model node)  
  - Think (LangChain tool think node)  
  - Parse & Structure PQL (Set node)  
  - Sticky Note4 and Sticky Note5 (AI setup and ICP doc instructions)

- **Node Details:**  

  - **ICP Guidelines**  
    - *Type:* Google Docs Tool  
    - *Role:* Retrieves ICP scoring criteria document from Google Docs  
    - *Input:* Triggered by agent to fetch scoring rules  
    - *Output:* Raw text content of ICP document  
    - *Edge Cases:* Missing doc ID, permission errors, API limits  
    - *Credential:* Google Docs OAuth2 API  

  - **PQL Scoring Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Manages orchestration of AI scoring using usage/enrichment data and ICP criteria  
    - *Configuration:*  
      - Prompt instructs agent to fetch ICP rules from Google Docs tool  
      - Scores components: usage intensity, ICP fit, intent signals, timing  
      - Applies detailed scoring rubric and returns structured JSON output with score, routing, sales advice  
    - *Input:* Enriched PQL data plus ICP Guidelines output  
    - *Output:* AI-scored PQL JSON  
    - *Edge Cases:* AI response errors, invalid JSON output, unavailability of ICP doc  
    - *Sub-nodes:* Uses Anthropic Chat Model for language model calls, Think node for intermediate reasoning  

  - **Anthropic Chat Model**  
    - *Type:* LangChain AI Language Model Node  
    - *Role:* Executes Anthropic Claude model (Claude 4 Sonnet) calls for scoring  
    - *Credential:* Anthropic API key  
    - *Edge Cases:* API rate limits, timeouts, malformed responses  

  - **Think**  
    - *Type:* LangChain Tool (Think)  
    - *Role:* Assists reasoning or intermediate steps during scoring agent execution  

  - **Parse & Structure PQL**  
    - *Type:* Set node  
    - *Role:* Cleans and reassigns AI output for downstream use (extracting `.output`)  

  - **Sticky Note4:** Explains AI Scoring Agent setup and inputs  
  - **Sticky Note5:** Describes ICP Criteria document structure and Google Docs setup  

---

#### 2.6 Slack Alert Formatting & Delivery

- **Overview:**  
Formats hot lead scoring outputs into Slack-friendly messages and sends alerts to a configured Slack channel for immediate sales action.

- **Nodes Involved:**  
  - Format Hot PQL Slack (Set node)  
  - Send Hot PQL Alert (Slack node)  
  - Sticky Note6 (Slack setup instructions)

- **Node Details:**  

  - **Format Hot PQL Slack**  
    - *Type:* Set node  
    - *Role:* Constructs Slack message text with emojis, lead details, score, insights, conversation starters, and next action recommendations  
    - *Input:* Parsed and structured AI scoring output  
    - *Output:* JSON with `prompt` field containing Slack-formatted message  
    - *Edge Cases:* Missing fields in AI output, formatting errors  

  - **Send Hot PQL Alert**  
    - *Type:* Slack node  
    - *Role:* Sends message to Slack channel using OAuth2 credentials  
    - *Configuration:*  
      - Text extracted from formatted prompt  
      - Channel ID must be replaced with actual Slack channel ID  
      - Requires Slack OAuth2 credential with `chat:write` and `channels:read` scopes  
    - *Edge Cases:* Invalid channel ID, OAuth token expiry, Slack API errors  

  - **Sticky Note6:** Detailed Slack app and credential setup instructions, reminder to update channel ID  

---

### 3. Summary Table

| Node Name                | Node Type                        | Functional Role                         | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                    |
|--------------------------|---------------------------------|---------------------------------------|--------------------------------|---------------------------------|---------------------------------------------------------------|
| Sticky Note              | Sticky Note                     | Overview of entire workflow            |                                |                                 | 🎯 PQL Scoring Workflow overview and required setup            |
| Amplitude Cohort Webhook | Webhook                        | Receives Amplitude cohort webhook      |                                | Parse Webhook Data              | Fires instantly when user enters PQL cohort                    |
| Sticky Note1             | Sticky Note                     | Amplitude webhook setup instructions   |                                |                                 | 📡 Amplitude Setup instructions                                |
| Parse Webhook Data       | Code                           | Parse and validate webhook payload     | Amplitude Cohort Webhook        | Filter: Entering Cohort Only    | Extracts user data and usage metrics from webhook              |
| Filter: Entering Cohort Only | Filter                     | Filters users entering the cohort      | Parse Webhook Data              | PDL Enrich                     | Only process users entering cohort, not exiting                |
| Sticky Note2             | Sticky Note                     | PDL API key setup instructions         |                                |                                 | 🔍 PDL Enrichment API setup instructions                        |
| PDL Enrich               | HTTP Request                   | Enriches lead with People Data Labs    | Filter: Entering Cohort Only    | Company Research, Merge All Sources | Enriches with company/person data                              |
| Sticky Note3             | Sticky Note                     | Perplexity API key instructions        |                                |                                 | 🔎 Company Research AI setup instructions                       |
| Company Research         | Perplexity AI Node             | AI-based company research               | PDL Enrich                    | Merge All Sources              | Research company stage, funding, tech stack, budget signals    |
| Merge All Sources        | Merge                          | Combines enrichment and usage data     | Company Research, PDL Enrich, Filter: Entering Cohort Only | Merge PQL Data                 |                                                               |
| Merge PQL Data           | Code                           | Consolidates all lead data and scores  | Merge All Sources              | PQL Scoring Agent              | Merges usage, enrichment, research into unified PQL profile    |
| Sticky Note4             | Sticky Note                     | AI Scoring agent setup and requirements |                                |                                 | 🤖 AI Scoring Agent setup instructions                          |
| ICP Guidelines           | Google Docs Tool               | Fetches ICP criteria document           |                                | PQL Scoring Agent              | 📋 ICP Criteria Document instructions                           |
| PQL Scoring Agent        | LangChain Agent               | AI lead scoring and routing             | Merge PQL Data, ICP Guidelines | Parse & Structure PQL          | AI scores PQL and assigns routing category                      |
| Anthropic Chat Model     | LangChain AI Language Model    | Runs Anthropic Claude model             | PQL Scoring Agent              | PQL Scoring Agent              |                                                               |
| Think                    | LangChain Tool (Think)          | Intermediate reasoning during AI scoring | PQL Scoring Agent              | PQL Scoring Agent              |                                                               |
| Parse & Structure PQL    | Set                            | Cleans AI output for formatting         | PQL Scoring Agent              | Format Hot PQL Slack           |                                                               |
| Sticky Note5             | Sticky Note                     | ICP Criteria document example           |                                |                                 | 📋 ICP Criteria document example and Google Docs setup         |
| Format Hot PQL Slack     | Set                            | Formats AI output into Slack message    | Parse & Structure PQL          | Send Hot PQL Alert             |                                                               |
| Sticky Note6             | Sticky Note                     | Slack app and OAuth2 credential setup   |                                |                                 | 💬 Slack Alert setup instructions, update channel ID reminder |
| Send Hot PQL Alert       | Slack                          | Sends formatted PQL alert to Slack      | Format Hot PQL Slack           |                                 | ⚠️ UPDATE CHANNEL ID: Replace placeholder with real Slack channel |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: `Amplitude Cohort Webhook`  
   - Path: `amplitude-pql-cohort`  
   - HTTP Method: POST  

2. **Add Code Node to Parse Webhook Data:**  
   - Name: `Parse Webhook Data`  
   - Paste JavaScript to:  
     - Extract first user from `payload.users` array  
     - Validate and extract email, name, userId, domain  
     - Extract usage metrics (session count, plan, features, etc.)  
     - Throw errors if no users or invalid email  

3. **Add Filter Node to Allow Only Entering Cohort:**  
   - Name: `Filter: Entering Cohort Only`  
   - Condition: `$json.inCohort == true`  

4. **Create HTTP Request Node for PDL Enrichment:**  
   - Name: `PDL Enrich`  
   - Method: GET (default)  
   - URL: `https://api.peopledatalabs.com/v5/person/enrich`  
   - Query Parameters:  
     - `email` = `{{$json.email}}`  
     - `pretty` = `true`  
   - Authentication: HTTP Header Auth with header name `X-Api-Key` and your PDL API key  
   - Continue on fail: Enabled  

5. **Create Perplexity AI Node for Company Research:**  
   - Name: `Company Research`  
   - Credential: Perplexity API key  
   - Prompt:  
     ```
     Research {{ $json.data.job_company_name }} for product-led growth context.
     
     Focus on:
     1. Company growth stage and funding
     2. Tech stack and product complexity
     3. Team size and structure
     4. Expansion signals (hiring, new offices)
     5. Budget indicators
     
     Provide under 150 words.
     
     Format:
     **COMPANY STAGE:** [Growth stage, funding, maturity]
     **TECH SOPHISTICATION:** [Tech stack, product complexity]
     **EXPANSION SIGNALS:** [Hiring, growth indicators]
     **BUDGET INDICATORS:** [High/Medium/Low likelihood to pay]
     ```  
   - Continue on fail: Enabled  

6. **Add Merge Node to Combine Outputs:**  
   - Name: `Merge All Sources`  
   - Inputs: Outputs from `Company Research`, `PDL Enrich`, and filtered webhook data  
   - Mode: Default (combine inputs into array)  

7. **Add Code Node to Merge PQL Data:**  
   - Name: `Merge PQL Data`  
   - JavaScript code to:  
     - Extract and parse company research text into structured sections  
     - Extract PDL enrichment fields: full name, job title, company size, industry, LinkedIn URL, location  
     - Extract Amplitude usage and cohort info  
     - Calculate quality score (0-100) based on data completeness  
     - Return single enriched PQL JSON object  

8. **Add Google Docs Tool Node to Fetch ICP Criteria:**  
   - Name: `ICP Guidelines`  
   - Operation: Get document content  
   - Credential: Google Docs OAuth2 with access to your ICP criteria document  

9. **Add LangChain Agent Node for AI Scoring:**  
   - Name: `PQL Scoring Agent`  
   - Prompt: Instruct AI to:  
     - Fetch ICP criteria from Google Doc (using Google Docs Tool)  
     - Score usage, ICP fit, intent, timing (0-10 scale)  
     - Return structured JSON with routing category and sales advice  
   - AI Model Node: Anthropic Chat Model (Claude 4 Sonnet)  
   - Connect AI Model and Google Docs Tool as AI tools to agent  

10. **Add LangChain Think Node (optional):**  
    - Name: `Think`  
    - Connect as a tool to the AI agent for intermediate reasoning  

11. **Add Set Node to Parse AI Output:**  
    - Name: `Parse & Structure PQL`  
    - Set JSON output to `{{ $json.output }}` to clean AI agent response  

12. **Add Set Node to Format Slack Message:**  
    - Name: `Format Hot PQL Slack`  
    - Construct message with emojis, lead name, company, score, insights, conversation starters, next action  
    - Use Slack markdown formatting with bullets and bold text  

13. **Add Slack Node to Send Alert:**  
    - Name: `Send Hot PQL Alert`  
    - Authentication: Slack OAuth2 credential with `chat:write` and `channels:read` scopes  
    - Channel ID: Replace placeholder `YOUR_SLACK_CHANNEL` with your Slack sales channel ID  
    - Message Text: Use formatted Slack message from previous node  

14. **Connect Nodes:**  
    - `Amplitude Cohort Webhook` → `Parse Webhook Data` → `Filter: Entering Cohort Only`  
    - `Filter: Entering Cohort Only` → `PDL Enrich` and also directly to `Merge All Sources`  
    - `PDL Enrich` → `Company Research` → `Merge All Sources`  
    - `Merge All Sources` → `Merge PQL Data` → `PQL Scoring Agent`  
    - `ICP Guidelines` → `PQL Scoring Agent` (as AI Tool input)  
    - `PQL Scoring Agent` → `Parse & Structure PQL` → `Format Hot PQL Slack` → `Send Hot PQL Alert`  
    - `PQL Scoring Agent` uses `Anthropic Chat Model` and `Think` as AI language model tools  

15. **Set Credentials:**  
    - People Data Labs API key with HTTP Header Auth (header `X-Api-Key`)  
    - Perplexity API key credential  
    - Anthropic API key for Claude model  
    - Google Docs OAuth2 credential with access to ICP doc  
    - Slack OAuth2 credential with necessary bot scopes  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Workflow requires an Amplitude cohort webhook configured with real-time cadence for instant trigger.                                                                                                                         | Amplitude Data → Destinations → Cohort Webhooks                    |
| PDL API key setup requires creating HTTP Header Auth credential named `X-Api-Key` with your PDL key.                                                                                                                          | https://peopledatalabs.com                                          |
| Perplexity AI API key needed for company research node.                                                                                                                                                                       | https://perplexity.ai                                               |
| Anthropic Claude API key required for AI scoring agent.                                                                                                                                                                       | https://www.anthropic.com                                          |
| Google Docs document with ICP scoring criteria must be prepared and accessible via Google Docs API credential.                                                                                                                | Example structure in Sticky Note5                                   |
| Slack app must be created with bot token scopes `chat:write` and `channels:read`; OAuth2 credential created in n8n; Slack channel ID replaced in alert node.                                                                | https://api.slack.com                                               |
| Replace placeholder `YOUR_SLACK_CHANNEL` with actual Slack channel ID in `Send Hot PQL Alert` node to ensure message delivery.                                                                                               | Slack channel ID (e.g., C01234ABCDE)                               |
| AI scoring prompt requires JSON-only output for parsing; ensure prompt instructions remain precise to avoid markdown or extraneous text.                                                                                      | Inside `PQL Scoring Agent` prompt                                  |
| Workflow tolerates enrichment or AI failures by continuing execution, avoiding blocking lead processing.                                                                                                                      | `continueOnFail` enabled on HTTP Request and Perplexity nodes     |
| Email validation uses regex and rejects invalid or missing emails to prevent enrichment failures downstream.                                                                                                                 | In `Parse Webhook Data` code node                                  |
| Slack formatting uses markdown compatible with Slack including emojis and bullet points for readability by sales teams.                                                                                                      | In `Format Hot PQL Slack` node                                     |

---

**Disclaimer:** The provided text is generated from an automated n8n workflow exporting relevant configurations and instructions. All data processed is legal and publicly available, and the workflow respects content policies.

---