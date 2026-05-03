Track Competitor SEO Keywords with Decodo + GPT-4.1-mini + Google Sheets

https://n8nworkflows.xyz/workflows/track-competitor-seo-keywords-with-decodo---gpt-4-1-mini---google-sheets-10002


# Track Competitor SEO Keywords with Decodo + GPT-4.1-mini + Google Sheets

### 1. Workflow Overview

This workflow automates competitive SEO keyword tracking by scraping competitor webpages, analyzing the content for SEO-relevant keywords and metadata using AI (OpenAI GPT-4.1-mini via LangChain), and storing structured insights in Google Sheets for ongoing trend monitoring. It is targeted at SEO professionals and digital marketers who need automated, data-driven insights on competitor keyword strategies.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Parameter Setup:** Manual trigger and setting the target webpage URL and geographic region.
- **1.2 Web Scraping via Decodo API:** Extracts webpage content and metadata for analysis.
- **1.3 AI-Powered SEO Content Analysis:** Uses GPT-4.1-mini (LangChain integration) to analyze scraped content and generate structured SEO insights in JSON.
- **1.4 Structured Data Parsing:** Cleans and parses the AI-generated JSON output.
- **1.5 Data Export to Google Sheets:** Appends or updates the structured SEO data in a Google Sheets document for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parameter Setup

**Overview:**  
This block initiates the workflow manually and sets the initial parameters for the URL and geographical location to analyze.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Set the Input Fields

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on demand  
  - Configuration: No parameters; simple manual trigger  
  - Connections: Output → Set the Input Fields  
  - Edge cases: None typical; workflow won't start without manual trigger.

- **Set the Input Fields**  
  - Type: Set Node  
  - Role: Defines input parameters for the scraping (URL and geo)  
  - Configuration:  
    - url = "https://dev.to" (target competitor webpage)  
    - geo = "france" (geographical targeting for Decodo API)  
  - Connections: Output → Decodo  
  - Edge cases: Hardcoded values; requires manual update for different targets.

---

#### 2.2 Web Scraping via Decodo API

**Overview:**  
This block extracts the webpage content and metadata from the specified URL and geo location using the Decodo web scraping API.

**Nodes Involved:**  
- Decodo

**Node Details:**  

- **Decodo**  
  - Type: Custom Decodo node (web scraping API)  
  - Role: Fetches raw webpage content and metadata  
  - Configuration:  
    - Parameters dynamically set from previous node: `url` and `geo`  
    - Credentials: Decodo API credentials  
    - Retry on fail enabled for robustness  
  - Connections: Output → Analyze Keywords  
  - Edge cases:  
    - API authentication errors  
    - Network timeouts  
    - Invalid or unreachable URL  
    - Geo parameter errors  
  - Note: Critical to ensure valid API credentials and input parameters.

---

#### 2.3 AI-Powered SEO Content Analysis

**Overview:**  
Analyzes the scraped content using OpenAI GPT-4.1-mini via LangChain to generate a rich, structured SEO analysis including keywords, scores, recommendations, and metadata insights.

**Nodes Involved:**  
- Analyze Keywords (LangChain LLM Chain)  
- OpenAI Chat Model for Keyword Analysis (OpenAI LLM)

**Node Details:**  

- **Analyze Keywords**  
  - Type: LangChain Chain LLM Node  
  - Role: Sends webpage content to GPT-4.1-mini with a detailed prompt requesting structured SEO JSON analysis  
  - Configuration:  
    - Prompt defines expected JSON output with fields such as primary_keywords, seo_strength_score, readability_score, optimization_recommendations, metadata_insights, content_summary, sentiment, etc.  
    - Input text dynamically pulled from Decodo node output `data.results[0].content`  
    - Retry on fail enabled  
    - Always output data to ensure downstream availability  
  - Connections: Output → Extract Structured JSON  
  - Edge cases:  
    - LLM API rate limits or errors  
    - JSON formatting issues in the response  
    - Incorrect or unexpected content leading to incomplete output

- **OpenAI Chat Model for Keyword Analysis**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Underlying language model for Analyze Keywords chain  
  - Configuration:  
    - Model set to “gpt-4o-mini” (OpenAI GPT-4.1-mini)  
    - Credentials: OpenAI API credentials  
  - Connections: AI Language Model input to Analyze Keywords node  
  - Edge cases: API key issues, rate limits, model availability

---

#### 2.4 Structured Data Parsing

**Overview:**  
This block cleans the raw text output from the AI model and parses it into structured JSON to be consumed by the next step.

**Nodes Involved:**  
- Extract Structured JSON (Code Node)

**Node Details:**  

- **Extract Structured JSON**  
  - Type: Code Node (JavaScript)  
  - Role: Cleans markdown code block wrappers (```json ... ```) from AI output and parses the JSON string into an object  
  - Configuration:  
    - Custom JS code removes ```json and ``` wrappers, trims text, and parses JSON  
    - Outputs structured JSON object for Google Sheets  
  - Connections: Output → Google Sheets Append Row  
  - Edge cases:  
    - Malformed JSON from AI output causing parse errors  
    - Missing or corrupted AI output text

---

#### 2.5 Data Export to Google Sheets

**Overview:**  
Stores the parsed SEO analysis data by appending or updating rows in a Google Sheets document to enable ongoing tracking and visualization.

**Nodes Involved:**  
- Google Sheets Append Row

**Node Details:**  

- **Google Sheets Append Row**  
  - Type: Google Sheets Node  
  - Role: Appends or updates a row in a specific Google Sheet with the structured SEO data  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Sheet Name: Sheet1 (gid=0)  
    - Document ID: Google Sheets document tracking SEO competitor keywords  
    - Column to match on: keyword_density_summary (used as unique key for update)  
    - Data Mode: autoMapInputData (automatically maps JSON keys to columns)  
    - Credentials: Google Sheets OAuth2  
  - Connections: Terminal node  
  - Edge cases:  
    - Authentication failures with Google Sheets API  
    - Data mapping errors if JSON keys do not match columns  
    - API rate limits or quota exceeded

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                           | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                                             |
|-------------------------------|----------------------------------|-----------------------------------------|-------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Initiates the workflow manually         |                               | Set the Input Fields         | ![Logo](https://cdn.brandfetch.io/idIeG9_eXK/w/100/h/100/theme/dark/icon.jpeg?c=1bxid64Mup7aczewSAYMX&t=1756483136894) Automatically scrapes competitor page using Decodo Web Scraping API, extracts SEO keywords and metadata, summarizes keyword opportunities with OpenAI GPT-4.1-mini, and stores the structured data in Google Sheets for trend tracking. |
| Set the Input Fields           | Set Node                         | Defines target URL and geo parameters    | When clicking ‘Execute workflow’ | Decodo                      |                                                                                                                         |
| Decodo                        | Decodo API Node                  | Scrapes webpage content and metadata     | Set the Input Fields           | Analyze Keywords            |                                                                                                                         |
| Analyze Keywords              | LangChain Chain LLM Node          | Analyzes scraped content using GPT       | Decodo                        | Extract Structured JSON     | **Purpose:** Automates competitive SEO research by scraping competitor pages, analyzing keyword patterns using GPT, and storing structured insights in Google Sheets. |
| OpenAI Chat Model for Keyword Analysis | LangChain OpenAI Chat Model    | Provides GPT-4.1-mini language model     | None (AI languageModel input) | Analyze Keywords            |                                                                                                                         |
| Extract Structured JSON       | Code Node                        | Cleans and parses AI JSON output          | Analyze Keywords              | Google Sheets Append Row    |                                                                                                                         |
| Google Sheets Append Row      | Google Sheets Node               | Appends or updates SEO data in Google Sheets | Extract Structured JSON       |                             |                                                                                                                         |
| Sticky Note1                 | Sticky Note                      | Workflow Summary and branding            | None                          | None                        | See first row for full content                                                                                          |
| Sticky Note                  | Sticky Note                      | Detailed purpose and flow summary         | None                          | None                        | See Analyze Keywords row for content                                                                                     |
| Sticky Note2                 | Sticky Note                      | Labels Structured Data Extractor block   | None                          | None                        | ## Structured Data Extractor                                                                                            |
| Sticky Note3                 | Sticky Note                      | Labels Export Data Handling block         | None                          | None                        | ## Export Data Handling                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To start the workflow manually.

2. **Create a Set Node**  
   - Name: `Set the Input Fields`  
   - Set two string fields:  
     - `url`: `"https://dev.to"` (or any competitor URL)  
     - `geo`: `"france"` (or desired geographic location)  
   - Connect Manual Trigger output to this node’s input.

3. **Add Decodo Node**  
   - Name: `Decodo`  
   - Credentials: Use valid Decodo API credentials (setup via n8n credentials)  
   - Parameters:  
     - `url`: Expression `{{$json["url"]}}` from Set Node  
     - `geo`: Expression `{{$json["geo"]}}` from Set Node  
   - Enable retry on failure.  
   - Connect Set Node output to Decodo input.

4. **Add LangChain Chain LLM Node**  
   - Name: `Analyze Keywords`  
   - Credentials: Use OpenAI API credentials configured in n8n  
   - Model: Connect to a separate LangChain OpenAI Chat Model node (see next step)  
   - Parameters:  
     - Set prompt as defined, requesting detailed JSON SEO analysis of the webpage content.  
     - Use expression to pass Decodo output content: `{{ $json.data.results[0].content }}`  
   - Enable retry on failure and always output data.  
   - Connect Decodo output to this node.

5. **Add LangChain OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model for Keyword Analysis`  
   - Model: Select GPT-4.1-mini (model name: `gpt-4o-mini`)  
   - Credentials: OpenAI API credentials  
   - Connect this node as AI language model input to the `Analyze Keywords` node.

6. **Add Code Node**  
   - Name: `Extract Structured JSON`  
   - Purpose: To clean and parse GPT JSON response.  
   - Paste the JS code:  
     ```js
     let text =  $input.first().json.text;
     const output = [];

     // Remove ```json ... ``` or ``` ... ``` wrappers
     text = text
           .replace(/```json\s*/gi, '')
           .replace(/```/g, '')
           .trim();

     // Parse the cleaned JSON text
     const parsed = JSON.parse(text);
     output.push({ json: parsed });

     return output;
     ```  
   - Connect output of `Analyze Keywords` to this node.

7. **Add Google Sheets Node**  
   - Name: `Google Sheets Append Row`  
   - Operation: `appendOrUpdate`  
   - Document ID: Your Google Sheet ID for SEO tracking  
   - Sheet Name: e.g., `Sheet1` (gid=0)  
   - Column to match on: `keyword_density_summary` (to avoid duplicate rows)  
   - Data Mode: `autoMapInputData` to map JSON keys automatically  
   - Credentials: Google Sheets OAuth2 credentials  
   - Connect Code Node output to this node.

8. **Add Sticky Notes** (optional for documentation inside workflow)  
   - Add notes describing the workflow purpose, block labels, and key details as per original workflow.

9. **Test the workflow**  
   - Manually execute the trigger node to validate end-to-end operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow leverages Decodo API for reliable web scraping of competitor pages tailored by geo location.                                                                                                                       | Decodo API documentation and credentials setup needed.                                                         |
| OpenAI GPT-4.1-mini is integrated via LangChain nodes for structured SEO content analysis, providing rich JSON outputs suitable for automated processing.                                                                       | OpenAI API; LangChain integration in n8n.                                                                      |
| Google Sheets serves as a flexible storage and visualization backend, enabling continuous tracking of SEO metrics and keyword trends.                                                                                           | Google Sheets API with OAuth2 credentials.                                                                      |
| The prompt used in the Analyze Keywords node is highly detailed and expects a comprehensive JSON structure capturing multiple SEO metrics and recommendations. This ensures actionable insights downstream.                      | Prompt design best practices for AI SEO analysis.                                                               |
| Sticky notes in the workflow provide an at-a-glance summary and branding, aiding maintainability and onboarding of new users.                                                                                                   | Visual documentation inside n8n workflow editor.                                                                |
| Potential failure points include API authentication errors, network issues, malformed AI output JSON, and data mapping mismatches in Google Sheets, which should be monitored and handled with retries and error notifications. | Error handling and monitoring best practices for long-running production workflows.                              |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.