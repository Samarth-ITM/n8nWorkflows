Google Maps Reviews to Google Sheets Sync using SerpApi

https://n8nworkflows.xyz/workflows/google-maps-reviews-to-google-sheets-sync-using-serpapi-8371


# Google Maps Reviews to Google Sheets Sync using SerpApi

### 1. Workflow Overview

This workflow synchronizes Google Maps reviews for any search query into a Google Sheets document using the SerpApi Google Maps and Google Maps Reviews APIs. It is designed to handle queries that return either a single place or multiple places and fetches up to a configurable limit of reviews per place. The workflow logically consists of the following blocks:

- **1.1 Input Initialization and Query Setup**: Accepts a Google Maps query string and sets parameters such as review limits.
- **1.2 Search and Result Type Handling**: Queries Google Maps via SerpApi and determines if results are for a single place or multiple places.
- **1.3 Place Looping**: If multiple places are found, splits the results into individual places and loops over them.
- **1.4 Review Fetching and Pagination**: Fetches reviews for each place in batches, handling pagination until the review limit is reached.
- **1.5 Data Appending**: Appends fetched reviews into a Google Sheet.
- **1.6 Control Flow Routing**: Routes the workflow to fetch additional review pages, move to the next place, or end the workflow based on conditions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Query Setup

- **Overview:**  
  Sets the maximum number of reviews to fetch and triggers the workflow manually to start the process.

- **Nodes Involved:**  
  - Execute Workflow (Manual Trigger)  
  - Set Review Limit

- **Node Details:**

  - **Execute Workflow**  
    - Type: Manual Trigger  
    - Role: Entry point for manual test triggering.  
    - Inputs: None  
    - Outputs: Triggers the next node.

  - **Set Review Limit**  
    - Type: Set  
    - Role: Defines `review_limit` variable with default value 50.  
    - Configuration: Sets `review_limit` to 50 (number).  
    - Inputs: Triggered by Execute Workflow  
    - Outputs: Passes review_limit to next node.  
    - Edge cases: User can increase or decrease this limit; setting too high may cause excessive API calls and consume SerpApi credits.

---

#### 2.2 Search and Result Type Handling

- **Overview:**  
  Performs a Google Maps search using SerpApi for the configured query and distinguishes whether the result is a single place or multiple places.

- **Nodes Involved:**  
  - Search Google Maps  
  - If - Local or Place Results

- **Node Details:**

  - **Search Google Maps**  
    - Type: SerpApi  
    - Role: Queries Google Maps with the user-defined query string (default: "italian restaurants in austin tx").  
    - Configuration: Uses SerpApi Google Maps operation; restricts JSON to `local_results` and `place_results` to limit response size.  
    - Inputs: Receives `review_limit` from Set Review Limit.  
    - Outputs: Sends results to conditional node to check result type.  
    - Edge cases: Query must be updated by the user to suit needs. API quota and network errors possible.

  - **If - Local or Place Results**  
    - Type: If (conditional)  
    - Role: Checks if the response has `place_results` (single place) or `local_results` (multiple places).  
    - Configuration: Condition checks if `place_results` exists.  
    - Inputs: Output of Search Google Maps.  
    - Outputs:  
      - True: Single place path (goes to Initialize Vars).  
      - False: Multiple places path (goes to Split Local Results).  
    - Edge cases: If response format changes or no results found, workflow may fail or loop incorrectly.

---

#### 2.3 Place Looping

- **Overview:**  
  For multiple places, splits the array of places and loops over each one sequentially to process their reviews.

- **Nodes Involved:**  
  - Split Local Results  
  - Loop Over Places  
  - Initialize Vars (also used for single place)

- **Node Details:**

  - **Split Local Results**  
    - Type: Split Out  
    - Role: Splits the `local_results` array into individual place objects.  
    - Configuration: Splits field `local_results`.  
    - Inputs: From If node (multiple places path).  
    - Outputs: Each place sent one-by-one to Loop Over Places.  
    - Edge cases: Empty or missing `local_results` array.

  - **Loop Over Places**  
    - Type: Split In Batches  
    - Role: Processes each place individually through the workflow.  
    - Configuration: Default batch size (usually 1).  
    - Inputs: Output from Split Local Results.  
    - Outputs: To Initialize Vars for each place to start review fetching.  
    - Edge cases: Large number of places may increase runtime; batch size controls concurrency.

  - **Initialize Vars**  
    - Type: Code  
    - Role: Initializes variables for review fetching like `place_id`, `place_name`, `review_limit`, pagination tokens, counters, and flags indicating single or multiple places.  
    - Configuration: Extracts place info from input JSON, sets defaults and flags.  
    - Inputs: Either from Loop Over Places (multiple places) or If node (single place).  
    - Outputs: To Get Reviews node.  
    - Edge cases: If expected JSON fields missing, variables may be initialized incorrectly.

---

#### 2.4 Review Fetching and Pagination

- **Overview:**  
  Fetches reviews for each place in pages, handles review count and pagination tokens, and determines when to fetch more pages or move on.

- **Nodes Involved:**  
  - Get Reviews  
  - If Reviews Present  
  - Split Reviews  
  - Append Reviews  
  - Update Review Count & Next Page Token  
  - Route Next Step  
  - Set num for Pagination

- **Node Details:**

  - **Get Reviews**  
    - Type: SerpApi  
    - Role: Calls SerpApi Google Maps Reviews API for a given place_id, with pagination parameters.  
    - Configuration: Uses `place_id`, `num` (number of reviews per page), and `next_page_token` from static data.  
    - Inputs: From Initialize Vars or Set num for Pagination.  
    - Outputs: To If Reviews Present.  
    - Edge cases: API errors, missing or invalid place_id, pagination tokens.

  - **If Reviews Present**  
    - Type: If  
    - Role: Checks if there are any reviews returned or if the current place is a single business (to decide on processing or skipping).  
    - Configuration: Condition checks if `reviews` array exists or if flagged as single business.  
    - Inputs: From Get Reviews.  
    - Outputs:  
      - True: Proceed to Split Reviews for processing.  
      - False: Go to Loop Over Places to process the next place.  
    - Edge cases: Empty reviews array, malformed data.

  - **Split Reviews**  
    - Type: Split Out  
    - Role: Splits the reviews array into individual review objects.  
    - Configuration: Splits on field `reviews`, includes `place_info.title` as extra field.  
    - Inputs: From If Reviews Present.  
    - Outputs: To Append Reviews.  
    - Edge cases: Empty reviews array.

  - **Append Reviews**  
    - Type: Google Sheets  
    - Role: Appends each review with relevant fields (`rating`, `snippet`, `iso_date`, `place_name`) to the configured Google Sheet.  
    - Configuration: Maps review fields to sheet columns using expressions referencing initialized variables and review JSON. Uses append mode.  
    - Inputs: From Split Reviews.  
    - Outputs: To Update Review Count & Next Page Token.  
    - Edge cases: Google Sheets API quota, invalid sheet ID or missing permissions.

  - **Update Review Count & Next Page Token**  
    - Type: Code  
    - Role: Increments the count of reviews fetched, updates pagination token for next page if available. Stores data in global static data.  
    - Inputs: From Append Reviews.  
    - Outputs: To Route Next Step.  
    - Edge cases: Missing pagination info, incorrect counting.

  - **Route Next Step**  
    - Type: Switch  
    - Role: Routes workflow based on whether to fetch more review pages, move to next place, or end workflow.  
    - Configuration:  
      - Get More Pages: if next_page_token exists and review limit not reached.  
      - End Workflow: if single place and no more pages or limit reached.  
      - Get Next Place: if multiple places and no more pages or limit reached.  
    - Inputs: From Update Review Count & Next Page Token.  
    - Outputs:  
      - Get More Pages: to Set num for Pagination node.  
      - End Workflow: ends execution.  
      - Get Next Place: to Loop Over Places to process next place.  
    - Edge cases: Logic errors in conditions may cause infinite loops or premature ending.

  - **Set num for Pagination**  
    - Type: Code  
    - Role: Sets `num` parameter to 20 for subsequent review pages (max reviews per page).  
    - Inputs: From Route Next Step (Get More Pages output).  
    - Outputs: To Get Reviews.  
    - Edge cases: None significant.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                              | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                     |
|--------------------------------|---------------------|----------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Execute Workflow               | Manual Trigger      | Entry point to start workflow manually       |                              | Set Review Limit             |                                                                                                                 |
| Set Review Limit               | Set                 | Sets maximum review limit per place (default 50) | Execute Workflow              | Search Google Maps           | Update the review limit here to customize maximum reviews per place. Default is 50.                            |
| Search Google Maps             | SerpApi             | Queries Google Maps for places matching query | Set Review Limit              | If - Local or Place Results  | Update the "Search Google Maps" node to set your own query.                                                    |
| If - Local or Place Results    | If                  | Determines if result is single or multiple places | Search Google Maps            | Initialize Vars (single place), Split Local Results (multiple places) | Checks whether results are for single place or multiple places.                                               |
| Split Local Results            | Split Out           | Splits multiple places into individual place records | If - Local or Place Results   | Loop Over Places             | Splits out places in the `local_results` array and loops over each.                                            |
| Loop Over Places               | Split In Batches    | Processes places one-by-one                    | Split Local Results, Route Next Step (Get Next Place) | Initialize Vars, Loop Over Places (next batch) |                                                                                                                 |
| Initialize Vars               | Code                | Initializes variables for each place           | If - Local or Place Results (single), Loop Over Places | Get Reviews                  | Initializes variables for review fetching. Do not update unless necessary.                                     |
| Get Reviews                   | SerpApi             | Fetches reviews for a place with pagination   | Initialize Vars, Set num for Pagination | If Reviews Present           | Fetches a page of reviews. Handles pagination tokens and page size.                                            |
| If Reviews Present            | If                  | Checks if reviews exist or if single business | Get Reviews                  | Split Reviews (if true), Loop Over Places (if false) |                                                                                                                 |
| Split Reviews                 | Split Out           | Splits review objects for appending           | If Reviews Present            | Append Reviews               |                                                                                                                 |
| Append Reviews               | Google Sheets        | Appends review data to Google Sheet            | Split Reviews                 | Update Review Count & Next Page Token | Appends reviews to Google Sheet with mapped columns.                                                          |
| Update Review Count & Next Page Token | Code          | Updates count of reviews and pagination token | Append Reviews                | Route Next Step              | Increments count and sets next page token if available.                                                        |
| Route Next Step              | Switch               | Routes workflow to fetch more pages, next place, or end | Update Review Count & Next Page Token | Set num for Pagination (Get More Pages), Loop Over Places (Get Next Place) | Routes fetching logic: more pages, next place, or workflow end.                                                |
| Set num for Pagination       | Code                 | Sets number of reviews per page (20)           | Route Next Step (Get More Pages) | Get Reviews                 |                                                                                                                 |
| Sticky Note                  | Sticky Note          | Various notes explaining blocks and usage      |                              |                              | Multiple sticky notes provide detailed explanations and usage instructions for nodes and workflow design.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: Execute Workflow  
   - Purpose: To manually start the workflow.

2. **Add a Set node**  
   - Name: Set Review Limit  
   - Purpose: Define `review_limit` variable (default 50).  
   - Configuration: Add number field `review_limit` = 50.  
   - Connect: Execute Workflow → Set Review Limit.

3. **Add SerpApi node for Google Maps Search**  
   - Name: Search Google Maps  
   - Credentials: Configure SerpApi credentials with your API key.  
   - Operation: `google_maps`  
   - Query: Set to your desired search (default: "italian restaurants in austin tx").  
   - Additional Fields: Restrict JSON output to `local_results,place_results` to reduce data.  
   - Connect: Set Review Limit → Search Google Maps.

4. **Add If node to check single or multiple places**  
   - Name: If - Local or Place Results  
   - Condition: Check if `place_results` exists in JSON (object exists).  
   - If true: Single place, route to Initialize Vars.  
   - If false: Multiple places, route to Split Local Results.  
   - Connect: Search Google Maps → If - Local or Place Results.

5. **Add Split Out node to split multiple places**  
   - Name: Split Local Results  
   - Field to split out: `local_results`  
   - Connect: If - Local or Place Results (false output) → Split Local Results.

6. **Add Split In Batches node to loop over places**  
   - Name: Loop Over Places  
   - Batch size: 1 (default)  
   - Connect: Split Local Results → Loop Over Places.

7. **Add Code node to initialize variables**  
   - Name: Initialize Vars  
   - Script:  
     ```javascript
     const data = $getWorkflowStaticData('global');
     data.review_limit = $('Set Review Limit').first().json.review_limit;
     data.place_id = $input.first().json.place_id || $input.first().json.place_results?.place_id;
     data.place_name = $input.first().json.title || $input.first().json.place_results?.title;
     data.is_single_business = !!$input.first().json.place_results;
     data.num = null;
     data.count = 0;
     data.next_page_token = null;
     return data;
     ```
   - Connect:  
     - If - Local or Place Results (true output) → Initialize Vars  
     - Loop Over Places → Initialize Vars

8. **Add SerpApi node to get reviews**  
   - Name: Get Reviews  
   - Credentials: SerpApi credentials  
   - Operation: `google_maps_reviews`  
   - Parameters:  
     - `place_id`: `={{ $json.place_id }}` (from initialized vars)  
     - `num`: `={{ $json.num }}` (pagination count)  
     - `next_page_token`: `={{ $json.next_page_token }}`  
   - Connect: Initialize Vars → Get Reviews  
     & Set num for Pagination → Get Reviews (for subsequent pages)

9. **Add If node to check if reviews exist**  
   - Name: If Reviews Present  
   - Condition: `{{ !!$json.reviews || $('Initialize Vars').item.json.is_single_business }}` (boolean true)  
   - True: Continue to Split Reviews  
   - False: Loop Over Places (next place)  
   - Connect: Get Reviews → If Reviews Present

10. **Add Split Out node to split reviews**  
    - Name: Split Reviews  
    - Field to split out: `reviews`  
    - Include additional field: `place_info.title`  
    - Connect: If Reviews Present (true) → Split Reviews

11. **Add Google Sheets node to append reviews**  
    - Name: Append Reviews  
    - Credentials: Google Sheets OAuth2 credentials  
    - Operation: Append  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: Your target sheet  
    - Columns mapping (all as expressions):  
      - `place_name`: `={{ $('Initialize Vars').first().json.place_name }}`  
      - `iso_date`: `={{ $json.reviews.iso_date }}`  
      - `rating`: `={{ $json.reviews.rating }}`  
      - `snippet`: `={{ $json.reviews.extracted_snippet.original }}`  
    - Connect: Split Reviews → Append Reviews

12. **Add Code node to update review count and pagination token**  
    - Name: Update Review Count & Next Page Token  
    - Script:  
      ```javascript
      const data = $getWorkflowStaticData('global');
      data.count += $('Get Reviews').first().json.reviews.length;
      if ($('Get Reviews').first().json.serpapi_pagination) {
        data.next_page_token = $('Get Reviews').first().json.serpapi_pagination.next_page_token;
      }
      return data;
      ```
    - Connect: Append Reviews → Update Review Count & Next Page Token

13. **Add Switch node to route next step**  
    - Name: Route Next Step  
    - Rules:  
      - Output "Get More Pages":  
        Condition: `!!$json.next_page_token && ($json.review_limit > $json.count + 20)`  
      - Output "End Workflow":  
        Condition: `$json.is_single_business && (!$json.next_page_token || $json.review_limit < $json.count + 20)`  
      - Output "Get Next Place":  
        Condition: `!$json.is_single_business && (!$json.next_page_token || $json.review_limit < $json.count + 20)`  
    - Connect: Update Review Count & Next Page Token → Route Next Step

14. **Add Code node to set num for pagination**  
    - Name: Set num for Pagination  
    - Script:  
      ```javascript
      const data = $getWorkflowStaticData('global');
      data.num = 20;
      return data;
      ```
    - Connect: Route Next Step (Get More Pages output) → Set num for Pagination

15. **Connect workflow loops:**  
    - Set num for Pagination → Get Reviews  
    - Route Next Step (Get Next Place output) → Loop Over Places (to process next place)  
    - Route Next Step (End Workflow output) → No connection (workflow ends)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow uses SerpApi's Google Maps API to perform place searches and Google Maps Reviews API to fetch reviews. Each API call consumes search credits on SerpApi.                                                               | https://serpapi.com/google-maps-api <br> https://serpapi.com/google-maps-reviews-api              |
| Create a Google Sheet with the following headers: `place_name`, `iso_date`, `rating`, `snippet` before running the workflow.                                                                                                       | Google Sheets setup                                                                              |
| Update the "Search Google Maps" node query field to customize the places searched.                                                                                                                                                   | Node configuration                                                                                |
| The workflow can only paginate reviews, not the initial places search results. The place search returns a maximum of 20 results.                                                                                                    | Workflow limitation                                                                              |
| Be cautious about SerpApi search credits; fetching many places and many reviews can consume large amounts of credits.                                                                                                              | Usage warning                                                                                     |
| Helpful blog post with n8n and SerpApi integration details: [Boost your n8n workflows with SerpApi’s verified node](https://serpapi.com/blog/boost-your-n8n-workflows-with-serpapis-verified-node/)                                  | https://serpapi.com/blog/boost-your-n8n-workflows-with-serpapis-verified-node/                    |
| Google Sheets integration guide for n8n can be found here: https://n8n.io/integrations/google-sheets/                                                                                                                              | https://n8n.io/integrations/google-sheets/                                                      |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.