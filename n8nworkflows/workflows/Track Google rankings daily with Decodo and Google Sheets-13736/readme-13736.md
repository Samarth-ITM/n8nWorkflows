Track Google rankings daily with Decodo and Google Sheets

https://n8nworkflows.xyz/workflows/track-google-rankings-daily-with-decodo-and-google-sheets-13736


# Track Google rankings daily with Decodo and Google Sheets

This document provides a technical analysis and reference for the n8n workflow: **Track Google Rankings Automatically with Decodo & Google Sheets**.

---

### 1. Workflow Overview

The purpose of this workflow is to automate the daily monitoring of search engine results pages (SERPs) for specific keywords. It utilizes the Decodo API to scrape Google Search results and stores the findings in a Google Sheet for SEO tracking and analysis.

The workflow is organized into four main logical phases:
1.  **Initiation & Configuration:** Triggers the process and defines search parameters (keyword, country, device).
2.  **Data Acquisition:** Executes the search via Decodo.
3.  **Data Parsing & Normalization:** Extracts specific organic search results from the nested API response and formats them into flat records.
4.  **Validation & Logging:** Filters results based on rank thresholds and writes data to either a "Success" sheet or an "Errors" sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Initiation & Configuration
*   **Overview:** Sets the schedule and defines the search criteria used throughout the execution.
*   **Nodes Involved:** `Schedule Run`, `Input Defaults`.
*   **Node Details:**
    *   **Schedule Run (Trigger):** Executes daily at 09:00.
    *   **Input Defaults (Set):** Defines static variables: `keyword` (e.g., "best standing desk"), `country` ("USA"), `language` ("en"), `device` ("desktop"), and `top_n` (the rank threshold, default: 5).

#### 2.2 SERP Acquisition
*   **Overview:** Interacts with the external Decodo service to perform the Google search.
*   **Nodes Involved:** `Fetch Google SERP`.
*   **Node Details:**
    *   **Fetch Google SERP (Decodo Node):** Uses the "Google Search" operation. It passes the keyword, locale, and geo parameters from the previous step. 
    *   **Configuration:** `Continue on Fail` is enabled to ensure the workflow can log errors rather than just stopping.
    *   **Edge Cases:** Invalid API keys or Decodo service downtime will trigger the error path in the next block.

#### 2.3 Data Extraction & Normalization
*   **Overview:** Transforms the complex, nested JSON response from Decodo into individual rows representing search results.
*   **Nodes Involved:** `Check Results`, `Split Payload Results`, `Extract Organic List`, `Split Organic Items`, `Map Result Row`.
*   **Node Details:**
    *   **Check Results (If):** Validates if the `results` array exists and contains at least one item.
    *   **Split Payload Results (Split Out):** Breaks down the initial response wrapper.
    *   **Extract Organic List (Set):** Navigates the deep JSON path (e.g., `results.content.results.results.organic`) to isolate the list of organic rankings.
    *   **Split Organic Items (Split Out):** Turns the array of organic results into individual items for row-by-row processing.
    *   **Map Result Row (Set):** Maps the raw API data to clean fields: `rank`, `title`, `url`, `description`, and `checkedAt` (ISO timestamp).

#### 2.4 Validation & Storage
*   **Overview:** Final filtering and writing to Google Sheets.
*   **Nodes Involved:** `Check Row Valid`, `Save SERP Results`, `Build Empty Error`, `Build Row Error`, `Save SERP Errors`.
*   **Node Details:**
    *   **Check Row Valid (If):** Filters results to ensure the URL is not empty and the rank is less than or equal to the `top_n` value defined in the first block.
    *   **Save SERP Results (Google Sheets):** Appends valid rows to the "SERP_Results" sheet.
    *   **Build Empty Error / Build Row Error (Set):** Constructs an error object containing the error stage (no_results vs invalid_item), error message, and execution ID.
    *   **Save SERP Errors (Google Sheets):** Logs failures to the "SERP_Errors" sheet for debugging.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Run | Schedule Trigger | Workflow entry point | None | Input Defaults | Runs daily and defines the search and scrape with decodo |
| Input Defaults | Set | Configures search params | Schedule Run | Fetch Google SERP | Runs daily and defines the search and scrape with decodo |
| Fetch Google SERP | Decodo | API Scraping | Input Defaults | Check Results | Runs daily and defines the search and scrape with decodo |
| Check Results | If | Check API response | Fetch Google SERP | Split Payload / Build Empty Error | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Split Payload Results | Split Out | Itemize response | Check Results | Extract Organic List | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Extract Organic List | Set | Path extraction | Split Payload Results | Split Organic Items | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Split Organic Items | Split Out | Itemize rankings | Extract Organic List | Map Result Row | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Map Result Row | Set | Data normalization | Split Organic Items | Check Row Valid | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Check Row Valid | If | Rank/URL filter | Map Result Row | Save Results / Build Row Error | Keeps only valid top-ranked rows and appends them to the results sheet. |
| Save SERP Results | Google Sheets | Final data storage | Check Row Valid | None | Keeps only valid top-ranked rows and appends them to the results sheet. |
| Build Empty Error | Set | Format API error | Check Results | Save SERP Errors | Keeps only valid top-ranked rows and appends them to the results sheet. |
| Build Row Error | Set | Format row error | Check Row Valid | Save SERP Errors | Keeps only valid top-ranked rows and appends them to the results sheet. |
| Save SERP Errors | Google Sheets | Error logging | Build Empty Error, Build Row Error | None | Keeps only valid top-ranked rows and appends them to the results sheet. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule Trigger** node set to run daily at your preferred hour.
2.  **Parameters:** Add a **Set** node ("Input Defaults") and define variables: `keyword`, `country`, `language`, `device`, and `top_n` (number).
3.  **API Integration:**
    *   Install the **Decodo** community node if not present.
    *   Add a **Decodo** node, select "Google Search".
    *   Link parameters from "Input Defaults" using expressions (e.g., `{{ $json.keyword }}`).
    *   In the "Settings" tab, enable `Continue on Fail`.
4.  **Data Logic:**
    *   Add an **If** node to check if results exist.
    *   Add a **Split Out** node to handle the `results` array.
    *   Add a **Set** node to extract the organic path: `($json.results.content.results.results.organic || [])`.
    *   Add another **Split Out** node for the `organic` array.
    *   Add a **Set** node to map fields like `rank`, `title`, and `url`. Use `$now.toISO()` for the timestamp.
5.  **Filtering:**
    *   Add an **If** node. Condition 1: `url` is not empty. Condition 2: `rank` <= `top_n`.
6.  **Google Sheets Integration:**
    *   Create a Google Sheet with two tabs: `SERP_Results` and `SERP_Errors`.
    *   Add a **Google Sheets** node (Append) for successful results mapping `URL`, `Rank`, `Title`, `Keyword`, and `Description`.
    *   Add a **Set** node for Error Mapping to capture `errorStage` and `errorMessage`.
    *   Add a final **Google Sheets** node (Append) to the `SERP_Errors` tab.
7.  **Credentials:** Ensure OAuth2 credentials for Google Sheets and API Key for Decodo are configured and selected in the respective nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| How it works: Runs on a daily schedule, calls Decodo, extracts organic results, validates rows, and saves to Google Sheets. | [Flow Summary Note] |
| Setup Checklist: Connect Google Sheets, Add Decodo API credentials, Edit search inputs, Update Sheet IDs. | [Flow Summary Note] |
| Data Mapping: Uses pos_overall or pos for ranking extraction. | Logic inside "Map Result Row" |
| Error Handling: Redirects both "No Results" and "Invalid Items" to a dedicated error logging sheet. | [Results Mapping Note] |