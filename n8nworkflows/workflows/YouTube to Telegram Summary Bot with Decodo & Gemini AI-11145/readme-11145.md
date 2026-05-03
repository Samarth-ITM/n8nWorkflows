YouTube to Telegram Summary Bot with Decodo & Gemini AI

https://n8nworkflows.xyz/workflows/youtube-to-telegram-summary-bot-with-decodo---gemini-ai-11145


# YouTube to Telegram Summary Bot with Decodo & Gemini AI

### 1. Workflow Overview

This workflow is a YouTube-to-Telegram summary bot designed to transform long YouTube videos into concise, structured TL;DR summaries delivered within Telegram chats. It targets users who want quick insights from YouTube videos without watching full content. The bot listens for YouTube URLs sent via Telegram, validates and extracts the video ID, fetches transcript and metadata via the Decodo Scraper API, generates an AI-powered summary using Google’s Gemini AI model, and sends the formatted summary back to the user in manageable message chunks.

The workflow is composed of five logical blocks:

- **1.1 Input Reception and Validation:** Receives Telegram messages, filters for YouTube URLs, and handles invalid inputs.
- **1.2 Data Extraction via Decodo API:** Retrieves YouTube transcript and metadata using Decodo API HTTP requests.
- **1.3 Data Parsing and AI Summarization:** Parses raw API data, prepares context, and uses Gemini AI to generate a detailed summary.
- **1.4 Formatting and Telegram Delivery:** Formats the AI summary into HTML, splits into message batches, and sends them sequentially to the user.
- **1.5 Error Handling and Alerts:** Captures workflow errors, formats error details, and notifies an admin via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block listens for new Telegram messages, detects if the message contains a valid YouTube URL, and sends either a "Processing..." confirmation or an "Invalid URL" warning.

**Nodes Involved:**  
- On New Message  
- Filter: YouTube URL?  
- Send "Processing..."  
- Send "Invalid URL"  

**Node Details:**

- **On New Message**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point that triggers on incoming Telegram messages.  
  - *Config:* Listens specifically for "message" updates.  
  - *Connections:* Outputs to "Filter: YouTube URL?"  
  - *Failure Modes:* Telegram API downtime or webhook misconfiguration.  

- **Filter: YouTube URL?**  
  - *Type:* IF node  
  - *Role:* Checks if the message text contains "youtube." or "youtu.be".  
  - *Config:* Two OR conditions using string "contains" operation on message text.  
  - *Inputs:* From "On New Message"  
  - *Outputs:* True → "Send Processing...", False → "Send Invalid URL"  
  - *Edge Cases:* Case sensitivity could cause misses if URLs are uppercase; partial URLs might pass incorrectly.  

- **Send "Processing..."**  
  - *Type:* Telegram node  
  - *Role:* Sends an italicized "Processing your input..." message to user.  
  - *Config:* Uses HTML parse mode; chatId dynamically from original message.  
  - *Input:* True branch from filter node  
  - *Output:* To "Set: Video ID & Config"  
  - *Failure Modes:* Telegram API rate limits or chatId missing.  

- **Send "Invalid URL"**  
  - *Type:* Telegram node  
  - *Role:* Sends "not a youtube url" message to user when validation fails.  
  - *Config:* Plain text message; chatId from original message.  
  - *Input:* False branch from filter node  
  - *Output:* None (terminal for invalid input)  
  - *Failure Modes:* Same as above.  

---

#### 2.2 Data Extraction via Decodo API

**Overview:**  
Extracts the YouTube video ID from the URL, sets language code, then fetches both the transcript and metadata from Decodo's scraper API using authenticated HTTP POST requests.

**Nodes Involved:**  
- Set: Video ID & Config  
- Decodo Youtube Transcript Scrapper  
- Extract Transcript  
- Decodo Youtube Metadata Scrapper  
- Parse Metadata  

**Node Details:**

- **Set: Video ID & Config**  
  - *Type:* Set node  
  - *Role:* Extracts videoId from Telegram message text using regex, sets languageCode to "en" by default.  
  - *Config:* Uses expression `{{$json.message.text.match(/youtu\\.be\\/([^?]+)/)[1]}}` to parse videoId from short URLs.  
  - *Input:* From "Send Processing..."  
  - *Output:* To both Decodo scrapper nodes  
  - *Failure Modes:* Regex fails if URL format varies (e.g., full YouTube URLs, query parameters), no fallback implemented.  

- **Decodo Youtube Transcript Scrapper**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to Decodo API to scrape YouTube transcript for videoId and languageCode.  
  - *Config:* URL: https://scraper-api.decodo.com/v2/scrape, body with parameters: target="youtube_transcript", query=videoId, language_code=languageCode; uses HTTP header auth with Decodo API key.  
  - *Input:* From "Set: Video ID & Config"  
  - *Output:* To "Extract Transcript"  
  - *Failures:* API auth failure, network errors, transcript unavailable or empty.  

- **Extract Transcript**  
  - *Type:* Code node  
  - *Role:* Parses transcript JSON from Decodo response, concatenates text segments into single transcript string and counts total segments.  
  - *Config:* Custom JS looping through transcriptSegmentRenderer runs to accumulate texts.  
  - *Input:* From "Decodo Youtube Transcript Scrapper"  
  - *Output:* To "Generate TLDR"  
  - *Failures:* Unexpected JSON structure could cause runtime errors; no explicit error handling inside code.  

- **Decodo Youtube Metadata Scrapper**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to Decodo API to scrape YouTube video metadata for videoId.  
  - *Config:* URL: same as above, body with target="youtube_metadata", query=videoId; uses same credentials.  
  - *Input:* From "Set: Video ID & Config"  
  - *Output:* To "Parse Metadata"  
  - *Failures:* Same as transcript scrapper, plus possible missing metadata fields.  

- **Parse Metadata**  
  - *Type:* Code node  
  - *Role:* Processes Decodo metadata response; formats duration, dates, numbers; filters chapters; compiles a structured metadata object including video URL, thumbnails, and summary context for AI.  
  - *Config:* JS code with helper functions for formatting duration, numbers, dates; excludes "Ad" chapters; creates a summaryForAI object for downstream prompts.  
  - *Input:* From "Decodo Youtube Metadata Scrapper"  
  - *Output:* To "Merge: Data + Summary"  
  - *Failures:* If Decodo response format changes, parsing may fail. Some fields nullable or defaulted.  

---

#### 2.3 Data Parsing and AI Summarization

**Overview:**  
Combines transcript text and parsed metadata, sends transcript to Google Gemini AI for generating a detailed TL;DR summary, and merges the AI output with metadata.

**Nodes Involved:**  
- Generate TLDR  
- Gemini 3.0  
- Merge: Data + Summary  

**Node Details:**

- **Gemini 3.0**  
  - *Type:* Google Gemini Chat model node (Langchain)  
  - *Role:* Provides the AI language model interface.  
  - *Config:* Model set to "models/gemini-3-pro-preview", credentials use Google Palm API key.  
  - *Input:* From "Extract Transcript" (via "Generate TLDR")  
  - *Output:* To "Generate TLDR" (ai_languageModel input)  
  - *Failures:* API quota limits, auth errors, network issues, or model updates.  

- **Generate TLDR**  
  - *Type:* Langchain Chain LLM node  
  - *Role:* Sends the transcript text as prompt to Gemini AI with a detailed instruction to produce a structured summary: one-line summary, key points, topics, tools mentioned, target audience, practical takeaways, and optional quotes.  
  - *Config:* Uses the transcript as input text, prompt precisely defines output format for AI.  
  - *Input:* Transcript text from "Extract Transcript", AI model from "Gemini 3.0"  
  - *Output:* To "Merge: Data + Summary"  
  - *Failures:* Prompt might generate unexpected output; large transcripts might time out or cause token limits.  

- **Merge: Data + Summary**  
  - *Type:* Merge node  
  - *Role:* Combines the AI-generated summary and parsed metadata into a single JSON object for downstream processing.  
  - *Config:* Combine mode set to "combineAll".  
  - *Inputs:* One from "Generate TLDR", one from "Parse Metadata"  
  - *Output:* To "Send Video Info" and "Split & Format HTML"  
  - *Failures:* Mismatched input sizes or empty inputs could cause no output.  

---

#### 2.4 Formatting and Telegram Delivery

**Overview:**  
Formats the merged summary and metadata into HTML, splits long messages into chunks suitable for Telegram, then sends each chunk sequentially with a rate limiter to avoid API flooding.

**Nodes Involved:**  
- Send Video Info  
- Split & Format HTML  
- Loop: Message Batches  
- Send Summary Part  
- Wait: Rate Limiter  

**Node Details:**

- **Send Video Info**  
  - *Type:* Telegram node  
  - *Role:* Sends a first message with a summary of the video metadata and overall structure, formatted in Markdown.  
  - *Config:* ChatId from original message; message includes video title, channel, duration, category, tags, views, likes, chapters; Markdown parse mode.  
  - *Input:* From "Merge: Data + Summary"  
  - *Output:* To "Split & Format HTML"  
  - *Failures:* Markdown formatting errors, Telegram API errors.  

- **Split & Format HTML**  
  - *Type:* Code node  
  - *Role:* Converts the AI summary text from Markdown-like format to HTML, escapes special characters, splits the long text into chunks <4000 chars for Telegram, adds headers/footers with video info, and prepares messages for sending.  
  - *Config:* Complex JS code with helpers for escaping HTML, converting Markdown to HTML tags, formatting bullet points, and splitting intelligently by paragraphs and lines.  
  - *Input:* From "Merge: Data + Summary"  
  - *Output:* To "Loop: Message Batches"  
  - *Failures:* Edge cases with unexpected formatting, very long paragraphs, or special characters might cause formatting issues.  

- **Loop: Message Batches**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates through each chunk message to send sequentially.  
  - *Config:* Default batch size of 1; no additional options.  
  - *Input:* From "Split & Format HTML"  
  - *Output:* To "Send Summary Part" (main output) and empty output (end of batches)  
  - *Failures:* Large number of messages may cause long running workflows or rate limits.  

- **Send Summary Part**  
  - *Type:* Telegram node  
  - *Role:* Sends one chunk of the summary in HTML format to the user.  
  - *Config:* Text from batch item, chatId from original message, parse_mode = HTML, no attribution appended.  
  - *Input:* From "Loop: Message Batches"  
  - *Output:* To "Wait: Rate Limiter"  
  - *Failures:* Telegram API rate limits or message size limits.  

- **Wait: Rate Limiter**  
  - *Type:* Wait node  
  - *Role:* Pauses 1 second between sending messages to respect Telegram rate limits.  
  - *Config:* Wait time set to 1 second.  
  - *Input:* From "Send Summary Part"  
  - *Output:* Loops back to "Loop: Message Batches" to send next batch.  
  - *Failures:* Workflow could get stuck if errors occur during iteration.  

---

#### 2.5 Error Handling and Alerts

**Overview:**  
Triggers on any workflow error, formats a detailed error message, and sends an alert to a Telegram admin chat.

**Nodes Involved:**  
- On Error  
- Format Error Log  
- Alert Admin  

**Node Details:**

- **On Error**  
  - *Type:* Error Trigger  
  - *Role:* Catches any error from the workflow execution.  
  - *Config:* Default settings, triggers on any error.  
  - *Output:* To "Format Error Log"  
  - *Failures:* Should not fail; fallback for errors.  

- **Format Error Log**  
  - *Type:* Code node  
  - *Role:* Extracts error details from input, including workflow name, node name, messages, HTTP codes, stack trace (only first line), and execution timestamp; formats a compact HTML message.  
  - *Config:* JS code with helper function for safe nested value extraction and formatting.  
  - *Input:* From "On Error"  
  - *Output:* To "Alert Admin"  
  - *Failures:* If error data format changes or missing fields, defaults used.  

- **Alert Admin**  
  - *Type:* Telegram node  
  - *Role:* Sends the formatted error message to a specified admin Telegram chat ID.  
  - *Config:* ChatId hardcoded as "YOUR_CHAT_ID" (must be replaced by user), parse mode HTML.  
  - *Input:* From "Format Error Log"  
  - *Output:* None  
  - *Failures:* If chatId invalid or Telegram API down, alerts fail.  

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                          | Input Node(s)                 | Output Node(s)                         | Sticky Note                                                                                         |
|-------------------------------|---------------------------------|----------------------------------------|------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------|
| On New Message                 | Telegram Trigger                | Entry point: receives Telegram messages| -                            | Filter: YouTube URL?                   | ## Input Validation                                                                              |
| Filter: YouTube URL?           | IF                              | Validates presence of YouTube URL      | On New Message               | Send "Processing...", Send "Invalid URL" | ## Input Validation                                                                              |
| Send "Processing..."           | Telegram                        | Sends processing confirmation          | Filter: YouTube URL? (true)  | Set: Video ID & Config                 | ## Input Validation                                                                              |
| Send "Invalid URL"             | Telegram                        | Sends invalid URL warning               | Filter: YouTube URL? (false) | -                                     | ## Input Validation                                                                              |
| Set: Video ID & Config         | Set                             | Extracts videoId and sets language code| Send "Processing..."         | Decodo Youtube Transcript Scrapper, Decodo Youtube Metadata Scrapper | ## Extract and Process Video                                                                     |
| Decodo Youtube Transcript Scrapper | HTTP Request                  | Fetches YouTube transcript via Decodo API | Set: Video ID & Config       | Extract Transcript                    | ## Extract and Process Video                                                                     |
| Extract Transcript             | Code                            | Parses transcript JSON into text       | Decodo Youtube Transcript Scrapper | Generate TLDR                     | ## Extract and Process Video                                                                     |
| Decodo Youtube Metadata Scrapper | HTTP Request                  | Fetches YouTube metadata via Decodo API| Set: Video ID & Config       | Parse Metadata                       | ## Extract and Process Video                                                                     |
| Parse Metadata                | Code                            | Parses and formats metadata             | Decodo Youtube Metadata Scrapper | Merge: Data + Summary                | ## Extract and Process Video                                                                     |
| Gemini 3.0                    | Langchain Google Gemini AI       | AI model for text summarization         | - (connected via Generate TLDR) | Generate TLDR                       |                                                                                                   |
| Generate TLDR                 | Langchain Chain LLM              | Sends transcript to AI for TLDR summary| Extract Transcript, Gemini 3.0| Merge: Data + Summary                |                                                                                                   |
| Merge: Data + Summary         | Merge                           | Combines AI summary with metadata       | Generate TLDR, Parse Metadata | Send Video Info, Split & Format HTML |                                                                                                   |
| Send Video Info               | Telegram                        | Sends video metadata summary to user   | Merge: Data + Summary        | Split & Format HTML                  | ## Send Result to Telegram                                                                       |
| Split & Format HTML           | Code                            | Converts summary to HTML, splits messages | Merge: Data + Summary        | Loop: Message Batches                | ## Send Result to Telegram                                                                       |
| Loop: Message Batches         | SplitInBatches                  | Iterates over message chunks to send   | Split & Format HTML          | Send Summary Part                    | ## Send Result to Telegram                                                                       |
| Send Summary Part             | Telegram                        | Sends one chunk of summary text         | Loop: Message Batches        | Wait: Rate Limiter                   | ## Send Result to Telegram                                                                       |
| Wait: Rate Limiter            | Wait                            | Delays between sending messages         | Send Summary Part            | Loop: Message Batches                | ## Send Result to Telegram                                                                       |
| On Error                     | Error Trigger                   | Captures workflow errors                 | -                           | Format Error Log                    | ## Error Handling                                                                               |
| Format Error Log              | Code                            | Formats error message for admin alert   | On Error                    | Alert Admin                        | ## Error Handling                                                                               |
| Alert Admin                  | Telegram                        | Sends error alert to admin Telegram chat| Format Error Log             | -                                 | ## Error Handling                                                                               |
| Sticky Note                  | Sticky Note                    | Workflow overview and setup instructions| -                           | -                                 | # YouTube Summary Generator via Telegram (Full workflow explanation and setup instructions)     |
| Sticky Note3                 | Sticky Note                    | Error handling explanation               | -                           | -                                 | ## Error Handling                                                                               |
| Sticky Note4                 | Sticky Note                    | Extract and process video explanation    | -                           | -                                 | ## Extract and Process Video                                                                     |
| Sticky Note5                 | Sticky Note                    | Sending results explanation               | -                           | -                                 | ## Send Result to Telegram                                                                       |
| Sticky Note6                 | Sticky Note                    | Input validation explanation              | -                           | -                                 | ## Input Validation                                                                              |
| Sticky Note9                 | Sticky Note                    | Decodo credentials setup instructions    | -                           | -                                 | ## Create Decodo Credentials (with API key link)                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("On New Message")**  
   - Node Type: Telegram Trigger  
   - Parameters: Listen for "message" updates  
   - Credential: Your Telegram Bot API credentials  
   - Position: Start node  

2. **Add IF Node ("Filter: YouTube URL?")**  
   - Node Type: IF  
   - Condition: Message text contains "youtube." OR "youtu.be" (case sensitive)  
   - Connect: Output from "On New Message" → Input here  

3. **Add Telegram Node ("Send \"Processing...\"")**  
   - Node Type: Telegram  
   - Text: `<i>Processing your input...</i>`  
   - Parse Mode: HTML  
   - Chat ID: Expression `{{$json.message.chat.id}}`  
   - Connect: True output from IF node → This node  

4. **Add Telegram Node ("Send \"Invalid URL\"")**  
   - Node Type: Telegram  
   - Text: `not a youtube url`  
   - Chat ID: Expression `{{$json.message.chat.id}}`  
   - Connect: False output from IF node → This node  

5. **Add Set Node ("Set: Video ID & Config")**  
   - Node Type: Set  
   - Assign:  
     - videoId = Extracted by regex from message text: `{{$json.message.text.match(/youtu\.be\/([^?]+)/)[1]}}`  
     - languageCode = `"en"` (default English)  
   - Connect: Output from "Send Processing..."  

6. **Add HTTP Request Node ("Decodo Youtube Transcript Scrapper")**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://scraper-api.decodo.com/v2/scrape`  
   - Authentication: HTTP Header Auth with Decodo API key  
   - Body parameters (JSON):  
     - target: "youtube_transcript"  
     - query: `{{$json.videoId}}`  
     - language_code: `{{$json.languageCode}}`  
   - Headers: Accept: application/json  
   - Connect: Output from Set node  

7. **Add Code Node ("Extract Transcript")**  
   - Node Type: Code  
   - JS Code: Extract transcript text segments from Decodo response and join into one string (see node details above)  
   - Connect: Output from transcript HTTP node  

8. **Add HTTP Request Node ("Decodo Youtube Metadata Scrapper")**  
   - Node Type: HTTP Request  
   - Same configuration as transcript node but body target: "youtube_metadata"  
   - Connect: Output from Set node  

9. **Add Code Node ("Parse Metadata")**  
   - Node Type: Code  
   - JS Code: Parse metadata fields (title, channel, duration, views, chapters, description, thumbnails, video URL, etc.) and format for AI prompt context  
   - Connect: Output from metadata HTTP node  

10. **Add Langchain Google Gemini Node ("Gemini 3.0")**  
    - Node Type: Langchain LM Chat Google Gemini  
    - Model: models/gemini-3-pro-preview  
    - Credentials: Your Google Palm API credentials  

11. **Add Langchain Chain LLM Node ("Generate TLDR")**  
    - Node Type: Chain LLM  
    - Text Input: Transcript text from "Extract Transcript"  
    - AI Model Input: Connect to "Gemini 3.0" ai_languageModel input  
    - Prompt: Detailed summarization instructions (as in original node)  
    - Connect: Output from "Extract Transcript" and AI model from "Gemini 3.0"  

12. **Add Merge Node ("Merge: Data + Summary")**  
    - Node Type: Merge  
    - Mode: Combine all inputs  
    - Inputs:  
      - From "Generate TLDR"  
      - From "Parse Metadata"  

13. **Add Telegram Node ("Send Video Info")**  
    - Node Type: Telegram  
    - Text: Formatted metadata summary with Markdown (title, channel, duration, stats, chapters)  
    - Parse Mode: Markdown  
    - Chat ID: `{{$json.message.chat.id}}` (from original message)  
    - Connect: Output from Merge node  

14. **Add Code Node ("Split & Format HTML")**  
    - Node Type: Code  
    - JS Code: Convert AI summary text to HTML, split into ≤4000 char chunks, add headers/footers with metadata and video URL, prepare message objects for sending  
    - Connect: Output from Merge node  

15. **Add SplitInBatches Node ("Loop: Message Batches")**  
    - Node Type: SplitInBatches  
    - Batch Size: 1 (default)  
    - Connect: Output from "Split & Format HTML"  

16. **Add Telegram Node ("Send Summary Part")**  
    - Node Type: Telegram  
    - Text: Batch message chunk (HTML format)  
    - Parse Mode: HTML  
    - Chat ID: From original message  
    - Connect: Output from "Loop: Message Batches"  

17. **Add Wait Node ("Wait: Rate Limiter")**  
    - Node Type: Wait  
    - Wait Time: 1 second  
    - Connect: Output from "Send Summary Part"  
    - Output: Loops back to "Loop: Message Batches" (to send next batch)  

18. **Add Error Trigger Node ("On Error")**  
    - Node Type: Error Trigger  
    - Connect: none (activates on error)  

19. **Add Code Node ("Format Error Log")**  
    - Node Type: Code  
    - JS Code: Format error details into clean HTML message for alert  
    - Connect: Output from "On Error"  

20. **Add Telegram Node ("Alert Admin")**  
    - Node Type: Telegram  
    - Text: Formatted error message from previous node  
    - Chat ID: Your admin Telegram chat ID (replace `"YOUR_CHAT_ID"`)  
    - Parse Mode: HTML  
    - Connect: Output from "Format Error Log"  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow acts as your personal "TL;DV" (Too Long; Didn't View) assistant directly inside Telegram. Drop a YouTube link to your bot, and it quickly converts the video into a readable summary using Decodo API and Google Gemini AI. It requires three credentials: Telegram API for bot messaging, Decodo API for scraping transcripts and metadata, and Google Gemini Chat model for AI summarization. You can customize language codes, AI prompts, and admin alert chat ID.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note (Workflow Overview)                                                                             |
| Error Handling block catches workflow failures, formats error details, and sends Telegram alerts to an admin for monitoring.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note3 (Error Handling)                                                                                |
| To create Decodo credentials, use HTTP Header Auth with your Decodo API key in Authorization header (Basic [YOUR_DECODO_API_KEY]). API docs and key management are available at https://dashboard.decodo.com/web-scraping-api/scraper?target=google_maps                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note9 (Decodo Credentials Setup)                                                                     |
| The workflow is designed to parse short YouTube URLs (youtu.be) for videoId. For full YouTube URLs or other formats, you may need to enhance regex or add URL parsing logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Observation based on Set node regex usage                                                                    |
| Telegram message length limit is 4096 characters. The Split & Format HTML code node ensures that summary messages are split into smaller chunks below this limit, escaping HTML and converting Markdown-like formatting for compatibility.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Implementation detail in "Split & Format HTML" node                                                         |
| Google Gemini AI model and Decodo API usage are subject to their respective quotas, rate limits, and possible changes in API structure; maintain credentials securely and monitor usage to avoid disruptions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Operational note                                                                                              |

---

**Disclaimer:** This documentation is generated exclusively based on an n8n workflow exported JSON. The workflow processes only legal and public data, adhering strictly to content policies. No illegal, offensive, or protected content is involved.