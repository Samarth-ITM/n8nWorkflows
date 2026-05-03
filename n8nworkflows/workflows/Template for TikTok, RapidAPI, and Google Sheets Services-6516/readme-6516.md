Template for TikTok, RapidAPI, and Google Sheets Services

https://n8nworkflows.xyz/workflows/template-for-tiktok--rapidapi--and-google-sheets-services-6516


# Template for TikTok, RapidAPI, and Google Sheets Services

### 1. Workflow Overview

This n8n workflow automates the process of collecting TikTok user analytics by integrating RapidAPI TikTok endpoints and Google Sheets. It is designed to run on a scheduled basis, fetch TikTok profile and video statistics for multiple usernames read from a Google Sheet, and append the aggregated results back into another Google Sheet for reporting or analysis.

The workflow is composed of the following logical blocks:

- **1.1 Scheduled Trigger & Input Reception:** Initiates the workflow on a time-based schedule and reads a list of TikTok usernames from a Google Sheet.
- **1.2 Batch Processing Loop:** Iterates over each username individually to process data in manageable batches.
- **1.3 TikTok Data Retrieval:** For each username, calls RapidAPI endpoints to fetch profile details and video metrics.
- **1.4 Data Output & Logging:** Appends the collected analytics data into designated Google Sheets for storage and reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Input Reception

- **Overview:** This block triggers the workflow automatically at preset intervals and reads the initial list of TikTok usernames from a Google Sheets document.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Google Sheets (Read)

##### Node: Schedule Trigger
- **Type and Role:** Trigger node; initiates the workflow based on a schedule.
- **Configuration:** Uses a simple interval trigger (default unspecified interval in JSON, typically set to daily/hourly).
- **Expressions/Variables:** None.
- **Connections:** Outputs to Google Sheets node.
- **Edge Cases:** Misconfiguration of interval; workflow won’t start if disabled.
- **Version:** 1.2.

##### Node: Google Sheets (Read)
- **Type and Role:** Reads input data (TikTok usernames) from a Google Sheets document using service account authentication.
- **Configuration:**  
  - Reads from a specified Google Sheets document and sheet (documentId and sheetName are URL mode placeholders to be configured).  
  - Authentication via Google Service Account credentials.
- **Expressions:** None; uses static or configured document URLs.
- **Connections:** Outputs to Loop Over Items node.
- **Edge Cases:**  
  - Google Sheets API auth failure.  
  - Empty or malformed sheet data.  
  - Incorrect documentId or sheetName.
- **Version:** 4.6.

---

#### 1.2 Batch Processing Loop

- **Overview:** Splits the list of usernames into batches and processes each username sequentially to avoid request overload.
- **Nodes Involved:**  
  - Loop Over Items

##### Node: Loop Over Items
- **Type and Role:** Split batches node; iterates over each username item.
- **Configuration:** Default batch size (not explicitly set, so uses default 1 item per batch).
- **Expressions:** Pulls username from Google Sheets node item JSON.
- **Connections:**  
  - Main output (empty) loops back to itself after Videos Stats node.  
  - Secondary output triggers Fetch Profile node for API calls.
- **Edge Cases:**  
  - Empty input array causes no iterations.  
  - Large inputs may require batch size tuning.
- **Version:** 3.

---

#### 1.3 TikTok Data Retrieval

- **Overview:** For each username, calls RapidAPI TikTok endpoints to fetch profile and video statistics.
- **Nodes Involved:**  
  - Fetch Profile (user_profile.php)  
  - Profile Stats (Google Sheets Append)  
  - Fetch Videos (view_count.php)

##### Node: Fetch Profile
- **Type and Role:** HTTP Request; fetches TikTok user profile data.
- **Configuration:**  
  - POST request to `https://tiktok-api42.p.rapidapi.com/user_profile.php`.  
  - Sends username as form-data in body parameters.  
  - Includes RapidAPI headers (`x-rapidapi-host`, `x-rapidapi-key`), where `x-rapidapi-key` must be replaced with a valid key.  
  - Content-Type: multipart/form-data.
- **Expressions:** Username taken from Loop Over Items current item JSON.
- **Connections:** Outputs to Profile Stats node.
- **Edge Cases:**  
  - API key invalid or rate-limited.  
  - Username not found or invalid.  
  - HTTP timeouts or network errors.
- **Version:** 4.2.

##### Node: Profile Stats (Google Sheets Append)
- **Type and Role:** Google Sheets node; appends profile data to a designated sheet.
- **Configuration:**  
  - Appends a row with fields: date (current timestamp), likes, videos, username, followers, following.  
  - Data fields are mapped from the HTTP response JSON and the Loop Over Items username.  
  - Uses service account authentication.  
  - Target documentId and sheetName configured as URLs (placeholders).
- **Expressions:**  
  - Date: `{{$now}}`  
  - Likes, videos, followers, following from `userProfile` object returned by Fetch Profile node.  
  - Username from Loop Over Items node.
- **Connections:** Outputs to Fetch Videos node.
- **Edge Cases:**  
  - Missing or malformed profile data.  
  - Google Sheets API errors.
- **Version:** 4.6.

##### Node: Fetch Videos
- **Type and Role:** HTTP Request; fetches TikTok video statistics.
- **Configuration:**  
  - POST request to `https://tiktok-api42.p.rapidapi.com/view_count.php`.  
  - Sends username as multipart/form-data body parameter.  
  - Includes RapidAPI headers similar to Fetch Profile node.  
  - Content-Type: multipart/form-data.
- **Expressions:** Username taken from Loop Over Items current item JSON.
- **Connections:** Outputs to Videos Stats node.
- **Edge Cases:**  
  - API errors similar to Fetch Profile.  
  - Missing video data in response.
- **Version:** 4.2.

---

#### 1.4 Data Output & Logging

- **Overview:** Appends collected video statistics and metadata to a Google Sheet for record-keeping and reporting.
- **Nodes Involved:**  
  - Videos Stats (Google Sheets Append)

##### Node: Videos Stats
- **Type and Role:** Google Sheets node; appends video stats and metadata.
- **Configuration:**  
  - Appends a row with: date, days, user, videos, success, username, total_posts, total_views.  
  - Values are mapped dynamically from Fetch Videos response and Loop Over Items username.  
  - Uses service account authentication.  
  - Target documentId and sheetName placeholders.
- **Expressions:**  
  - Date: `{{$now}}`  
  - Other fields from JSON output of Fetch Videos or Loop Over Items.
- **Connections:** Outputs back to Loop Over Items node (main output), allowing next batch processing.
- **Edge Cases:**  
  - Google Sheets API errors.  
  - Missing or null data fields.
- **Version:** 4.6.

---

### 3. Summary Table

| Node Name          | Node Type             | Functional Role                                | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                      |
|--------------------|-----------------------|------------------------------------------------|-----------------------|----------------------|------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Trigger               | Starts workflow on a defined schedule          | —                     | Google Sheets         | **Schedule Trigger** triggers the entire workflow on a predefined schedule (e.g., daily/hourly).                |
| Google Sheets      | Google Sheets (Read)   | Reads TikTok usernames from source sheet       | Schedule Trigger       | Loop Over Items       | **Google Sheets** reads a list of TikTok usernames from a master Google Spreadsheet.                            |
| Loop Over Items    | SplitInBatches         | Iterates over each username in batches          | Google Sheets          | Fetch Profile (2nd output), (main output loops back from Videos Stats) | **Loop Over Items** iterates over each username, allowing individual API requests.                             |
| Fetch Profile      | HTTP Request           | Fetches TikTok user profile data                | Loop Over Items        | Profile Stats         | Calls `user_profile.php` endpoint via RapidAPI to retrieve user details like followers, likes, videos, etc.     |
| Profile Stats      | Google Sheets (Append) | Appends profile data to result sheet            | Fetch Profile          | Fetch Videos          | Appends collected profile data to a Google Sheet for analytics tracking.                                        |
| Fetch Videos       | HTTP Request           | Fetches TikTok video statistics                  | Profile Stats          | Videos Stats          | Sends POST request to `view_count.php` on RapidAPI to retrieve video views, posts count, and other stats.       |
| Videos Stats       | Google Sheets (Append) | Appends video stats and metadata to result sheet| Fetch Videos           | Loop Over Items       | Appends all collected data as a final output log or dashboard for TikTok data tracking.                         |
| Sticky Note        | Sticky Note            | Documentation and workflow description          | —                     | —                    | # 📊 TikTok Analytics Automation … (full descriptive content)                                                  |
| Sticky Note1       | Sticky Note            | Schedule Trigger explanation                      | —                     | —                    | **Schedule Trigger** triggers the entire workflow based on a predefined schedule (e.g., daily or hourly).       |
| Sticky Note2       | Sticky Note            | Google Sheets input explanation                   | —                     | —                    | **Google Sheets** reads a list of TikTok usernames from a master Google Spreadsheet.                            |
| Sticky Note3       | Sticky Note            | Loop Over Items explanation                        | —                     | —                    | **Loop Over Items** iterates over each username for API requests.                                              |
| Sticky Note4       | Sticky Note            | Fetch Profile explanation                          | —                     | —                    | Calls `user_profile.php` endpoint on RapidAPI for current username, retrieves user profile details.             |
| Sticky Note5       | Sticky Note            | Fetch Videos explanation                           | —                     | —                    | Sends POST request to `view_count.php` on RapidAPI using the username, returns video stats.                     |
| Sticky Note6       | Sticky Note            | Videos Stats explanation                           | —                     | —                    | Appends collected data to result sheet, acts as final output log/dashboard for TikTok data.                     |
| Sticky Note7       | Sticky Note            | Profile Stats explanation                          | —                     | —                    | Appends collected profile data to result sheet, acts as output log/dashboard for TikTok data.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**
   - Type: Schedule Trigger
   - Configure interval (e.g., daily at a specific time).
   - No credentials needed.

2. **Add Google Sheets Node to Read Usernames**
   - Type: Google Sheets
   - Operation: Read rows
   - Parameters:  
     - Document ID: Set to your source Google Spreadsheet URL or ID containing TikTok usernames.  
     - Sheet Name: Specify the exact sheet name.  
     - Authentication: Use a Google API Service Account credential with access to the spreadsheet.
   - Connect Schedule Trigger output to this node.

3. **Add SplitInBatches Node (Loop Over Items)**
   - Type: SplitInBatches
   - Default batch size (1 item per batch recommended).
   - Connect Google Sheets node output to this node.

4. **Add HTTP Request Node (Fetch Profile)**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://tiktok-api42.p.rapidapi.com/user_profile.php`
   - Content-Type: multipart/form-data
   - Body Parameters: Add `username` parameter with value expression `={{ $json.username }}`
   - Headers:  
     - `x-rapidapi-host`: `tiktok-api42.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key (store securely in environment variables or credentials)
   - Connect secondary output of SplitInBatches node to this node.

5. **Add Google Sheets Node (Profile Stats Append)**
   - Type: Google Sheets
   - Operation: Append
   - Document ID & Sheet Name: Set to your results spreadsheet and target sheet for profile stats.
   - Authentication: Same Google Service Account credential.
   - Map columns: `date` = `{{$now}}`, `likes`, `videos`, `username`, `followers`, `following` — map these from Fetch Profile response JSON and SplitInBatches username.
   - Connect Fetch Profile output to this node.

6. **Add HTTP Request Node (Fetch Videos)**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://tiktok-api42.p.rapidapi.com/view_count.php`
   - Content-Type: multipart/form-data
   - Body Parameters: `username` = `={{ $json.username }}`
   - Headers: Same as Fetch Profile.
   - Connect Profile Stats node output to this node.

7. **Add Google Sheets Node (Videos Stats Append)**
   - Type: Google Sheets
   - Operation: Append
   - Document ID & Sheet Name: Set to your results spreadsheet and target sheet for video stats.
   - Authentication: Google Service Account credential.
   - Map columns: `date` = `{{$now}}`, and other fields (`days`, `user`, `videos`, `success`, `username`, `total_posts`, `total_views`) mapped from Fetch Videos response and SplitInBatches username.
   - Connect Fetch Videos output to this node.

8. **Connect Videos Stats output back to SplitInBatches main output**
   - This allows the loop to continue processing next username batch.

9. **Set up Credentials**
   - Google API Service Account with access to the relevant Google Sheets documents.
   - RapidAPI key for TikTok API endpoints, stored securely.

10. **Optional: Add Sticky Notes**
    - Add descriptive sticky notes to document each node or block, including API key usage, scheduling, and error handling tips.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                             | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates TikTok analytics fetching via RapidAPI and logs results in Google Sheets; supports scheduled runs and batch processing for multiple users.                                                          | Workflow overview and use case description.                                                        |
| Tips to use environment variables in n8n for API keys to enhance security and avoid hardcoding.                                                                                                                         | Security best practices.                                                                            |
| For heavy loads, consider implementing error handling, retries, and logging of failed usernames for robustness.                                                                                                         | Workflow robustness and scalability advice.                                                        |
| RapidAPI TikTok endpoints require valid API keys and have rate limits; monitor usage to avoid blocking.                                                                                                                 | API integration considerations.                                                                    |
| Google Sheets nodes require service account credentials with proper access to the target sheets.                                                                                                                        | Credential setup.                                                                                   |
| See n8n documentation for details on configuring SplitInBatches nodes to optimize batch sizes according to API rate limits and performance.                                                                             | n8n official docs: https://docs.n8n.io/nodes/n8n-nodes-base.splitinbatches/                         |
| Consider using environment variables or n8n credentials to store sensitive information like RapidAPI keys and Google credentials instead of inline values.                                                                | Security best practice link: https://docs.n8n.io/code-examples/variables/                           |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It adheres strictly to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.