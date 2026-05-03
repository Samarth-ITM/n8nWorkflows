Filter TikTok Real Estate Videos for Couples with AI, Apify and Google Sheets

https://n8nworkflows.xyz/workflows/filter-tiktok-real-estate-videos-for-couples-with-ai--apify-and-google-sheets-11091


# Filter TikTok Real Estate Videos for Couples with AI, Apify and Google Sheets

### 1. Workflow Overview

This workflow automates the process of filtering TikTok real estate videos specifically for couples in their 20s, leveraging Apify for data scraping, AI agents for content filtering and recommendation generation, Google Sheets for data storage, and Slack for notification. It is designed for real estate agents and house hunters who want to efficiently discover and evaluate rental property videos on TikTok without manual browsing.

**Logical Blocks:**

- **1.1 Input Reception and Data Acquisition:**  
  Triggering the workflow manually and scraping TikTok videos tagged with specific hashtags using Apify.

- **1.2 AI Filtering and URL Extraction:**  
  Using an AI agent to analyze scraped videos and extract URLs of properties suitable for couples in their 20s.

- **1.3 AI Recommendation Generation:**  
  Feeding the filtered URLs back into another AI agent to generate reasons why each property is recommended for that demographic.

- **1.4 Data Storage and Notification:**  
  Appending or updating rows in Google Sheets with the filtered data and sending a Slack message with the recommendations.

- **1.5 Workflow Setup and Documentation:**  
  Sticky notes providing setup instructions, target audience, workflow explanation, and requirements.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Acquisition

**Overview:**  
This block initiates the workflow manually and runs a TikTok scraping actor via Apify to collect real estate video data filtered by a hashtag.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Run an Actor and get dataset

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow via manual execution in n8n UI.  
  - Configuration: No parameters; default manual trigger.  
  - Inputs: None  
  - Outputs: Connected to “Run an Actor and get dataset”  
  - Failure modes: None (manual trigger)  
  - No version-specific requirements.

- **Run an Actor and get dataset**  
  - Type: Apify node (Run actor and fetch dataset)  
  - Role: Executes the “TikTok Scraper” actor to scrape videos with hashtag “賃貸” (meaning “rental” in Japanese).  
  - Configuration:  
    - Actor ID: “GdWCkxBtKWOsKjdch” (TikTok Scraper)  
    - Input JSON includes:  
      - excludePinnedPosts: false  
      - hashtags: [“賃貸”]  
      - maxProfilesPerQuery: 1  
      - proxyCountryCode: None  
      - resultsPerPage: 1  
      - scrapeRelatedVideos and all download options set to false, indicating minimal data download.  
    - Authentication: OAuth2 configured for Apify  
  - Inputs: Trigger from manual node  
  - Outputs: Dataset passed to AI Agent  
  - Failure modes:  
    - API authentication failure  
    - Actor execution timeout or errors  
    - Empty or malformed dataset  
  - Version-specific: Requires Apify node version supporting “Run actor and get dataset” operation.

#### 1.2 AI Filtering and URL Extraction

**Overview:**  
This block uses a LangChain AI agent to analyze each video record and extract URLs of videos that feature rental properties suitable for couples in their 20s.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model

**Node Details:**  

- **AI Agent**  
  - Type: LangChain Agent node  
  - Role: Processes input JSON from the scraper, applies a prompt to select URLs for 20-something couples without extra explanation.  
  - Configuration:  
    - Text prompt:  
      ```
      20代カップル向けの物件を動画内から選択して下さい

      説明や前置きは省いて下さい
      URLのみを出力して下さい
      ---
      #URL
      {{ $json.webVideoUrl }}
      ```
    - Prompt Type: define (custom prompt)  
    - Inputs: Dataset from Apify node (each item includes video data with `webVideoUrl`)  
    - Outputs: Passes filtered URL to next AI agent  
  - Expressions: Uses `{{ $json.webVideoUrl }}` to inject URL into prompt.  
  - Failure modes:  
    - AI model timeout or rate limit  
    - Improper prompt formatting causing empty or invalid output  
    - Missing or malformed URLs in input JSON  

- **OpenRouter Chat Model**  
  - Type: LangChain Chat Language Model node  
  - Role: Provides the language model backend for the AI Agent.  
  - Configuration: Default options, connected as AI model to “AI Agent.”  
  - Failure modes: API key issues, rate limits, connectivity errors.

#### 1.3 AI Recommendation Generation

**Overview:**  
After URL extraction, this block uses a second AI agent to watch the video URLs and generate concise reasons why the property is recommended for 20-something couples.

**Nodes Involved:**  
- AI Agent1  
- OpenRouter Chat Model1

**Node Details:**  

- **AI Agent1**  
  - Type: LangChain Agent node  
  - Role: Takes the filtered URLs and prompts AI to output reasons tailored to couples in their 20s, without explanations or instructions, separating URL and description.  
  - Configuration:  
    - Text prompt:  
      ```
      URLの動画を見て、20代カップルにおすすめな理由を出力して下さい。
      説明や、指示文などは入れないで下さい
      URLと説明で分けて下さい

      {{ $json.output }}
      ```  
    - Prompt Type: define  
    - Input: The output from previous AI agent (filtered URLs)  
    - Output: JSON containing URL and recommendation text  
  - Expressions: Uses `{{ $json.output }}` to pass filtered URLs to prompt.  
  - Failure modes: Similar to first AI agent plus potential for empty or irrelevant recommendations if input URLs are invalid.

- **OpenRouter Chat Model1**  
  - Type: LangChain Chat Language Model node  
  - Role: Backend language model for AI Agent1.  
  - Configuration: Default.  
  - Failure modes: Same as previous OpenRouter model node.

#### 1.4 Data Storage and Notification

**Overview:**  
This block appends or updates the Google Sheet with the recommended videos and reasons, then sends a Slack message notifying the team about the new recommendations.

**Nodes Involved:**  
- Append or update row in sheet  
- Send a message

**Node Details:**  

- **Append or update row in sheet**  
  - Type: Google Sheets node  
  - Role: Stores the AI-generated recommendations in a specified sheet.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Sheet name: “gid=0” (default first sheet)  
    - Document ID: User-configured Google Sheets document ID  
    - Columns: Mapping set to auto map input data with a column named “output” containing the AI recommendation text.  
    - Matching columns: None specified (append only)  
    - Authentication: OAuth2 credentials for Google Sheets  
  - Inputs: From “AI Agent1” output with recommendation JSON  
  - Outputs: Connected to Slack notification node  
  - Failure modes:  
    - Credential expiration or invalid scopes  
    - Sheet ID or name misconfiguration  
    - Data format mismatch errors  

- **Send a message**  
  - Type: Slack node  
  - Role: Sends a message with the recommended property information to a Slack channel.  
  - Configuration:  
    - Text:  
      ```
      おすすめの物件情報です
      {{ $json.output }}
      ```  
    - Channel: User-selected Slack channel ID  
    - Authentication: OAuth2 Slack credentials  
  - Inputs: From Google Sheets node  
  - Failure modes:  
    - Slack API rate limits or authentication failures  
    - Invalid channel IDs  
    - Message formatting issues

#### 1.5 Workflow Setup and Documentation

**Overview:**  
Sticky notes provide contextual documentation, usage instructions, audience, and setup requirements.

**Nodes Involved:**  
- Sticky Note (positioned near AI Agent)  
- Sticky Note1 (positioned near manual trigger)  
- Sticky Note2 (positioned near Google Sheets node)

**Node Details:**  

- **Sticky Note**  
  - Content: Setup instructions explaining credential configuration, spreadsheet and Slack channel selection, and optional parameter adjustments.  
  - Useful for onboarding and initial testing.

- **Sticky Note1**  
  - Content: Explanation of intended users (real estate agents, house hunters) and workflow operation summary.

- **Sticky Note2**  
  - Content: Lists requirements including n8n version (v1.0+), active API accounts for Apify, OpenRouter, Google Cloud, and Slack with proper API keys and OAuth credentials.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                          | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                      |
|--------------------------------|----------------------------------|----------------------------------------|------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger                   | Starts the workflow manually           | None                         | Run an Actor and get dataset    | ### Who’s it for & How it works: This workflow helps real estate agents and house hunters automate TikTok research... |
| Run an Actor and get dataset    | Apify Node                      | Scrapes TikTok rental videos           | When clicking ‘Execute workflow’| AI Agent                      | Same as above                                                                                   |
| AI Agent                       | LangChain Agent                  | Filters videos to extract URLs for 20s couples | Run an Actor and get dataset | AI Agent1                      | ### How to set up: Import JSON, configure Apify, OpenRouter, Google Sheets, Slack credentials... |
| OpenRouter Chat Model           | LangChain Chat Model             | Provides AI language model for AI Agent| None (used internally)        | Connected to AI Agent           | Same as above                                                                                   |
| AI Agent1                      | LangChain Agent                  | Generates recommendation reasons       | AI Agent                     | Append or update row in sheet   | Same as above                                                                                   |
| OpenRouter Chat Model1          | LangChain Chat Model             | Provides AI language model for AI Agent1| None (used internally)        | Connected to AI Agent1          | Same as above                                                                                   |
| Append or update row in sheet   | Google Sheets                   | Stores recommendations in Google Sheets| AI Agent1                    | Send a message                 | ### Requirements: Active n8n instance, valid API keys for Apify, OpenRouter, Google Cloud, Slack |
| Send a message                 | Slack                          | Sends Slack notification with results  | Append or update row in sheet | None                          | Same as above                                                                                   |
| Sticky Note                   | Sticky Note                    | Setup instructions                      | None                         | None                          | ### How to set up: Import the JSON code into n8n and configure credentials...                    |
| Sticky Note1                  | Sticky Note                    | Workflow audience and summary           | None                         | None                          | ### Who’s it for & How it works: This workflow helps real estate agents and house hunters...     |
| Sticky Note2                  | Sticky Note                    | Workflow requirements                   | None                         | None                          | ### Requirements: You need an active n8n instance (v1.0+) and valid accounts for Apify...         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add node “Manual Trigger”  
   - Name: “When clicking ‘Execute workflow’”  
   - No special configuration needed.

2. **Create Apify Node**  
   - Add node “Apify” (ensure Apify credentials configured)  
   - Name: “Run an Actor and get dataset”  
   - Set operation to “Run actor and get dataset”  
   - Set Actor ID to “GdWCkxBtKWOsKjdch” (TikTok Scraper)  
   - Configure input JSON as:  
     ```json
     {
       "excludePinnedPosts": false,
       "hashtags": ["賃貸"],
       "maxProfilesPerQuery": 1,
       "proxyCountryCode": "None",
       "resultsPerPage": 1,
       "scrapeRelatedVideos": false,
       "shouldDownloadAvatars": false,
       "shouldDownloadCovers": false,
       "shouldDownloadMusicCovers": false,
       "shouldDownloadSlideshowImages": false,
       "shouldDownloadSubtitles": false,
       "shouldDownloadVideos": false
     }
     ```  
   - Connect “When clicking ‘Execute workflow’” → “Run an Actor and get dataset”.

3. **Create First AI Agent Node**  
   - Add “LangChain Agent” node (set up OpenRouter API credentials)  
   - Name: “AI Agent”  
   - Prompt Type: define  
   - Text prompt:  
     ```
     20代カップル向けの物件を動画内から選択して下さい

     説明や前置きは省いて下さい
     URLのみを出力して下さい
     ---
     #URL
     {{ $json.webVideoUrl }}
     ```  
   - Add “OpenRouter Chat Model” node, connect it as AI language model to “AI Agent”.  
   - Connect “Run an Actor and get dataset” → “AI Agent”.

4. **Create Second AI Agent Node**  
   - Add another “LangChain Agent” node  
   - Name: “AI Agent1”  
   - Prompt Type: define  
   - Text prompt:  
     ```
     URLの動画を見て、20代カップルにおすすめな理由を出力して下さい。
     説明や、指示文などは入れないで下さい
     URLと説明で分けて下さい

     {{ $json.output }}
     ```  
   - Add “OpenRouter Chat Model1” node and connect as AI language model for “AI Agent1”.  
   - Connect “AI Agent” → “AI Agent1”.

5. **Create Google Sheets Node**  
   - Add “Google Sheets” node  
   - Name: “Append or update row in sheet”  
   - Set operation to “appendOrUpdate”  
   - Define Sheet Name as first sheet (e.g., “gid=0”)  
   - Provide Google Sheets document ID (credential needed)  
   - Columns mapping mode: Auto map input data, ensure “output” column exists in the sheet  
   - Connect “AI Agent1” → “Append or update row in sheet”.

6. **Create Slack Node**  
   - Add “Slack” node  
   - Name: “Send a message”  
   - Configure OAuth2 credentials for Slack  
   - Select target channel ID  
   - Message text:  
     ```
     おすすめの物件情報です
     {{ $json.output }}
     ```  
   - Connect “Append or update row in sheet” → “Send a message”.

7. **Add Sticky Notes for Documentation**  
   - Add three sticky notes with content from the original workflow for:  
     - Who the workflow is for and how it works  
     - Setup instructions and credential configuration  
     - Requirements and version notes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow automates TikTok real estate video filtering for couples aged 20s, combining Apify scraping, LangChain AI filtering, Google Sheets storage, and Slack notifications. | Workflow Purpose Summary                                                                           |
| Requires valid API keys and OAuth2 credentials for Apify, OpenRouter, Google Cloud (Sheets), and Slack.                                                          | Credential and Access Requirements                                                                |
| Adjust hashtags, AI prompts, Google Sheets document ID, and Slack channel ID as per your specific use case before running.                                       | Configuration Flexibility                                                                          |
| Original TikTok Scraper Actor used: https://console.apify.com/actors/GdWCkxBtKWOsKjdch                                                                           | Apify Actor Link                                                                                   |
| For AI processing, OpenRouter integration is used as the language model backend.                                                                                  | OpenRouter API Reference                                                                           |

---

**Disclaimer:** The provided content is derived exclusively from an automated n8n workflow. It complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.