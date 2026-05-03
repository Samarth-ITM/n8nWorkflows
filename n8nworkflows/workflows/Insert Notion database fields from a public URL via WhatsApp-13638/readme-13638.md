Insert Notion database fields from a public URL via WhatsApp

https://n8nworkflows.xyz/workflows/insert-notion-database-fields-from-a-public-url-via-whatsapp-13638


# Insert Notion database fields from a public URL via WhatsApp

# Workflow Reference: Insert Notion Database Fields from a Public URL via WhatsApp

This document provides a comprehensive technical breakdown of the n8n workflow designed to automate the extraction of book information from a public URL (sent via WhatsApp) and its insertion into a structured Notion database.

---

### 1. Workflow Overview

The workflow acts as a personal digital librarian. It monitors incoming WhatsApp messages for book titles or URLs, searches for the corresponding book on a specific retailer site (Casa del Libro), scrapes the page for metadata using Apify and standard HTTP requests, cleans the data using JavaScript, and finally populates a Notion database.

**Logical Blocks:**
*   **1.1 Input & Search:** Captures the WhatsApp message and performs a Google Search via SerpAPI to find the most relevant product page.
*   **1.2 Data Extraction (Scraping):** Uses Apify’s website crawler and a direct HTTP GET request to retrieve the raw HTML of the target page.
*   **1.3 AI-Free Data Processing:** A custom JavaScript block parses JSON-LD metadata from the HTML, cleans titles (Smart Title Case), and handles fallbacks.
*   **1.4 Database Integration:** Maps the structured data (Title, Subtitle, Author, Link) into a specific Notion database.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Search
**Overview:** Receives the trigger from WhatsApp and searches for a specific book link.

*   **WhatsApp Trigger:**
    *   **Type:** WhatsApp Trigger
    *   **Role:** Entry point. Listens for `messages` updates via the WhatsApp Business API.
    *   **Configuration:** Configured to trigger on new incoming messages.
*   **Search (SerpAPI):**
    *   **Type:** SerpAPI Node
    *   **Role:** Uses the WhatsApp message body to search Google.
    *   **Expression:** `site:casadellibro.com "{{ $json.messages[0].text.body }}" libro`
    *   **Output:** A list of organic search results.
*   **Extract URL:**
    *   **Type:** Code Node (JavaScript)
    *   **Role:** Isolates the `link`, `title`, and `displayed_link` from the first organic search result to pass to the scraper.

#### 2.2 Data Extraction (Scraping)
**Overview:** Navigates to the URL found in the previous step to pull the full page content.

*   **Scrape URL Apify:**
    *   **Type:** HTTP Request (Apify API)
    *   **Role:** Triggers a synchronous run of the `6sigmag/fast-website-content-crawler` actor.
    *   **Configuration:** Sends the URL extracted from Search as a `startUrls` JSON body. Requires an Apify API Token.
*   **HTTP Request:**
    *   **Type:** HTTP Request
    *   **Role:** Performs a secondary fetch of the URL to ensure the raw HTML is available for the parser.
    *   **Configuration:** Uses a custom `User-Agent` (Chrome/120) and `Accept-Language: es-ES` to ensure the website serves the correct localized content.

#### 2.3 AI-Free Data Processing
**Overview:** Extracts structured data from raw HTML without using expensive LLM tokens.

*   **Code:**
    *   **Type:** Code Node (JavaScript)
    *   **Role:** Technical parser. It searches the HTML for `<script type="application/ld+json">` tags.
    *   **Key Logic:**
        *   `extractJsonLd()`: Finds and parses JSON-LD blocks.
        *   `pickBest()`: Prioritizes types "Book" or "Product".
        *   `smartTitleCaseEs()`: A sophisticated function that normalizes Spanish titles, handling acronyms (USA, ONU) and stopwords (de, la, el) correctly.
    *   **Output:** Returns `titleExact`, `subtitle`, `author`, and `link`.

#### 2.4 Database Integration
**Overview:** Finalizes the process by creating a record in Notion.

*   **Insert in Notion DB:**
    *   **Type:** Notion Node
    *   **Resource:** Database Page
    *   **Configuration:**
        *   **Database:** Linked to a "Books" database.
        *   **Mapping:** 
            *   Title → `title`
            *   Subtitle → `rich_text`
            *   Author → `rich_text`
            *   Link → `url`
            *   Status → Set to "to read" (Select)

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| WhatsApp Trigger | WhatsApp Trigger | Event Listener | None | Search | |
| Search | SerpAPI | Web Search | WhatsApp Trigger | Extract Url | ### 1.- Search URL and extract info |
| Extract Url | Code | Data Formatting | Search | Scrape URL Apify | ### 1.- Search URL and extract info |
| Scrape URL Apify | HTTP Request | Web Scraping | Extract Url | HTTP Request | ### 1.- Search URL and extract info |
| HTTP Request | HTTP Request | HTML Fetching | Scrape URL Apify | Code | ### 2.- Convert data into fields and insert in DB |
| Code | Code | Data Parsing | HTTP Request | Insert in Notion DB | ### 2.- Convert data into fields and insert in DB |
| Insert in Notion DB | Notion | DB Entry | Code | None | ### 2.- Convert data into fields and insert in DB |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Add a **WhatsApp Trigger** node. Connect your WhatsApp Business Cloud API credentials. Set "Updates" to `messages`.
2.  **Search Logic:** Add a **SerpAPI** node. Connect it to the trigger. Set the Query to search for the message body filtered by your target site (e.g., `site:example.com {{ $json.messages[0].text.body }}`).
3.  **URL Extraction:** Add a **Code** node. Use JavaScript to return `results[0].link`.
4.  **Scraping (Apify):** Add an **HTTP Request** node.
    *   **Method:** POST
    *   **URL:** `https://api.apify.com/v2/acts/6sigmag~fast-website-content-crawler/run-sync-get-dataset-items`
    *   **Query Parameter:** `token` (Your Apify Token).
    *   **Body:** JSON containing `startUrls` as an array with the link from step 3.
5.  **HTML Retrieval:** Add another **HTTP Request** node (GET). Use the URL from step 3. In **Options**, set Response Format to `text`. Add a `User-Agent` header to prevent bot blocking.
6.  **Data Parsing:** Add a **Code** node. Copy the provided JavaScript logic to extract `@type: Book` from JSON-LD tags. Ensure the code includes the title-casing function for data cleanliness.
7.  **Notion Setup:** Add a **Notion** node.
    *   **Resource:** Database Page, **Operation:** Create.
    *   **Database ID:** Select your "Books" database.
    *   **Property Mapping:** Map the parsed fields (Title, Subtitle, Author, URL) to your Notion columns.
8.  **Connections:** Connect the nodes in the order described above. Ensure all "Error Handling" settings are set to "Continue Regular Output" if you want the workflow to attempt to finish even if scraping a specific field fails.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| How it works & Setup steps | [Documentation provided in Workflow Sticky Note2] |
| Visual reference of Notion Result | [Image Link](https://res.cloudinary.com/dofqaxnpv/image/upload/v1771874412/books_zdxjju.jpg) |
| Recommended Target Sites | Amazon, Goodreads, or Casa del Libro |
| Customization Tip | Update the scraping logic in the Code node if your target source uses different JSON-LD schemas (e.g., Recipes or Articles). |