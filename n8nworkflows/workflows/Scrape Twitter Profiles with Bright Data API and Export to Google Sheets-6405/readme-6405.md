Scrape Twitter Profiles with Bright Data API and Export to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-twitter-profiles-with-bright-data-api-and-export-to-google-sheets-6405


# Scrape Twitter Profiles with Bright Data API and Export to Google Sheets

### 1. Workflow Overview

This workflow automates scraping Twitter profile data using the Bright Data API and exports the collected data into a Google Sheet. It accepts user input via a form for a Twitter profile URL and a date range, triggers a Bright Data scraping job, monitors its progress, downloads the scraped data once ready, and appends it to a predefined Google Sheet for further analysis or reporting.

Logical blocks in the workflow are structured as follows:  
- **1.1 Input Reception:** Collect Twitter profile URL and date range via a form trigger.  
- **1.2 Scraping Trigger:** Send scraping request to Bright Data API with user inputs.  
- **1.3 Scraping Progress Monitoring:** Periodically check the scraping job status until completion.  
- **1.4 Data Retrieval:** Fetch the scraped data snapshot from Bright Data once ready.  
- **1.5 Data Storage:** Append the retrieved data to a Google Sheet with a specific schema.  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures user input through a web form, including the Twitter profile URL and a date range for scraping.

**Nodes Involved:**  
- 📥 User Input Trigger  
- Sticky Note (descriptive)

**Node Details:**  

- **📥 User Input Trigger**  
  - Type: Form Trigger  
  - Configuration: Form titled "Twitter profile url" with three fields:  
    - Twitter Profile url (text)  
    - start date (date)  
    - end date (date)  
  - Inputs: External user submission via webhook  
  - Outputs: JSON object containing form fields  
  - Edge Cases: Input validation not shown; improper URLs or date ranges might cause API errors downstream  
  - Notes: Starts the workflow on form submission  

- **Sticky Note**  
  - Content: "→ Starts flow with Twitter URL & date range from form"  
  - Role: Documentation only  

---

#### 2.2 Scraping Trigger

**Overview:**  
Sends a POST request to Bright Data API to initiate scraping of the provided Twitter profile URL over the specified date range.

**Nodes Involved:**  
- 🚀 Trigger Twitter Scraping  
- Sticky Note1

**Node Details:**  

- **🚀 Trigger Twitter Scraping**  
  - Type: HTTP Request (POST)  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/trigger`  
    - Body (JSON): Includes input array with URL, start_date, and end_date from user input; requests specific output fields  
    - Query Parameters: dataset_id, include_errors, type (discover_new), discover_by (profile_url)  
    - Headers: Authorization with Bearer token for Bright Data API (environment variable `BRIGHT_DATA_API_KEY`)  
  - Inputs: JSON from form trigger  
  - Outputs: JSON containing scraping job info including `snapshot_id` and status  
  - Edge Cases: API authentication failure, invalid URL or dates, API rate limits, malformed JSON expressions  
  - Notes: Initiates scraping job  

- **Sticky Note1**  
  - Content: "→ Sends scrape request to BrightData with user input"  

---

#### 2.3 Scraping Progress Monitoring

**Overview:**  
Periodically queries Bright Data API to check the status of the scraping job until it reports as "ready".

**Nodes Involved:**  
- 🔄 Monitor Scraping Progress  
- Sticky Note2  
- ⏱️ Delay Before Recheck  
- Sticky Note3  
- ✅ Is Scraping Ready? (IF)  
- Sticky Note4

**Node Details:**  

- **🔄 Monitor Scraping Progress**  
  - Type: HTTP Request (GET)  
  - Configuration:  
    - URL templated with `snapshot_id` from previous response:  
      `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
    - Headers: Authorization with Bright Data API token  
  - Inputs: From scraping trigger node (contains `snapshot_id`)  
  - Outputs: JSON with current scraping status  
  - Edge Cases: Network issues, invalid snapshot_id, authorization errors  

- **Sticky Note2**  
  - Content: "→ Checks if scraping is still running or ready"  

- **⏱️ Delay Before Recheck**  
  - Type: Wait node  
  - Configuration: Waits 1 minute before allowing next check  
  - Inputs: Output from progress monitor  
  - Edge Cases: Potential unnecessary delay if job finishes quickly or long wait if job stalls  

- **Sticky Note3**  
  - Content: "→ Waits 1 minute before checking status again"  

- **✅ Is Scraping Ready?**  
  - Type: If node  
  - Condition: Checks if `status` field equals "ready" (case-sensitive, strict validation)  
  - Inputs: From delay node  
  - Outputs:  
    - True branch: Proceed to fetch data  
    - False branch: Loop back to monitor progress node for rechecking  
  - Edge Cases: Unexpected or missing status field, infinite loop risk if status never changes  

- **Sticky Note4**  
  - Content: "→ If ready, fetch data; if not, repeat check loop"  

---

#### 2.4 Data Retrieval

**Overview:**  
Once the scraping job is complete, fetch the scraped Twitter data snapshot in JSON format.

**Nodes Involved:**  
- 📦 Fetch Twitter Data  
- Sticky Note5

**Node Details:**  

- **📦 Fetch Twitter Data**  
  - Type: HTTP Request (GET)  
  - Configuration:  
    - URL templated with `snapshot_id`:  
      `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
    - Query parameter `format=json`  
    - Headers: Authorization with Bright Data API token  
  - Inputs: From "Is Scraping Ready?" node (true branch)  
  - Outputs: JSON array of scraped tweets/posts data  
  - Edge Cases: Authorization issues, snapshot expiration, empty results  

- **Sticky Note5**  
  - Content: "→ Downloads scraped tweet data using snapshot ID"  

---

#### 2.5 Data Storage

**Overview:**  
Appends the scraped Twitter data to a Google Sheet, mapping fields to specific columns.

**Nodes Involved:**  
- 📊 Store Twitter Data in Google Sheet  
- Sticky Note6  
- Sticky Note8 (general instruction)

**Node Details:**  

- **📊 Store Twitter Data in Google Sheet**  
  - Type: Google Sheets node (append operation)  
  - Configuration:  
    - Document ID: Provided as Google Sheet URL or ID (placeholder `YOUR_GOOGLE_SHEET_ID`)  
    - Sheet Name: Default Sheet1 (gid=0)  
    - Mapping Mode: Explicit column mapping, matching on `id` field to avoid duplicates  
    - Columns mapped: id, user_posted, name, description, date_posted, photos, quoted_post, tagged_users, replies, reposts, likes, views, hashtags, followers, posts_count, profile_image_link, following, is_verified, quotes, external_image_urls, videos, external_video_urls, user_id, timestamp  
    - Credentials: Google Sheets OAuth2 required  
  - Inputs: Array of Twitter post data from fetch node  
  - Outputs: Confirmation or error of append operation  
  - Edge Cases: Credential expiration, sheet not found, column mismatch, data conversion issues  

- **Sticky Note6**  
  - Content: "→ Saves post data (likes, replies, user, etc.) into Sheet"  

- **Sticky Note8**  
  - Content:  
    "Create a Google Sheet with the following columns:  
    id | user_posted | name | description | date_posted | photos | quoted_post | tagged_users | replies | reposts | likes | views | hashtags | followers | posts_count | profile_image_link | following | is_verified | quotes | external_image_urls | videos | external_video_urls | user_id | timestamp"  

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                         | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                          |
|--------------------------------|--------------------|---------------------------------------|---------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| 📥 User Input Trigger           | Form Trigger       | Collect Twitter URL and date range     | (start node)                    | 🚀 Trigger Twitter Scraping     | → Starts flow with Twitter URL & date range from form                                              |
| Sticky Note                    | Sticky Note        | Documentation                         | None                            | None                           | → Starts flow with Twitter URL & date range from form                                              |
| 🚀 Trigger Twitter Scraping     | HTTP Request (POST)| Initiate scraping job at Bright Data  | 📥 User Input Trigger           | 🔄 Monitor Scraping Progress    | → Sends scrape request to BrightData with user input                                               |
| Sticky Note1                   | Sticky Note        | Documentation                         | None                            | None                           | → Sends scrape request to BrightData with user input                                               |
| 🔄 Monitor Scraping Progress    | HTTP Request (GET) | Check scraping job progress            | 🚀 Trigger Twitter Scraping     | ⏱️ Delay Before Recheck          | → Checks if scraping is still running or ready                                                     |
| Sticky Note2                   | Sticky Note        | Documentation                         | None                            | None                           | → Checks if scraping is still running or ready                                                     |
| ⏱️ Delay Before Recheck          | Wait               | Wait 1 minute before next status check | 🔄 Monitor Scraping Progress    | ✅ Is Scraping Ready?            | → Waits 1 minute before checking status again                                                      |
| Sticky Note3                   | Sticky Note        | Documentation                         | None                            | None                           | → Waits 1 minute before checking status again                                                      |
| ✅ Is Scraping Ready?           | If                 | Branch based on scraping job status   | ⏱️ Delay Before Recheck          | 📦 Fetch Twitter Data (true) / 🔄 Monitor Scraping Progress (false) | → If ready, fetch data; if not, repeat check loop                                                  |
| Sticky Note4                   | Sticky Note        | Documentation                         | None                            | None                           | → If ready, fetch data; if not, repeat check loop                                                  |
| 📦 Fetch Twitter Data            | HTTP Request (GET) | Retrieve scraped data snapshot         | ✅ Is Scraping Ready? (true)    | 📊 Store Twitter Data in Google Sheet | → Downloads scraped tweet data using snapshot ID                                                   |
| Sticky Note5                   | Sticky Note        | Documentation                         | None                            | None                           | → Downloads scraped tweet data using snapshot ID                                                   |
| 📊 Store Twitter Data in Google Sheet | Google Sheets     | Append scraped data to sheet           | 📦 Fetch Twitter Data           | None                           | → Saves post data (likes, replies, user, etc.) into Sheet                                          |
| Sticky Note6                   | Sticky Note        | Documentation                         | None                            | None                           | → Saves post data (likes, replies, user, etc.) into Sheet                                          |
| Sticky Note8                   | Sticky Note        | Google Sheet setup instruction        | None                            | None                           | Create a Google Sheet with the following columns: id, user_posted, name, description, date_posted, photos, quoted_post, tagged_users, replies, reposts, likes, views, hashtags, followers, posts_count, profile_image_link, following, is_verified, quotes, external_image_urls, videos, external_video_urls, user_id, timestamp |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Node Type: Form Trigger  
   - Configuration:  
     - Form Title: "Twitter profile url"  
     - Fields:  
       - "Twitter Profile url" (text)  
       - "start date" (date)  
       - "end date" (date)  
   - Purpose: Collect input from user to start the workflow  

2. **Create HTTP Request Node to Trigger Bright Data Scrape**  
   - Node Name: "🚀 Trigger Twitter Scraping"  
   - Node Type: HTTP Request (POST)  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Authentication: None in node, use header with Bearer token (credential `BRIGHT_DATA_API_KEY`)  
   - Headers:  
     - Authorization: `Bearer BRIGHT_DATA_API_KEY` (set as environment variable or credential)  
   - Query Parameters:  
     - dataset_id = "gd_lwxkxvnf1cynvib9co"  
     - include_errors = "true"  
     - type = "discover_new"  
     - discover_by = "profile_url"  
   - Body (JSON):  
     ```json
     {
       "input": [
         {
           "url": "{{ $json['Twitter Profile url'] }}",
           "start_date": "{{ $json['start date'] }}",
           "end_date": "{{ $json['end date'] }}"
         }
       ],
       "custom_output_fields": [
         "id", "user_posted", "name", "description", "date_posted", "photos", "url", "quoted_post",
         "tagged_users", "replies", "reposts", "likes", "views", "external_url", "hashtags", "followers",
         "posts_count", "profile_image_link", "following", "is_verified", "quotes", "parent_post_details",
         "external_image_urls", "videos", "external_video_urls", "user_id", "timestamp"
       ]
     }
     ```  
   - Connect output of Form Trigger to this node  

3. **Create HTTP Request Node to Monitor Scraping Progress**  
   - Node Name: "🔄 Monitor Scraping Progress"  
   - Node Type: HTTP Request (GET)  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Headers: Authorization Bearer token as above  
   - Connect output of Scraping Trigger node to this node  

4. **Create Wait Node**  
   - Node Name: "⏱️ Delay Before Recheck"  
   - Node Type: Wait  
   - Configuration: 1 minute delay  
   - Connect output of Progress Monitor node to this node  

5. **Create If Node to Check Scraping Status**  
   - Node Name: "✅ Is Scraping Ready?"  
   - Node Type: If  
   - Condition: Check if `{{ $json.status }}` equals `"ready"` (case-sensitive)  
   - True Output: Proceed to fetch data  
   - False Output: Loop back to Progress Monitor node to recheck status  

6. **Create HTTP Request Node to Fetch Scraped Data**  
   - Node Name: "📦 Fetch Twitter Data"  
   - Node Type: HTTP Request (GET)  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query Parameter: format=json  
   - Headers: Authorization Bearer token as above  
   - Connect True output of If node to this node  

7. **Create Google Sheets Node to Append Data**  
   - Node Name: "📊 Store Twitter Data in Google Sheet"  
   - Node Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your Google Sheet ID or URL  
   - Sheet Name: "Sheet1" or appropriate sheet  
   - Credentials: Google Sheets OAuth2 (configure OAuth2 credentials in n8n)  
   - Columns mapping: Explicitly map fields from fetched JSON to sheet columns as per list:  
     `id, user_posted, name, description, date_posted, photos, quoted_post, tagged_users, replies, reposts, likes, views, hashtags, followers, posts_count, profile_image_link, following, is_verified, quotes, external_image_urls, videos, external_video_urls, user_id, timestamp`  
   - Connect output of Fetch Twitter Data node to this node  

8. **Connect False Output of If Node to Progress Monitor Node**  
   - To keep checking until scraping is ready  

9. **Create Google Sheet**  
   - Create a Google Sheet with columns as listed above to match the data mapping exactly  

10. **Configure Environment Variables or Credentials**  
    - Set `BRIGHT_DATA_API_KEY` securely in n8n credentials  
    - Configure Google Sheets OAuth2 credential with access to the target spreadsheet  

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Bright Data API requires valid API key and may have rate limits or usage quotas. Ensure your account is properly configured.      | https://brightdata.com/api-docs                                                                                   |
| Google Sheets node requires OAuth2 credentials with edit access to the target sheet.                                               | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                                    |
| The workflow expects the Twitter profile URL to be valid and accessible by Bright Data’s scraping dataset.                         |                                                                                                                   |
| Ensure the Google Sheet columns exactly match the column names used in the mapping to avoid data misalignment.                    |                                                                                                                   |
| The waiting period between progress checks is set to 1 minute but can be adjusted based on expected scraping duration.            |                                                                                                                   |
| Potential errors include API authentication failures, network timeouts, invalid input data, or missing snapshot IDs.               |                                                                                                                   |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.