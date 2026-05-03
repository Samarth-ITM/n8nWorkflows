Scrape TikTok Influencer Profiles with Bright Data API to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-tiktok-influencer-profiles-with-bright-data-api-to-google-sheets-5434


# Scrape TikTok Influencer Profiles with Bright Data API to Google Sheets

### 1. Workflow Overview

This workflow automates scraping TikTok influencer profile data using the Bright Data API and appends the extracted data into a Google Sheet. It is designed for users who want to submit TikTok profile URLs via a form, trigger Bright Data’s scraping service, monitor the scraping status, retrieve the scraped influencer details once ready, and save the results for further analysis or reporting.

The workflow is logically divided into these functional blocks:

- **1.1 Input Reception:** Captures TikTok profile URL input from a user-submitted form.
- **1.2 Trigger Scraping:** Sends the profile URLs to Bright Data’s API to initiate scraping.
- **1.3 Polling and Status Checking:** Continuously checks Bright Data for completion of the scraping job, with wait intervals.
- **1.4 Data Retrieval:** Once scraping is complete, fetches the detailed influencer data snapshot.
- **1.5 Data Storage:** Saves the retrieved influencer profile data into a specified Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Starts the workflow by receiving user input through a form where the TikTok profile URL is submitted to be scraped.

- **Nodes Involved:**  
  - Search by Profile URL (Form Trigger)  
  - Sticky Note (Trigger description)

- **Node Details:**  
  - **Search by Profile URL**  
    - *Type:* Form Trigger  
    - *Role:* Entry point to accept a TikTok profile URL from a user form submission.  
    - *Config:* Form titled "TikTok Scraper" with a single input field labeled "Search by Profile".  
    - *Input:* External user form submission.  
    - *Output:* Passes form data downstream.  
    - *Edge cases:* Missing or invalid URLs submitted; no validation is visible here so malformed inputs might cause issues downstream.  
  - **Sticky Note**  
    - *Role:* Documentation to mark the trigger node as "Start on TikTok profile form submit".  
    - *No functional impact.*

#### 1.2 Trigger Scraping

- **Overview:**  
  Takes the profile URLs from the form and triggers Bright Data’s scraping job via their API, specifying dataset and output fields.

- **Nodes Involved:**  
  - Sends profile URLs to Bright Data to trigger scraping (HTTP Request)  
  - Sticky Note1 (Trigger API description)

- **Node Details:**  
  - **Sends profile URLs to Bright Data to trigger scraping**  
    - *Type:* HTTP Request  
    - *Role:* POST request to Bright Data API to start scraping dataset `gd_l1villgoiiidt09ci` with given profile URLs and options.  
    - *Config:*  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Query params: dataset ID, include errors, type (discover_new), discover_by (search_url), limit per input = 2  
      - JSON body includes sample input URLs (fixed in this config, but in a real dynamic setting would be replaced by form input), and custom output fields specifying influencer data properties to retrieve.  
      - Authorization header uses `Bearer BRIGHT_DATA_API_KEY` (credential to be set).  
    - *Input:* TikTok profile URLs (in this exported workflow the body is hardcoded JSON; a real implementation should inject the form input here).  
    - *Output:* Response from Bright Data with snapshot ID and status.  
    - *Edge cases:* API key invalid or expired, rate limits, network errors, malformed request, or empty input URLs.  

  - **Sticky Note1**  
    - Notes this node as API trigger sending TikTok profile to scrape.

#### 1.3 Polling and Status Checking

- **Overview:**  
  Polls Bright Data’s API repeatedly to check if the scraping job for the snapshot ID is completed (status "ready"). Uses a wait node to delay retries.

- **Nodes Involved:**  
  - Checks scraping progress from Bright Data (HTTP Request)  
  - Waits 30 seconds before retrying snapshot check (Wait)  
  - IF condition to check if Bright Data returned ready status (IF)  
  - Sticky Notes 2, 3, and 4 (status check, wait, and decision explanation)

- **Node Details:**  
  - **Checks scraping progress from Bright Data**  
    - *Type:* HTTP Request  
    - *Role:* GET request to `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
    - *Config:* Uses snapshot_id from previous node output to query progress status. Authorization header with Bearer token.  
    - *Input:* snapshot_id from trigger response.  
    - *Output:* Status JSON indicating if scraping is done.  
    - *Edge cases:* Snapshot ID missing or invalid, API errors, timeouts.  
  - **Waits 30 seconds before retrying snapshot check**  
    - *Type:* Wait  
    - *Role:* Pauses workflow execution for 30 seconds before next polling attempt.  
    - *Config:* Fixed 30 seconds delay.  
  - **IF condition to check if Bright Data returned ready status**  
    - *Type:* IF  
    - *Role:* Branches execution based on whether status is "ready".  
    - *Config:* Checks if `$json.status` equals "ready".  
    - *Output:*  
      - If true, proceed to data retrieval.  
      - If false, loop back to check progress again.  
  - **Sticky Note2, 3, 4**  
    - Describe the process of checking status, waiting between retries, and conditional branching.

#### 1.4 Data Retrieval

- **Overview:**  
  Once Bright Data indicates scraping is complete, fetches the full influencer data snapshot in JSON format.

- **Nodes Involved:**  
  - Gets TikTok influencer details from Bright Data using snapshot ID (HTTP Request)  
  - Sticky Note5 (data extraction note)

- **Node Details:**  
  - **Gets TikTok influencer details from Bright Data using snapshot ID**  
    - *Type:* HTTP Request  
    - *Role:* GET request to `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}` to retrieve scraped influencer data.  
    - *Config:* Query parameter `format=json` for JSON response, Authorization header with Bearer token.  
    - *Input:* snapshot_id from previous IF node output.  
    - *Output:* Detailed influencer profile data as JSON.  
    - *Edge cases:* Snapshot ID invalid, data not ready despite status, API errors.  
  - **Sticky Note5**  
    - Marks this node as the data extraction step.

#### 1.5 Data Storage

- **Overview:**  
  Appends or updates the influencer data in a specified Google Sheet for storage and further use.

- **Nodes Involved:**  
  - Google Sheets (Google Sheets node)  
  - Sticky Note6 (save to sheet explanation)

- **Node Details:**  
  - **Google Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Appends or updates rows in the sheet named "TikTok profile by url" within a specific Google Sheets document.  
    - *Config:*  
      - Document ID and sheet name explicitly set to target the correct spreadsheet and tab.  
      - Columns defined: "status" and "message" (string types).  
      - Operation: appendOrUpdate.  
      - Credential: Google Sheets OAuth2 API credential configured (named "Google Sheets demo").  
    - *Input:* Influencer data JSON from Bright Data snapshot node.  
    - *Output:* Confirmation of data written.  
    - *Edge cases:* Credential expiration, API limits, schema mismatch, network errors.  
  - **Sticky Note6**  
    - Describes this node as saving data to Google Sheet.

---

### 3. Summary Table

| Node Name                                         | Node Type                | Functional Role                           | Input Node(s)                         | Output Node(s)                                      | Sticky Note                                                                                  |
|--------------------------------------------------|--------------------------|-----------------------------------------|-------------------------------------|----------------------------------------------------|----------------------------------------------------------------------------------------------|
| Search by Profile URL                             | Form Trigger             | Receives TikTok profile URL input       | -                                   | Sends profile URLs to Bright Data to trigger scraping | 📝 Trigger – Start on TikTok profile form submit                                             |
| Sticky Note                                      | Sticky Note              | Documentation for trigger node          | -                                   | -                                                  | 📝 Trigger – Start on TikTok profile form submit                                             |
| Sends profile URLs to Bright Data to trigger scraping | HTTP Request             | Initiates scraping job on Bright Data   | Search by Profile URL                | Checks scraping progress from Bright Data           | 📤 Trigger API – Send TikTok profile to scrape                                              |
| Sticky Note1                                     | Sticky Note              | Documentation for trigger API node      | -                                   | -                                                  | 📤 Trigger API – Send TikTok profile to scrape                                              |
| Checks scraping progress from Bright Data        | HTTP Request             | Polls Bright Data for scraping progress | Sends profile URLs to Bright Data to trigger scraping, IF condition | Waits 30 seconds before retrying snapshot check, IF condition | ⏳ Check Status – Is profile data ready?                                                    |
| Waits 30 seconds before retrying snapshot check  | Wait                     | Waits before retrying status check      | Checks scraping progress from Bright Data | IF condition to check if Bright Data returned ready status | ⏱️ Wait – Pause before retrying Bright Data                                                  |
| IF condition to check if Bright Data returned ready status | IF                       | Branches flow based on scraping status  | Waits 30 seconds before retrying snapshot check | Gets TikTok influencer details from Bright Data using snapshot ID, Checks scraping progress from Bright Data | ✅ Is Ready? – Yes: proceed, No: retry                                                      |
| Gets TikTok influencer details from Bright Data using snapshot ID | HTTP Request             | Retrieves scraped influencer data       | IF condition to check if Bright Data returned ready status (true branch) | Google Sheets                                        | 📥 Get Data – Extract scraped influencer info                                               |
| Sticky Note5                                     | Sticky Note              | Documentation for data extraction node  | -                                   | -                                                  | 📥 Get Data – Extract scraped influencer info                                               |
| Google Sheets                                    | Google Sheets            | Saves influencer data to Google Sheet   | Gets TikTok influencer details from Bright Data using snapshot ID | -                                                  | 📄 Save to Sheet – Write data to Google Sheet                                               |
| Sticky Note6                                     | Sticky Note              | Documentation for save to sheet node    | -                                   | -                                                  | 📄 Save to Sheet – Write data to Google Sheet                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger: "Search by Profile URL"**  
   - Node type: Form Trigger  
   - Configure form title: "TikTok Scraper"  
   - Add one form field: Label it "Search by Profile" (type: string or URL)  
   - Position: Entry node  
   - No credentials needed  

2. **Add Sticky Note (optional):**  
   - Content: "📝 Trigger – Start on TikTok profile form submit"  
   - Position near Form Trigger node for clarity  

3. **Create HTTP Request: "Sends profile URLs to Bright Data to trigger scraping"**  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Query parameters:  
     - dataset_id: `gd_l1villgoiiidt09ci`  
     - include_errors: `true`  
     - type: `discover_new`  
     - discover_by: `search_url`  
     - limit_per_input: `2`  
   - Body (JSON, dynamic): Inject TikTok profile URL(s) from the form trigger (replace static example URLs with form input)  
   - Custom output fields: Include influencer data fields as per original (account_id, nickname, biography, etc.)  
   - Headers: Authorization Bearer token with Bright Data API key credential (create and configure credential with your Bright Data API key)  
   - Connect Form Trigger output to this node input  

4. **Add Sticky Note (optional):**  
   - Content: "📤 Trigger API – Send TikTok profile to scrape"  

5. **Create HTTP Request: "Checks scraping progress from Bright Data"**  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (use expression to insert snapshot_id from previous response)  
   - Headers: Authorization Bearer token with Bright Data API key  
   - Always output data enabled (to prevent node from stopping on empty response)  
   - Connect output of trigger scraping node to this node  

6. **Add Sticky Note (optional):**  
   - Content: "⏳ Check Status – Is profile data ready?"  

7. **Create Wait node: "Waits 30 seconds before retrying snapshot check"**  
   - Wait time: 30 seconds  
   - Connect output of status check node to this wait node  

8. **Add Sticky Note (optional):**  
   - Content: "⏱️ Wait – Pause before retrying Bright Data"  

9. **Create IF node: "IF condition to check if Bright Data returned ready status"**  
   - Condition: `$json.status` equals "ready" (case sensitive)  
   - Connect Wait node output to IF node input  

10. **Add Sticky Note (optional):**  
    - Content: "✅ Is Ready? – Yes: proceed, No: retry"  

11. **Connect IF node outputs:**  
    - True branch (ready): Connect to next HTTP Request node to get scraped data  
    - False branch (not ready): Connect back to "Checks scraping progress from Bright Data" node to poll again  

12. **Create HTTP Request: "Gets TikTok influencer details from Bright Data using snapshot ID"**  
    - Method: GET  
    - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
    - Query parameter: `format=json`  
    - Headers: Authorization Bearer token with Bright Data API key  
    - Connect True output of IF node to this node  

13. **Add Sticky Note (optional):**  
    - Content: "📥 Get Data – Extract scraped influencer info"  

14. **Create Google Sheets node: "Google Sheets"**  
    - Operation: Append or Update  
    - Document ID: `1OeqtCFm4Wek9DI5YFOWQXTpQJS-SJxC10iAPKEKkmiY` (replace with your own spreadsheet ID)  
    - Sheet Name: "TikTok profile by url" (or your sheet tab name)  
    - Define columns: "status" and "message" (string)  
    - Credential: Configure Google Sheets OAuth2 credential with appropriate access  
    - Connect output of data retrieval node to this Google Sheets node  

15. **Add Sticky Note (optional):**  
    - Content: "📄 Save to Sheet – Write data to Google Sheet"  

16. **Test the workflow:**  
    - Trigger the form with a TikTok profile URL.  
    - Confirm Bright Data API calls are authorized and data flows through nodes.  
    - Verify data appears in the target Google Sheet.  

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| The workflow requires valid API credentials for Bright Data API (Bearer token) and Google Sheets OAuth2.       | Ensure you create and configure these credentials in n8n before executing the workflow.                           |
| The Bright Data API key is referenced as `BRIGHT_DATA_API_KEY` in HTTP Request headers — replace with real key. | https://brightdata.com/docs/api/datasets                                                                          |
| Google Sheets document and sheet names are hardcoded and must be accessible by the OAuth2 credential used.      | https://developers.google.com/sheets/api/guides/authorizing                                                     |
| The workflow uses a polling mechanism with a 30-second wait between retries to handle asynchronous scraping.    | Adjust wait time as needed for API limits or performance considerations.                                          |
| Sticky Notes provide inline documentation for each logical step, improving maintainability and clarity.         | Sticky notes do not affect workflow execution but are valuable for collaborative understanding.                  |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.