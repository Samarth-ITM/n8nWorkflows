Trustpilot Insights Scraper: Auto Reviews via Bright Data + Google Sheets Sync

https://n8nworkflows.xyz/workflows/trustpilot-insights-scraper--auto-reviews-via-bright-data---google-sheets-sync-5145


# Trustpilot Insights Scraper: Auto Reviews via Bright Data + Google Sheets Sync

### 1. Workflow Overview

This workflow automates the extraction of Trustpilot reviews using Bright Data’s scraping service and appends the retrieved data into a Google Sheet. It is designed for users who want to periodically collect and analyze customer reviews from Trustpilot without manual scraping. The workflow is structured into several logical blocks:

- **1.1 Input Reception:** Captures the Trustpilot URL input from a user via a form trigger.
- **1.2 Scraping Trigger:** Sends a request to Bright Data's API to initiate a scraping dataset for the given Trustpilot URL.
- **1.3 Snapshot Progress Monitoring:** Periodically polls Bright Data to check if the scraping snapshot is ready.
- **1.4 Data Retrieval:** When ready, downloads the snapshot of scraped reviews.
- **1.5 Data Sync:** Appends the extracted reviews data into a specified Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This initial block receives the Trustpilot website URL from the user through a manual input form, triggering the workflow execution.

- **Nodes Involved:**  
  - `Form Trigger`

- **Node Details:**  
  - **Form Trigger**  
    - Type: Form Trigger  
    - Role: Entry point node capturing user input.  
    - Configuration: Single form field labeled `Trustpilot Website URL`.  
    - Key Variables: `$json['Trustpilot Website URL']` used downstream to specify the target URL for scraping.  
    - Input: None (trigger node).  
    - Output: JSON containing the user input URL.  
    - Potential Failures: Missing input or invalid URL format (workflow does not explicitly validate URL format).  

#### 1.2 Scraping Trigger

- **Overview:**  
  This block sends a POST request to Bright Data's API to trigger scraping of the specified Trustpilot URL using a pre-configured dataset ID.

- **Nodes Involved:**  
  - `POST to Bright Data`

- **Node Details:**  
  - **POST to Bright Data**  
    - Type: HTTP Request (POST)  
    - Role: Triggers Bright Data scraping job.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Body: JSON including the input URL and a list of custom output fields to be retrieved from the scrape.  
      - Query Parameters: dataset_id set to `gd_lm5zmhwd2sni130p`, include_errors=true, limit_multiple_results=2  
      - Header: Authorization bearer token placeholder `BRIGHT_DATA_API_KEY` to be replaced with actual API key credentials.  
    - Key Expressions: `{{ $json['Trustpilot Website URL'] }}` dynamically injected into the body.  
    - Input: Triggered from Form Trigger output.  
    - Output: JSON including a `snapshot_id` representing the scraping job instance.  
    - Potential Failures: HTTP errors (auth failure, rate limiting), incorrect dataset ID, malformed request body.  
    - Credential Requirements: Valid Bright Data API key with dataset access.  

#### 1.3 Snapshot Progress Monitoring

- **Overview:**  
  This block periodically checks if the Bright Data scraping snapshot is ready by polling the dataset progress endpoint. If not ready, it waits and retries.

- **Nodes Involved:**  
  - `GET - snapshot status`  
  - `IF snapshot_id not ready than wait`  
  - `Wait_1_mins`  
  - `If`

- **Node Details:**  
  - **GET - snapshot status**  
    - Type: HTTP Request (GET)  
    - Role: Checks the status of the scraping snapshot using the `snapshot_id`.  
    - Configuration:  
      - URL template: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
      - Header: Authorization with Bright Data API key.  
    - Input: From the output of `POST to Bright Data` or from `Wait_1_mins`.  
    - Output: JSON containing the `status` field, expected to be `"ready"` when scraping completes.  
    - Potential Failures: HTTP errors, invalid snapshot_id, network timeouts.  

  - **IF snapshot_id not ready than wait**  
    - Type: IF  
    - Role: Conditional check on snapshot status.  
    - Condition: Checks if `$json.status` equals `"ready"`.  
    - Input: Output of `GET - snapshot status`.  
    - Output:  
      - If true (ready): proceeds to `If` node.  
      - If false (not ready): proceeds to `Wait_1_mins`.  
    - Potential Failures: Expression evaluation errors if snapshot_id or status fields are missing.  

  - **Wait_1_mins**  
    - Type: Wait  
    - Role: Pauses workflow for 1 minute before retrying status check.  
    - Input: From `IF snapshot_id not ready than wait` (false branch).  
    - Output: After 1 minute, triggers `GET - snapshot status` again.  
    - Potential Failures: None significant, but prolonged waits if snapshot never completes.  

  - **If**  
    - Type: IF  
    - Role: Additional check on whether the retrieved snapshot contains records (non-empty).  
    - Condition: Checks if `$json.records` is not equal to 0.  
    - Input: Output of `IF snapshot_id not ready than wait` (true branch).  
    - Output: If records exist, proceeds to download snapshot; else, stops or skips.  
    - Potential Failures: Missing `records` field or empty data sets.

#### 1.4 Data Retrieval

- **Overview:**  
  When the snapshot is ready and contains data, this block downloads the complete scraped dataset.

- **Nodes Involved:**  
  - `GET - snapshot download`

- **Node Details:**  
  - **GET - snapshot download**  
    - Type: HTTP Request (GET)  
    - Role: Fetches the full snapshot JSON data using the `snapshot_id`.  
    - Configuration:  
      - URL template: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
      - Header: Authorization with Bright Data API key.  
      - Query: format=json  
    - Input: Output of `If` node confirming data presence.  
    - Output: JSON array of reviews with detailed fields.  
    - Potential Failures: HTTP errors, invalid snapshot_id, data corruption.

#### 1.5 Data Sync

- **Overview:**  
  This final block appends the downloaded review data into a designated Google Sheet for further analysis or storage.

- **Nodes Involved:**  
  - `Google Sheets (Append)`

- **Node Details:**  
  - **Google Sheets (Append)**  
    - Type: Google Sheets Node (Append Operation)  
    - Role: Inserts the scraped review data rows into the specified Google Sheet tab.  
    - Configuration:  
      - Document ID: Google Sheet document identified by ID `1yQ10Q2qSjm-hhafHF2sXu-hohurW5_KD8fIv4IXEA3I`  
      - Sheet Name: `Trustpilot` (tab within the sheet)  
      - Columns: Auto-mapped fields including `company_name`, `review_rating`, `review_title`, `review_content`, `review_date`, `reviewer_name`, `company_location`, `company_overall_rating`, and many more (total fields defined in the node schema).  
      - Credential: Google Sheets OAuth2 connected account with edit permissions.  
    - Input: JSON data from `GET - snapshot download`.  
    - Output: Confirmation of appended rows.  
    - Potential Failures: Credential expiration, API rate limits, schema mismatches, exceeding Google Sheets row limits.  

---

### 3. Summary Table

| Node Name                         | Node Type             | Functional Role                              | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                      |
|----------------------------------|-----------------------|----------------------------------------------|------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Form Trigger                     | Form Trigger          | User input capture for Trustpilot URL        | None                         | POST to Bright Data             | 1. 📝 **Form Trigger Node** - Accepts manual input from the user. Field: `Trustpilot Website URL` |
| POST to Bright Data              | HTTP Request (POST)   | Trigger Bright Data scraping for Trustpilot URL | Form Trigger                 | GET - snapshot status           | 2. 🌐 **HTTP Request (Trigger Scraping on Bright Data)** - Sends a POST to Bright Data’s API.    |
| GET - snapshot status            | HTTP Request (GET)    | Polls Bright Data for snapshot readiness     | POST to Bright Data, Wait_1_mins | IF snapshot_id not ready than wait | 3. ⌛ **Snapshot Progress Check** - Checks if snapshot is ready.                                 |
| IF snapshot_id not ready than wait | IF                    | Checks if snapshot status is "ready"          | GET - snapshot status         | If (ready), Wait_1_mins (not ready) | 5. ✅ **IF Node (Check Completion)** - Proceeds if ready, else waits.                           |
| Wait_1_mins                     | Wait                  | Waits 1 minute before retrying status check  | IF snapshot_id not ready than wait (false branch) | GET - snapshot status           | 4. 🕒 **Wait Node** - Pauses 1 minute before re-checking.                                       |
| If                             | IF                    | Checks if snapshot contains records (non-zero) | IF snapshot_id not ready than wait (true branch) | GET - snapshot download         | 5. ✅ **IF Node (Check Completion)** - Proceeds if records exist.                               |
| GET - snapshot download          | HTTP Request (GET)    | Downloads the snapshot data                   | If                           | Google Sheets (Append)          | 6. 📥 **Download Final Snapshot** - Pulls full review data using snapshot ID.                   |
| Google Sheets (Append)           | Google Sheets Append  | Appends scraped reviews to Google Sheet      | GET - snapshot download       | None                           | 7. 📊 **Google Sheets (Append)** - Appends review data to sheet `Trustpilot`.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Add a `Form Trigger` node.  
   - Set `Form Title` to "Website URL".  
   - Add one form field: Label it `Trustpilot Website URL`.  
   - This node acts as the starting point to accept user input.

2. **Create POST to Bright Data Node**  
   - Add an `HTTP Request` node configured for POST request.  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Method: POST  
   - Headers: Add `Authorization` header with value `Bearer <Your Bright Data API Key>`.  
   - Query Parameters:  
     - `dataset_id`: `gd_lm5zmhwd2sni130p`  
     - `include_errors`: `true`  
     - `limit_multiple_results`: `2`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "input": [
         {
           "url": "{{ $json['Trustpilot Website URL'] }}"
         }
       ],
       "custom_output_fields": [
         "company_name",
         "review_id",
         "review_date",
         "review_rating",
         "review_title",
         "review_content",
         "is_verified_review",
         "review_date_of_experience",
         "reviewer_location",
         "reviews_posted_overall",
         "review_replies",
         "review_useful_count",
         "reviewer_name",
         "company_logo",
         "url",
         "company_rating_name",
         "company_overall_rating",
         "is_verified_company",
         "company_total_reviews",
         "5_star",
         "4_star",
         "3_star",
         "2_star",
         "1_star",
         "company_about",
         "company_email",
         "company_phone",
         "company_location",
         "company_country",
         "breadcrumbs",
         "company_category",
         "company_id",
         "company_website",
         "company_other_categories",
         "review_url",
         "date_posted"
       ]
     }
     ```
   - Connect the `Form Trigger` node output to this node input.

3. **Create GET - snapshot status Node**  
   - Add an `HTTP Request` node configured for GET.  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Headers: `Authorization: Bearer <Your Bright Data API Key>`  
   - Query Parameters: `format=json`  
   - Connect output of `POST to Bright Data` to this node input.  

4. **Create IF snapshot_id not ready than wait Node**  
   - Add an `IF` node.  
   - Condition: Check if `$json.status` equals `"ready"`.  
   - Connect `GET - snapshot status` output to this node.  

5. **Create Wait_1_mins Node**  
   - Add a `Wait` node.  
   - Set to wait for 1 minute.  
   - Connect the false branch (not ready) of the IF node to this node.  

6. **Loop back from Wait to GET - snapshot status**  
   - Connect the output of `Wait_1_mins` back to `GET - snapshot status` node to re-check snapshot status.  

7. **Create If Node to check records**  
   - Add another `IF` node.  
   - Condition: Check if `$json.records` is not equal to `0`.  
   - Connect the true branch (ready) of the previous IF node to this node.  

8. **Create GET - snapshot download Node**  
   - Add an `HTTP Request` node configured for GET.  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Headers: `Authorization: Bearer <Your Bright Data API Key>`  
   - Query Parameters: `format=json`  
   - Connect the true branch of the `If` node (records exist) to this node.  

9. **Create Google Sheets (Append) Node**  
   - Add a `Google Sheets` node.  
   - Operation: Append  
   - Credentials: Connect a valid Google Sheets OAuth2 credential with edit access to the target Sheet.  
   - Document ID: `1yQ10Q2qSjm-hhafHF2sXu-hohurW5_KD8fIv4IXEA3I`  
   - Sheet Name: `Trustpilot` tab  
   - Columns: Enable auto-mapping of input fields to columns. Include all fields as defined in the original node (e.g., `company_name`, `review_rating`, etc.)  
   - Connect the output of `GET - snapshot download` to this node.  

10. **Validate all connections** to ensure the flow:  
    Form Trigger → POST to Bright Data → GET - snapshot status → IF snapshot_id not ready than wait  
    - IF true → If (records check) → GET - snapshot download → Google Sheets (Append)  
    - IF false → Wait_1_mins → loop back to GET - snapshot status  

11. **Set credentials**  
    - Bright Data API key must be stored securely and referenced in HTTP request headers.  
    - Google Sheets OAuth2 credential must be authorized for the target spreadsheet.  

12. **Activate and test the workflow** with a valid Trustpilot URL input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow relies on Bright Data’s Dataset API for scraping Trustpilot reviews. Requires valid Bright Data API key and dataset ID. | Official Bright Data API docs: https://brightdata.com/api/datasets                              |
| Google Sheets node uses OAuth2 credentials; ensure proper permissions for editing the target spreadsheet.                        | Google Sheets API docs: https://developers.google.com/sheets/api                                |
| The workflow includes wait-and-retry logic to handle asynchronous scraping snapshot readiness.                                   | This pattern avoids unnecessary API calls and respects rate limits.                            |
| Input validation on the Trustpilot URL is not implemented; recommended to add URL format validation for robustness.             | Could be implemented via a Function node or external validator before triggering scraping.     |
| Sticky notes in the workflow visually document each major step for easier maintenance and understanding in the n8n editor.      | Useful for collaborative environments and onboarding new users to the workflow logic.          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.