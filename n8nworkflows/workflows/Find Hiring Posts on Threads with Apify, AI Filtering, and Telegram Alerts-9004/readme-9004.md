Find Hiring Posts on Threads with Apify, AI Filtering, and Telegram Alerts

https://n8nworkflows.xyz/workflows/find-hiring-posts-on-threads-with-apify--ai-filtering--and-telegram-alerts-9004


# Find Hiring Posts on Threads with Apify, AI Filtering, and Telegram Alerts

### 1. Workflow Overview

This workflow is designed to automatically scrape Meta Threads for social media posts where authors are hiring specific roles related to automation engineering with n8n. It gathers posts matching a set of keywords, filters them intelligently using AI to detect genuine hiring intents (excluding self-promotion and irrelevant posts), applies geographic filters, deduplicates results using Google Sheets, and sends notifications via Telegram for immediate action.

The workflow is segmented into these logical blocks:

- **1.1 Scheduled Trigger and Data Collection:** Periodically initiates scraping jobs via Apify using multiple relevant keywords.
- **1.2 Data Merging and Normalization:** Aggregates results from multiple scraping requests and normalizes fields for consistent processing.
- **1.3 AI Filtering and Deduplication:** Uses an AI agent (LangChain with OpenAI GPT-4.1-mini) to classify posts strictly by hiring intent, applies geographic filtering based solely on post text, and checks Google Sheets to avoid duplicates.
- **1.4 Data Persistence and Alerts:** Saves approved posts to Google Sheets and sends Telegram alerts with post details.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Data Collection

**Overview:**  
This block triggers the workflow on a regular interval and launches multiple HTTP POST requests to the Apify Meta Threads Scraper actor to fetch recent posts matching specific keywords related to n8n automation roles.

**Nodes Involved:**  
- 3 hours1 (Schedule Trigger)  
- HTTP Request2  
- HTTP Request7  
- HTTP Request8  
- HTTP Request9  
- HTTP Request10  
- HTTP Request11  

**Node Details:**  

- **3 hours1**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow every 12 hours.  
  - *Configuration:* Interval set to 12 hours.  
  - *Connections:* Outputs to six HTTP Request nodes.  
  - *Failure cases:* Trigger skipping if n8n environment is down or scheduler disabled.

- **HTTP Request2** *(Keyword: "AI automation expert")*  
  - *Type:* HTTP Request (POST)  
  - *Role:* Calls Apify API to scrape recent posts containing keyword “AI automation expert.”  
  - *Configuration:* JSON body includes keyword, limit=50, mode=keyword, searchFilter=recent; headers include Authorization bearer token and Accept header.  
  - *Input:* Trigger from 3 hours1.  
  - *Output:* Posts data to Merge1 node.  
  - *Edge cases:* HTTP request failure, invalid API token, rate limiting.

- **HTTP Request7** *(Keyword: "n8n freelance projects")*  
  - Same as HTTP Request2 but with keyword “n8n freelance projects.”  

- **HTTP Request8** *(Keyword: "n8n")*  
  - Same structure, keyword “n8n.”  

- **HTTP Request9** *(Keyword: "hire n8n")*  
  - Same structure, keyword “hire n8n.”  

- **HTTP Request10** *(Keyword: "n8n expert")*  
  - Same structure, keyword “n8n expert.”  

- **HTTP Request11** *(Keyword: "AI automation developer")*  
  - Same structure, keyword “AI automation developer.”  

**Notes:**  
- All HTTP Request nodes call the same Apify actor endpoint but with different keywords to broaden coverage.  
- It is critical to replace the placeholder API tokens with valid Apify API keys for authentication.  
- The actor used is documented here: https://apify.com/futurizerush/meta-threads-scraper  

---

#### 2.2 Data Merging and Normalization

**Overview:**  
Combines the multiple incoming keyword-based scraped datasets into a unified stream and normalizes the data fields to a consistent shape for processing.

**Nodes Involved:**  
- Merge1  
- Edit Fields  

**Node Details:**  

- **Merge1**  
  - *Type:* Merge  
  - *Role:* Joins six input streams (one from each HTTP request) into one data stream.  
  - *Configuration:* Number of inputs set to 6, merging with default mode (append).  
  - *Input:* Six HTTP Request nodes.  
  - *Output:* Single stream to Edit Fields node.  
  - *Possible issues:* If any input is missing or delayed, merge may wait or fail.

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Normalizes and assigns the following fields explicitly from incoming JSON: text_content, created_at, post_url, username, profile_url.  
  - *Configuration:* Assign each field from raw JSON to normalized key with string typing.  
  - *Input:* Merge1 output.  
  - *Output:* To AI Agent node.  
  - *Edge cases:* If any field is missing or malformed, might propagate empty or incorrect data.

---

#### 2.3 AI Filtering and Deduplication

**Overview:**  
Uses an AI Agent node powered by LangChain and OpenAI GPT-4.1-mini to classify each post strictly by the hiring intent rules, geographic filters, and deduplication via Google Sheets. Posts failing classification or found as duplicates are discarded.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Get row(s) in sheet in Google Sheets  
- If  
- Code  

**Node Details:**  

- **AI Agent**  
  - *Type:* LangChain Agent (AI Agent)  
  - *Role:* Processes each post (fields text_content, created_at, username, profile_url, post_url) with a detailed prompt to:  
    - Check if post signals hiring an n8n automation engineer in allowed geographies (US, UK, UAE, CA) using text_content only.  
    - Exclude self-promotion, irrelevant roles, replies/retweets unless restated by author as hirer.  
    - Deduplicate by querying Google Sheets for existing post_url.  
  - *Configuration:* Prompt includes systemMessage with detailed rules for hiring detection, geographic validation, deduplication logic, and expected JSON output format (strict JSON or no output).  
  - *Input:* Normalized post data from Edit Fields.  
  - *Output:* Raw string JSON if accepted, empty if rejected or duplicate.  
  - *Version:* LangChain agent node v2.2 with AI language model and tool integration (OpenAI + Google Sheets read).  
  - *Failure modes:* API key issues, response timeouts, parsing errors in AI output, mismatches in deduplication query.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Backend LLM used by AI Agent to process classification prompt.  
  - *Configuration:* Model set to "gpt-4.1-mini."  
  - *Credentials:* OpenAI API key required.  
  - *Input:* AI Agent node passes prompt text.  
  - *Output:* Text completion with classification decision.  
  - *Failure:* OpenAI API errors, rate limits, invalid keys.

- **Get row(s) in sheet in Google Sheets**  
  - *Type:* Google Sheets Tool node  
  - *Role:* Acts as an AI Agent tool to query the sheet for existing post_url to check duplicates.  
  - *Configuration:* Reads from specific Google Sheet (document ID and sheet gid=0).  
  - *Credentials:* Google Sheets OAuth2 credential required.  
  - *Input:* AI Agent uses this tool internally for deduplication.  
  - *Failure:* Auth errors, sheet unavailability.

- **If**  
  - *Type:* If node  
  - *Role:* Filters AI Agent output; only proceeds if output is not an empty JSON object ("{}").  
  - *Configuration:* Condition checks that AI Agent’s output field does not equal "{}".  
  - *Input:* AI Agent output.  
  - *Output:* Valid parsed posts to Code node; invalid posts are dropped.  
  - *Failure:* Misconfigured expression might block valid outputs.

- **Code**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the AI Agent output string into JSON objects for downstream use.  
  - *Configuration:* Iterates over all input items, attempts JSON.parse on output field, collects parsed objects, skips invalid parses silently.  
  - *Input:* Filtered AI Agent output from If node.  
  - *Output:* Structured JSON posts for Google Sheets append and Telegram alert.  
  - *Failure:* Parsing invalid JSON strings; currently catches and skips failures gracefully.

---

#### 2.4 Data Persistence and Alerts

**Overview:**  
Stores approved and unique hiring posts in Google Sheets with normalized fields and sends Telegram messages alerting the user with post details.

**Nodes Involved:**  
- Append or update row in sheet  
- Send a text message1  

**Node Details:**  

- **Append or update row in sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends new rows or updates existing rows in a specific Google Sheet tab (named "Thread") with post details: Post, Author, Post URL, Created at, Profile URL.  
  - *Configuration:* Uses "appendOrUpdate" operation with matching on "Post" column to avoid duplicates.  
  - *Credentials:* Google Sheets OAuth2 API credential required.  
  - *Input:* Parsed JSON posts from Code node.  
  - *Output:* None (used for persistence).  
  - *Failure:* Sheet access errors, rate limits, incorrect mapping.

- **Send a text message1**  
  - *Type:* Telegram node  
  - *Role:* Sends a Telegram message to a configured bot channel with the post details formatted in text.  
  - *Configuration:* Message text includes post content, creation time, author, post URL, profile URL using expressions.  
  - *Credentials:* Telegram API credential (bot token) required.  
  - *Input:* Same as Append or update row node (from Code node).  
  - *Failure:* Telegram API errors, invalid token, message formatting issues.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                        | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                                          |
|----------------------------|---------------------------------|-------------------------------------|---------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| 3 hours1                   | Schedule Trigger                 | Periodic trigger every 12 hours     | -                               | HTTP Request2, HTTP Request7, HTTP Request8, HTTP Request9, HTTP Request10, HTTP Request11 | Automatically scrape Meta Threads for posts hiring specific roles (e.g. automation engineers, video editors, graphic designers), filter true hiring intent, deduplicate, and send alerts. |
| HTTP Request2              | HTTP Request                    | Scrape posts with keyword "AI automation expert" | 3 hours1                       | Merge1                           | See Sticky Note2: Replace API keys and update keywords. Apify actor: https://apify.com/futurizerush/meta-threads-scraper            |
| HTTP Request7              | HTTP Request                    | Scrape posts with keyword "n8n freelance projects" | 3 hours1                       | Merge1                           | See Sticky Note2                                                                                                                    |
| HTTP Request8              | HTTP Request                    | Scrape posts with keyword "n8n"    | 3 hours1                       | Merge1                           | See Sticky Note2                                                                                                                    |
| HTTP Request9              | HTTP Request                    | Scrape posts with keyword "hire n8n" | 3 hours1                       | Merge1                           | See Sticky Note2                                                                                                                    |
| HTTP Request10             | HTTP Request                    | Scrape posts with keyword "n8n expert" | 3 hours1                       | Merge1                           | See Sticky Note2                                                                                                                    |
| HTTP Request11             | HTTP Request                    | Scrape posts with keyword "AI automation developer" | 3 hours1                       | Merge1                           | See Sticky Note2                                                                                                                    |
| Merge1                     | Merge                          | Combine all scraped data streams    | HTTP Request2,7,8,9,10,11       | Edit Fields                     |                                                                                                                                      |
| Edit Fields                | Set                            | Normalize fields for AI processing  | Merge1                         | AI Agent                       |                                                                                                                                      |
| AI Agent                   | LangChain Agent                | Classify hiring intent + geo filter + deduplicate | Edit Fields                    | If                             |                                                                                                                                      |
| OpenAI Chat Model          | LangChain OpenAI Chat Model    | Provides GPT-4.1-mini completions   | AI Agent (languageModel)        | AI Agent                      |                                                                                                                                      |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool          | Deduplication lookup for post_url   | AI Agent (tool)                 | AI Agent                      |                                                                                                                                      |
| If                         | If                             | Filter AI output for accepted posts | AI Agent                      | Code                          |                                                                                                                                      |
| Code                       | Code (JS)                      | Parse AI JSON output to structured data | If                            | Append or update row in sheet, Send a text message1 |                                                                                                                                      |
| Append or update row in sheet | Google Sheets                 | Save unique hiring posts            | Code                          | -                             |                                                                                                                                      |
| Send a text message1       | Telegram                       | Send alert message for qualified posts | Code                          | -                             |                                                                                                                                      |
| Sticky Note                | Sticky Note                   | Documentation note                  | -                             | -                             | Automatically scrape Meta Threads for posts hiring specific roles; filter; deduplicate; send alerts.                                |
| Sticky Note1               | Sticky Note                   | Credits and contact info             | -                             | -                             | # Looking for tailored workflows? Book through Reddit or website. See https://www.reddit.com/user/designbyaze/ and https://hushtech.io/ |
| Sticky Note2               | Sticky Note                   | Setup instructions                  | -                             | -                             | Replace API keys; add specified role in JSON; Apify actor URL: https://apify.com/futurizerush/meta-threads-scraper                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger every 12 hours.

2. **Create Six HTTP Request Nodes** (One for each keyword)  
   - Type: HTTP Request (POST)  
   - URL: `https://api.apify.com/v2/acts/futurizerush~meta-threads-scraper/run-sync-get-dataset-items?`  
   - Method: POST  
   - Body Type: JSON  
   - Headers:  
     - Accept: application/json  
     - Authorization: Bearer `<YOUR_APIFY_API_TOKEN>` (replace with your valid token)  
   - Body JSON (change keyword for each):  
     ```json
     {
       "headless": true,
       "keyword": "<keyword>",
       "limit": 50,
       "mode": "keyword",
       "searchFilter": "recent"
     }
     ```  
   - Keywords to create nodes for:  
     - "AI automation expert"  
     - "n8n freelance projects"  
     - "n8n"  
     - "hire n8n"  
     - "n8n expert"  
     - "AI automation developer"  
   - Connect Schedule Trigger output to all six HTTP Requests.

3. **Create a Merge Node**  
   - Type: Merge  
   - Number of Inputs: 6  
   - Connect outputs of all HTTP Request nodes to respective inputs of Merge node.

4. **Create a Set Node (Edit Fields)**  
   - Type: Set  
   - Add fields and assign values from merged input:  
     - text_content = `{{$json["text_content"]}}`  
     - created_at = `{{$json["created_at"]}}`  
     - post_url = `{{$json["post_url"]}}`  
     - username = `{{$json["username"]}}`  
     - profile_url = `{{$json["profile_url"]}}`  
   - Connect Merge node output to this node.

5. **Create an AI Agent Node**  
   - Type: LangChain Agent  
   - Configure prompt with detailed system message including:  
     - Hiring detection rules (author is hiring n8n automation engineer, exclude self-promo)  
     - Geography filters (US, UK, UAE, CA) based on text_content only  
     - Deduplication via Google Sheets query on post_url  
     - Output strict JSON or no output if rejected or duplicate  
   - Set input text combining fields as:  
     ```
     This is the text that you have to check  {{ $json.text_content }}{{ $json.created_at }}{{ $json.username }}{{ $json.profile_url }}{{ $json.post_url }}
     ```  
   - Add AI language model node: OpenAI Chat Model configured with GPT-4.1-mini and your OpenAI API credentials.  
   - Add AI tool: Google Sheets node configured to find rows by post_url for deduplication.  
   - Connect Edit Fields node to AI Agent.

6. **Create an If Node**  
   - Type: If  
   - Condition: Check that AI Agent output field (`$json.output`) is NOT equal to `"{}"` (empty JSON)  
   - Connect AI Agent output to If node.

7. **Create a Code Node**  
   - Type: Code (JavaScript)  
   - Paste code to parse AI Agent output JSON string into JSON objects:  
     ```javascript
     const results = [];

     for (const item of $input.all()) {
       const data = item.json.output; // string to parse

       if (typeof data === 'string' && data.trim() !== '') {
         try {
           const obj = JSON.parse(data);
           results.push({ json: obj });
         } catch (e) {
           // skip invalid JSON
         }
       }
     }

     return results;
     ```  
   - Connect If node’s "true" output to Code node.

8. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Operation: Append or Update Row  
   - Document ID and Sheet Name: Use your Google Sheet tracking document and sheet tab (e.g., "Thread")  
   - Map columns:  
     - Post = `{{$json.text_content}}`  
     - Author = `{{$json.username}}`  
     - Post URL = `{{$json.post_url}}`  
     - Created at = `{{$json.created_at}}`  
     - Profile URL = `{{$json.profile_url}}`  
   - Matching column: Post (to avoid duplicates)  
   - Connect Code node output to this node.

9. **Create Telegram Node**  
   - Type: Telegram  
   - Credential: Your Telegram bot API token  
   - Message Text:  
     ```
     Post (using to match): {{ $json.text_content }},   
     Created at: {{ $json.created_at }},   
     Author: {{ $json.username }},   
     Post URL: {{ $json.post_url }},   
     Profile URL: {{ $json.profile_url }}
     ```  
   - Connect Code node output to Telegram node.

10. **Configure Credentials**  
    - Set up Apify API key in HTTP Request nodes Authorization headers.  
    - Set up OpenAI API credentials for LangChain OpenAI Chat Model node.  
    - Set up Google Sheets OAuth2 credentials with read/write access for Google Sheets nodes.  
    - Set up Telegram Bot API credentials for Telegram node.

11. **Test the Workflow**  
    - Run manually first to verify data is fetched, filtered, saved, and alerts sent.  
    - Adjust keywords or filters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Automatically scrape Meta Threads for posts hiring specific roles (e.g. automation engineers, video editors, graphic designers), filter true hiring intent, deduplicate, and send alerts.                                           | Workflow purpose summary (Sticky Note)                  |
| Replace API keys in each HTTP Request node before running; add or modify keywords in JSON body to target different roles. The Apify actor used is https://apify.com/futurizerush/meta-threads-scraper                               | API setup instructions (Sticky Note2)                   |
| For tailored workflows or custom automation, contact via Reddit https://www.reddit.com/user/designbyaze/ or website https://hushtech.io/                                                                                          | Credits and contact (Sticky Note1)                       |
| AI Agent prompt enforces strict JSON output format and includes detailed hiring, geographic, and deduplication rules to ensure high signal-to-noise ratio.                                                                         | AI prompt design best practices                          |
| Google Sheets acts as both deduplication database and tracking sheet; ensure correct permissions and schema alignment.                                                                                                            | Integration note                                         |
| Telegram notifications provide real-time alerts for quick response to new hiring posts. Consider replacing or adding Slack or Discord nodes for alternative notifications.                                                         | Notification customization idea                          |

---

**Disclaimer:**  
The provided text and workflow are exclusively generated from an n8n automation workflow. The process strictly adheres to content policies and handles only legal and public data. No illegal, offensive, or protected content is involved.