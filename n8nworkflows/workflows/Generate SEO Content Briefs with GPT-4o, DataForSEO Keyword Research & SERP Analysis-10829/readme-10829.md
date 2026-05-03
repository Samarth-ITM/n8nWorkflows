Generate SEO Content Briefs with GPT-4o, DataForSEO Keyword Research & SERP Analysis

https://n8nworkflows.xyz/workflows/generate-seo-content-briefs-with-gpt-4o--dataforseo-keyword-research---serp-analysis-10829


# Generate SEO Content Briefs with GPT-4o, DataForSEO Keyword Research & SERP Analysis

### 1. Workflow Overview

This workflow automates the generation of SEO content briefs by integrating AI-powered brief writing (GPT-4o-mini), real-time keyword research from DataForSEO, SERP analysis via SerpAPI, and historical content retrieval from Google Sheets. It processes chat inputs (topics or intents), performs comprehensive SEO analysis including keyword metrics, competitor insights, and content gaps, then generates a structured SEO brief with quality scoring and version control. The final briefs are stored in Google Sheets and optionally trigger Slack alerts if quality thresholds are not met.

**Logical Blocks:**

- **1.1 Input Reception and Normalization:** Captures chat input and standardizes it for downstream processing.
- **1.2 Keyword Research:** Fetches live keyword metrics (volume, difficulty, CPC) from DataForSEO.
- **1.3 Historical Context Retrieval:** Retrieves past content briefs from Google Sheets to enable versioning and context-aware brief generation.
- **1.4 SERP Analysis:** Obtains competitor and SERP data via SerpAPI.
- **1.5 AI Brief Generation:** The core AI agent combines all data to generate a detailed SEO content brief in JSON.
- **1.6 Quality Scoring and Validation:** Calculates quality scores based on SEO and content criteria, validates the brief, and routes accordingly.
- **1.7 HTML Preview Generation & Storage:** Creates a styled HTML preview of the brief and stores it with version control in Google Sheets.
- **1.8 Notification:** Sends Slack alerts for briefs failing quality standards (optional).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Normalization

- **Overview:**  
Receives and normalizes incoming chat messages into a structured JSON containing intent, topic, content, and parameters.

- **Nodes Involved:**  
  - Chat Trigger  
  - Normalize Input

- **Node Details:**  

  - **Chat Trigger**  
    - Type: Chat Trigger (LangChain)  
    - Role: Entry point for user chat input, triggering the workflow.  
    - Config: Listens for incoming chat messages via a webhook.  
    - Inputs: External chat message via webhook.  
    - Outputs: Raw message JSON forwarded to Normalize Input.  
    - Potential Failures: Webhook connectivity issues, malformed input.

  - **Normalize Input**  
    - Type: Set Node  
    - Role: Standardizes input fields into a consistent format (`intent`, `topic`, `content`, `parameter`).  
    - Config: Uses expressions to fallback on defaults if fields missing:  
      - `intent`: defaults to 'brief'  
      - `topic`: uses input topic or chatInput fallback  
      - `content`: empty string if none  
      - `parameter`: empty object if none  
    - Inputs: From Chat Trigger.  
    - Outputs: Normalized JSON for downstream nodes.  
    - Edge Cases: Missing or unexpected input fields gracefully handled by defaults.

---

#### 1.2 Keyword Research

- **Overview:**  
Fetches real-time keyword metrics (search volume, difficulty, CPC) from DataForSEO API based on the normalized topic.

- **Nodes Involved:**  
  - Fetch Keyword Metrics from DataForSEO

- **Node Details:**  

  - **Fetch Keyword Metrics from DataForSEO**  
    - Type: HTTP Request  
    - Role: Calls DataForSEO API to retrieve live keyword data.  
    - Config:  
      - POST request to `https://api.dataforseo.com/v3/keywords_data/google/search_volume/live`  
      - Body includes topic keyword, language code `en`, location code `2840` (USA).  
      - Auth: HTTP Basic Auth with DataForSEO credentials.  
    - Inputs: Normalized topic from Normalize Input node.  
    - Outputs: Keyword metrics JSON passed to AI agent.  
    - Failures: API errors, authentication failures, rate limits.  
    - Notes: Sticky note mentions this node fetches search volume, keyword difficulty, and CPC data.

---

#### 1.3 Historical Context Retrieval

- **Overview:**  
Obtains existing briefs and versions from Google Sheets to provide context and enable version control.

- **Nodes Involved:**  
  - Retrieve Historical Content Context

- **Node Details:**  

  - **Retrieve Historical Content Context**  
    - Type: Google Sheets Tool  
    - Role: Reads existing content briefs from the `content_versions` sheet in a specified Google Sheets document.  
    - Config:  
      - Document ID and Sheet Name set to the SEO Content Automation sheet and `content_versions` tab.  
      - Uses OAuth2 credentials for Google Sheets.  
    - Inputs: Triggered within AI Agent context to provide content history.  
    - Outputs: Historical briefs data to AI Agent.  
    - Failures: Authentication errors, sheet not found, permission issues.

---

#### 1.4 SERP Analysis

- **Overview:**  
Fetches SERP competitor data using SerpAPI to enrich AI brief generation with competitor intelligence.

- **Nodes Involved:**  
  - SERP Analysis Tool

- **Node Details:**  

  - **SERP Analysis Tool**  
    - Type: LangChain SerpApi tool node  
    - Role: Performs real-time SERP analysis based on topic keyword.  
    - Config: Uses SerpAPI credentials.  
    - Inputs: Provided as an AI tool input to the AI Agent node.  
    - Outputs: Competitor and SERP data injected into AI prompt context.  
    - Failures: API limits, authentication, unexpected data format.

---

#### 1.5 AI Brief Generation

- **Overview:**  
Generates a structured SEO content brief by combining keyword data, SERP analysis, competitor intelligence, and historical briefs using GPT-4o-mini.

- **Nodes Involved:**  
  - OpenAI GPT-4o-mini Model  
  - Short-Term Memory  
  - AI Agent (Brief Writer)  
  - Structured JSON Output Parser

- **Node Details:**  

  - **OpenAI GPT-4o-mini Model**  
    - Type: LangChain OpenAI Chat model  
    - Role: Executes GPT-4o-mini prompt for generating the SEO brief.  
    - Config: Model set to "gpt-4o-mini".  
    - Credentials: OpenAI API key with GPT-4o-mini access.  
    - Inputs: Text prompt from AI Agent.  
    - Outputs: Raw AI response passed to AI Agent.  
    - Edge Cases: Model rate limits, unexpected output format.

  - **Short-Term Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains session context to allow continuity in multi-step AI interactions.  
    - Config: Uses a custom session key "brief-writer-session".  
    - Inputs: Receives AI Agent context.  
    - Outputs: Contextual memory for AI Agent.  
    - Failures: Memory overflow, session mismatch.

  - **AI Agent (Brief Writer)**  
    - Type: LangChain Agent node  
    - Role: Orchestrates AI brief creation by combining keyword metrics, SERP data, historical context, and user inputs.  
    - Config:  
      - System prompt instructs the agent as senior SEO strategist.  
      - Uses all data sources to generate JSON with detailed fields: title, meta description, keywords, outline, word count, tone, CTAs, internal links, semantic entities, content gaps, and competitive intel.  
      - Implements versioning logic using Google Sheets data.  
      - Outputs JSON only.  
    - Inputs: Keyword data, SERP tool output, historical context, memory, and normalized input.  
    - Outputs: Structured JSON brief for downstream parsing.  
    - Version Requirement: Requires n8n version with LangChain Agent node support (v2.1+).  
    - Edge Cases: AI hallucination, prompt truncation, data source inconsistency.

  - **Structured JSON Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI agent JSON output into usable structured JSON for further processing.  
    - Config: Uses a JSON schema example to enforce structure.  
    - Inputs: Raw AI output from AI Agent.  
    - Outputs: Parsed JSON object.  
    - Failures: Parsing errors if AI output deviates from schema.

---

#### 1.6 Quality Scoring and Validation

- **Overview:**  
Calculates SEO, differentiation, completeness, and overall quality scores for the generated brief, then validates against thresholds to decide further action.

- **Nodes Involved:**  
  - Calculate Quality Scores  
  - Validate Brief Quality

- **Node Details:**  

  - **Calculate Quality Scores**  
    - Type: Code Node (JavaScript)  
    - Role: Applies custom logic to compute multiple quality scores based on brief contents and competitive intel.  
    - Config:  
      - Scores SEO factors like keyword count, meta description length, title length, outline size, semantic entities.  
      - Differentiation scores content gaps, advantages, word count vs competitors.  
      - Completeness checks outline, CTA ideas, internal links, keywords, semantic entities.  
      - Overall score is average of the three, rounded to 1 decimal place.  
    - Inputs: Parsed JSON brief and competitive intel.  
    - Outputs: JSON with added quality_scores field.  
    - Failures: Missing fields causing runtime errors if AI output incomplete.

  - **Validate Brief Quality**  
    - Type: If Node  
    - Role: Checks if the brief meets minimal quality criteria (outline length, keyword count, word count, overall score).  
    - Config: Passes if any condition is true: outline ≥5 sections, keywords ≥3, word count ≥800, overall score ≥6 or >8.  
    - Inputs: Quality scored JSON.  
    - Outputs:  
      - True branch: proceeds to HTML Preview generation.  
      - False branch: triggers Slack alert.  
    - Edge Cases: False positives or negatives if thresholds not well calibrated.

---

#### 1.7 HTML Preview Generation & Storage

- **Overview:**  
Generates an HTML preview of the brief with styled layout, then stores the brief with version control in Google Sheets.

- **Nodes Involved:**  
  - Generate HTML Preview  
  - Store Brief with Version Control

- **Node Details:**  

  - **Generate HTML Preview**  
    - Type: Code Node (JavaScript)  
    - Role: Creates a fully styled HTML page summarizing the brief, quality scores, meta info, keywords, outline, and competitive intel.  
    - Config:  
      - Uses template literals to build HTML with CSS styling.  
      - Converts keywords and outline arrays into HTML elements.  
      - Includes meta data and version info in header.  
      - Returns HTML as JSON and as base64-encoded binary for preview download.  
    - Inputs: JSON brief with quality scores and metadata.  
    - Outputs: JSON with html_preview and binary preview file.  
    - Failures: Encoding issues, missing data fields causing broken HTML.

  - **Store Brief with Version Control**  
    - Type: Google Sheets  
    - Role: Appends or updates the brief record in `content_versions` sheet with all relevant fields and versioning info.  
    - Config:  
      - Maps JSON fields to Google Sheets columns including topic, meta title/desc, outline, keywords, tone, word count, CTAs, context_used, timestamps, content_id, version info.  
      - Uses appendOrUpdate mode matching by timestamp.  
      - Auth: Google Sheets OAuth2 credentials.  
    - Inputs: HTML preview node output for storing finalized brief.  
    - Outputs: Confirmation of sheet update.  
    - Failures: Sheet permissions, API limits, mapping errors.

---

#### 1.8 Notification

- **Overview:**  
Sends Slack alert notifications if the generated brief fails quality validation.

- **Nodes Involved:**  
  - Send Slack Quality Alert

- **Node Details:**  

  - **Send Slack Quality Alert**  
    - Type: HTTP Request  
    - Role: Posts a formatted Slack message to a webhook URL notifying low-quality brief generation.  
    - Config:  
      - POSTs JSON payload containing topic and quality score.  
      - Webhook URL must be replaced by user.  
    - Inputs: From Validate Brief Quality false branch.  
    - Outputs: Slack message delivery confirmation.  
    - Failures: Incorrect webhook URL, network issues.

---

### 3. Summary Table

| Node Name                       | Node Type                           | Functional Role                             | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                                      |
|--------------------------------|-----------------------------------|---------------------------------------------|------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------|
| Chat Trigger                   | LangChain Chat Trigger             | Receives user chat input                     | -                            | Normalize Input                 |                                                                                                                  |
| Normalize Input               | Set Node                          | Standardizes input fields                     | Chat Trigger                 | Fetch Keyword Metrics from DataForSEO | "Standardizes incoming chat messages into a consistent format (intent, topic, content, parameters) for downstream processing." |
| Fetch Keyword Metrics from DataForSEO | HTTP Request                     | Retrieves keyword metrics (volume, difficulty, CPC) | Normalize Input              | AI Agent (Brief Writer)          | "Fetches search volume, keyword difficulty, and CPC data"                                                       |
| Retrieve Historical Content Context | Google Sheets Tool                | Gets past content briefs for context and versioning | AI Agent (Brief Writer)       | AI Agent (Brief Writer)          |                                                                                                                  |
| SERP Analysis Tool            | LangChain SerpApi Tool             | Performs real-time SERP competitor analysis  | AI Agent (Brief Writer)       | AI Agent (Brief Writer)          |                                                                                                                  |
| OpenAI GPT-4o-mini Model      | LangChain OpenAI Model             | Generates AI brief based on prompt           | AI Agent (Brief Writer)       | AI Agent (Brief Writer)          |                                                                                                                  |
| Short-Term Memory             | LangChain Memory Buffer Window     | Maintains session context for AI agent       | AI Agent (Brief Writer)       | AI Agent (Brief Writer)          |                                                                                                                  |
| AI Agent (Brief Writer)       | LangChain Agent                   | Combines all data sources to generate brief  | Fetch Keyword Metrics, SERP Tool, Historical Context, Memory, Normalize Input | Calculate Quality Scores         | "The AI agent combines keyword data, SERP analysis, competitor intelligence, and historical content from Google Sheets to generate comprehensive briefs with structured JSON output." |
| Structured JSON Output Parser | LangChain Output Parser Structured | Parses AI JSON output into structured data   | AI Agent (Brief Writer)       | Calculate Quality Scores         |                                                                                                                  |
| Calculate Quality Scores      | Code Node                        | Computes SEO, differentiation, completeness, and overall quality scores | Structured JSON Output Parser | Validate Brief Quality           |                                                                                                                  |
| Validate Brief Quality        | If Node                         | Validates brief quality against thresholds   | Calculate Quality Scores      | Generate HTML Preview, Send Slack Quality Alert | "Calculates quality scores across SEO, differentiation, and completeness dimensions. Validates briefs against minimum thresholds..." |
| Generate HTML Preview         | Code Node                       | Creates styled HTML preview for the brief    | Validate Brief Quality (true) | Store Brief with Version Control |                                                                                                                  |
| Store Brief with Version Control | Google Sheets                   | Stores the brief with version control         | Generate HTML Preview         | -                               | "All briefs are automatically stored in Google Sheets with version control."                                     |
| Send Slack Quality Alert      | HTTP Request                    | Sends alert if brief quality is low           | Validate Brief Quality (false) | -                               | "Set Slack webhook URL in this node for low-quality brief notifications."                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger node**  
   - Type: LangChain Chat Trigger  
   - Configure webhook ID (auto-generated) to receive chat messages.

2. **Add Normalize Input node**  
   - Type: Set node  
   - Assign variables:  
     - `intent`: `={{ $json.intent || 'brief' }}`  
     - `topic`: `={{ $json.topic || $json.chatInput }}`  
     - `content`: `={{ $json.content || '' }}`  
     - `parameter`: `={{ $json.parameter || {} }}`  
   - Connect from Chat Trigger.

3. **Add HTTP Request node for DataForSEO**  
   - Name: Fetch Keyword Metrics from DataForSEO  
   - Method: POST  
   - URL: `https://api.dataforseo.com/v3/keywords_data/google/search_volume/live`  
   - Auth: HTTP Basic Auth with DataForSEO credentials  
   - Body Parameters:  
     - keywords: `[ "{{ $('Normalize Input').item.json.topic }}" ]`  
     - language_code: `en`  
     - location_code: `2840` (USA)  
   - Connect from Normalize Input.

4. **Add Google Sheets Tool node for Historical Context**  
   - Name: Retrieve Historical Content Context  
   - Document ID: Your Google Sheets SEO Content Automation document  
   - Sheet: `content_versions`  
   - Auth: OAuth2 Google Sheets credentials  
   - Connect output to AI Agent node.

5. **Add SerpAPI Tool node**  
   - Name: SERP Analysis Tool  
   - Configure SerpAPI credentials  
   - Connect output to AI Agent node.

6. **Add OpenAI GPT-4o-mini Model node**  
   - Model: `gpt-4o-mini`  
   - Configure OpenAI API credentials  
   - Connect output to AI Agent node.

7. **Add Short-Term Memory (LangChain Memory Buffer Window)**  
   - Session Key: `brief-writer-session`  
   - Session ID Type: customKey  
   - Connect output to AI Agent node.

8. **Add AI Agent (Brief Writer) node**  
   - Type: LangChain Agent  
   - System Prompt: Provide detailed SEO strategist instructions for brief generation including use of keyword data, competitor intel, content gaps, internal linking, versioning logic, and strict JSON output format.  
   - Inputs: Connect from Fetch Keyword Metrics, SERP Analysis Tool, Retrieve Historical Content Context, Short-Term Memory, OpenAI GPT-4o-mini Model, and Normalize Input.  
   - Outputs: Connect to Structured JSON Output Parser.

9. **Add Structured JSON Output Parser node**  
   - Load JSON schema example for brief structure (title, meta, keywords, outline, scores, metadata, etc.)  
   - Connect from AI Agent node.  
   - Output connects to Calculate Quality Scores.

10. **Add Calculate Quality Scores node**  
    - Type: Code node  
    - JavaScript logic to compute SEO, differentiation, completeness, and overall scores based on brief and competitive intel fields.  
    - Connect from Structured JSON Output Parser.  
    - Output connects to Validate Brief Quality.

11. **Add Validate Brief Quality node**  
    - Type: If node  
    - Conditions: Pass if any of:  
      - outline length ≥5  
      - target keywords length ≥3  
      - word count ≥800  
      - overall score ≥6 or >8  
    - True branch connects to Generate HTML Preview.  
    - False branch connects to Send Slack Quality Alert.

12. **Add Generate HTML Preview node**  
    - Type: Code node  
    - JavaScript code builds full HTML preview with styling for title, meta info, keywords, outline, scores, and competitive intel.  
    - Connect from Validate Brief Quality true branch.  
    - Output connects to Store Brief with Version Control.

13. **Add Store Brief with Version Control node**  
    - Type: Google Sheets node  
    - Document ID and Sheet: same as Historical content node  
    - Operation: Append or update based on timestamp  
    - Map all brief fields (content_id, version_id, version_no, topic, meta_title, meta_desc, outline, keywords, tone, word_count, cta_ideas, context_used, timestamp)  
    - Connect from Generate HTML Preview.

14. **Add Send Slack Quality Alert node (optional)**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Slack webhook (replace placeholder with actual URL)  
    - JSON body: formatted alert message with topic and quality score.  
    - Connect from Validate Brief Quality false branch.

15. **Add Sticky Notes** (optional for clarity)  
    - Add notes describing each section and node as per workflow overview and block descriptions.

16. **Credential setup:**  
    - OpenAI API with GPT-4o-mini access  
    - DataForSEO HTTP Basic Auth  
    - SerpAPI credentials  
    - Google Sheets OAuth2 with read/write access to the correct spreadsheet  
    - Slack webhook URL (optional)

17. **Testing:**  
    - Send chat input via webhook with fields `intent`, `topic`, `content`.  
    - Verify keyword metrics retrieved, AI brief generated, quality scored, and stored in Google Sheets.  
    - Confirm Slack alerts for low-quality briefs if configured.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow uses GPT-4o-mini model for cost-effective AI brief generation with consistent quality.                                                                                                                           | Mentioned in Overview Sticky Note.                                                                                  |
| Google Sheet must have a sheet named `content_versions` with columns: content_id, version_no, version_id, topic, meta_title, meta_desc, outline, keywords, tone, word_count, cta_ideas, context_used, timestamp                     | Setup instructions in Overview Sticky Note.                                                                        |
| Slack webhook URL must be updated in the “Send Slack Quality Alert” node to enable notifications.                                                                                                                              | Credential Info Sticky Note.                                                                                        |
| Workflow designed for US English keyword data (location code 2840, language code "en") but can be adjusted as needed.                                                                                                          | Parameter in Fetch Keyword Metrics node.                                                                             |
| Versioning logic: existing topics reuse content_id and increment version_no, new topics get CNT-<timestamp> content_id and version_no=1.                                                                                       | Detailed in AI Agent system prompt.                                                                                  |
| HTML preview node outputs both JSON and base64-encoded HTML binary for preview download or email attachment.                                                                                                                  | Generate HTML Preview node description.                                                                             |
| Workflow uses multiple LangChain nodes requiring n8n version supporting these features (typically n8n v0.205+).                                                                                                               | Node version references.                                                                                            |
| For further optimization, consider handling API rate limits and errors explicitly with retries or error nodes.                                                                                                                | Suggested best practice.                                                                                            |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All processed data is lawful and public.