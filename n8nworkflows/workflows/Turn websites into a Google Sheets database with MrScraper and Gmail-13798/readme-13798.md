Turn websites into a Google Sheets database with MrScraper and Gmail

https://n8nworkflows.xyz/workflows/turn-websites-into-a-google-sheets-database-with-mrscraper-and-gmail-13798


# Turn websites into a Google Sheets database with MrScraper and Gmail

# Workflow Reference: Turn Internet Into Database

This document provides a technical breakdown of an n8n workflow designed to automate the extraction of data from websites and store it in Google Sheets, with notifications sent via Gmail. It utilizes **MrScraper** as the core scraping engine.

## 1. Workflow Overview

The workflow automates a multi-stage web scraping pipeline that transitions from high-level domain discovery to deep data extraction. It is designed to turn unstructured web content into a structured Google Sheets database.

The logic is divided into four main functional phases:
*   **1.1 Discovery (Phase 1):** Crawling a target domain to identify relevant listing or category pages.
*   **1.2 Listing Extraction (Phase 2):** Processing discovered listing pages to extract specific detail page URLs.
*   **1.3 Data Scraping (Phase 3):** Navigating to each detail page to extract structured data fields (e.g., prices, descriptions).
*   **1.4 Export & Notification (Phase 4):** Flattening the extracted data, saving it to Google Sheets, and sending a summary via Gmail.

---

## 2. Block-by-Block Analysis

### 2.1 Phase 1: Discover URL (Crawling)
**Overview:** This block identifies the structure of the target website and finds all relevant URLs based on inclusion/exclusion patterns.
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Run map agent scraper`.
*   **Node Details:**
    *   **Run map agent scraper (MrScraper Node):** 
        *   **Role:** Performs a "Map" operation to crawl the domain.
        *   **Configuration:** Requires a `scraperId` and a starting `url`. Supports `includePatterns` (to target specific paths like `/category/`) and `limit` (to control the number of pages found).
        *   **Edge Cases:** Rate limiting on the target domain or overly restrictive include patterns resulting in zero URLs.

### 2.2 Phase 2: Scrape Listing Page
**Overview:** Iterates through the URLs found in Phase 1 to identify specific item/product links.
*   **Nodes Involved:** `Looping Listing Page url`, `Run listing agent scraper`.
*   **Node Details:**
    *   **Looping Listing Page url (Split In Batches):**
        *   **Role:** Manages the iteration over the list of URLs returned by the Map Agent.
    *   **Run listing agent scraper (MrScraper Node):**
        *   **Role:** Uses a "Listing Agent" configuration to extract multiple links (URLs) from a single search/category page.
        *   **Configuration:** Requires a specific `scraperId` optimized for list-view pages.

### 2.3 Phase 3: Scrape Detail Data
**Overview:** The core extraction phase where individual product or post data is gathered and prepared for storage.
*   **Nodes Involved:** `Extract All Url`, `Looping Detail Page url`, `Run general agent scraper`, `Flatten Object`.
*   **Node Details:**
    *   **Extract All Url (Code Node - Python):**
        *   **Role:** Parses the JSON response from the Listing Agent to extract a clean, deduplicated list of detail URLs using a Python `set`.
    *   **Looping Detail Page url (Split In Batches):**
        *   **Role:** Handles the batch processing of individual detail pages.
    *   **Run general agent scraper (MrScraper Node):**
        *   **Role:** Navigates to the specific item page to scrape structured data (title, price, description, etc.).
    *   **Flatten Object (Code Node - JavaScript):**
        *   **Role:** Recursively flattens nested JSON objects into a single-level key-value structure suitable for spreadsheet columns (e.g., `details.price` becomes `details_price`).

### 2.4 Phase 4: Export & Notify
**Overview:** Finalizes the process by updating the database and alerting the user.
*   **Nodes Involved:** `Append or update row in sheet`, `Send a message`.
*   **Node Details:**
    *   **Append or update row in sheet (Google Sheets Node):**
        *   **Role:** Commits the flattened data to a specific Google Sheet.
        *   **Configuration:** Uses `appendOrUpdate` operation. It is recommended to use a unique URL as the matching key to avoid duplicates.
    *   **Send a message (Gmail Node):**
        *   **Role:** Sends a notification once the processing is complete.
        *   **Configuration:** Configurable subject and body to provide a summary of the run.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | Manual Trigger | Entry Point | None | Run map agent scraper | |
| Run map agent scraper | MrScraper | URL Discovery | Manual Trigger | Looping Listing Page url | Find listing/search pages automatically from a domain. |
| Looping Listing Page url | Split In Batches | Loop Controller | Run map agent scraper, Run listing agent scraper | Extract All Url, Run listing agent scraper | Extract detail page URLs from each listing/search page. |
| Run listing agent scraper | MrScraper | List Scraping | Looping Listing Page url | Looping Listing Page url | Extract detail page URLs from each listing/search page. |
| Extract All Url | Code (Python) | Data Parsing | Looping Listing Page url | Looping Detail Page url | |
| Looping Detail Page url | Split In Batches | Loop Controller | Extract All Url, Run general agent scraper | Flatten Object, Run general agent scraper | Extract structured fields from each detail page. |
| Run general agent scraper | MrScraper | Detail Scraping | Looping Detail Page url | Looping Detail Page url | Extract structured fields from each detail page. |
| Flatten Object | Code (JS) | Data Cleaning | Looping Detail Page url | Append or update row in sheet | Normalize the output: flatten nested JSON, format arrays. |
| Append or update row in sheet | Google Sheets | Data Storage | Flatten Object | Send a message | Save results into Google Sheets (Append/Upsert). |
| Send a message | Gmail | Notification | Append or update row in sheet | None | Send a Gmail summary/alert. |

---

## 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create 3 agents in MrScraper: **Map Agent** (discovery), **Listing Agent** (link extraction), and **General Agent** (data extraction). Note their IDs.
    *   Prepare a Google Sheet with headers matching your expected data.

2.  **Initial Discovery:**
    *   Add a **Manual Trigger** node.
    *   Connect it to a **MrScraper** node (Map Agent). Set the `url` and `scraperId`.

3.  **The First Loop (Listings):**
    *   Add a **Split In Batches** node ("Looping Listing Page url").
    *   Connect it to another **MrScraper** node (Listing Agent).
    *   Loop the output of the Listing Agent back into the Batch node.

4.  **Data Extraction:**
    *   Add a **Code Node** (Python) to extract URLs from the Listing Agent's output. Use a `set()` to ensure uniqueness.
    *   Add a second **Split In Batches** node ("Looping Detail Page url").
    *   Connect it to a **MrScraper** node (General Agent) and loop it back.

5.  **Data Transformation:**
    *   Add a **Code Node** (JavaScript) to flatten the JSON objects returned by the General Agent. This ensures the data fits into Google Sheets columns.

6.  **Integration & Notification:**
    *   Add a **Google Sheets** node. Set the operation to `Append or Update`. Map the fields from the Flatten node.
    *   Add a **Gmail** node to send a final "Workflow Complete" notification.

7.  **Credentials:**
    *   Configure **MrScraper API** (API Token).
    *   Configure **Google Sheets OAuth2**.
    *   Configure **Gmail OAuth2**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure MrScraper API access is enabled | Required for n8n communication |
| Google Sheets Upsert Strategy | Use `source_url` as a unique key to prevent data duplication |
| MrScraper Documentation | [https://docs.mrscraper.com](https://docs.mrscraper.com) |
| n8n Split In Batches Guide | Handling large data sets without timeouts |