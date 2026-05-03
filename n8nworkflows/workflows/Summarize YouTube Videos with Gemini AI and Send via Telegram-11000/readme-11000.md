Summarize YouTube Videos with Gemini AI and Send via Telegram

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-with-gemini-ai-and-send-via-telegram-11000


# Summarize YouTube Videos with Gemini AI and Send via Telegram

### 1. Workflow Overview

This workflow automates the process of summarizing YouTube videos by extracting their transcripts and metadata, generating AI-based summaries, and delivering the results through Telegram messages. It is designed to serve users who want quick, concise summaries of YouTube content without watching full videos, making it ideal for information digesting and content review.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Receives new messages from Telegram, filters for valid YouTube URLs, and handles invalid inputs.
- **1.2 Video Identification and Configuration:** Extracts the YouTube video ID and sets language preferences for transcript retrieval.
- **1.3 Data Extraction from Decodo API:** Requests video transcript and metadata via Decodo API using the video ID.
- **1.4 Data Parsing and Preparation:** Processes raw transcript and metadata into structured formats, including formatting durations, counts, and chapters.
- **1.5 AI Summarization:** Sends the transcript to Google’s Gemini AI model to generate a detailed TL;DR summary.
- **1.6 Data Merging and Formatting:** Combines metadata and AI summary, formats the summary into HTML chunks suitable for Telegram.
- **1.7 Sending Messages on Telegram:** Sends video info and the segmented summary messages to the user, with rate limiting to avoid flooding.
- **1.8 Error Handling and Admin Alerting:** Captures workflow errors, formats error logs, and alerts an admin via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block listens for new Telegram messages and verifies if the message contains a valid YouTube URL. It provides immediate feedback to the user if the URL is invalid.

**Nodes Involved:**  
- On New Message  
- Filter: YouTube URL?  
- Send "Processing..."  
- Send "Invalid URL"  

**Node Details:**

- **On New Message**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point for workflow, triggers on incoming Telegram messages.  
  - *Configuration:* Listens for "message" update type, uses Telegram API credentials.  
  - *Input/Output:* Outputs message JSON data.  
  - *Failures:* Telegram API connectivity issues, webhook misconfiguration.

- **Filter: YouTube URL?**  
  - *Type:* If  
  - *Role:* Checks if the message text contains "youtube." or "youtu.be" substrings.  
  - *Configuration:* String contains condition, case sensitive.  
  - *Input:* JSON from Telegram message.  
  - *Output:* Routes to "Send Processing..." if true, else "Send Invalid URL."  
  - *Failures:* Expression evaluation errors if `message.text` is missing.

- **Send "Processing..."**  
  - *Type:* Telegram  
  - *Role:* Sends an HTML formatted message indicating processing has started.  
  - *Configuration:* Sets chat ID dynamically from incoming message, uses Telegram bot credentials.  
  - *Input:* From Filter node.  
  - *Output:* Proceeds to set video ID and config.  
  - *Failures:* Telegram API errors, invalid chat ID.

- **Send "Invalid URL"**  
  - *Type:* Telegram  
  - *Role:* Sends a plain text message notifying user the URL is invalid.  
  - *Configuration:* Similar chat ID setup as above.  
  - *Input:* From Filter node if validation fails.  
  - *Failures:* Same as above.

---

#### 2.2 Video Identification and Configuration

**Overview:**  
Extracts the video ID from the YouTube URL and sets the default language code for transcript scraping.

**Nodes Involved:**  
- Set: Video ID & Config  

**Node Details:**

- **Set: Video ID & Config**  
  - *Type:* Set  
  - *Role:* Parses the video ID from the Telegram message text using regex, sets the language code to English by default.  
  - *Configuration:* Uses JavaScript regex expression to extract video ID from shortened YouTube URLs (`youtu.be/...`).  
  - *Input:* Telegram message JSON.  
  - *Output:* JSON with `videoId` and `languageCode` fields to be used in API requests.  
  - *Failures:* If message text format changes or regex fails to match, videoId may be undefined, potentially causing downstream errors.

---

#### 2.3 Data Extraction from Decodo API

**Overview:**  
Uses Decodo’s scraping API to retrieve the full transcript and metadata of the identified YouTube video.

**Nodes Involved:**  
- Decodo Youtube Transcript Scrapper  
- Decodo Youtube Metadata Scrapper  

**Node Details:**

- **Decodo Youtube Transcript Scrapper**  
  - *Type:* HTTP Request  
  - *Role:* POST request to Decodo API endpoint to fetch video transcript.  
  - *Configuration:*  
    - URL: `https://scraper-api.decodo.com/v2/scrape`  
    - Body parameters specify target: `youtube_transcript`, query: videoId, language_code.  
    - Authentication via HTTP Header with Decodo API key.  
  - *Input:* JSON with `videoId` and `languageCode`.  
  - *Output:* Raw transcript data JSON.  
  - *Failures:* HTTP errors, invalid API key, rate limits, malformed response.

- **Decodo Youtube Metadata Scrapper**  
  - *Type:* HTTP Request  
  - *Role:* POST request to Decodo API endpoint to fetch video metadata.  
  - *Configuration:*  
    - URL: Same as transcript scrapper.  
    - Body parameters target: `youtube_metadata`, query: videoId.  
    - Authentication with Decodo API key.  
  - *Input:* JSON with `videoId`.  
  - *Output:* Raw metadata JSON.  
  - *Failures:* Same as above.

---

#### 2.4 Data Parsing and Preparation

**Overview:**  
Processes raw API outputs to extract clean and structured transcript text, metadata fields, and formats them for further use.

**Nodes Involved:**  
- Extract Transcript  
- Parse Metadata  

**Node Details:**

- **Extract Transcript**  
  - *Type:* Code  
  - *Role:* Extracts transcript text segments from Decodo’s nested JSON, concatenates into a single string and array of texts.  
  - *Configuration:* JavaScript code loops through transcript segments, collects text snippets.  
  - *Input:* Raw transcript JSON from Decodo.  
  - *Output:* JSON with `transcript` (string), `textArray` (array), `totalSegments` (count).  
  - *Failures:* Unexpected JSON structure, missing fields.

- **Parse Metadata**  
  - *Type:* Code  
  - *Role:* Extracts video metadata like title, channel, duration, views, likes, chapters; formats durations and numbers; prepares a summary object for AI context.  
  - *Configuration:* JavaScript with helper functions for formatting duration, numbers, dates; filters out ads in chapters.  
  - *Input:* Raw metadata JSON from Decodo.  
  - *Output:* Structured JSON with multiple fields including formatted and raw stats, chapter info, and a nested summary object for AI use.  
  - *Failures:* Missing or malformed metadata fields; formatting errors.

---

#### 2.5 AI Summarization

**Overview:**  
Feeds the extracted transcript text into Google’s Gemini AI model to generate a structured, comprehensive TL;DR summary.

**Nodes Involved:**  
- Generate TLDR  
- Gemini Flash  

**Node Details:**

- **Gemini Flash**  
  - *Type:* Google Gemini Chat LLM  
  - *Role:* Provides conversational AI capabilities; acts as underlying model for summarization chain.  
  - *Configuration:* Uses Google Palm API credentials.  
  - *Input:* Not directly connected to data; chained to Generate TLDR.  
  - *Output:* AI-generated responses.  
  - *Failures:* API quota limits, network issues, auth failures.

- **Generate TLDR**  
  - *Type:* LangChain Chain LLM  
  - *Role:* Defines prompt instructing AI to create a multi-part summary (one-line, key points, topics, tools, audience, takeaways, quotes).  
  - *Configuration:* Uses the transcript text as input, with detailed prompt for content structure.  
  - *Input:* Transcript text from Extract Transcript node.  
  - *Output:* AI summary text.  
  - *Failures:* AI model errors, prompt execution failures.

---

#### 2.6 Data Merging and Formatting

**Overview:**  
Combines the parsed metadata and AI summary into one dataset, then splits and formats the summary into Telegram-compatible HTML message chunks.

**Nodes Involved:**  
- Merge: Data + Summary  
- Send Video Info  
- Split & Format HTML  

**Node Details:**

- **Merge: Data + Summary**  
  - *Type:* Merge  
  - *Role:* Combines outputs from Parse Metadata and Generate TLDR into a single JSON object.  
  - *Configuration:* Mode set to "combine all."  
  - *Input:* Two inputs: metadata and summary.  
  - *Output:* Unified JSON with all relevant data.  
  - *Failures:* Data incompatibility, missing inputs.

- **Send Video Info**  
  - *Type:* Telegram  
  - *Role:* Sends a formatted Markdown message summarizing video info (title, channel, duration, stats, chapters) to user.  
  - *Configuration:* Uses chat ID from Telegram message; markdown parse mode; disables attribution.  
  - *Input:* Merged data JSON.  
  - *Output:* Telegram message sent.  
  - *Failures:* Telegram API errors.

- **Split & Format HTML**  
  - *Type:* Code  
  - *Role:*  
    - Splits long AI summary text into chunks ≤4000 characters for Telegram.  
    - Converts Markdown-style formatting to HTML tags.  
    - Adds headers/footers to messages with video info and links.  
  - *Configuration:* Complex JavaScript handling text escaping, markdown conversion, chunk splitting by paragraphs and lines, and formatting for Telegram HTML.  
  - *Input:* Merged JSON with AI summary and metadata.  
  - *Output:* Array of message objects ready for sending.  
  - *Failures:* Edge cases in text parsing, unexpected characters breaking HTML, chunk size miscalculation.

---

#### 2.7 Sending Messages on Telegram

**Overview:**  
Sends the segmented summary messages to the user sequentially, implementing rate limiting to avoid spamming.

**Nodes Involved:**  
- Loop: Message Batches  
- Send Summary Part  
- Wait: Rate Limiter  

**Node Details:**

- **Loop: Message Batches**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over the array of message chunks produced by Split & Format HTML, processing one at a time.  
  - *Configuration:* Default batch size 1 (processes single messages sequentially).  
  - *Input:* Array of message JSON objects.  
  - *Output:* Single message JSON per iteration.  
  - *Failures:* Large batch arrays might introduce memory issues.

- **Send Summary Part**  
  - *Type:* Telegram  
  - *Role:* Sends one chunk of the summary as an HTML message to the Telegram chat.  
  - *Configuration:* Uses chat ID dynamically; parse mode HTML; disables attribution.  
  - *Input:* Single message JSON per batch iteration.  
  - *Output:* Telegram message sent.  
  - *Failures:* Telegram API errors, message size limits.

- **Wait: Rate Limiter**  
  - *Type:* Wait  
  - *Role:* Inserts a 1-second delay between sending messages to comply with Telegram rate limits.  
  - *Configuration:* Fixed wait time 1 second.  
  - *Input:* After sending each message.  
  - *Output:* Delayed continuation to next batch.  
  - *Failures:* None expected, but long queues could delay processing.

---

#### 2.8 Error Handling and Admin Alerting

**Overview:**  
Triggers on any workflow error, formats a concise and informative message, then sends an alert to an admin Telegram chat.

**Nodes Involved:**  
- On Error  
- Format Error Log  
- Alert Admin  

**Node Details:**

- **On Error**  
  - *Type:* Error Trigger  
  - *Role:* Captures any error in the workflow execution.  
  - *Input:* Error context JSON.  
  - *Output:* Passes error data to Format Error Log.  
  - *Failures:* None, but if misconfigured, errors may go unnoticed.

- **Format Error Log**  
  - *Type:* Code  
  - *Role:* Extracts workflow name, node name, error message, HTTP codes, stack trace, execution time, and formats a compact alert message in HTML.  
  - *Input:* Error JSON from On Error.  
  - *Output:* JSON containing a formatted error message string.  
  - *Failures:* Unexpected error object structures.

- **Alert Admin**  
  - *Type:* Telegram  
  - *Role:* Sends the formatted error alert to a predefined admin chat.  
  - *Configuration:* Requires replacement of `YOUR_CHAT_ID` with actual admin chat ID. Uses Telegram API credentials.  
  - *Input:* Formatted error message JSON.  
  - *Output:* Telegram message sent to admin.  
  - *Failures:* Telegram API errors, invalid admin chat ID.

---

### 3. Summary Table

| Node Name                     | Node Type                     | Functional Role                         | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                  |
|-------------------------------|-------------------------------|---------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| On New Message                 | Telegram Trigger              | Entry point - receives Telegram messages |                               | Filter: YouTube URL?           |                                                                                                              |
| Filter: YouTube URL?           | If                            | Validates if message contains YouTube URL | On New Message                | Send "Processing...", Send "Invalid URL" | ## Input Validation                                                                                           |
| Send "Processing..."           | Telegram                      | Sends "Processing..." feedback to user | Filter: YouTube URL?          | Set: Video ID & Config          |                                                                                                              |
| Send "Invalid URL"             | Telegram                      | Sends invalid URL notification        | Filter: YouTube URL?          |                               |                                                                                                              |
| Set: Video ID & Config         | Set                           | Extracts video ID and sets language code | Send "Processing..."          | Decodo Youtube Transcript Scrapper, Decodo Youtube Metadata Scrapper |                                                                                                              |
| Decodo Youtube Transcript Scrapper | HTTP Request                 | Fetches transcript data from Decodo API | Set: Video ID & Config        | Extract Transcript             | ## Create Decodo Credentials                                                                                  |
| Extract Transcript             | Code                          | Extracts and concatenates transcript text | Decodo Youtube Transcript Scrapper | Generate TLDR               | ## Extract and Process Video                                                                                   |
| Decodo Youtube Metadata Scrapper | HTTP Request                 | Fetches video metadata from Decodo API | Set: Video ID & Config        | Parse Metadata                | ## Create Decodo Credentials                                                                                  |
| Parse Metadata                | Code                          | Parses and formats video metadata      | Decodo Youtube Metadata Scrapper | Merge: Data + Summary         | ## Extract and Process Video                                                                                   |
| Gemini Flash                  | Google Gemini Chat LLM         | Provides AI model for summarization    |                               | Generate TLDR                 |                                                                                                              |
| Generate TLDR                 | LangChain Chain LLM            | Generates detailed video summary       | Extract Transcript, Gemini Flash | Merge: Data + Summary         |                                                                                                              |
| Merge: Data + Summary         | Merge                         | Combines metadata and AI summary       | Parse Metadata, Generate TLDR | Send Video Info, Split & Format HTML |                                                                                                              |
| Send Video Info               | Telegram                      | Sends video info summary to user       | Merge: Data + Summary         |                               | ## Send Result to Telegram                                                                                     |
| Split & Format HTML           | Code                          | Splits and converts summary into HTML chunks | Merge: Data + Summary         | Loop: Message Batches         | ## Send Result to Telegram                                                                                     |
| Loop: Message Batches         | Split In Batches              | Iterates over message chunks            | Split & Format HTML           | Send Summary Part             | ## Send Result to Telegram                                                                                     |
| Send Summary Part             | Telegram                      | Sends one chunk of the summary          | Loop: Message Batches         | Wait: Rate Limiter            | ## Send Result to Telegram                                                                                     |
| Wait: Rate Limiter            | Wait                          | Waits 1 second between sending messages | Send Summary Part             | Loop: Message Batches         | ## Send Result to Telegram                                                                                     |
| On Error                     | Error Trigger                 | Captures workflow errors                |                               | Format Error Log              | ## Error Handling                                                                                              |
| Format Error Log             | Code                          | Formats error details for alert         | On Error                     | Alert Admin                  | ## Error Handling                                                                                              |
| Alert Admin                 | Telegram                      | Sends error alert to admin               | Format Error Log             |                               | ## Error Handling                                                                                              |
| Sticky Note                  | Sticky Note                   | Documentation overview                   |                               |                               | # YouTube Summary Generator via Telegram... [Full note content about workflow purpose and setup]             |
| Sticky Note3                 | Sticky Note                   | Documentation note on error handling    |                               |                               | ## Error Handling                                                                                              |
| Sticky Note4                 | Sticky Note                   | Documentation note on extraction & processing |                               |                               | ## Extract and Process Video                                                                                   |
| Sticky Note5                 | Sticky Note                   | Documentation note on sending results   |                               |                               | ## Send Result to Telegram                                                                                     |
| Sticky Note6                 | Sticky Note                   | Documentation note on input validation  |                               |                               | ## Input Validation                                                                                            |
| Sticky Note9                 | Sticky Note                   | Documentation note on Decodo credential setup |                               |                               | ## Create Decodo Credentials... [Includes API key & link]                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("On New Message")**  
   - Type: Telegram Trigger  
   - Configure: Use your Telegram bot API credentials. Trigger on "message" update type.

2. **Add an If Node ("Filter: YouTube URL?")**  
   - Type: If  
   - Condition: Check if incoming message text contains "youtube." OR "youtu.be" (case sensitive).  
   - Connect "On New Message" output to this node.

3. **Add Telegram Nodes for User Feedback**  
   - "Send \"Processing...\"": Sends HTML message "<i>Processing your input...</i>"  
     - Chat ID: Set dynamically from incoming message.  
     - Connect If node's "true" output to this.  
   - "Send \"Invalid URL\"": Sends plain text "not a youtube url"  
     - Chat ID: Same dynamic setup.  
     - Connect If node's "false" output here.

4. **Add Set Node ("Set: Video ID & Config")**  
   - Type: Set  
   - Assign `videoId` by extracting from message text with regex: `$('On New Message').item.json.message.text.match(/youtu\.be\/([^?]+)/)[1]`  
   - Assign `languageCode` as `"en"` (default English)  
   - Connect "Send Processing..." node to this node.

5. **Add HTTP Request Node ("Decodo Youtube Transcript Scrapper")**  
   - Method: POST  
   - URL: `https://scraper-api.decodo.com/v2/scrape`  
   - Body Parameters: target=`youtube_transcript`, query=`={{ $json.videoId }}`, language_code=`={{ $json.languageCode }}`  
   - Authentication: HTTP Header Auth with header "Authorization" set to your Decodo API key (Basic Auth)  
   - Connect "Set: Video ID & Config" to this node.

6. **Add HTTP Request Node ("Decodo Youtube Metadata Scrapper")**  
   - Method: POST  
   - URL: Same as transcript scrapper  
   - Body Parameters: target=`youtube_metadata`, query=`={{ $json.videoId }}`  
   - Authentication: Same as above  
   - Connect "Set: Video ID & Config" to this node.

7. **Add Code Node ("Extract Transcript")**  
   - Input: Output from transcript scrapper  
   - Code: Extract transcript text segments and concatenate into one string.  
   - Connect "Decodo Youtube Transcript Scrapper" output to this node.

8. **Add Code Node ("Parse Metadata")**  
   - Input: Output from metadata scrapper  
   - Code: Parse video info, format durations, numbers, dates, extract chapters, and prepare summary context.  
   - Connect "Decodo Youtube Metadata Scrapper" output to this node.

9. **Add Google Gemini Chat LLM Node ("Gemini Flash")**  
   - Configure with your Google Palm API credentials.

10. **Add LangChain Chain LLM Node ("Generate TLDR")**  
    - Input: Transcript text from "Extract Transcript" node.  
    - Model: Connect to "Gemini Flash" node as underlying LLM.  
    - Prompt: Use the detailed prompt for multi-part summary as described.  
    - Connect "Extract Transcript" output to this node.

11. **Add Merge Node ("Merge: Data + Summary")**  
    - Mode: Combine all  
    - Inputs: From "Parse Metadata" and "Generate TLDR" nodes.  
    - Output: Combined JSON with metadata and summary.

12. **Add Telegram Node ("Send Video Info")**  
    - Text: Format with video title, channel, duration, stats, chapters (Markdown)  
    - Chat ID: From original Telegram message.  
    - Connect "Merge: Data + Summary" output to this node.

13. **Add Code Node ("Split & Format HTML")**  
    - Input: Combined JSON from Merge node  
    - Code: Split summary into ≤4000 char chunks, convert markdown to HTML, add message headers and footers with video info.  
    - Output: Array of message objects ready for sending.  
    - Connect "Merge: Data + Summary" output to this node.

14. **Add Split In Batches Node ("Loop: Message Batches")**  
    - Batch Size: 1 (default)  
    - Input: Array output from Split & Format HTML node.

15. **Add Telegram Node ("Send Summary Part")**  
    - Sends each chunk as an HTML message.  
    - Chat ID: Dynamic from original message.  
    - Connect "Loop: Message Batches" output to this node.

16. **Add Wait Node ("Wait: Rate Limiter")**  
    - Wait time: 1 second  
    - Connect "Send Summary Part" output to this node.  
    - Loop back "Wait" output to "Loop: Message Batches" to process next message.

17. **Configure Error Handling:**  
    - Add Error Trigger node ("On Error")  
    - Add Code node ("Format Error Log") to parse and format error details from On Error input.  
    - Add Telegram node ("Alert Admin") to send error message to your admin Telegram chat.  
    - Connect "On Error" → "Format Error Log" → "Alert Admin".

18. **Add Sticky Notes for Documentation:**  
    - Add notes with overview, block descriptions, credential setup instructions, and error handling notes as per your preference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow acts as your personal "TL;DV" (Too Long; Didn't View) assistant directly inside Telegram. Drop a YouTube link to the bot, and it instantly summarizes the video content using Decodo API and Google Gemini AI.                                          | Main workflow purpose (Sticky Note)                                                                              |
| Create Decodo credentials with HTTP Header Auth, Authorization header "Basic [YOUR_DECODO_API_KEY]". Get API key at https://dashboard.decodo.com/web-scraping-api/scraper?target=google_maps                                                                            | Decodo API credential setup (Sticky Note)                                                                        |
| Error Handling block catches workflow failures, formats error details, and sends Telegram alerts to admin. Replace `YOUR_CHAT_ID` in Alert Admin node with your Telegram user or group chat ID for error notifications.                                             | Error handling instructions (Sticky Note)                                                                        |
| Customize the `languageCode` in Set node to generate summaries in other languages (e.g., `id` for Indonesian). Modify AI prompt in Generate TLDR node to adjust summary style and content structure.                                                                | Customization tips                                                                                                |
| Telegram message size limit is 4096 characters; this workflow splits summaries into chunks ≤4000 characters with HTML formatting to ensure delivery.                                                                                                               | Message formatting constraint                                                                                    |
| Google Gemini Flash model requires valid Google Palm API credentials and quota; ensure your API limits are adequate for expected usage.                                                                                                                            | AI integration requirement                                                                                        |

---

This comprehensive documentation enables developers and AI agents to understand, reproduce, and customize the workflow fully, anticipating potential errors and integration details.