Personal Knowledge Assistant: Extract & Retrieve Summaries via Telegram & Google Sheets

https://n8nworkflows.xyz/workflows/personal-knowledge-assistant--extract---retrieve-summaries-via-telegram---google-sheets-7215


# Personal Knowledge Assistant: Extract & Retrieve Summaries via Telegram & Google Sheets

### 1. Workflow Overview

This workflow, titled **Personal Knowledge Assistant: Extract & Retrieve Summaries via Telegram & Google Sheets**, automates the extraction, summarization, storage, and retrieval of knowledge from YouTube videos and article links. It is designed for users who want to build a personal knowledge base by:

- Automatically extracting transcripts or article content from URLs added to a Google Sheet,
- Using Google Gemini AI models to summarize the content,
- Storing summarized data in a structured Google Sheet,
- Enabling querying of the stored summaries via Telegram chat.

The workflow is logically divided into three main functional blocks:

- **1.1 Input Reception and Query Handling:** Accepts user requests through Telegram and queries stored summaries in Google Sheets.
- **1.2 Data Ingestion and Processing:** Detects new URLs added to a Google Sheet, fetches content (YouTube transcripts or article data), and processes it.
- **1.3 Summarization and Storage:** Uses Google Gemini AI models to generate refined summaries and appends them back to Google Sheets for future querying.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Query Handling

**Overview:**  
This block manages incoming user queries via Telegram, retrieves relevant data from Google Sheets, and responds with the best-matched summary or an apology if no data is found.

**Nodes Involved:**  
- Telegram Trigger  
- AI Agent  
- Google Gemini Chat Model  
- Get row(s) in sheet in Google Sheets  
- Send a text message

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Trigger node for Telegram messages  
  - *Role:* Listens for incoming Telegram messages (updates of type "message").  
  - *Configuration:* Uses Telegram API credentials connected to a Telegram bot.  
  - *Inputs:* Incoming Telegram messages  
  - *Outputs:* Passes message JSON to AI Agent  
  - *Edge cases:* Telegram API connectivity issues, user sending unsupported message types.

- **AI Agent**  
  - *Type:* LangChain Agent for AI-driven conversation  
  - *Role:* Processes user message text to query stored summaries.  
  - *Configuration:* Prompts the AI to answer using Google Sheets data or apologize politely if no data found.  
  - *Key Expression:* `User Request: {{ $json.message.text }}`  
  - *Inputs:* Telegram message JSON  
  - *Outputs:* AI-generated response text  
  - *Dependencies:* Uses "Google Gemini Chat Model" as language model and "Get row(s) in sheet in Google Sheets" as tool to search data.  
  - *Edge cases:* AI model response errors, empty or ambiguous user queries, token limits.

- **Google Gemini Chat Model**  
  - *Type:* Google PaLM AI chat model  
  - *Role:* Provides AI language comprehension and generation for the AI Agent.  
  - *Configuration:* Uses Google Gemini API credentials.  
  - *Inputs:* Text prompts from AI Agent  
  - *Outputs:* AI-generated text responses  
  - *Edge cases:* API quota exceeded, authentication errors.

- **Get row(s) in sheet in Google Sheets**  
  - *Type:* Google Sheets node (read operation)  
  - *Role:* Searches for relevant rows matching user queries in the Google Sheet "YouTube Video and Article Data."  
  - *Configuration:* Reads from Sheet1 (gid=0) within the specified spreadsheet.  
  - *Inputs:* Query parameters from AI Agent  
  - *Outputs:* Rows matching query for AI Agent to process  
  - *Edge cases:* API authentication errors, empty sheet, query mismatch.

- **Send a text message**  
  - *Type:* Telegram node for sending messages  
  - *Role:* Sends AI Agent's response back to the Telegram chat.  
  - *Configuration:* Uses Telegram bot credentials; sends message text from AI Agent output; targets chat ID from Telegram Trigger.  
  - *Inputs:* AI Agent response text  
  - *Outputs:* Message sent confirmation  
  - *Edge cases:* Telegram API errors, rate limits.

---

#### 1.2 Data Ingestion and Processing

**Overview:**  
This block detects when a new URL is added to a designated Google Sheet, filters for new entries, and determines if the URL is a YouTube video or an article link to fetch the appropriate content.

**Nodes Involved:**  
- Google Sheets Trigger  
- Filter  
- If  
- HTTP Request (YouTube transcript fetch)  
- HTTP Request1 (Article fetch)

**Node Details:**

- **Google Sheets Trigger**  
  - *Type:* Trigger node for Google Sheets row additions  
  - *Role:* Monitors a specific sheet (Sheet2, gid=800289465) for new rows added containing URLs.  
  - *Configuration:* Polls every minute, uses Google Sheets OAuth credentials.  
  - *Inputs:* New rows added to Sheet2  
  - *Outputs:* New row JSON data forwarded to Filter.  
  - *Edge cases:* Polling delays, API rate limiting.

- **Filter**  
  - *Type:* Filter node  
  - *Role:* Ensures the incoming row has a URL and has not been marked as "Stored" (i.e., new, unprocessed).  
  - *Configuration:*  
    - Checks that "URL " field exists and is non-empty.  
    - Checks that "Stored" field is empty.  
  - *Inputs:* New row data from Google Sheets Trigger  
  - *Outputs:* Passes only new, unprocessed URLs to "If" node.  
  - *Edge cases:* Malformed or missing fields, empty rows.

- **If**  
  - *Type:* Conditional node  
  - *Role:* Determines if the URL corresponds to a YouTube video link or an article.  
  - *Configuration:* Checks if the URL contains "youtu.be" or "youtube.com".  
  - *Inputs:* Filtered row JSON with URL  
  - *Outputs:*  
    - True branch to YouTube transcript HTTP Request  
    - False branch to article HTTP Request  
  - *Edge cases:* URLs not matching either criteria, malformed URLs.

- **HTTP Request (YouTube transcript fetch)**  
  - *Type:* HTTP POST request node  
  - *Role:* Calls Apify Actor "YouTube Transcript Ninja" endpoint to retrieve transcript JSON for the YouTube URL.  
  - *Configuration:*  
    - POST with JSON body including language (English), no timestamps, and startUrls with the URL parameter.  
    - URL is parameterized with the new row's "URL ".  
  - *Inputs:* YouTube URL from If node  
  - *Outputs:* Transcript JSON to Information Extractor1.  
  - *Edge cases:* Apify API key missing or invalid, transcript not available, HTTP errors.

- **HTTP Request1 (Article fetch)**  
  - *Type:* HTTP GET request node  
  - *Role:* Fetches raw article HTML/data from the provided URL.  
  - *Configuration:* Direct GET request to the URL from the Google Sheet.  
  - *Inputs:* Article URL from If node  
  - *Outputs:* HTML content to Markdown node.  
  - *Edge cases:* HTTP errors, invalid URLs, timeouts, non-HTML responses.

---

#### 1.3 Summarization and Storage

**Overview:**  
This block converts raw content (YouTube transcripts or article HTML) into summaries using Google Gemini AI models, then stores the summaries and metadata back into Google Sheets to build a structured knowledge base.

**Nodes Involved:**  
- Information Extractor1 (YouTube transcript)  
- Google Gemini Chat Model2 (YouTube)  
- Append row in sheet (YouTube)  
- Append or update row in sheet (YouTube)  
- Markdown (Article HTML to text)  
- Information Extractor (Article)  
- Google Gemini Chat Model1 (Article)  
- Append row in sheet1 (Article)  
- Append or update row in sheet2 (Article)

**Node Details:**

- **Markdown**  
  - *Type:* Markdown node  
  - *Role:* Converts raw HTML article content into markdown/plain text for summarization.  
  - *Configuration:* Uses the field `data` from HTTP Request1 node as HTML input.  
  - *Inputs:* HTML content from HTTP Request1  
  - *Outputs:* Markdown/plain text to Information Extractor.  
  - *Edge cases:* Malformed HTML, empty content.

- **Information Extractor1 (YouTube transcript)**  
  - *Type:* LangChain Information Extractor  
  - *Role:* Extracts structured information (Title and refined summary) from YouTube transcript text.  
  - *Configuration:*  
    - Text template: `"YouTube Video Transcript: {{ $json.transcript }}"`  
    - Extracts two required attributes:  
      - Title (a title for the video)  
      - Article Refined Data (detailed summary, with instruction not to start with "this video")  
  - *Inputs:* Transcript JSON from HTTP Request  
  - *Outputs:* Structured extracted data to Append row in sheet.  
  - *AI Model:* Uses Google Gemini Chat Model2 as AI language model.  
  - *Edge cases:* Extraction failures, incomplete transcripts.

- **Google Gemini Chat Model2**  
  - *Type:* Google PaLM AI chat model  
  - *Role:* Provides AI summarization for YouTube transcripts.  
  - *Configuration:* Connected to Information Extractor1.  
  - *Edge cases:* API errors, quota limits.

- **Append row in sheet (YouTube)**  
  - *Type:* Google Sheets append operation  
  - *Role:* Appends extracted video title and summary to Sheet1 (main data sheet).  
  - *Configuration:* Columns: Title and Data (summary).  
  - *Inputs:* Output from Information Extractor1  
  - *Outputs:* Confirmation to Append or update row in sheet.  
  - *Edge cases:* API errors, permission issues.

- **Append or update row in sheet (YouTube)**  
  - *Type:* Google Sheets append or update operation  
  - *Role:* Updates Sheet2 to mark the URL as "Stored" ✅ indicating processing complete.  
  - *Configuration:* Matches rows by "URL " column; sets "Stored" to "✅".  
  - *Inputs:* URL from Google Sheets Trigger node JSON.  
  - *Outputs:* Final confirmation.  
  - *Edge cases:* Concurrency issues, partial updates.

- **Information Extractor (Article)**  
  - *Type:* LangChain Information Extractor  
  - *Role:* Extracts structured information (Title and refined summary) from article markdown text.  
  - *Configuration:*  
    - Text template: `"Article: {{ $json.data }}"`  
    - Extracts two attributes similar to YouTube extractor: Title and Article Refined Data.  
  - *Inputs:* Markdown text from Markdown node  
  - *Outputs:* Structured extracted data to Append row in sheet1.  
  - *AI Model:* Uses Google Gemini Chat Model1 as AI language model.  
  - *Edge cases:* Extraction quality affected by markdown conversion.

- **Google Gemini Chat Model1**  
  - *Type:* Google PaLM AI chat model  
  - *Role:* Summarizes article content for Information Extractor.  
  - *Configuration:* Connected to Information Extractor.  
  - *Edge cases:* API limits.

- **Append row in sheet1 (Article)**  
  - *Type:* Google Sheets append operation  
  - *Role:* Appends article title and summary to Sheet1.  
  - *Inputs:* Extractor output  
  - *Outputs:* Confirmation to Append or update row in sheet2.  
  - *Edge cases:* Same as Append row in sheet (YouTube).

- **Append or update row in sheet2 (Article)**  
  - *Type:* Google Sheets append or update operation  
  - *Role:* Marks the article URL as "Stored" ✅ in Sheet2 to avoid reprocessing.  
  - *Inputs:* URL from Google Sheets Trigger JSON  
  - *Outputs:* Final confirmation.  
  - *Edge cases:* Same as YouTube append/update node.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                              | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                                                                                                                                                    |
|---------------------------|----------------------------------------|----------------------------------------------|-----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger          | telegramTrigger                        | Listen to Telegram messages                   | —                           | AI Agent                      |                                                                                                                                                                                                                                                |
| AI Agent                  | langchain.agent                       | Process user query, search Google Sheets     | Telegram Trigger, Google Gemini Chat Model, Get row(s) in sheet in Google Sheets | Send a text message           |                                                                                                                                                                                                                                                |
| Google Gemini Chat Model  | langchain.lmChatGoogleGemini          | AI language model for AI Agent                | AI Agent                    | AI Agent                      |                                                                                                                                                                                                                                                |
| Get row(s) in sheet in Google Sheets | googleSheetsTool                  | Search stored summaries in Google Sheets     | AI Agent                    | AI Agent                      |                                                                                                                                                                                                                                                |
| Send a text message       | telegram                              | Reply to Telegram user                        | AI Agent                    | —                             |                                                                                                                                                                                                                                                |
| Google Sheets Trigger     | googleSheetsTrigger                   | Trigger on new URLs added to Sheet2           | —                           | Filter                        |                                                                                                                                                                                                                                                |
| Filter                   | filter                               | Filter new URLs not yet processed             | Google Sheets Trigger       | If                           |                                                                                                                                                                                                                                                |
| If                       | if                                   | Detect if URL is YouTube or Article           | Filter                      | HTTP Request (YouTube), HTTP Request1 (Article) |                                                                                                                                                                                                                                                |
| HTTP Request             | httpRequest                         | Fetch YouTube transcript via Apify API        | If (YouTube branch)         | Information Extractor1         |                                                                                                                                                                                                                                                |
| Information Extractor1    | langchain.informationExtractor       | Extract title and summary from transcript     | HTTP Request                 | Append row in sheet            |                                                                                                                                                                                                                                                |
| Google Gemini Chat Model2 | langchain.lmChatGoogleGemini          | Summarize YouTube transcript                   | Information Extractor1      | Information Extractor1         |                                                                                                                                                                                                                                                |
| Append row in sheet       | googleSheets                        | Append YouTube summary to Sheet1               | Information Extractor1      | Append or update row in sheet  |                                                                                                                                                                                                                                                |
| Append or update row in sheet | googleSheets                        | Mark YouTube URL as stored in Sheet2          | Append row in sheet         | —                             |                                                                                                                                                                                                                                                |
| HTTP Request1            | httpRequest                         | Fetch article HTML content                      | If (Article branch)         | Markdown                      |                                                                                                                                                                                                                                                |
| Markdown                 | markdown                            | Convert HTML article to markdown               | HTTP Request1               | Information Extractor          |                                                                                                                                                                                                                                                |
| Information Extractor     | langchain.informationExtractor       | Extract title and summary from article         | Markdown                    | Append row in sheet1           |                                                                                                                                                                                                                                                |
| Google Gemini Chat Model1 | langchain.lmChatGoogleGemini          | Summarize article content                       | Information Extractor       | Information Extractor          |                                                                                                                                                                                                                                                |
| Append row in sheet1      | googleSheets                        | Append article summary to Sheet1                | Information Extractor       | Append or update row in sheet2 |                                                                                                                                                                                                                                                |
| Append or update row in sheet2 | googleSheets                        | Mark article URL as stored in Sheet2           | Append row in sheet1        | —                             |                                                                                                                                                                                                                                                |
| Sticky Note               | stickyNote                         | Note: Requesting Data                           | —                           | —                             | # Requesting Data                                                                                                                                                                                                                              |
| Sticky Note1              | stickyNote                         | Note: Storing Data through YouTube/Article URLs | —                           | —                             | # Storing Data through YouTube Video URL / Article Link                                                                                                                                                                                      |
| Sticky Note2              | stickyNote                         | Setup guide                                     | —                           | —                             | # 🛠 Setup Guide ...\n\nAuthor: [Rakin Jakaria](https://www.youtube.com/@rakinjakaria)\n\nLinks to Telegram Bot, Apify, Gemini, Google Sheets provided                                                                                         |
| Sticky Note3              | stickyNote                         | Purpose of Agent                                | —                           | —                             | # 1️⃣ Purpose of This Agent\n\nExtract summaries, store, and query via Telegram.                                                                                                                                                              |
| Sticky Note4              | stickyNote                         | How to Use                                      | —                           | —                             | # 2️⃣ How to Use\n\nAdd URLs to Sheet2, auto-process, query via Telegram.                                                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials.  
   - Set to listen for "message" updates.

2. **Create Google Gemini Chat Model node (for AI Agent)**  
   - Type: LangChain Google Gemini Chat Model  
   - Configure with your Google PaLM API credentials.

3. **Create Get row(s) in sheet in Google Sheets node**  
   - Type: Google Sheets Tool (read)  
   - Connect to your Google Sheets account with OAuth2.  
   - Configure to read from your knowledge base sheet (Sheet1, gid=0) in your specific spreadsheet ID.

4. **Create AI Agent node**  
   - Type: LangChain Agent  
   - Set input text to `User Request: {{ $json.message.text }}`  
   - System message: "when user asks something give the answer from the given google sheet and after searching the google sheet if you don't found then politely apologies to the user."  
   - Add the Google Gemini Chat Model as AI language model.  
   - Add the Google Sheets read node as a tool.  
   - Connect Telegram Trigger to AI Agent.

5. **Create Send a text message node**  
   - Type: Telegram  
   - Configure credentials same as Telegram Trigger.  
   - Set chatId to `{{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Set text to `{{ $json.output }}` from AI Agent.  
   - Connect AI Agent to this node.

6. **Create Google Sheets Trigger node**  
   - Type: Google Sheets Trigger  
   - Configure OAuth2 for Google Sheets.  
   - Set to watch for row additions on Sheet2 (gid=800289465) in your spreadsheet.

7. **Create Filter node**  
   - Type: Filter  
   - Conditions:  
     - "URL " field exists and is not empty  
     - "Stored" field is empty  
   - Connect Google Sheets Trigger to Filter.

8. **Create If node**  
   - Type: If  
   - Condition: Check if "URL " contains "youtu.be" OR "youtube.com"  
   - Connect Filter to If.

9. **Create HTTP Request node (YouTube transcript)**  
   - Type: HTTP Request (POST)  
   - URL: Apify Actor YouTube Transcript Ninja endpoint (replace with your Apify API URL)  
   - Body (JSON):  
     ```json
     {
       "includeTimestamps": "No",
       "language": "English",
       "startUrls": ["{{ $json['URL '] }}"]
     }
     ```  
   - Connect If true branch to this node.

10. **Create HTTP Request1 node (Article fetch)**  
    - Type: HTTP Request (GET)  
    - URL: `{{ $json['URL '] }}`  
    - Connect If false branch to this node.

11. **Create Markdown node**  
    - Type: Markdown  
    - Set HTML input to `{{ $json.data }}` from HTTP Request1 node.  
    - Connect HTTP Request1 to Markdown.

12. **Create Google Gemini Chat Model1 node (for Article summarization)**  
    - Configure with Google PaLM credentials.  
    - Connect as AI language model to Information Extractor.

13. **Create Information Extractor node (Article)**  
    - Type: LangChain Information Extractor  
    - Text: `Article: {{ $json.data }}`  
    - Attributes:  
      - Title (required, description: "a title for this article")  
      - Article Refined Data (required, description: "a detailed summary from the article to add that in my Supabase. Don't start with this article or this thing.")  
    - Connect Markdown node output to this node.  
    - Link Google Gemini Chat Model1 as its AI language model.

14. **Create Append row in sheet1 node**  
    - Type: Google Sheets Append  
    - Document ID and Sheet: Sheet1 (gid=0) of your spreadsheet  
    - Columns: Title and Data mapped from