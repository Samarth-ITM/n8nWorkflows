Enrich and Score Leads Automatically with Claude AI, PDL and Perplexity

https://n8nworkflows.xyz/workflows/enrich-and-score-leads-automatically-with-claude-ai--pdl-and-perplexity-9433


# Enrich and Score Leads Automatically with Claude AI, PDL and Perplexity

### 1. Workflow Overview

This workflow automates the process of enriching, researching, scoring, and routing B2B sales leads based on email input. It is designed to accelerate lead qualification by combining multiple AI-powered data sources and firmographic enrichment tools, then scoring leads against defined Ideal Customer Profile (ICP) criteria.

**Target Use Cases:**  
- Sales and marketing teams seeking automated lead enrichment and qualification.  
- Businesses wanting to prioritize outreach by lead quality and engagement signals.  
- Teams using Slack and email for lead alerts and CRM integration for lead tracking.

---

**Logical Blocks:**

- **1.1 Input Reception & Validation**  
  Receives lead input (email and optional name) via webhook, validates format, and parses essential details.

- **1.2 Data Enrichment**  
  Enriches lead data by calling People Data Labs (PDL), scraping LinkedIn, and querying Perplexity AI for individual and company research.

- **1.3 Data Merging & Structuring**  
  Merges data from all sources into a unified lead profile, calculates a data quality score.

- **1.4 Lead Scoring AI**  
  Uses Claude AI with ICP rules fetched from Google Docs to score and categorize leads (hot, warm, cold).

- **1.5 Output Formatting & Routing**  
  Parses AI output, formats data for CRM, Slack alerts, and personalized email drafts. Routes leads based on score.

- **1.6 Notifications & CRM Upsert**  
  Sends Slack alerts/emails for hot leads, digests warm leads to Slack, and upserts all leads into HubSpot CRM.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Validation

- **Overview:**  
  Accepts HTTP POST webhook requests with lead info, validates and parses the email and name inputs, and flags free email domains.

- **Nodes Involved:**  
  - Webhook  
  - Validate & Parse Input

- **Node Details:**  

  - **Webhook**  
    - *Type:* n8n Webhook  
    - *Config:* POST method, path `/lead-intake`  
    - *Input:* HTTP POST JSON payload, e.g., `{"email": "lead@company.com", "name": "Optional Name"}`  
    - *Output:* Passes raw input JSON to next node  
    - *Potential Failures:* Invalid HTTP method, missing body

  - **Validate & Parse Input**  
    - *Type:* Code node (JavaScript)  
    - *Logic:*  
      - Trims and splits input string (`email[, name]`)  
      - Validates email format regex  
      - Identifies if email domain is free (gmail.com, yahoo.com, etc.)  
      - Returns structured JSON with email, name, domain, isFreeEmail, timestamp, and validation status  
    - *Output:* Parsed and validated lead data or error JSON  
    - *Potential Failures:* Missing or badly formatted input, invalid email format

- **Connections:**  
  Webhook → Validate & Parse Input

---

#### 2.2 Data Enrichment

- **Overview:**  
  Enriches lead data using multiple external sources: People Data Labs API for firmographic/person data, Perplexity AI for individual and company research, LinkedIn profile scraping via Apify.

- **Nodes Involved:**  
  - PDL Enrich  
  - Individual Research (Perplexity)  
  - Company Research (Perplexity)  
  - LinkedIn Profile Scraper (Apify)  

- **Node Details:**  

  - **PDL Enrich**  
    - *Type:* HTTP Request  
    - *Config:* GET to PDL API `/v5/person/enrich` with email query parameter  
    - *Authentication:* Header Auth with API Key (X-Api-Key)  
    - *Output:* JSON with enriched person and company data  
    - *Failure Modes:* Auth failure, API limits, invalid email

  - **Individual Research**  
    - *Type:* Perplexity AI node  
    - *Config:* Prompts Perplexity to research individual’s recent career moves, achievements, speaking engagements, and timing signals, limited to 150 words  
    - *Input Variables:* Full name and company from PDL output  
    - *Output:* AI-generated actionable sales insights for the individual  
    - *Failure Modes:* API timeout, response parsing errors

  - **Company Research**  
    - *Type:* Perplexity AI node  
    - *Config:* Focus on company’s recent funding, executive changes, product launches, tech stack changes, growth signals, limited to 150 words  
    - *Input Variables:* Company name from PDL output  
    - *Output:* AI-generated company insights for sales context  
    - *Failure Modes:* Same as above

  - **LinkedIn Profile Scraper**  
    - *Type:* Apify actor node  
    - *Config:* Runs Apify LinkedIn Scraper with profile URL from PDL data  
    - *Output:* Profile headline, summary, posts, connections count  
    - *Failure Modes:* Missing/invalid profile URL, scraper quota exceeded

- **Connections:**  
  Validate & Parse Input → PDL Enrich → Merge All Sources (input 1)  
  PDL Enrich → Individual Research  
  PDL Enrich → Company Research  
  PDL Enrich → LinkedIn Profile Scraper

---

#### 2.3 Data Merging & Structuring

- **Overview:**  
  Combines all enrichment outputs into a single JSON structure, extracts key sections from AI text, and calculates a data quality score.

- **Nodes Involved:**  
  - Merge All Sources  
  - Merge Enrichment Data

- **Node Details:**  

  - **Merge All Sources**  
    - *Type:* Merge node  
    - *Config:* Waits for 5 inputs (PDL, Individual Research, Company Research, LinkedIn, and Parsed Input)  
    - *Role:* Aggregates data streams into array for processing  
    - *Failure:* Missing inputs break merge or cause partial data

  - **Merge Enrichment Data**  
    - *Type:* Code node  
    - *Logic:*  
      - Iterates all inputs by index to identify source type  
      - Extracts named sections (e.g., RECENT ACTIVITY) from AI research nodes using regex  
      - Builds structured enrichment object with pdl, individual, company, linkedin data  
      - Tracks successful and failed sources  
      - Calculates a quality score based on completeness of enrichment  
    - *Output:* Unified enriched lead JSON with metadata and quality score  
    - *Failure Modes:* Parsing errors if AI text format changes, missing data

- **Connections:**  
  PDL Enrich, Individual Research, Company Research, LinkedIn Profile Scraper, Validate & Parse Input → Merge All Sources → Merge Enrichment Data

---

#### 2.4 Lead Scoring AI

- **Overview:**  
  Uses Claude AI agent powered by LangChain to score the lead using ICP rules fetched dynamically from a Google Doc.

- **Nodes Involved:**  
  - ICP & Use Case (Google Docs Tool)  
  - AI Agent (LangChain Agent)  
  - AI Reasoning (Tool Think node)  
  - Anthropic Chat Model (Claude AI)  

- **Node Details:**  

  - **ICP & Use Case**  
    - *Type:* Google Docs Tool node  
    - *Config:* Fetches ICP scoring rules document from specified Google Docs URL  
    - *Output:* Provides text content of ICP rules to AI Agent  
    - *Failure:* Credential issues, inaccessible document, invalid URL

  - **AI Agent**  
    - *Type:* LangChain agent node  
    - *Prompt:* Receives merged enrichment data plus ICP rules, applies detailed scoring logic:  
      - Scores company fit, title fit, buying signals, timing (0-3,0-3,0-2,0-2)  
      - Computes total leadScore (sum 0-10)  
      - Assigns routing category (hot:8-10, warm:5-7, cold:0-4)  
      - Returns structured JSON with detailed scoring breakdown and recommendations  
    - *Failure:* AI response format errors, timeout, API quota

  - **Anthropic Chat Model**  
    - *Type:* LangChain Claude AI model node  
    - *Role:* Provides AI language model interface to Claude for scoring and reasoning  
    - *Failure:* Authentication failure, latency

  - **AI Reasoning**  
    - *Type:* LangChain Tool Think node  
    - *Role:* Intermediate AI tool to assist agent in reasoning steps (invokes AI Agent)  
    - *Failure:* Internal LangChain errors

- **Connections:**  
  Merge Enrichment Data → AI Agent → Parse & Structure Output  
  ICP & Use Case → AI Agent (tool input)  
  Anthropic Chat Model → AI Agent (language model input)  
  AI Reasoning → AI Agent (tool input)

---

#### 2.5 Output Formatting & Routing

- **Overview:**  
  Parses AI’s JSON lead scoring output, extracts fields, formats data for CRM insertion, Slack messaging, and email generation. Routes leads by score.

- **Nodes Involved:**  
  - Parse & Structure Output  
  - Route by Score  
  - Format for CRM  
  - Format Hot Lead Slack  
  - Format Hot Lead Email  
  - Format Warm Lead Slack

- **Node Details:**  

  - **Parse & Structure Output**  
    - *Type:* Code node  
    - *Logic:* Cleans AI output JSON string, extracts fields (email, name, scores, insights, flags, etc.) robustly with regex fallback  
    - *Output:* Clean structured lead scoring JSON  
    - *Failure:* JSON parse errors if AI output malformed

  - **Route by Score**  
    - *Type:* Switch node  
    - *Config:* Routes leads to 3 branches based on leadScore  
      - Hot Lead: score ≥ 8  
      - Warm Lead: score ≥ 5 and < 8  
      - Cold Lead: score < 5  
    - *Output:* Controls which formatting and notification nodes receive lead

  - **Format for CRM**  
    - *Type:* Code node  
    - *Logic:* Structures lead fields to match CRM property names and formats, including splitting names, joining arrays as strings  
    - *Output:* JSON ready for HubSpot or other CRM upsert  
    - *Failure:* Missing critical fields can cause incomplete CRM data

  - **Format Hot Lead Slack**  
    - *Type:* LangChain Google Gemini AI node  
    - *Prompt:* Creates concise, scannable Slack alert message for hot leads with key insights, urgency, and conversation starters  
    - *Output:* Slack message in mrkdwn format

  - **Format Hot Lead Email**  
    - *Type:* LangChain Google Gemini AI node  
    - *Prompt:* Generates personalized warm welcome email for hot leads with relevant insights and soft CTA, returns JSON with email, subject, body  
    - *Output:* Structured email JSON

  - **Format Warm Lead Slack**  
    - *Type:* LangChain Google Gemini AI node  
    - *Prompt:* Creates Slack digest message for warm leads summarizing key insights and next steps  
    - *Output:* Slack message text

- **Connections:**  
  AI Agent → Parse & Structure Output → Route by Score  
  Route by Score (Hot) → Format for CRM, Format Hot Lead Slack, Format Hot Lead Email  
  Route by Score (Warm) → Format for CRM, Format Warm Lead Slack  
  Route by Score (Cold) → Format for CRM

---

#### 2.6 Notifications & CRM Upsert

- **Overview:**  
  Sends notifications to Slack and email for hot leads, posts warm lead summaries to Slack channel, and upserts all leads into HubSpot CRM.

- **Nodes Involved:**  
  - Send Hot Lead Slack Alert  
  - Send Hot Lead Email  
  - Send Warm Lead to Digest  
  - Parse Email JSON  
  - Upsert to HubSpot CRM

- **Node Details:**  

  - **Send Hot Lead Slack Alert**  
    - *Type:* Slack node  
    - *Config:* Posts formatted hot lead message to dedicated Slack channel  
    - *Credentials:* Slack OAuth2  
    - *Failure:* Slack API errors, invalid channel

  - **Send Hot Lead Email**  
    - *Type:* Gmail node  
    - *Config:* Sends personalized email to hot lead using generated subject and body  
    - *Credentials:* Gmail OAuth2  
    - *Failure:* Email sending errors, invalid email

  - **Send Warm Lead to Digest**  
    - *Type:* Slack node  
    - *Config:* Posts warm lead digest message to Slack channel  
    - *Credentials:* Slack OAuth2

  - **Parse Email JSON**  
    - *Type:* Code node  
    - *Logic:* Parses JSON email object from AI output before sending email  
    - *Failure:* Parsing errors if AI output malformed

  - **Upsert to HubSpot CRM**  
    - *Type:* HubSpot node  
    - *Config:* Upserts lead record with key properties and lead score  
    - *Credentials:* HubSpot OAuth2  
    - *Failure:* CRM API errors, missing required fields

- **Connections:**  
  Format Hot Lead Slack → Send Hot Lead Slack Alert  
  Format Hot Lead Email → Parse Email JSON → Send Hot Lead Email  
  Format Warm Lead Slack → Send Warm Lead to Digest  
  Format for CRM → Upsert to HubSpot CRM

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                                    | Input Node(s)                          | Output Node(s)                           | Sticky Note                                                                                              |
|----------------------------|-----------------------------------|---------------------------------------------------|--------------------------------------|-----------------------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook                    | n8n Webhook                       | Receives lead input via HTTP POST                  |                                      | Validate & Parse Input                   |                                                                                                        |
| Validate & Parse Input      | Code                             | Validates and parses email and name input          | Webhook                              | PDL Enrich, Merge All Sources            |                                                                                                        |
| PDL Enrich                 | HTTP Request                     | Enriches lead with People Data Labs API             | Validate & Parse Input                | Merge All Sources, Individual Research, Company Research, LinkedIn Profile Scraper | Setup Required: Create Header Auth credential with X-Api-Key for PDL API key. Alternative: Apollo or Clearbit |
| Individual Research         | Perplexity AI                    | AI research on individual’s career and signals     | PDL Enrich                          | Merge All Sources                        |                                                                                                        |
| Company Research            | Perplexity AI                    | AI research on company’s recent developments       | PDL Enrich                          | Merge All Sources                        |                                                                                                        |
| LinkedIn Profile Scraper    | Apify Actor                     | Scrapes LinkedIn profile data                        | PDL Enrich                          | Merge All Sources                        | Optional: Get API key from https://apify.com/curious_coder/linkedin-profile-scraper. Add OAuth2 creds.  |
| Merge All Sources           | Merge                           | Aggregates all enrichment data streams              | Validate & Parse Input, PDL Enrich, Individual Research, Company Research, LinkedIn Profile Scraper | Merge Enrichment Data                     |                                                                                                        |
| Merge Enrichment Data       | Code                             | Combines enrichment data into unified lead profile | Merge All Sources                   | AI Agent                                |                                                                                                        |
| ICP & Use Case              | Google Docs Tool                | Fetches ICP scoring rules document                  |                                      | AI Agent (tool input)                    | Setup Required: Replace Google Docs URL with your ICP rules document URL. Add OAuth2 credentials.       |
| AI Reasoning                | LangChain Tool Think            | Assists AI agent in reasoning steps                 |                                      | AI Agent (tool input)                    |                                                                                                        |
| Anthropic Chat Model        | LangChain Claude AI Model       | Provides Claude AI language model                    |                                      | AI Agent (language model input)          |                                                                                                        |
| AI Agent                   | LangChain Agent                 | Scores lead using ICP rules and enrichment data     | Merge Enrichment Data, ICP & Use Case, Anthropic Chat Model, AI Reasoning | Parse & Structure Output                 |                                                                                                        |
| Parse & Structure Output    | Code                             | Cleans and extracts structured lead scoring output | AI Agent                            | Route by Score                          |                                                                                                        |
| Route by Score              | Switch                          | Routes leads by leadScore into hot/warm/cold paths | Parse & Structure Output             | Format for CRM, Format Hot Lead Slack, Format Hot Lead Email, Format Warm Lead Slack |                                                                                                        |
| Format for CRM              | Code                             | Formats lead data for CRM upsert                     | Route by Score                      | Upsert to HubSpot CRM                    | Optional: Enable for HubSpot, Salesforce, Pipedrive or custom CRM integration                          |
| Format Hot Lead Slack       | LangChain Google Gemini AI      | Creates Slack alert message for hot leads            | Route by Score (hot)                | Send Hot Lead Slack Alert                |                                                                                                        |
| Format Hot Lead Email       | LangChain Google Gemini AI      | Creates personalized email draft for hot leads      | Route by Score (hot)                | Parse Email JSON                         |                                                                                                        |
| Parse Email JSON            | Code                             | Parses email JSON from AI output before sending     | Format Hot Lead Email               | Send Hot Lead Email                      |                                                                                                        |
| Send Hot Lead Slack Alert   | Slack                          | Sends Slack alert for hot leads                      | Format Hot Lead Slack               |                                         |                                                                                                        |
| Send Hot Lead Email         | Gmail                          | Sends personalized email to hot leads                | Parse Email JSON                   |                                         |                                                                                                        |
| Format Warm Lead Slack      | LangChain Google Gemini AI      | Creates Slack digest message for warm leads          | Route by Score (warm)               | Send Warm Lead to Digest                 |                                                                                                        |
| Send Warm Lead to Digest    | Slack                          | Sends Slack message for warm leads                    | Format Warm Lead Slack             |                                         |                                                                                                        |
| Upsert to HubSpot CRM       | HubSpot                        | Inserts or updates lead record in HubSpot CRM        | Format for CRM                     |                                         | Optional: Enable after configuring HubSpot credentials                                               |
| Sticky Note1                | Sticky Note                    | Overview and instructions                             |                                      |                                         | Enrich and score leads with AI - setup instructions and overview                                     |
| Sticky Note2                | Sticky Note                    | PDL API Key setup instructions                        |                                      |                                         | Setup Required: Create Header Auth credential with X-Api-Key                                       |
| Sticky Note3                | Sticky Note                    | Apify LinkedIn scraper optional setup                 |                                      |                                         | Optional: Get API key from Apify LinkedIn scraper, add OAuth2 credentials                            |
| Sticky Note4                | Sticky Note                    | ICP Google Docs setup instructions                     |                                      |                                         | Setup Required: Replace Google Docs URL with your ICP rules doc, add OAuth2 credentials             |
| Sticky Note5                | Sticky Note                    | CRM integration optional instructions                  |                                      |                                         | Optional: Enable node and add credentials for HubSpot, Salesforce, Pipedrive, or custom CRM        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `lead-intake`  
   - Purpose: Receive incoming lead data JSON.

2. **Add Code Node "Validate & Parse Input"**  
   - Paste provided JavaScript code that:  
     - Validates presence of input  
     - Parses input string to extract email and optional name  
     - Validates email format  
     - Identifies if email domain is free  
     - Outputs structured JSON with validation flag  
   - Connect Webhook output to this node.

3. **Add HTTP Request Node "PDL Enrich"**  
   - Method: GET  
   - URL: `https://api.peopledatalabs.com/v5/person/enrich`  
   - Query Parameters: email (from Validate & Parse Input JSON), pretty=true  
   - Authentication: Header Auth with key name `X-Api-Key` set to your PDL API key  
   - Enable "Continue on Fail"  
   - Connect Validate & Parse Input output to this node.

4. **Add Perplexity AI Node "Individual Research"**  
   - Set prompt to research individual with given structure focusing on recent activity, achievements, etc.  
   - Use variables from PDL Enrich output for full name and company.  
   - Provide Perplexity API credentials.  
   - Connect PDL Enrich output to this node.

5. **Add Perplexity AI Node "Company Research"**  
   - Set prompt focusing on recent company developments for sales intelligence.  
   - Use company name from PDL Enrich output.  
   - Provide Perplexity API credentials.  
   - Connect PDL Enrich output to this node.

6. **Add Apify Actor Node "LinkedIn Profile Scraper" (Optional)**  
   - Use Apify LinkedIn Scraper actor URL  
   - Pass LinkedIn profile URL from PDL Enrich output as input  
   - Provide Apify OAuth2 credentials  
   - Enable "Continue on Fail" to prevent blocking.  
   - Connect PDL Enrich output to this node.

7. **Add Merge Node "Merge All Sources"**  
   - Set number of inputs to 5.  
   - Connect outputs of Individual Research, Company Research, LinkedIn Scraper, PDL Enrich, and Validate & Parse Input to this node (in any order but keep track for merging).

8. **Add Code Node "Merge Enrichment Data"**  
   - Paste provided JavaScript code to extract sections from AI responses, merge all inputs, and compute data quality score.  
   - Connect Merge All Sources output to this node.

9. **Add Google Docs Tool Node "ICP & Use Case"**  
   - Operation: Get  
   - Document URL: your Google Docs ICP rules document URL  
   - Provide Google Docs OAuth2 credentials.

10. **Add LangChain Anthropic Chat Model Node "Anthropic Chat Model"**  
    - Select Claude AI model (e.g., `claude-sonnet-4-20250514`)  
    - Provide Anthropic API credentials.

11. **Add LangChain Tool Think Node "AI Reasoning"**  
    - No special config required.

12. **Add LangChain Agent Node "AI Agent"**  
    - Configure prompt per description, including scoring logic and ICP rules usage.  
    - Set inputs: merged enrichment data, ICP rules from Google Docs, Anthropic model, and AI Reasoning tool.  
    - Connect Merge Enrichment Data, ICP & Use Case, Anthropic Chat Model, AI Reasoning outputs as inputs.

13. **Add Code Node "Parse & Structure Output"**  
    - Paste provided code to parse AI JSON output robustly.  
    - Connect AI Agent output to this node.

14. **Add Switch Node "Route by Score"**  
    - Configure rules:  
      - Hot Lead if leadScore ≥ 8  
      - Warm Lead if leadScore ≥5 and <8  
      - Cold Lead if leadScore <5  
    - Connect Parse & Structure Output to this node.

15. **Add Code Node "Format for CRM"**  
    - Paste code to format lead data for CRM ingestion.  
    - Connect all three outputs of Route by Score to this node (hot, warm, cold).

16. **Add LangChain Google Gemini AI Node "Format Hot Lead Slack"**  
    - Configure prompt to create concise Slack alert for hot leads.  
    - Connect Hot output of Route by Score to this node.

17. **Add LangChain Google Gemini AI Node "Format Hot Lead Email"**  
    - Configure prompt to create personalized welcome email.  
    - Connect Hot output of Route by Score to this node.

18. **Add LangChain Google Gemini AI Node "Format Warm Lead Slack"**  
    - Configure prompt for warm lead Slack digest message.  
    - Connect Warm output of Route by Score to this node.

19. **Add Code Node "Parse Email JSON"**  
    - Use provided code to parse email JSON from AI output before sending email.  
    - Connect Format Hot Lead Email output to this node.

20. **Add Slack Node "Send Hot Lead Slack Alert"**  
    - Configure to send message to your Slack channel for hot leads.  
    - Provide Slack OAuth2 credentials.  
    - Connect Format Hot Lead Slack output to this node.

21. **Add Gmail Node "Send Hot Lead Email"**  
    - Configure to send personalized email to hot lead.  
    - Provide Gmail OAuth2 credentials.  
    - Connect Parse Email JSON output to this node.

22. **Add Slack Node "Send Warm Lead to Digest"**  
    - Configure to send warm lead digest messages to Slack channel.  
    - Provide Slack OAuth2 credentials.  
    - Connect Format Warm Lead Slack output to this node.

23. **Add HubSpot Node "Upsert to HubSpot CRM" (Optional)**  
    - Configure to upsert lead record with fields mapped from Format for CRM output.  
    - Provide HubSpot OAuth2 credentials.  
    - Connect Format for CRM output (all three paths) to this node.

24. **Add Sticky Notes for documentation**  
    - Add notes describing setup requirements for PDL API key, Apify scraper, ICP Google Docs, CRM integration, and overall workflow purpose.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow automates lead enrichment and scoring with AI, reducing manual research time from ~20 minutes to 30-60 seconds.       | Sticky Note1                                                                                            |
| PDL API key must be set up as HTTP Header Auth with name `X-Api-Key` for People Data Labs enrichment.                          | Sticky Note2                                                                                            |
| LinkedIn profile scraping is optional and requires Apify account and OAuth2 credentials.                                         | https://apify.com/curious_coder/linkedin-profile-scraper                                                |
| ICP scoring rules document must be created and shared via Google Docs; URL replaced in ICP & Use Case node.                     | Sticky Note4                                                                                            |
| CRM upsert node is optional and can be enabled for HubSpot, Salesforce, Pipedrive, or any custom CRM with suitable mapping.    | Sticky Note5                                                                                            |
| Slack alerts and email notifications use OAuth2 credentials; ensure scopes include sending messages and sending email.          |                                                                                                          |
| Perplexity AI and Claude AI require API keys and credentials configured in n8n for respective nodes.                           |                                                                                                          |
| Google Gemini (PaLM) nodes for message formatting require Google API credentials with access to PaLM models.                   |                                                                                                          |

---

**Disclaimer:**  
This document is based exclusively on an n8n workflow JSON. It complies with all content policies and contains only legal, public data processing steps. No illegal or protected data is involved.