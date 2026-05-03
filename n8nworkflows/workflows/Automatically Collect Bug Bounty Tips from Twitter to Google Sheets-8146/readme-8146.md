Automatically Collect Bug Bounty Tips from Twitter to Google Sheets

https://n8nworkflows.xyz/workflows/automatically-collect-bug-bounty-tips-from-twitter-to-google-sheets-8146


# Automatically Collect Bug Bounty Tips from Twitter to Google Sheets

### 1. Workflow Overview

This workflow, titled **"Automatically Collect Bug Bounty Tips from Twitter to Google Sheets"**, is designed to automate the collection of bug bounty-related tweets from Twitter and archive them systematically into a Google Sheets document for further analysis or tracking.

The main use cases include:
- Continuously harvesting fresh bug bounty tips from Twitter without manual intervention.
- Filtering and formatting tweets to extract relevant data fields.
- Preventing duplicates by matching tweet IDs when inserting data into Google Sheets.
- Scheduling periodic runs to keep the dataset updated every 4 hours.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception**: Scheduled trigger initiates the workflow every 4 hours.
- **1.2 Twitter Data Retrieval**: HTTP Request node calls the Twitter API via twitterapi.io using a specific query to fetch recent bug bounty related tweets.
- **1.3 Data Processing**: A Code node processes the raw API response, normalizing tweet data into a clean structure and formatting dates.
- **1.4 Data Storage**: Google Sheets node appends or updates rows into a Google Sheet, using Tweet IDs to avoid duplicates.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow execution every 4 hours to automate data collection without manual input.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node (Schedule)  
    - Configuration: Set to run every 4 hours (interval in hours = 4).  
    - Key parameters: Interval scheduling, no additional filters.  
    - Input connections: None (trigger node).  
    - Output connections: Connects to HTTP Request node to start data retrieval.  
    - Edge Cases: If the n8n instance is down or paused during the scheduled time, the run will be missed. No retry logic is configured here.

#### 1.2 Twitter Data Retrieval

- **Overview:**  
  This block fetches recent tweets matching specific bug bounty hashtags and filters from the Twitter API using twitterapi.io.

- **Nodes Involved:**  
  - HTTP Request  
  - Sticky Note "📒 Twitter API" (documentation node)

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Configuration:  
      - URL set to `https://api.twitterapi.io/twitter/tweet/advanced_search`  
      - Query parameters:  
        - `query` set to `(#bugbountytips OR #bugbounty OR #bugbountytip) -Writeups -discount -phishing -fuck` to filter relevant tweets and exclude unwanted terms.  
        - `queryType` set to `Latest` to get recent tweets.  
      - Authentication: HTTP Header Auth with header `x-api-key` using Twitter API key credential from twitterapi.io.  
    - Input: Triggered by Schedule Trigger node.  
    - Output: Sends API response JSON to the Code node.  
    - Edge Cases:  
      - API rate limits or quota exceeded errors.  
      - Invalid or expired API keys causing authentication errors.  
      - HTTP request timeouts or network failures.  
      - Unexpected API response formats or structure changes.

  - **Sticky Note "📒 Twitter API"**  
    - Purpose: Provides instructions on setting up the Twitter API key and authentication header.  
    - Content:  
      - Sign up at https://twitterapi.io/  
      - Obtain API key  
      - Add HTTP Header Auth with header `x-api-key` and the API key value.

#### 1.3 Data Processing

- **Overview:**  
  This block processes the raw Twitter API response, extracting relevant fields from each tweet, formatting dates into a human-readable form, and normalizing the output to a consistent structure suitable for Google Sheets input.

- **Nodes Involved:**  
  - Code  
  - Sticky Note "📒 Processing"

- **Node Details:**

  - **Code**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Handles both single tweet responses and collections of tweets.  
      - Extracts fields: tweetId, url, content, likeCount, retweetCount, replyCount, quoteCount, viewCount, createdAt.  
      - Formats `createdAt` dates into a readable string like "March 13, 2025 at 19:25".  
      - Returns an array of normalized tweet objects if multiple tweets are present or a single normalized object otherwise.  
      - Includes console logs for debugging input structure and processing steps.  
    - Input: Receives raw JSON from HTTP Request node.  
    - Output: Produces formatted JSON items for Google Sheets ingestion.  
    - Edge Cases:  
      - Missing or malformed date string causing formatting errors (caught and fallback to raw string).  
      - Unexpected data structure in API response.  
      - Empty tweet arrays resulting in zero output items.

  - **Sticky Note "📒 Processing"**  
    - Purpose: Documentation of data processing logic.  
    - Content:  
      - Extracts tweet content, URLs, engagement metrics.  
      - Formats date for sorting.  
      - Handles both single and multiple tweets.

#### 1.4 Data Storage

- **Overview:**  
  This block appends or updates rows in a Google Sheet with the processed tweet data, preventing duplicates by matching on Tweet IDs.

- **Nodes Involved:**  
  - Append or update row in sheet (Google Sheets node)  
  - Sticky Note "📊 Google Sheets"

- **Node Details:**

  - **Append or update row in sheet**  
    - Type: Google Sheets node  
    - Operation: Append or update (upsert) rows in the sheet.  
    - Configuration:  
      - Document ID: to be replaced with user's Google Sheet ID.  
      - Sheet Name: defaults to Sheet1 (`gid=0`).  
      - Columns mapped:  
        - Url from tweet `url`  
        - Date: parsed and reformatted from `createdAt` (converted to `YYYY-MM-DD HH:mm:ss` format)  
        - Content from tweet `content`  
        - TweetID from tweet `tweetId` (used as matching column to prevent duplicates)  
        - Created At from formatted string `createdAt`  
      - Matching Columns: `TweetID` used to detect existing rows for update.  
      - Credentials: Google Sheets OAuth2 configured by user.  
    - Input: Receives formatted tweet data from Code node.  
    - Output: None (end node).  
    - Edge Cases:  
      - Missing or incorrect Google Sheet ID causing failure to write.  
      - Google OAuth token expiration or insufficient permissions.  
      - Sheet schema mismatch (missing columns).  
      - Network connectivity issues.

  - **Sticky Note "📊 Google Sheets"**  
    - Purpose: Setup instructions for Google Sheets integration.  
    - Content:  
      - Create Google Sheet with columns: Date, Created At, TweetID, Content, Url  
      - Set up Google Sheets OAuth2 credentials  
      - Replace documentId with your Sheet ID  
      - Prevents duplicates using TweetID matching  

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                         | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                     |
|------------------------------|---------------------------|---------------------------------------|------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger          | Triggers workflow every 4 hours       | None                   | HTTP Request                 |                                                                                                |
| HTTP Request                | HTTP Request             | Fetches tweets from Twitter API       | Schedule Trigger       | Code                         |                                                                                                |
| Code                        | Code                     | Processes and formats tweet data      | HTTP Request           | Append or update row in sheet |                                                                                                |
| Append or update row in sheet | Google Sheets             | Appends/updates data into Google Sheet| Code                   | None                         |                                                                                                |
| 📒 Workflow Overview        | Sticky Note              | Workflow purpose and setup instructions| None                   | None                         | ## 🎯 Bug Bounty Tip Harvester\n\nAutomatically collects bug bounty tips from Twitter every 4 hours and saves to Google Sheets.\n\n**Setup required:**\n1. Get API key from https://twitterapi.io/\n2. Configure Google Sheets credentials\n3. Update Google Sheet ID |
| 📒 Twitter API              | Sticky Note              | Twitter API setup instructions        | None                   | None                         | ## 🔐 Twitter API Setup\n\n1. Sign up at https://twitterapi.io/\n2. Get your API key\n3. Add HTTP Header Auth:\n   - **Header**: `x-api-key`\n   - **Value**: `YOUR_API_KEY`                               |
| 📒 Processing               | Sticky Note              | Explains data extraction and formatting| None                   | None                         | ## ⚙️ Data Processing\n\nExtracts tweet data and formats for Google Sheets:\n- Tweet content, URLs, engagement metrics\n- Date formatting for proper sorting\n- Handles both single tweets and collections        |
| 📊 Google Sheets            | Sticky Note              | Google Sheets integration instructions| None                   | None                         | ## 📊 Google Sheets Setup\n\n1. Create Google Sheet with columns: Date, Created At, TweetID, Content, Url\n2. Set up Google Sheets OAuth2 credentials\n3. Replace empty documentId with your Sheet ID\n\n**Prevents duplicates using TweetID matching** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "BB Tip Harvester").

2. **Add a Schedule Trigger node:**  
   - Set it to execute every 4 hours (`interval` → `hours` → `4`).  
   - No additional filters are needed.

3. **Add an HTTP Request node:**  
   - Connect the Schedule Trigger's main output to this node's input.  
   - Set the HTTP Method to GET.  
   - URL: `https://api.twitterapi.io/twitter/tweet/advanced_search`  
   - Add query parameters:  
     - `query`: `(#bugbountytips OR #bugbounty OR #bugbountytip) -Writeups -discount -phishing -fuck`  
     - `queryType`: `Latest`  
   - Authentication:  
     - Create and select HTTP Header Auth credentials with header name `x-api-key` and your API key from https://twitterapi.io/.  
   - Ensure the query parameters are sent as URL query parameters.

4. **Add a Code node:**  
   - Connect HTTP Request node output to this node input.  
   - Paste the following JavaScript code:  
     ```javascript
     function formatDate(dateString) {
       if (!dateString) return '';
       try {
         const date = new Date(dateString);
         return date.toLocaleString('en-US', {
           year: 'numeric',
           month: 'long',
           day: 'numeric',
           hour: '2-digit',
           minute: '2-digit',
         });
       } catch {
         return dateString;
       }
     }
     
     if ($input.item.json.tweets && Array.isArray($input.item.json.tweets)) {
       return $input.item.json.tweets.map(tweet => ({
         json: {
           tweetId: tweet.id || '',
           url: tweet.url || '',
           content: tweet.text || '',
           likeCount: tweet.likeCount || 0,
           retweetCount: tweet.retweetCount || 0,
           replyCount: tweet.replyCount || 0,
           quoteCount: tweet.quoteCount || 0,
           viewCount: tweet.viewCount || 0,
           createdAt: formatDate(tweet.createdAt),
         }
       }));
     } else {
       const t = $input.item.json;
       return {
         json: {
           tweetId: t.id || '',
           url: t.url || '',
           content: t.text || '',
           likeCount: t.likeCount || 0,
           retweetCount: t.retweetCount || 0,
           replyCount: t.replyCount || 0,
           quoteCount: t.quoteCount || 0,
           viewCount: t.viewCount || 0,
           createdAt: formatDate(t.createdAt),
         }
       };
     }
     ```
   - This code normalizes tweet data for further processing.

5. **Add a Google Sheets node:**  
   - Set operation to "Append or Update".  
   - Provide your Google Sheets OAuth2 credentials (configure via n8n credentials).  
   - Set Document ID to your Google Sheet's ID (found in the sheet URL).  
   - Set Sheet Name to `Sheet1` or your target sheet tab name.  
   - Define columns and mappings:  
     - `Url`: `{{$json.url}}`  
     - `Date`: Use an expression to parse and convert the formatted date string from `createdAt` into a sortable timestamp:  
       ```
       {{
         (() => {
           const raw = $json.createdAt; 
           const [monthStr, day, yearTime, , time, meridian] = raw.split(/[\s,]+/);
           const months = { January:"01", February:"02", March:"03", April:"04", May:"05", June:"06", July:"07", August:"08", September:"09", October:"10", November:"11", December:"12" };
           const [hour, minute] = time.split(":").map(Number);
           let hh = meridian === "PM" && hour !== 12 ? hour + 12 : (meridian === "AM" && hour === 12 ? 0 : hour);
           const formattedHour = String(hh).padStart(2, '0');
           return `${yearTime}-${months[monthStr]}-${String(day).padStart(2, '0')} ${formattedHour}:${minute}:00`;
         })()
       }}
       ```  
     - `Content`: `{{$json.content}}`  
     - `TweetID`: `{{$json.tweetId}}`  
     - `Created At`: `{{$json.createdAt}}`  
   - Set matching columns to `TweetID` to avoid duplicates.

6. **Connect the Code node's output to the Google Sheets node input.**

7. **Add Sticky Note nodes:**
   - For clarity, add notes with setup instructions for:  
     - Workflow overview and setup steps  
     - Twitter API key acquisition and header setup  
     - Data processing explanation  
     - Google Sheets setup and credential instructions  

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow requires valid API credentials from https://twitterapi.io/ for Twitter data access.                                                               | Twitter API setup                              |
| Google Sheets OAuth2 credentials must be configured with access to the target sheet where data will be stored.                                                  | Google Sheets integration                      |
| The workflow automatically prevents duplicate entries by matching Tweet IDs during insert/update operations.                                                    | Data integrity and duplicate prevention       |
| The Twitter API query excludes tweets containing "Writeups", "discount", "phishing", "fuck" to reduce noise.                                                    | Twitter query refinement                       |
| Date formatting in the Code node enhances readability and sorting capabilities in Google Sheets.                                                                | Date handling in data processing               |
| For more info on twitterapi.io, visit https://twitterapi.io/                                                                                                     | External resource for Twitter API key          |

---

This documentation enables advanced users and automation agents to fully understand, reproduce, and modify the workflow, anticipate potential errors, and integrate it effectively with Twitter and Google Sheets APIs.