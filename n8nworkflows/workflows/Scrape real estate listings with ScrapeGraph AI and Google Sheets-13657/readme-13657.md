Scrape real estate listings with ScrapeGraph AI and Google Sheets

https://n8nworkflows.xyz/workflows/scrape-real-estate-listings-with-scrapegraph-ai-and-google-sheets-13657


# Scrape real estate listings with ScrapeGraph AI and Google Sheets

This document provides a comprehensive technical analysis of the **Real Estate Listing Scraper** workflow. This automation is designed to extract property listings from real estate websites, process them through AI for structured data extraction, and store the results in Google Sheets.

---

### 1. Workflow Overview

The workflow automates the end-to-end process of real estate data collection. It is designed to handle paginated search result pages, extract specific listing URLs, and then visit each listing to pull detailed attributes (price, size, features, etc.).

**Logical Blocks:**
*   **1.1 Parameter Configuration:** Sets the target URL, pagination depth, and URL structure.
*   **1.2 URL Generation & Discovery:** Creates a list of paginated search pages and uses ScrapeGraph AI to find individual listing links.
*   **1.3 Data Consolidation:** Aggregates all found URLs and prepares them for individual processing.
*   **1.4 Deep Scraping & Extraction:** Visits each individual property page to extract granular data using a defined JSON schema.
*   **1.5 Data Persistence:** Saves or updates the extracted property details into a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Parameter Configuration
*   **Overview:** Defines the starting point for the scraper, including the base URL and how the website handles page numbers.
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Set params`.
*   **Node Details:**
    *   **Set params (Set):** Defines `url` (e.g., Immobiliare.it), `max_pages` (number of pages to crawl), and `page_format_value` (the URL parameter for pagination, like `pag`).
    *   **Edge Cases:** Incorrect `page_format_value` will result in broken URLs in the next step.

#### 2.2 URL Generation & Discovery
*   **Overview:** Generates a list of all search result pages and scrapes them to find the specific links to property listings.
*   **Nodes Involved:** `Generate Urls`, `Split Out`, `Loop Over Items`, `Scrape listings`, `Extract individual URL`, `Wait`.
*   **Node Details:**
    *   **Generate Urls (Code):** JavaScript loop that appends the pagination parameter to the base URL for the requested number of pages.
    *   **Scrape listings (ScrapeGraphAI):** Uses LLM-powered scraping to identify and return URLs matching a specific pattern (e.g., `https://www.xxx.it/annunci/xxxx`).
    *   **Extract individual URL (Information Extractor):** Uses Google Gemini to parse the raw scraping output into a clean JSON array of URLs.
    *   **Wait:** A 10-second delay to prevent rate-limiting or IP blocking by the target website.

#### 2.3 Data Consolidation
*   **Overview:** Collects all URLs found across all paginated search results into a single flat list.
*   **Nodes Involved:** `Aggregate`, `Unified`, `Split Out1`, `Limit`.
*   **Node Details:**
    *   **Aggregate:** Combines outputs from the discovery loop into a single array.
    *   **Unified (Code):** Flattens nested arrays of URLs into a single, clean list.
    *   **Limit (Disabled):** Can be used during testing to process only the first few items.

#### 2.4 Deep Scraping & Extraction
*   **Overview:** Iterates through every unique listing URL found to perform a "deep scrape" of property details.
*   **Nodes Involved:** `Loop Over Items1`, `Extract data`.
*   **Node Details:**
    *   **Extract data (ScrapeGraphAI):** The core intelligence node. It uses a strict **JSON Schema** to extract fields: `title`, `description`, `price`, `area`, `bedrooms`, `bathrooms`, `floor`, `rooms`, `balcony`, `terrace`, `cellar`, `heating`, `air_conditioning`, and `image_urls`.
    *   **Output Schema:** Configured with `renderHeavyJs: true` to ensure content rendered via JavaScript (common on modern real estate sites) is captured.

#### 2.5 Data Persistence
*   **Overview:** Transfers the AI-extracted data into a structured Google Spreadsheet.
*   **Nodes Involved:** `Update real estate listings`.
*   **Node Details:**
    *   **Update real estate listings (Google Sheets):** Uses the `appendOrUpdate` operation. It uses the property **URL** as the unique key to prevent duplicate entries if the workflow is run multiple times.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Set params | Set | Configuration | Trigger | Generate Urls | STEP 1 - Config Params: Enter the GET pagination parameter... |
| Generate Urls | Code | URL Construction | Set params | Split Out | |
| Split Out | Split Out | Data Normalization | Generate Urls | Loop Over Items | |
| Loop Over Items | Split In Batches | Pagination Loop | Split Out, Wait | Aggregate, Scrape listings | |
| Scrape listings | ScrapeGraphAI | Listing Discovery | Loop Over Items | Extract individual URL | |
| Extract individual URL | Info Extractor | Data Parsing | Scrape listings | Wait | |
| Wait | Wait | Rate Limiting | Extract individual URL | Loop Over Items | |
| Aggregate | Aggregate | Data Collection | Loop Over Items | Unified | |
| Unified | Code | Array Flattening | Aggregate | Split Out1 | STEP 2 - Extract Urls: All collected listing URLs are aggregated... |
| Split Out1 | Split Out | Item Isolation | Unified | Limit | STEP 3 - Extract Urls: All collected listing URLs are aggregated... |
| Limit | Limit | Testing Control | Split Out1 | Loop Over Items1 | |
| Loop Over Items1 | Split In Batches | Detail Loop | Limit, Update... | Extract data | |
| Extract data | ScrapeGraphAI | Deep Scraping | Loop Over Items1 | Update... | STEP 4 - extracts detailed property... processes each listing URL... |
| Update... listings | Google Sheets | Data Storage | Extract data | Loop Over Items1 | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Ensure you have credentials for **ScrapeGraphAI**, **Google Gemini (PaLM)**, and **Google Sheets (OAuth2)**.
2.  **Configuration:**
    *   Create a **Set** node. Define strings for `url` (target search page), `max_pages` (e.g., 2), and `page_format_value` (e.g., `pag`).
3.  **URL Logic:**
    *   Add a **Code** node to generate URLs using a `for` loop that combines the base URL with the page parameter.
    *   Connect a **Split Out** node (field: `generated_urls`).
4.  **Discovery Loop:**
    *   Add **Split In Batches** (Loop Over Items).
    *   Connect **ScrapeGraphAI**. Prompt: "Extract the URLs of individual listings."
    *   Connect **Information Extractor** (using Google Gemini). Provide a schema for an array of strings (`url`).
    *   Connect a **Wait** node (10 seconds) and loop back to the **Split In Batches** node.
5.  **Processing Found URLs:**
    *   Connect the "Done" path of the loop to an **Aggregate** node.
    *   Add a **Code** node to flatten the results (`item.json.output.flat()`).
    *   Connect another **Split Out** node to separate individual property URLs.
6.  **Detail Extraction Loop:**
    *   Add a second **Split In Batches** (Loop Over Items1).
    *   Add a **ScrapeGraphAI** node. Use a detailed JSON Schema (Objects: title, price, area, rooms, etc.). Set "Render Heavy JS" to `true`.
7.  **Storage:**
    *   Connect a **Google Sheets** node.
    *   Set operation to `appendOrUpdate`.
    *   Map the JSON result from ScrapeGraphAI to the Sheet columns. Use `URL` as the "Matching Column."
    *   Loop back to the second batch node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets Template | [Clone this Sheet](https://docs.google.com/spreadsheets/d/1jtMyMglBbekD9Z407q8-0vn-cDDXhM81Uj1oAZIJGX8/edit?usp=sharing) |
| Target Testing Site | Optimized for [Immobiliare.it](https://www.immobiliare.it) |
| AI Model used for Extraction | Google Gemini (PaLM) |
| Scraping Engine | ScrapeGraphAI (LLM-based web scraping) |