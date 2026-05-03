Scrape Google Maps Leads and Find Emails with Apify and Anymailfinder

https://n8nworkflows.xyz/workflows/scrape-google-maps-leads-and-find-emails-with-apify-and-anymailfinder-8472


# Scrape Google Maps Leads and Find Emails with Apify and Anymailfinder

### 1. Workflow Overview

This workflow automates the process of scraping Google Maps leads, cleaning and deduplicating the data, enriching it by finding associated email addresses via Anymailfinder, and saving the results to a NocoDB database. It is designed to facilitate targeted local business lead generation by integrating form inputs, external scraping and email-finding APIs, and a cloud database.

**Target use cases:**  
- Marketing teams seeking local business leads with verified emails.  
- Sales prospecting automation requiring bulk business data extraction and enrichment.  
- Data aggregation projects combining location-based business info with contact details.

**Logical blocks:**  
- **1.1 Input Reception:** Triggered by a form submission capturing search query, city, country code, and number of results.  
- **1.2 Google Maps Scraping:** Calls Apify Google Places Scraper API to retrieve business listings matching the form inputs.  
- **1.3 Data Cleaning and Domain Extraction:** Processes raw scraped data to normalize website URLs, extract domains, and filter out social media domains.  
- **1.4 Deduplication:** Checks the NocoDB database for existing place IDs to avoid duplicates.  
- **1.5 Email Enrichment:** For entries with a valid domain, performs email lookup via Anymailfinder API.  
- **1.6 Data Aggregation and Persistence:** Merges enriched data and saves new leads into NocoDB, then loops to the next item.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Starts the workflow when a user submits the "Local Biz Search" form with parameters to control scraping.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - Type: Form Trigger  
  - Configuration:  
    - Form title: "Local Biz Search"  
    - Fields:  
      - query (required) — search term  
      - city (required) — location city  
      - country code (optional)  
      - number (optional) — max results to scrape  
  - Inputs: External HTTP form submission  
  - Outputs to: Scrape Google Maps node  
  - Edge cases: Missing required fields block trigger; invalid inputs may cause API errors downstream.

---

#### 1.2 Google Maps Scraping

- **Overview:**  
  Sends a POST request to the Apify Google Places Scraper actor with user inputs to get business data.

- **Nodes Involved:**  
  - Scrape Google Maps

- **Node Details:**  
  - Type: HTTP Request  
  - Configuration:  
    - POST URL: Apify's crawler-google-places run-sync endpoint  
    - JSON body includes city, countryCode, locationQuery, maxCrawledPlacesPerSearch, searchStringsArray with query, skipClosedPlaces=true  
    - Authentication: HTTP Query Auth using stored Apify API token  
  - Inputs: Form submission output  
  - Outputs to: Loop Over Items  
  - Edge cases: API quota exceeded, invalid API token, malformed request body, empty results.

- **Sticky Note:**  
  - Links to Apify Google Maps Scraper input settings: https://console.apify.com/actors/nwua9Gu5YrADL7ZDj/input

---

#### 1.3 Loop Over Results and Data Extraction

- **Overview:**  
  Loops over each scraped business item for individual processing and extracts standardized fields.

- **Nodes Involved:**  
  - Loop Over Items  
  - Extract data

- **Node Details:**  
  - Loop Over Items: Splits the array into single items for processing.  
  - Extract data: Sets structured fields from raw JSON: title, address, neighborhood, website, phone, totalScore, subTitle, categories (joined string), city, countryCode, postalCode, placeId, and formatted date.  
  - Inputs: Scrape Google Maps output  
  - Outputs to: Clean Data  
  - Edge cases: Missing fields, malformed data, empty arrays.

---

#### 1.4 Data Cleaning and Domain Extraction

- **Overview:**  
  Cleans website URLs, extracts root website and domain, removes social network domains, normalizes data.

- **Nodes Involved:**  
  - Clean Data

- **Node Details:**  
  - Type: Code Node (JavaScript)  
  - Logic:  
    - Ensures website URLs have protocol  
    - Extracts website root and domain  
    - Removes "www." prefix  
    - Filters out domains belonging to major social networks (facebook.com, instagram.com, etc.)  
    - Sets domain strategy flags  
  - Inputs: Extract data output  
  - Outputs to: Get all the recorded placeIds  
  - Edge cases: Malformed URLs, missing website field, unexpected URL formats.

- **Sticky Note:**  
  - "Remove wrong URLs, and clean extensions"

---

#### 1.5 Deduplication Against NocoDB

- **Overview:**  
  Checks if scraped items already exist in the NocoDB database by comparing place IDs to avoid duplicates.

- **Nodes Involved:**  
  - Get all the recorded placeIds  
  - Avoid duplicates  
  - If (to filter items with placeId)

- **Node Details:**  
  - Get all the recorded placeIds: Queries NocoDB table "m8xjj32j1tvty0z" filtering by current item's placeId to get existing records.  
  - Avoid duplicates: Filters out already recorded placeIds by comparing with all existing ones.  
  - If: Checks existence of placeId to decide next steps.  
  - Inputs: Clean Data output and database query results  
  - Outputs:  
    - If true (new placeId) → Filter items with domain  
    - If false → Loop Over Items (to skip duplicates)  
  - Edge cases: Database connection errors, missing placeId, API rate limits.

- **Sticky Note:**  
  - "Avoid Duplicating Items from the DB"  
  - "If the item already exists we go to the next one in the loop"

---

#### 1.6 Email Enrichment and Final Processing

- **Overview:**  
  For each unique item with a valid domain, search emails via Anymailfinder, merge results, and save to NocoDB.

- **Nodes Involved:**  
  - Filter items with domain (If node)  
  - Get Emails (HTTP Request to Anymailfinder)  
  - Extract Emails (Set node)  
  - Merge1 (Merge node)  
  - Create a row (NocoDB insert)  
  - Loop Over Items (loop back to next item)

- **Node Details:**  
  - Filter items with domain: Checks if the domain field exists  
  - Get Emails: Makes POST request to Anymailfinder API with the domain to find emails; configured with header authentication; continues on error and retries on fail  
  - Extract Emails: Extracts emails and status from Anymailfinder response and enriches with business data from filtered item  
  - Merge1: Merges email data with original business data; if no domain, skips email search and merges original data directly  
  - Create a row: Adds a new record in NocoDB with all combined data fields including emails and validation status  
  - Loop Over Items: Loops back to process next item  
  - Inputs: Items filtered by domain presence and deduplication  
  - Outputs: Iterates until all items processed  
  - Edge cases: API key invalid, request timeout, empty email results, database write failures.

- **Sticky Notes:**  
  - "If we have a domain we search the emails with anymailfinder - check if we have a domain - Yes: search for the email then merge the email with the rest of the data - No: Go to the merge node directly"  
  - "Record the item into the NocoDB and go to the next Item"

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                              | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                     |
|---------------------------|------------------------|----------------------------------------------|-------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| On form submission        | Form Trigger           | Receives user form input                      | (external trigger)             | Scrape Google Maps           |                                                                                                |
| Scrape Google Maps         | HTTP Request           | Calls Apify to scrape Google Maps data       | On form submission             | Loop Over Items              | ## Apify Google Maps Scraper https://console.apify.com/actors/nwua9Gu5YrADL7ZDj/input          |
| Loop Over Items            | SplitInBatches         | Iterates over each scraped business          | Scrape Google Maps             | Extract data, Loop Over Items|                                                                                                |
| Extract data               | Set                    | Extracts and structures relevant fields      | Loop Over Items                | Clean Data                   |                                                                                                |
| Clean Data                 | Code                   | Cleans URLs, extracts domain, filters social| Extract data                  | Get all the recorded placeIds| ## Remove wrong URLs, and clean extensions                                                     |
| Get all the recorded placeIds | NocoDB             | Queries DB to find existing place IDs        | Clean Data                    | Avoid duplicates             |                                                                                                |
| Avoid duplicates           | Code                   | Removes already recorded place IDs           | Get all the recorded placeIds | If                          | ## Avoid Duplicating Items from the DB                                                        |
| If                        | If                     | Checks if placeId exists (to skip duplicates)| Avoid duplicates              | Filter items with domain, Loop Over Items | ## If the item already exists we go to the next one in the loop                           |
| Filter items with domain   | If                     | Checks if domain exists                        | If                           | Get Emails, Merge1           | ## If we have a domain we search the emails with anymailfinder - check if we have a domain...  |
| Get Emails                 | HTTP Request           | Calls Anymailfinder to find emails            | Filter items with domain       | Extract Emails              | [Anymailfinder](https://anymailfinder.com?via=alexandra)                                     |
| Extract Emails             | Set                    | Extracts emails and status, merges business data | Get Emails                 | Merge1                      |                                                                                                |
| Merge1                    | Merge                  | Merges email info with business data          | Filter items with domain, Extract Emails | Create a row           |                                                                                                |
| Create a row               | NocoDB                 | Saves the enriched lead into NocoDB           | Merge1                        | Loop Over Items              | ## Record the item into the NocoDB and go to the next Item                                    |
| Sticky Note               | Sticky Note            | Various comments and links                     | -                             | -                           | ## Links [Anymailfinder](https://anymailfinder.com?via=alexandra) [Apify](https://www.apify.com?fpr=g5q0f) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Form Title: "Local Biz Search"  
   - Fields:  
     - query (required)  
     - city (required)  
     - country code (optional)  
     - number (optional)  
   - No credentials needed. Connect output to "Scrape Google Maps".

2. **Create HTTP Request Node ("Scrape Google Maps"):**  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/compass~crawler-google-places/run-sync-get-dataset-items`  
   - Authentication: Generic HTTP Query Auth with Apify token credentials.  
   - Request Body (JSON):  
     ```json
     {
       "city": "={{ $json.city }}",
       "countryCode": "={{ $json['country code'] }}",
       "locationQuery": "={{ $json.city }}",
       "maxCrawledPlacesPerSearch": "={{ $json.number ? $json.number : 99999 }}",
       "searchStringsArray": ["={{ $json.query }}"],
       "skipClosedPlaces": true
     }
     ```
   - Connect output to "Loop Over Items".

3. **Create SplitInBatches Node ("Loop Over Items"):**  
   - Default settings to split array into single items.  
   - Connect first output to "Extract data" and second output back to itself for looping.

4. **Create Set Node ("Extract data"):**  
   - Assign fields: title, address, neighborhood, website, phone, totalScore, subTitle, categories (joined by comma), city, countryCode, postalCode, placeId, date (format dd/LL/yyyy).  
   - Connect output to "Clean Data".

5. **Create Code Node ("Clean Data"):**  
   - JavaScript logic to:  
     - Normalize website URLs (add protocol if missing)  
     - Extract website root and domain  
     - Remove "www." prefix  
     - Filter out social domain hosts (facebook.com, instagram.com, etc.)  
     - Set domain strategy flags  
   - Connect output to "Get all the recorded placeIds".

6. **Create NocoDB Node ("Get all the recorded placeIds"):**  
   - Operation: Get All  
   - Table: "m8xjj32j1tvty0z"  
   - Project ID: "p75lcduba1trkxn"  
   - Query: Where placeId equals current item's placeId (`=(placeId,eq,{{ $json.placeId }})`)  
   - Authentication: NocoDB API Token credentials  
   - Connect output to "Avoid duplicates".

7. **Create Code Node ("Avoid duplicates"):**  
   - JavaScript filters out items whose placeId already exists in DB results.  
   - Connect output to "If".

8. **Create If Node ("If"):**  
   - Condition: Check if placeId exists (`={{ $json.placeId }}` exists)  
   - True branch: Connect to "Filter items with domain"  
   - False branch: Connect back to "Loop Over Items" to skip duplicates.

9. **Create If Node ("Filter items with domain"):**  
   - Condition: Check if domain exists (`={{ $json.domain }}` exists)  
   - True branch: Connect to "Get Emails"  
   - False branch: Connect directly to "Merge1".

10. **Create HTTP Request Node ("Get Emails"):**  
    - Method: POST  
    - URL: `https://api.anymailfinder.com/v5.1/find-email/company`  
    - Authentication: Header Auth with Anymailfinder API key  
    - Body Parameters: domain = current item's domain  
    - On error: Continue (do not fail workflow)  
    - Retry on fail: Enabled  
    - Connect output to "Extract Emails".

11. **Create Set Node ("Extract Emails"):**  
    - Assign: emails (joined), email_status, domain, and all other business details from "Filter items with domain" node.  
    - Connect output to "Merge1".

12. **Create Merge Node ("Merge1"):**  
    - Merge mode: Append or merge outputs from "Filter items with domain" (no domain branch) and "Extract Emails" (email enriched branch).  
    - Connect output to "Create a row".

13. **Create NocoDB Node ("Create a row"):**  
    - Operation: Create  
    - Table: "m8xjj32j1tvty0z"  
    - Project ID: "p75lcduba1trkxn"  
    - Fields: Map all relevant fields (title, website, phone, emails, email validation, address, neighborhood, rating, categories, city, country, postal code, domain, placeId, date).  
    - Authentication: NocoDB API Token credentials  
    - Connect output back to "Loop Over Items" to process next item.

14. **Add Sticky Notes:**  
    - Add notes for references, explanations, and links as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                               |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| Apify Google Maps Scraper Input Settings                                                      | https://console.apify.com/actors/nwua9Gu5YrADL7ZDj/input                      |
| Anymailfinder API for email enrichment                                                       | https://anymailfinder.com?via=alexandra                                       |
| Apify platform for scraping and automation                                                   | https://www.apify.com?fpr=g5q0f                                               |
| Social networks are filtered from domains to avoid irrelevant email searches                  | Workflow code logic in "Clean Data" node                                      |
| Deduplication is done by placeId against NocoDB database to avoid redundant entries           | NocoDB node configuration and "Avoid duplicates" code node                   |
| The workflow gracefully handles missing domains by skipping email search and continuing      | Controlled by "Filter items with domain" If node                              |
| Email search failures do not stop the workflow; errors are logged and processing continues    | "Get Emails" node configured with "continue on error" and retry on fail       |

---

This document fully describes the structure, logic, and reproduction steps for the "Scrape Google Maps Leads and Find Emails with Apify and Anymailfinder" workflow. It enables both manual reimplementation and automated analysis.