Monitor X/Twitter for Hiring Posts with Apify, AI Filtering & Telegram Alerts

https://n8nworkflows.xyz/workflows/monitor-x-twitter-for-hiring-posts-with-apify--ai-filtering---telegram-alerts-9005


# Monitor X/Twitter for Hiring Posts with Apify, AI Filtering & Telegram Alerts

### 1. Workflow Overview

This workflow automates the monitoring of X (formerly Twitter) for hiring posts specifically targeting n8n/AI automation engineer roles. It leverages Apify’s Tweet scraper to fetch recent tweets containing predefined hiring-related keywords, uses AI filtering to identify genuine hiring posts while excluding self-promotion or irrelevant content, deduplicates results by checking against a Google Sheets database, and finally logs new valid posts into the sheet and sends Telegram alerts to notify users.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger**: Periodically initiates the workflow every 12 hours.
- **1.2 Data Scraping**: Uses Apify HTTP API to fetch recent tweets matching hiring-related keywords.
- **1.3 Data Normalization**: Maps and normalizes tweet data fields to a consistent schema.
- **1.4 AI Filtering & Deduplication**: Employs an AI agent powered by LangChain and OpenAI GPT-4 to classify posts as genuine hiring offers and filters duplicates by querying Google Sheets.
- **1.5 Validation & Parsing**: Validates and parses AI output JSON safely.
- **1.6 Output Handling**: Stores unique hiring posts in Google Sheets and sends Telegram notifications.
- **1.7 Documentation & Notes**: Provides sticky notes with instructions, usage context, and customization guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically every 12 hours to keep the monitoring process continuous and fresh.

- **Nodes Involved:**  
  - `6 hours` (Schedule Trigger)

- **Node Details:**

  - **6 hours**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow at fixed intervals (every 12 hours)  
    - Configuration: Interval set to every 12 hours  
    - Inputs: None (trigger node)  
    - Outputs: Connects to `HTTP Request` to start tweet scraping  
    - Possible Failures: None typical, but workflow won't run if n8n server is down or misconfigured scheduling  
    - Notes: None  

#### 1.2 Data Scraping

- **Overview:**  
  Fetches up to 50 of the latest tweets containing specified hiring-related keywords using Apify’s Tweet Scraper API.

- **Nodes Involved:**  
  - `HTTP Request`

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Calls Apify API to scrape tweets  
    - Configuration:  
      - URL: Apify tweet scraper endpoint  
      - Method: POST  
      - Headers: Accept JSON, Authorization Bearer token (must be replaced with valid API key)  
      - Body: JSON with maxItems=50, searchTerms including "n8n developer", "looking for n8n", etc., sorted by Latest  
    - Inputs: Triggered by `6 hours` node  
    - Outputs: Returns scraped tweets JSON array  
    - Failure Modes: Invalid API key, rate limits, network errors, malformed response  
    - Notes: Sticky note reminds to replace API keys and adjust keywords and references Apify actor URL https://apify.com/apidojo/tweet-scraper

#### 1.3 Data Normalization

- **Overview:**  
  Standardizes tweet data fields to a defined schema used downstream in AI processing and storage.

- **Nodes Involved:**  
  - `Edit Fields`

- **Node Details:**

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Maps raw tweet data into structured fields: url, text, author.userName, author.url, author.location  
    - Configuration: Assigns each field from input JSON to normalized field names  
    - Inputs: Output of `HTTP Request` node  
    - Outputs: Passes normalized data to AI Agent  
    - Failure Modes: Missing or malformed tweet properties could cause empty fields  
    - Notes: None  

#### 1.4 AI Filtering & Deduplication

- **Overview:**  
  Uses a LangChain AI agent with OpenAI GPT-4 to determine if the post is a genuine hiring announcement for n8n/AI automation roles and checks Google Sheets to avoid duplicates.

- **Nodes Involved:**  
  - `AI Agent1` (LangChain Agent)  
  - `OpenAI Chat Model1` (GPT-4)  
  - `Get row(s) in sheet in Google Sheets1` (Google Sheets Query)  
  - `If` (Filter based on AI output)

- **Node Details:**

  - **AI Agent1**  
    - Type: LangChain Agent Node  
    - Role: Runs classification using custom prompt and rules for hiring detection and deduplication  
    - Configuration:  
      - System message defines role as "Hiring Post Classifier & Extractor"  
      - Input: Normalized tweet data (url, text, author info)  
      - Output: Strict JSON only for posts passing all rules, empty otherwise  
      - Uses Google Sheets tool to check for duplicates by URL  
    - Inputs: Normalized data from `Edit Fields`  
    - Outputs: JSON payload or empty data  
    - Failure Modes: AI rate limits, malformed prompt, API auth errors, Google Sheets access issues, empty results if no match  
    - Notes: Uses `OpenAI Chat Model1` and `Get row(s) in sheet in Google Sheets1` as AI language model and tool respectively

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model Node  
    - Role: Provides GPT-4.1-mini model for AI Agent  
    - Configuration: Model set to "gpt-4.1-mini"  
    - Inputs: From `AI Agent1` (ai_languageModel)  
    - Outputs: AI responses to `AI Agent1`  
    - Failure Modes: API key invalid, quota exhausted, network errors  

  - **Get row(s) in sheet in Google Sheets1**  
    - Type: Google Sheets Node (Tool)  
    - Role: Queries Google Sheet to check for existing post URL to avoid duplicates  
    - Configuration: Connects to Google Sheet named "X" by document ID and sheet GID=0  
    - Inputs: From `AI Agent1` (ai_tool)  
    - Outputs: Sheet data for deduplication  
    - Failure Modes: Credential issues, sheet not found, rate limits  

  - **If**  
    - Type: If Node  
    - Role: Filters only AI outputs that have non-empty `output` field (indicating a valid hiring post)  
    - Configuration: Condition checks if `{{$json.output}}` is not empty and not equal to empty JSON object `{}`  
    - Inputs: Output from `AI Agent1`  
    - Outputs: Passes valid posts to `Code1`; drops others  
    - Failure Modes: Expression errors if data malformed  

#### 1.5 Validation & Parsing

- **Overview:**  
  Safely extracts and parses the JSON output from the AI agent’s raw string response to ensure well-formed data downstream.

- **Nodes Involved:**  
  - `Code1`

- **Node Details:**

  - **Code1**  
    - Type: Code Node (JavaScript)  
    - Role: Parses the AI agent’s raw string output to extract valid JSON object safely  
    - Configuration:  
      - JavaScript code extracts substring between first '{' and last '}', then tries JSON.parse  
      - Errors logged but do not halt workflow  
    - Inputs: Output from `If` node  
    - Outputs: Parsed JSON objects with normalized hiring post data  
    - Failure Modes: Parsing errors if AI output malformed; handled gracefully  
    - Notes: Ensures entire workflow does not fail on single bad item  

#### 1.6 Output Handling

- **Overview:**  
  Appends or updates validated hiring posts into Google Sheets and sends alert notifications via Telegram with post details.

- **Nodes Involved:**  
  - `Append or update row in sheet1` (Google Sheets)  
  - `Send a text message1` (Telegram)

- **Node Details:**

  - **Append or update row in sheet1**  
    - Type: Google Sheets Node  
    - Role: Adds new rows or updates existing rows in the "X" Google Sheet based on post URL as unique key  
    - Configuration:  
      - Columns: url, text, author.userName, author.location, author.url  
      - Matching column: url (to detect duplicates)  
      - Document ID and Sheet Name configured for target spreadsheet  
    - Inputs: Parsed data from `Code1`  
    - Outputs: None (end of branch)  
    - Failure Modes: Credential expiration, API limits, sheet permission errors  

  - **Send a text message1**  
    - Type: Telegram Node  
    - Role: Sends a Telegram message alert with tweet details for new validated hiring posts  
    - Configuration:  
      - Text message template includes Tweet URL, Author, Author URL, Location, and Text  
      - Uses Telegram bot credentials  
    - Inputs: Parsed data from `Code1`  
    - Outputs: None (end of branch)  
    - Failure Modes: Bot token invalid, Telegram API errors, network issues  

#### 1.7 Documentation & Notes

- **Overview:**  
  Provides detailed workflow explanation, usage instructions, customization tips, and contact info in sticky notes for users and maintainers.

- **Nodes Involved:**  
  - `Sticky Note`  
  - `Sticky Note1`  
  - `Sticky Note2`

- **Node Details:**

  - **Sticky Note**  
    - Content:  
      - Explains full workflow purpose and step-by-step logic  
      - Target audience (freelancers, agencies, job seekers)  
      - Customization ideas (keywords, notifications, AI rules, CRM usage)  
    - Position: Left side, center of workflow canvas

  - **Sticky Note1**  
    - Content:  
      - Promotional info for tailored workflows via Reddit and website  
      - Links:  
        - Reddit: https://www.reddit.com/user/designbyaze/  
        - Website: https://hushtech.io/  
    - Position: Bottom-left  

  - **Sticky Note2**  
    - Content:  
      - Important setup instructions: Replace API keys, adjust role keywords, Apify actor reference  
      - Apify actor URL: https://apify.com/apidojo/tweet-scraper  
    - Position: Near HTTP Request node  

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                              | Input Node(s)              | Output Node(s)                       | Sticky Note                                                                                              |
|-----------------------------|----------------------------------|----------------------------------------------|----------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------|
| 6 hours                     | Schedule Trigger                  | Triggers workflow every 12 hours              | -                          | HTTP Request                       |                                                                                                        |
| HTTP Request                | HTTP Request                     | Scrapes tweets from Apify API                  | 6 hours                    | Edit Fields                       | ## 1. Replace the API keys in each Https node  ## 2. Add the specified role in the json part.  ## 3, The apify actor used: https://apify.com/apidojo/tweet-scraper |
| Edit Fields                 | Set                              | Normalizes tweet data fields                    | HTTP Request               | AI Agent1                        |                                                                                                        |
| AI Agent1                   | LangChain Agent                  | AI classification and deduplication             | Edit Fields                | If                               |                                                                                                        |
| OpenAI Chat Model1          | LangChain OpenAI Chat Model      | Provides GPT-4 model for AI processing          | AI Agent1 (ai_languageModel) | AI Agent1                       |                                                                                                        |
| Get row(s) in sheet in Google Sheets1 | Google Sheets Node (Tool)       | Queries Google Sheets to check duplicates       | AI Agent1 (ai_tool)        | AI Agent1                        |                                                                                                        |
| If                         | If                               | Filters only posts with non-empty AI output     | AI Agent1                  | Code1                           |                                                                                                        |
| Code1                      | Code                             | Parses AI raw string response into JSON         | If                         | Send a text message1, Append or update row in sheet1 |                                                                                                        |
| Append or update row in sheet1 | Google Sheets                    | Stores unique hiring posts into Google Sheets   | Code1                      | -                               |                                                                                                        |
| Send a text message1        | Telegram                         | Sends Telegram alert with hiring post details   | Code1                      | -                               |                                                                                                        |
| Sticky Note                 | Sticky Note                      | Workflow overview, instructions, use cases      | -                          | -                               | Automatically scrape X (Twitter) for posts hiring specific roles (e.g., automation engineers, video editors, graphic designers), filter true hiring intent with AI, deduplicate in Google Sheets, and alert via Telegram. See detailed instructions and customization ideas inside. |
| Sticky Note1                | Sticky Note                      | Promotional info and contact links              | -                          | -                               | # Looking for tailored workflows? Book through my Reddit or website—and follow for future templates. Reddit: https://www.reddit.com/user/designbyaze/ Website: https://hushtech.io/ |
| Sticky Note2                | Sticky Note                      | Setup instructions and Apify actor reference    | -                          | -                               | ## 1. Replace the API keys in each Https node ## 2. Add the specified role in the json part. ## 3, The apify actor used: https://apify.com/apidojo/tweet-scraper |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: `6 hours`  
   - Type: Schedule Trigger  
   - Set interval to every 12 hours  

2. **Create an HTTP Request Node**  
   - Name: `HTTP Request`  
   - Type: HTTP Request  
   - Connect input from `6 hours`  
   - Configure:  
     - URL: `https://api.apify.com/v2/acts/apidojo~tweet-scraper/run-sync-get-dataset-items?`  
     - Method: POST  
     - Headers:  
       - `Accept: application/json`  
       - `Authorization: Bearer <YOUR_APIFY_API_KEY>` (replace with your Apify API token)  
     - Body (JSON):  
       ```json
       {
         "maxItems": 50,
         "searchTerms": [
           "n8n developer",
           "looking for n8n",
           "n8n expert",
           "hire AI automation",
           "looking for AI automation"
         ],
         "sort": "Latest"
       }
       ```  
     - Set "Specify Body" to JSON  
   - Save credentials properly for API authentication  

3. **Create a Set Node to Normalize Data**  
   - Name: `Edit Fields`  
   - Type: Set  
   - Connect input from `HTTP Request`  
   - Assign fields:  
     - `url` = `{{$json.url}}`  
     - `text` = `{{$json.text}}`  
     - `author.userName` = `{{$json.author.userName}}`  
     - `author.url` = `{{$json.author.url}}`  
     - `author.location` = `{{$json.author.location}}`  

4. **Create LangChain OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model1`  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4.1-mini`  
   - Configure OpenAI API credentials  

5. **Create Google Sheets Node (Tool) for Deduplication**  
   - Name: `Get row(s) in sheet in Google Sheets1`  
   - Type: Google Sheets Node (Tool)  
   - Connect Google Sheets OAuth2 credentials  
   - Set Document ID to your target Google Sheet ID  
   - Set Sheet Name to `gid=0` (or your desired sheet)  

6. **Create LangChain Agent Node**  
   - Name: `AI Agent1`  
   - Type: LangChain Agent  
   - Connect input from `Edit Fields`  
   - Connect AI language model input from `OpenAI Chat Model1`  
   - Connect AI tool input from `Get row(s) in sheet in Google Sheets1`  
   - Paste the provided system message and prompt text (classifier rules, output format, deduplication instructions)  
   - Enable output parser to parse AI responses  

7. **Create If Node**  
   - Name: `If`  
   - Type: If  
   - Connect input from `AI Agent1`  
   - Set condition: `{{$json.output}}` is not empty and not equal to `{}` (empty JSON object)  

8. **Create Code Node to Parse AI Output Safely**  
   - Name: `Code1`  
   - Type: Code  
   - Connect input from `If` node (true branch)  
   - Paste JavaScript code to extract the JSON substring from AI raw output and parse it safely with error handling  

9. **Create Google Sheets Node to Append or Update Rows**  
   - Name: `Append or update row in sheet1`  
   - Type: Google Sheets  
   - Connect input from `Code1`  
   - Use same Google Sheets credentials and target sheet as before  
   - Configure columns:  
     - url  
     - text  
     - author (username)  
     - location  
     - url author  
   - Set matching column to `url` for deduplication  

10. **Create Telegram Node to Send Notifications**  
    - Name: `Send a text message1`  
    - Type: Telegram  
    - Connect input from `Code1`  
    - Configure Telegram Bot API credentials  
    - Set text to template:  
      ```
      Tweet URL: {{ $json.url }} 
      Author: {{ $json.author.userName }}
      Author URL: {{ $json.author.url }}
      Location: {{ $json.author.location }}
      Text: {{ $json.text }}
      ```  

11. **Connect Outputs**  
    - From `Code1` connect outputs to both `Append or update row in sheet1` and `Send a text message1` nodes  

12. **Add Sticky Notes for Documentation** (optional but recommended)  
    - Add workflow overview, instructions, API key reminders, and contact links as sticky notes for maintainers and users  

13. **Test the Workflow**  
    - Run manually or wait for schedule trigger  
    - Confirm tweets are fetched, filtered, deduplicated, stored, and alerts sent correctly  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Automatically scrape X (Twitter) for posts hiring specific roles (automation engineers, video editors, graphic designers), filter true hiring intent with AI, deduplicate in Google Sheets, and alert via Telegram. See detailed instructions and customization ideas inside. | Workflow sticky note in canvas                   |
| Replace API keys in HTTP Request nodes and Telegram credentials for proper authentication. Adjust keywords in the JSON body to monitor other roles if needed.                                                                                         | Sticky Note2 near HTTP Request node              |
| Apify Tweet Scraper actor used: https://apify.com/apidojo/tweet-scraper                                                                                                                                                                              | Sticky Note2 and HTTP Request node                |
| For tailored workflow development and templates, visit Reddit and the developer’s website: https://www.reddit.com/user/designbyaze/ and https://hushtech.io/                                                                                         | Sticky Note1                                       |
| AI filtering logic uses strict rules to exclude self-promotion and ensure only genuine hiring posts for n8n/AI automation roles. Deduplication relies on Google Sheets URL matching to avoid repeated alerts.                                           | AI Agent system prompt and workflow design notes |

---

This structured documentation provides a comprehensive understanding of the "Monitor X/Twitter for Hiring Posts with Apify, AI Filtering & Telegram Alerts" workflow, enabling users and developers to reproduce, maintain, and extend the automation confidently.