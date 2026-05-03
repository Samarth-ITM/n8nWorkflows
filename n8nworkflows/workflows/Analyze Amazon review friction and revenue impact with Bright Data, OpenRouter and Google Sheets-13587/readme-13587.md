Analyze Amazon review friction and revenue impact with Bright Data, OpenRouter and Google Sheets

https://n8nworkflows.xyz/workflows/analyze-amazon-review-friction-and-revenue-impact-with-bright-data--openrouter-and-google-sheets-13587


# Analyze Amazon review friction and revenue impact with Bright Data, OpenRouter and Google Sheets

This document provides a comprehensive technical analysis of the **Customer Friction & Conversion-Loss Intelligence Engine** n8n workflow. This system automates the extraction of Amazon product reviews, identifies specific customer pain points (friction), uses AI to quantify the revenue impact of these issues, and logs prioritized insights into Google Sheets.

---

### 1. Workflow Overview

The workflow is designed to transform raw customer feedback into actionable business intelligence. It is structured into six functional logic blocks:

*   **1.1 Input & Configuration:** Captures the target Amazon product URL.
*   **1.2 Review Scraping (Bright Data):** Manages the asynchronous process of triggering, monitoring, and downloading web-scraped review data.
*   **1.3 Friction Signal Detection:** Executes an initial keyword-based filter to identify potentially problematic reviews.
*   **1.4 AI Impact Analysis:** Uses a Large Language Model (LLM) via OpenRouter to categorize friction types and estimate revenue loss potential.
*   **1.5 Scoring & Prioritization:** Applies a weighted mathematical formula to correlate AI findings with specific friction types.
*   **1.6 Reporting & Logging:** Routes the intelligence into specific Google Sheets tabs based on the severity and type of friction detected.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration
This block serves as the entry point, defining which product the engine should analyze.
*   **Nodes Involved:** `Run Friction Analysis`, `Set Product URL`.
*   **Node Details:**
    *   **Run Friction Analysis:** Manual trigger for on-demand execution.
    *   **Set Product URL:** Defines a variable `url` containing the full Amazon product listing link.

#### 2.2 Review Scraping (Bright Data)
Handles the complex lifecycle of a Bright Data "Web Scraper" request, including polling for completion and error handling.
*   **Nodes Involved:** `Start Review Scraping (Bright Data)`, `Wait Before Status Check`, `Check Scraping Status`, `Is Snapshot Ready?`, `Download Review Data`, `Format Bright Data Error`, `Log Scraping Error`.
*   **Node Details:**
    *   **Start Review Scraping:** Triggers a collection job using a specific Dataset ID (`gd_le8e811kzy4ggddlq`).
    *   **Wait Before Status Check:** Pauses for 30 seconds to allow Bright Data to process the request.
    *   **Check Scraping Status:** Polls the snapshot ID to see if the dataset is ready.
    *   **Is Snapshot Ready?:** A conditional check (`status == ready`). If not ready, it loops back to the Wait node.
    *   **Download Review Data:** Retrieves the final structured JSON dataset of reviews.
    *   **Error Handling:** If the initial trigger fails, the workflow formats the error message and logs it to a dedicated Google Sheet tab.

#### 2.3 Friction Signal Detection
A pre-processing step to filter out "noise" (neutral/positive reviews) and focus on specific operational issues.
*   **Nodes Involved:** `Detect Friction Signals`, `Any Friction Detected?`, `No Friction Detected`.
*   **Node Details:**
    *   **Detect Friction Signals (Code):** A Javascript node that scans `review_text` for keywords related to: `delivery_delay`, `returns_friction`, `product_durability_issue`, `sizing_issue`, and `support_issue`.
    *   **Any Friction Detected?:** An IF node checking if the `frictionSignals` array is not empty.
    *   **No Friction Detected:** A fallback node that labels the review as "No issue" if no keywords are matched.

#### 2.4 AI Friction Impact Analysis
Leverages AI to perform deep qualitative analysis on the filtered reviews.
*   **Nodes Involved:** `Prepare Review for AI Analysis`, `AI Friction Impact Analyzer` (Agent), `Friction Impact Analyzer` (Model), `Validate AI Friction Output`.
*   **Node Details:**
    *   **AI Agent:** Configured with a system message to act as an "ecommerce conversion intelligence engine."
    *   **Model:** Uses OpenRouter to access high-performance LLMs.
    *   **Output Parser:** Strictly enforces a JSON schema requiring `frictionCategory`, `estimatedImpact` (Low/Med/High), `impactScore` (0-100), and `reasoning`.

#### 2.5 Scoring & Routing
Quantifies the qualitative AI output into a single "Correlated Score" for prioritization.
*   **Nodes Involved:** `Calculate Correlated Friction Score`, `Is High-Risk Revenue Friction?`, `Tag as High Priority`, `Tag as Moderate / Low Priority`.
*   **Node Details:**
    *   **Calculate Correlated Friction Score (Code):** Adds weights to AI scores based on signal types. Example: `checkout_failure` adds 40 points, while `support_issue` adds 15 points.
    *   **Is High-Risk?:** Branches the workflow if the `correlatedScore` is greater than 60.

#### 2.6 Reporting & Logging
The final stage where data is formatted for stakeholders.
*   **Nodes Involved:** `Log Checkout Optimization Opportunity`, `Prepare Delivery & Returns Risk Flags`, `Log Delivery & Returns Risk`.
*   **Node Details:**
    *   **Log Checkout Optimization Opportunity:** Appends high-priority items to a "Checkout Optimisation List" tab.
    *   **Log Delivery & Returns Risk:** Appends detailed analysis (including AI reasoning and risk booleans) to the "Delivery & Returns Risk Report" tab.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Friction Analysis | Manual Trigger | Entry Point | None | Set Product URL | Input & Configuration |
| Set Product URL | Set | Variable Definition | Run Friction Analysis | Start Review Scraping | Input & Configuration |
| Start Review Scraping | Bright Data | Data Extraction | Set Product URL | Wait, Format Error | Review Scraping Flow |
| Wait Before Status Check | Wait | Polling Delay | Start Scraping, Is Snapshot Ready? | Check Scraping Status | Review Scraping Flow |
| Check Scraping Status | Bright Data | Progress Monitoring | Wait | Is Snapshot Ready? | Review Scraping Flow |
| Is Snapshot Ready? | If | Polling Logic | Check Status | Download Data, Wait | Review Scraping Flow |
| Download Review Data | Bright Data | Data Retrieval | Is Snapshot Ready? | Detect Friction Signals | Review Scraping Flow |
| Detect Friction Signals | Code | Keyword Filtering | Download Data | Any Friction Detected? | Friction Signal Detection |
| Any Friction Detected? | If | Logic Branching | Detect Signals | Prepare AI, No Friction | Friction Signal Detection |
| Prepare Review for AI | Set | AI Data Prep | Any Friction Detected? | AI Analyzer | AI Friction Impact Analysis |
| AI Friction Impact Analyzer | AI Agent | Qualitative Analysis | Prepare Review | Calculate Score | AI Friction Impact Analysis |
| Validate AI Output | Output Parser | Data Integrity | AI Analyzer | AI Analyzer | AI Friction Impact Analysis |
| Calculate Correlated Score | Code | Mathematical Scoring | AI Analyzer | Is High-Risk? | AI Friction Impact Analysis |
| Is High-Risk? | If | Severity Branching | Calculate Score | Tag High, Tag Moderate | Friction Prioritization |
| Tag as High Priority | Set | Labeling | Is High-Risk? | Log Checkout Opp | Friction Prioritization |
| Log Checkout Opp | Google Sheets | Reporting | Tag High Priority | None | Friction Prioritization |
| Prepare Risk Flags | Set | Data Formatting | Is High-Risk? | Log Risk Report | Friction Prioritization |
| Log Risk Report | Google Sheets | Reporting | Prepare Risk Flags | None | Friction Prioritization |
| Format Bright Data Error | Set | Error Formatting | Start Scraping | Log Scraping Error | Review Scraping Flow |
| Log Scraping Error | Google Sheets | Debugging | Format Error | None | Review Scraping Flow |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Create a Google Spreadsheet with three tabs: `Checkout Optimisation List`, `Delivery & Returns Risk Report`, and `log bright data error`.
    *   Add headers to each tab as specified in the "Google Sheets Setup" section below.
2.  **Credentials:**
    *   Configure **Bright Data API** (for scraping).
    *   Configure **OpenRouter API** (for AI analysis).
    *   Configure **Google Sheets OAuth2** (for logging).
3.  **Input Block:**
    *   Add a **Manual Trigger** node.
    *   Add a **Set** node to define the `url` string for the Amazon product.
4.  **Scraping Loop:**
    *   Add a **Bright Data** node (Operation: `Trigger Collection by URL`).
    *   Add a **Wait** node (30 seconds).
    *   Add a **Bright Data** node (Operation: `Monitor Progress Snapshot`).
    *   Add an **If** node to check if the status is "ready". Connect the "false" output back to the Wait node.
    *   Add a **Bright Data** node (Operation: `Download Snapshot`) connected to the "true" output.
5.  **Signal Detection Logic:**
    *   Add a **Code** node. Use Javascript to iterate through reviews and use `.includes()` to find keywords (e.g., "shipping delay", "broke", "refund"). Create an array named `frictionSignals`.
    *   Add an **If** node to check if `frictionSignals` is not empty.
6.  **AI Integration:**
    *   Add an **AI Agent** node.
    *   Connect an **OpenRouter Chat Model** node to it.
    *   Connect a **Structured Output Parser** node. Define a schema for: `frictionCategory`, `estimatedImpact`, `impactScore`, `confidenceScore`, and `reasoning`.
7.  **Scoring Logic:**
    *   Add a **Code** node to calculate a `correlatedScore`. Use weights (e.g., multiply the AI impact score by 0.7 and add fixed weights based on specific friction signals).
8.  **Output Routing:**
    *   Add an **If** node (`correlatedScore > 60`).
    *   **High Path:** Add **Set** node ("Priority: High") -> **Google Sheets** (Append to Checkout tab).
    *   **Risk Path:** Add **Set** node (Map risk booleans) -> **Google Sheets** (Append to Risk Report tab).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Bright Data Setup** | Requires a Scraper ID from [Bright Data](https://brightdata.com). |
| **OpenRouter Setup** | Recommend using models like `anthropic/claude-3-haiku` for cost-effective analysis. [OpenRouter](https://openrouter.ai). |
| **Google Sheets Header (Checkout)** | Brand, Friction Type, Revenue Impact, Priority, Impact Score, ASIN, Timestamp. |
| **Google Sheets Header (Risk)** | brand, productName, asin, reviewText, impactScore, reasoning, deliveryRisk, returnsRisk. |
| **Google Sheets Header (Errors)** | errorSource, errorMessage, errorCode, timestamp, hasError. |