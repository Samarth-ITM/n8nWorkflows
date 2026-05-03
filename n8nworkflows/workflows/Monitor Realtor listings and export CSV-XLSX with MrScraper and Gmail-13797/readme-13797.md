Monitor Realtor listings and export CSV/XLSX with MrScraper and Gmail

https://n8nworkflows.xyz/workflows/monitor-realtor-listings-and-export-csv-xlsx-with-mrscraper-and-gmail-13797


# Monitor Realtor listings and export CSV/XLSX with MrScraper and Gmail

# Workflow Reference: Real Estate Price Monitoring (Realtor & MrScraper)

This document provides a technical breakdown of the n8n workflow designed to automate the extraction of real estate listings from Realtor.com using MrScraper, followed by data consolidation and export to Google Drive or Gmail.

---

### 1. Workflow Overview

The workflow automates the end-to-end process of monitoring real estate prices for specific regions. It navigates pagination, extracts property details (price, title, location), and generates a structured report.

**Logical Blocks:**
*   **1.1 Config & Trigger:** Initiation via manual execution or a weekly schedule.
*   **1.2 Phase 1: Pagination Discovery:** Uses MrScraper to determine how many result pages exist and generates a list of URLs.
*   **1.3 Phase 2: Data Extraction Loop:** Iterates through all discovered pages to scrape property-level data.
*   **1.4 Phase 3: Export & Delivery:** Merges data, converts it to an Excel (.xlsx) file, and distributes it via Gmail and Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Config & Trigger
This block handles the entry points for the workflow.
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Schedule Trigger`.
*   **Node Details:**
    *   **Manual Trigger:** Allows for on-demand testing.
    *   **Schedule Trigger:** Configured to run every **week**.
    *   **Edge Cases:** Overlapping executions if the scraping process takes longer than the schedule interval (unlikely for weekly runs).

#### 2.2 Phase 1: Pagination Discovery
Calculates the scope of the search by identifying total pages.
*   **Nodes Involved:** `Get Total Page`, `Filtering Result`.
*   **Node Details:**
    *   **Get Total Page (MrScraper):** Calls a specific scraper ID (`0953fe53...`) designed to find the "Total Results" or "Last Page" number on Realtor.com.
    *   **Filtering Result (Code):** 
        *   *Logic:* Takes the total page count and the base URL. It generates an array of URLs (e.g., `.../pg-1`, `.../pg-2`).
        *   *Variables:* `totalPages`, `url`.
    *   **Failure Types:** If the scraper ID is deleted in MrScraper or if Realtor.com changes its URL structure for pagination.

#### 2.3 Phase 2: Scrape and Extract Data
The core engine that visits each page and pulls property information.
*   **Nodes Involved:** `Loop Over Items`, `Extract Data`, `Filter and Merge Result`.
*   **Node Details:**
    *   **Loop Over Items (Split in Batches):** Processes URLs in batches of 20 to avoid timeouts or rate limits.
    *   **Extract Data (MrScraper):** Uses a different scraper ID (`a9cbb053...`) focused on property details (Price, Title, Location). It receives the URL dynamically from the loop.
    *   **Filter and Merge Result (Code):**
        *   *Logic:* Flattens the nested JSON results from MrScraper into a clean list of objects ready for spreadsheet conversion.
    *   **Sub-workflow Note:** This relies on pre-configured scrapers within the MrScraper platform.

#### 2.4 Phase 3: Convert and Send Result
Finalizes the data into a shareable format.
*   **Nodes Involved:** `Convert to File`, `Send a message`, `Upload file`.
*   **Node Details:**
    *   **Convert to File:** Transforms the JSON array into an **XLSX** file. 
        *   *Filename Expression:* `House_Germantown_TN_{{ $today.format("yyyy-MM-dd") }}.xlsx`.
    *   **Send a message (Gmail):** Emails the file as an attachment (currently set to *Disabled* in the JSON).
    *   **Upload file (Google Drive):** Saves the XLSX to the root folder of the connected Google Drive.
    *   **Failure Types:** Authentication expiry for Google/Gmail; "File too large" errors for Gmail attachments.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **When clicking ‘Execute workflow’** | Manual Trigger | Manual Start | None | Get Total Page | Phase 1 — Config |
| **Schedule Trigger** | Schedule Trigger | Periodic Start | None | Get Total Page | Phase 1 — Config |
| **Get Total Page** | MrScraper | Meta-data Scraping | Manual/Schedule | Filtering Result | Phase 1 — Get All Page Url |
| **Filtering Result** | Code | URL Generation | Get Total Page | Loop Over Items | Phase 1 — Get All Page Url |
| **Loop Over Items** | Split In Batches | Batch Control | Filtering Result, Extract Data | Filter & Merge, Extract Data | Phase 2 — Scrape and Extract Data |
| **Extract Data** | MrScraper | Detailed Scraping | Loop Over Items | Loop Over Items | Phase 2 — Scrape and Extract Data |
| **Filter and Merge Result** | Code | Data Normalization | Loop Over Items | Convert to File | Phase 2 — Scrape and Extract Data |
| **Convert to File** | Convert to File | Excel Formatting | Filter & Merge | Gmail, Google Drive | Phase 3 — Convert and Send The Result |
| **Send a message** | Gmail | Email Delivery | Convert to File | None | Phase 3 — Convert and Send The Result |
| **Upload file** | Google Drive | Storage | Convert to File | None | Phase 3 — Convert and Send The Result |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** 
    *   Create two scrapers in **MrScraper**: 
        *   Scraper A: To find the total page count.
        *   Scraper B: To extract property fields (Price, Address, etc.).
2.  **Triggers:** Create a `Manual Trigger` and a `Schedule Trigger` (set to Weekly). Connect both to the next node.
3.  **Discovery (Phase 1):**
    *   Add a **MrScraper** node (`Get Total Page`). Input the Realtor search URL and use Scraper A's ID.
    *   Add a **Code** node (`Filtering Result`). Use Javascript to loop from 1 to the total page count, creating a list of objects containing `url: base-url/pg-X`.
4.  **Extraction (Phase 2):**
    *   Add a **Split in Batches** node. Set batch size to 20.
    *   Add a **MrScraper** node (`Extract Data`). Set the URL parameter to an expression referencing the current batch item: `{{ $json.url }}`. Use Scraper B's ID.
    *   Connect `Extract Data` back to the `Split in Batches` node to continue the loop.
    *   Add a **Code** node to the "Done" output of the loop to flatten the results into a single array.
5.  **Export (Phase 3):**
    *   Add a **Convert to File** node. Set the operation to `XLSX`.
    *   Add a **Google Drive** node. Set the action to `Upload`. Use the binary file from the previous node.
    *   Add a **Gmail** node. Set the action to `Send`. Attach the binary file.
6.  **Credentials:** Ensure `mrscraperApi`, `gmailOAuth2`, and `googleDriveOAuth2Api` are configured and selected in their respective nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **MrScraper Platform** | Setup scrapers here: [https://app.mrscraper.com/](https://app.mrscraper.com/) |
| **Target Use Case** | Real estate investors, agents, and proptech teams. |
| **Customization** | The search URL in the MrScraper nodes can be changed to any Realtor.com search result. |
| **Performance** | Batching is used to ensure stability during large scrapes. |