Generate AI-Powered Competitor Analysis Reports with GPT-4, Apify & Google Docs

https://n8nworkflows.xyz/workflows/generate-ai-powered-competitor-analysis-reports-with-gpt-4--apify---google-docs-6580


# Generate AI-Powered Competitor Analysis Reports with GPT-4, Apify & Google Docs

### 1. Workflow Overview

This workflow automates the generation of AI-powered competitor and market analysis reports by combining data extraction, AI-driven analysis, and document creation, culminating in team delivery via Slack. It is designed for marketing teams, business strategists, and analysts seeking seamless, automated competitor insights.

The workflow consists of the following logical blocks:

- **1.1 Input Initiation:** Manual trigger to start the process.
- **1.2 Data Acquisition:** Web data scraping via Apify.
- **1.3 Data Processing:** Aggregation and preparation of raw data.
- **1.4 AI Analysis:** Strategic interpretation of aggregated data using GPT-4.
- **1.5 Report Generation:** Crafting a polished report with GPT-4.
- **1.6 Document Creation:** Creating a Google Docs report file.
- **1.7 Team Notification:** Delivering the final report to Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initiation

- **Overview:** This block starts the workflow execution manually, allowing users to control when the analysis runs.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type*: Manual Trigger  
    - *Role*: Entry point to initiate the workflow manually.  
    - *Configuration*: Default manual trigger with no additional parameters.  
    - *Inputs*: None (trigger node).  
    - *Outputs*: Triggers the Apify HTTP Request node.  
    - *Edge Cases*: None typical; user must manually trigger.  
    - *Version Requirements*: Standard manual trigger node, no special requirements.

#### 1.2 Data Acquisition

- **Overview:** This block calls Apify's API to scrape competitor or market data from web sources, providing raw information for analysis.
- **Nodes Involved:**  
  - Apify

- **Node Details:**

  - **Apify**  
    - *Type*: HTTP Request  
    - *Role*: Fetches competitor/market data via Apify API.  
    - *Configuration*: HTTP request node; parameters such as URL, method, headers, and authentication presumably set to call Apify actor endpoints.  
    - *Inputs*: Triggered by manual trigger node.  
    - *Outputs*: Sends raw scraped data to the Data Aggregation node.  
    - *Edge Cases*:  
      - API authentication failure (invalid or expired token).  
      - HTTP errors (timeouts, 4xx/5xx errors).  
      - Unexpected or malformed API responses.  
    - *Version Requirements*: HTTP Request node v4.2 supports latest HTTP features.

#### 1.3 Data Processing

- **Overview:** Aggregates and structures the raw data from Apify into a format suitable for AI analysis.
- **Nodes Involved:**  
  - Data Aggregation

- **Node Details:**

  - **Data Aggregation**  
    - *Type*: Code (JavaScript)  
    - *Role*: Processes raw JSON or other data formats from Apify, consolidating and cleaning data.  
    - *Configuration*: Contains custom JavaScript code to parse, filter, and aggregate data points.  
    - *Inputs*: Receives data from Apify node.  
    - *Outputs*: Passes cleaned and structured data to The Strategic Analyst node.  
    - *Edge Cases*:  
      - Unexpected data structures causing parsing errors.  
      - Empty or null data sets leading to empty outputs.  
    - *Version Requirements*: Code node v2 supports ES6+ JavaScript.

#### 1.4 AI Analysis

- **Overview:** Uses GPT-4 via LangChain OpenAI node to interpret the aggregated data, generating strategic insights.
- **Nodes Involved:**  
  - The Strategic Analyst

- **Node Details:**

  - **The Strategic Analyst**  
    - *Type*: LangChain OpenAI (GPT-4)  
    - *Role*: Performs advanced AI-driven competitor and market analysis based on aggregated data.  
    - *Configuration*: OpenAI integration configured to use GPT-4 model. Input prompt likely includes aggregated data with instructions to analyze strategically.  
    - *Inputs*: Receives structured data from Data Aggregation node.  
    - *Outputs*: Passes AI-generated analysis text to The Report Generator.  
    - *Edge Cases*:  
      - API quota limits or authentication errors with OpenAI.  
      - Model response latency or timeout.  
      - Unexpected or incomplete AI outputs.  
    - *Version Requirements*: LangChain OpenAI node v1.8 supports GPT-4.

#### 1.5 Report Generation

- **Overview:** Further refines AI output into a well-formatted report text ready for document creation.
- **Nodes Involved:**  
  - The Report Generator

- **Node Details:**

  - **The Report Generator**  
    - *Type*: LangChain OpenAI (GPT-4)  
    - *Role*: Transforms strategic insights into a comprehensive competitor analysis report.  
    - *Configuration*: Uses GPT-4 with prompt engineering to generate structured report content (e.g., sections, summaries).  
    - *Inputs*: Receives strategic analysis from The Strategic Analyst node.  
    - *Outputs*: Sends finalized report text to Google Docs node.  
    - *Edge Cases*:  
      - Same as The Strategic Analyst regarding API and response issues.  
    - *Version Requirements*: LangChain OpenAI node v1.8.

#### 1.6 Document Creation

- **Overview:** Creates a Google Docs document containing the AI-generated report.
- **Nodes Involved:**  
  - Create Report

- **Node Details:**

  - **Create Report**  
    - *Type*: Google Docs  
    - *Role*: Creates a new Google Docs document, inserting the report content.  
    - *Configuration*:  
      - Uses Google OAuth2 credentials.  
      - Inserts text content received from The Report Generator.  
      - May specify document title, formatting options.  
    - *Inputs*: Receives report text from The Report Generator node.  
    - *Outputs*: Provides document metadata/URL to Deliver Report to Team node.  
    - *Edge Cases*:  
      - Google API quota or permission errors.  
      - Network timeouts.  
      - Formatting or insertion errors with large content.  
    - *Version Requirements*: Google Docs node v2.

#### 1.7 Team Notification

- **Overview:** Sends the generated report link or notification to a Slack channel to inform the team.
- **Nodes Involved:**  
  - Deliver Report to Team

- **Node Details:**

  - **Deliver Report to Team**  
    - *Type*: Slack  
    - *Role*: Posts a message (likely including a Google Docs link) to a Slack channel.  
    - *Configuration*:  
      - Slack credentials (OAuth2) set up.  
      - Channel configured.  
      - Message includes report info from Create Report node.  
    - *Inputs*: Takes document metadata from Create Report node.  
    - *Outputs*: None (final node).  
    - *Edge Cases*:  
      - Slack API rate limits or authentication failures.  
      - Channel missing or permission errors.  
      - Message formatting issues.  
    - *Version Requirements*: Slack node v2.3.

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                      | Input Node(s)                | Output Node(s)                | Sticky Note |
|-----------------------------|----------------------------|------------------------------------|-----------------------------|------------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Workflow start trigger              | None                        | Apify                        |             |
| Apify                       | HTTP Request               | Web data extraction via Apify API  | When clicking ‘Execute workflow’ | Data Aggregation            |             |
| Data Aggregation            | Code                       | Raw data processing and aggregation | Apify                       | The Strategic Analyst         |             |
| The Strategic Analyst       | LangChain OpenAI (GPT-4)   | AI-driven strategic competitor analysis | Data Aggregation            | The Report Generator          |             |
| The Report Generator        | LangChain OpenAI (GPT-4)   | Generates structured report text   | The Strategic Analyst       | Create Report                |             |
| Create Report              | Google Docs                | Creates Google Docs report document | The Report Generator        | Deliver Report to Team         |             |
| Deliver Report to Team      | Slack                      | Sends report notification to Slack | Create Report               | None                         |             |
| Sticky Note                | Sticky Note                | Used for annotations in workflow   | None                        | None                         |             |
| Sticky Note1               | Sticky Note                | Used for annotations in workflow   | None                        | None                         |             |
| Sticky Note2               | Sticky Note                | Used for annotations in workflow   | None                        | None                         |             |
| Sticky Note3               | Sticky Note                | Used for annotations in workflow   | None                        | None                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the manual trigger node:**
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.
   - No parameters needed.

2. **Set up the Apify HTTP Request node:**
   - Add an **HTTP Request** node named `Apify`.
   - Configure the HTTP method and URL to call the Apify actor or API that scrapes competitor/market data.
   - Add authentication headers or tokens per Apify API requirements.
   - Connect `When clicking ‘Execute workflow’` → `Apify`.

3. **Add the Data Aggregation code node:**
   - Add a **Code** node named `Data Aggregation`.
   - Write JavaScript code to parse and aggregate the raw data from Apify into a structured format.
   - Connect `Apify` → `Data Aggregation`.

4. **Add the AI analysis node - The Strategic Analyst:**
   - Add a **LangChain OpenAI** node named `The Strategic Analyst`.
   - Select GPT-4 model.
   - Set up OpenAI API credentials.
   - Configure the prompt to interpret the aggregated data strategically.
   - Connect `Data Aggregation` → `The Strategic Analyst`.

5. **Add the AI report generation node - The Report Generator:**
   - Add another **LangChain OpenAI** node named `The Report Generator`.
   - Use GPT-4 model.
   - Set up prompt to generate a formal competitor analysis report based on strategic insights.
   - Connect `The Strategic Analyst` → `The Report Generator`.

6. **Configure Google Docs node for report creation:**
   - Add a **Google Docs** node named `Create Report`.
   - Set up Google OAuth2 credentials with permissions to create and edit documents.
   - Configure document title and insert content from `The Report Generator`.
   - Connect `The Report Generator` → `Create Report`.

7. **Set up Slack notification node:**
   - Add a **Slack** node named `Deliver Report to Team`.
   - Configure Slack credentials (OAuth2).
   - Set the target Slack channel.
   - Compose a message including the Google Docs document link or summary.
   - Connect `Create Report` → `Deliver Report to Team`.

8. **Test Workflow:**
   - Run the workflow manually via `When clicking ‘Execute workflow’`.
   - Verify data is retrieved, processed, analyzed, reported, document created, and Slack notified.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow leverages GPT-4 through LangChain nodes for advanced AI capabilities.                    | OpenAI GPT-4 API integration via LangChain nodes                                                |
| Google Docs node requires OAuth2 credentials with Google Drive write permissions.                 | Google Cloud Console setup for OAuth2 and Drive API                                            |
| Slack node requires OAuth2 setup for posting messages to team channels.                           | Slack App creation and OAuth2 token generation                                                 |
| Apify actor or API must be pre-configured to scrape desired competitor or market data sources.    | Apify platform documentation: https://apify.com/docs                                            |
| For best performance, monitor API quotas on OpenAI, Google, and Slack to avoid rate limiting.    | Relevant API dashboards for usage monitoring                                                   |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.