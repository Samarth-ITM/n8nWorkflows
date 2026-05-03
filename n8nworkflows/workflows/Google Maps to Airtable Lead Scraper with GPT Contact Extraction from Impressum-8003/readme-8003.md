Google Maps to Airtable Lead Scraper with GPT Contact Extraction from Impressum

https://n8nworkflows.xyz/workflows/google-maps-to-airtable-lead-scraper-with-gpt-contact-extraction-from-impressum-8003


# Google Maps to Airtable Lead Scraper with GPT Contact Extraction from Impressum

### 1. Workflow Overview

This n8n workflow automates a lead scraping process that extracts business contact information from Google Maps and enhances it by scraping the company’s Impressum (legal disclosure) webpage using AI assistance. It targets use cases such as sales lead generation, market research, or business data enrichment by querying businesses in specific cities and countries, then extracting detailed contact info, including decision makers’ names, emails, and phone numbers.

The workflow is logically divided into the following blocks:

- **1.1 Initialization & Input Handling:** Fetches city data from Airtable and triggers the scraping process for each city.
- **1.2 Google Maps Search & Detail Retrieval:** Queries Google Maps Places API for businesses matching search terms, paginates results, and retrieves detailed place information.
- **1.3 Deduplication & Already Scraped Check:** Avoids processing businesses already scraped, managing duplicates and progress status.
- **1.4 Impressum URL Construction & HTML Scraping:** Builds the Impressum page URL from the business website, fetches HTML content.
- **1.5 AI-Powered Information Extraction:** Uses OpenAI GPT model to extract decision maker contact details from Impressum HTML content.
- **1.6 Data Validation & Formatting:** Filters results ensuring valid email presence, cleans and sets fields for output.
- **1.7 Data Writing to Airtable:** Upserts the enriched lead data back into Airtable for further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Input Handling  
**Overview:**  
Starts the workflow by loading city search terms from Airtable and triggers the scraper for each city. Marks cities as processed to allow resuming and avoids duplicate scraping.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Airtable2  
- Filter5  
- Loop Over Items2  
- Execute Workflow  
- Airtable1  
- Sticky Note (explanatory)

**Node Details:**  
- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Receives inputs `Stadt,Land` (City, Country) and `Suchbegriff` (Search Term) to start the process externally.  
  - *Inputs:* Workflow trigger parameters  
  - *Outputs:* Starts scraping for specified city and term  
  - *Failure Modes:* Missing inputs, trigger misfire  

- **Airtable2**  
  - *Type:* Airtable (Search)  
  - *Role:* Retrieves city list from Airtable table "Städte" for processing.  
  - *Config:* Limited to 25 records per run  
  - *Inputs:* None (manual or triggered)  
  - *Outputs:* City records to process  
  - *Failure Modes:* Airtable API rate limits, credential failures  

- **Filter5**  
  - *Type:* Filter  
  - *Role:* Filters cities to only those not marked "Check" and where the country is "Deutschland"  
  - *Conditions:* Auswählen != "Check" AND Land equals "Deutschland"  
  - *Failure Modes:* No matching records, logic errors  

- **Loop Over Items2**  
  - *Type:* SplitInBatches  
  - *Role:* Processes cities in batches for scalability  
  - *Batch Size:* Default (not explicitly set)  
  - *Failure Modes:* Batch processing errors, partial failures  

- **Execute Workflow**  
  - *Type:* Execute Workflow  
  - *Role:* Calls the main scraping sub-workflow "Maps Scraper" with parameters for city and search term  
  - *Inputs:* City and search term from Airtable data  
  - *Config:* Passes `Stadt,Land` and `Suchbegriff` mapped from Airtable fields  
  - *Failure Modes:* Sub-workflow unavailability, param errors  

- **Airtable1**  
  - *Type:* Airtable (Upsert)  
  - *Role:* Marks cities processed by updating Airtable with "Check" status  
  - *Inputs:* City record IDs  
  - *Failure Modes:* Airtable write failures  

- **Sticky Note**  
  - *Content:* Explains city-wise processing, marking completed cities to support resuming if interrupted.

---

#### 1.2 Google Maps Search & Detail Retrieval  
**Overview:**  
Searches Google Maps Places API for businesses matching the search term and city, handles pagination, then fetches detailed place info for each business.

**Nodes Involved:**  
- Searchword (set)  
- search Place (HTTP Request)  
- Split Out (SplitOut)  
- Place Details (HTTP Request)

**Node Details:**  
- **Searchword**  
  - *Type:* Set  
  - *Role:* Constructs the search query combining the search term and city-country string, e.g., "Seo Agentur Frauenfeld, Schweiz"  
  - *Inputs:* Receives from workflow trigger or Airtable  
  - *Outputs:* JSON containing `Suchbegriff` string  

- **search Place**  
  - *Type:* HTTP Request  
  - *Role:* Calls Google Maps Places Text Search API with the constructed query  
  - *Config:* Uses pagination via `next_page_token` to fetch additional pages, with controlled maxRequests=1 and interval=20 seconds  
  - *Authentication:* HTTP Query Auth with Google API key stored in credentials  
  - *Failure Modes:* API rate limits, invalid API key, malformed query, network timeouts  

- **Split Out**  
  - *Type:* SplitOut  
  - *Role:* Splits the array of search results (`results`) into individual items to process each place separately  
  - *Inputs:* Output of search Place  
  - *Outputs:* One item per place  

- **Place Details**  
  - *Type:* HTTP Request  
  - *Role:* Fetches detailed place information from Google Places Details API using `place_id` from each result  
  - *Config:* Also supports pagination in case of multiple detail pages (rare for details)  
  - *Authentication:* Same as search Place node  
  - *Failure Modes:* API errors, missing place_id, rate limits  

---

#### 1.3 Deduplication & Already Scraped Check  
**Overview:**  
Avoids scraping businesses already processed by checking Airtable for existing entries. Aggregates all scraped company names for duplicate detection.

**Nodes Involved:**  
- already scraped (Airtable search)  
- Array all Companies scraped (Summarize)  
- Places already scraped (If)  
- Filter5 (in Init block)  

**Node Details:**  
- **already scraped**  
  - *Type:* Airtable (Search)  
  - *Role:* Retrieves all previously scraped leads from Airtable table "Mapsscraper"  
  - *Failure Modes:* Airtable API limits, credential issues  

- **Array all Companies scraped**  
  - *Type:* Summarize  
  - *Role:* Concatenates all scraped company names into a single string for quick membership check  
  - *Failure Modes:* Large data causing performance issues  

- **Places already scraped**  
  - *Type:* If  
  - *Role:* Checks if the current place’s name exists in the concatenated string of scraped companies to skip duplicates  
  - *Failure Modes:* False negatives/positives due to name variations  

---

#### 1.4 Impressum URL Construction & HTML Scraping  
**Overview:**  
Constructs the company’s Impressum page URL from its website domain and scrapes the page HTML content for further analysis.

**Nodes Involved:**  
- Get Impressum-URL (Set)  
- Get Impressum HTML (HTTP Request)  
- HTML (HTML Extract)  
- Successfully scraped Impressumpages? (Filter)  
- Wait2 (Wait)  
- Loop Over Items3 (SplitInBatches)  

**Node Details:**  
- **Get Impressum-URL**  
  - *Type:* Set  
  - *Role:* Extracts domain from `result.website` and appends `/impressum` path to construct the target URL  
  - *Error Handling:* Continues on error (e.g., missing website URL)  
  - *Expressions:* Uses string methods for URL extraction  

- **Get Impressum HTML**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the HTML content of the constructed Impressum URL  
  - *Error Handling:* Continues on error to avoid workflow stopping  
  - *Failure Modes:* 404 pages, blocked requests, network errors  

- **HTML**  
  - *Type:* HTML Extract  
  - *Role:* Extracts main body text from the HTML excluding noise elements (ads, nav, footer, scripts, etc.) using CSS selectors  
  - *Failure Modes:* Malformed HTML, missing selectors  

- **Successfully scraped Impressumpages?**  
  - *Type:* Filter  
  - *Role:* Checks if the Impressum scraping did not produce errors, filters only successful fetches to next step  

- **Wait2**  
  - *Type:* Wait  
  - *Role:* Introduces delay between requests to avoid being blocked or rate limited  

- **Loop Over Items3**  
  - *Type:* SplitInBatches  
  - *Role:* Processes Impressum pages in batches for efficiency  

---

#### 1.5 AI-Powered Information Extraction  
**Overview:**  
Uses an OpenAI GPT model via LangChain node to extract structured contact information (decision maker name, position, phone, email) from the Impressum HTML text.

**Nodes Involved:**  
- Relevant Infos from Impressum (Information Extractor)  
- OpenAI Chat Model  
- Wait (Wait)  
- Email not Empty (Filter)  
- Edit Fields1 (Set)  

**Node Details:**  
- **OpenAI Chat Model**  
  - *Type:* LangChain LM Chat OpenAI  
  - *Role:* Provides GPT-4.1-mini model for conversational AI processing  
  - *Credentials:* OpenAI API key stored in creds  
  - *Config:* Minimal options, cached results enabled  
  - *Failure Modes:* API key invalid, rate limits, prompt errors  

- **Relevant Infos from Impressum**  
  - *Type:* LangChain Information Extractor  
  - *Role:* Uses system prompt to extract executive name/position, phone, and email from the provided HTML text  
  - *Attributes:* Outputs fields: Entscheidername (decision maker), Telefonnummer, Emailadresse  
  - *Prompt:* Detailed template instructing extraction and fallback to null if info is missing  
  - *Failure Modes:* Unexpected text formats, incomplete data  

- **Wait**  
  - *Type:* Wait  
  - *Role:* Adds delay after AI extraction to control request pacing  

- **Email not Empty**  
  - *Type:* Filter  
  - *Role:* Allows only results with a non-empty email address to proceed, ensuring data quality  

- **Edit Fields1**  
  - *Type:* Set  
  - *Role:* Splits Entscheidername into two separate fields: name and position (if available)  
  - *Expressions:* Uses split by comma and fallback to empty string  

---

#### 1.6 Data Writing to Airtable  
**Overview:**  
Final step writes the enriched lead data including contact info back into Airtable, updating or inserting records by company name.

**Nodes Involved:**  
- Write in DB (Airtable Upsert)  

**Node Details:**  
- **Write in DB**  
  - *Type:* Airtable (Upsert)  
  - *Role:* Inserts or updates lead record with detailed fields: company name, email, phone, website, address, search term, decision maker info  
  - *Matching Column:* Firmenname (Company Name) for upsert  
  - *Failure Modes:* Airtable write errors, schema mismatches  

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                                   | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                  |
|------------------------------|-------------------------------|-------------------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger      | Starts workflow with city and search term input | —                              | already scraped               |                                                                                                              |
| Airtable2                    | Airtable (Search)             | Fetches cities from Airtable                      | When Executed by Another Workflow | Filter5                      |                                                                                                              |
| Filter5                     | Filter                        | Filters cities to those not processed and in DE  | Airtable2                     | Loop Over Items2               |                                                                                                              |
| Loop Over Items2             | SplitInBatches                | Processes cities in batches                        | Filter5                      | Execute Workflow, Airtable1   |                                                                                                              |
| Execute Workflow             | Execute Workflow              | Calls scraping sub-workflow with city parameters | Loop Over Items2              | Airtable1                    |                                                                                                              |
| Airtable1                   | Airtable (Upsert)             | Marks cities as processed                          | Execute Workflow              | Loop Over Items2               |                                                                                                              |
| Searchword                  | Set                          | Creates Google Maps search query                   | Array all Companies scraped    | search Place                 |                                                                                                              |
| search Place                | HTTP Request                 | Queries Google Maps Places API                     | Searchword                   | Split Out                    |                                                                                                              |
| Split Out                   | SplitOut                     | Splits Places API results into single items       | search Place                 | Place Details                |                                                                                                              |
| Place Details               | HTTP Request                 | Fetches detailed place info                         | Split Out                   | Places already scraped        |                                                                                                              |
| already scraped             | Airtable (Search)             | Retrieves previously scraped companies             | When Executed by Another Workflow | Array all Companies scraped  |                                                                                                              |
| Array all Companies scraped | Summarize                    | Concatenates all scraped company names             | already scraped              | Searchword                   |                                                                                                              |
| Places already scraped      | If                           | Checks if place already scraped                     | Place Details               | Get Impressum-URL            |                                                                                                              |
| Get Impressum-URL           | Set                          | Constructs Impressum page URL                        | Places already scraped       | Loop Over Items3             |                                                                                                              |
| Loop Over Items3            | SplitInBatches                | Processes Impressum URLs in batches                 | Get Impressum-URL            | Successfully scraped Impressumpages?, Get Impressum HTML |                                                                                                              |
| Get Impressum HTML          | HTTP Request                 | Fetches HTML content of Impressum page              | Loop Over Items3             | HTML, Wait2                  |                                                                                                              |
| HTML                       | HTML Extract                 | Extracts main text content from HTML                 | Get Impressum HTML           | Wait2                        |                                                                                                              |
| Wait2                      | Wait                         | Adds delay to avoid rate limits                      | HTML, Get Impressum HTML     | Loop Over Items3             |                                                                                                              |
| Successfully scraped Impressumpages? | Filter                        | Filters out failed Impressum fetches                  | Loop Over Items3             | Loop Over Items1             |                                                                                                              |
| Loop Over Items1            | SplitInBatches                | Processes leads for AI extraction and filtering     | Successfully scraped Impressumpages? | Email not Empty, Relevant Infos from Impressum |                                                                                                              |
| Relevant Infos from Impressum | LangChain Information Extractor | Extracts contact info from Impressum HTML using AI  | Loop Over Items1             | Wait                        |                                                                                                              |
| OpenAI Chat Model           | LangChain LM Chat OpenAI      | Provides GPT-4.1-mini model for AI extraction       | Relevant Infos from Impressum | Relevant Infos from Impressum |                                                                                                              |
| Wait                       | Wait                         | Adds delay after AI extraction                        | Relevant Infos from Impressum | Loop Over Items1             |                                                                                                              |
| Email not Empty             | Filter                       | Ensures extracted email is not empty                  | Loop Over Items1             | Edit Fields1                 |                                                                                                              |
| Edit Fields1                | Set                          | Splits decision maker field into name and position  | Email not Empty              | Write in DB                  |                                                                                                              |
| Write in DB                 | Airtable (Upsert)             | Writes enriched lead data back to Airtable            | Edit Fields1                | —                            |                                                                                                              |
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual start point for testing                        | —                            | Airtable2                   |                                                                                                              |
| Sticky Note                 | Sticky Note                  | Explains initial city list scraping logic             | —                            | —                            | ## Init Scraper for Each City: Explains city processing and resuming if interrupted                          |
| Sticky Note1                | Sticky Note                  | Explains Impressum scraping step                       | —                            | —                            | Get Impressum-Pages and scrape trough them                                                                  |
| Sticky Note2                | Sticky Note                  | Explains AI extraction from Impressum                  | —                            | —                            | ## Extract Relevant Information from Impressum HTML: Decision Maker, Email, Telephone Number                  |
| Sticky Note3                | Sticky Note                  | Explains duplicate check and place detail fetching    | —                            | —                            | Check if the company has already been scraped → search for places using the search term → fetch details      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Type: Execute Workflow Trigger  
   - Parameters: Define inputs `Stadt,Land` (string), `Suchbegriff` (string)  

2. **Add Airtable Node (Airtable2)**  
   - Operation: Search  
   - Base: your Airtable base ID  
   - Table: "Städte"  
   - Limit: 25  
   - Credentials: Airtable API Token  
   - Purpose: Fetch cities to process  

3. **Add Filter Node (Filter5)**  
   - Condition:  
     - `Auswählen` not equals "Check"  
     - `Land` equals "Deutschland"  
   - Purpose: Select only unchecked German cities  

4. **Add SplitInBatches Node (Loop Over Items2)**  
   - Batch Size: default or set as needed  
   - Purpose: Process cities in manageable batches  

5. **Add Execute Workflow Node (Execute Workflow)**  
   - Workflow: Link to your scraping sub-workflow ("Maps Scraper")  
   - Inputs: Map `Stadt,Land` and `Suchbegriff` from batch items  
   - Purpose: Launch scraping for each city and search term  

6. **Add Airtable Node (Airtable1)**  
   - Operation: Upsert  
   - Base & Table: Same as Airtable2 or appropriate  
   - Matching Column: `id`  
   - Fields to Update: `Auswählen` set to "Check" to mark processed cities  
   - Purpose: Mark city as processed  

7. **In the Sub-workflow ("Maps Scraper"):**  

   a. **Set Node (Searchword)**  
      - Compose search query: `{{ Suchbegriff }} {{ Stadt,Land }}`  

   b. **HTTP Request Node (search Place)**  
      - URL: `https://maps.googleapis.com/maps/api/place/textsearch/json`  
      - Query Params: `query={{$json.Suchbegriff}}`, `pagetoken` from response  
      - Auth: Google API key via HTTP Query Auth  
      - Pagination: Enable with maxRequests=1, interval=20s  

   c. **SplitOut Node (Split Out)**  
      - Field: `results`  
      - Purpose: Iterate over each search result  

   d. **HTTP Request Node (Place Details)**  
      - URL: `https://maps.googleapis.com/maps/api/place/details/json`  
      - Query Param: `place_id={{$json.place_id}}`  
      - Auth: Same as search Place  

   e. **Airtable Node (already scraped)**  
      - Search all previously scraped companies from Airtable  
      - Credentials: Airtable API  

   f. **Summarize Node (Array all Companies scraped)**  
      - Field: `Firmenname` concatenated  
      - Purpose: Compile names for duplicate detection  

   g. **If Node (Places already scraped)**  
      - Condition: Check if current place name is in concatenated scraped names  
      - Branch to skip duplicates  

   h. **Set Node (Get Impressum-URL)**  
      - Expression: Extract domain from `result.website` and append `/impressum`  
      - Continue on error  

   i. **SplitInBatches Node (Loop Over Items3)**  
      - Batch process Impressum URLs  

   j. **HTTP Request Node (Get Impressum HTML)**  
      - URL: `https://{{ $json.result.website }}`  
      - Continue on error  

   k. **HTML Extract Node (HTML)**  
      - Extract `body` content excluding selectors: `img, wp, nav, header, footer, aside, script, style, .sidebar, .ads, .ad, .advertisement, .breadcrumb, .cookies, .newsletter, .popup, .modal`  

   l. **Filter Node (Successfully scraped Impressumpages?)**  
      - Condition: No error exists on HTTP response  

   m. **SplitInBatches Node (Loop Over Items1)**  
      - Process Impressum pages for AI extraction  

   n. **LangChain OpenAI Chat Model Node**  
      - Model: `gpt-4.1-mini` or equivalent  
      - Credentials: OpenAI API key  

   o. **LangChain Information Extractor (Relevant Infos from Impressum)**  
      - Input Text: Extracted mainContent  
      - Prompt: Extract decision maker name and position, phone, email; null if missing  

   p. **Wait Node**  
      - Delay 20 seconds to respect rate limits  

   q. **Filter Node (Email not Empty)**  
      - Only allow records with a non-empty email address  

   r. **Set Node (Edit Fields1)**  
      - Split `Entscheidername` by comma into `Entscheidername` and `Entscheiderposition` fields  

   s. **Airtable Node (Write in DB)**  
      - Operation: Upsert  
      - Matching Column: Firmenname  
      - Fields: Email, Website, Standort, Firmenname, Suchbegriff, Telefonnummer, Entscheidername, Entscheiderposition  
      - Credentials: Airtable API key  

8. **Add Wait Nodes** where indicated to manage API rate limits and pacing.

9. **Add Manual Trigger Node (optional)** for manual start/testing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| ## Init Scraper for Each City: Explains city-wise processing and resuming if interrupted                                                               | Sticky Note node near Initialization block      |
| Get Impressum-Pages and scrape through them                                                                                                           | Sticky Note near Impressum scraping block       |
| ## Extract Relevant Information from Impressum HTML: Decision Maker, Email, Telephone Number                                                           | Sticky Note near AI extraction block             |
| Check if the company has already been scraped → search for places using the search term → fetch detailed information via morePlaceDetails.             | Sticky Note near deduplication logic              |
| Workflow uses Google Maps Places API and Airtable. Credentials must be configured with valid API keys including Google API key and Airtable token.    | Credential setup reminder                         |
| OpenAI GPT model usage requires valid API key and may incur costs; rate limits and delays are implemented to manage usage.                            | AI usage consideration                           |
| The workflow uses pagination and batch processing nodes to handle large result sets efficiently and avoid API limits.                                  | Performance note                                 |

---

This documentation fully describes the workflow "Google Maps to Airtable Lead Scraper with GPT Contact Extraction from Impressum," enabling reproduction, customization, and troubleshooting.