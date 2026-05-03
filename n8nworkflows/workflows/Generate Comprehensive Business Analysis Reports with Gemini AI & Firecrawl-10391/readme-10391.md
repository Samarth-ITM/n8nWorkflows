Generate Comprehensive Business Analysis Reports with Gemini AI & Firecrawl

https://n8nworkflows.xyz/workflows/generate-comprehensive-business-analysis-reports-with-gemini-ai---firecrawl-10391


# Generate Comprehensive Business Analysis Reports with Gemini AI & Firecrawl

---

# Comprehensive Reference Document for  
**Workflow:** Generate Comprehensive Business Analysis Reports with Gemini AI & Firecrawl  
**Workflow Name:** Business Analysis Pro Workflow Free (20 Oct 2025)  

---

### 1. Workflow Overview

This workflow automates the generation of detailed business analysis reports by integrating multiple AI-driven data sources and scraping tools. It targets digital marketers, SEO professionals, business analysts, and competitive intelligence teams aiming to extract, analyze, and compile comprehensive intelligence on a client’s website, market, and competitors, with localization support for country and language.

The workflow logically divides into four main blocks:

- **1.1 Input Reception & Data Preparation:** Collects user input via a web form, maps country and language names to standardized codes, and generates a unique session identifier.
- **1.2 Business Intelligence Data Gathering:** Scrapes the client’s website, discovers internal links, and performs various AI-assisted online and deep research tasks to collect comprehensive business, market, and competitor insights.
- **1.3 Data Validation, Storage & Report Generation:** Validates the aggregated data, inserts the processed business intelligence fields into a data table, and generates a customized business analysis report using Google Docs and Drive integrations.
- **1.4 Report Delivery:** Converts the report to PDF, stores it in Google Drive, and sends it via email to a specified recipient.

This structure ensures a systematic flow from raw input through multi-source intelligence gathering to validated report delivery.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Data Preparation

**Overview:**  
This block receives the user's input via a web form, standardizes geographic and linguistic parameters by mapping country and language names to corresponding codes, and generates a unique session ID for tracking the workflow execution.

**Nodes Involved:**  
- Submit Form  
- Get Country & Language Code (Code Node)  
- Session ID (Code Node)  

**Node Details:**

- **Submit Form**  
  - Type: Form Trigger  
  - Role: Entry point capturing user input fields: Page URL, Country, Language.  
  - Configuration: Dropdowns for Country and Language with predefined options; URL input as text.  
  - Input: Webhook trigger via HTTP POST when form submitted.  
  - Output: JSON containing user selections.

- **Get Country & Language Code**  
  - Type: Code (JavaScript)  
  - Role: Maps user-selected country and language to DataForSEO API codes.  
  - Configuration: Uses two mapping objects hardcoded in JS for country-to-location_code and language-to-language_code.  
  - Key Expressions: Extracts `Country` and `Language` from input, assigns codes or null if unmatched.  
  - Input: Data from Submit Form node.  
  - Output: JSON enriched with `location_code` and `language_code`.  
  - Edge Cases: Missing or unrecognized country/language leads to null codes, which may cause downstream API errors.

- **Session ID**  
  - Type: Code (JavaScript)  
  - Role: Generates a unique session identifier for the workflow execution (format: "cc" + 5 random digits).  
  - Configuration: Random number generation ensures uniqueness per run.  
  - Input: From Get Country & Language Code node output.  
  - Output: JSON with `session_id`.  
  - Edge Cases: Very low risk of collision due to random 5-digit number; no persistence check implemented.

---

#### 2.2 Business Intelligence Data Gathering

**Overview:**  
This core block orchestrates multiple AI and scraping tools to extract in-depth business intelligence data, including website content scraping, internal link discovery, brand and customer research, competitor keyword analysis, and synthesis of all data into predefined business intelligence fields.

**Nodes Involved:**  
- URL_search (Firecrawl HTTP Request)  
- get_internal_links (Sub-workflow Call)  
- Think_tool (AI Synthesis Tool)  
- online_Search (Perplexity AI)  
- deep_research (Perplexity AI, sonar-pro model)  
- serp_Search (LangChain Sub-workflow)  
- ai_Mode (LangChain Sub-workflow)  
- Business Analyst (Google Gemini AI)  
- insert_row_in_data_table (Data Table Insert)  

**Node Details:**

- **URL_search**  
  - Type: HTTP Request (Firecrawl API)  
  - Role: Scrapes main content of the client’s website URL.  
  - Configuration: POST to `https://api.firecrawl.dev/v1/scrape` with JSON body specifying URL, markdown format, main content only, max age 4 hours.  
  - Headers: Authorization Bearer token (must be configured).  
  - Input: URL dynamically injected from AI output `url_search`.  
  - Output: Website content data.  
  - Edge Cases: HTTP request failures, auth errors, invalid URL, empty content.

- **get_internal_links**  
  - Type: Sub-workflow call (LangChain tool workflow)  
  - Role: Retrieves all internal links from the client website to map site structure.  
  - Configuration: Calls external workflow "Laboratory — Business Analysis Workflow Tools" with function=internal_links and URL input.  
  - Input: Page URL from form submission.  
  - Output: List of internal links with metadata.  
  - Edge Cases: No or partial link extraction, timeouts.

- **Think_tool**  
  - Type: LangChain Think Tool (AI synthesis)  
  - Role: Synthesizes and analyzes data at multiple phases (core data extraction, brand analysis, customer intelligence, quality validation).  
  - Configuration: Uses detailed prompt templates embedded in the Business Analyst node.  
  - Input: Outputs of previous research nodes.  
  - Output: Structured textual analysis and extracted intelligence fields.  
  - Edge Cases: AI hallucinations, incomplete synthesis, token limits.

- **online_Search**  
  - Type: Perplexity AI Tool  
  - Role: Performs brand research using AI search assistant in the target language.  
  - Configuration: System prompt enforces contextual accuracy, language adherence, citation rules, and journalistic tone. Returns related questions as well.  
  - Input: Business name and industry keyword from AI overrides.  
  - Output: Brand-related insights and data.  
  - Edge Cases: API rate limits, empty results, language mismatches.

- **deep_research**  
  - Type: Perplexity AI Tool (sonar-pro model)  
  - Role: Conducts in-depth, multi-source market and audience research in target language.  
  - Configuration: Detailed prompt includes structure for executive summary, key insights, strategic recommendations.  
  - Input: Business type and target audience query.  
  - Output: Comprehensive market intelligence report.  
  - Edge Cases: API errors, incomplete data, language issues.

- **serp_Search** and **ai_Mode**  
  - Type: LangChain tool workflow calls  
  - Role: Fetch competitor keywords and perform SERP analysis; ai_Mode is an AI-enhanced variant.  
  - Configuration: Calls external workflow "Laboratory — Business Analysis Workflow Tools" with function parameters `serp_search` or `ai_mode`.  
  - Input: Keyword, location_code, language_code from earlier nodes.  
  - Output: Competitor keyword lists and SERP data.  
  - Edge Cases: API failure, empty keyword sets.

- **Business Analyst**  
  - Type: Google Gemini AI Node  
  - Role: Central AI orchestrator controlling the execution sequence and consolidating all intelligence fields; configured with detailed prompt and strict execution phases.  
  - Configuration: Model "models/gemini-2.5-pro"; temperature 0.2, topP 0.9; maximum 7000 tokens; strict prompt enforcing exact 14-field extraction and validation.  
  - Input: Outputs from all research nodes above.  
  - Output: JSON containing all 14 business intelligence fields.  
  - Edge Cases: AI token limits, hallucinations, incomplete extraction, validation failures.

- **insert_row_in_data_table**  
  - Type: Data Table Tool (Insert)  
  - Role: Inserts the finalized 14 business intelligence fields into the configured data table for storage.  
  - Configuration: Maps AI outputs to table columns; requires exact field adherence.  
  - Input: Data from Business Analyst node.  
  - Output: Confirmation of insert operation.  
  - Edge Cases: Data format mismatch, missing fields, DB connection errors.

---

#### 2.3 Data Validation, Storage & Report Generation

**Overview:**  
This block retrieves stored business data, aggregates it, generates a copy of a Google Docs report template, fills in dynamic placeholders with business data, and prepares the report for final delivery.

**Nodes Involved:**  
- Get Business Data (Data Table Get)  
- Get All Data (Set Node)  
- Make Copy of Template (Google Drive Copy)  
- Change Variables in Doc (Google Docs Update)  
- Generate Doc (Google Drive Download)  
- Add Doc to Drive (Google Drive Upload)  
- Generate PDF (Google Drive Download & Convert)  
- Add PDF To Drive (Google Drive Upload)  

**Node Details:**

- **Get Business Data**  
  - Type: Data Table Get  
  - Role: Fetches the latest inserted data for the current session using session_id filter.  
  - Input: session_id from Session ID node.  
  - Output: JSON with all stored business intelligence fields.  
  - Edge Cases: Missing or multiple rows, session_id mismatch.

- **Get All Data**  
  - Type: Set Node  
  - Role: Consolidates and renames fields from Get Business Data for easy referencing.  
  - Input: Output from Get Business Data.  
  - Output: Structured JSON for document generation.  
  - Edge Cases: Null or missing fields.

- **Make Copy of Template**  
  - Type: Google Drive Copy  
  - Role: Creates a personalized copy of the business analysis report template document.  
  - Configuration: Uses client_name in filename for clarity.  
  - Input: Client name from Get All Data.  
  - Output: New Google Docs file metadata (id, name).  
  - Edge Cases: Drive permission errors, quota limits.

- **Change Variables in Doc**  
  - Type: Google Docs Update  
  - Role: Replaces placeholder tags in the copied document with actual business data fields.  
  - Configuration: Multiple replaceAll actions for each intelligence field, including date.  
  - Input: Document ID from Make Copy of Template, data from Get All Data.  
  - Output: Updated document metadata.  
  - Edge Cases: Incorrect placeholders, failed replacements.

- **Generate Doc**  
  - Type: Google Drive Download  
  - Role: Downloads the updated Google Doc in OpenDocument Text format (.odt).  
  - Input: Document ID from Change Variables in Doc.  
  - Output: Binary ODT file data.  
  - Edge Cases: Download failures, format errors.

- **Add Doc to Drive**  
  - Type: Google Drive Upload  
  - Role: Uploads the generated ODT file back to Google Drive in a specified folder for archiving.  
  - Input: Binary file from Generate Doc, filename from Make Copy of Template.  
  - Output: Drive file metadata including webViewLink.  
  - Edge Cases: Upload failures, folder access issues.

- **Generate PDF**  
  - Type: Google Drive Download + Convert  
  - Role: Converts the updated Google Doc into PDF format for easier sharing.  
  - Input: Document ID from Change Variables in Doc.  
  - Output: Binary PDF file data.  
  - Edge Cases: Conversion errors.

- **Add PDF To Drive**  
  - Type: Google Drive Upload  
  - Role: Uploads the generated PDF file to a Drive folder for distribution.  
  - Input: Binary PDF file from Generate PDF, filename from Make Copy of Template.  
  - Output: Drive file metadata including PDF webViewLink.  
  - Edge Cases: Upload failures.

---

#### 2.4 Report Delivery

**Overview:**  
Final block sends the generated report (both Google Doc and PDF links) by email to a predefined recipient, including session and client metadata.

**Nodes Involved:**  
- Get Links (Set Node)  
- Send email (Gmail Node)  
- Get Session Data (Set Node)  

**Node Details:**

- **Get Links**  
  - Type: Set Node  
  - Role: Aggregates links and metadata for the email body: document name, webViewLinks (Doc and PDF), client name, session ID.  
  - Input: Outputs from Add Doc to Drive, Add PDF To Drive, Get All Data.  
  - Output: JSON for email content.  
  - Edge Cases: Missing or invalid link URLs.

- **Send email**  
  - Type: Gmail Node  
  - Role: Sends the business analysis report email with links and session info.  
  - Configuration: Uses OAuth2 credentials; recipient email hardcoded (should be updated per use).  
  - Input: Email content from Get Links.  
  - Output: Email send status.  
  - Edge Cases: SMTP errors, auth issues, invalid recipient.

- **Get Session Data**  
  - Type: Set Node  
  - Role: Collects session and related codes for potential further processing or logging after email sent.  
  - Input: None (triggered after Send email).  
  - Output: Session metadata JSON.  
  - Edge Cases: None significant.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                         | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                                  |
|----------------------------|--------------------------------|---------------------------------------|---------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Submit Form                | Form Trigger                   | Collect user input (URL, Country, Language) | None                            | Get Country & Language Code     | # -> Input                                                                                                                   |
| Get Country & Language Code | Code                          | Map country and language names to codes | Submit Form                     | Session ID                     | # -> Input                                                                                                                   |
| Session ID                 | Code                          | Generate unique session ID             | Get Country & Language Code     | Business Analyst               | # -> Input                                                                                                                   |
| URL_search                 | HTTP Request (Firecrawl)       | Scrape website main content            | Business Analyst (AI tool)      | Business Analyst               | ## -> Get all Data and prepare report                                                                                        |
| get_internal_links         | Sub-workflow (LangChain)       | Extract internal site links            | Business Analyst (AI tool)      | Business Analyst               | ## -> Get all Data and prepare report                                                                                        |
| Think_tool                 | LangChain AI Tool              | Synthesize data at multiple phases     | Business Analyst (AI tool)      | Business Analyst               | ## -> Get all Data and prepare report                                                                                        |
| online_Search              | Perplexity AI                  | Brand research in target language      | Business Analyst (AI tool)      | Business Analyst               | ## -> Get all Data and prepare report                                                                                        |
| deep_research              | Perplexity AI (sonar-pro)      | In-depth audience and market research  | Business Analyst (AI tool)      | Business Analyst               | ## -> Get all Data and prepare report                                                                                        |
| serp_Search                | LangChain tool workflow        | Competitor keywords & SERP data        | Business Analyst (AI tool)      | Business Analyst               | ### **serp_search**, **get_internal_links**, **ai_mode** tools Sub-workflow inputs: function, keyword, url, location_code, language_code |
| ai_Mode                    | LangChain tool workflow        | AI-enhanced competitor keyword analysis | Business Analyst (AI tool)      | Business Analyst               | ### **serp_search**, **get_internal_links**, **ai_mode** tools Sub-workflow inputs: function, keyword, url, location_code, language_code |
| Business Analyst           | Google Gemini AI               | Central AI orchestrator & data extractor | Session ID, URL_search, get_internal_links, online_Search, deep_research, serp_Search, ai_Mode, Think_tool, insert_row_in_data_table | Get Business Data             | ## -> Get all Data and prepare report                                                                                        |
| insert_row_in_data_table   | Data Table Insert              | Save extracted fields into data table  | Business Analyst (AI tool)      | Business Analyst               | ## -> Get all Data and prepare report                                                                                        |
| Get Business Data          | Data Table Get                 | Retrieve stored business data for session | Business Analyst               | Get All Data                  | **1. `[-Business Data-]`** Data Table (session_id, client_name, etc.)                                                        |
| Get All Data               | Set                           | Aggregate & prepare data for report generation | Get Business Data             | Make Copy of Template         | ## -> Business Analysis Report                                                                                               |
| Make Copy of Template      | Google Drive Copy              | Duplicate Google Docs report template   | Get All Data                   | Change Variables in Doc       | ## -> Business Analysis Report                                                                                               |
| Change Variables in Doc    | Google Docs Update             | Replace placeholders with actual data   | Make Copy of Template          | Generate Doc                  | ## -> Business Analysis Report                                                                                               |
| Generate Doc               | Google Drive Download          | Download updated doc as ODT              | Change Variables in Doc        | Add Doc to Drive              | ## -> Business Analysis Report                                                                                               |
| Add Doc to Drive           | Google Drive Upload            | Upload generated ODT to Drive            | Generate Doc                  | Generate PDF                  | ## -> Business Analysis Report                                                                                               |
| Generate PDF               | Google Drive Download & Convert| Convert Google Doc to PDF                 | Add Doc to Drive              | Add PDF To Drive              | ## -> Business Analysis Report                                                                                               |
| Add PDF To Drive           | Google Drive Upload            | Upload PDF version to Drive               | Generate PDF                  | Get Links                    | ## -> Business Analysis Report                                                                                               |
| Get Links                  | Set                           | Prepare email content with report links  | Add PDF To Drive, Add Doc to Drive, Get All Data | Send email                | ## -> Business Analysis Report                                                                                               |
| Send email                 | Gmail                         | Send report email to recipient           | Get Links                     | Get Session Data              | ## -> Business Analysis Report                                                                                               |
| Get Session Data           | Set                           | Collect session metadata post-email      | Send email                   | None                         | ## -> Business Analysis Report                                                                                               |
| Sticky Notes (multiple)    | Sticky Note                   | Documentation and instructions            | N/A                           | N/A                          | Includes workflow guidance, setup instructions, branding, and links to resources                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Type: `Form Trigger`  
   - Name: "Submit Form"  
   - Configure with fields:  
     - Page URL (text)  
     - Country (dropdown with predefined country list)  
     - Language (dropdown with predefined language list)  
   - Capture webhook URL for form submissions.

2. **Create a Code Node for Country & Language Code Mapping**  
   - Type: `Code`  
   - Name: "Get Country & Language Code"  
   - Paste JS code that maps submitted Country and Language to DataForSEO `location_code` and `language_code`.  
   - Input: Output from "Submit Form".  
   - Output: JSON enriched with `location_code` and `language_code`.

3. **Create a Code Node for Session ID Generation**  
   - Type: `Code`  
   - Name: "Session ID"  
   - Generate unique ID with format "cc" + 5 random digits.  
   - Input: Output from "Get Country & Language Code".  
   - Output: JSON with `session_id`.

4. **Create HTTP Request Node for Firecrawl Scraping**  
   - Type: `HTTP Request`  
   - Name: "URL_search"  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v1/scrape`  
   - Body (JSON): URL from AI override, formats: markdown, onlyMainContent=true, maxAge=14400000 (4 hours)  
   - Header: Authorization Bearer token (set your Firecrawl API token)  
   - Input: From Business Analyst AI tool.

5. **Set up Sub-workflow Node for Internal Links Extraction**  
   - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Name: "get_internal_links"  
   - Configure with workflow ID of "Laboratory — Business Analysis Workflow Tools"  
   - Inputs: URL from "Submit Form", function="internal_links"  
   - Output: List of internal links.

6. **Add LangChain Think Tool Node for AI Synthesis**  
   - Type: `@n8n/n8n-nodes-langchain.toolThink`  
   - Name: "Think_tool"  
   - Used multiple times for phased AI analysis.

7. **Add Perplexity AI Nodes for Market Research**  
   - online_Search: Configure with system prompt enforcing language, date, market context, and query from AI override keyword in target language.  
   - deep_research: Use sonar-pro model with prompt for deep, structured market and audience analysis.

8. **Add LangChain Sub-workflows for SERP and AI Mode Analysis**  
   - serp_Search and ai_Mode: Both call the external workflow "Laboratory — Business Analysis Workflow Tools" with respective function parameters.  
   - Inputs: keyword, language_code, location_code.

9. **Add Google Gemini AI Node for Business Analyst**  
   - Model: "models/gemini-2.5-pro"  
   - Configure with detailed prompt defining 14 business intelligence fields, execution phases, validation rules, and exact formats.  
   - Inputs: Outputs of all prior research and scraping nodes plus Session ID.  
   - Outputs: Structured JSON with business intelligence fields.

10. **Add Data Table Insert Node**  
    - Type: Data Table Tool  
    - Name: "insert_row_in_data_table"  
    - Map the 14 fields from Business Analyst output to the data table columns.  
    - Credential: Configure Data Table access.

11. **Add Data Table Get Node to Retrieve Business Data**  
    - Name: "Get Business Data"  
    - Filter by session_id from Session ID node.

12. **Add Set Node to Aggregate All Data**  
    - Name: "Get All Data"  
    - Map all fields from "Get Business Data" for document insertion.

13. **Create Google Drive Node to Make Copy of Template**  
    - Name: "Make Copy of Template"  
    - Template fileId: ID of the report template document.  
    - Name the copy with client_name.

14. **Create Google Docs Node to Replace Placeholders**  
    - Name: "Change Variables in Doc"  
    - Replace placeholders with data from "Get All Data" node.

15. **Create Google Drive Download Node for ODT**  
    - Name: "Generate Doc"  
    - Download updated Google Doc as .odt.

16. **Create Google Drive Upload Node for ODT**  
    - Name: "Add Doc to Drive"  
    - Upload ODT to target folder.

17. **Create Google Drive Download & Convert Node for PDF**  
    - Name: "Generate PDF"  
    - Convert Google Doc to PDF.

18. **Create Google Drive Upload Node for PDF**  
    - Name: "Add PDF To Drive"  
    - Upload PDF to target folder.

19. **Create Set Node to Prepare Email Content**  
    - Name: "Get Links"  
    - Aggregate document names, webViewLinks, session info.

20. **Create Gmail Node to Send Email**  
    - Name: "Send email"  
    - Configure OAuth2 Gmail credentials.  
    - Recipient email set accordingly.  
    - Send report links and metadata.

21. **Create Set Node for Session Data (optional)**  
    - Name: "Get Session Data"  
    - Collect session and codes post-email for further use.

22. **Connect nodes in order:**  
    Submit Form → Get Country & Language Code → Session ID → Business Analyst (calls research nodes + think_tool) → insert_row_in_data_table → Get Business Data → Get All Data → Make Copy of Template → Change Variables in Doc → Generate Doc → Add Doc to Drive → Generate PDF → Add PDF To Drive → Get Links → Send email → Get Session Data

23. **Credential Setup:**  
    - Firecrawl API token for URL_search node.  
    - Perplexity API credentials for online_Search and deep_research nodes.  
    - Google Gemini API key for Business Analyst node.  
    - Data Table credentials for insert_row_in_data_table and Get Business Data nodes.  
    - Google Drive OAuth2 for all Google Drive and Docs nodes.  
    - Gmail OAuth2 for Send email node.

24. **Create Data Table in n8n:**  
    - Fields: session_id (string), client_name (string), client_country (string), client_language (string), client_website (string), client_description (string), target_audience_personas (string), brand_personality_matrix (string), unique_value_proposition (string), people_ask (string), customer_journey (string), customer_persona_trait (string), eeat_signal_integration (string), GEO_tactic (string), call_to_action (string).

25. **Test each node individually before activating workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow created by [@AgriciDaniel](https://www.youtube.com/@AgriciDaniel)                                                                                                      | Creator credit                                                                                                                             |
| AI Marketing Hub and AI Marketing Hub Pro training programs available at [AI Marketing Hub](https://www.skool.com/ai-marketing-hub) and [AI Marketing Hub Pro](https://www.skool.com/ai-marketing-hub-pro) | Learning resources                                                                                                                         |
| Firecrawl API documentation and signup: https://www.firecrawl.dev/                                                                                                              | Firecrawl scraping service                                                                                                                 |
| Perplexity AI documentation: https://docs.perplexity.ai/                                                                                                                        | Perplexity AI tools                                                                                                                        |
| DataForSEO API: https://dataforseo.com/                                                                                                                                          | Country/language code standards                                                                                                           |
| Google AI Studio for Gemini API key: https://aistudio.google.com                                                                                                                | Gemini AI credentials                                                                                                                      |
| Google API setup guide for n8n: [YouTube Video](https://www.youtube.com/watch?v=3Ai1EPznlAc&t)                                                                                   | Setup instructions for Google Drive, Docs, Gmail API credentials                                                                           |
| Business Analysis Report Template (Google Docs): [Primary Link](https://docs.google.com/document/d/1Q6LvBw6cSGdcNbsEd0Tjo-pVfQsXI1_jUpeUaBsRDdg/edit?usp=sharing)                 | Editable report template used in workflow                                                                                                 |
| Backup report template (docx): [Download Link](https://pub-70c57f6d49284d1da8650039c2add836.r2.dev/Business%20Analysis%20Report%20Template.docx)                                 | Alternative template format                                                                                                                |
| Common formatting errors and strict validation rules enforced to ensure data quality (e.g., exact 5 questions in numbered format, no "https://" in client_website field, etc.)     | Ensures consistent data structure                                                                                                         |
| Sticky notes in workflow provide detailed setup instructions, usage guidelines, and important best practices.                                                                    | Refer to node sticky notes for in-context help                                                                                            |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. All content respects applicable content policies and contains no illegal or protected material. Data processed is legal and publicly available.

---