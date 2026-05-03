Prioritize Amazon competitor gaps using Bright Data and Google Sheets

https://n8nworkflows.xyz/workflows/prioritize-amazon-competitor-gaps-using-bright-data-and-google-sheets-13588


# Prioritize Amazon competitor gaps using Bright Data and Google Sheets

This document provides a technical breakdown and reconstruction guide for the **Competitive Assortment & Pricing Gap Intelligence Engine** workflow in n8n.

---

### 1. Workflow Overview

This workflow automates the process of competitive benchmarking for Amazon products. It scrapes product data, uses AI to identify market gaps (missing variants, pricing weaknesses, and bundle opportunities), scores these opportunities based on potential impact, and logs the results into a structured Google Sheets dashboard.

**Logical Blocks:**
1.  **Input & Configuration:** Defines the target Amazon URL.
2.  **Data Collection & Normalization:** Extracts raw data via Bright Data and cleans it into a standard format.
3.  **AI Intelligence Engine:** Uses an LLM to analyze the product against competitive standards and output structured JSON.
4.  **Gap Scoring & Prioritization:** Applies logic to quantify the "opportunity" level of the detected gaps.
5.  **Multi-Channel Reporting:** Routes data to specific Google Sheet tabs based on priority and category (Missing Variants, Bundles, or Pricing Gaps).

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration
Initializes the process by defining which product needs to be analyzed.
*   **Nodes Involved:** `Run Competitive Analysis`, `Set Target URL`.
*   **Node Details:**
    *   **Run Competitive Analysis:** Manual trigger for on-demand execution.
    *   **Set Target URL:** Defines a string variable `URL` containing the full Amazon product link.

#### 2.2 Data Collection & Normalization
Fetches live data and prepares it for AI processing.
*   **Nodes Involved:** `Scrape Product Data (Bright Data)`, `Normalize & Structure Product Data`, `Format Scraping Error Log`, `Log Scraping Failure`.
*   **Node Details:**
    *   **Scrape Product Data (Bright Data):** Uses the Amazon Products dataset (`gd_l7q7dkf244hwjntr0`). It is configured to "Continue on Fail" to allow the Error Log branch to capture timeouts or blocking issues.
    *   **Normalize & Structure Product Data:** A JavaScript Code node that extracts brand, price (buybox, final, or initial), variations (ASIN, price, currency), and categories.
    *   **Error Branch:** If scraping fails, the workflow captures the error message and logs it to a dedicated "log error" sheet.

#### 2.3 AI Intelligence Engine
Analyzes the cleaned data to find strategic opportunities.
*   **Nodes Involved:** `AI Competitive Gap Analyzer`, `Competitive Gap Analyzer (OpenRouter)`, `Validate AI Structured Output`.
*   **Node Details:**
    *   **AI Competitive Gap Analyzer (Agent):** Orchestrates the prompt. It asks the AI to cluster the product, detect missing variants, and suggest bundle opportunities.
    *   **Competitive Gap Analyzer (LM Chat):** The model provider (OpenRouter) used by the agent.
    *   **Validate AI Structured Output:** A LangChain parser ensuring the AI response strictly follows a JSON schema with keys: `productName`, `competitor`, `cluster`, `missingVariants`, `bundleOpportunity`, and `positioningGap`.

#### 2.4 Gap Scoring & Prioritization
Evaluates the AI's findings to determine the urgency of the opportunity.
*   **Nodes Involved:** `Score Pricing & Assortment Gap`, `Is High-Priority Opportunity?`.
*   **Node Details:**
    *   **Score Pricing & Assortment Gap:** A Code node that calculates a `pricingGapScore`.
        *   +40 points for "Premium" or "High" positioning gaps.
        *   +30 points if missing variants are identified.
        *   +30 points if bundle opportunities are found.
    *   **Is High-Priority Opportunity?:** An IF node that checks if the `pricingGapScore` is greater than 50.

#### 2.5 Multi-Channel Reporting
Logs insights into different sheets for stakeholders.
*   **Nodes Involved:** `Format High-Priority Record`, `Log Missing Variants`, `Log Bundle Opportunities`, `Log Pricing & Positioning Gaps`, `Format Standard Opportunity Record`, `Log All Standard Opportunities`.
*   **Node Details:**
    *   **Format Nodes:** Standardize field names (e.g., adding a "Priority" label) before sending to Google Sheets.
    *   **Google Sheets Nodes:** Use the `Append` operation. High-priority items are split across three specialized tabs; lower-priority items go to a general "Standard Opportunities" log.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Competitive Analysis | Manual Trigger | Entry point | None | Set Target URL | Initial configuration block... |
| Set Target URL | Set | Config | Run Competitive Analysis | Scrape Product Data | Initial configuration block... |
| Scrape Product Data (Bright Data) | Bright Data | Data Retrieval | Set Target URL | Normalize Data / Error Log | Scrapes and prepares competitor product data... |
| Normalize & Structure Product Data | Code (JS) | Formatting | Scrape Product Data | AI Gap Analyzer | Scrapes and prepares competitor product data... |
| AI Competitive Gap Analyzer | AI Agent | Analysis | Normalize Data | Score Pricing & Assortment | Analyzes competitor products using AI... |
| Validate AI Structured Output | Output Parser | Schema Control | None | AI Gap Analyzer | Analyzes competitor products using AI... |
| Competitive Gap Analyzer | OpenRouter | LLM Provider | None | AI Gap Analyzer | Analyzes competitor products using AI... |
| Score Pricing & Assortment Gap | Code (JS) | Scoring | AI Gap Analyzer | Is High-Priority? | Analyzes competitor products using AI... |
| Is High-Priority Opportunity? | IF | Routing | Score Pricing | Format High / Format Standard | Filters prioritized opportunities... |
| Format High-Priority Record | Set | Formatting | Is High-Priority? | Log Variants / Bundles / Pricing | Formats and logs high-priority competitive... |
| Log Missing Variants | Google Sheets | Reporting | Format High-Priority | None | Formats and logs high-priority competitive... |
| Log Bundle Opportunities | Google Sheets | Reporting | Format High-Priority | None | Formats and logs high-priority competitive... |
| Log Pricing & Positioning Gaps | Google Sheets | Reporting | Format High-Priority | None | Formats and logs high-priority competitive... |
| Format Standard Opportunity Record | Set | Formatting | Is High-Priority? | Log All Standard | Filters prioritized opportunities... |
| Log All Standard Opportunities | Google Sheets | Reporting | Format Standard | None | Filters prioritized opportunities... |
| Format Scraping Error Log | Set | Error Handling | Scrape Product Data | Log Scraping Failure | Scrapes and prepares competitor product data... |
| Log Scraping Failure | Google Sheets | Error Handling | Format Scraping Error | None | Scrapes and prepares competitor product data... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with 5 tabs: `Missing Variants`, `Bundle Opportunities`, `Pricing & Positioning Gaps`, `All Standard Opportunities`, and `log error`.
    *   Add headers to Row 1 as specified in Section 5.

2.  **Configuration:**
    *   Add a **Manual Trigger** node.
    *   Connect a **Set** node named "Set Target URL". Create a string value named `URL` with an Amazon product link.

3.  **Bright Data Integration:**
    *   Add the **Bright Data** node. Connect your Bright Data credentials.
    *   Set Resource to "Web Scrapper" and select the "Amazon products" dataset.
    *   In the URL field, use the expression `{{ $json.URL }}`.
    *   Set "On Error" to `Continue (populate error output)`.

4.  **JS Normalization:**
    *   Add a **Code** node. Map the raw Bright Data output to a clean JSON object containing `category`, `competitor`, `productName`, `price`, and `variants`.

5.  **AI Setup:**
    *   Add an **AI Agent** node.
    *   Connect an **LM Chat Model** (OpenRouter) and an **Output Parser (Structured)**.
    *   In the Parser, define the JSON schema (Properties: productName, competitor, cluster, missingVariants [array], bundleOpportunity, positioningGap).
    *   In the Agent's "Text" prompt, pass the product data and instruct the AI to respond *only* in the defined JSON format.

6.  **Scoring Logic:**
    *   Add a **Code** node. Create a variable `score`. Use logic to increment `score` based on the length of the `missingVariants` array and keywords in `positioningGap`.

7.  **Routing & Logging:**
    *   Add an **IF** node: `{{ $json.pricingGapScore }} > 50`.
    *   **True Branch:** Use a **Set** node to add the "High" priority label, then connect three **Google Sheets** nodes (Append) for the specific tabs.
    *   **False Branch:** Use a **Set** node to add the "Moderate/low" label, then connect one **Google Sheets** node for the "All Standard Opportunities" tab.

8.  **Error Handling:**
    *   Connect the error output of the Bright Data node to a **Set** node that captures `errorMessage` and `status`, then log it to the "log error" sheet.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Credential: Bright Data** | Required for Amazon web scraping. [brightdata.com](https://brightdata.com) |
| **Credential: OpenRouter** | Required for AI classification. [openrouter.ai](https://openrouter.ai) |
| **Credential: Google Sheets OAuth** | Required for dashboard logging. |
| **Sheet Setup: Missing Variants** | Category, Competitor, Product, Missing Variants, Priority |
| **Sheet Setup: Bundle Opportunities** | category, product, recommendedVariants, bundleOpportunity, priority |
| **Sheet Setup: Pricing Gaps** | category, competitor, product, price, pricingGapScore, positioningGap |
| **Sheet Setup: Standard Ops** | productName, competitor, cluster, missingVariants, bundleOpportunity, positioningGap, pricingGapScore, Priority |
| **Sheet Setup: Error Log** | error, errorSource, errorMessage, status |