Scrape Property Listings from Zillow with Olostep API and Store in Data Tables

https://n8nworkflows.xyz/workflows/scrape-property-listings-from-zillow-with-olostep-api-and-store-in-data-tables-11130


# Scrape Property Listings from Zillow with Olostep API and Store in Data Tables

### 1. Workflow Overview

This n8n workflow automates the scraping of property listings from Zillow using the Olostep API and stores the extracted data in an n8n Data Table. It is designed to collect property price, listing URL, and location information from Zillow search results for a user-specified location. The workflow handles pagination to scrape multiple pages, parses the JSON response from Olostep’s API, splits the data into individual listings, removes duplicates, and saves each listing row-by-row for further use such as filtering or exporting.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Pagination Setup:** Captures user input for the location, generates a list of page numbers to paginate through Zillow search results.
- **1.2 Zillow Scraping with Olostep API:** Iterates over each page number to send scraping requests to Olostep with parameters including location and page.
- **1.3 Data Parsing and Splitting:** Cleans and parses the JSON response from Olostep, splitting the results into individual property listings.
- **1.4 Data Storage:** Inserts each individual property listing into the configured n8n Data Table.
- **1.5 Loop Control:** Manages batch processing and looping over pages and items to ensure orderly execution.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Pagination Setup

- **Overview:**  
  This block receives the location input from the user via a form, then generates an array of page numbers (1 through 7) to paginate Zillow search results.

- **Nodes Involved:**  
  - `On form submission`  
  - `Edit Fields`  
  - `Split Out1`  
  - `Loop Over Items`

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point for the workflow; collects the user input, specifically the 'place' (location) to scrape from Zillow.  
    - *Configuration:* A form titled "Scrape Zillow With Olostep API" with one required field "place" (e.g., "Manhattan New York NY").  
    - *Input/Output:* Outputs JSON with a field `place` used downstream.  
    - *Potential Failures:* Webhook trigger failure if endpoint not reachable.  
    - *Sticky Note:* "Form Trigger - Enter a location (e.g. 'Manhattan New York NY') to build the Zillow URL."

  - **Edit Fields**  
    - *Type:* Set Node  
    - *Role:* Defines the pagination counter array `[1,2,3,4,5,6,7]` representing Zillow pages to scrape.  
    - *Configuration:* Sets a variable `counter` with the array of page numbers.  
    - *Input:* Receives output from form submission.  
    - *Output:* Passes data including `counter` array.  
    - *Sticky Note:* "Pagination - Generates page numbers (1–7) to scrape multiple Zillow pages."

  - **Split Out1**  
    - *Type:* Split Out  
    - *Role:* Splits the `counter` array to process each page number individually in subsequent nodes.  
    - *Configuration:* Splits field `counter`.  
    - *Input:* Receives array from `Edit Fields`.  
    - *Output:* Sends one page number per item downstream.  
    - *Potential Failures:* Empty or malformed counter array would cause no iterations.

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Controls batch processing of pages for scraping.  
    - *Configuration:* Default batch options (no limit specified).  
    - *Input:* Receives individual page numbers from `Split Out1`.  
    - *Output:* Loops over page numbers, triggering scraping requests.  
    - *Version Specific:* Requires n8n version supporting SplitInBatches v3.  
    - *Potential Failures:* Loop interruptions or empty batches if input missing.

---

#### 2.2 Zillow Scraping with Olostep API

- **Overview:**  
  For each page number and location, this block sends a POST request to the Olostep API to scrape Zillow listings. The API is instructed to extract price, URL, and location from each listing using a custom LLM extraction schema.

- **Nodes Involved:**  
  - `scrape places`  
  - `parsedInfo`  
  - `Split Out`

- **Node Details:**

  - **scrape places**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to Olostep’s scraping API with JSON body containing the Zillow URL for the specified place and page number.  
    - *Configuration:*  
      - URL: `https://api.olostep.com/v1/scrapes`  
      - Method: POST  
      - Headers: Authorization with Bearer token (placeholder `<token>` to be replaced with valid API key).  
      - Body: JSON including  
        - `url_to_scrape` dynamically built with location from form and current page number.  
        - `formats`: markdown and json.  
        - `llm_extract` schema specifying extraction of `price`, `url`, and `location`.  
        - Screen size set to desktop 1920x1080.  
      - Retry on fail enabled; onError set to continue regular output to avoid workflow halt on certain failures.  
    - *Input:* Page number and place from previous blocks.  
    - *Output:* Raw API response including JSON content with scraped data.  
    - *Potential Failures:*  
      - Authorization errors if token invalid or missing.  
      - 504 Gateway Timeout due to LLM extraction complexity (see sticky note warning).  
      - Network or API downtime.  
    - *Sticky Note:* "Olostep Zillow Scrape - Extracts price, listing URL, and location from each page."  
    - *Warning Note:* If 504 errors occur, removing or simplifying llm_extract schema is recommended.

  - **parsedInfo**  
    - *Type:* Set Node  
    - *Role:* Cleans the raw JSON by removing backslash escapes and prepares the JSON array of listings for splitting.  
    - *Configuration:* Uses JavaScript expression to replace escaped backslashes in the `result.json_content` field from Olostep response and assigns it to `parsedJson` as an array.  
    - *Input:* Receives raw response from `scrape places`.  
    - *Output:* JSON array `parsedJson` of individual listings.  
    - *Potential Failures:* Expression failure if `result.json_content` missing or malformed.  
    - *Sticky Note:* "Parse & Split - Cleans the JSON and splits Zillow results into individual listings."

  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Splits the array `parsedJson` into individual listing items for further processing.  
    - *Configuration:* Splits on field `parsedJson`.  
    - *Input:* Receives cleaned array from `parsedInfo`.  
    - *Output:* One JSON item per property listing.  
    - *Potential Failures:* Empty or invalid array results in no output.

---

#### 2.3 Data Storage

- **Overview:**  
  Inserts each individual property listing (price, URL, location) into an n8n Data Table row-by-row, enabling structured storage and downstream analysis.

- **Nodes Involved:**  
  - `Insert row`

- **Node Details:**

  - **Insert row**  
    - *Type:* Data Table  
    - *Role:* Saves each property listing’s data into a specified Data Table.  
    - *Configuration:*  
      - Data Table ID set to the target table storing Zillow places.  
      - Columns mapped explicitly:  
        - `url` from `$json.url`  
        - `price` from `$json.price`  
        - `location` from `$json.location`  
      - Mapping mode: defineBelow.  
    - *Input:* Receives individual listings from `Split Out`.  
    - *Output:* Passes control back to loop to process next item.  
    - *Potential Failures:* Table ID invalid or permission errors; data type mismatches.  
    - *Sticky Note:* "Insert Row - Saves each property into Google Sheets / Data Table."

---

#### 2.4 Loop Control and Workflow Flow

- **Overview:**  
  Coordinates the iterative process over pages and listings, managing the looping and batch processing to ensure complete scraping and insertion.

- **Nodes Involved:**  
  - `Loop Over Items` (used twice in connections)  

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:*  
      - First use: loops over page numbers generated (pagination).  
      - Second use: loops over individual listing insertions (though here mainly the first loop is active, second loop branch empty).  
    - *Input/Output:* Manages iteration through pages and listings.  
    - *Potential Failures:* Infinite loops if not configured correctly; empty inputs halting workflow.

---

### 3. Summary Table

| Node Name         | Node Type          | Functional Role                           | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                      |
|-------------------|--------------------|-----------------------------------------|-----------------------|----------------------|------------------------------------------------------------------------------------------------|
| On form submission | Form Trigger       | Receives user input (location)          | —                     | Edit Fields          | "Form Trigger - Enter a location (e.g. “Manhattan New York NY”) to build the Zillow URL."       |
| Edit Fields       | Set                | Defines pagination array [1..7]         | On form submission    | Split Out1           | "Pagination - Generates page numbers (1–7) to scrape multiple Zillow pages."                    |
| Split Out1        | Split Out          | Splits pagination array for looping     | Edit Fields           | Loop Over Items      |                                                                                                |
| Loop Over Items    | SplitInBatches     | Iterates over pages for scraping        | Split Out1            | scrape places        |                                                                                                |
| scrape places     | HTTP Request       | Sends scraping request to Olostep API   | Loop Over Items       | parsedInfo           | "Olostep Zillow Scrape - Extracts price, listing URL, and location from each page."              |
| parsedInfo        | Set                | Cleans JSON response, prepares array    | scrape places         | Split Out            | "Parse & Split - Cleans the JSON and splits Zillow results into individual listings."            |
| Split Out         | Split Out          | Splits listings into individual items   | parsedInfo            | Insert row           |                                                                                                |
| Insert row        | Data Table         | Inserts listings into Data Table         | Split Out             | Loop Over Items      | "Insert Row - Saves each property into Google Sheets / Data Table."                             |
| Sticky Note1      | Sticky Note        | Pagination explanation                   | —                     | —                    | "Pagination  \nGenerates page numbers (1–7) to scrape multiple Zillow pages."                    |
| Sticky Note2      | Sticky Note        | Olostep scraping explanation             | —                     | —                    | "Olostep Zillow Scrape  \nExtracts price, listing URL, and location from each page."             |
| Sticky Note3      | Sticky Note        | Data insertion explanation                | —                     | —                    | "Insert Row  \nSaves each property into Google Sheets / Data Table."                            |
| Sticky Note4      | Sticky Note        | Warning about 504 errors in scraping     | —                     | —                    | "WARNING\nIf the http request runs through a 504 gateway timeout error, that's because the \"llm_extract\".\n\nYou can remove all the fields in the schema (inside the llm_extract) and then use a information extractor node or llm node to extract the desired information." |
| Sticky Note5      | Sticky Note        | Parsing explanation                       | —                     | —                    | "Parse & Split  \nCleans the JSON and splits Zillow results into individual listings."          |
| Sticky Note6      | Sticky Note        | Workflow overview and instructions       | —                     | —                    | See full detailed content in section 5.                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow named "zillow".**

2. **Add a Form Trigger node:**  
   - Name: `On form submission`  
   - Configure webhook with any unique ID.  
   - Set form title: "Scrape Zillow With Olostep API"  
   - Add one required field:  
     - Field Label: `place`  
     - Placeholder: "Manhatten New York NY"  
   - No attribution appended.

3. **Add a Set node:**  
   - Name: `Edit Fields`  
   - Purpose: Create pagination array.  
   - Assign a new field `counter` of type array with value `[1,2,3,4,5,6,7]` (representing Zillow pages).

4. **Connect `On form submission` → `Edit Fields`.**

5. **Add a Split Out node:**  
   - Name: `Split Out1`  
   - Field to split out: `counter`  
   - Connect `Edit Fields` → `Split Out1`.

6. **Add a SplitInBatches node:**  
   - Name: `Loop Over Items`  
   - Use default options (no batch size set).  
   - Connect `Split Out1` → `Loop Over Items`.

7. **Add an HTTP Request node:**  
   - Name: `scrape places`  
   - Method: POST  
   - URL: `https://api.olostep.com/v1/scrapes`  
   - Headers: Add header `Authorization` with value `Bearer <token>` (replace `<token>` with your Olostep API key).  
   - Body Content: JSON (select 'JSON' body mode)  
   - JSON Body: Use an expression to build the body dynamically:  
     ```json
     {
       "url_to_scrape": "https://www.zillow.com/{{ $('On form submission').item.json.place }}/?searchQueryState={\"pagination\":{\"currentPage\":{{ $json.counter }}}}",
       "formats": ["markdown", "json"],
       "wait_before_scraping": 2,
       "remove_css_selectors": "default",
       "llm_extract": {
         "schema": {
           "type": "array",
           "description": "A list of businesses found on the Google Maps search results page.",
           "items": {
             "type": "object",
             "properties": {
               "price": { "type": "string", "description": "The sale/rent price of the place." },
               "url": { "type": "string", "description": "The url for the place." },
               "location": { "type": "string", "description": "A short snippet of the place address or location." }
             },
             "required": ["location", "url"]
           }
         }
       },
       "screen_size": {
         "screen_type": "desktop",
         "screen_width": 1920,
         "screen_height": 1080
       }
     }
     ```  
   - Enable `Send Body` and `Send Headers`.  
   - Set `Retry On Fail` to true.  
   - Set `On Error` to "Continue Regular Output" to avoid failing the entire workflow on errors.

8. **Connect `Loop Over Items` → `scrape places`.**

9. **Add a Set node:**  
   - Name: `parsedInfo`  
   - Purpose: Clean the JSON string response.  
   - Add a new field `parsedJson` of type array with expression:  
     `={{ $json.result.json_content.replace(/\\\\/g, '') }}`  
   - Connect `scrape places` → `parsedInfo`.

10. **Add a Split Out node:**  
    - Name: `Split Out`  
    - Field to split out: `parsedJson`  
    - Connect `parsedInfo` → `Split Out`.

11. **Add a Data Table node:**  
    - Name: `Insert row`  
    - Select or create a Data Table to store Zillow listings.  
    - Map columns explicitly:  
      - `url` → `{{$json.url}}`  
      - `price` → `{{$json.price}}`  
      - `location` → `{{$json.location}}`  
    - Connect `Split Out` → `Insert row`.

12. **Connect `Insert row` → `Loop Over Items` (to continue looping).**

13. **Add Sticky Notes to document the workflow steps as per the sticky content from the original workflow (optional but recommended).**

14. **Configure credentials:**  
    - Add Olostep API key as credential for HTTP Request node.

15. **Activate the workflow.**  
    - Deploy and start by submitting the form with desired location.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow enables fast, scalable property scraping using Zillow combined with Olostep API without requiring browser automation or manual copying. It extracts property price, URL, and location, removes duplicates, and stores results in a Data Table or Google Sheet for analysis. Setup requires inserting your Olostep API key and configuring your Data Table.                                                                                                                                                                                                                                                                                | Workflow overview and setup instructions (Sticky Note6 content)                                                |
| **Important:** If the HTTP request to Olostep API returns a 504 Gateway Timeout error, it is likely due to the complexity of the `llm_extract` schema. To mitigate, remove or simplify the schema fields inside `llm_extract` and consider using an information extractor or LLM node afterward to parse the data.                                                                                                                                                                                                                                                                                                                                           | Warning about 504 Gateway Timeout from Sticky Note4                                                             |
| Olostep API documentation and API key management should be consulted to generate a valid Bearer token.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Olostep API (https://olostep.com or official docs)                                                              |
| Pagination is hardcoded to pages 1 through 7 for moderate scraping depth; this number can be adjusted in the `Edit Fields` node if more pages are required.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Pagination logic explanation (Sticky Note1)                                                                     |
| The Data Table used for storage is identified by ID `2NjhLmZVrLPM3jf2`; replace with your own Data Table ID if different.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Data Table node configuration                                                                                   |
| The form trigger webhook URL must be exposed publicly or through n8n’s webhook forwarding for external access.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Form Trigger deployment note                                                                                     |

---

This comprehensive documentation should allow advanced users and AI agents to understand, reproduce, and modify the Zillow scraping workflow reliably while anticipating potential issues and integration points.