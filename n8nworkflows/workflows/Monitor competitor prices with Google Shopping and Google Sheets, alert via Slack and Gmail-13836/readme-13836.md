Monitor competitor prices with Google Shopping and Google Sheets, alert via Slack and Gmail

https://n8nworkflows.xyz/workflows/monitor-competitor-prices-with-google-shopping-and-google-sheets--alert-via-slack-and-gmail-13836


# Monitor competitor prices with Google Shopping and Google Sheets, alert via Slack and Gmail

# Workflow Reference: Monitor competitor prices and alert via Slack and email

This document provides a technical breakdown of the n8n workflow designed to automate e-commerce competitive intelligence. The workflow tracks product prices on Google Shopping, compares them against internal data, and automates multi-channel notifications.

---

### 1. Workflow Overview

The workflow is a scheduled system that monitors market pricing to maintain competitiveness while protecting profit margins. It operates in four distinct logical phases:

*   **1.1 Data Intake & Normalization:** Retrieves product lists from Google Sheets and standardizes various column naming conventions.
*   **1.2 Competitive Intelligence:** Iterates through products to fetch real-time market data from Google Shopping via the SearchAPI.
*   **1.3 Real-time Analysis & Logging:** Compares internal prices against market averages, calculates suggested adjustments, logs data to a history sheet, and triggers instant alerts for significant price gaps.
*   **1.4 Reporting:** Aggregates all results from a single run to generate a comprehensive summary report sent via Slack and Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block initiates the process and ensures the data is clean and predictable for the rest of the workflow.

*   **Nodes Involved:** `Daily Schedule Trigger`, `Get Products from Sheet`, `Clear Memory and Normalize Columns`.
*   **Node Details:**
    *   **Daily Schedule Trigger:** Fires every 24 hours.
    *   **Get Products from Sheet:** Connects to a Google Sheet (ID required). It fetches the `products` tab.
    *   **Clear Memory and Normalize Columns (Code Node):** 
        *   *Function:* Clears the workflow's static data storage and uses fuzzy matching to find columns like `product_name`, `price`, and `cost` (e.g., it treats "item" or "title" as "product_name").
        *   *Variables:* Extracts `product_name`, `sku`, `my_price`, and `my_cost`.
        *   *Failure Mode:* Returns an error item if no products are found or required columns are missing.

#### 2.2 Competitive Analysis (The Loop)
This block processes each product individually using a batching mechanism.

*   **Nodes Involved:** `Loop Through Each Product`, `Prepare Search Query`, `Search Google Shopping Prices`, `Analyze Pricing Gap`.
*   **Node Details:**
    *   **Loop Through Each Product:** Splits the input list to process one item at a time.
    *   **Search Google Shopping Prices (HTTP Request):** 
        *   *Action:* Calls the `searchapi.io` endpoint.
        *   *Parameters:* Engine is `google_shopping`, query is the product name, region is `us`.
        *   *Auth:* Requires an environment variable `SEARCHAPI_KEY`.
    *   **Analyze Pricing Gap (Code Node):** 
        *   *Logic:* Extracts prices from search results. It calculates the `average`, `lowest`, and `highest` market prices. 
        *   *Business Rules:* 
            *   **Critical Underpriced:** < 85% of market avg. 
            *   **Warning Underpriced:** < 95% of market avg.
            *   **Critical Overpriced:** > 115% of market avg.
            *   **Margin Protection:** Suggested prices are never allowed to fall below 115% of `my_cost`.
        *   *Output:* Returns a detailed JSON object with `signal`, `severity`, `suggested_price`, and `gap_pct`.

#### 2.3 Storage & Instant Alerts
Handles data persistence and immediate notifications for high-priority items.

*   **Nodes Involved:** `Save Result to Memory`, `Log to Price History Sheet`, `Filter Critical or Warning Alerts`, `Send Slack Pricing Alert`, `Continue to Next Product`.
*   **Node Details:**
    *   **Save Result to Memory (Code Node):** Appends the analysis to `workflowStaticData` for the final report.
    *   **Log to Price History Sheet:** Appends a row to a Google Sheet tab named `price_log`.
    *   **Filter Critical or Warning Alerts:** Only allows items with `severity` of "critical" or "warning" to pass.
    *   **Send Slack Pricing Alert:** Sends a formatted block to Slack with emojis (🚨 for critical, ⚠️ for warning) and the suggested action.

#### 2.4 Summary Reporting
Executes after the loop finishes to provide an executive overview.

*   **Nodes Involved:** `Build Daily Summary Report`, `Post Daily Summary to Slack`, `Email Daily Report`.
*   **Node Details:**
    *   **Build Daily Summary Report:** Retrieves all saved items from memory. It constructs both a plain-text summary (for Slack) and a styled HTML table (for Email).
    *   **Email Daily Report (Gmail):** Sends the HTML summary to a specified recipient.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily Schedule Trigger | Schedule | Workflow Entry | (None) | Get Products from Sheet | |
| Get Products from Sheet | Google Sheets | Data Retrieval | Daily Schedule Trigger | Clear Memory... | 1. Load Products. Schedule trigger fires daily. |
| Clear Memory and Normalize Columns | Code | Data Sanitization | Get Products from Sheet | Loop Through Each Product | |
| Loop Through Each Product | SplitInBatches | Iterator | Clear Memory... | Prepare Search Query / Build Daily Summary | |
| Prepare Search Query | Code | Query Formatting | Loop Through Each Product | Search Google Shopping... | 2. Search and Analyze. Each product is searched... |
| Search Google Shopping Prices | HTTP Request | Market Data API | Prepare Search Query | Analyze Pricing Gap | ⚠️ Set your SearchAPI key as an n8n environment variable: SEARCHAPI_KEY. |
| Analyze Pricing Gap | Code | Business Logic | Search Google Shopping... | Save Result to Memory | |
| Save Result to Memory | Code | State Management | Analyze Pricing Gap | Log to... / Filter... | 3. Log and Alert. Results are saved to memory... |
| Log to Price History Sheet | Google Sheets | Data Logging | Save Result to Memory | Continue to Next Product | |
| Filter Critical or Warning Alerts | Filter | Conditional Logic | Save Result to Memory | Send Slack Pricing Alert | |
| Send Slack Pricing Alert | Slack | Instant Notification | Filter... | (None) | |
| Continue to Next Product | Code | Loop Control | Log to Price History Sheet | Loop Through Each Product | |
| Build Daily Summary Report | Code | Aggregation | Loop Through Each Product | Post Daily... / Email... | 4. Daily Summary. After all products are processed... |
| Post Daily Summary to Slack | Slack | Daily Reporting | Build Daily Summary... | (None) | |
| Email Daily Report | Gmail | Daily Reporting | Build Daily Summary... | (None) | ⚠️ Update the email address in this node to your own email. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with two tabs: `products` (columns: `product_name`, `my_price`, `sku`, `my_cost`) and `price_log` (columns matching Section 2.3).
    *   Obtain a [SearchAPI.io](https://www.searchapi.io) key.
2.  **Trigger:** Add a **Schedule Trigger** set to 24 hours.
3.  **Source Data:** Add a **Google Sheets** node ("Get Many Rows"). Connect your credentials and select the `products` sheet.
4.  **Standardization:** Add a **Code Node** to normalize headers. Ensure it handles cases where "Price" might be "my_price" or "selling_price".
5.  **Iteration:** Add a **Split in Batches** node.
6.  **Market Search:**
    *   Inside the loop, add an **HTTP Request** node.
    *   Method: `GET`. URL: `https://www.searchapi.io/api/v1/search`.
    *   Query Parameters: `engine=google_shopping`, `api_key={{$env.SEARCHAPI_KEY}}`, `q={{$json.product_name}}`.
7.  **Analysis Logic:** Add a **Code Node**. Use Javascript to calculate the average of `shopping_results`. Compare it to `my_price`.
8.  **Data Persistence:**
    *   Add a **Code Node** using `$getWorkflowStaticData('global').allResults.push(...)` to save data for the final report.
    *   Add a **Google Sheets** node ("Append Row") to log the analysis into the `price_log` tab.
9.  **Alerting:** Add a **Filter** node checking if `severity` is "critical" or "warning". Connect its "True" path to a **Slack** node.
10. **Loop Back:** Connect the Google Sheets logging node back to the **Split in Batches** node via a simple **Code Node** pass-through.
11. **Final Report:**
    *   Connect the "Done" path of the Loop to a **Code Node**. 
    *   Iterate through the global static data to build an HTML string and a plain text summary.
    *   Connect this to **Slack** and **Gmail** nodes for the final daily distribution.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| SearchAPI Documentation | [searchapi.io/docs/google-shopping](https://www.searchapi.io) |
| Environment Variables in n8n | [n8n Docs: Env Vars](https://docs.n8n.io/hosting/configuration/environment-variables/) |
| Flexible Header Matching | This workflow allows the Google Sheet to be modified (e.g., changing "Product" to "Name") without breaking the logic. |
| Margin Protection | The logic includes a hard check: Suggested Price >= Cost * 1.15. |