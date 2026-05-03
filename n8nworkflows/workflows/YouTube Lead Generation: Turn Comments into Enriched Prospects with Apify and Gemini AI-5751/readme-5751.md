YouTube Lead Generation: Turn Comments into Enriched Prospects with Apify and Gemini AI

https://n8nworkflows.xyz/workflows/youtube-lead-generation--turn-comments-into-enriched-prospects-with-apify-and-gemini-ai-5751


# YouTube Lead Generation: Turn Comments into Enriched Prospects with Apify and Gemini AI

### 1. Workflow Overview

This workflow automates the process of generating leads by extracting YouTube video comments, enriching the profiles of comment authors using AI-assisted web research, and storing the data in Google Sheets. It is designed for marketing, sales, or research teams aiming to identify and gather contact and social media information about active YouTube commenters.

The workflow is logically divided into two main blocks:

**1.1 YouTube Comment Extraction**  
This block fetches YouTube video URLs from a Google Sheet, scrapes the comments on those videos using an Apify actor, and stores both the processed videos and extracted comments into Google Sheets. It supports both manual and scheduled triggers for flexibility.

**1.2 Lead Research & Enrichment (AI Agent)**  
This block uses an AI agent powered by Google Gemini via OpenRouter, along with web scraping tools and Google Search API, to autonomously research the profiles of comment authors. It conducts systematic, exhaustive searches across social media platforms and websites to enrich author data and save results into a dedicated “leads” Google Sheet. It also marks comments as processed after enrichment.

---

### 2. Block-by-Block Analysis

#### 2.1 YouTube Comment Extraction

**Overview:**  
Automates retrieval of YouTube video URLs, scrapes comments for each video using an external Apify actor, and stores the comments and video status in Google Sheets. Supports manual and scheduled execution.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Schedule Trigger1 (Schedule Trigger)  
- Get rvideo urls (Google Sheets Tool)  
- HTTP apify get comments from video (HTTP Request)  
- mark video url as scrapped (Google Sheets Tool)  
- Save scrapped comments (Google Sheets Tool)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Configuration: Default, no parameters  
  - Outputs to: Get rvideo urls  
  - Edge cases: Manual trigger might be forgotten or misused during automation phases

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Automates the start of comment extraction every hour  
  - Configuration: Interval set to every hour  
  - Outputs to: Get rvideo urls  
  - Edge cases: Network failures or schedule misconfiguration may cause missed runs

- **Get rvideo urls**  
  - Type: Google Sheets Tool  
  - Role: Reads YouTube video URLs from the “videos” sheet of “youtube leads” Google Sheet  
  - Configuration: Filters rows where `scrapped` column is empty or false to avoid duplicates  
  - Inputs from: When clicking ‘Execute workflow’ or Schedule Trigger1  
  - Outputs to: HTTP apify get comments from video, mark video url as scrapped  
  - Edge cases: Google Sheets API quota limits, malformed URLs, or missing data

- **HTTP apify get comments from video**  
  - Type: HTTP Request  
  - Role: Calls Apify Actor (mohamedgb00714/youtube-video-comments) to scrape comments from a video URL  
  - Configuration: POST request with JSON body containing `"videoUrl": "{{ $json.url }}"` dynamically populated  
  - Inputs from: Get rvideo urls  
  - Outputs to: Save scrapped comments  
  - Edge cases: Apify token invalid or expired, API timeouts, malformed video URLs, unexpected response format

- **mark video url as scrapped**  
  - Type: Google Sheets Tool  
  - Role: Updates “videos” sheet to mark the processed video URL as scraped (`scrapped = TRUE`)  
  - Configuration: Matches rows on `url` column, updates `scrapped` column  
  - Inputs from: Get rvideo urls (parallel to HTTP apify get comments)  
  - Outputs: None (terminal for this branch)  
  - Edge cases: Update failures due to Google Sheets API issues, concurrent write conflicts

- **Save scrapped comments**  
  - Type: Google Sheets Tool  
  - Role: Appends scraped comment data into the “comments” sheet  
  - Configuration: Auto-maps comment fields like `author`, `content`, `publishedTime` to the sheet columns  
  - Inputs from: HTTP apify get comments from video  
  - Outputs: None (terminal node)  
  - Edge cases: Data mapping issues, Google Sheets API quota limits, malformed comment data

---

#### 2.2 Lead Research & Enrichment (AI Agent)

**Overview:**  
Uses an AI agent to autonomously research YouTube comment authors, find social media profiles and contact details, and update a “leads” Google Sheet with enriched data. It uses a chat trigger or schedule trigger to begin processing, calls multiple tools including Google Search API, Apify web scrapers, and manages state with memory nodes. Marks comments as processed after enrichment.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- Schedule Trigger (Schedule Trigger)  
- Code  
- AI Agent (LangChain Agent)  
- OpenRouter Chat Model (LLM)  
- Simple Memory (Memory Buffer Window)  
- get comments (Google Sheets Tool)  
- search google (HTTP Request Tool)  
- get url mardown (HTTP Request Tool)  
- create a row for new search user (Google Sheets Tool)  
- update result for user (Google Sheets Tool)  
- instagram full profile scraper (HTTP Request Tool)  
- mark comment as processed (Google Sheets Tool)

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (LangChain)  
  - Role: Entry point to start AI agent upon receiving a chat message  
  - Configuration: Webhook with ID for chat events  
  - Outputs to: AI Agent  
  - Edge cases: Missed or malformed chat events, webhook unavailability

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers the lead enrichment process every 1 minute (rapid automation)  
  - Outputs to: Code  
  - Edge cases: Overlapping runs if processing time exceeds interval, API rate limits

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Prepares initial input for AI Agent, setting a detailed `chatInput` prompt instructing full autonomous user info gathering, generates a unique `sessionId`  
  - Inputs from: Schedule Trigger  
  - Outputs to: AI Agent  
  - Edge cases: Code syntax errors, improper prompt generation

- **AI Agent**  
  - Type: LangChain AI Agent  
  - Role: Orchestrates autonomous research workflow, systematically calls tools for data gathering, processes input comments/authors  
  - Configuration:  
    - Max iterations: 100  
    - System message defines exhaustive social media and contact info search behavior  
    - Tools integrated: get_comments(), Google Search, Browse, create_new_row(), save_to_sheet()  
  - Inputs from: Code, get comments (as AI tool), Simple Memory (AI memory), search google (AI tool), get url mardown (AI tool), instagram full profile scraper (AI tool), create a row for new search user (AI tool), update result for user (AI tool), mark comment as processed (AI tool)  
  - Outputs to: None directly - updates sheets via tools  
  - Edge cases: LLM errors, API rate limits, malformed data from tools, incomplete or ambiguous search results  
  - On error: configured to continue regular output to avoid halting

- **OpenRouter Chat Model**  
  - Type: LangChain LLM (OpenRouter)  
  - Role: Provides reasoning and generation for AI Agent using Google Gemini 2.5 model  
  - Configuration: Model `google/gemini-2.5-flash-preview-05-20`, no extra options  
  - Inputs from: AI Agent  
  - Outputs to: AI Agent  
  - Edge cases: API key invalid, request timeouts, model response errors

- **Simple Memory**  
  - Type: Memory Buffer Window  
  - Role: Maintains short-term conversational memory for AI Agent per session  
  - Inputs from: AI Agent  
  - Outputs to: AI Agent  
  - Edge cases: Memory overflow, loss of context if node fails

- **get comments**  
  - Type: Google Sheets Tool  
  - Role: Retrieves comments from “comments” sheet where `processed` is false or empty for AI Agent processing  
  - Configuration: Auto-detects data range, filters on `processed` column  
  - Inputs from: None (triggered by AI Agent)  
  - Outputs to: AI Agent as a tool  
  - Edge cases: API quota, data inconsistencies

- **search google**  
  - Type: HTTP Request Tool (Serper API)  
  - Role: Performs Google searches to find social media profiles, emails, or websites  
  - Configuration:  
    - POST to `https://google.serper.dev/search`  
    - Query parameter dynamically provided by AI Agent  
    - Location fixed to United States  
    - Number of results: 100  
    - API key required (Serper API)  
  - Inputs from: AI Agent  
  - Outputs to: AI Agent  
  - Edge cases: API quota exceeded, invalid API key, incomplete or irrelevant results

- **get url mardown**  
  - Type: HTTP Request Tool (Apify Actor)  
  - Role: Scrapes website content as Markdown from URLs found during research  
  - Configuration:  
    - POST to Apify firescraper AI website content scraper  
    - Takes URL and crawl flag from AI Agent variables  
    - Proxy enabled for requests  
    - Max pages set to 10  
  - Inputs from: AI Agent  
  - Outputs to: AI Agent  
  - Edge cases: Apify token issues, scraping blocked by sites, unexpected content structure

- **create a row for new search user**  
  - Type: Google Sheets Tool  
  - Role: Adds or updates a new row in “leads” sheet for a newly discovered user  
  - Configuration: Matching on `username` column, populates username field  
  - Inputs from: AI Agent  
  - Outputs to: AI Agent  
  - Edge cases: Duplicate usernames, Google Sheets API rate limits

- **update result for user**  
  - Type: Google Sheets Tool  
  - Role: Appends or updates enriched data collected for a user to “leads” sheet  
  - Configuration: Matches on `username`, updates fields like email, bio, social links, short description  
  - Inputs from: AI Agent  
  - Outputs to: AI Agent  
  - Edge cases: Partial data, update conflicts

- **instagram full profile scraper**  
  - Type: HTTP Request Tool (Apify Actor)  
  - Role: Given an Instagram username, scrapes detailed profile information  
  - Configuration:  
    - POST to Apify Instagram full profile scraper  
    - Sends Instagram usernames array dynamically from AI Agent  
  - Inputs from: AI Agent  
  - Outputs to: AI Agent  
  - Edge cases: Instagram anti-bot protections, Apify token expiration

- **mark comment as processed**  
  - Type: Google Sheets Tool  
  - Role: Updates “comments” sheet to mark a comment as processed (`processed = true`), preventing re-processing  
  - Configuration: Matches on `avatarAccessibilityText` column  
  - Inputs from: AI Agent  
  - Outputs: None  
  - Edge cases: Update failures, concurrency issues

---

### 3. Summary Table

| Node Name                          | Node Type                      | Functional Role                                 | Input Node(s)                       | Output Node(s)                                      | Sticky Note                                                                                                              |
|----------------------------------|--------------------------------|------------------------------------------------|-----------------------------------|----------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Manual start of comment extraction             | —                                 | Get rvideo urls                                    | See Sticky Note1 for full workflow description                                                                           |
| Schedule Trigger1                | Schedule Trigger               | Automated hourly start of comment extraction   | —                                 | Get rvideo urls                                    | See Sticky Note1 for full workflow description                                                                           |
| Get rvideo urls                 | Google Sheets Tool             | Reads YouTube video URLs to process             | When clicking ‘Execute workflow’, Schedule Trigger1 | HTTP apify get comments from video, mark video url as scrapped | See Sticky Note1 for full workflow description                                                                           |
| HTTP apify get comments from video | HTTP Request                  | Calls Apify actor to scrape video comments      | Get rvideo urls                   | Save scrapped comments                             | See Sticky Note1 for full workflow description                                                                           |
| mark video url as scrapped     | Google Sheets Tool             | Marks video URL as scraped                        | Get rvideo urls                   | —                                                  | See Sticky Note1 for full workflow description                                                                           |
| Save scrapped comments         | Google Sheets Tool             | Saves scraped comments to Google Sheet           | HTTP apify get comments from video | —                                                  | See Sticky Note1 for full workflow description                                                                           |
| When chat message received      | Chat Trigger                  | Triggers AI agent on chat message                | —                                 | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| Schedule Trigger               | Schedule Trigger               | Automates AI agent invocation every 1 minute    | —                                 | Code                                               | See Sticky Note1 for full workflow description                                                                           |
| Code                          | Code (JavaScript)              | Prepares input prompt and sessionId for AI Agent | Schedule Trigger                 | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| AI Agent                      | LangChain Agent                | Core AI orchestrator for autonomous lead enrichment | Code, get comments, Simple Memory, search google, get url mardown, instagram full profile scraper, create a row for new search user, update result for user, mark comment as processed | OpenRouter Chat Model                                  | See Sticky Note1 for full workflow description                                                                           |
| OpenRouter Chat Model          | LLM (OpenRouter)               | Provides reasoning and generation for AI Agent  | AI Agent                         | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| Simple Memory                  | Memory Buffer Window           | Maintains session memory for AI Agent            | AI Agent                         | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| get comments                  | Google Sheets Tool             | Retrieves unprocessed comments for AI Agent      | —                                 | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| search google                 | HTTP Request Tool             | Performs Google searches for profile enrichment  | AI Agent                         | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| get url mardown               | HTTP Request Tool             | Scrapes website content in Markdown format       | AI Agent                         | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| create a row for new search user | Google Sheets Tool           | Adds new user entry into leads sheet              | AI Agent                         | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| update result for user        | Google Sheets Tool             | Updates enriched user data in leads sheet         | AI Agent                         | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| instagram full profile scraper | HTTP Request Tool             | Scrapes Instagram profile details given username | AI Agent                         | AI Agent                                           | See Sticky Note1 for full workflow description                                                                           |
| mark comment as processed     | Google Sheets Tool             | Marks comment as processed in comments sheet      | AI Agent                         | —                                                  | See Sticky Note1 for full workflow description                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Execute workflow’` with default settings.

2. **Create a Schedule Trigger node** named `Schedule Trigger1` configured to run every hour.

3. **Create a Google Sheets Tool node** named `Get rvideo urls`:
   - Connect inputs from `When clicking ‘Execute workflow’` and `Schedule Trigger1`.
   - Configure to read from Google Sheet document ID `1OGX1bKAOeym8tEENP-zJAYXyZD6XnH6T5FV85J38H54`, sheet with gid `0` (named "videos").
   - Add filter to select rows where column `scrapped` is not TRUE.
   - Use Google Sheets OAuth2 credentials.

4. **Create an HTTP Request node** named `HTTP apify get comments from video`:
   - Connect input from `Get rvideo urls`.
   - Method: POST.
   - URL: `https://api.apify.com/v2/acts/mohamedgb00714~youtube-video-comments/run-sync-get-dataset-items?token={{your apify token}}`.
   - Content type: JSON.
   - Body: `{"videoUrl": "{{ $json.url }}"}` (use expression to pull URL from incoming data).
   - Specify body as JSON.
   - Version 4.2 or higher recommended.

5. **Create a Google Sheets Tool node** named `mark video url as scrapped`:
   - Connect input from `Get rvideo urls` (parallel to HTTP Request node).
   - Operation: Update.
   - Update rows in the same sheet as `Get rvideo urls` (document and sheet same).
   - Matching column: `url`.
   - Set column `scrapped` to `TRUE`.
   - Use Google Sheets OAuth2 credentials.

6. **Create a Google Sheets Tool node** named `Save scrapped comments`:
   - Connect input from `HTTP apify get comments from video`.
   - Operation: Append.
   - Sheet: "comments" sheet with gid `6484598` in same Google Sheet document.
   - Map all relevant comment fields automatically.
   - Use Google Sheets OAuth2 credentials.

7. **Create a Schedule Trigger node** named `Schedule Trigger`:
   - Configure to run every 1 minute.

8. **Create a Code node** named `Code`:
   - Connect input from `Schedule Trigger`.
   - Paste given JavaScript code that sets a detailed prompt in `chatInput` and generates a unique `sessionId`.
   - Version 2 or higher.

9. **Create a LangChain Chat Trigger node** named `When chat message received`:
   - Configure webhook ID.
   - This node acts as second entry point for AI Agent to start from chat messages.

10. **Create a LangChain Agent node** named `AI Agent`:
    - Connect inputs from `Code`, `When chat message received`, and AI tools (see below).
    - Set system message with detailed instructions for autonomous research.
    - Enable max iterations 100.
    - On error: continue regular output.
    - Connect AI language model and AI memory as below.

11. **Create a LangChain LLM node** named `OpenRouter Chat Model`:
    - Connect input from `AI Agent` (ai_languageModel).
    - Model: `google/gemini-2.5-flash-preview-05-20`.
    - Use OpenRouter API credentials.

12. **Create a LangChain Memory Buffer Window node** named `Simple Memory`:
    - Connect input and output to `AI Agent` (ai_memory).
    - Default settings.

13. **Create a Google Sheets Tool node** named `get comments`:
    - Used as AI tool by `AI Agent`.
    - Reads from "comments" sheet.
    - Filter: only comments where `processed` is false or empty.
    - Use Google Sheets OAuth2 credentials.

14. **Create an HTTP Request Tool node** named `search google`:
    - Used as AI tool.
    - POST to `https://google.serper.dev/search`.
    - Headers include `X-API-KEY` with your Serper API key.
    - Body parameters: query string `q` dynamically from AI Agent.
    - Location fixed to United States.
    - Number of results: 100.

15. **Create an HTTP Request Tool node** named `get url mardown`:
    - Used as AI tool.
    - POST to Apify firescraper AI website content scraper endpoint with your Apify token.
    - Body includes `startUrls` with URL from AI Agent.
    - Proxy enabled.

16. **Create a Google Sheets Tool node** named `create a row for new search user`:
    - Used as AI tool.
    - Operation: Append or Update.
    - Target sheet: "leads" sheet (gid `395221622`) in same document.
    - Matching column: `username`.
    - Columns: at minimum `username`.
    - Use Google Sheets OAuth2 credentials.

17. **Create a Google Sheets Tool node** named `update result for user`:
    - Used as AI tool.
    - Operation: Append or Update.
    - Same "leads" sheet.
    - Matching column: `username`.
    - Columns updated: bio, email, GitHub, TikTok, Facebook, LinkedIn, Instagram, Twitter, short Description.
    - Use Google Sheets OAuth2 credentials.

18. **Create an HTTP Request Tool node** named `instagram full profile scraper`:
    - Used as AI tool.
    - POST to Apify Instagram full profile scraper with your Apify token.
    - Body includes Instagram usernames from AI Agent.

19. **Create a Google Sheets Tool node** named `mark comment as processed`:
    - Used as AI tool.
    - Operation: Update.
    - Matches on `avatarAccessibilityText`.
    - Sets `processed` to true.
    - Use Google Sheets OAuth2 credentials.

20. **Connect all AI tools to the AI Agent node** via appropriate ai_tool connections.

21. **Connect the OpenRouter Chat Model as `ai_languageModel` and Simple Memory as `ai_memory` to the AI Agent.**

22. **Ensure all credentials (Google Sheets OAuth2, Apify token, Serper API key, OpenRouter API key) are created and linked in n8n.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates YouTube lead generation by scraping video comments, then enriching commenter profiles via AI-driven research combining Google Search, website scraping, and social media profile extraction. It saves all data into Google Sheets for further use. The AI agent is designed for fully autonomous, exhaustive research without repeated confirmations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Workflow purpose and design principles                                                                                                                                                                                             |
| Apify Actors used: [YouTube Video Comments Scraper](https://apify.com/mohamedgb00714/youtube-video-comments), [FireScraper AI Website Content Markdown Scraper](https://apify.com/mohamedgb00714/firescraper-ai-website-content-markdown-scraper), [Instagram Full Profile Scraper](https://apify.com/mohamedgb00714/instagram-full-profile-scraper).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | External scraping tools integrated                                                                                                                                                                                                |
| OpenRouter API key and usage instructions: Obtain from [openrouter.ai](https://openrouter.ai/keys). Used for accessing Google Gemini models.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | AI language model access                                                                                                                                                                                                            |
| Serper API key: Obtain from [serper.dev](https://serper.dev) to enable Google Search capability within the workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Search API for profile enrichment                                                                                                                                                                                                 |
| Google Sheets OAuth2 credentials: Configure in n8n Credentials section with access to relevant Google Drive and Sheets.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Google Sheets API access                                                                                                                                                                                                            |
| The system message in the AI Agent includes detailed instructions for persistent, exhaustive search including social media platforms to target: Instagram, Twitter (X), LinkedIn, Facebook, TikTok, GitHub, and others. It enforces autonomous operation without intermediate confirmations and requires explicit marking of processed comments and users.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | AI Agent operational guidelines                                                                                                                                                                                                    |
| Sticky notes within the workflow provide detailed explanations of sections, node purposes, and API key management instructions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Documentation embedded in workflow for user reference                                                                                                                                                                            |

---

This document fully describes the YouTube Lead Generation workflow, enabling understanding, reproduction, and modification by users or AI agents. It highlights all nodes, their configuration, and the logical flow between them, as well as integration considerations and potential failure points.