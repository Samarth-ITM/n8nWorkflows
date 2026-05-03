Google Maps Phone Scraper via Bright Data API with Google Sheets Sync

https://n8nworkflows.xyz/workflows/google-maps-phone-scraper-via-bright-data-api-with-google-sheets-sync-5043


# Google Maps Phone Scraper via Bright Data API with Google Sheets Sync

### 1. Workflow Overview

This workflow automates the extraction of business phone numbers and related details from Google Maps by leveraging the Bright Data API and synchronizing the results into a Google Sheets document. It is designed for businesses or marketers who want to gather local business information based on specific locations and keywords.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user input (location and keywords) from a form trigger.
- **1.2 Scraping Request:** Sends the scraping request to the Bright Data API to start data extraction.
- **1.3 Scraping Status Check:** Periodically checks if the scraping task is completed.
- **1.4 Data Validation:** Ensures that business records exist before proceeding.
- **1.5 Data Retrieval:** Fetches the scraped business data once ready.
- **1.6 Data Storage:** Saves the retrieved business information into a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initiates the workflow by receiving input parameters (Location and Keywords) via a form submission.
- **Nodes Involved:**  
  - Form Trigger - Submit Location and Keywords  
  - Sticky Note (Trigger description)  

- **Node Details:**  
  - **Form Trigger - Submit Location and Keywords**  
    - Type: Form Trigger  
    - Role: Listens for form submissions to start the workflow.  
    - Configuration: Expects two required fields: "Location" and "keywords".  
    - Input: HTTP request from form submission.  
    - Output: JSON containing form fields.  
    - Edge Cases: Missing required fields or malformed input could stop the workflow.  
    - Version: 2.2  
  - **Sticky Note**  
    - Type: Informational node  
    - Content: Indicates this is the workflow trigger start point.

---

#### 2.2 Scraping Request

- **Overview:** Sends a POST request to Bright Data API to trigger data scraping based on the provided location and keywords.
- **Nodes Involved:**  
  - Bright Data API - Request Business Data  
  - Sticky Note1  

- **Node Details:**  
  - **Bright Data API - Request Business Data**  
    - Type: HTTP Request  
    - Role: Sends scraping request to Bright Data API.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Body: JSON containing input location and keywords, requesting specific output fields such as URL, name, phone number, rating, address, etc.  
      - Query Parameters: dataset_id, include_errors, type, discover_by, limit_per_input.  
      - Headers: Authorization with Bearer token "BRIGHT_DATA_API_KEY" (placeholder).  
    - Expressions: Uses `{{$json.Location}}` and `{{$json.keywords}}` from form trigger input.  
    - Input: JSON from form trigger.  
    - Output: JSON with scraping task metadata including snapshot_id.  
    - Edge Cases: API authentication failure, invalid location/keywords, network errors.  
    - Version: 4.2  
  - **Sticky Note1**  
    - Content: "Sends scraping request to Bright Data API"

---

#### 2.3 Scraping Status Check

- **Overview:** Polls Bright Data API to check if the data scraping has completed.
- **Nodes Involved:**  
  - Check Scraping Status  
  - Check If Status Ready (IF node)  
  - Wait Before Retry (Wait node)  
  - Sticky Note2, Sticky Note3, Sticky Note4  

- **Node Details:**  
  - **Check Scraping Status**  
    - Type: HTTP Request  
    - Role: Queries the status of the scraping job using `snapshot_id`.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
      - Method: GET  
      - Headers: Authorization with Bearer token.  
      - Query: format=json  
    - Input: JSON including `snapshot_id` from previous node.  
    - Output: JSON with status field.  
    - Edge Cases: Invalid snapshot_id, API downtime, rate limits.  
    - Version: 4.2  
  - **Check If Status Ready**  
    - Type: IF  
    - Role: Checks if status equals "ready".  
    - Configuration: Compares `$json.status` to string "ready".  
    - Input: JSON from status check.  
    - Output: Two branches: true (ready), false (not ready).  
    - Edge Cases: Missing status field, unexpected status values.  
    - Version: 2.2  
  - **Wait Before Retry**  
    - Type: Wait  
    - Role: Pauses the workflow for 1 minute before re-checking status.  
    - Configuration: 1 minute delay.  
    - Input: From IF false branch.  
    - Output: Loops back to status check.  
    - Version: 1.1  
  - **Sticky Notes 2,3,4**  
    - Describe the status check process, readiness decision, and wait for retry.

---

#### 2.4 Data Validation

- **Overview:** Ensures that the scraping job returned records before proceeding to fetch data.
- **Nodes Involved:**  
  - Check Records Exist (IF node)  
  - Sticky Note5  

- **Node Details:**  
  - **Check Records Exist**  
    - Type: IF  
    - Role: Checks if the `records` field is not zero.  
    - Configuration: Numeric comparison: `$json.records` != 0  
    - Input: JSON from "Check If Status Ready" true branch.  
    - Output: true (records exist), false (no records).  
    - Edge Cases: Missing or malformed `records` field.  
    - Version: 2.2  
  - **Sticky Note5**  
    - Content: "Has Data? - Proceed only if business records found"

---

#### 2.5 Data Retrieval

- **Overview:** Retrieves the detailed business data from Bright Data API using the snapshot ID once scraping is complete and records exist.
- **Nodes Involved:**  
  - Fetch Business Data  
  - Sticky Note6  

- **Node Details:**  
  - **Fetch Business Data**  
    - Type: HTTP Request  
    - Role: Fetches the complete business data snapshot.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
      - Method: GET  
      - Headers: Authorization with Bearer token  
      - Query: format=json  
    - Input: JSON with `snapshot_id` from previous nodes.  
    - Output: JSON array of business data records.  
    - Edge Cases: Snapshot expired, API errors, large payloads.  
    - Version: 4.2  
  - **Sticky Note6**  
    - Content: "Fetch Data - Get business info including phone numbers"

---

#### 2.6 Data Storage

- **Overview:** Appends the fetched business data (name, URL, rating, address, phone number) into a specified Google Sheets document.
- **Nodes Involved:**  
  - Save to Google Sheets  
  - Sticky Note7  

- **Node Details:**  
  - **Save to Google Sheets**  
    - Type: Google Sheets  
    - Role: Stores business data into Google Sheets.  
    - Configuration:  
      - Operation: Append rows  
      - Document ID: Placeholder "YOUR_GOOGLE_SHEET_ID"  
      - Sheet Name: "GMB" (Sheet tab ID 619305781)  
      - Columns Mapped: URL, Name, Rating, Address, Phone Number from JSON fields.  
    - Credentials: Google Sheets OAuth2 credential required.  
    - Input: Business data JSON from Fetch Business Data node.  
    - Output: Operation result confirmation.  
    - Edge Cases: Authentication failure, sheet access issues, data type mismatches.  
    - Version: 4.6  
  - **Sticky Note7**  
    - Content: "Saves business data to the GMB sheet in your Google Sheet"

---

### 3. Summary Table

| Node Name                        | Node Type         | Functional Role                   | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                  |
|---------------------------------|-------------------|---------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note                     | Sticky Note       | Workflow start info              | None                          | Form Trigger - Submit Location and Keywords | 📝 Trigger - Start when form is submitted                                                    |
| Form Trigger - Submit Location and Keywords | Form Trigger      | Input reception - receives location and keywords | Sticky Note                    | Bright Data API - Request Business Data    |                                                                                              |
| Sticky Note1                   | Sticky Note       | Description of scraping request | Bright Data API - Request Business Data | Bright Data API - Request Business Data   | Sends scraping request to Bright Data API                                                  |
| Bright Data API - Request Business Data | HTTP Request      | Sends scraping request to Bright Data API | Form Trigger                  | Check Scraping Status             |                                                                                              |
| Sticky Note2                   | Sticky Note       | Status check description         | Check Scraping Status          | Check If Status Ready            | ⏳ Check Status - Is data scraping completed?                                               |
| Check Scraping Status          | HTTP Request      | Checks scraping progress         | Bright Data API - Request Business Data | Check If Status Ready            |                                                                                              |
| Sticky Note3                   | Sticky Note       | Status readiness decision info   | Check If Status Ready          | Check Records Exist / Wait Before Retry | ✅ Is Ready? - If ready, continue; if not, wait                                             |
| Check If Status Ready          | IF                | Determines if scraping is ready  | Check Scraping Status          | Check Records Exist / Wait Before Retry |                                                                                              |
| Sticky Note4                   | Sticky Note       | Wait description before retry    | Wait Before Retry              | Check Scraping Status            | ⏱️ Wait - Pause 1 min before checking again                                                |
| Wait Before Retry              | Wait              | Pauses workflow 1 minute before retrying status check | Check If Status Ready (false branch) | Check Scraping Status            |                                                                                              |
| Sticky Note5                   | Sticky Note       | Validates if records exist       | Check If Status Ready (true branch) | Check Records Exist              | 📊 Has Data? - Proceed only if business records found                                       |
| Check Records Exist            | IF                | Checks if records count > 0      | Check If Status Ready (true branch) | Fetch Business Data              |                                                                                              |
| Sticky Note6                   | Sticky Note       | Describes data fetching          | Check Records Exist            | Fetch Business Data              | 📥 Fetch Data - Get business info including phone numbers                                   |
| Fetch Business Data            | HTTP Request      | Retrieves scraped business data  | Check Records Exist            | Save to Google Sheets            |                                                                                              |
| Sticky Note7                   | Sticky Note       | Describes Google Sheets storage  | Fetch Business Data            | Save to Google Sheets            | Saves business data to the GMB sheet in your Google Sheet📄 Save to Sheet - Store business name, number, URL, etc. |
| Save to Google Sheets          | Google Sheets     | Stores business data into sheet  | Fetch Business Data            | None                           |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**
   - Type: Form Trigger  
   - Configure form title: "GMB"  
   - Add two required fields: "Location" and "keywords"  
   - Save webhook ID (auto-generated) for external form submissions.

2. **Create HTTP Request Node (Scraping Request):**
   - Name: "Bright Data API - Request Business Data"  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Headers: Add Authorization header with value `Bearer BRIGHT_DATA_API_KEY` (replace with your actual API key).  
   - Query Parameters:  
     - dataset_id = `gd_m8ebnr0q2qlklc02fz`  
     - include_errors = `true`  
     - type = `discover_new`  
     - discover_by = `location`  
     - limit_per_input = `2`  
   - Body (JSON format):  
     ```json
     {
       "input": [
         {
           "country": "={{ $json.Location }}",
           "keyword": "={{ $json.keywords }}",
           "lat": ""
         }
       ],
       "custom_output_fields": [
         "url",
         "country",
         "name",
         "address",
         "description",
         "open_hours",
         "reviews_count",
         "rating",
         "reviews",
         "services_provided",
         "open_website",
         "phone_number",
         "permanently_closed",
         "photos_and_videos",
         "people_also_search"
       ]
     }
     ```
   - Connect output of Form Trigger to this node.

3. **Create HTTP Request Node (Check Scraping Status):**
   - Name: "Check Scraping Status"  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Query Parameter: format=json  
   - Header: Authorization: `Bearer BRIGHT_DATA_API_KEY`  
   - Connect output of Bright Data API Request node to this node.

4. **Create IF Node (Check If Status Ready):**
   - Name: "Check If Status Ready"  
   - Condition: `$json.status` equals "ready" (string)  
   - Connect output of Check Scraping Status to this IF node.

5. **Create Wait Node (Wait Before Retry):**
   - Name: "Wait Before Retry"  
   - Wait Duration: 1 minute  
   - Connect false output of IF node (status not ready) to this node.  
   - Connect output of Wait node back to "Check Scraping Status" node for polling loop.

6. **Create IF Node (Check Records Exist):**
   - Name: "Check Records Exist"  
   - Condition: `$json.records` not equals 0 (number)  
   - Connect true output of "Check If Status Ready" to this node.

7. **Create HTTP Request Node (Fetch Business Data):**
   - Name: "Fetch Business Data"  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query Parameter: format=json  
   - Header: Authorization: `Bearer BRIGHT_DATA_API_KEY`  
   - Connect true output of "Check Records Exist" to this node.

8. **Create Google Sheets Node (Save to Google Sheets):**
   - Name: "Save to Google Sheets"  
   - Operation: Append  
   - Document ID: Your Google Sheets document ID (replace placeholder)  
   - Sheet Name: "GMB" (or your sheet tab name/ID)  
   - Map columns:  
     - URL: `={{ $json.url }}`  
     - Name: `={{ $json.name }}`  
     - Rating: `={{ $json.rating }}`  
     - Address: `={{ $json.address }}`  
     - Phone Number: `={{ $json.phone_number }}`  
   - Configure Google Sheets OAuth2 credentials.  
   - Connect output of "Fetch Business Data" node to this node.

9. **Add Sticky Notes (Optional):**
   - Add sticky notes near each block for clarity, labeling triggers, API requests, status checks, waiting, data validation, fetching, and storage.

10. **Test the Workflow:**
    - Ensure API keys and OAuth credentials are correctly set.  
    - Submit form with test Location and Keywords.  
    - Monitor logs for polling and successful data storage.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow uses Bright Data API – requires valid API key with access to dataset `gd_m8ebnr0q2qlklc02fz` | Bright Data API documentation: https://brightdata.com/api-docs                                    |
| Google Sheets OAuth2 credentials must be configured with access to target Google Sheets document      | Google Sheets API: https://developers.google.com/sheets/api                                        |
| Polling interval set to 1 minute to avoid rate limits and API overload                               | Consider adjusting wait duration based on API rate limits and expected scraping duration           |
| Ensure Google Sheets document has a sheet named "GMB" or update sheetName and sheet ID accordingly    | Google Sheets URL format: https://docs.google.com/spreadsheets/d/YOUR_GOOGLE_SHEET_ID/edit#gid=sheetId |
| This workflow is designed for small batches (limit_per_input=2) – adjust as needed for larger scraping | API limits and costs should be considered when scaling up scraping requests                         |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.