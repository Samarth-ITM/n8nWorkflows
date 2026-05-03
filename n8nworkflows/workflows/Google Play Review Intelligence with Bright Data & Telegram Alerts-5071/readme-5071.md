Google Play Review Intelligence with Bright Data & Telegram Alerts

https://n8nworkflows.xyz/workflows/google-play-review-intelligence-with-bright-data---telegram-alerts-5071


# Google Play Review Intelligence with Bright Data & Telegram Alerts

### 1. Workflow Overview

This workflow automates the process of scraping Google Play Store app reviews using the Bright Data API, storing the extracted data in a Google Sheet, and sending Telegram alerts for apps with low ratings. It is designed for app developers, marketers, and analysts who want to monitor app performance and user sentiment efficiently.

The workflow is organized into these logical blocks:

- **1.1 Input Reception:** Receives user input via a form with the Google Play app URL and number of reviews to fetch.
- **1.2 Scraping Request Initiation:** Sends a request to Bright Data API to start scraping app data and reviews.
- **1.3 Polling for Scrape Status:** Periodically checks the status of the scraping job until completion.
- **1.4 Data Retrieval and Storage:** Fetches the completed dataset and appends relevant data into a Google Sheet.
- **1.5 Review Analysis and Alerting:** Filters reviews with ratings below 4 and sends Telegram alerts for low-performing apps.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures user inputs from a web form to specify the target Google Play Store URL and number of reviews to scrape.
- **Nodes Involved:**  
  - ✅ Trigger Input Form  
  - Sticky Note4 (contextual description)

- **Node Details:**  
  - **✅ Trigger Input Form**  
    - Type: Form Trigger  
    - Role: Entry point for the workflow, triggering execution upon form submission.  
    - Configuration:  
      - Form titled "Google Play Store".  
      - Fields: "URL" (text), "No of review" (number).  
    - Inputs: User submits form data.  
    - Outputs: JSON containing URL and number of reviews.  
    - Edge Cases: Missing or malformed URL/number inputs could cause API errors downstream. Validation is assumed to be handled outside n8n or via form constraints.  

#### 2.2 Scraping Request Initiation

- **Overview:** Sends a POST request to Bright Data API to initiate scraping of Google Play app reviews and metadata based on user input.
- **Nodes Involved:**  
  - 🚀 Start Scraping Request  
  - Sticky Note3

- **Node Details:**  
  - **🚀 Start Scraping Request**  
    - Type: HTTP Request  
    - Role: Starts the scraping dataset job on Bright Data platform.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Body: JSON containing user input URL, number of reviews, and country "US".  
      - Custom output fields requested to get detailed app and review data.  
      - Query parameters include dataset ID and limiting multiple results.  
      - Authorization header with Bright Data API token.  
    - Inputs: Workflow JSON from form node.  
    - Outputs: JSON including a `snapshot_id` for the scraping job.  
    - Edge Cases: API authentication failures, malformed input data, API rate limits, or service outages.

#### 2.3 Polling for Scrape Status

- **Overview:** Polls the Bright Data API repeatedly every 45 seconds to check if the scraping job is complete.
- **Nodes Involved:**  
  - 🔄 Check Scrape Status  
  - ⏱️ Wait for Response 45 sec  
  - 🧩 Verify Completion  
  - Sticky Note, Sticky Note1, Sticky Note2

- **Node Details:**  
  - **🔄 Check Scrape Status**  
    - Type: HTTP Request  
    - Role: Queries Bright Data API using the `snapshot_id` to retrieve the current status of the scraping job.  
    - Configuration: GET request to endpoint with dynamic snapshot ID.  
    - Input: JSON including `snapshot_id` from the previous node.  
    - Output: JSON with status field, expected values like "ready" when done.  
    - Edge Cases: Missing snapshot ID, API failures, network issues.  
  - **⏱️ Wait for Response 45 sec**  
    - Type: Wait  
    - Role: Pauses workflow for 45 seconds before retrying status check to avoid API overload.  
    - Configuration: Fixed wait time of 45 seconds.  
    - Inputs and outputs: Passes data forward without modification.  
  - **🧩 Verify Completion**  
    - Type: If  
    - Role: Checks if scraping status equals "ready".  
    - Configuration: Condition `status == "ready"`.  
    - If true, proceeds to fetch final data; if false, loops back to check status again.  
    - Edge Cases: Unexpected status values, missing status, infinite loops if status never becomes "ready".  

#### 2.4 Data Retrieval and Storage

- **Overview:** Once the scraping job is complete, fetches the scraped dataset and appends relevant app and review data into a Google Sheet for persistent storage.
- **Nodes Involved:**  
  - 📥 Fetch Scraped Data  
  - 📊 Save to Google Sheet  
  - Sticky Note5, Sticky Note6

- **Node Details:**  
  - **📥 Fetch Scraped Data**  
    - Type: HTTP Request  
    - Role: Retrieves the completed scraping dataset as JSON from Bright Data using the snapshot ID.  
    - Configuration: GET request with snapshot ID and `format=json`.  
    - Inputs: JSON containing snapshot ID after verification node.  
    - Outputs: Array of reviews and app details.  
    - Edge Cases: Snapshot ID expiration, API errors, large data payloads leading to timeouts.  
  - **📊 Save to Google Sheet**  
    - Type: Google Sheets  
    - Role: Appends each review and app data row into a specified Google Sheet for analysis and record-keeping.  
    - Configuration:  
      - Operation: Append  
      - Document ID and Sheet Name configured to target Google Sheet.  
      - Columns mapped explicitly: URL, review, review_id, app_rating, app_country, review_date, app_what_new, review_rating, reviewer_name, app_number_of_reviews, etc.  
      - Credentials: Google Sheets OAuth2.  
    - Inputs: JSON data array from previous node.  
    - Outputs: Confirmation of appended rows.  
    - Edge Cases: Google Sheets API quota limits, invalid credentials, mapping mismatches, large batch sizes.  

#### 2.5 Review Analysis and Alerting

- **Overview:** Filters reviews with ratings below 4 stars and sends Telegram alerts to notify stakeholders about potential low app performance.
- **Nodes Involved:**  
  - ⚠️ Check Low Ratings  
  - 📣 Send Alert to Telegram  
  - Sticky Note7, Sticky Note8

- **Node Details:**  
  - **⚠️ Check Low Ratings**  
    - Type: If  
    - Role: Filters reviews where `review_rating` is less than 4.  
    - Configuration: Numeric comparison `review_rating < 4`.  
    - Inputs: Review data from Google Sheets append node.  
    - Outputs: Two branches — true (low rating) and false (high rating).  
    - Edge Cases: Non-numeric or missing ratings, false positives/negatives due to rating scale variations.  
  - **📣 Send Alert to Telegram**  
    - Type: Telegram  
    - Role: Sends formatted alert messages about low-rated apps via Telegram Bot.  
    - Configuration:  
      - Message includes app title, developer, rating, number of reviews, and Play Store URL.  
      - Chat ID and Telegram Bot credentials required.  
    - Inputs: Filtered low-rating reviews.  
    - Outputs: Telegram API confirmation.  
    - Edge Cases: Telegram API rate limits, invalid chat IDs, bot permission issues.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                           | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                  |
|---------------------------|---------------------|-----------------------------------------|----------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| ✅ Trigger Input Form      | Form Trigger        | Entry point: receives user input form   | -                          | 🚀 Start Scraping Request      | Triggers the workflow with user input — takes Google Play Store URL and number of reviews.   |
| 🚀 Start Scraping Request  | HTTP Request        | Initiates scraping job via Bright Data  | ✅ Trigger Input Form       | 🔄 Check Scrape Status         | Sends a POST request to Bright Data API to start scraping reviews and app details.          |
| 🔄 Check Scrape Status     | HTTP Request        | Polls scraping job status                | 🚀 Start Scraping Request   | ⏱️ Wait for Response 45 sec    | Checks the current processing status of the dataset request using the snapshot ID.          |
| ⏱️ Wait for Response 45 sec | Wait               | Waits 45 seconds before re-polling       | 🔄 Check Scrape Status      | 🧩 Verify Completion           | Pauses the workflow for 45 seconds before checking the status again — used for polling.     |
| 🧩 Verify Completion       | If                  | Checks if scraping job is complete       | ⏱️ Wait for Response 45 sec | 📥 Fetch Scraped Data / 🔄 Check Scrape Status | Evaluates if the dataset scraping process is complete (status: "ready"). Loops if not.       |
| 📥 Fetch Scraped Data      | HTTP Request        | Retrieves final scraped data             | 🧩 Verify Completion       | 📊 Save to Google Sheet / ⚠️ Check Low Ratings | Retrieves the final scraped data (reviews and app info) once the dataset is ready.           |
| 📊 Save to Google Sheet    | Google Sheets       | Stores scraped data in Google Sheets     | 📥 Fetch Scraped Data       | ⚠️ Check Low Ratings           | Appends selected app and review data into a connected Google Sheet for storage.             |
| ⚠️ Check Low Ratings       | If                  | Filters reviews with rating < 4           | 📊 Save to Google Sheet     | (none) / 📣 Send Alert to Telegram | Filters out reviews with rating below 4 to flag low-performing apps.                        |
| 📣 Send Alert to Telegram  | Telegram            | Sends alerts for low-rated apps           | ⚠️ Check Low Ratings        | -                             | Sends a Telegram alert if the app has low ratings, indicating poor performance.             |
| Sticky Note                | Sticky Note         | Evaluates scraping completion status     | -                          | -                             | Evaluates if the dataset scraping process is complete (status: "ready"). Loops if not.       |
| Sticky Note1               | Sticky Note         | Describes wait between status checks     | -                          | -                             | Pauses the workflow for 45 seconds before checking the status again — used for polling.     |
| Sticky Note2               | Sticky Note         | Describes status check                    | -                          | -                             | Checks the current processing status of the dataset request using the snapshot ID.          |
| Sticky Note3               | Sticky Note         | Describes scraping job initiation        | -                          | -                             | Sends a POST request to Bright Data API to start scraping reviews and app details.          |
| Sticky Note4               | Sticky Note         | Describes input form                      | -                          | -                             | Triggers the workflow with user input — takes Google Play Store URL and number of reviews.   |
| Sticky Note5               | Sticky Note         | Describes data retrieval                  | -                          | -                             | Retrieves the final scraped data (reviews and app info) once the dataset is ready.           |
| Sticky Note6               | Sticky Note         | Describes Google Sheets storage           | -                          | -                             | Appends selected app and review data into a connected Google Sheet for storage.             |
| Sticky Note7               | Sticky Note         | Describes low rating filtering             | -                          | -                             | Filters out reviews with rating below 4 to flag low-performing apps.                        |
| Sticky Note8               | Sticky Note         | Describes Telegram alert                   | -                          | -                             | Sends a Telegram alert if the app has low ratings, indicating poor performance.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Input Form Node**  
   - Type: Form Trigger  
   - Configure a new form titled "Google Play Store".  
   - Add two fields:  
     - "URL" (text input)  
     - "No of review" (number input)  
   - Purpose: Collect user input URL and number of reviews to fetch.

2. **Create HTTP Request Node "Start Scraping Request"**  
   - Connect from the Trigger Input Form node.  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Body (JSON):  
     ```json
     {
       "input": [
         {
           "url": "={{ $json.URL }}",
           "num_of_reviews": {{ $json["No of review"] }},
           "country": "US"
         }
       ],
       "custom_output_fields": [
         "url",
         "review_id",
         "reviewer_name",
         "review_date",
         "review_rating",
         "review",
         "app_url",
         "app_title",
         "app_developer",
         "app_images",
         "app_rating",
         "app_number_of_reviews",
         "app_what_new",
         "app_content_rating",
         "app_country",
         "num_of_reviews"
       ]
     }
     ```  
   - Query Parameters:  
     - dataset_id = `gd_m6zagkt024uwvvwuyu`  
     - include_errors = true  
     - limit_multiple_results = 5  
   - Headers:  
     - Authorization: Bearer `YOUR_BRIGHT_DATA_API_KEY` (set your Bright Data API credentials here)  
   - Purpose: Start scraping job and receive a snapshot ID.

3. **Create HTTP Request Node "Check Scrape Status"**  
   - Connect from "Start Scraping Request".  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Headers:  
     - Authorization: Bearer `YOUR_BRIGHT_DATA_API_KEY`  
   - Purpose: Poll status of scraping job.

4. **Create Wait Node "Wait for Response 45 sec"**  
   - Connect from "Check Scrape Status".  
   - Wait Time: 45 seconds.  
   - Purpose: Delay between status polls.

5. **Create If Node "Verify Completion"**  
   - Connect from "Wait for Response 45 sec".  
   - Condition: Check if `status` field equals `ready`.  
   - If True: Connect to "Fetch Scraped Data" node (next step).  
   - If False: Connect back to "Check Scrape Status" node for another polling cycle.

6. **Create HTTP Request Node "Fetch Scraped Data"**  
   - Connect from "Verify Completion" (True branch).  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query Parameter: format = json  
   - Headers:  
     - Authorization: Bearer `YOUR_BRIGHT_DATA_API_KEY`  
   - Purpose: Retrieve final scraped app review data.

7. **Create Google Sheets Node "Save to Google Sheet"**  
   - Connect from "Fetch Scraped Data".  
   - Operation: Append  
   - Document ID: Your Google Sheet ID where data will be stored.  
   - Sheet Name: Typically "Sheet1" or the first sheet (gid=0).  
   - Column Mapping: Map fields such as url, review, review_id, app_rating, app_country, review_date, app_what_new, review_rating, reviewer_name, app_number_of_reviews, etc., exactly as per the scraped data fields.  
   - Credentials: Set up Google Sheets OAuth2 credentials with write permissions.  
   - Purpose: Store scraped reviews and app metadata.

8. **Create If Node "Check Low Ratings"**  
   - Connect from "Save to Google Sheet".  
   - Condition: `review_rating < 4` (numeric comparison)  
   - If True: Connect to "Send Alert to Telegram" node.

9. **Create Telegram Node "Send Alert to Telegram"**  
   - Connect from "Check Low Ratings" (True branch).  
   - Chat ID: Your Telegram chat ID to receive alerts.  
   - Message Text:  
     ```
     ⚠️ *Low App Performance Alert*  
     📱 *App:* {{ $json.app_title }}  
     🧑‍💻 *Developer:* {{ $json.app_developer }}  
     ⭐ *Rating:* {{ $json.app_rating }}  
     📝 *Reviews:* {{ $json.app_number_of_reviews }}  
     🔗 [View on Play Store]({{ $json.url }})
     ```  
   - Credentials: Telegram Bot API credentials setup (token).  
   - Purpose: Notify stakeholders of low-rated apps.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                 |
|----------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Bright Data API documentation is essential to understand dataset triggering and status APIs. | https://brightdata.com/docs/api-reference/datasets             |
| Google Sheets OAuth2 setup requires enabling Google Sheets API in Google Cloud Console.      | https://developers.google.com/sheets/api/quickstart/nodejs     |
| Telegram Bot creation guide to obtain API token and chat ID is necessary.                   | https://core.telegram.org/bots/api                             |
| Consider adding input validation on the form to prevent invalid URLs or review number inputs.| Best practice for robust workflows                              |
| Watch out for API rate limits from Bright Data and Telegram; implement error handling if needed.| To avoid workflow failures or throttling                        |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.