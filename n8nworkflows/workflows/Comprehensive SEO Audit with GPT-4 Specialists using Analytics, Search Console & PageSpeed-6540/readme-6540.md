Comprehensive SEO Audit with GPT-4 Specialists using Analytics, Search Console & PageSpeed

https://n8nworkflows.xyz/workflows/comprehensive-seo-audit-with-gpt-4-specialists-using-analytics--search-console---pagespeed-6540


# Comprehensive SEO Audit with GPT-4 Specialists using Analytics, Search Console & PageSpeed

---

## 1. Workflow Overview

This workflow automates a comprehensive monthly SEO audit for a specified website, leveraging AI specialists powered by GPT-4. It integrates multiple data sources — Google Analytics 4 (GA4), Google Search Console, Google PageSpeed Insights, and a live website crawl — to generate a detailed, actionable SEO report. The workflow is designed for website owners, SEO consultants, and digital marketing teams focusing on AI and technology sector websites.

The workflow is logically divided into four main blocks:

- **1.1 Data Collection Phase**: Concurrently gathers raw data from Google Analytics, Search Console, PageSpeed Insights, and performs a live crawl of the website homepage for on-page SEO analysis.
- **1.2 Specialist AI Analysis**: Processes each data stream using dedicated GPT-4 AI agents specialized in Analytics, Search Console SEO, Performance, and Technical SEO audits to produce expert reports.
- **1.3 Aggregation & Master Synthesis**: Merges all specialist reports and feeds them to a Master Analyst AI agent, which synthesizes a 360° SEO summary with prioritized recommendations.
- **1.4 Final Output & Storage**: Automatically saves all individual and consolidated reports to Google Sheets for historical tracking and progress monitoring.

---

## 2. Block-by-Block Analysis

### 2.1 Data Collection Phase

**Overview:**  
This block collects essential SEO and performance data from key platforms and the website itself, running in parallel branches for efficiency.

**Nodes Involved:**  
- Schedule Trigger  
- Set Target Website  
- Get a report (Google Analytics)  
- Fetch PageSpeed Data (Google PageSpeed Insights)  
- Search console (Google Search Console API)  
- Crawl Website Homepage (HTTP Request)  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow monthly at 9 AM (configurable).  
  - *Config:* Trigger interval set monthly, fixed hour 9.  
  - *Connections:* Triggers “Set Target Website”.  
  - *Failure cases:* Misconfiguration leads to no runs; time zone considerations.  

- **Set Target Website**  
  - *Type:* Set Node  
  - *Role:* Defines the target domain URL for audit; used by downstream nodes.  
  - *Config:* A string parameter `domain` holding the website URL (default placeholder).  
  - *Connections:* Outputs to Google Analytics, PageSpeed, Search Console, and Crawl Homepage nodes.  
  - *Edge cases:* Must be updated to valid target URL before use; improper URLs cause API errors.

- **Get a report**  
  - *Type:* Google Analytics Node  
  - *Role:* Fetch GA4 data metrics for the target domain.  
  - *Config:* Property ID and metrics/dimensions configured externally (empty placeholders).  
  - *Credentials:* Google Analytics OAuth2.  
  - *Connections:* Outputs to “Format GA Data”.  
  - *Failures:* Auth errors, API quota limits, misconfigured property ID.

- **Fetch PageSpeed Data**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves PageSpeed Insights data for performance and Core Web Vitals.  
  - *Config:* URL dynamically constructed using target domain; categories include performance, accessibility, best practices, SEO.  
  - *Credentials:* Generic HTTP Query Auth with Google API key.  
  - *Connections:* Outputs to “Format PageSpeed Data”.  
  - *Failures:* API key invalid, quota exceeded, no data returned.

- **Search console**  
  - *Type:* HTTP Request  
  - *Role:* Post query request to Google Search Console API to fetch top queries and clicks.  
  - *Config:* URL built using encoded target domain; date range last month; dimension “query”; row limit 10.  
  - *Credentials:* Google OAuth2 API.  
  - *Connections:* Outputs to “Format Search Console Data”.  
  - *Failures:* Authentication issues, empty data for new sites.

- **Crawl Website Homepage**  
  - *Type:* HTTP Request  
  - *Role:* Scrapes homepage HTML content for on-page SEO analysis.  
  - *Config:* URL set to target domain; custom headers include User-Agent for SEO audit.  
  - *Connections:* Outputs to “Extract On-Page SEO Data”.  
  - *Failures:* Site unreachable, HTTP errors, blocked by robots.txt or firewall.

---

### 2.2 Specialist AI Analysis

**Overview:**  
Four GPT-4-based AI agents analyze each data source separately, applying domain-specific expertise to interpret raw data and produce actionable insights.

**Nodes Involved:**  
- Format GA Data (Code)  
- Analytics Specialist Agent (Langchain Agent)  
- Format PageSpeed Data (Code)  
- Performance Specialist Agent (Langchain Agent)  
- Format Search Console Data (Code)  
- SEO Specialist Agent (Langchain Agent)  
- Extract On-Page SEO Data (Code)  
- Score On-Page SEO (Code)  
- Technical Audit Agent (Langchain Agent)  

**Node Details:**

- **Format GA Data**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Aggregates multiple GA4 data points into a summarized textual report with key metrics (sessions, users, pageviews, countries, URLs).  
  - *Config:* Custom JS code with formatting and derived metrics calculation.  
  - *Connections:* Outputs to Analytics Specialist Agent.  
  - *Failures:* Input data missing or malformed; numeric parsing errors.

- **Analytics Specialist Agent**  
  - *Type:* Langchain Agent (OpenAI GPT-4)  
  - *Role:* Expert-level analysis of GA data focusing on traffic, behavior, trends, and AI/tech niche specifics.  
  - *Config:* Prompt includes instructions to output structured JSON plus executive summary.  
  - *Credentials:* OpenAI API key.  
  - *Connections:* Outputs merged with other AI agent outputs.  
  - *Failures:* API errors, prompt failures, rate limits.

- **Format PageSpeed Data**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Processes PageSpeed raw JSON to create a readable report including Lighthouse scores and Core Web Vitals.  
  - *Config:* Checks data validity, extracts scores, formats summary text.  
  - *Connections:* Outputs to Performance Specialist Agent.  
  - *Failures:* Missing or invalid API response.

- **Performance Specialist Agent**  
  - *Type:* Langchain Agent (OpenAI GPT-4)  
  - *Role:* Analyzes PageSpeed report, focusing on Core Web Vitals and technical performance improvements for AI/tech sites.  
  - *Config:* Detailed prompt instructing actionable prioritization and impact estimation.  
  - *Credentials:* OpenAI API key.  
  - *Connections:* Outputs merged for final aggregation.  
  - *Failures:* Similar to other AI nodes.

- **Format Search Console Data**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Converts Search Console API response into a concise text report; handles empty data scenario (new sites).  
  - *Connections:* Outputs to SEO Specialist Agent.  
  - *Failures:* No data or API errors.

- **SEO Specialist Agent**  
  - *Type:* Langchain Agent (OpenAI GPT-4)  
  - *Role:* Evaluates Search Console data, providing SEO strategy recommendations and acceleration plans for new/tech sites.  
  - *Config:* Prompt specifies focus on keyword priorities, content strategy, and ROI-oriented recommendations.  
  - *Credentials:* OpenAI API key.  
  - *Connections:* Outputs merged with other AI results.  
  - *Failures:* API or prompt processing issues.

- **Extract On-Page SEO Data**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Parses homepage HTML to extract SEO elements: title, meta description, H1 tags, image counts with/without alt text.  
  - *Connections:* Outputs to “Score On-Page SEO”.  
  - *Failures:* Empty or invalid HTML.

- **Score On-Page SEO**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Scores the extracted SEO elements into a technical and on-page SEO score out of 100; highlights critical issues and recommendations.  
  - *Connections:* Outputs to Technical Audit Agent.  
  - *Failures:* Missing input data.

- **Technical Audit Agent**  
  - *Type:* Langchain Agent (OpenAI GPT-4)  
  - *Role:* Performs detailed technical SEO audit synthesis with prioritized action plan and ROI estimates.  
  - *Config:* Prompt includes instructions to save summary in Google Sheets before final response.  
  - *Credentials:* OpenAI API key, Google Sheets OAuth2 for appending rows.  
  - *Connections:* Outputs merged for master aggregation.  
  - *Failures:* API errors, Google Sheets write failure.

---

### 2.3 Aggregation & Master Synthesis

**Overview:**  
Collects all four AI-generated reports, consolidates their content, and passes the combined data to a Master Analyst AI agent for strategic synthesis.

**Nodes Involved:**  
- Merge (4-input)  
- Aggregate All Reports (Code)  
- Master Analyst Agent (Langchain Agent)  

**Node Details:**

- **Merge**  
  - *Type:* Merge Node  
  - *Role:* Combines outputs from Analytics, Performance, SEO, and Technical Audit agents into one data stream.  
  - *Config:* Number of inputs set to 4.  
  - *Connections:* Outputs to “Aggregate All Reports”.  
  - *Failures:* Missing inputs if any AI agent fails.

- **Aggregate All Reports**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Generates a consolidated text summary concatenating the first 1000 characters of each report for the master agent.  
  - *Connections:* Outputs to “Master Analyst Agent”.  
  - *Failures:* Missing or malformed inputs.

- **Master Analyst Agent**  
  - *Type:* Langchain Agent (OpenAI GPT-4)  
  - *Role:* Produces a 360° SEO executive summary and prioritized roadmap combining all insights; outputs a tracking dashboard.  
  - *Config:* Prompt mandates output in French with clear, actionable recommendations tailored for AI/tech websites.  
  - *Credentials:* OpenAI API key.  
  - *Connections:* Outputs to “Save Final Report to Sheet”.  
  - *Failures:* API errors, prompt or data format issues.

---

### 2.4 Final Output & Storage

**Overview:**  
Saves the final comprehensive SEO report into a Google Sheet for record-keeping and historical analysis.

**Nodes Involved:**  
- Save Final Report to Sheet (Google Sheets node)  

**Node Details:**

- **Save Final Report to Sheet**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends the final report text and metadata to a designated Google Sheets document.  
  - *Config:* Sheet name and document ID placeholders to be replaced with user’s actual Google Sheet.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Failures:* Authentication errors, API quota limits, sheet not found.

---

## 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                               | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                                         |
|---------------------------|----------------------------------|-----------------------------------------------|--------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger                 | Initiates monthly run                          | -                              | Set Target Website                    | # 🚀 Automated SEO Audit: Setup & Overview … critical setup instructions                                              |
| Set Target Website        | Set                             | Defines target domain URL                       | Schedule Trigger               | Get a report, Fetch PageSpeed Data, Search console, Crawl Website Homepage | # 🚀 Automated SEO Audit: Setup & Overview … critical setup instructions                                              |
| Get a report              | Google Analytics                 | Fetch GA4 data                                 | Set Target Website             | Format GA Data                       | # 1. Data Collection Phase                                                                                           |
| Format GA Data            | Code                            | Aggregate GA4 data and format report           | Get a report                  | Analytics Specialist Agent           | # 2. Specialist AI Analysis                                                                                          |
| Analytics Specialist Agent| Langchain Agent (OpenAI GPT-4)  | Analyze GA data with AI                         | Format GA Data                 | Merge                              | # 2. Specialist AI Analysis                                                                                          |
| Fetch PageSpeed Data      | HTTP Request                    | Fetch PageSpeed Insights data                   | Set Target Website             | Format PageSpeed Data                | # 1. Data Collection Phase                                                                                           |
| Format PageSpeed Data     | Code                            | Format PageSpeed report                         | Fetch PageSpeed Data           | Performance Specialist Agent         | # 2. Specialist AI Analysis                                                                                          |
| Performance Specialist Agent| Langchain Agent (OpenAI GPT-4) | Analyze PageSpeed data with AI                  | Format PageSpeed Data          | Merge                              | # 2. Specialist AI Analysis                                                                                          |
| Search console           | HTTP Request                    | Fetch Search Console data                       | Set Target Website             | Format Search Console Data           | # 1. Data Collection Phase                                                                                           |
| Format Search Console Data| Code                            | Format Search Console report                    | Search console                | SEO Specialist Agent                 | # 2. Specialist AI Analysis                                                                                          |
| SEO Specialist Agent      | Langchain Agent (OpenAI GPT-4)  | Analyze Search Console data with AI             | Format Search Console Data     | Merge                              | # 2. Specialist AI Analysis                                                                                          |
| Crawl Website Homepage    | HTTP Request                    | Scrape homepage HTML                            | Set Target Website             | Extract On-Page SEO Data             | # 1. Data Collection Phase                                                                                           |
| Extract On-Page SEO Data  | Code                            | Extract SEO elements from HTML                  | Crawl Website Homepage         | Score On-Page SEO                   | # 2. Specialist AI Analysis                                                                                          |
| Score On-Page SEO         | Code                            | Score and audit on-page SEO                      | Extract On-Page SEO Data       | Technical Audit Agent                | # 2. Specialist AI Analysis                                                                                          |
| Technical Audit Agent     | Langchain Agent (OpenAI GPT-4)  | Technical SEO audit and action plan             | Score On-Page SEO              | Merge                              | # 2. Specialist AI Analysis                                                                                          |
| Merge                    | Merge                           | Combine all AI reports                          | Analytics Specialist Agent, SEO Specialist Agent, Performance Specialist Agent, Technical Audit Agent | Aggregate All Reports             | # 3. Aggregation & Master Synthesis                                                                                  |
| Aggregate All Reports     | Code                            | Consolidate all reports into summary text       | Merge                         | Master Analyst Agent                | # 3. Aggregation & Master Synthesis                                                                                  |
| Master Analyst Agent      | Langchain Agent (OpenAI GPT-4)  | Generate final strategic summary and roadmap    | Aggregate All Reports          | Save Final Report to Sheet          | # 3. Aggregation & Master Synthesis                                                                                  |
| Save Final Report to Sheet| Google Sheets                   | Store final report in Google Sheets              | Master Analyst Agent           | -                                 | # 4. Final Output & Storage                                                                                          |
| Get GA History            | Google Sheets Tool              | Retrieve historical GA data                      | -                            | Analytics Specialist Agent           | # 2. Specialist AI Analysis                                                                                          |
| Get Search Console History| Google Sheets Tool              | Retrieve historical Search Console data          | -                            | SEO Specialist Agent                 | # 2. Specialist AI Analysis                                                                                          |
| Get PageSpeed History     | Google Sheets Tool              | Retrieve historical PageSpeed data                | -                            | Performance Specialist Agent         | # 2. Specialist AI Analysis                                                                                          |
| Get Technical Audit History| Google Sheets Tool             | Retrieve historical Technical Audit data          | -                            | Technical Audit Agent                | # 2. Specialist AI Analysis                                                                                          |
| Append row               | Google Sheets Tool              | Append report summary for Technical Audit         | Technical Audit Agent          | -                                 | # 2. Specialist AI Analysis                                                                                          |
| Get row(s) in sheet in Google Sheets (multiple)| Google Sheets Tool | Retrieve rows for specialist agents and master      | -                            | Respective AI agents                 | # 2. Specialist AI Analysis                                                                                          |
| OpenAI Chat Model (multiple)| Langchain LM Chat OpenAI      | Language model interface for specialist agents   | -                            | Respective AI Agents                 | # 2. Specialist AI Analysis                                                                                          |
| Sticky Notes (4 total)    | Sticky Note                    | Documentation and setup guidance                  | -                            | -                                 | # All phases: Setup, Data Collection, Specialist AI, Aggregation, Final Output                                        |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Set to trigger monthly at 9 AM.  
   - This node starts the audit automatically.

2. **Add a Set Node "Set Target Website":**  
   - Create a string parameter `domain` for the website URL (e.g., `https://www.your-website.com`).  
   - Connect Schedule Trigger’s output to this node.

3. **Configure Google Analytics Data Fetch:**  
   - Add a Google Analytics node ("Get a report").  
   - Set the GA4 Property ID for your website.  
   - Configure required metrics and dimensions as per your needs.  
   - Connect from "Set Target Website".  
   - Add credentials for Google Analytics OAuth2.

4. **Add a Code Node "Format GA Data":**  
   - Paste the provided JavaScript code to aggregate GA data into a textual summary.  
   - Connect from "Get a report".

5. **Add a Langchain Agent "Analytics Specialist Agent":**  
   - Set model to GPT-4.1-nano with temperature 0.3.  
   - Use OpenAI API credentials.  
   - Paste prompt instructing detailed GA analysis with structured JSON output.  
   - Connect from "Format GA Data".

6. **Configure Google Search Console Data Fetch:**  
   - Add HTTP Request node ("Search console").  
   - Set URL to Google Search Console API with domain parameter.  
   - Use POST method with JSON body querying last month’s data.  
   - Authenticate via Google OAuth2.  
   - Connect from "Set Target Website".

7. **Add Code Node "Format Search Console Data":**  
   - Paste code to format Search Console response or handle empty data.  
   - Connect from "Search console".

8. **Add Langchain Agent "SEO Specialist Agent":**  
   - Configure as GPT-4.1-nano with temperature 0.3 and OpenAI credentials.  
   - Use prompt specialized in Search Console analysis and SEO acceleration strategy.  
   - Connect from "Format Search Console Data".

9. **Configure PageSpeed Insights Data Fetch:**  
   - Add HTTP Request node ("Fetch PageSpeed Data").  
   - URL constructed dynamically with target domain and required categories.  
   - Authenticate using a generic credential with Google API key.  
   - Connect from "Set Target Website".

10. **Add Code Node "Format PageSpeed Data":**  
    - Paste JavaScript code that extracts Lighthouse scores and Core Web Vitals.  
    - Connect from "Fetch PageSpeed Data".

11. **Add Langchain Agent "Performance Specialist Agent":**  
    - Configure GPT-4.1-nano with OpenAI credentials.  
    - Insert prompt focused on performance analysis and technical SEO recommendations.  
    - Connect from "Format PageSpeed Data".

12. **Configure Live Website Crawl:**  
    - Add HTTP Request node ("Crawl Website Homepage").  
    - URL set to target domain.  
    - Set headers with User-Agent and Accept as specified.  
    - Connect from "Set Target Website".

13. **Add Code Node "Extract On-Page SEO Data":**  
    - Paste JavaScript to parse HTML and extract SEO elements (title, meta, H1, images).  
    - Connect from "Crawl Website Homepage".

14. **Add Code Node "Score On-Page SEO":**  
    - Paste JavaScript that scores extracted SEO data and generates audit report.  
    - Connect from "Extract On-Page SEO Data".

15. **Add Langchain Agent "Technical Audit Agent":**  
    - Configure GPT-4.1-nano with OpenAI key.  
    - Use prompt defining technical SEO audit and action plan plus instruction to save results in Sheets.  
    - Connect from "Score On-Page SEO".

16. **Add Merge Node:**  
    - Set number of inputs to 4.  
    - Connect outputs of Analytics Specialist Agent, SEO Specialist Agent, Performance Specialist Agent, and Technical Audit Agent to this Merge node.

17. **Add Code Node "Aggregate All Reports":**  
    - Paste code that concatenates all four reports into a consolidated text summary.  
    - Connect from Merge.

18. **Add Langchain Agent "Master Analyst Agent":**  
    - Configure GPT-4.1-nano with OpenAI credentials.  
    - Insert prompt for master SEO synthesis, actionable roadmap, and KPI dashboard.  
    - Connect from "Aggregate All Reports".

19. **Add Google Sheets Node "Save Final Report to Sheet":**  
    - Configure with your Google Sheet document ID and sheet name.  
    - Connect from "Master Analyst Agent".  
    - Use Google Sheets OAuth2 credentials.

20. **Set Credentials:**  
    - OpenAI API key added to all Langchain Agent nodes.  
    - Google OAuth2 credentials set for Google Analytics, Search Console, and Google Sheets nodes.  
    - Generic credential with Google API key set for PageSpeed Insights HTTP request.

21. **Update All Google Sheets Node Document IDs and Sheet Names:**  
    - Replace placeholders with your actual Google Sheets document ID and sheet names for storing reports and histories.

22. **Optional: Add History Retrieval Nodes:**  
    - Add Google Sheets nodes to retrieve historical GA, Search Console, PageSpeed, and Technical Audit data if desired, connecting these inputs to respective AI agents.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow automates a full monthly SEO audit using specialized AI agents analyzing Google Analytics, Search Console, PageSpeed, and live crawl data. Setup requires updating target domain, credentials, and Google Sheet IDs. | Setup instructions are detailed in the first sticky note at workflow start.                            |
| For custom prompts and further tuning, all AI agents use GPT-4.1-nano with temperature set to 0.3 for controlled creativity and precision.                                                                                     | Node configurations of Langchain Agent nodes.                                                          |
| The live crawl uses a custom User-Agent "Mozilla/5.0 (compatible; n8n-seo-audit/1.0)" to identify automated requests and avoid blocking.                                                                                      | Crawl Website Homepage node headers configuration.                                                      |
| The workflow’s modular design allows easy extension or substitution of any data source or AI prompt, supporting evolving SEO strategies or other domain focuses beyond AI/tech.                                                | Workflow modularity and extensibility.                                                                  |
| Comprehensive error handling is embedded in code nodes for scenarios like empty API responses, invalid data, or missing HTML; however, credential failures and quota limits require manual monitoring and alerts.                | Code node scripts and node failure considerations.                                                      |
| For advanced users: The Technical Audit Agent automatically appends summaries to Google Sheets before producing final output to maintain audit history.                                                                         | Technical Audit Agent prompt instructions.                                                             |
| Official Google API documentation for integration details: https://developers.google.com/apis-explorer                                                                                                                        | Reference for Google Analytics, Search Console, and PageSpeed API usage.                                |
| n8n official docs for OAuth2 and credential setup: https://docs.n8n.io/credentials/google-oauth2/                                                                                                                               | For configuring Google OAuth2 credentials in n8n.                                                       |

---

**Disclaimer:** The content provided stems exclusively from an automated workflow created in n8n, adhering strictly to current content policies with no illegal, offensive, or protected elements. All handled data is legal and public.

---