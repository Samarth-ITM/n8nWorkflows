Monitor supplier financial distress and risk with Bright Data, OpenRouter and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-supplier-financial-distress-and-risk-with-bright-data--openrouter-and-google-sheets-13586


# Monitor supplier financial distress and risk with Bright Data, OpenRouter and Google Sheets

### 1. Workflow Overview

This workflow is an automated **Financial Distress & Credit Risk Intelligence Engine**. It is designed to proactively monitor suppliers or partner companies by scanning public web data for early warning signs of financial instability, such as bankruptcy filings, insolvency proceedings, and negative financial news.

The logic is divided into five functional blocks:
1.  **Input & Validation:** Defines the target company and parameters.
2.  **Multi-Source Data Acquisition:** Uses Bright Data to scrape Google search results for filings, registers, and news.
3.  **Signal Extraction & Normalization:** Parses raw HTML/text to identify specific distress keywords and consolidates them into a structured format.
4.  **AI-Driven Risk Scoring:** Utilizes Large Language Models (LLMs) via OpenRouter to classify the severity and probability of distress.
5.  **Business Logic & Reporting:** Applies confidence guardrails and business rules to generate actionable recommendations (e.g., "Reject" onboarding or "High Risk" monitoring) and logs results to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Risk Configuration
*   **Overview:** Sets the initial context for the scan, defining which company to research and which indicators to prioritize.
*   **Nodes Involved:** `Run Financial Risk Scan`, `Set Company & Risk Parameters`, `Validate Company Input`.
*   **Node Details:**
    *   **Set Company & Risk Parameters:** (Set) Defines variables like `company` (e.g., Wirecard AG), `country`, and an array of `riskIndicators`.
    *   **Validate Company Input:** (IF) Ensures the `company` string is not empty before proceeding to prevent API waste.

#### 2.2 Multi-Source Signal Collection (Bright Data)
*   **Overview:** Performs targeted web scraping across three distinct categories of financial data.
*   **Nodes Involved:** `Scrape Financial Filings`, `Scrape Insolvency Register`, `Scrape Financial News`, `Log Scraping Error`.
*   **Node Details:**
    *   **Scrape Nodes:** (Bright Data) Each node uses the `n8n_unlocker` zone. They perform Google searches using encoded URI components: `https://www.google.com/search?q={{encodeURIComponent($json.company)}}%20[topic]`.
    *   **Log Scraping Error:** (Google Sheets) Connected to the "Error" output of the scrapers. It logs status codes and error messages to a dedicated "BD log error" tab.

#### 2.3 Signal Extraction & Normalization
*   **Overview:** Converts raw scraped data into a list of confirmed "distress signals" by checking for specific keywords.
*   **Nodes Involved:** `Detect Filing Distress Signals`, `Detect Insolvency Signals`, `Detect News Distress Signals`, `Merge All Distress Signals`, `Categorize & Normalize Signals`.
*   **Node Details:**
    *   **Detect [X] Signals:** (Code) JavaScript nodes that search the HTML body for keywords like "Chapter 11", "liquidation", or "default".
    *   **Merge All Distress Signals:** (Merge) Combines outputs from the three scrapers into a single flow.
    *   **Categorize & Normalize Signals:** (Code) Removes duplicates, maps signals back to their source, and groups them into buckets (Bankruptcy, Insolvency, Restructuring, Filing Issues).

#### 2.4 AI Financial Distress Classification
*   **Overview:** Uses AI to interpret the signals and assign a conservative risk score.
*   **Nodes Involved:** `Prepare Signals for AI Analysis`, `AI Financial Distress Classifier`, `Financial Distress Classifier` (Model), `Validate AI Distress Output` (Parser).
*   **Node Details:**
    *   **AI Financial Distress Classifier:** (LangChain Agent) Takes the distress signals and prompts the LLM to provide a JSON response containing `distressType`, `probability`, `riskScore`, and `explanation`.
    *   **Validate AI Distress Output:** (Output Parser) Enforces a strict JSON schema to ensure the agent's output is machine-readable.

#### 2.5 Confidence Guardrails & Business Decisioning
*   **Overview:** Refines the AI's assessment by checking source reliability and converting scores into business actions.
*   **Nodes Involved:** `Apply Probability Weighting`, `Apply Confidence Guardrails`, `Generate Business Risk Decisions`, `Business Risk Decision Engine` (Model), `Validate Business Decision Output` (Parser).
*   **Node Details:**
    *   **Apply Confidence Guardrails:** (Code) A critical logic node that penalizes scores if they only come from a single source (especially news) and rewards scores confirmed by 3+ sources.
    *   **Generate Business Risk Decisions:** (LangChain Agent) Translates numerical scores into specific business outputs: "Supplier Monitoring", "Onboarding Decision", and "Portfolio Exposure".

#### 2.6 Logging & Action
*   **Overview:** Distributes the final intelligence to relevant Google Sheets tabs.
*   **Nodes Involved:** `Is High Supplier Risk?`, `Log Supplier Monitoring Risk`, `Log Onboarding Decision`, `Log Portfolio Exposure Status`, `No Action Required`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Financial Risk Scan | Manual Trigger | Entry Point | - | Set Company & Risk Parameters | Input & Risk Configuration |
| Set Company & Risk Parameters | Set | Config | Run Financial Risk Scan | Validate Company Input | Input & Risk Configuration |
| Validate Company Input | IF | Logic Gate | Set Company & Risk Parameters | Scrape nodes | Input & Risk Configuration |
| Scrape Financial Filings | Bright Data | Data Collection | Validate Company Input | Detect Filing Signals / Log Error | Multi-Source Signal Collection |
| Scrape Insolvency Register | Bright Data | Data Collection | Validate Company Input | Detect Insolvency Signals / Log Error | Multi-Source Signal Collection |
| Scrape Financial News | Bright Data | Data Collection | Validate Company Input | Detect News Signals / Log Error | Multi-Source Signal Collection |
| Log Scraping Error | Google Sheets | Error Handling | Scrape nodes (Error) | - | Multi-Source Signal Collection |
| Detect Filing Distress Signals | Code | Extraction | Scrape Financial Filings | Merge All Distress Signals | Multi-Source Signal Collection |
| Detect Insolvency Signals | Code | Extraction | Scrape Insolvency Register | Merge All Distress Signals | Multi-Source Signal Collection |
| Detect News Distress Signals | Code | Extraction | Scrape Financial News | Merge All Distress Signals | Multi-Source Signal Collection |
| Merge All Distress Signals | Merge | Data Consolidation | Detect nodes | Categorize & Normalize Signals | Multi-Source Signal Collection |
| Categorize & Normalize Signals | Code | Data Cleanup | Merge All Distress Signals | Prepare Signals for AI Analysis | AI Classification & Risk Scoring |
| Prepare Signals for AI Analysis | Set | Data Mapping | Categorize & Normalize | AI Financial Distress Classifier | AI Classification & Risk Scoring |
| AI Financial Distress Classifier | AI Agent | Analysis | Prepare Signals | Apply Probability Weighting | AI Classification & Risk Scoring |
| Apply Probability Weighting | Code | Score Adjustment | AI Financial Distress Classifier | Apply Confidence Guardrails | AI Classification & Risk Scoring |
| Apply Confidence Guardrails | Code | Logic Guardrails | Apply Probability Weighting | Generate Business Risk Decisions | AI Classification & Risk Scoring |
| Generate Business Risk Decisions | AI Agent | Decision Engine | Apply Confidence Guardrails | IF / Log nodes | Confidence Guardrails & Business Decisioning |
| Is High Supplier Risk? | IF | Filter | Generate Business Risk Decisions | Log Supplier Monitoring / NoOp | Confidence Guardrails & Business Decisioning |
| Log Onboarding Decision | Google Sheets | Output | Generate Business Risk Decisions | - | Confidence Guardrails & Business Decisioning |
| Log Portfolio Exposure Status | Google Sheets | Output | Generate Business Risk Decisions | - | Confidence Guardrails & Business Decisioning |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with four tabs: `Supplier Monitoring`, `Onboarding Snapshot Sheet`, `portfolioExposure`, and `BD log error`.
    *   Add the headers specified in the **Google Sheets Setup** (Section 5) to Row 1 of each tab.
    *   Obtain API keys for **Bright Data** and **OpenRouter**.

2.  **Configuration:**
    *   Add a **Manual Trigger** node.
    *   Add a **Set** node to define `company`, `country`, and an array of `riskIndicators`.
    *   Add an **IF** node to check if `{{ $json.company }}` is not empty.

3.  **Data Acquisition (The Scrapers):**
    *   Create three **Bright Data** nodes. Configure each with the `n8n_unlocker` zone.
    *   Set the URL to a Google search query for: 1) Financial filings, 2) Insolvency register, 3) Financial news.
    *   Enable **Continue On Fail** for these nodes and connect their error outputs to a **Google Sheets** node (Append mode) targeting the "BD log error" tab.

4.  **Signal Processing:**
    *   Connect each Bright Data node to a **Code** node. Use JavaScript to lowercase the body text and search for an array of keywords (e.g., "bankruptcy", "default", "liquidation").
    *   Use a **Merge** node (set to 3 inputs) to collect outputs from the three Code nodes.
    *   Add a final **Code** node to categorize signals into bankruptcy, insolvency, restructuring, or filing issues.

5.  **AI Analysis (Level 1):**
    *   Add an **AI Agent** node (LangChain).
    *   **Model:** Use **LM Chat OpenRouter**.
    *   **Output Parser:** Use **Structured Output Parser** with properties: `distressType` (string), `probability` (enum: Low, Medium, High), `riskScore` (number 0-100), and `explanation` (string).
    *   **System Prompt:** Instruct the AI to be conservative and only use provided signals.

6.  **Refinement Logic:**
    *   Add a **Code** node to adjust the `riskScore` based on the `probability` (e.g., +10 for High probability).
    *   Add a second **Code** node for **Confidence Guardrails**. Implement logic: if only 1 source is found, set `confidenceLevel` to Low. If 3+ sources confirm, set to High. Penalize if the only source is "financial news".

7.  **Business Decisioning (Level 2):**
    *   Add a second **AI Agent** (LangChain) with OpenRouter.
    *   **Output Parser:** Define a schema for three objects: `supplierMonitoring`, `onboardingSnapshot`, and `portfolioExposure`.
    *   **Prompt:** Provide the AI with the business rules (e.g., `confidenceScore >= 75 → Reject`).

8.  **Output Logging:**
    *   Connect the final AI Agent to three **Google Sheets** nodes, each appending data to its respective tab (`Onboarding`, `Portfolio`, and `Monitoring`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Bright Data API** | Required for scraping search results without getting blocked. [brightdata.com](https://brightdata.com) |
| **OpenRouter API** | Provides access to various LLMs for risk classification. [openrouter.ai](https://openrouter.ai) |
| **Google Sheets Tabs** | **Supplier Monitoring:** companyName, country, riskStatus, distressType, probability, confidenceScore, lastUpdated, explanation |
| **Google Sheets Tabs** | **Onboarding Snapshot:** CompanyName, Country, approvalRecommendation, distressType, confidenceScore, TimeStamp |
| **Google Sheets Tabs** | **Portfolio Exposure:** countryName, company, exposureCategory, portfolioRiskWeight |
| **Google Sheets Tabs** | **BD log error:** errorSource, errorMessage, errorCode, timestamp, hasError |