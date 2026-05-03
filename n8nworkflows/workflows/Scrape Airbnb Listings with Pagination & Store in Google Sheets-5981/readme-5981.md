Scrape Airbnb Listings with Pagination & Store in Google Sheets

https://n8nworkflows.xyz/workflows/scrape-airbnb-listings-with-pagination---store-in-google-sheets-5981


# Scrape Airbnb Listings with Pagination & Store in Google Sheets

---

## 1. Workflow Overview

This n8n workflow automates the process of scraping Airbnb listings for a specified location with support for pagination, detailed data extraction, and storage into a Google Sheet. It is designed to iteratively fetch multiple pages of Airbnb search results, parse and enhance listing details, and aggregate all listings into a structured dataset for further analysis or reporting.

The workflow is logically divided into the following functional blocks:

- **1.1 Trigger & Initialization:** Manual start and initial setup of pagination state.
- **1.2 Airbnb Search & Pagination Loop:** Execution of Airbnb search queries with cursor-based pagination, managing the loop over multiple pages.
- **1.3 Data Parsing & Aggregation:** Parsing raw search results JSON, extracting listings, and accumulating listings across pages.
- **1.4 Listing Detail Enhancement:** Fetching additional details for each individual listing, including house rules, highlights, description, and amenities.
- **1.5 Data Formatting & Google Sheets Integration:** Formatting extracted data and updating/appending it into a Google Sheet for persistent storage.
- **1.6 Final Aggregation:** Summarizing the complete dataset with metadata after the pagination loop completes.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Initialization

- **Overview:**  
  This block starts the workflow via a manual trigger and initializes pagination variables such as the current page count, cursor, and the aggregated list of all listings.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Initial Set1 (Set)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow execution manually.  
    - Configuration: No parameters, simply waits for manual execution.  
    - Inputs: None  
    - Outputs: Triggers the next node to start the workflow.  
    - Edge cases: None; manual trigger does not fail but requires user action.  

  - **Initial Set1**  
    - Type: Set  
    - Role: Initializes key pagination and aggregation variables:
      - `loopCount`: Number of pages processed (default 0).  
      - `cursor`: Pagination cursor string (default empty).  
      - `all_listings`: Aggregated array of all listings fetched so far (default empty array).  
    - Configuration: Values are conditionally set from incoming JSON or defaulted.  
    - Inputs: From manual trigger and Airbnb Search (loop iteration).  
    - Outputs: Passes pagination state to the next processing nodes.  
    - Edge cases: Must handle first run where no prior data exists.

---

### 2.2 Airbnb Search & Pagination Loop

- **Overview:**  
  This block executes the Airbnb search query for the specified location and parameters, handles pagination through cursor tokens, and governs the loop continuation conditions.

- **Nodes Involved:**  
  - Airbnb Search (MCP Client)  
  - Merge5 (Merge)  
  - Parse Listing Data2 (Code)  
  - If1 (If)  

- **Node Details:**

  - **Airbnb Search**  
    - Type: MCP Client (custom integration)  
    - Role: Calls the Airbnb search tool with location, guest counts, dates, and pagination cursor.  
    - Configuration:  
      - Location: "London" (modifiable)  
      - Adults: 7  
      - Children: 1  
      - Checkin: "2025-08-14"  
      - Checkout: "2025-08-17"  
      - Cursor: from previous pagination state (`{{$json.cursor || ''}}`)  
    - Inputs: Pagination state from Initial Set1 or loop iteration.  
    - Outputs: Raw Airbnb search JSON results.  
    - Edge cases: API errors, rate limits, invalid cursor, network timeouts.  
    - Requires valid MCP Client API credentials with Airbnb tools access.  

  - **Merge5**  
    - Type: Merge (Combine)  
    - Role: Combines outputs from Airbnb Search and Initial Set1 into a single data stream for parsing.  
    - Configuration: Combine by position (array index).  
    - Inputs: Airbnb Search and Initial Set1 outputs.  
    - Outputs: Combined data for parsing.  
    - Edge cases: Misaligned inputs or missing data could cause downstream failures.  

  - **Parse Listing Data2**  
    - Type: Code  
    - Role: Parses raw Airbnb search JSON, extracts listings, pagination cursor, increments loop count, and aggregates listings.  
    - Key logic:  
      - Parses `result.content[0].text` or `content[0].text` for listings and pagination info.  
      - Extracts listing properties: name, price, rating, reviews, URL, coordinates, checkin/out dates, badges.  
      - Extracts listing ID from URL regex.  
      - Aggregates listings across pages into `all_listings`.  
      - Generates metadata object containing all listings, pagination cursor, loop count, and counts.  
    - Inputs: Combined data from Merge5.  
    - Outputs: Metadata object for loop control and accumulation.  
    - Edge cases: Invalid JSON, missing fields, no listings, malformed URLs.  

  - **If1**  
    - Type: If  
    - Role: Controls pagination loop continuation.  
    - Configuration:  
      - Condition 1: `loopCount` less than 2 (page limit)  
      - Condition 2: `cursor` is not empty (more pages available)  
      - Both must be true to continue looping.  
    - Inputs: Metadata from Parse Listing Data2.  
    - Outputs:  
      - True branch: Continue loop (send to Initial Set1 and Airbnb Search again)  
      - False branch: Exit loop and proceed to final aggregation.  
    - Edge cases: Cursor missing or loopCount exceeding limit causes loop to terminate.

---

### 2.3 Data Parsing & Aggregation

- **Overview:**  
  This block further processes the aggregated listings, formats data for downstream usage, and prepares for detail enhancement.

- **Nodes Involved:**  
  - Final Results (Code)  
  - Code2 (Code)  

- **Node Details:**

  - **Final Results**  
    - Type: Code  
    - Role: Produces a summary object with total listings, pages processed, search parameters, and the full listings array.  
    - Configuration: Constructs a JSON object containing:  
      - `total_listings`, `pages_processed` from metadata  
      - `listings` array of all listings  
      - `summary` object with search info (location: "Da Nang" [note: different from actual search], dates, guests)  
    - Inputs: Metadata object from Parse Listing Data2 (loop exit).  
    - Outputs: Aggregated final data for storage.  
    - Edge cases: Empty listings, mismatch in summary data.  

  - **Code2**  
    - Type: Code  
    - Role: Extracts and maps relevant fields from final aggregated data for Google Sheets update.  
    - Configuration: Extracts fields like id, name, beds_rooms, price, rating, reviews, location, badge, checkin/out, url.  
    - Inputs: Final Results output.  
    - Outputs: Array of listings formatted for Google Sheets node.  
    - Edge cases: Missing fields, null values.

---

### 2.4 Listing Detail Enhancement

- **Overview:**  
  This block loops over each listing to fetch detailed Airbnb listing information like house rules, highlights, description, and amenities.

- **Nodes Involved:**  
  - Edit Fields (Set)  
  - Loop Over Items (SplitInBatches)  
  - Get Airbnb Listing Details (MCP Client)  
  - Format Data (Code)  

- **Node Details:**

  - **Edit Fields**  
    - Type: Set  
    - Role: Ensures each item has an `id` field properly assigned for detail fetching.  
    - Configuration: Assigns `id` from each listing JSON.  
    - Inputs: Listings array from Code2 or prior step.  
    - Outputs: Individual items ready for batch processing.  
    - Edge cases: Missing or invalid id fields.  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes listings in batches (default batch size) to avoid overloading API calls.  
    - Configuration: Default options (single batch size not explicitly set).  
    - Inputs: Items from Edit Fields.  
    - Outputs: Batches of listing ids for detail lookup.  
    - Edge cases: Batch size too large causing timeouts or rate limits.  

  - **Get Airbnb Listing Details**  
    - Type: MCP Client  
    - Role: Calls Airbnb listing details tool for each listing id to fetch detailed data.  
    - Configuration: Passes listing `id` as tool parameter.  
    - Inputs: Each batch item from Loop Over Items.  
    - Outputs: Raw detailed JSON for each listing.  
    - Edge cases: API errors, invalid ids, rate limits.  

  - **Format Data**  
    - Type: Code  
    - Role: Parses detailed JSON for each listing, extracts specific fields (house rules, highlights, description, amenities, location).  
    - Configuration:  
      - Parses JSON from raw text field.  
      - Extracts listingUrl and id.  
      - Iterates over sections to extract location coordinates, house rules, highlights, description, amenities.  
      - Returns structured JSON with these fields.  
    - Inputs: Raw detail JSON from Get Airbnb Listing Details.  
    - Outputs: Formatted, clean data ready for Google Sheets.  
    - Edge cases: Missing or malformed JSON, empty fields.

---

### 2.5 Data Formatting & Google Sheets Integration

- **Overview:**  
  This block writes the fully processed and enriched listing data into a Google Sheet, handling both appending and updating records.

- **Nodes Involved:**  
  - Clear Google Sheet (Google Sheets)  
  - Update Google Sheet (Google Sheets)  

- **Node Details:**

  - **Clear Google Sheet**  
    - Type: Google Sheets  
    - Role: Appends or updates enriched listing data into the Google Sheet (sheet ID: 15IOJquaQ8CBtFilmFTuW8UFijux10NwSVzStyNJ1MsA, gid=0).  
    - Configuration:  
      - Operation: appendOrUpdate  
      - Matching Column: "id" (to avoid duplicate entries)  
      - Columns: id, amenities, highlights, houseRules, description (string fields mapped from JSON)  
    - Inputs: Formatted listing data from Format Data node.  
    - Outputs: Google Sheets update results.  
    - Credentials: Google Sheets OAuth2 credentials configured with access rights.  
    - Edge cases: Authentication failure, quota limits, invalid spreadsheet ID.  

  - **Update Google Sheet**  
    - Type: Google Sheets  
    - Role: Appends basic listing information (id, url, name, badge, rating, reviews, location, beds_rooms, total_price, price_details, price_per_night) into the same Google Sheet.  
    - Configuration:  
      - Operation: append  
      - Columns mapped accordingly with matching column "id".  
    - Inputs: Listings array from Code2.  
    - Credentials: Same as Clear Google Sheet.  
    - Edge cases: Same as above, plus data type mismatches.

---

## 3. Summary Table

| Node Name                | Node Type                  | Functional Role                          | Input Node(s)                       | Output Node(s)                   | Sticky Note                                                             |
|--------------------------|----------------------------|----------------------------------------|-----------------------------------|---------------------------------|-------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Start workflow manually                 | None                              | Airbnb Search, Initial Set1       |                                                                         |
| Initial Set1             | Set                        | Initialize pagination and aggregation state | When clicking ‘Execute workflow’  | Merge5                          |                                                                         |
| Airbnb Search            | MCP Client (Airbnb tool)   | Search Airbnb listings with pagination | Initial Set1                      | Merge5                          |                                                                         |
| Merge5                   | Merge (Combine)            | Combine search results and state        | Airbnb Search, Initial Set1        | Parse Listing Data2              |                                                                         |
| Parse Listing Data2      | Code                       | Parse listings, extract metadata        | Merge5                           | If1                            |                                                                         |
| If1                      | If                         | Control pagination loop continuation    | Parse Listing Data2               | Initial Set1 (True), Final Results (False) |                                                                         |
| Final Results            | Code                       | Aggregate final results summary          | If1 (False branch)                | Code2                          |                                                                         |
| Code2                    | Code                       | Format listings for Google Sheets update | Final Results                    | Update Google Sheet             |                                                                         |
| Update Google Sheet      | Google Sheets              | Append listing info to Google Sheet      | Code2                            | Edit Fields                    |                                                                         |
| Edit Fields              | Set                        | Prepare listings for detail fetching     | Update Google Sheet               | Loop Over Items                |                                                                         |
| Loop Over Items          | SplitInBatches             | Process listings in batches               | Edit Fields                     | Format Data, Get Airbnb Listing Details |                                                                         |
| Get Airbnb Listing Details | MCP Client (Airbnb tool)   | Fetch detailed Airbnb listing info       | Loop Over Items                  | Loop Over Items                |                                                                         |
| Format Data              | Code                       | Parse and format detailed listing data   | Loop Over Items                  | Clear Google Sheet             |                                                                         |
| Clear Google Sheet       | Google Sheets              | Append or update detailed data in Google Sheet | Format Data                     | None                          |                                                                         |
| Sticky Note              | Sticky Note                | Informational note                        | None                            | None                          | ## Airbnb Search Flow - Description & Notes\n## Overview\nThis n8n workflow implements a paginated **Airbnb search system** that automatically fetches multiple pages of listings and aggregates them into a single comprehensive dataset. The flow uses a loop mechanism to handle pagination and collects all results efficiently. |
| Sticky Note1             | Sticky Note                | Informational note                        | None                            | None                          | This n8n workflow processes Airbnb listing data in a loop structure. Here's how it flows:\n\n## Flow Overview\nThe workflow creates a processing loop that fetches and formats Airbnb listing details for multiple properties. |
| Sticky Note2             | Sticky Note                | Setup instructions and prerequisites     | None                            | None                          | # Setup Steps\n## Prerequisites\nn8n instance with MCP (Model Context Protocol) support\nGoogle Sheets API credentials configured\nAirbnb MCP client properly set up\n\n## Configuration Steps\n- Configure MCP Client\nSet up the Airbnb MCP client with credential ID:\nEnsure the client has access to airbnb_search and airbnb_listing_details tools\n- Google Sheets Setup\nCreate a Google Sheet with ID: 15IOJquaQ8CBtFilmFTuW8UFijux10NwSVzStyNJ1MsA\nConfigure Google Sheets OAuth2 credentials (ID: 6YhBlgb8cXMN3Ra2)\n-- Ensure the sheet has these column headers:\n\"id, name, url, price_per_night, total_price, price_details\nbeds_rooms, rating, reviews, badge, location\nhouseRules, highlights, description, amenities\"\n\n\n- Search Parameters\nLocation: \"London\" (can be modified in the \"Airbnb Search\" node)\nAdults: 7\nChildren: 1\nCheck-in: \"2025-08-14\"\nCheck-out: \"2025-08-17\"\nPage limit: 2 (can be adjusted in the \"If1\" condition node)\n\n\n- Execution\nUse the manual trigger \"When clicking 'Execute workflow'\" to start the process\nMonitor the workflow execution through the n8n interface\nCheck the Google Sheet for populated data after completion |
| Sticky Note3             | Sticky Note                | Workflow description and architecture overview | None                            | None                          | # Description\n### This n8n workflow automatically **scrapes Airbnb listings** from a specified location and **saves the data to a Google Sheet**. It performs pagination to collect listings across multiple pages, extracts detailed information for each property, and organizes the data in a structured format for easy analysis.\n\n# How it Works\nThe workflow operates through these high-level steps:\n\n- Search Initialization: Starts with an Airbnb search for a specific location (London) with defined check-in/check-out dates and guest count\n- Pagination Loop: Automatically processes multiple pages of search results using cursor-based pagination\n- Data Extraction: Parses listing information including names, prices, ratings, reviews, and URLs\n- Detail Enhancement: Fetches additional details for each listing (house rules, highlights, descriptions, amenities)\n- Data Storage: Saves all collected data to a Google Sheet with proper formatting\n- Loop Control: Continues until reaching the page limit (2 pages) or no more results are available |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow execution.

2. **Create a Set Node for Initialization**  
   - Name: `Initial Set1`  
   - Assign variables:  
     - `loopCount` (Number): `={{$json.loopCount || 0}}`  
     - `cursor` (String): `={{$json.cursor || ''}}`  
     - `all_listings` (Array): `={{$json.all_listings || []}}`  
   - Connect `When clicking ‘Execute workflow’` → `Initial Set1`.

3. **Create MCP Client Node for Airbnb Search**  
   - Name: `Airbnb Search`  
   - Tool Name: `airbnb_search`  
   - Operation: `executeTool`  
   - Tool Parameters (JSON):  
     ```json
     {
       "location": "London",
       "adults": 7,
       "children": 1,
       "checkin": "2025-08-14",
       "checkout": "2025-08-17",
       "cursor": "{{ $json.cursor || '' }}"
     }
     ```  
   - Set credentials for MCP Client API with Airbnb access.  
   - Connect `Initial Set1` → `Airbnb Search`.

4. **Add a Merge Node to Combine Search and State**  
   - Name: `Merge5`  
   - Mode: Combine  
   - Combine By: Position  
   - Connect `Initial Set1` and `Airbnb Search` as inputs. Output goes to next node.

5. **Create Code Node to Parse Listing Data**  
   - Name: `Parse Listing Data2`  
   - JavaScript code: Parses raw JSON from search results, extracts listings, pagination cursor, increments loop count, merges listings.  
   - Connect `Merge5` → `Parse Listing Data2`.

6. **Create If Node to Control Pagination Loop**  
   - Name: `If1`  
   - Conditions:  
     - `loopCount < 2` (number)  
     - `cursor` is not empty (string not empty)  
     - Both conditions combined with AND.  
   - Connect `Parse Listing Data2` → `If1`.

7. **Connect If Node True Branch back to Initialization**  
   - Connect `If1` True output → `Initial Set1` to loop next page.

8. **Create Code Node to Aggregate Final Results**  
   - Name: `Final Results`  
   - JavaScript code: Produces summary JSON with total listings, pages processed, search parameters, and full listing array.  
   - Connect `If1` False output → `Final Results`.

9. **Create Code Node to Format Listings for Google Sheets**  
   - Name: `Code2`  
   - JavaScript code: Extracts and formats listing fields for Google Sheets ingestion.  
   - Connect `Final Results` → `Code2`.

10. **Create Google Sheets Node to Append Basic Listing Info**  
    - Name: `Update Google Sheet`  
    - Operation: Append  
    - Document ID: `15IOJquaQ8CBtFilmFTuW8UFijux10NwSVzStyNJ1MsA`  
    - Sheet Name: `gid=0`  
    - Map columns: id, url, name, badge, rating, reviews, location, beds_rooms, total_price, price_details, price_per_night  
    - Credentials: Google Sheets OAuth2 API configured with correct access.  
    - Connect `Code2` → `Update Google Sheet`.

11. **Create Set Node to Prepare for Detail Fetching**  
    - Name: `Edit Fields`  
    - Assign `id` from each listing JSON.  
    - Connect `Update Google Sheet` → `Edit Fields`.

12. **Create SplitInBatches Node to Process Listings in Batches**  
    - Name: `Loop Over Items`  
    - Default batch size (can be adjusted if needed).  
    - Connect `Edit Fields` → `Loop Over Items`.

13. **Create MCP Client Node to Fetch Listing Details**  
    - Name: `Get Airbnb Listing Details`  
    - Tool Name: `airbnb_listing_details`  
    - Operation: `executeTool`  
    - Tool Parameters: JSON with `"id": "{{$json.id}}"`  
    - Credentials: MCP Client API with Airbnb access.  
    - Connect `Loop Over Items` → `Get Airbnb Listing Details`.

14. **Connect `Get Airbnb Listing Details` output back to `Loop Over Items` input**  
    - This creates a loop to process all batches.

15. **Create Code Node to Format Detailed Listing Data**  
    - Name: `Format Data`  
    - JavaScript code: Parses detailed JSON, extracts house rules, highlights, description, amenities, location coordinates, and listing id.  
    - Connect `Loop Over Items` → `Format Data`.

16. **Create Google Sheets Node to Append or Update Detailed Data**  
    - Name: `Clear Google Sheet`  
    - Operation: Append Or Update  
    - Document ID and Sheet Name same as above.  
    - Map columns: id, houseRules, highlights, description, amenities  
    - Credentials: Same Google Sheets OAuth2.  
    - Connect `Format Data` → `Clear Google Sheet`.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow requires an n8n instance with MCP (Model Context Protocol) support and configured MCP Client API credentials with access to Airbnb tools: `airbnb_search` and `airbnb_listing_details`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Setup prerequisite                                                                                       |
| Google Sheets API credentials must be configured with OAuth2 and access granted to the target spreadsheet with ID `15IOJquaQ8CBtFilmFTuW8UFijux10NwSVzStyNJ1MsA`. The sheet should contain columns: `id, name, url, price_per_night, total_price, price_details, beds_rooms, rating, reviews, badge, location, houseRules, highlights, description, amenities`.                                                                                                                                                                                                                                                                                                                                                                          | Google Sheets setup                                                                                      |
| Search parameters such as location, guests, and dates can be modified in the `Airbnb Search` MCP Client node parameters. The pagination page limit is currently set to 2 pages in the `If1` node but can be adjusted for larger data collection.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Configuration flexibility                                                                                |
| Use the manual trigger node to start execution. Monitor progress in the n8n execution UI. After completion, verify data in the Google Sheet.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Execution instructions                                                                                   |
| The workflow handles errors such as invalid JSON parsing in code nodes by returning error messages within JSON objects. However, API errors, timeouts, or rate limiting may require external monitoring or retries outside this workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Error handling notes                                                                                      |
| Airbnb search location in final summary is shown as “Da Nang” (hardcoded), which differs from actual search location (“London”). This may require adjustment for consistency.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Data consistency note                                                                                    |
| Sticky notes within the workflow provide detailed explanation and setup instructions. It is recommended to review these notes for operational context and prerequisites.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Embedded documentation                                                                                   |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.