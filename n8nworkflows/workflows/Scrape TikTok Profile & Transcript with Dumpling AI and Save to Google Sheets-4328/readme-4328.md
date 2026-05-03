Scrape TikTok Profile & Transcript with Dumpling AI and Save to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-tiktok-profile---transcript-with-dumpling-ai-and-save-to-google-sheets-4328


# Scrape TikTok Profile & Transcript with Dumpling AI and Save to Google Sheets

### 1. Workflow Overview

This workflow automates the process of monitoring a Google Sheet for new TikTok video URLs, extracting profile data and video transcripts via Dumpling AI APIs, and saving the aggregated information back to the Google Sheet. It is designed for content analysts, social media managers, or data teams who want to collect TikTok user statistics and video transcripts programmatically without manual scraping.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception**: Watches a Google Sheet for new TikTok video URLs.
- **1.2 Username Extraction**: Parses the TikTok username from the video URL using a regular expression.
- **1.3 Profile Data Retrieval**: Queries Dumpling AI to get TikTok profile statistics (followers, likes, etc.) using the extracted username.
- **1.4 Video Transcript Retrieval**: Requests the video transcript from Dumpling AI using the full video URL.
- **1.5 Data Persistence**: Appends the username, profile stats, video URL, and transcript back into the Google Sheet for record-keeping.

This modular approach ensures clear data flow from input detection through data enrichment to final storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new TikTok video URL is added to a specified Google Sheet. It uses a polling mechanism to check for new rows every minute.

- **Nodes Involved:**  
  - Watch for New TikTok Links in Sheet

- **Node Details:**  
  - **Name:** Watch for New TikTok Links in Sheet  
  - **Type:** Google Sheets Trigger  
  - **Role:** Initiates workflow execution upon detection of a newly added row.  
  - **Configuration:**  
    - Event: `rowAdded`  
    - Polling frequency: every 1 minute  
    - Target Sheet: Document ID linked to a Google Sheet titled “Videos”, Sheet Name: “Sheet1”  
    - Credentials: OAuth2 for Google Sheets Trigger  
  - **Inputs:** None (trigger node)  
  - **Outputs:** JSON containing the new row data, expected to contain a field named `USERNAME Video` with the TikTok URL.  
  - **Edge Cases / Failure Modes:**  
    - Google Sheets API rate limits or auth expiration could stop triggers.  
    - Misconfigured document or sheet names lead to silent failures or no triggers.  
  - **Version Requirements:** n8n Google Sheets Trigger v1  

#### 2.2 Username Extraction

- **Overview:**  
  Extracts the TikTok username (handle) from the full video URL using a JavaScript regular expression.

- **Nodes Involved:**  
  - Extract Username from TikTok URL

- **Node Details:**  
  - **Name:** Extract Username from TikTok URL  
  - **Type:** Set Node (variable assignment)  
  - **Role:** Parses username from URL string to feed into the profile data API.  
  - **Configuration:**  
    - Uses expression:  
      ```js
      {{$json["USERNAME Video"] && $json["USERNAME Video"].match(/@([^\/]+)/) ? $json["USERNAME Video"].match(/@([^\/]+)/)[1] : null}}
      ```  
    - Assigns result to variable `USERNAME`  
  - **Input:** JSON with `USERNAME Video` field containing TikTok URL  
  - **Output:** JSON with new field `USERNAME` containing extracted handle or null if no match  
  - **Edge Cases / Failure Modes:**  
    - Invalid or malformed URLs that don't contain a username matching `@handle` pattern will result in `USERNAME` = null, leading to downstream API calls with invalid inputs.  
    - Expression errors if input field is missing or not string.  
  - **Version Requirements:** Set Node v3.4  

#### 2.3 Profile Data Retrieval

- **Overview:**  
  Sends the extracted username to Dumpling AI API to obtain TikTok profile statistics such as follower count, following count, total likes (hearts), and video count.

- **Nodes Involved:**  
  - Get TikTok Profile Data using Dumpling AI

- **Node Details:**  
  - **Name:** Get TikTok Profile Data using Dumpling AI  
  - **Type:** HTTP Request  
  - **Role:** Queries external API with username to get profile data.  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/get-tiktok-profile`  
    - Body (JSON): `{ "handle": "{{ $json.USERNAME }}" }`  
    - Authentication: HTTP header auth with n8n credential named `n8n`  
  - **Input:** JSON with `USERNAME` field  
  - **Output:** JSON with profile stats under `stats` field (e.g., followerCount, heart, followingCount, videoCount)  
  - **Edge Cases / Failure Modes:**  
    - Null or invalid `USERNAME` leads to API errors or empty responses.  
    - HTTP errors (timeouts, auth failures, rate limiting).  
    - API changes or downtime.  
  - **Version Requirements:** HTTP Request v4.2  

#### 2.4 Video Transcript Retrieval

- **Overview:**  
  Uses Dumpling AI to obtain the transcript text for the TikTok video URL.

- **Nodes Involved:**  
  - Get TikTok Video Transcript using Dumpling AI

- **Node Details:**  
  - **Name:** Get TikTok Video Transcript using Dumpling AI  
  - **Type:** HTTP Request  
  - **Role:** Fetches transcript for the specific video URL.  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/get-tiktok-transcript`  
    - Body (JSON): `{ "videoUrl": "{{ $('Watch for New TikTok Links in Sheet').item.json['USERNAME Video'] }}" }`  
    - Authentication: HTTP header auth with same credential as profile data node  
  - **Input:** Original TikTok video URL from trigger node  
  - **Output:** JSON containing `transcript` field with video transcript text  
  - **Edge Cases / Failure Modes:**  
    - Invalid or inaccessible video URL leads to empty or error responses.  
    - API failures, auth errors, or rate limiting.  
  - **Version Requirements:** HTTP Request v4.2  

#### 2.5 Data Persistence

- **Overview:**  
  Appends all collected data — username, profile stats, original video URL, and transcript — back into the Google Sheet for ongoing tracking and analysis.

- **Nodes Involved:**  
  - Save Profile Stats and Transcript to Google Sheet

- **Node Details:**  
  - **Name:** Save Profile Stats and Transcript to Google Sheet  
  - **Type:** Google Sheets node (append operation)  
  - **Role:** Writes data row to sheet with predefined columns.  
  - **Configuration:**  
    - Operation: Append  
    - Document ID and Sheet Name: same Google Sheet as trigger node  
    - Columns mapped explicitly:  
      - Username: from `Extract Username from TikTok URL` node’s `USERNAME`  
      - Transcript: from transcript node’s `transcript` field  
      - Video Count, Heart Count, Follower Count, Following Count: from profile stats node’s `stats` object  
      - USERNAME Video: original video URL from trigger node  
    - Mapping mode: manual column definition  
    - Credentials: OAuth2 Google Sheets account  
  - **Input:** JSON containing all required fields from upstream nodes  
  - **Output:** Confirmation of append success  
  - **Edge Cases / Failure Modes:**  
    - Google Sheets API quota or permission errors  
    - Data type mismatches or missing fields causing incomplete rows  
  - **Version Requirements:** Google Sheets node v4.5  

---

### 3. Summary Table

| Node Name                              | Node Type               | Functional Role                          | Input Node(s)                          | Output Node(s)                              | Sticky Note                                                                                                                                |
|--------------------------------------|-------------------------|----------------------------------------|--------------------------------------|--------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Watch for New TikTok Links in Sheet  | Google Sheets Trigger   | Detects new TikTok video URLs in sheet| None                                 | Extract Username from TikTok URL            | 🎯 Workflow Purpose: Monitors Google Sheet for new TikTok links, triggers full workflow                                                      |
| Extract Username from TikTok URL      | Set                     | Extracts TikTok username from URL      | Watch for New TikTok Links in Sheet  | Get TikTok Profile Data using Dumpling AI  | 🧩 Node Breakdown: Uses regex to isolate TikTok handle                                                                                      |
| Get TikTok Profile Data using Dumpling AI | HTTP Request          | Retrieves TikTok profile stats via API| Extract Username from TikTok URL     | Get TikTok Video Transcript using Dumpling AI | Sends username to Dumpling AI to get follower, likes, video count stats                                                                     |
| Get TikTok Video Transcript using Dumpling AI | HTTP Request          | Retrieves video transcript via API     | Get TikTok Profile Data using Dumpling AI | Save Profile Stats and Transcript to Google Sheet | Uses video URL to fetch transcript                                                                                                         |
| Save Profile Stats and Transcript to Google Sheet | Google Sheets          | Appends data to Google Sheet           | Get TikTok Video Transcript using Dumpling AI | None                                      | Saves username, stats, transcript, and video URL back to sheet                                                                              |
| Sticky Note                          | Sticky Note             | Documentation and overview              | None                                 | None                                       | 🎯 Workflow Purpose: Monitors a Google Sheet for new TikTok video links. Uses Dumpling AI to extract profile data and transcripts. Saves back. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Name: `Watch for New TikTok Links in Sheet`  
   - Type: Google Sheets Trigger  
   - Set Event to `rowAdded`  
   - Configure Polling to every 1 minute  
   - Link to Google Sheet document (ID of your “Videos” sheet) and Sheet Name `Sheet1`  
   - Use OAuth2 credentials for Google Sheets Trigger  

2. **Create Set Node to Extract Username**  
   - Name: `Extract Username from TikTok URL`  
   - Type: Set  
   - Add a new field `USERNAME` (string)  
   - Use expression to extract username from the trigger’s `USERNAME Video` field:  
     ```js
     {{$json["USERNAME Video"] && $json["USERNAME Video"].match(/@([^\/]+)/) ? $json["USERNAME Video"].match(/@([^\/]+)/)[1] : null}}
     ```  
   - Connect input from `Watch for New TikTok Links in Sheet`

3. **Create HTTP Request Node for Profile Data**  
   - Name: `Get TikTok Profile Data using Dumpling AI`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/get-tiktok-profile`  
   - Body: JSON with key `handle` set to `{{$json.USERNAME}}`  
   - Authentication: HTTP Header Auth using a credential with required API key for Dumpling AI  
   - Connect input from `Extract Username from TikTok URL`

4. **Create HTTP Request Node for Video Transcript**  
   - Name: `Get TikTok Video Transcript using Dumpling AI`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/get-tiktok-transcript`  
   - Body: JSON with key `videoUrl` set to `{{ $('Watch for New TikTok Links in Sheet').item.json['USERNAME Video'] }}`  
   - Authentication: Same HTTP Header Auth credential as above  
   - Connect input from `Get TikTok Profile Data using Dumpling AI`

5. **Create Google Sheets Node to Append Data**  
   - Name: `Save Profile Stats and Transcript to Google Sheet`  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID and Sheet Name: same as trigger node  
   - Map columns manually:  
     - `Username ` → `{{ $('Extract Username from TikTok URL').item.json.USERNAME }}`  
     - `Transcript` → `{{ $json.transcript }}`  
     - `Video Count` → `{{ $('Get TikTok Profile Data using Dumpling AI').item.json.stats.videoCount }}`  
     - `heart count` → `{{ $('Get TikTok Profile Data using Dumpling AI').item.json.stats.heart }}`  
     - `Follower count` → `{{ $('Get TikTok Profile Data using Dumpling AI').item.json.stats.followerCount }}`  
     - `USERNAME Video` → `{{ $('Watch for New TikTok Links in Sheet').item.json['USERNAME Video'] }}`  
     - `Following Count` → `{{ $('Get TikTok Profile Data using Dumpling AI').item.json.stats.followingCount }}`  
   - Use OAuth2 credentials for Google Sheets  
   - Connect input from `Get TikTok Video Transcript using Dumpling AI`

6. **Optional: Add Sticky Note for Documentation**  
   - Add a Sticky Note node anywhere in the canvas  
   - Paste the workflow purpose and node breakdown content for future reference

7. **Activate Workflow**  
   - Save and activate to start watching for new TikTok URLs

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow depends on Dumpling AI APIs for TikTok profile and transcript extraction, requiring valid API credentials.         | Dumpling AI API docs (https://app.dumplingai.com/docs)                                                  |
| Google Sheets OAuth2 credentials must allow both read (trigger) and write (append) access to the target spreadsheet.            | Google Cloud Console OAuth2 setup for Google Sheets API                                                  |
| TikTok video URLs must contain the username in the format `@username` for the regex to successfully extract the handle.        | Regex pattern used: `/@([^\/]+)/`                                                                       |
| Dumpling AI API rate limits or downtime may cause delays or failures in data retrieval; implement retry or error handling.      | Consider n8n error workflows or additional nodes for robustness                                         |
| For large spreadsheets or frequent updates, consider optimizing polling interval or using Google Sheets push triggers if available. | n8n Google Sheets Trigger node documentation                                                            |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.