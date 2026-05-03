Scrape Trustpilot Reviews to Google Sheets + HelpfulCrowd compatible csv

https://n8nworkflows.xyz/workflows/scrape-trustpilot-reviews-to-google-sheets---helpfulcrowd-compatible-csv-3168


# Scrape Trustpilot Reviews to Google Sheets + HelpfulCrowd compatible csv

### 1. Workflow Overview

This workflow automates the scraping of Trustpilot reviews for a specified company profile and saves the extracted data into Google Sheets. It supports both on-demand and scheduled execution, enabling continuous or manual data collection. The workflow is designed to output two Google Sheets tabs: one with raw Trustpilot review data and another formatted to be compatible with HelpfulCrowd’s product review import requirements.

**Logical Blocks:**

- **1.1 Trigger Block:** Handles manual or scheduled initiation of the workflow.
- **1.2 Configuration Block:** Sets global parameters such as the Trustpilot company ID and pagination limits.
- **1.3 Data Retrieval Block:** Fetches Trustpilot review pages via HTTP requests with pagination.
- **1.4 Parsing Block:** Extracts review data from the HTML content using code.
- **1.5 Data Splitting Block:** Splits the aggregated review array into individual review items.
- **1.6 Data Transformation Block:** Prepares two separate data sets:
  - General edits for raw Trustpilot data storage.
  - HelpfulCrowd edits for formatting data to HelpfulCrowd CSV specifications.
- **1.7 Google Sheets Storage Block:** Appends or updates data in two Google Sheets tabs:
  - Raw Trustpilot reviews.
  - HelpfulCrowd-compatible reviews.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:** Initiates the workflow either manually or on a schedule.
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)  
  - `Schedule Trigger` (Scheduled Trigger)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing or on-demand runs.  
    - Configuration: No parameters; triggers workflow immediately when clicked.  
    - Inputs: None  
    - Outputs: Connected to `Global` node.  
    - Edge Cases: None specific; user must manually trigger.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Enables automatic periodic execution.  
    - Configuration: Default interval (every minute, hour, day, etc. can be customized).  
    - Inputs: None  
    - Outputs: Connected to `Global` node.  
    - Edge Cases: Scheduling misconfiguration could cause unintended frequency.

#### 1.2 Configuration Block

- **Overview:** Defines global variables for the Trustpilot company ID and maximum pages to scrape.
- **Nodes Involved:**  
  - `Global` (Set node)  
  - `Sticky Note` (Instructional note)

- **Node Details:**

  - **Global**  
    - Type: Set  
    - Role: Stores configuration parameters used downstream.  
    - Configuration:  
      - `company_id`: String, e.g., `"n8n.io"` (Trustpilot business name)  
      - `max_page`: Number, e.g., `100` (maximum pagination pages to fetch)  
    - Inputs: From triggers (`When clicking ‘Test workflow’` or `Schedule Trigger`)  
    - Outputs: Connected to `Get reviews` node.  
    - Edge Cases: Incorrect company ID or excessive max_page may cause errors or long runtimes.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Instruction to edit the `Global` node with company name and max pages.  
    - Content:  
      ```
      ## Edit this node 👇
      Change to the name of the company registered on Trustpilot and the maximum number of pages to scrape
      ```
    - Inputs/Outputs: None

#### 1.3 Data Retrieval Block

- **Overview:** Fetches Trustpilot review pages using HTTP requests with pagination support.
- **Nodes Involved:**  
  - `Get reviews`

- **Node Details:**

  - **Get reviews**  
    - Type: HTTP Request  
    - Role: Retrieves HTML content of Trustpilot review pages.  
    - Configuration:  
      - URL template: `https://trustpilot.com/review/{{ $json.company_id }}`  
      - Pagination: Uses query parameter `page` incremented per request, up to `max_page`.  
      - Pagination stops on HTTP 404 status code (no more pages).  
      - Query parameter: `sort=recency` to get most recent reviews first.  
      - Request interval: 5000 ms between requests to avoid rate limits.  
    - Inputs: From `Global` node (provides `company_id` and `max_page`)  
    - Outputs: Connected to `Parse reviews` node.  
    - Edge Cases:  
      - HTTP errors (e.g., 404 to stop pagination, 429 rate limiting)  
      - Network timeouts  
      - Incorrect company ID leading to empty or error pages

#### 1.4 Parsing Block

- **Overview:** Parses the HTML content from Trustpilot pages to extract review data using embedded JSON.
- **Nodes Involved:**  
  - `Parse reviews`

- **Node Details:**

  - **Parse reviews**  
    - Type: Code (JavaScript)  
    - Role: Extracts review objects from the page HTML using Cheerio to parse DOM and JSON embedded in `#__NEXT_DATA__` script tag.  
    - Configuration:  
      - Loads each page’s HTML content.  
      - Parses JSON from `#__NEXT_DATA__` script tag.  
      - Extracts `reviews` array from JSON path `props.pageProps.reviews`.  
      - Iterates over all pages’ data, accumulates reviews.  
      - Returns an array of review objects.  
    - Inputs: From `Get reviews` node (HTML content per page)  
    - Outputs: Returns a single JSON object with a `reviews` array. Connected to `Split Out`.  
    - Edge Cases:  
      - Missing or malformed `#__NEXT_DATA__` tag  
      - JSON parse errors  
      - Empty or missing reviews array  
      - Unexpected page structure changes by Trustpilot

#### 1.5 Data Splitting Block

- **Overview:** Splits the aggregated reviews array into individual review items for further processing.
- **Nodes Involved:**  
  - `Split Out`

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Takes the `reviews` array and outputs each review as a separate item.  
    - Configuration: Field to split out: `reviews`  
    - Inputs: From `Parse reviews` node (single item with array)  
    - Outputs: Two parallel outputs connected to `HelpfulCrowd edits` and `General edits` nodes.  
    - Edge Cases: Empty array results in no output items.

#### 1.6 Data Transformation Block

- **Overview:** Prepares two different data formats for storage: raw Trustpilot data and HelpfulCrowd-compatible data.
- **Nodes Involved:**  
  - `General edits`  
  - `HelpfulCrowd edits`

- **Node Details:**

  - **General edits**  
    - Type: Set  
    - Role: Maps raw review fields to a simplified structure for the general Trustpilot sheet.  
    - Configuration:  
      - Fields set:  
        - `Date`: review published date (ISO string)  
        - `Author`: consumer display name  
        - `Body`: review text  
        - `Heading`: review title  
        - `Rating`: review rating  
        - `Location`: consumer country code  
        - `review_id`: unique review ID  
    - Inputs: From `Split Out` node  
    - Outputs: Connected to `General sheet` node.  
    - Edge Cases: Missing or malformed fields in review data.

  - **HelpfulCrowd edits**  
    - Type: Set  
    - Role: Formats review data to match HelpfulCrowd’s import CSV schema, including additional fields for upsert support.  
    - Configuration:  
      - Fields set (with expressions referencing review JSON):  
        - `product_id`: empty string (to be filled manually if needed)  
        - `rating`, `title`, `feedback`, `customer_name`, `customer_email`, `comment`, `status`, `review_date`, `verified`, `media_1` to `media_5`, `review_id`  
      - `status` is set conditionally: `'pending'` if review is pending, else `'published'`  
      - `verified` is set to `'yes'` or `'no'` based on verification label  
    - Inputs: From `Split Out` node  
    - Outputs: Connected to `HelpfulCrowd Sheets` node.  
    - Edge Cases: Missing fields, empty product_id may require manual update.

#### 1.7 Google Sheets Storage Block

- **Overview:** Stores the processed review data into two separate Google Sheets tabs, appending or updating rows based on unique review IDs.
- **Nodes Involved:**  
  - `General sheet`  
  - `HelpfulCrowd Sheets`  
  - `Sticky Note1` (Spreadsheet clone instruction)  
  - `Sticky Note2` (HelpfulCrowd docs link)

- **Node Details:**

  - **General sheet**  
    - Type: Google Sheets  
    - Role: Appends or updates raw Trustpilot reviews in the `trustpilot` sheet tab.  
    - Configuration:  
      - Document ID: points to a specific Google Sheets document (template to be cloned)  
      - Sheet Name: `trustpilot` (sheet ID 323953858)  
      - Columns mapped: Date, Author, Body, Heading, Rating, Location, review_id  
      - Matching column: `review_id` for upsert operation  
      - Operation: `appendOrUpdate`  
    - Inputs: From `General edits` node  
    - Outputs: None  
    - Credentials: Google Sheets OAuth2 API  
    - Edge Cases:  
      - API authentication errors  
      - Sheet access permissions  
      - Duplicate review_id handling

  - **HelpfulCrowd Sheets**  
    - Type: Google Sheets  
    - Role: Appends or updates reviews formatted for HelpfulCrowd import in the `helpfulcrowd` sheet tab.  
    - Configuration:  
      - Document ID: same as above  
      - Sheet Name: `helpfulcrowd` (sheet ID 1811842087)  
      - Columns mapped: product_id, rating, title, feedback, customer_name, customer_email, comment, status, review_date, verified, media_1 to media_5, review_id  
      - Matching column: `review_id` for upsert operation  
      - Operation: `appendOrUpdate`  
    - Inputs: From `HelpfulCrowd edits` node  
    - Outputs: None  
    - Credentials: Google Sheets OAuth2 API  
    - Edge Cases:  
      - API authentication errors  
      - Sheet access permissions  
      - Missing required fields for HelpfulCrowd import

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Instruction to clone the Google Sheets template before use.  
    - Content:  
      ```
      ## Clone this spreadsheet
      https://docs.google.com/spreadsheets/d/19nndnEO186vNmApxce8bA1AnLYrY8bR8VgYlwOU_FYA/edit?gid=0#gid=0
      ```
    - Inputs/Outputs: None

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Provides HelpfulCrowd import documentation link for reference.  
    - Content:  
      ```
      ### HelpfulCrowd column
      Check this docs
      https://www.guides.helpfulcrowd.com/en/article/import-product-reviews-wof0oy/
      ```
    - Inputs/Outputs: None

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                          | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                      |
|----------------------------|--------------------|----------------------------------------|----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Manual workflow initiation              | None                             | Global                            |                                                                                                |
| Schedule Trigger           | Schedule Trigger   | Scheduled workflow initiation           | None                             | Global                            |                                                                                                |
| Global                    | Set                | Sets company ID and max pages           | When clicking ‘Test workflow’, Schedule Trigger | Get reviews                      | ## Edit this node 👇 Change to the name of the company registered on Trustpilot and the maximum number of pages to scrape |
| Get reviews               | HTTP Request       | Fetches Trustpilot review pages         | Global                           | Parse reviews                    |                                                                                                |
| Parse reviews             | Code               | Parses review data from HTML content    | Get reviews                     | Split Out                       |                                                                                                |
| Split Out                 | Split Out          | Splits reviews array into individual items | Parse reviews                   | HelpfulCrowd edits, General edits |                                                                                                |
| HelpfulCrowd edits        | Set                | Formats data for HelpfulCrowd CSV       | Split Out                      | HelpfulCrowd Sheets             |                                                                                                |
| General edits             | Set                | Formats data for raw Trustpilot sheet   | Split Out                      | General sheet                   |                                                                                                |
| General sheet             | Google Sheets      | Stores raw reviews in Google Sheets     | General edits                  | None                           | ## Clone this spreadsheet https://docs.google.com/spreadsheets/d/19nndnEO186vNmApxce8bA1AnLYrY8bR8VgYlwOU_FYA/edit?gid=0#gid=0 |
| HelpfulCrowd Sheets       | Google Sheets      | Stores HelpfulCrowd formatted reviews   | HelpfulCrowd edits             | None                           | ### HelpfulCrowd column Check this docs https://www.guides.helpfulcrowd.com/en/article/import-product-reviews-wof0oy/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’` with default settings.
   - Add a **Schedule Trigger** node named `Schedule Trigger` with desired interval (e.g., daily).

2. **Create Configuration Node:**
   - Add a **Set** node named `Global`.
   - Add two fields:  
     - `company_id` (String): Set to the Trustpilot business name, e.g., `"n8n.io"`.  
     - `max_page` (Number): Set maximum pages to scrape, e.g., `100`.
   - Connect outputs of both trigger nodes to this node.

3. **Create HTTP Request Node:**
   - Add an **HTTP Request** node named `Get reviews`.
   - Set HTTP Method to `GET`.
   - Set URL to:  
     ```
     https://trustpilot.com/review/{{ $json.company_id }}
     ```
   - Enable Pagination:  
     - Pagination type: Query Parameter  
     - Parameter name: `page`  
     - Value: `={{ $pageCount + 1 }}`  
     - Max requests: `={{ $json.max_page }}`  
     - Request interval: 5000 ms  
     - Stop pagination on HTTP status code 404  
   - Add query parameter: `sort=recency`.
   - Connect `Global` node output to this node.

4. **Create Code Node for Parsing:**
   - Add a **Code** node named `Parse reviews`.
   - Use JavaScript with Cheerio library to parse HTML and extract reviews from `#__NEXT_DATA__` script tag.
   - The code should:  
     - Iterate over all input items (pages).  
     - Parse JSON from the script tag.  
     - Extract `reviews` array.  
     - Accumulate all reviews into one array.  
     - Return an object with a `reviews` array.
   - Connect `Get reviews` output to this node.

5. **Create Split Out Node:**
   - Add a **Split Out** node named `Split Out`.
   - Set field to split: `reviews`.
   - Connect `Parse reviews` output to this node.

6. **Create Data Transformation Nodes:**
   - Add a **Set** node named `General edits`.
     - Map fields:  
       - `Date`: `={{ $json.dates.publishedDate }}`  
       - `Author`: `={{ $json.consumer.displayName }}`  
       - `Body`: `={{ $json.text }}`  
       - `Heading`: `={{ $json.title }}`  
       - `Rating`: `={{ $json.rating }}`  
       - `Location`: `={{ $json.consumer.countryCode }}`  
       - `review_id`: `={{ $json.id }}`
     - Connect one output of `Split Out` to this node.

   - Add a **Set** node named `HelpfulCrowd edits`.
     - Map fields according to HelpfulCrowd schema:  
       - `product_id`: `""` (empty string)  
       - `rating`: `={{ $json.rating }}`  
       - `title`: `={{ $json.title }}`  
       - `feedback`: `={{ $json.text }}`  
       - `customer_name`: `={{ $json.consumer.displayName }}`  
       - `customer_email`: `""` (empty string)  
       - `comment`: `""` (empty string)  
       - `status`: `={{ $json.pending ? 'pending' : 'published' }}`  
       - `review_date`: `={{ $json.dates.publishedDate.split('T')[0] }}`  
       - `verified`: `={{ $json.labels.verification.isVerified ? 'yes' : 'no' }}`  
       - `media_1` to `media_5`: `""` (empty strings)  
       - `review_id`: `={{ $json.id }}`
     - Connect other output of `Split Out` to this node.

7. **Create Google Sheets Nodes:**
   - Add a **Google Sheets** node named `General sheet`.
     - Set operation to `appendOrUpdate`.
     - Document ID: Use your cloned Google Sheets document ID.
     - Sheet name: `trustpilot` (or sheet ID 323953858).
     - Mapping columns: Date, Author, Body, Heading, Rating, Location, review_id.
     - Matching column for upsert: `review_id`.
     - Connect `General edits` output to this node.
     - Configure Google Sheets OAuth2 credentials.

   - Add a **Google Sheets** node named `HelpfulCrowd Sheets`.
     - Set operation to `appendOrUpdate`.
     - Document ID: Same as above.
     - Sheet name: `helpfulcrowd` (or sheet ID 1811842087).
     - Mapping columns: product_id, rating, title, feedback, customer_name, customer_email, comment, status, review_date, verified, media_1 to media_5, review_id.
     - Matching column for upsert: `review_id`.
     - Connect `HelpfulCrowd edits` output to this node.
     - Configure Google Sheets OAuth2 credentials.

8. **Add Sticky Notes for Instructions (Optional):**
   - Add a sticky note near `Global` node with instructions to edit company ID and max pages.
   - Add a sticky note near Google Sheets nodes with the link to clone the Google Sheets template:  
     ```
     https://docs.google.com/spreadsheets/d/19nndnEO186vNmApxce8bA1AnLYrY8bR8VgYlwOU_FYA/edit?gid=0#gid=0
     ```
   - Add a sticky note near HelpfulCrowd Sheets node with HelpfulCrowd import guide link:  
     ```
     https://www.guides.helpfulcrowd.com/en/article/import-product-reviews-wof0oy/
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Clone this Google Sheets template before running the workflow:                                  | https://docs.google.com/spreadsheets/d/19nndnEO186vNmApxce8bA1AnLYrY8bR8VgYlwOU_FYA/edit?gid=0#gid=0      |
| HelpfulCrowd import guide for product reviews CSV format:                                       | https://www.guides.helpfulcrowd.com/en/article/import-product-reviews-wof0oy/                           |
| Workflow supports both manual and scheduled execution for flexible scraping intervals.          |                                                                                                         |
| Requires Google Sheets API credentials with OAuth2 configured for access to the target spreadsheet.|                                                                                                         |
| Trustpilot scraping depends on page structure and embedded JSON data; changes on Trustpilot may require workflow updates.|                                                                                                         |
| Pagination stops automatically on HTTP 404 status code, indicating no more review pages.        |                                                                                                         |
| Export the `helpfulcrowd` sheet as CSV for upload to HelpfulCrowd platform.                     |                                                                                                         |
| For more n8n templates by the author, visit:                                                   | https://n8n.io/creators/bangank36/                                                                      |

---

This document fully describes the workflow structure, node configurations, and instructions to reproduce or modify the workflow, including potential failure points and integration considerations.