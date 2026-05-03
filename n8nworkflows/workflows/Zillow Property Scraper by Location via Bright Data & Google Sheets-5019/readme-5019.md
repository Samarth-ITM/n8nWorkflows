Zillow Property Scraper by Location via Bright Data & Google Sheets

https://n8nworkflows.xyz/workflows/zillow-property-scraper-by-location-via-bright-data---google-sheets-5019


# Zillow Property Scraper by Location via Bright Data & Google Sheets

### 1. Workflow Overview

This workflow automates the scraping of Zillow property listings based on user-specified locations and listing categories (“House for rent” or “House for sale”). It leverages the Bright Data API for data extraction and saves the results into a Google Sheet for further analysis or use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user inputs (location and listing category) via a form trigger.
- **1.2 Scraping Job Initiation:** Sends a request to Bright Data API to start a Zillow data scraping job with the user inputs.
- **1.3 Scraping Job Monitoring:** Periodically checks the status of the scraping job until completion.
- **1.4 Data Validation and Retrieval:** Validates if data is available and fetches the scraped property listings using the snapshot ID.
- **1.5 Data Storage:** Appends the fetched property data to a configured Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block waits for a user to submit a form specifying the location and listing category. It serves as the entry point for the workflow.

- **Nodes Involved:**  
  - 📝 Form Trigger - Start Property Search  
  - Form Submission Note (Sticky Note)

- **Node Details:**

  - **📝 Form Trigger - Start Property Search**  
    - Type: Form Trigger  
    - Role: Entry point capturing user inputs  
    - Configuration:  
      - Form title: “Zillow Property Search”  
      - Fields:  
        - Location (required, text input)  
        - Listing Category (dropdown with options “House for rent” and “House for sale”)  
    - Expressions: Uses `$json.Location` and `$json['Listing Category']` later in the workflow  
    - Input: External user submission  
    - Output: Starts the workflow with form data  
    - Edge cases: Missing required field for Location will prevent submission; invalid category not possible due to dropdown  
    - Notes: Accompanied by sticky note explaining the form submission start  

  - **Form Submission Note**  
    - Type: Sticky Note  
    - Content: “Starts workflow when user submits location & category via form”  

#### 2.2 Scraping Job Initiation

- **Overview:**  
  This block triggers the Bright Data API to start the scraping job for the given location and listing category.

- **Nodes Involved:**  
  - 📤 Trigger Bright Data Scraping Job  
  - API Trigger Note (Sticky Note)

- **Node Details:**

  - **📤 Trigger Bright Data Scraping Job**  
    - Type: HTTP Request  
    - Role: Sends POST request to Bright Data API endpoint to trigger scraping  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Headers: Authorization Bearer token with placeholder `BRIGHT_DATA_API_KEY`  
      - Query parameters: dataset_id, include_errors, type, discover_by, limit_per_input  
      - JSON Body: Includes user inputs for location and listing category; HomeType and days_on_zillow are empty strings  
      - Expressions: Injects form data dynamically (`{{ $json.Location }}`, `{{ $json['Listing Category'] }}`)  
    - Input: Form trigger output  
    - Output: JSON response with job details including `snapshot_id`  
    - Edge cases: Authentication failure if API key invalid; network errors; malformed JSON body; API rate limits  
    - Notes: Sticky note clarifies purpose  

  - **API Trigger Note**  
    - Type: Sticky Note  
    - Content: “Sends search request to Bright Data API to trigger Zillow data scraping”  

#### 2.3 Scraping Job Monitoring

- **Overview:**  
  This block checks the status of the scraping job repeatedly until it is marked as “ready”, waiting 1 minute between retries.

- **Nodes Involved:**  
  - ⏳ Check Scraping Job Status  
  - ✅ Check If Scraping Complete (If node)  
  - ⏱️ Wait Before Retry  
  - Status Check Note (Sticky Note)  
  - Wait Timer Note (Sticky Note)

- **Node Details:**

  - **⏳ Check Scraping Job Status**  
    - Type: HTTP Request  
    - Role: Queries Bright Data API for job progress using `snapshot_id`  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (dynamic snapshot_id)  
      - Method: GET  
      - Headers: Authorization Bearer token  
      - Query parameter: format=json  
    - Input: Output from scraping job trigger or wait node  
    - Output: JSON with job status field `status`  
    - Edge cases: Invalid or expired snapshot_id; API auth errors; network timeouts  

  - **✅ Check If Scraping Complete**  
    - Type: If node  
    - Role: Checks if the `status` field equals `ready`  
    - Configuration:  
      - Condition: `$json.status == "ready"`  
    - Input: Status check node output  
    - Output:  
      - True: Continue to validate property data  
      - False: Loop back to wait node to retry  

  - **⏱️ Wait Before Retry**  
    - Type: Wait node  
    - Role: Pauses workflow for 1 minute before next status check  
    - Configuration: Wait 1 minute (unit: minutes, amount: 1)  
    - Input: False branch of If node  
    - Output: Loops back to status check node  

  - **Status Check Note**  
    - Type: Sticky Note  
    - Content: “Monitors scraping job status and waits for completion”  

  - **Wait Timer Note**  
    - Type: Sticky Note  
    - Content: “Waits 1 minute before rechecking scraping job status”  

#### 2.4 Data Validation and Retrieval

- **Overview:**  
  Once the scraping job is ready, this block validates if property data exists (non-zero records). If data is present, it fetches the actual property listings using the snapshot ID.

- **Nodes Involved:**  
  - 📊 Validate Property Data Exists (If node)  
  - 📥 Fetch Property Listing Data  
  - Data Validation Note (Sticky Note)  
  - Data Retrieval Note (Sticky Note)

- **Node Details:**

  - **📊 Validate Property Data Exists**  
    - Type: If node  
    - Role: Checks that the `records` field in JSON is not zero, confirming data presence  
    - Configuration:  
      - Condition: `$json.records != 0`  
    - Input: Output from job status complete node  
    - Output:  
      - True: Proceed to fetch data  
      - False: No further action (workflow ends or could be extended)  
    - Edge cases: Missing or malformed `records` field; zero records indicating no data scraped  

  - **📥 Fetch Property Listing Data**  
    - Type: HTTP Request  
    - Role: Retrieves snapshot data (property listings) from Bright Data API  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
      - Method: GET  
      - Headers: Authorization Bearer token  
      - Query param: format=json  
    - Input: True branch from data validation node  
    - Output: JSON array of property listings  
    - Edge cases: Invalid snapshot_id; API errors; empty results  

  - **Data Validation Note**  
    - Type: Sticky Note  
    - Content: “Validates if property data was found in scraping results”  

  - **Data Retrieval Note**  
    - Type: Sticky Note  
    - Content: “Retrieves the actual property data using snapshot ID”  

#### 2.5 Data Storage

- **Overview:**  
  This block appends the fetched property listing data into a Google Sheet for record-keeping and further use.

- **Nodes Involved:**  
  - 📄 Save Property Data to Google Sheets  
  - Google Sheets Save Note (Sticky Note)  
  - Google Sheet Template Note (Sticky Note)

- **Node Details:**

  - **📄 Save Property Data to Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends property data rows to a specific sheet  
    - Configuration:  
      - Operation: Append  
      - Sheet Name: “Zillow”  
      - Document ID: Placeholder `YOUR_GOOGLE_SHEET_ID` (must be replaced)  
      - Columns mapped explicitly for fields such as URL, City, Country, Home Type, Zestimate, Year Built, Agent Phone, Home Status, School Rating, Street Address, Interior Details  
      - Expressions map from JSON fields (e.g., `={{ $json.hdpUrl }}`, `={{ $json.city }}`, etc.)  
    - Credentials: Google Sheets OAuth2 (configured separately)  
    - Input: Property data from Bright Data API snapshot  
    - Output: Confirmation of row append operation  
    - Edge cases: Credential auth failure; incorrect sheet or document ID; data type mismatches; Google API rate limits  

  - **Google Sheets Save Note**  
    - Type: Sticky Note  
    - Content: “Saves the scraped property data to your Google Sheet”  

  - **Google Sheet Template Note**  
    - Type: Sticky Note  
    - Content:  
      ```
      Sample Google Sheet Template:
      https://docs.google.com/spreadsheets/d/SAMPLE_SHEET_ID/edit

      Make a copy and update the workflow with your Sheet ID
      ```  

---

### 3. Summary Table

| Node Name                          | Node Type          | Functional Role                             | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                    |
|-----------------------------------|--------------------|--------------------------------------------|----------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| 📝 Form Trigger - Start Property Search | Form Trigger       | Capture user location & category input     | (Trigger)                       | 📤 Trigger Bright Data Scraping Job | Starts workflow when user submits location & category via form                                                |
| 📤 Trigger Bright Data Scraping Job | HTTP Request       | Send scraping job trigger to Bright Data API | 📝 Form Trigger - Start Property Search | ⏳ Check Scraping Job Status      | Sends search request to Bright Data API to trigger Zillow data scraping                                        |
| ⏳ Check Scraping Job Status       | HTTP Request       | Query scraping job progress by snapshot_id | 📤 Trigger Bright Data Scraping Job / ⏱️ Wait Before Retry | ✅ Check If Scraping Complete     | Monitors scraping job status and waits for completion                                                        |
| ✅ Check If Scraping Complete      | If                 | Check if scraping job status is “ready”    | ⏳ Check Scraping Job Status       | 📊 Validate Property Data Exists / ⏱️ Wait Before Retry |                                                                                                               |
| ⏱️ Wait Before Retry               | Wait               | Wait 1 minute before retrying status check | ✅ Check If Scraping Complete     | ⏳ Check Scraping Job Status      | Waits 1 minute before rechecking scraping job status                                                          |
| 📊 Validate Property Data Exists   | If                 | Ensure scraped data exists (records > 0)   | ✅ Check If Scraping Complete      | 📥 Fetch Property Listing Data   | Validates if property data was found in scraping results                                                      |
| 📥 Fetch Property Listing Data     | HTTP Request       | Retrieve scraped property listings          | 📊 Validate Property Data Exists  | 📄 Save Property Data to Google Sheets | Retrieves the actual property data using snapshot ID                                                          |
| 📄 Save Property Data to Google Sheets | Google Sheets      | Append property listings to Google Sheet    | 📥 Fetch Property Listing Data    | (End)                           | Saves the scraped property data to your Google Sheet                                                          |
| Form Submission Note               | Sticky Note        | Explains form trigger start                  | -                                | -                               | Starts workflow when user submits location & category via form                                                |
| API Trigger Note                  | Sticky Note        | Explains Bright Data API trigger             | -                                | -                               | Sends search request to Bright Data API to trigger Zillow data scraping                                        |
| Status Check Note                 | Sticky Note        | Explains job status monitoring                | -                                | -                               | Monitors scraping job status and waits for completion                                                        |
| Wait Timer Note                  | Sticky Note        | Explains wait node purpose                     | -                                | -                               | Waits 1 minute before rechecking scraping job status                                                          |
| Data Validation Note             | Sticky Note        | Explains data validation check                 | -                                | -                               | Validates if property data was found in scraping results                                                      |
| Data Retrieval Note              | Sticky Note        | Explains data retrieval step                    | -                                | -                               | Retrieves the actual property data using snapshot ID                                                          |
| Google Sheets Save Note          | Sticky Note        | Explains Google Sheets data saving             | -                                | -                               | Saves the scraped property data to your Google Sheet                                                          |
| Google Sheet Template Note       | Sticky Note        | Provides sample Google Sheet template link    | -                                | -                               | Sample Google Sheet Template: https://docs.google.com/spreadsheets/d/SAMPLE_SHEET_ID/edit Make a copy and update the workflow with your Sheet ID |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: Form Trigger  
   - Name: “📝 Form Trigger - Start Property Search”  
   - Configure form title as “Zillow Property Search”  
   - Add form fields:  
     - Location (Text, Required)  
     - Listing Category (Dropdown with options: “House for rent”, “House for sale”)  
   - This node serves as the entry point.

2. **Add HTTP Request Node to Trigger Scraping Job:**  
   - Name: “📤 Trigger Bright Data Scraping Job”  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Headers: Add Authorization header with Bearer token `BRIGHT_DATA_API_KEY` (replace with valid key)  
   - Query Parameters:  
     - dataset_id = gd_lfqkr8wm13ixtbd8f5  
     - include_errors = true  
     - type = discover_new  
     - discover_by = input_filters  
     - limit_per_input = 2  
   - Body (JSON, raw mode):  
     ```json
     {
       "input": [
         {
           "location": "={{ $json.Location }}",
           "listingCategory": "={{ $json['Listing Category'] }}",
           "HomeType": "",
           "days_on_zillow": ""
         }
       ]
     }
     ```
   - Connect output of Form Trigger node to this node.

3. **Add HTTP Request Node to Check Scraping Job Status:**  
   - Name: “⏳ Check Scraping Job Status”  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Headers: Authorization Bearer token (same as above)  
   - Query Param: format=json  
   - Connect output of scraping trigger node to this node.

4. **Add If Node to Check If Scraping Is Complete:**  
   - Name: “✅ Check If Scraping Complete”  
   - Condition: `$json.status` equals `ready`  
   - Connect output of status check node to this If node.

5. **Add Wait Node to Pause Before Retry:**  
   - Name: “⏱️ Wait Before Retry”  
   - Wait Duration: 1 minute  
   - Connect false output of If node to this wait node.

6. **Loop Back From Wait Node to Status Check Node:**  
   - Connect output of wait node back to the input of the “⏳ Check Scraping Job Status” node.

7. **Add If Node to Validate Property Data Exists:**  
   - Name: “📊 Validate Property Data Exists”  
   - Condition: `$json.records` not equal to `0`  
   - Connect true output of “✅ Check If Scraping Complete” node to this validation node.

8. **Add HTTP Request Node to Fetch Property Listing Data:**  
   - Name: “📥 Fetch Property Listing Data”  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Headers: Authorization Bearer token  
   - Query Param: format=json  
   - Connect true output of validation node to this node.

9. **Add Google Sheets Node to Save Data:**  
   - Name: “📄 Save Property Data to Google Sheets”  
   - Operation: Append  
   - Sheet Name: “Zillow” (create this sheet in your Google Sheet document)  
   - Document ID: Your Google Sheet ID (replace placeholder)  
   - Columns: Map the fields as follows (expressions):  
     - URL → `={{ $json.hdpUrl }}`  
     - City → `={{ $json.city }}`  
     - Country → `={{ $json.country }}`  
     - Home Type → `={{ $json.homeType }}`  
     - Zestimate → `={{ $json.zestimate }}`  
     - Year Built → `={{ $json.yearBuilt }}`  
     - Agent Phone → `={{ $json.listing_provided_by.phone_number }}`  
     - Home Status → `={{ $json.homeStatus }}`  
     - School Rating → `={{ $json.schools[0].rating }}`  
     - Street Address → `={{ $json.address }}`  
     - Interior Details → `={{ $json.interior_full[0].values }}`  
   - Set Google Sheets OAuth2 credentials (create and configure in n8n)  
   - Connect output of fetch data node to this node.

10. **Final Connections and Testing:**  
    - Test the form by submitting location and listing category.  
    - Ensure API keys and Google Sheets credentials are valid.  
    - Adjust sheet name and document ID as needed.  
    - Monitor execution and logs for errors.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Sample Google Sheet Template: https://docs.google.com/spreadsheets/d/SAMPLE_SHEET_ID/edit          | Use this to create your own Google Sheet for data storage; update workflow with your Sheet ID           |
| Make a copy and update the workflow with your Sheet ID                                             | Instruction for users to personalize the Google Sheets integration                                      |
| Bright Data API documentation and authentication are required for this workflow to operate correctly | Refer to https://brightdata.com/docs/api for API usage and key management                               |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.