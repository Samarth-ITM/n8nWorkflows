Google Maps Scraper

https://n8nworkflows.xyz/workflows/google-maps-scraper-2063


# Google Maps Scraper

### 1. Workflow Overview

This workflow is designed to efficiently scrape Google Maps data using the SerpAPI service, providing a cost-effective alternative to the official Google Maps API. By inputting a Google Maps search URL, users can obtain detailed information about places including phone numbers, websites, ratings, reviews, addresses, and more.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Scheduling:** Handles manual or scheduled triggering and retrieves search URLs to process from a Google Sheet.
- **1.2 URL Parsing:** Extracts search keywords and geographic coordinates from the Google Maps URLs.
- **1.3 SerpAPI Querying and Pagination:** Sends requests to SerpAPI to retrieve data, handles pagination by extracting the next page token, and loops through all pages.
- **1.4 Data Aggregation and Cleaning:** Merges data from multiple pages, filters out empty and duplicate entries, and formats data appropriately.
- **1.5 Output and Status Management:** Inserts cleaned data into a Google Sheet and updates the status of each processed search URL in the input sheet.
- **1.6 Error Handling and Control Flow:** Manages errors during API requests and controls the continuation or termination of loops.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Scheduling

**Overview:**  
This block manages how the workflow starts. It can be manually triggered or scheduled to run hourly. It then fetches the list of Google Maps search URLs to scrape from a Google Sheet.

**Nodes Involved:**  
- When clicking "Execute Workflow"  
- Run workflow every hours  
- Google Sheets - Get searches to scrap  
- Sticky Note1 (Adjust frequency)  
- Sticky Note (Execute Workflow info)  
- Sticky Note2 (Google Sheets template link)  
- Sticky Note4 (Workflow overview)

**Node Details:**

- **When clicking "Execute Workflow"**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual start of the workflow.  
  - *Inputs:* None  
  - *Outputs:* Starts flow to get search URLs.  
  - *Failure modes:* None typical; manual start dependent on user interaction.

- **Run workflow every hours**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers workflow every hour.  
  - *Configuration:* Interval set to 1 hour.  
  - *Inputs:* None  
  - *Outputs:* Initiates fetching search URLs for scraping.  
  - *Failure modes:* Errors in scheduling or time zone misconfiguration.

- **Google Sheets - Get searches to scrap**  
  - *Type:* Google Sheets node (read)  
  - *Role:* Reads search URLs from a Google Sheet to process.  
  - *Configuration:* Reads sheet with filter on "Status" column (presumably to get pending searches).  
  - *Credentials:* Google Sheets OAuth2 account.  
  - *Inputs:* Trigger nodes above.  
  - *Outputs:* JSON items containing URLs and metadata for scraping.  
  - *Failure modes:* Authentication failure, sheet access permission, empty or malformed data.

- **Sticky Notes (1, 2, 4)**  
  - Provide instructions on scheduling adjustment, template spreadsheet link, and workflow overview.

---

#### 2.2 URL Parsing

**Overview:**  
Extracts the search keyword and geographic coordinates (latitude/longitude) from the provided Google Maps URLs to prepare parameters for SerpAPI calls.

**Nodes Involved:**  
- Extract keyword and location from URL  
- Sticky Note3 (SerpAPI API key instructions)

**Node Details:**

- **Extract keyword and location from URL**  
  - *Type:* Set node  
  - *Role:* Uses regex to parse `keyword` and `geo` from the URL string.  
  - *Configuration:*  
    - Extracts keyword from URL path using regex `/\/search\/(.*?)\//`.  
    - Extracts geo coordinates from URL using regex `/(@[^\/?]+)/`.  
  - *Inputs:* Output from Google Sheets node.  
  - *Outputs:* Adds `keyword` and `geo` fields to the item JSON.  
  - *Failure modes:* If URL format is unexpected or malformed, regex may fail causing empty or incorrect fields.

- **Sticky Note3**  
  - Instruction to add SerpAPI API key obtained from serpapi.com account.

---

#### 2.3 SerpAPI Querying and Pagination

**Overview:**  
This block sends HTTP requests to the SerpAPI service to retrieve Google Maps data, manages pagination by extracting the next page token, and loops accordingly until all pages are processed.

**Nodes Involved:**  
- SERPAPI - Scrape Google Maps URL  
- Extract next start value  
- Continue IF Loop is complete  
- Update Status to Error  
- Sticky Note3 (API Key instructions repeated here)

**Node Details:**

- **SERPAPI - Scrape Google Maps URL**  
  - *Type:* HTTP Request  
  - *Role:* Queries SerpAPI with parameters to fetch Google Maps search results.  
  - *Configuration:*  
    - URL: `https://serpapi.com/search.json`  
    - Query parameters include: `engine=google_maps`, `q` (search keyword), `ll` (lat/lng), `type=search`, and `start` (pagination offset).  
    - Authentication: Uses SerpAPI API key credential.  
  - *Input:* Parameters from URL parsing node (keyword, geo, start).  
  - *Outputs:* JSON response with search results and pagination info.  
  - *Error Handling:* On error, continues workflow but triggers Update Status to Error node.  
  - *Failure modes:* API key invalid, rate limits, network timeouts, malformed queries.

- **Extract next start value**  
  - *Type:* Code node  
  - *Role:* Parses the `next` URL from SerpAPI pagination data to extract the `start` value for the next page.  
  - *Logic:* If `serpapi_pagination.next` exists, extracts `start` parameter from URL and sets it for next request.  
  - *Inputs:* Output from SERPAPI node.  
  - *Outputs:* Item with updated `start` value for pagination.  
  - *Failure modes:* Pagination info missing or malformed, causing failure in split or access.  
  - *Version:* Uses typeVersion 2 supporting JavaScript code.

- **Continue IF Loop is complete**  
  - *Type:* IF node  
  - *Role:* Checks if pagination should continue by verifying if `start` and `serpapi_pagination.next` are present and not empty.  
  - *Inputs:* Output from Extract next start value.  
  - *Outputs:*  
    - If true: loops back to SERPAPI node for next page.  
    - If false: proceeds to data merging.  
  - *Failure modes:* Logic error if condition is inaccurate or data missing.

- **Update Status to Error**  
  - *Type:* Google Sheets node (update)  
  - *Role:* Marks the search URL status as "❌" in the input Google Sheet in case of API query failure.  
  - *Inputs:* Triggered on error from SERPAPI node.  
  - *Outputs:* None further.  
  - *Failure modes:* Google Sheets authentication issues, update conflicts.

---

#### 2.4 Data Aggregation and Cleaning

**Overview:**  
Aggregates data results from all paginated API calls, splits combined lists, filters out empty or null entries, transforms data into the expected format, and removes duplicates.

**Nodes Involved:**  
- Merge all values from SERPAPI  
- Split out items  
- Remove empty values  
- Transform data in the right format  
- Remove duplicate items

**Node Details:**

- **Merge all values from SERPAPI**  
  - *Type:* Code node  
  - *Role:* Loops through all SERPAPI node outputs, collects `local_results` arrays, and merges into a single array.  
  - *Logic:* Uses a `do...while(true)` with try-catch to accumulate pages until no more pages are found.  
  - *Inputs:* Multiple output items from SERPAPI node (paginated).  
  - *Outputs:* Single item with merged data array under `allData`.  
  - *Failure modes:* Infinite loop risk if try-catch does not break appropriately; malformed data structure could cause exceptions.

- **Split out items**  
  - *Type:* ItemLists node  
  - *Role:* Splits the aggregated `allData` array into individual items for further processing.  
  - *Inputs:* Output from merge node.  
  - *Outputs:* Individual place items.  
  - *Failure modes:* If `allData` is empty or null, may produce empty output.

- **Remove empty values**  
  - *Type:* Filter node  
  - *Role:* Filters out empty or null items to ensure only valid data proceeds.  
  - *Condition:* Checks if the first property in the JSON object is not empty.  
  - *Inputs:* Output from Split out items.  
  - *Outputs:* Filtered list.  
  - *Failure modes:* If data structure unexpected, filter may exclude valid items or fail.

- **Transform data in the right format**  
  - *Type:* Code node  
  - *Role:* Flattens and merges nested JSON data objects into a single array, removes null entries.  
  - *Logic:* Iterates over input items, pushes contained objects into a merged array, filters out nulls.  
  - *Inputs:* Filtered item list.  
  - *Outputs:* Array of cleaned and formatted place data objects.  
  - *Failure modes:* Unexpected data nesting could cause mapping errors.

- **Remove duplicate items**  
  - *Type:* ItemLists node  
  - *Role:* Removes duplicates based on `place_id` field to ensure unique place entries.  
  - *Inputs:* Transformed data array.  
  - *Outputs:* Deduplicated list.  
  - *Failure modes:* If `place_id` missing or inconsistent, deduplication may fail or remove incorrect items.

---

#### 2.5 Output and Status Management

**Overview:**  
Appends or updates the cleaned and deduplicated place data into a Google Sheet, then updates the status of the original search URL as successful.

**Nodes Involved:**  
- Add rows in Google Sheets  
- Update Status to Success

**Node Details:**

- **Add rows in Google Sheets**  
  - *Type:* Google Sheets node (append or update)  
  - *Role:* Inserts or updates place data rows into a specified result sheet.  
  - *Configuration:*  
    - Matches on `place_id` column for update or append.  
    - Maps multiple place data fields (phone, website, rating, reviews, address, coordinates, etc.) automatically.  
    - Uses Google Sheets OAuth2 credential.  
  - *Inputs:* Deduplicated place data from prior node.  
  - *Outputs:* Confirms data insertion.  
  - *Failure modes:* Sheet access issues, quota limits, malformed data fields.

- **Update Status to Success**  
  - *Type:* Google Sheets node (update)  
  - *Role:* Marks the search URL status as "✅" in the input Google Sheet to indicate processing success.  
  - *Inputs:* Triggered after successful data insertion.  
  - *Outputs:* None further.  
  - *Failure modes:* Google Sheets write permission errors.

---

#### 2.6 Error Handling and Control Flow

**Overview:**  
Manages error conditions during SerpAPI calls by updating status, and controls the looping logic to process all pages until complete.

**Nodes Involved:**  
- Update Status to Error  
- Continue IF Loop is complete

**Node Details:**  
Discussed previously in block 2.3 and 2.5.

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                        | Input Node(s)                         | Output Node(s)                         | Sticky Note                          |
|----------------------------------|-------------------------|-------------------------------------|-------------------------------------|--------------------------------------|------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger          | Manual start of workflow             | None                                | Google Sheets - Get searches to scrap | Click on Execute Workflow to run manually |
| Run workflow every hours          | Schedule Trigger        | Automatic hourly start               | None                                | Google Sheets - Get searches to scrap | Adjust frequency to your own needs |
| Google Sheets - Get searches to scrap | Google Sheets (read)    | Reads search URLs to process         | Manual Trigger, Schedule Trigger    | Extract keyword and location from URL | Copy template and connect to n8n (link provided) |
| Extract keyword and location from URL | Set                     | Extracts keyword and geo from URL    | Google Sheets - Get searches to scrap | SERPAPI - Scrape Google Maps URL      | Add SERPAPI API key instructions   |
| SERPAPI - Scrape Google Maps URL | HTTP Request            | Queries SerpAPI for places data      | Extract keyword and location from URL, Continue IF Loop is complete | Extract next start value, Update Status to Error | Add SERPAPI API key instructions   |
| Extract next start value          | Code                    | Extracts pagination start value      | SERPAPI - Scrape Google Maps URL    | Continue IF Loop is complete          |                                    |
| Continue IF Loop is complete      | IF                      | Controls pagination loop continuation| Extract next start value            | SERPAPI - Scrape Google Maps URL, Merge all values from SERPAPI |                                    |
| Merge all values from SERPAPI     | Code                    | Aggregates all pages data            | Continue IF Loop is complete        | Split out items                      |                                    |
| Split out items                  | ItemLists               | Splits merged array into items       | Merge all values from SERPAPI       | Remove empty values                  |                                    |
| Remove empty values              | Filter                  | Filters out empty/null entries       | Split out items                     | Transform data in the right format    |                                    |
| Transform data in the right format| Code                    | Flattens and cleans data             | Remove empty values                 | Remove duplicate items               |                                    |
| Remove duplicate items           | ItemLists               | Removes duplicate place entries      | Transform data in the right format  | Add rows in Google Sheets             |                                    |
| Add rows in Google Sheets        | Google Sheets (append/update) | Inserts or updates place data        | Remove duplicate items              | Update Status to Success             |                                    |
| Update Status to Success         | Google Sheets (update)  | Marks search URL as successfully processed | Add rows in Google Sheets           | None                                 |                                    |
| Update Status to Error           | Google Sheets (update)  | Marks search URL as failed            | SERPAPI - Scrape Google Maps URL (on error) | None                           |                                    |
| Sticky Note1                    | Sticky Note             | Instruction on scheduling frequency  | None                               | None                                 | Adjust frequency to your own needs |
| Sticky Note2                    | Sticky Note             | Template Google Sheets link           | None                               | None                                 | https://docs.google.com/spreadsheets/d/170osqaLBql9M-4RAH3_lBKR7ZMaQqyLUkAD-88xGuEQ/edit?usp=sharing |
| Sticky Note3                    | Sticky Note             | Instruction to add SerpAPI API key    | None                               | None                                 | Start a free trial at serpapi.com and get your API key in "Your account" section |
| Sticky Note4                    | Sticky Note             | Workflow overview and description     | None                               | None                                 | https://lempire.notion.site/Scrape-Google-Maps-places-with-n8n-b7f1785c3d474e858b7ee61ad4c21136?pvs=4 |
| Sticky Note                     | Sticky Note             | Manual execution instruction          | None                               | None                                 | Click on Execute Workflow to run the workflow manually |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking "Execute Workflow"` for manual execution start.

2. **Create a Schedule Trigger node** named `Run workflow every hours`, set to trigger every hour.

3. **Create a Google Sheets node** named `Google Sheets - Get searches to scrap`:  
   - Operation: Read rows  
   - Sheet name: `Add your search here` (or your own sheet with search URLs)  
   - Filter: On `Status` column to get pending entries  
   - Credentials: Set up Google Sheets OAuth2 credentials

4. **Connect both trigger nodes** (Manual and Schedule) to `Google Sheets - Get searches to scrap`.

5. **Create a Set node** named `Extract keyword and location from URL`:  
   - Add fields:  
     - `keyword` with expression: `{{$json.URL.match(/\/search\/(.*?)\//)[1]}}`  
     - `geo` with expression: `{{$json.URL.match(/(@[^\/?]+)/)[1]}}`

6. **Connect** `Google Sheets - Get searches to scrap` to `Extract keyword and location from URL`.

7. **Create an HTTP Request node** named `SERPAPI - Scrape Google Maps URL`:  
   - HTTP Method: GET  
   - URL: `https://serpapi.com/search.json`  
   - Query Parameters:  
     - `engine`: `google_maps`  
     - `q`: `={{$json?.search_parameters?.q || $json.keyword}}`  
     - `ll`: `={{$json?.search_parameters?.ll || $json.geo}}`  
     - `type`: `search`  
     - `start`: `={{$json.start || 0}}`  
   - Authentication: Use SerpAPI API key credential (must be created beforehand)  
   - On Error: Continue (to allow error handling downstream)

8. **Connect** `Extract keyword and location from URL` to `SERPAPI - Scrape Google Maps URL`.

9. **Create a Code node** named `Extract next start value`:  
   - Mode: Run once per item  
   - JS Code (logic to extract next pagination start from SerpAPI response):  
     ```js
     let nextUrl;
     if ($json && $json["serpapi_pagination"] && $json["serpapi_pagination"]["next"]) {
         nextUrl = $json["serpapi_pagination"]["next"];
         $input.item.json.start = nextUrl.split('&').find(param => param.startsWith('start=')).split('=')[1];
     }
     return $input.item;
     ```

10. **Connect** `SERPAPI - Scrape Google Maps URL` to `Extract next start value`.

11. **Create an IF node** named `Continue IF Loop is complete`:  
    - Condition:  
      - `search_parameters.start` is not empty (number)  
      - AND `serpapi_pagination.next` is not empty (string)  
    - True output: Connect back to `SERPAPI - Scrape Google Maps URL`  
    - False output: Connect to data merging node

12. **Connect** `Extract next start value` to `Continue IF Loop is complete`.

13. **Create a Code node** named `Merge all values from SERPAPI`:  
    - JS Code to loop through all SERPAPI outputs and merge `local_results` arrays into one array.  
    - Implement with try-catch to detect end of pages.

14. **Connect** the false output of `Continue IF Loop is complete` to `Merge all values from SERPAPI`.

15. **Create an ItemLists node** named `Split out items`:  
    - Operation: Split out based on `allData` field.

16. **Connect** `Merge all values from SERPAPI` to `Split out items`.

17. **Create a Filter node** named `Remove empty values`:  
    - Condition: The first property in JSON is not empty.

18. **Connect** `Split out items` to `Remove empty values`.

19. **Create a Code node** named `Transform data in the right format`:  
    - Flatten nested data arrays into a single array and remove null entries.  
    - JS code iterates over `$input.all()` and pushes all nested objects into a merged array.

20. **Connect** `Remove empty values` to `Transform data in the right format`.

21. **Create an ItemLists node** named `Remove duplicate items`:  
    - Operation: Remove duplicates based on `place_id` field.

22. **Connect** `Transform data in the right format` to `Remove duplicate items`.

23. **Create a Google Sheets node** named `Add rows in Google Sheets`:  
    - Operation: Append or update  
    - Sheet name: your results sheet (e.g., "Results")  
    - Match column: `place_id`  
    - Map columns for all relevant place fields (phone, website, rating, reviews, address, coordinates, etc.)  
    - Credentials: Google Sheets OAuth2

24. **Connect** `Remove duplicate items` to `Add rows in Google Sheets`.

25. **Create a Google Sheets node** named `Update Status to Success`:  
    - Operation: Update  
    - Sheet name: input sheet (same as step 3)  
    - Match column: `URL`  
    - Set column `Status` to "✅"  
    - Credentials: Google Sheets OAuth2

26. **Connect** `Add rows in Google Sheets` to `Update Status to Success`.

27. **Create a Google Sheets node** named `Update Status to Error`:  
    - Operation: Update  
    - Sheet name: input sheet  
    - Match column: `URL`  
    - Set `Status` to "❌"  
    - Credentials: Google Sheets OAuth2

28. **Connect** the error output of `SERPAPI - Scrape Google Maps URL` to `Update Status to Error`.

29. **Add Sticky Notes** at various points to document:  
    - Workflow overview and description (link to Notion guide)  
    - Instructions for setting up SerpAPI API key  
    - Scheduling adjustment recommendations  
    - Template Google Sheet link

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                                           |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Full guide to implement the workflow is here: Scrape Google Maps places with n8n                          | https://lempire.notion.site/Scrape-Google-Maps-places-with-n8n-b7f1785c3d474e858b7ee61ad4c21136?pvs=4                     |
| Template Google Sheets for input and output data                                                          | https://docs.google.com/spreadsheets/d/170osqaLBql9M-4RAH3_lBKR7ZMaQqyLUkAD-88xGuEQ/edit?usp=sharing                      |
| SerpAPI API key setup instructions: Start a free trial at serpapi.com and get your API key in "Your account" section | https://serpapi.com                                                                                                       |

---

This documentation fully describes the Google Maps Scraper workflow, enabling advanced users and AI systems to understand, reproduce, and modify it precisely. It anticipates key error types such as API authentication errors, pagination handling, data formatting issues, and Google Sheets access problems.