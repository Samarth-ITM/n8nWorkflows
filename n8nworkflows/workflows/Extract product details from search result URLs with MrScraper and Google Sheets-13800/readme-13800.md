Extract product details from search result URLs with MrScraper and Google Sheets

https://n8nworkflows.xyz/workflows/extract-product-details-from-search-result-urls-with-mrscraper-and-google-sheets-13800


# Extract product details from search result URLs with MrScraper and Google Sheets

This document provides a technical breakdown of the n8n workflow designed to automate the extraction of product information. The process starts with search result URLs stored in Google Sheets, uses **MrScraper** to identify individual product links, scrapes the detailed data from those links, and finally saves the structured results back to a spreadsheet and sends a notification via Gmail.

---

### 1. Workflow Overview

The workflow is designed as a two-tier scraping engine. It first processes "Listing Pages" (search results) to find all relevant product URLs, then navigates to each "Detail Page" to extract specific data fields (like price, description, or attributes).

**Logical Blocks:**
*   **1.1 Data Input:** Retrieves the initial list of search URLs from a Google Sheet.
*   **1.2 Listing Scrape Loop:** Iterates through search URLs using MrScraper’s Listing Agent to find individual product links.
*   **1.3 URL Extraction & Normalization:** A Python-based processor that collects and deduplicates all discovered URLs.
*   **1.4 Detail Scrape Loop:** Iterates through every discovered product URL using MrScraper’s General Agent.
*   **1.5 Data Transformation:** A JavaScript-based processor that flattens complex JSON objects into a flat structure suitable for spreadsheets.
*   **1.6 Export & Notification:** Appends the data to Google Sheets and sends a summary email via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Input
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Get List Search Page`.
*   **Overview:** Triggers the workflow and fetches the source URLs.
*   **Node Details:**
    *   **Get List Search Page (Google Sheets):** Reads rows from a specified spreadsheet. It targets a column containing URLs of search result pages.
    *   **Potential Failures:** Spreadsheet not found, incorrect tab name, or empty rows.

#### 2.2 Listing Scrape Loop
*   **Nodes Involved:** `Looping Listing Page url`, `Run listing agent scraper`.
*   **Overview:** Processes search pages one by one to find product links.
*   **Node Details:**
    *   **Looping Listing Page url (Split in Batches):** Ensures each search URL is processed sequentially.
    *   **Run listing agent scraper (MrScraper):** Uses the `listingAgent` operation. Requires a `scraperId` configured in the MrScraper dashboard.
    *   **Edge Cases:** Page timeouts or "No results found" on the search page.

#### 2.3 URL Extraction & Normalization
*   **Nodes Involved:** `Extract All Url `.
*   **Overview:** Extracts and deduplicates URLs from the raw MrScraper response.
*   **Node Details:**
    *   **Type:** Python Code Node.
    *   **Logic:** It iterates through the nested MrScraper response, looking for `url` keys in the listings and the main `search_link`. It uses a Python `set()` to ensure all URLs are unique before passing them to the next stage.

#### 2.4 Detail Scrape Loop
*   **Nodes Involved:** `Looping Detail Page url`, `Run general agent scraper`.
*   **Overview:** Visits every individual product link found in the previous step.
*   **Node Details:**
    *   **Looping Detail Page url (Split in Batches):** Handles the list of individual product URLs.
    *   **Run general agent scraper (MrScraper):** Uses the `generalAgent` operation to extract structured fields (Title, Price, etc.) from the product page.
    *   **Dependencies:** Relies on a `GENERAL_SCRAPER_ID` from the MrScraper platform.

#### 2.5 Data Transformation
*   **Nodes Involved:** `Flatten Object`.
*   **Overview:** Converts nested AI-generated JSON into a flat format.
*   **Node Details:**
    *   **Type:** JavaScript Code Node.
    *   **Logic:** Recursively traverses the JSON object. If it finds a nested object, it flattens the keys (e.g., `details.price` becomes `details_price`). It also joins arrays into comma-separated strings to ensure compatibility with Google Sheets cells.

#### 2.6 Export & Notification
*   **Nodes Involved:** `Append or update row in sheet`, `Send a message`.
*   **Overview:** Finalizes the process by saving data and notifying the user.
*   **Node Details:**
    *   **Append or update row in sheet (Google Sheets):** Maps the flattened JSON keys to spreadsheet columns. Uses the `appendOrUpdate` operation.
    *   **Send a message (Gmail):** Sends a text-based email notification.
    *   **Configuration:** Requires Gmail OAuth2 and specific recipient details.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **When clicking ‘Execute workflow’** | Manual Trigger | Manual Start | None | Get List Search Page | |
| **Get List Search Page** | Google Sheets | Data Fetching | Manual Trigger | Looping Listing Page url | Phase 1: Load List search Page Url |
| **Looping Listing Page url** | Split In Batches | Loop Controller | Get List Search Page, Run listing agent scraper | Extract All Url, Run listing agent scraper | Phase 2: Scrape Listing Page |
| **Run listing agent scraper** | MrScraper | AI Scraping | Looping Listing Page url | Looping Listing Page url | Phase 2: Scrape Listing Page |
| **Extract All Url** | Code (Python) | Data Parsing | Looping Listing Page url | Looping Detail Page url | Phase 2: Scrape Listing Page |
| **Looping Detail Page url** | Split In Batches | Loop Controller | Extract All Url, Run general agent scraper | Flatten Object, Run general agent scraper | Phase 3: Scrape Detail Data |
| **Run general agent scraper** | MrScraper | AI Scraping | Looping Detail Page url | Looping Detail Page url | Phase 3: Scrape Detail Data |
| **Flatten Object** | Code (JS) | Transformation | Looping Detail Page url | Append or update... | Phase 3: Scrape Detail Data |
| **Append or update row...** | Google Sheets | Data Storage | Flatten Object | Send a message | Phase 4: Export to Spreadsheet + Notify via Gmail |
| **Send a message** | Gmail | Notification | Append or update... | None | Phase 4: Export to Spreadsheet + Notify via Gmail |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation (External):**
    *   Create two scrapers in **MrScraper**: a "Listing Agent" (to find links) and a "General Agent" (to extract details). Note their `scraperId`s.
    *   Prepare a Google Sheet with a column of search result URLs.
2.  **Triggers and Inputs:**
    *   Add a **Manual Trigger** node.
    *   Add a **Google Sheets** node (`Get List Search Page`), set to "Read Rows" from your prepared sheet.
3.  **The First Loop (Search Results):**
    *   Add a **Split in Batches** node (`Looping Listing Page url`).
    *   Connect the batch node to a **MrScraper** node. Set operation to `Listing Agent` and input the `scraperId`.
    *   Connect the MrScraper node back to the batch node to complete the loop.
4.  **URL Processing:**
    *   Connect the "Done" output of the first batch node to a **Python Code** node.
    *   Paste logic to iterate through `data.response` and collect all `url` values into a list of objects.
5.  **The Second Loop (Product Details):**
    *   Add a second **Split in Batches** node (`Looping Detail Page url`) connected to the Python node.
    *   Connect this batch node to a **MrScraper** node. Set operation to `General Agent` and input the second `scraperId`.
    *   Connect this MrScraper node back to the second batch node.
6.  **Data Cleaning:**
    *   Connect the "Done" output of the second batch node to a **JavaScript Code** node.
    *   Paste logic to recursively flatten nested JSON objects and format arrays as strings.
7.  **Output:**
    *   Add a **Google Sheets** node set to `Append or Update`. Map the input fields to your spreadsheet columns.
    *   Add a **Gmail** node to send a success summary to your email.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **MrScraper API Setup** | Ensure API access is enabled in MrScraper settings to allow n8n execution. |
| **Authentication** | This workflow requires OAuth2 credentials for both Google Sheets and Gmail. |
| **Recursive Flattening** | The Flatten Object node is crucial for converting AI's nested responses into spreadsheet-friendly formats. |