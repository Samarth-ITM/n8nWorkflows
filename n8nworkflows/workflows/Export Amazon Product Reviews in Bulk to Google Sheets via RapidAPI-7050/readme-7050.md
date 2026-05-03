Export Amazon Product Reviews in Bulk to Google Sheets via RapidAPI

https://n8nworkflows.xyz/workflows/export-amazon-product-reviews-in-bulk-to-google-sheets-via-rapidapi-7050


# Export Amazon Product Reviews in Bulk to Google Sheets via RapidAPI

### 1. Workflow Overview

This workflow automates the bulk export of Amazon product reviews into a Google Sheets spreadsheet by leveraging the RapidAPI “Real-Time Amazon Data” API. It is designed for users who want to systematically scrape and analyze customer feedback for a specific product identified by its ASIN on Amazon. The workflow handles large data volumes by batching API calls across multiple pages and star rating filters, then appends the collected review data into a designated Google Sheet tab.

**Logical Blocks:**

- **1.1 Input Reception:** Triggered by a form submission containing product details and the target Google Sheets tab URL.
- **1.2 Parameter Preparation:** Extracts and sets constants from the form input, then builds multiple API parameter sets covering different star ratings, sorting orders, and pages.
- **1.3 API Request Loop:** Splits the parameter sets into batches and sequentially calls the Amazon reviews API with controlled pacing to respect rate limits.
- **1.4 Response Processing:** Checks API response status and review presence, splits review arrays, and appends individual reviews into the Google Sheet.
- **1.5 Rate Limiting and Error Handling:** Implements wait nodes between calls and branches to handle empty or error responses gracefully.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives user input via a web form submission, collecting product and spreadsheet details.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point; captures product brand, name, Amazon URL, and Google Sheets tab URL.  
    - Configuration: Form fields for Brand name, Product/Model Name, Amazon Product URL, and Tab URL to insert reviews; all required.  
    - Input/Output: No input; outputs JSON with form fields and submission timestamp.  
    - Edge cases: Invalid or malformed URLs entered by user may cause later failures in parsing or data storage.

#### 2.2 Parameter Preparation

- **Overview:**  
Parses the ASIN from the Amazon URL and sets constant parameters. Constructs comprehensive sets of API query parameters covering star ratings, sorting, and pagination.

- **Nodes Involved:**  
  - Set Constants  
  - Build Parameter Sets

- **Node Details:**

  - **Set Constants**  
    - Type: Set  
    - Role: Extracts ASIN from the Amazon URL using regex; sets fixed parameters like country (US), brand and product names, and submission timestamp.  
    - Configuration: Uses JavaScript expression to extract ASIN from URL string.  
    - Input: JSON from form submission.  
    - Output: JSON with fields: asin, country, brand_name, product_name, last_scraped.  
    - Edge cases: If Amazon URL format changes or is invalid, ASIN extraction will fail or throw error.

  - **Build Parameter Sets**  
    - Type: Code (JavaScript)  
    - Role: Generates an array of parameter objects for API calls: 50 calls for most recent reviews (5 star ratings × 10 pages), 20 calls for top reviews (1 and 5 stars × 10 pages).  
    - Configuration: Loops over star ratings and pages; creates objects with keys: asin, country, star_rating, sort_by, and page.  
    - Input: Output from Set Constants.  
    - Output: JSON array of parameter sets for API requests.  
    - Edge cases: Errors in code might cause incomplete or malformed parameter sets.

#### 2.3 API Request Loop

- **Overview:**  
Executes API calls in batches to retrieve review data; incorporates delays to avoid hitting rate limits.

- **Nodes Involved:**  
  - Loop Over Items (splitInBatches)  
  - Wait1  
  - HTTP Request

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes the parameter sets in manageable batches to control API call flow.  
    - Configuration: Default batch size (not explicitly set).  
    - Input: Array of parameter sets.  
    - Output: One batch of parameter sets per execution cycle.  
    - Output connections: Two outputs; primary for batch processing, secondary triggers a wait node.  
    - Edge cases: Large batch sizes may cause timeouts or quota exhaustion.

  - **Wait1**  
    - Type: Wait  
    - Role: Delays execution (3.7 seconds) between batches to throttle API calls.  
    - Input: Triggered after each batch is processed.  
    - Output: Moves execution to the next API request.  
    - Edge cases: Delay duration may need adjustment based on API rate limits.

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Calls RapidAPI’s Amazon product reviews endpoint with parameters from batch.  
    - Configuration:  
      - URL: https://real-time-amazon-data.p.rapidapi.com/product-reviews  
      - Query params dynamically set from batch item JSON (asin, country, page, star_rating, sort_by).  
      - Fixed query parameters: verified_purchases_only=true, images_or_videos_only=false, current_format_only=false.  
      - Headers include x-rapidapi-host.  
      - Authentication: HTTP Header Auth using stored RapidAPI key credentials.  
      - Retry on failure enabled.  
      - On error: Continue regular output to allow workflow to handle errors downstream.  
    - Input: Parameter set JSON from batch.  
    - Output: JSON response from API.  
    - Edge cases: Possible errors include quota exceeded (HTTP 429), network timeouts, invalid parameters, or malformed responses.

#### 2.4 Response Processing

- **Overview:**  
Validates API response success and presence of reviews, then splits reviews for individual processing and storage.

- **Nodes Involved:**  
  - If status "OK" and contains reviews  
  - Split Out Reviews  
  - Store reviews  
  - Wait2

- **Node Details:**

  - **If status "OK" and contains reviews**  
    - Type: If (conditional)  
    - Role: Checks if the API response status is "OK" and the reviews array is not empty.  
    - Configuration:  
      - Condition 1: `$json.status === 'OK'` (boolean true)  
      - Condition 2: `$json.data.reviews` array is not empty.  
      - Both conditions must be true (AND).  
    - Input: API response JSON.  
    - Output:  
      - True branch: proceed to split reviews.  
      - False branch: triggers Wait2 node as delay before next batch (60 seconds).  
    - Edge cases: Empty or error responses cause false branch; could lead to silent skips or delays.

  - **Split Out Reviews**  
    - Type: SplitOut  
    - Role: Separates the reviews array into individual review JSON objects for subsequent appending.  
    - Configuration: Splits on the field `data.reviews`.  
    - Input: API response JSON passing the If node’s true branch.  
    - Output: One review JSON per output item.  
    - Edge cases: Empty or malformed reviews array leads to no output.

  - **Store reviews**  
    - Type: Google Sheets  
    - Role: Appends or updates the individual review records into the specified Google Sheets tab.  
    - Configuration:  
      - Operation: appendOrUpdate  
      - Sheet name: dynamically set from form submission field "Tab URL to insert reviews" (URL mode).  
      - Document ID: fixed Google Sheets document ID (Amazon Reviews Database).  
      - Columns mapping: defined below data rows (dynamic).  
      - Credential: Google Sheets OAuth2.  
    - Input: Individual review JSON from Split Out Reviews.  
    - Output: Success triggers next batch processing.  
    - Edge cases:  
      - Incorrect sheet/tab URL causes data not to append correctly.  
      - OAuth token expiration or permission issues.  
      - Data format mismatches leading to append failures.

  - **Wait2**  
    - Type: Wait  
    - Role: Imposes longer delay (60 seconds) when no valid reviews are found or errors occur, before continuing.  
    - Input: False branch of If node.  
    - Output: Loops back to batch processing node.  
    - Edge cases: Excessive wait time may slow workflow unnecessarily if frequent empty responses occur.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                            | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                  |
|-------------------------------|------------------------|--------------------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger           | Receives product and sheet input            | -                            | Set Constants                | When filling in the form, ensure the Tab URL matches exactly the Google Sheet tab URL to insert reviews.                      |
| Set Constants                 | Set                    | Extracts ASIN and sets fixed parameters      | On form submission           | Build Parameter Sets         | -                                                                                                                            |
| Build Parameter Sets          | Code                   | Creates multiple API query parameter sets    | Set Constants                | Loop Over Items              | -                                                                                                                            |
| Loop Over Items               | SplitInBatches         | Processes parameter sets in batches           | Build Parameter Sets, Store reviews | HTTP Request, Wait1        | -                                                                                                                            |
| Wait1                        | Wait                   | Delays between API calls                       | Loop Over Items (secondary)  | HTTP Request                | -                                                                                                                            |
| HTTP Request                 | HTTP Request           | Calls Amazon reviews API with parameters       | Loop Over Items (primary), Wait1 | If status "OK" and contains reviews | To get your RapidAPI key, see https://rapidapi.com/letscrape-6bRBa3QguO5/api/real-time-amazon-data/playground/endpoint_1e42cec2-07bd-49bf-bde6-563c03f27bb3 |
| If status "OK" and contains reviews | If                    | Checks API response status and review presence | HTTP Request                | Split Out Reviews (true), Wait2 (false) | Known failure causes: insufficient reviews or quota exceeded (Error 429). See note for details.                               |
| Split Out Reviews            | SplitOut                | Splits reviews array into individual items     | If status "OK" and contains reviews | Store reviews               | -                                                                                                                            |
| Store reviews                | Google Sheets           | Appends individual reviews to Google Sheets    | Split Out Reviews            | Loop Over Items             | -                                                                                                                            |
| Wait2                       | Wait                   | Waits when no valid reviews or errors occur    | If status "OK" and contains reviews (false) | Loop Over Items             | -                                                                                                                            |
| Sticky Note                 | Sticky Note             | Provides detailed user manual and troubleshooting info | -                            | -                           | See detailed user manual and instructions in the content for usage, API quota, and error handling.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named `On form submission`:  
   - Form title: "Amazon Product Reviews Scraper"  
   - Fields (all required):  
     - Brand name (placeholder: "MANSCAPED")  
     - Product / Model Name (placeholder: "The Lawn Mower® 3.0 Plus")  
     - Amazon Product URL (placeholder: example URL)  
     - Tab URL to insert reviews (placeholder: Google Sheets tab URL)  
   - This node acts as the workflow entry point.

2. **Add a Set node** named `Set Constants`:  
   - Input: from `On form submission` node.  
   - Assign variables:  
     - `asin` (string): extract ASIN from Amazon URL using expression: `{{$json["Amazon Product URL"].match(/\/dp\/([A-Z0-9]{10})/)[1]}}`  
     - `country` (string): `"US"`  
     - `brand_name` (string): `{{$json["Brand name"]}}`  
     - `product_name` (string): `{{$json["Product / Model Name"]}}`  
     - `last_scraped` (string): `{{$json.submittedAt}}`

3. **Add a Code node** named `Build Parameter Sets`:  
   - Input: from `Set Constants`.  
   - JavaScript code:  
     ```javascript
     const asin = $input.first().json.asin;
     const country = $input.first().json.country;
     const items = [];
     // Core harvest: MOST RECENT reviews for 1-5 stars, pages 1-10
     ['1_STARS','2_STARS','3_STARS','4_STARS','5_STARS'].forEach(rating => {
       for (let page=1; page<=10; page++) {
         items.push({ asin, country, star_rating: rating, sort_by: 'MOST_RECENT', page });
       }
     });
     // Spotlight extremes: TOP REVIEWS for 1 and 5 stars, pages 1-10
     ['1_STARS','5_STARS'].forEach(rating => {
       for (let page=1; page<=10; page++) {
         items.push({ asin, country, star_rating: rating, sort_by: 'TOP_REVIEWS', page });
       }
     });
     return items.map(i => ({ json: i }));
     ```

4. **Add a SplitInBatches node** named `Loop Over Items`:  
   - Input: from `Build Parameter Sets`.  
   - Use default batch size or adjust as needed for API limits.  
   - Outputs:  
     - Primary output (batch items) connects to `HTTP Request`.  
     - Secondary output connects to `Wait1`.

5. **Add a Wait node** named `Wait1`:  
   - Input: from secondary output of `Loop Over Items`.  
   - Duration: 3.7 seconds.  
   - Output: connects to `HTTP Request`.

6. **Add an HTTP Request node** named `HTTP Request`:  
   - Input: from primary output of `Loop Over Items` and from `Wait1`.  
   - Request settings:  
     - URL: `https://real-time-amazon-data.p.rapidapi.com/product-reviews`  
     - Method: GET  
     - Query parameters (dynamic):  
       - country: `{{$json.country}}`  
       - asin: `{{$json.asin}}`  
       - page: `{{$json.page}}`  
       - sort_by: `{{$json.sort_by}}`  
       - star_rating: `{{$json.star_rating}}`  
       - verified_purchases_only: `true`  
       - images_or_videos_only: `false`  
       - current_format_only: `false`  
     - Headers:  
       - `x-rapidapi-host`: `real-time-amazon-data.p.rapidapi.com`  
     - Authentication: HTTP Header Auth with RapidAPI key credential  
     - Retry on Fail: Enabled  
     - Error handling: Continue regular output  
   - Output: connects to `If status "OK" and contains reviews`.

7. **Add an If node** named `If status "OK" and contains reviews`:  
   - Input: from `HTTP Request`.  
   - Conditions (AND):  
     - Expression: `{{$json.status === "OK"}}` → true  
     - Expression: `{{$json.data.reviews && $json.data.reviews.length > 0}}` → not empty array  
   - True output connects to `Split Out Reviews`.  
   - False output connects to `Wait2`.

8. **Add a SplitOut node** named `Split Out Reviews`:  
   - Input: from If node (true branch).  
   - Field to split out: `data.reviews`.  
   - Output: individual review JSON objects connected to `Store reviews`.

9. **Add a Google Sheets node** named `Store reviews`:  
   - Input: from `Split Out Reviews`.  
   - Operation: `appendOrUpdate`  
   - Document ID: fixed ID of your reviews spreadsheet, e.g., `1XsPm7V9cf9jufJ6flUXTVO2Kzaj_KOyl1_h7AZTomCM`.  
   - Sheet Name: dynamic via expression from form submission field `"Tab URL to insert reviews"`. Use URL mode with expression: `={{ $('On form submission').item.json["Tab URL to insert reviews"] }}`  
   - Columns: define dynamically below the data row.  
   - Credential: Google Sheets OAuth2 with appropriate permissions.  
   - Output: connects back to `Loop Over Items` to process next batch.

10. **Add a Wait node** named `Wait2`:  
    - Input: from If node (false branch).  
    - Duration: 60 seconds.  
    - Output: connects back to `Loop Over Items` to continue processing.

11. **Add a Sticky Note node** at the beginning of the workflow for user manual and troubleshooting instructions (optional but recommended).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                                                                                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When filling in the form, ensure that the Tab URL entered matches the exact URL of the corresponding tab in the Google Sheet to populate. Mismatches will cause data not to be stored correctly.                                                                                                                                                                                                                                                                                                                                              | Form submission instructions.                                                                                                                                                                                                                                       |
| The workflow performs 70 API calls per product: 50 calls retrieving the 10 most recent pages for each star rating (1-5 stars), and 20 calls retrieving top reviews for 1-star and 5-star ratings only.                                                                                                                                                                                                                                                                                                                                          | Workflow API call strategy.                                                                                                                                                                                                                                         |
| To obtain your RapidAPI key: go to the RapidAPI Playground for the Real-Time Amazon Data API, log in, test the endpoint to auto-generate the key, copy the `X-RapidAPI-Key` from the code snippet tab, and paste it into your n8n HTTP Request credentials under HTTP Header Auth.                                                                                                                                                                                                                                                           | https://rapidapi.com/letscrape-6bRBa3QguO5/api/real-time-amazon-data/playground/endpoint_1e42cec2-07bd-49bf-bde6-563c03f27bb3                                                                                                                                          |
| Known failure cases include: 1) insufficient reviews available (less than expected pages), causing the conditional node to skip processing, and 2) exceeding RapidAPI quota limits resulting in HTTP 429 errors. Check RapidAPI dashboard and logs for quota status.                                                                                                                                                                                                                                                                             | Error handling and troubleshooting advice.                                                                                                                                                                                                                          |
| The Google Sheets node requires OAuth2 credentials with write permissions to the target spreadsheet. Dynamic sheet tab selection depends on correct URL parsing from form input.                                                                                                                                                                                                                                                                                                                                                              | Google Sheets integration notes.                                                                                                                                                                                                                                    |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.