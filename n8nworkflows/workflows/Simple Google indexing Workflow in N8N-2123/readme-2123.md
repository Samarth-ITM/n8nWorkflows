Simple Google indexing Workflow in N8N

https://n8nworkflows.xyz/workflows/simple-google-indexing-workflow-in-n8n-2123


# Simple Google indexing Workflow in N8N

### 1. Workflow Overview

This workflow automates the process of submitting website URLs to Google for indexing using the Google Indexing API. It is designed to help website owners ensure their latest content is promptly discoverable by Google, thereby improving search visibility and SEO rankings.

**Primary Use Case:** Automatically fetch all URLs from a website’s sitemap and notify Google about updated URLs for indexing, either on-demand or on a scheduled basis.

**Logical Blocks:**

- **1.1 Input Reception:** Workflow triggers via manual execution or scheduled time.
- **1.2 Sitemap Retrieval and Parsing:** Fetch sitemap XML, convert to JSON, and parse individual URLs.
- **1.3 URL Preparation and Looping:** Prepare URLs and process them one by one in a loop.
- **1.4 Google Indexing API Submission:** Submit URLs to the Google Indexing API and handle the response.
- **1.5 Rate Limiting and Error Handling:** Wait between requests and stop the workflow with an error if the daily quota is exceeded.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Provides two ways to start the workflow: manually via button or automatically on a schedule.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)  
- Schedule Trigger

**Node Details:**

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Starts workflow on user command  
  - Configuration: Default (no parameters)  
  - Connections: Output → sitemap_set  
  - Edge Cases: None, but user must manually initiate

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically starts the workflow daily at 1 AM  
  - Configuration: Interval set to trigger at hour 1 (1 AM daily)  
  - Connections: Output → sitemap_set  
  - Edge Cases: Time zone considerations; scheduled time must align with desired crawling frequency

---

#### 1.2 Sitemap Retrieval and Parsing

**Overview:**  
Fetches the sitemap XML from the configured URL, converts it to JSON, then splits it into individual URLs.

**Nodes Involved:**  
- sitemap_set (HTTP Request)  
- sitemap_convert (XML to JSON conversion)  
- sitemap_parse (Split JSON array of URLs)

**Node Details:**

- **sitemap_set**  
  - Type: HTTP Request  
  - Role: Fetches sitemap XML from the specified URL  
  - Configuration: URL set to `https://bushidogym.fr/sitemap.xml` (user must replace with their sitemap URL)  
  - Connections: Output → sitemap_convert  
  - Edge Cases: Invalid URL, network failure, sitemap not accessible, HTTP errors (e.g., 404)  

- **sitemap_convert**  
  - Type: XML node  
  - Role: Converts fetched XML sitemap to JSON  
  - Configuration: Options enabled for trimming, normalizing, merging attributes, ignoring attributes, and normalizing tags to ensure clean JSON  
  - Connections: Output → sitemap_parse  
  - Edge Cases: Malformed XML, conversion errors  

- **sitemap_parse**  
  - Type: Split Out  
  - Role: Splits the JSON array under `urlset.url` into individual URL items for processing  
  - Configuration: Splits on field `urlset.url`, outputs each URL object under the field name `url`  
  - Connections: Output → url_set  
  - Edge Cases: Empty or missing URL list, unexpected JSON structure

---

#### 1.3 URL Preparation and Looping

**Overview:**  
Extracts the URL string from each JSON object and prepares it for submission, then processes URLs one at a time.

**Nodes Involved:**  
- url_set (Set)  
- loop (SplitInBatches)

**Node Details:**

- **url_set**  
  - Type: Set  
  - Role: Extracts the `loc` property from each URL JSON object and sets it as `url` field for downstream processing  
  - Configuration: Sets `url` to expression `{{$json.url.loc}}`; keeps only this field  
  - Connections: Output → loop  
  - Edge Cases: Missing `.loc` property, malformed URL  

- **loop**  
  - Type: SplitInBatches  
  - Role: Processes URLs one by one to respect API quotas and allow for delays between requests  
  - Configuration: Batch size set to 1 (process one URL at a time)  
  - Connections: Output → url_index (HTTP Request to Google API)  
  - Edge Cases: Large sitemaps could cause long processing times

---

#### 1.4 Google Indexing API Submission

**Overview:**  
Sends a POST request to Google’s Indexing API for each URL, indicates the URL is updated, and checks the response.

**Nodes Involved:**  
- url_index (HTTP Request)  
- index_check (If)  
- wait (Wait)

**Node Details:**

- **url_index**  
  - Type: HTTP Request  
  - Role: Submits URL update notification to Google Indexing API  
  - Configuration:  
    - URL: `https://indexing.googleapis.com/v3/urlNotifications:publish`  
    - Method: POST  
    - Authentication: Google API Service Account credentials (predefined credential type)  
    - Body Parameters: JSON object with `url` (from current item) and `type` set to `"URL_UPDATED"`  
    - Continue On Fail: Enabled to allow workflow to handle errors gracefully  
    - Always Output Data: Enabled to pass response to next node  
  - Connections: Output → index_check  
  - Edge Cases: Auth errors if credentials invalid or expired, quota exceeded, network timeouts, API errors  

- **index_check**  
  - Type: If  
  - Role: Checks if the API response indicates the URL was successfully updated  
  - Configuration: Compares `$json.urlNotificationMetadata.latestUpdate.type` to string `"URL_UPDATED"`  
  - Connections:  
    - True → wait  
    - False → Stop and Error  
  - Edge Cases: Unexpected response format, missing fields, rate limit exceeded signaled by false condition  

- **wait**  
  - Type: Wait  
  - Role: Pauses the workflow for 2 seconds before processing the next URL to avoid hitting API rate limits  
  - Configuration: Wait time set to 2 seconds  
  - Connections: Output → loop (to process next batch)  
  - Edge Cases: Delays add to workflow runtime, possible webhook ID present but not used here explicitly  

---

#### 1.5 Rate Limiting and Error Handling

**Overview:**  
Stops the workflow with an error message if the API daily quota limit is reached or if the URL update was not acknowledged.

**Nodes Involved:**  
- Stop and Error

**Node Details:**

- **Stop and Error**  
  - Type: Stop and Error  
  - Role: Terminates the workflow with a specific error message when the Google Indexing API quota is exceeded or update failed  
  - Configuration: Error message set to `"You have reached the Google Indexing API limit (200/day by default)"`  
  - Connections: None (end node)  
  - Edge Cases: Workflow termination; no retries unless manually restarted  

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                             | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                   |
|------------------------------|---------------------|--------------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger      | Manual workflow start                       | None                             | sitemap_set                     |                                                                                               |
| Schedule Trigger             | Schedule Trigger    | Scheduled workflow start at 1 AM            | None                             | sitemap_set                     |                                                                                               |
| sitemap_set                 | HTTP Request       | Fetch sitemap XML from website URL           | When clicking "Execute Workflow", Schedule Trigger | sitemap_convert                |                                                                                               |
| sitemap_convert             | XML                 | Convert sitemap XML to JSON                   | sitemap_set                     | sitemap_parse                   |                                                                                               |
| sitemap_parse               | Split Out           | Split JSON sitemap URLs into individual items | sitemap_convert                 | url_set                        |                                                                                               |
| url_set                    | Set                 | Extract and prepare URL field for API        | sitemap_parse                   | loop                           |                                                                                               |
| loop                       | SplitInBatches      | Process URLs one by one                       | url_set                        | url_index                      |                                                                                               |
| url_index                  | HTTP Request       | POST URL update notification to Google API   | loop                           | index_check                   |                                                                                               |
| index_check                | If                  | Verify if URL update was successful           | url_index                      | wait (if true), Stop and Error (if false) |                                                                                               |
| wait                       | Wait                | Wait 2 seconds between API calls              | index_check                   | loop                          |                                                                                               |
| Stop and Error             | Stop and Error      | Stops workflow on quota exceeded or error     | index_check                   | None                          |                                                                                               |
| Sticky Note                | Sticky Note         | Documentation and author note                  | None                          | None                          | ## Simple indexing workflow using the Google Indexing API\n\nThis workflow is the simplest indexing workflow. It simply extracts a sitemap, converts it to a JSON, and loops through each URL. It will output an error if your quota is reached.\n\n*Joachim* |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking "Execute Workflow"`  
   - Type: Manual Trigger  
   - Leave default configuration

2. **Create Schedule Trigger Node**  
   - Name: `Schedule Trigger`  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 1 AM (set interval to `triggerAtHour: 1`)

3. **Create HTTP Request Node for Sitemap**  
   - Name: `sitemap_set`  
   - Type: HTTP Request  
   - Set URL to your sitemap, e.g., `https://yourdomain.com/sitemap.xml`  
   - Leave other options default  
   - Connect outputs of both trigger nodes (`When clicking "Execute Workflow"` and `Schedule Trigger`) to this node’s input

4. **Create XML Node to Convert Sitemap**  
   - Name: `sitemap_convert`  
   - Type: XML  
   - Enable options: trim, normalize, merge attributes, ignore attributes, normalize tags  
   - Connect `sitemap_set` output to this node’s input

5. **Create Split Out Node to Extract URLs**  
   - Name: `sitemap_parse`  
   - Type: Split Out  
   - Configure to split on field: `urlset.url`  
   - Destination field name: `url`  
   - Connect `sitemap_convert` output to this node’s input

6. **Create Set Node to Extract URL String**  
   - Name: `url_set`  
   - Type: Set  
   - Add a string field named `url` with value expression: `{{$json.url.loc}}`  
   - Enable `Keep Only Set` option to discard other fields  
   - Connect `sitemap_parse` output to this node’s input

7. **Create SplitInBatches Node to Loop URLs**  
   - Name: `loop`  
   - Type: SplitInBatches  
   - Set batch size to 1 (to process one URL at a time)  
   - Connect `url_set` output to this node’s input

8. **Create HTTP Request Node for Google Indexing API**  
   - Name: `url_index`  
   - Type: HTTP Request  
   - Set URL: `https://indexing.googleapis.com/v3/urlNotifications:publish`  
   - Method: POST  
   - Authentication: Use Google API OAuth2 credentials configured with your Google Cloud Service Account JSON key  
   - Body parameters (JSON):  
     ```json
     {
       "url": "={{ $json.url }}",
       "type": "URL_UPDATED"
     }
     ```  
   - Enable `Continue On Fail` and `Always Output Data` options  
   - Connect `loop` output to this node’s input

9. **Create If Node to Check API Response**  
   - Name: `index_check`  
   - Type: If  
   - Condition: Check if `$json.urlNotificationMetadata.latestUpdate.type` equals `"URL_UPDATED"`  
   - Connect `url_index` output to this node’s input

10. **Create Wait Node to Pause Between Requests**  
    - Name: `wait`  
    - Type: Wait  
    - Set wait time to 2 seconds  
    - Connect `index_check` True output to this node’s input

11. **Connect Wait Node Back to Loop**  
    - Connect `wait` output back to `loop` input to continue processing next URL

12. **Create Stop and Error Node for Quota Limit**  
    - Name: `Stop and Error`  
    - Type: Stop and Error  
    - Set error message: `"You have reached the Google Indexing API limit (200/day by default)"`  
    - Connect `index_check` False output to this node’s input

13. **Credentials Setup:**  
    - Create and configure Google API credentials in n8n using your Google Service Account JSON key  
    - Assign these credentials to the `url_index` node’s authentication

14. **Final Setup:**  
    - Validate all connections and node configurations  
    - Update sitemap URL in `sitemap_set` node to your website’s sitemap  
    - Test manually using the manual trigger or wait for scheduled run

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Activate the Google Indexing API and create a Service Account with JSON key for authentication.  | Google Cloud Platform Console: https://console.cloud.google.com/apis/library/indexing.googleapis.com     |
| Add the Service Account email as an owner in Google Search Console for your site to grant access. | Google Search Console ownership settings                                                                |
| Replace the sitemap URL in the `sitemap_set` node with your actual sitemap URL.                    | Example URL format: `https://yourdomain.com/sitemap.xml`                                                 |
| Default Google Indexing API quota is 200 URL notifications per day; hitting this limit triggers error. | Workflow is designed to stop and notify when quota is reached                                             |
| Waiting 2 seconds between requests helps avoid hitting rate limits and API throttling.            |                                                                                                          |
| Sticky Note in workflow provides a summary and author credit: "Joachim"                           |                                                                                                          |

---

This documentation fully describes the workflow structure, node roles, configurations, and how to recreate the workflow independently. It also points out potential failure points like authentication errors, quota limits, and network issues, allowing users to anticipate and troubleshoot problems effectively.