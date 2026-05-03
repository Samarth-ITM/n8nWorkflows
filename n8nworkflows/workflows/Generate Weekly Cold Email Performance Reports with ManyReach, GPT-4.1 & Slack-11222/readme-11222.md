Generate Weekly Cold Email Performance Reports with ManyReach, GPT-4.1 & Slack

https://n8nworkflows.xyz/workflows/generate-weekly-cold-email-performance-reports-with-manyreach--gpt-4-1---slack-11222


# Generate Weekly Cold Email Performance Reports with ManyReach, GPT-4.1 & Slack

### 1. Workflow Overview

This workflow automates the generation and distribution of weekly cold email campaign performance reports based on data from ManyReach. It targets marketing or sales teams aiming to monitor and optimize cold email outreach efforts by leveraging AI analysis and streamlined reporting.

The workflow includes the following logical blocks:

- **1.1 Weekly Trigger & Campaign Fetching:** Automatically triggers every week to fetch the list of campaigns from ManyReach.
- **1.2 Campaign Filtering:** Filters campaigns to select only those that are both active and completed.
- **1.3 Campaign Data Loop:** Iterates over filtered campaigns to process each individually.
- **1.4 Detailed Campaign Fetch:** Fetches detailed data for each campaign.
- **1.5 AI-Driven Campaign Analysis:** Uses GPT-4.1 to generate a detailed performance report from campaign metrics.
- **1.6 Report Formatting:** Converts AI-generated Markdown reports to HTML, then formats the HTML into a Google Docs-compatible MIME multipart upload.
- **1.7 Report Upload & Notification:** Uploads the report to Google Drive and sends a link to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Weekly Trigger & Campaign Fetching

- **Overview:** Initiates the workflow on a weekly schedule and retrieves all campaigns from ManyReach.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch All Campaign  
  - Split Out  
- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Trigger workflow once weekly (every Monday)  
    - Configuration: Set to trigger weekly on Monday (field: weeks, triggerAtDay: 1)  
    - Inputs: None (start node)  
    - Outputs: Fetch All Campaign  
    - Edge cases: Trigger misconfiguration could cause no execution; time zone considerations.

  - **Fetch All Campaign**  
    - Type: HTTP Request  
    - Role: Calls ManyReach API to retrieve campaigns  
    - Configuration: GET https://app.manyreach.com/api/campaigns with `limit=100` query parameter  
    - Auth: HTTP Query Auth with ManyReach credentials  
    - Inputs: Trigger output  
    - Outputs: Split Out node  
    - Edge cases: API rate limits, authentication failure, network errors, empty dataset.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits campaign array from API `data` field into individual items  
    - Configuration: Splits on `data` field  
    - Inputs: Campaigns array JSON  
    - Outputs: Filter Active & Completed Campaign  
    - Edge cases: Empty or malformed data field.

---

#### 2.2 Campaign Filtering

- **Overview:** Filters the split campaigns to retain only those that are active and have a status of "completed".
- **Nodes Involved:**  
  - Filter Active & Completed Campaign  
- **Node Details:**

  - **Filter Active & Completed Campaign**  
    - Type: Filter  
    - Role: Filters campaigns based on Boolean `active` = true and string `campStatus` = "completed"  
    - Configuration: Conditions combined with AND: `active == true` AND `campStatus == "completed"`  
    - Inputs: Individual campaign items from Split Out  
    - Outputs: Loop Over Items (start batch processing)  
    - Edge cases: Missing or null fields `active` or `campStatus` could cause filter logic failures.

---

#### 2.3 Campaign Data Loop

- **Overview:** Processes campaigns one by one in batches to manage API calls and workflow performance.
- **Nodes Involved:**  
  - Loop Over Items  
- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes campaigns individually or in batches  
    - Configuration: Default batch size (not explicitly set)  
    - Inputs: Filtered campaigns  
    - Outputs: Fetch One Campaign (batch index 1), empty outputs for batch index 0  
    - Edge cases: Large batch size might cause timeouts, API limits.

---

#### 2.4 Detailed Campaign Fetch

- **Overview:** Retrieves detailed metrics for each campaign to feed into AI analysis.
- **Nodes Involved:**  
  - Fetch One Campaign  
- **Node Details:**

  - **Fetch One Campaign**  
    - Type: HTTP Request  
    - Role: Gets detailed campaign data by campaign ID  
    - Configuration: GET https://app.manyreach.com/api/campaigns/{{ $json.campaignID }}  
    - Auth: HTTP Query Auth (ManyReach)  
    - Inputs: Campaign batch item from Loop Over Items  
    - Outputs: Campaign Report Agent (AI analysis)  
    - Edge cases: API failure, missing campaign ID, data inconsistency.

---

#### 2.5 AI-Driven Campaign Analysis

- **Overview:** Uses GPT-4.1 to analyze campaign data comprehensively and generate a markdown report.
- **Nodes Involved:**  
  - Campaign Report Agent  
  - 4.1 (GPT-4.1 Language Model)  
- **Node Details:**

  - **Campaign Report Agent**  
    - Type: Langchain Agent  
    - Role: Formats campaign data into a detailed prompt for AI analysis  
    - Configuration: Supplies extensive campaign metrics and settings in the prompt text for analysis; requests a structured markdown report with executive summary, metrics, content analysis, recommendations, and next steps  
    - Inputs: Detailed campaign data JSON from Fetch One Campaign  
    - Outputs: GPT-4.1 node (via ai_languageModel connection)  
    - Edge cases: Missing or malformed data fields; prompt length limits.

  - **4.1**  
    - Type: Langchain LM Chat OpenAI  
    - Role: Executes GPT-4.1 model for report generation  
    - Configuration: Model set to "gpt-4.1"; no special options  
    - Auth: OpenAI API credentials  
    - Inputs: Campaign Report Agent prompt output  
    - Outputs: Markdown -> HTML node  
    - Edge cases: API rate limits, quota exhaustion, network errors, response timeouts.

---

#### 2.6 Report Formatting

- **Overview:** Converts AI-generated markdown to HTML and then prepares the content for Google Docs upload.
- **Nodes Involved:**  
  - Markdown -> HTML  
  - Set Details  
  - HTML -> Magic 🪄 (custom code)  
- **Node Details:**

  - **Markdown -> HTML**  
    - Type: Markdown  
    - Role: Converts AI markdown output to HTML with emoji and table support enabled  
    - Configuration: Mode set to markdownToHtml, emoji and tables enabled  
    - Inputs: GPT-4.1 output markdown text  
    - Outputs: Set Details  
    - Edge cases: Malformed markdown could cause conversion issues.

  - **Set Details**  
    - Type: Set  
    - Role: Assigns key variables for document creation, including document name, HTML content, and Google Drive folder ID (user must set folder ID here)  
    - Configuration:  
      - `document_name`: Placeholder text "dfdfdfdf" (should be customized)  
      - `html_content`: HTML content from Markdown -> HTML node  
      - `drive_folder_id`: Empty string by default (user sets folder ID)  
    - Inputs: Markdown -> HTML output  
    - Outputs: HTML -> Magic 🪄  
    - Edge cases: Missing or incorrect folder ID will affect upload.

  - **HTML -> Magic 🪄**  
    - Type: Code (JavaScript)  
    - Role: Builds a multipart MIME body for Google Docs API upload with HTML and metadata, including inline CSS styles for proper formatting  
    - Configuration: Uses document name, HTML content; commented out folder ID usage (should be enabled if folder ID is set)  
    - Inputs: Set Details output  
    - Outputs: Upload Doc HTTP Request  
    - Edge cases: JavaScript errors, missing content, boundary formatting issues.

---

#### 2.7 Report Upload & Notification

- **Overview:** Uploads the Google Docs report to Drive and posts the document link to a Slack channel.
- **Nodes Involved:**  
  - Upload Doc  
  - Send a Doc Link  
- **Node Details:**

  - **Upload Doc**  
    - Type: HTTP Request  
    - Role: Uploads the Google Doc using Drive API multipart upload  
    - Configuration:  
      - POST to https://www.googleapis.com/upload/drive/v3/files  
      - Content-Type: multipart/related; boundary=divider  
      - Query parameters: uploadType=multipart, supportsAllDrives=true  
      - Body: raw multipart data from HTML -> Magic 🪄  
      - Auth: Google Drive OAuth2  
    - Inputs: HTML -> Magic 🪄 output  
    - Outputs: Send a Doc Link  
    - Edge cases: Auth failures, quota limits, API errors, malformed request body.

  - **Send a Doc Link**  
    - Type: Slack  
    - Role: Sends message to Slack channel with link to the generated Google Doc report  
    - Configuration:  
      - Text includes campaign name from "Fetch One Campaign" node and a Docs URL with file ID from Upload Doc output  
      - Slack channel selected by ID (dynamic list with cached channel "manyreach")  
      - Markdown enabled, no workflow link included  
      - Auth: Slack OAuth2  
    - Inputs: Upload Doc output  
    - Outputs: Loop Over Items (continues processing next campaign)  
    - Edge cases: Slack API rate limits, invalid channel, missing file ID.

---

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                          | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                               |
|---------------------------|--------------------------------|----------------------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------------|
| Sticky Note1              | Sticky Note                    | Workflow overview and instructions     | None                  | None                     | 📧 ManyReach Weekly Cold Email Campaign Report📃 ... Setup steps and workflow description (large note)  |
| Schedule Trigger          | Schedule Trigger               | Weekly trigger to start workflow       | None                  | Fetch All Campaign       | ## 1. Weekly Trigger ... Fetch All Campaigns every week and fetched campaign used for further nodes      |
| Fetch All Campaign        | HTTP Request                  | Fetch all campaigns from ManyReach API | Schedule Trigger      | Split Out                |                                                                                                          |
| Split Out                 | Split Out                     | Split campaign array into individual   | Fetch All Campaign    | Filter Active & Completed Campaign | ## 2. Filter Data ... Process Only Active & also Completed Campaign for Generate Report              |
| Filter Active & Completed Campaign | Filter                       | Filter active and completed campaigns  | Split Out             | Loop Over Items          |                                                                                                          |
| Loop Over Items           | Split In Batches              | Batch processing of campaigns          | Filter Active & Completed Campaign | Fetch One Campaign (batch 1) | ## 3. Loop for ... It process all Campaign individually                                                  |
| Fetch One Campaign        | HTTP Request                  | Fetch detailed campaign data by ID     | Loop Over Items       | Campaign Report Agent    |                                                                                                          |
| Campaign Report Agent     | Langchain Agent               | Compose AI prompt for campaign analysis| Fetch One Campaign    | 4.1 (GPT-4.1)            |                                                                                                          |
| 4.1                      | Langchain LM Chat OpenAI      | Generate markdown report with GPT-4.1  | Campaign Report Agent | Markdown -> HTML         |                                                                                                          |
| Markdown -> HTML          | Markdown                      | Convert markdown report to HTML         | 4.1                   | Set Details              |                                                                                                          |
| Set Details               | Set                          | Set document name, HTML content, folder| Markdown -> HTML      | HTML -> Magic 🪄         |                                                                                                          |
| HTML -> Magic 🪄          | Code                         | Prepare multipart MIME body for upload | Set Details            | Upload Doc               |                                                                                                          |
| Upload Doc                | HTTP Request                  | Upload Google Doc to Drive              | HTML -> Magic 🪄       | Send a Doc Link          |                                                                                                          |
| Send a Doc Link           | Slack                        | Send document link notification to Slack | Upload Doc           | Loop Over Items          |                                                                                                          |
| Sticky Note2              | Sticky Note                   | Note for weekly trigger block           | None                  | None                     | ## 1. Weekly Trigger ... Fetch All Campaigns every week and fetched campaign used for further nodes      |
| Sticky Note3              | Sticky Note                   | Note for filter data block               | None                  | None                     | ## 2. Filter Data ... Process Only Active & also Completed Campaign for Generate Report                   |
| Sticky Note4              | Sticky Note                   | Note for looping block                   | None                  | None                     | ## 3. Loop for ... It process all Campaign individually                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set rule to trigger weekly on Monday (field: weeks, triggerAtDay: 1).  
   - Connect output to Fetch All Campaign node.

2. **Create Fetch All Campaign Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://app.manyreach.com/api/campaigns  
   - Query Parameter: limit=100  
   - Authentication: HTTP Query Auth with ManyReach credentials (set in n8n credentials panel).  
   - Connect output to Split Out node.

3. **Create Split Out Node**  
   - Type: Split Out  
   - Field to split: data (from API response)  
   - Connect output to Filter Active & Completed Campaign node.

4. **Create Filter Active & Completed Campaign Node**  
   - Type: Filter  
   - Conditions (AND):  
     - active equals true (boolean)  
     - campStatus equals "completed" (string)  
   - Connect output to Loop Over Items node.

5. **Create Loop Over Items Node**  
   - Type: Split In Batches  
   - Default batch size (optional: adjust for performance)  
   - Connect batch index 1 output to Fetch One Campaign node.

6. **Create Fetch One Campaign Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://app.manyreach.com/api/campaigns/{{ $json.campaignID }} (expression)  
   - Authentication: HTTP Query Auth with ManyReach credentials  
   - Connect output to Campaign Report Agent node.

7. **Create Campaign Report Agent Node**  
   - Type: Langchain Agent  
   - Set prompt text with detailed campaign data placeholders (see node details above)  
   - Use a system message defining the task for email campaign analysis and output report format in markdown.  
   - Connect output to GPT-4.1 node.

8. **Create GPT-4.1 Node (4.1)**  
   - Type: Langchain LM Chat OpenAI  
   - Model: gpt-4.1  
   - Credentials: OpenAI API credentials  
   - Connect output to Markdown -> HTML node.

9. **Create Markdown -> HTML Node**  
   - Type: Markdown  
   - Mode: markdownToHtml  
   - Enable emoji and tables options  
   - Connect output to Set Details node.

10. **Create Set Details Node**  
    - Type: Set  
    - Assign variables:  
      - document_name: set a meaningful name (e.g., "Weekly Campaign Report - {{ $json.data.name }}")  
      - html_content: set to output HTML from previous node (`={{ $json.data }}`)  
      - drive_folder_id: set your Google Drive folder ID as string  
    - Connect output to HTML -> Magic 🪄 node.

11. **Create HTML -> Magic 🪄 Node**  
    - Type: Code (JavaScript)  
    - Paste provided JS code that constructs multipart MIME message for Google Docs API upload using document_name, html_content, and drive_folder_id.  
    - (Optional) Uncomment and use drive_folder_id for setting parents in metadata.  
    - Connect output to Upload Doc node.

12. **Create Upload Doc Node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: https://www.googleapis.com/upload/drive/v3/files  
    - Content-Type: multipart/related; boundary=divider  
    - Query Parameters: uploadType=multipart, supportsAllDrives=true  
    - Body: raw data from previous code node  
    - Authentication: Google Drive OAuth2 credentials  
    - Connect output to Send a Doc Link node.

13. **Create Send a Doc Link Node**  
    - Type: Slack  
    - Message Text:  
      ```
      Campaign: {{ $('Fetch One Campaign').item.json.data.name }}

      Report Link: https://docs.google.com/document/d/{{ $json.id }}
      ```  
    - Channel: select your Slack channel by ID (use channel picker or hardcode)  
    - Options: Enable markdown, disable workflow link  
    - Authentication: Slack OAuth2 credentials  
    - Connect output back to Loop Over Items node to process next campaign.

14. **Credentials Setup**  
    - Configure ManyReach API credentials as HTTP Query Auth with required tokens or keys.  
    - Configure OpenAI credentials with API key.  
    - Configure Google Drive OAuth2 credentials with appropriate scopes for file upload.  
    - Configure Slack OAuth2 credentials with permissions for posting messages to channels.

15. **Final Configuration**  
    - Set `drive_folder_id` in Set Details node to your target Google Drive folder ID.  
    - Set Slack channel in Send a Doc Link node.  
    - Test the workflow manually or wait for scheduled trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Workflow automates weekly cold email campaign reporting using ManyReach data, GPT-4.1 AI analysis, Google Docs report generation, and Slack notifications.                                                                     | Workflow description, Sticky Note1                                                                              |
| User must ensure ManyReach API URLs match their regional account endpoint if different from default.                                                                                                                            | Sticky Note1                                                                                                    |
| Google Docs generation uses multipart HTTP upload with embedded styles for proper formatting.                                                                                                                                    | HTML -> Magic 🪄 node code                                                                                        |
| Slack message includes direct Google Docs link for easy team access.                                                                                                                                                             | Send a Doc Link node                                                                                             |
| Setup requires credentials for ManyReach API, OpenAI API, Google Drive OAuth2, and Slack OAuth2.                                                                                                                                | Sticky Note1 and node credentials details                                                                       |
| Industry benchmarks cited for cold email campaigns: Opens 40-50%, Replies 2-5%, Conversions 1-3%.                                                                                                                               | Campaign Report Agent system message                                                                              |

---

**Disclaimer:** The provided text is derived solely from an n8n automated workflow. The process strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.