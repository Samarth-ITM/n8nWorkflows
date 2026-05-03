Multi Platform Content Generator from YouTube using AI & RSS

https://n8nworkflows.xyz/workflows/multi-platform-content-generator-from-youtube-using-ai---rss-6843


# Multi Platform Content Generator from YouTube using AI & RSS

### 1. Workflow Overview

This workflow, titled **Multi Platform Content Generator from YouTube using AI & RSS**, automates the process of monitoring YouTube channels, extracting video transcripts, and generating tailored social media content for multiple platforms. It leverages AI-driven summarization and content generation to produce platform-optimized posts for LinkedIn, X (formerly Twitter), Threads, and Instagram. The workflow also integrates with Google Sheets for centralized data management and Telegram for content preview notifications.

The workflow is logically divided into four main blocks:

- **1.1 Channel Monitoring:** Periodic trigger initiates the process by fetching YouTube channel URLs from Google Sheets, extracting channel IDs via HTTP and HTML parsing, and reading the latest videos via RSS feeds.

- **1.2 Content Processing:** Using Supadata API, it retrieves full video transcripts, merges transcript segments, and applies AI summarization to create concise video summaries.

- **1.3 Multi-Platform Content Generation:** Based on the summarized transcript, AI models generate customized posts for LinkedIn, X, Threads, and Instagram, adhering to each platform's style and character limits.

- **1.4 Data Management and Notifications:** All video metadata, summaries, and generated posts are stored or updated in Google Sheets. Telegram messages deliver previews of generated content for easy monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Channel Monitoring

- **Overview:**  
  This block triggers the workflow on a schedule, reads YouTube channel URLs from Google Sheets, extracts channel IDs from the page source, and constructs corresponding RSS feed URLs to fetch recent video metadata.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get row(s) in sheet (Google Sheets read)  
  - HTTP Request  
  - HTML  
  - Code (JavaScript)  
  - RSS Read  
  - Append or update row in sheet (Google Sheets write)  
  - Get row(s) in sheet2 (Google Sheets read)  
  - If

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger  
    - Role: Initiates the workflow on a periodic schedule (default every minute/hour depending on config).  
    - Config: Interval-based trigger with default interval settings.  
    - Input: None  
    - Output: Triggers downstream nodes.  
    - Failures: None critical; misconfiguration or disabled trigger stops workflow runs.

  - **Get row(s) in sheet**  
    - Type: Google Sheets read  
    - Role: Reads YouTube channel URLs from a specified sheet and document ID.  
    - Config: Reads all rows from a sheet named "YOUR_SHEET_NAME" in document "YOUR_GOOGLE_SHEET_ID". Uses Google Sheets OAuth2 credentials.  
    - Expressions: None.  
    - Input: Trigger from Schedule Trigger  
    - Output: List of channel URLs.  
    - Failures: Authentication errors, invalid sheet or doc ID, empty data.  
    - Credential required: Google Sheets OAuth2.

  - **HTTP Request**  
    - Type: HTTP GET  
    - Role: Fetches the HTML content of each YouTube channel URL from the sheet.  
    - Config: URL dynamically set using expression `{{$json['channel url']}}` from sheet data.  
    - Input: Channel URLs  
    - Output: Raw HTML page content.  
    - Failures: Network errors, invalid URLs, HTTP errors (404, 403).  
    - Retry recommended in case of transient errors.

  - **HTML**  
    - Type: HTML parser  
    - Role: Extracts the canonical channel URL from the fetched HTML to obtain the channel ID.  
    - Config: CSS selector `link[rel="canonical"]`, extracts `href` attribute.  
    - Input: HTML content from HTTP Request  
    - Output: Extracted channel URL.  
    - Failures: If selector misses or HTML changes, extraction fails.

  - **Code**  
    - Type: JavaScript function  
    - Role: Parses the full channel URL to isolate the channel ID and builds the YouTube RSS feed URL.  
    - Config: Custom JS code splits URL string and constructs RSS feed URL `https://www.youtube.com/feeds/videos.xml?channel_id=CHANNEL_ID`.  
    - Input: Extracted channel URL  
    - Output: JSON with `channel_id` and `rss_url`.  
    - Failures: Malformed URLs or unexpected URL structure.

  - **RSS Read**  
    - Type: RSS feed reader  
    - Role: Reads the latest video entries from the constructed YouTube RSS feed URL.  
    - Config: URL set dynamically to `{{$json.rss_url}}`.  
    - Input: RSS feed URL from Code node  
    - Output: Video metadata including video ID, title, author, publication date, and link.  
    - Failures: RSS feed unreachable, empty feed, or invalid URL.

  - **Append or update row in sheet**  
    - Type: Google Sheets write  
    - Role: Stores or updates video metadata in the Google Sheet to maintain a record of videos.  
    - Config: Maps video fields (id, link, title, author, pubDate) into sheet columns, matching on video ID to avoid duplicates.  
    - Input: RSS video metadata  
    - Output: Updated Google Sheets row  
    - Failures: Authentication, sheet access, or schema mismatch.  
    - Credential required: Google Sheets OAuth2.

  - **Get row(s) in sheet2**  
    - Type: Google Sheets read  
    - Role: Reads the updated sheet to check if the video summary field is already populated, helping avoid duplicate processing.  
    - Config: Reads all rows from the same sheet.  
    - Input: After updating sheet with video metadata  
    - Output: Rows with existing data including summaries.  
    - Failures: Same as above.

  - **If**  
    - Type: Conditional check  
    - Role: Determines whether the video summary is empty (i.e., the video has not been processed yet).  
    - Config: Condition checks if `summary` field is empty.  
    - Input: Rows from Get row(s) in sheet2  
    - Output:  
      - True: Proceed to transcript extraction and content generation  
      - False: Skip processing (No Operation node)  
    - Failures: Expression errors or data inconsistencies.

---

#### 2.2 Content Processing

- **Overview:**  
  This block extracts full transcripts of videos via Supadata API, merges segmented transcripts into a complete text, and sets the output language for further AI summarization.

- **Nodes Involved:**  
  - If (true branch)  
  - HTTP Request1 (Supadata API call)  
  - Merge Transcripts (Code)  
  - Set Language

- **Node Details:**

  - **HTTP Request1**  
    - Type: HTTP GET  
    - Role: Calls Supadata API to retrieve the transcript of a YouTube video URL.  
    - Config: URL template with dynamic video link: `https://api.supadata.ai/v1/transcript?url={{ $json.link }}`  
      Includes header `x-api-key` with Supadata API key.  
    - Input: Video link from If node  
    - Output: Transcript data as JSON containing transcript parts.  
    - Failures: API key invalid/expired, rate limits, network errors.

  - **Merge Transcripts**  
    - Type: JavaScript function  
    - Role: Joins multiple transcript parts into a single continuous transcript string.  
    - Config: Extracts `content` array from API response and concatenates all `text` fields.  
    - Input: Transcript JSON from HTTP Request1  
    - Output: Single field `full_transcript` containing complete transcript text.  
    - Failures: Missing or malformed transcript data.

  - **Set Language**  
    - Type: Set node  
    - Role: Defines the language context (hardcoded as "English") for downstream AI summarization and content generation.  
    - Config: Static assignment of `"language": "English"`.  
    - Input: Merged transcript data  
    - Output: Adds `language` field to JSON.

---

#### 2.3 Multi-Platform Content Generation

- **Overview:**  
  This block uses AI language models to create a video summary and generate platform-specific posts tailored by style, tone, and character limits for LinkedIn, X, Threads, and Instagram.

- **Nodes Involved:**  
  - Basic LLM Chain (Summary generation)  
  - Append or update row in sheet1 (save summary)  
  - Basic LLM Chain1 + OpenRouter Chat Model1 (LinkedIn post)  
  - Basic LLM Chain2 + OpenRouter Chat Model2 (X post)  
  - Basic LLM Chain3 + OpenRouter Chat Model3 (Threads post)  
  - Basic LLM Chain4 + OpenRouter Chat Model4 (Instagram caption)  
  - Append or update row in sheet2 (LinkedIn write)  
  - Append or update row in sheet3 (X write)  
  - Append or update row in sheet5 (Threads write)  
  - Append or update row in sheet4 (Instagram write)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain  
    - Role: Generates a concise summary of the full transcript.  
    - Config: Prompts AI to summarize the video transcript focusing on main points, speaker, topics, insights, in English.  
    - Input: Full transcript, language context  
    - Output: Summary text in `text` field.  
    - Failures: AI service errors, token limits.

  - **Append or update row in sheet1**  
    - Type: Google Sheets write  
    - Role: Saves the generated summary to the sheet linked to the corresponding video ID.  
    - Config: Updates the `summary` column matched by `id`.  
    - Input: Summary text from Basic LLM Chain  
    - Output: Updated sheet row.

  - **Basic LLM Chain1**, **Basic LLM Chain2**, **Basic LLM Chain3**, **Basic LLM Chain4**  
    - Type: LangChain LLM Chain  
    - Role: Generate social media posts based on the summary for LinkedIn, X, Threads, and Instagram respectively.  
    - Config: Each chain uses a distinct prompt specifying tone, style, character limits, and format per platform.  
      Examples:  
      - LinkedIn: Storytelling, ≤1300 chars, professional tone  
      - X: Punchy, ≤280 chars, conversational  
      - Threads: Casual, friendly, 1-3 paragraphs  
      - Instagram: Short, emotionally resonant captions, ≤150 chars per sentence  
    - Input: Summary text from Basic LLM Chain  
    - Output: Platform-specific post text in `text` field.  
    - Failures: AI service errors, prompt misinterpretation.

  - **OpenRouter Chat Model1 - 4**  
    - Type: OpenRouter AI model nodes  
    - Role: Provide AI language model inference for each Basic LLM Chain node.  
    - Config: Uses Google Gemini 2.0 flash experimental free model.  
    - Input: Text prompts from Basic LLM Chain nodes  
    - Output: AI-generated text responses.  
    - Failures: API key issues, rate limits, model downtime.

  - **Append or update row in sheet2, sheet3, sheet4, sheet5**  
    - Type: Google Sheets write  
    - Role: Save generated posts into respective columns (`linkedin`, `x`, `instagram`, `threads`) in the sheet, matched by video ID.  
    - Config: Updates specific platform columns, matching by `id`.  
    - Input: Generated post texts from AI chains  
    - Output: Updated Google Sheets rows.  
    - Failures: Authentication, sheet permission or schema issues.

---

#### 2.4 Data Management and Notifications

- **Overview:**  
  After content generation, this block sends a Telegram message with the collected posts for quick review and monitoring.

- **Nodes Involved:**  
  - Send a text message (Telegram)  
  - Append or update row in sheet2, sheet3, sheet4, sheet5 (preceding writes)  
  - Sticky Note7 (comment)

- **Node Details:**

  - **Send a text message**  
    - Type: Telegram message  
    - Role: Sends a formatted message to a specified Telegram chat containing the video title, author, and all generated platform posts.  
    - Config: Message text composed with expressions referencing post texts and video metadata.  
    - Input: Outputs from Google Sheets update nodes (post texts)  
    - Output: Sent Telegram message  
    - Credential required: Telegram Bot Token and Chat ID  
    - Failures: Invalid token, chat ID, network errors.

---

### 3. Summary Table

| Node Name                     | Node Type                                   | Functional Role                                       | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                                         |
|-------------------------------|---------------------------------------------|------------------------------------------------------|------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | scheduleTrigger                             | Periodic trigger to start workflow                    | None                         | Get row(s) in sheet             | - Triggers the automation at regular intervals.                                                                     |
| Get row(s) in sheet           | googleSheets                                | Reads YouTube channel URLs from Google Sheets        | Schedule Trigger             | HTTP Request                   | - Reads YouTube channel URLs from a Google Sheets document.                                                         |
| HTTP Request                  | httpRequest                                | Fetches HTML content of YouTube channel pages         | Get row(s) in sheet          | HTML                          | - Sends HTTP GET request to the YouTube channel URL from Google Sheets.                                              |
| HTML                         | html                                        | Extracts canonical channel URL (to find channel ID)  | HTTP Request                 | Code                          | - Extracts channel links from YouTube pages to obtain the channel ID.                                                |
| Code                         | code                                        | Parses channel URL to get channel ID and RSS feed URL| HTML                         | RSS Read                      | - Extracts channel ID and constructs a YouTube RSS feed URL.                                                         |
| RSS Read                     | rssFeedRead                                | Reads latest video metadata from channel RSS feed    | Code                         | Append or update row in sheet   | - Reads the RSS feed from YouTube channel using constructed RSS URL.                                                 |
| Append or update row in sheet | googleSheets                                | Stores or updates video metadata in Google Sheets    | RSS Read                     | Get row(s) in sheet2            | - Saves video metadata to Google Sheets.                                                                             |
| Get row(s) in sheet2          | googleSheets                                | Reads updated sheet to check video summary presence  | Append or update row in sheet | If                            | - Reads back video data from updated sheet to check if summary exists.                                               |
| If                           | if                                           | Checks if video summary is empty to avoid duplicates | Get row(s) in sheet2          | HTTP Request1 / No Operation    | - Conditional check to prevent duplicate processing.                                                                 |
| No Operation, do nothing1     | noOp                                        | Skips processing for videos with existing summaries  | If (false branch)             | None                          |                                                                                                                     |
| HTTP Request1                | httpRequest                                | Calls Supadata API to get video transcript           | If (true branch)              | Merge Transcripts             | - Automatically retrieves transcripts from YouTube videos using API.                                                 |
| Merge Transcripts             | code                                        | Merges transcript parts into a single text            | HTTP Request1                | Set Language                  | - Combines transcript sections into one complete text.                                                               |
| Set Language                 | set                                         | Sets language context for AI summarization            | Merge Transcripts            | Basic LLM Chain               | - Sets output language used by LLM.                                                                                  |
| Basic LLM Chain              | chainLlm (LangChain)                        | Generates video summary from transcript                | Set Language                 | Append or update row in sheet1  | - Prompts AI to generate a summary from the transcript.                                                              |
| Append or update row in sheet1| googleSheets                                | Saves generated summary in Google Sheets               | Basic LLM Chain              | Basic LLM Chain1, 2, 3, 4       |                                                                                                                     |
| Basic LLM Chain1             | chainLlm (LangChain)                        | Creates LinkedIn post from summary                      | Append or update row in sheet1| Append or update row in sheet2  | - Generates LinkedIn content personalized per platform.                                                              |
| OpenRouter Chat Model1       | lmChatOpenRouter                           | AI language model for LinkedIn post                    | Basic LLM Chain1             | Basic LLM Chain1 output        |                                                                                                                     |
| Append or update row in sheet2| googleSheets                                | Saves LinkedIn post in Google Sheets                    | Basic LLM Chain1             | Send a text message            |                                                                                                                     |
| Basic LLM Chain2             | chainLlm (LangChain)                        | Creates X (Twitter) post from summary                   | Append or update row in sheet1| Append or update row in sheet3  | - Generates X (Twitter) content personalized per platform.                                                           |
| OpenRouter Chat Model2       | lmChatOpenRouter                           | AI language model for X post                            | Basic LLM Chain2             | Basic LLM Chain2 output        |                                                                                                                     |
| Append or update row in sheet3| googleSheets                                | Saves X post in Google Sheets                            | Basic LLM Chain2             | Send a text message            |                                                                                                                     |
| Basic LLM Chain3             | chainLlm (LangChain)                        | Creates Threads post from summary                        | Append or update row in sheet1| Append or update row in sheet5  | - Generates Threads content personalized per platform.                                                               |
| OpenRouter Chat Model3       | lmChatOpenRouter                           | AI language model for Threads post                      | Basic LLM Chain3             | Basic LLM Chain3 output        |                                                                                                                     |
| Append or update row in sheet5| googleSheets                                | Saves Threads post in Google Sheets                      | Basic LLM Chain3             | Send a text message            |                                                                                                                     |
| Basic LLM Chain4             | chainLlm (LangChain)                        | Creates Instagram caption from summary                   | Append or update row in sheet1| Append or update row in sheet4  | - Generates Instagram content personalized per platform.                                                             |
| OpenRouter Chat Model4       | lmChatOpenRouter                           | AI language model for Instagram caption                 | Basic LLM Chain4             | Basic LLM Chain4 output        |                                                                                                                     |
| Append or update row in sheet4| googleSheets                                | Saves Instagram caption in Google Sheets                 | Basic LLM Chain4             | Send a text message            |                                                                                                                     |
| Send a text message           | telegram                                    | Sends generated content previews via Telegram           | Append or update row in sheet2, sheet3, sheet4, sheet5 | None | - Sends data to Telegram for monitoring and faster results.                                                          |
| Sticky Notes (multiple)       | stickyNote                                  | Documentation and explanation for workflow sections     | None                        | None                          | See individual notes for detailed block explanations and references.                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to trigger at desired interval (e.g., every hour).  
   - No credentials required.

2. **Add Google Sheets node ("Get row(s) in sheet"):**  
   - Operation: Read rows.  
   - Sheet name: YOUR_SHEET_NAME  
   - Document ID: YOUR_GOOGLE_SHEET_ID  
   - Credentials: Google Sheets OAuth2.  
   - Connect Schedule Trigger output to this node.

3. **Add HTTP Request node:**  
   - Method: GET  
   - URL: Set to expression `={{ $json["channel url"] }}` from Google Sheets data.  
   - Connect output of Google Sheets node here.

4. **Add HTML node:**  
   - Operation: Extract HTML content.  
   - Extraction Values: Extract `href` attribute from `link[rel="canonical"]` CSS selector.  
   - Connect HTTP Request node output here.

5. **Add Code node:**  
   - Paste JavaScript to parse extracted URL, isolate channel ID, and construct YouTube RSS feed URL:  
     ```javascript
     return items.map(item => {
       const fullUrl = item.json.channel_id;
       const parts = fullUrl.split('/');
       const channelId = parts[parts.length - 1];
       return {
         json: {
           channel_id: channelId,
           rss_url: `https://www.youtube.com/feeds/videos.xml?channel_id=${channelId}`,
         }
       };
     });
     ```  
   - Connect HTML node output here.

6. **Add RSS Read node:**  
   - URL: Set to `={{ $json.rss_url }}`  
   - Connect Code node output here.

7. **Add Google Sheets node ("Append or update row in sheet"):**  
   - Operation: Append or Update row.  
   - Mapping columns: Map video fields (`id`, `link`, `title`, `author`, `pubDate`).  
   - Matching column: `id`.  
   - Sheet name and Document ID same as above.  
   - Credentials: Google Sheets OAuth2.  
   - Connect RSS Read output here.

8. **Add Google Sheets node ("Get row(s) in sheet2"):**  
   - Operation: Read rows from the same sheet.  
   - Connect Append/Update node output here.

9. **Add If node:**  
   - Condition: Check if `summary` field is empty (`{{ $json.summary === "" }}`).  
   - Connect "Get row(s) in sheet2" output here.

10. **Add No Operation node:**  
    - Connect to If node's False output (skip processing).

11. **Add HTTP Request node (Supadata API):**  
    - Method: GET  
    - URL: `https://api.supadata.ai/v1/transcript?url={{ $json.link }}`  
    - Headers: `x-api-key` with your Supadata API key.  
    - Connect If node True output here.

12. **Add Code node ("Merge Transcripts"):**  
    - JavaScript to concatenate transcript parts:  
      ```javascript
      const transcriptParts = items[0].json.content;
      const transcript = transcriptParts
        .map(part => part.text)
        .filter(Boolean)
        .join(' ');
      return [{ json: { full_transcript: transcript } }];
      ```  
    - Connect HTTP Request node output here.

13. **Add Set node ("Set Language"):**  
    - Assign a new field `language` with value `English`.  
    - Connect Merge Transcripts output here.

14. **Add Basic LLM Chain node (Summary):**  
    - Prompt: Summarize the transcript focusing on main points, speaker, insights, in the specified language.  
    - Input text: `={{ $('Merge Transcripts').item.json.full_transcript }}`  
    - Connect Set Language output here.

15. **Add Google Sheets node ("Append or update row in sheet1"):**  
    - Operation: Append or update row.  
    - Maps `id` and `summary` fields.  
    - Connect Basic LLM Chain output here.

16. **Add 4 parallel Basic LLM Chain nodes for LinkedIn, X, Threads, Instagram:**  
    - Each receives the summary text as input.  
    - Prompts tailored per platform:  
      - LinkedIn: Storytelling, ≤1300 characters, professional tone  
      - X: Punchy, ≤280 characters, conversational  
      - Threads: Casual, friendly tone, short paragraphs  
      - Instagram: Short captions, ≤150 characters per phrase  
    - Connect output of Append or update row in sheet1 to all four.

17. **Add 4 OpenRouter Chat Model nodes:**  
    - Each linked as AI model for respective Basic LLM Chain nodes.  
    - Model: `google/gemini-2.0-flash-exp:free`  
    - Connect each Basic LLM Chain node to corresponding OpenRouter node.

18. **Add 4 Google Sheets nodes to save generated posts for each platform:**  
    - Append or update row operation.  
    - Columns mapped: `linkedin`, `x`, `threads`, `instagram` accordingly.  
    - Connect outputs of Basic LLM Chain nodes here.

19. **Add Telegram node ("Send a text message"):**  
    - Text message composed using expressions to include video title, author, and generated posts.  
    - Chat ID: YOUR_TELEGRAM_CHAT_ID.  
    - Connect outputs of Google Sheets nodes that save posts to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow automates multi-platform content creation from YouTube videos using AI-powered transcript extraction and summarization. It supports LinkedIn, X, Threads, and Instagram content formats.                                                                                                  | Workflow description and sticky notes.                                                                          |
| Centralizes all video metadata, summaries, and posts in a single Google Sheets document for easy tracking and collaboration.                                                                                                                                                                         | [Google Sheets Template](https://docs.google.com/spreadsheets/d/17OjwIwx7eAwbkT5wtwvpCQU4rjrLH0v7j3fmC2Sc1CY)   |
| Requires credentials for Google Sheets OAuth2, Supadata API (for transcripts), OpenRouter API (for AI content generation), and Telegram Bot (for notifications).                                                                                                                                       | Credentials section in workflow description.                                                                     |
| Prompts are carefully designed to respect platform-specific tone, length, and style requirements.                                                                                                                                                                                                     | Prompt examples in Basic LLM Chain nodes.                                                                        |
| Telegram notifications provide fast previews to monitor content generation progress.                                                                                                                                                                                                                  | Sticky Note7 and Telegram node details.                                                                          |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.