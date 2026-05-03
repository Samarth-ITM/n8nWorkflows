Telegram Bot: Extract & Store TikTok and LinkedIn Data to Google Sheets with Dumpling AI

https://n8nworkflows.xyz/workflows/telegram-bot--extract---store-tiktok-and-linkedin-data-to-google-sheets-with-dumpling-ai-11546


# Telegram Bot: Extract & Store TikTok and LinkedIn Data to Google Sheets with Dumpling AI

### 1. Workflow Overview

This workflow automates the extraction of public profile data from TikTok and LinkedIn, using Telegram as the input interface. It supports both text and voice messages sent to a Telegram bot. The workflow employs Dumpling AI’s scraping APIs to retrieve profile details, then stores the extracted data in designated Google Sheets tabs. Upon completion, it sends an email notification confirming the successful save.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception and Preprocessing**: Receives Telegram messages (text or voice), downloads and transcribes voice messages, and formats messages for downstream AI processing.
  
- **1.2 AI Processing & Routing**: Uses an AI agent (based on OpenAI GPT-4.1-mini) to determine which social media platform is referenced and orchestrates calls to the appropriate scraping and saving nodes.
  
- **1.3 TikTok Data Extraction and Storage**: Calls Dumpling AI TikTok scraper, then saves extracted TikTok profile data to a Google Sheet.
  
- **1.4 LinkedIn Data Extraction and Storage**: Calls Dumpling AI LinkedIn scraper, then saves extracted LinkedIn profile data to a different Google Sheet tab.
  
- **1.5 Notification**: Sends a confirmation email upon successful data extraction and saving.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preprocessing

- **Overview:**  
  This block captures incoming Telegram messages, distinguishes between voice and text, downloads voice files, transcribes them using OpenAI Whisper, and formats the message text for subsequent AI processing.

- **Nodes Involved:**  
  - Receive Telegram Message  
  - Check for Voice Message  
  - Fetch Voice File from Telegram  
  - Transcribe Audio with Whisper  
  - Format Transcription  
  - Format Text Message

- **Node Details:**

  - **Receive Telegram Message**  
    - Type: Telegram Trigger  
    - Role: Entry point; listens for new Telegram messages (including media)  
    - Config: Listens to “message” updates, downloads media automatically  
    - Input: Telegram webhook input  
    - Output: JSON message data  
    - Possible failures: Telegram API downtime, webhook misconfiguration, media download failures  

  - **Check for Voice Message**  
    - Type: IF node  
    - Role: Conditionally routes based on presence of voice message in Telegram payload  
    - Config: Checks if `message.voice` exists  
    - Inputs: Output of Telegram trigger  
    - Outputs: Two branches — voice present or absent  
    - Edge cases: Unexpected message formats, missing voice object  

  - **Fetch Voice File from Telegram**  
    - Type: Telegram node (file resource)  
    - Role: Downloads the voice message file using Telegram API  
    - Config: Uses `message.voice.file_id` to fetch file  
    - Inputs: Voice branch from IF node  
    - Outputs: Raw audio file data  
    - Failures: File not found, Telegram API errors, network issues  

  - **Transcribe Audio with Whisper**  
    - Type: OpenAI Audio node (Whisper)  
    - Role: Transcribes voice audio to text  
    - Config: Uses OpenAI Whisper transcription  
    - Inputs: Audio file from Telegram node  
    - Outputs: Transcribed text JSON  
    - Failures: API limits, audio format issues  

  - **Format Transcription**  
    - Type: Set node  
    - Role: Extracts transcription text, maps it to field `Text` for uniformity  
    - Config: Sets `Text` = transcription text from previous node  
    - Inputs: Transcription output  
    - Outputs: JSON with `Text` key  

  - **Format Text Message**  
    - Type: Set node  
    - Role: For text messages, extracts message text and maps to `Text` field  
    - Config: Sets `Text` = `message.text` from Telegram  
    - Inputs: Text branch from IF node  
    - Outputs: JSON with `Text` key  

#### 2.2 AI Processing & Routing

- **Overview:**  
  This block employs an AI agent (GPT-4.1-mini) to interpret the user’s message text, determine the target social media platform (TikTok or LinkedIn), and orchestrate the correct scraping and saving operations.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Social Media Extraction Agent

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat  
    - Role: Provides advanced conversational AI capabilities (GPT-4.1-mini)  
    - Config: Model set to “gpt-4.1-mini”  
    - Credentials: OpenAI API key  
    - Inputs: Indirectly used by Social Media Extraction Agent as a language model tool  
    - Outputs: AI responses for agent processing  
    - Failures: API errors, rate limits  

  - **Social Media Extraction Agent**  
    - Type: LangChain Agent node  
    - Role: Main AI agent orchestrator with embedded logic and tool usage to scrape and save social media data  
    - Config:  
      - Receives unified text input (`Text`) from previous formatting nodes  
      - Contains system message with detailed instructions for TikTok and LinkedIn scraping protocols using paired tools (scraper + database saver)  
      - Handles error conditions (null results) and user feedback  
    - Inputs: Text from either transcription or direct message formatting nodes, AI language model, and AI tools (scrapers and savers)  
    - Outputs: Triggers scrape/save nodes and then sends completion email  
    - Edge cases: Ambiguous input messages, invalid usernames/URLs, API failures, scraper returning null, incomplete data  

#### 2.3 TikTok Data Extraction and Storage

- **Overview:**  
  This block scrapes TikTok profile data via Dumpling AI API and appends the data to the TikTok tab in a Google Sheet.

- **Nodes Involved:**  
  - tiktok_profile_scraper  
  - tiktok_database_saver

- **Node Details:**

  - **tiktok_profile_scraper**  
    - Type: HTTP Request Tool  
    - Role: Calls Dumpling AI TikTok profile API with username handle  
    - Config:  
      - POST request to `https://app.dumplingai.com/api/v1/get-tiktok-profile`  
      - Authenticated with Dumpling AI HTTP header credentials  
      - Parameter `handle` dynamically provided by AI agent  
    - Inputs: Called by AI agent tool interface  
    - Outputs: JSON with TikTok profile metrics and details  
    - Failures: API auth errors, invalid handle, network errors, rate limits  

  - **tiktok_database_saver**  
    - Type: Google Sheets Tool  
    - Role: Appends TikTok profile data to Google Sheet “TikTok” tab  
    - Config:  
      - Maps 9 profile fields (Username, verified, secUid, bioLink, followerCount, followingCount, heartCount, videoCount, friendCount) to sheet columns  
      - Document and sheet IDs configured (Google Sheets OAuth2 credentials)  
      - Appends data as new row  
    - Inputs: Data from TikTok scraper via AI agent tool chain  
    - Outputs: Confirmation to AI agent  
    - Failures: Google Sheets API errors, auth issues, schema mismatches, rate limits  

#### 2.4 LinkedIn Data Extraction and Storage

- **Overview:**  
  This block scrapes LinkedIn profile data via Dumpling AI API and appends it to the LinkedIn tab in the same Google Sheet.

- **Nodes Involved:**  
  - linkedin_profile_scraper  
  - linkedin_database_saver

- **Node Details:**

  - **linkedin_profile_scraper**  
    - Type: HTTP Request Tool  
    - Role: Calls Dumpling AI LinkedIn profile API with profile URL  
    - Config:  
      - POST request to `https://app.dumplingai.com/api/v1/linkedin/profile`  
      - Authenticated with Dumpling AI HTTP header credentials  
      - Parameter `url` dynamically provided by AI agent  
    - Inputs: Called by AI agent tool interface  
    - Outputs: JSON with LinkedIn profile details (name, about, image, location, followers, recentPosts link)  
    - Failures: API auth errors, invalid URL, network errors, rate limits  

  - **linkedin_database_saver**  
    - Type: Google Sheets Tool  
    - Role: Appends LinkedIn profile data to Google Sheet “LinkedIn” tab  
    - Config:  
      - Maps 6 profile fields (name, about, image, location, followers, recentPosts link) to sheet columns  
      - Document and sheet IDs configured (Google Sheets OAuth2 credentials)  
      - Appends data as new row  
    - Inputs: Data from LinkedIn scraper via AI agent tool chain  
    - Outputs: Confirmation to AI agent  
    - Failures: Google Sheets API errors, auth issues, schema mismatches, rate limits  

#### 2.5 Notification

- **Overview:**  
  Sends an email notification to a configured email address after successful completion of data extraction and storage.

- **Nodes Involved:**  
  - Send Completion Email

- **Node Details:**

  - **Send Completion Email**  
    - Type: Gmail node  
    - Role: Sends a plain text email notification confirming extraction completion  
    - Config:  
      - Recipient email address is hardcoded (editable)  
      - Subject: “Extraction complete”  
      - Body: Confirmation message about scraping and saving  
      - Uses Gmail OAuth2 credentials  
    - Inputs: Triggered after AI agent completes processing  
    - Outputs: None (end node)  
    - Failures: Gmail API errors, auth issues, quota limits  

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                            | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                            |
|----------------------------|----------------------------------|--------------------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Receive Telegram Message    | Telegram Trigger                  | Entry point; receives Telegram messages     | —                            | Check for Voice Message       | 📥 INPUT HANDLING: Receives messages via Telegram and processes both formats (voice and text). Supports natural language commands and voice messages.                                                                                                                                                                                                                                                 |
| Check for Voice Message     | IF                               | Determines if message is voice or text      | Receive Telegram Message      | Fetch Voice File from Telegram, Format Text Message | 📥 INPUT HANDLING: Branches processing based on voice presence.                                                                                                                                                                                                                                                                                                                                       |
| Fetch Voice File from Telegram | Telegram node (file resource)   | Downloads voice file from Telegram API      | Check for Voice Message       | Transcribe Audio with Whisper | 📥 INPUT HANDLING: Downloads voice audio for transcription.                                                                                                                                                                                                                                                                                                                                            |
| Transcribe Audio with Whisper | OpenAI Audio (Whisper)           | Transcribes voice audio to text              | Fetch Voice File from Telegram | Format Transcription          | 📥 INPUT HANDLING: Converts audio to text using OpenAI Whisper.                                                                                                                                                                                                                                                                                                                                       |
| Format Transcription        | Set                              | Normalizes transcription to `Text` field    | Transcribe Audio with Whisper | Social Media Extraction Agent | 📥 INPUT HANDLING: Prepares transcribed text for AI processing.                                                                                                                                                                                                                                                                                                                                        |
| Format Text Message         | Set                              | Extracts text message into `Text` field     | Check for Voice Message       | Social Media Extraction Agent | 📥 INPUT HANDLING: Prepares text messages for AI processing.                                                                                                                                                                                                                                                                                                                                           |
| OpenAI Chat Model           | LangChain OpenAI Chat            | Provides GPT-4.1-mini language model        | —                            | Social Media Extraction Agent |                                                                                                                                                                                                                                                                                                                                                                                                        |
| Social Media Extraction Agent | LangChain Agent                 | AI orchestrator: determines platform, calls scrapers & savers | Format Transcription, Format Text Message, OpenAI Chat Model, Scrapers and Savers | Send Completion Email         |                                                                                                                                                                                                                                                                                                                                                                                                        |
| tiktok_profile_scraper      | HTTP Request Tool                | Calls Dumpling AI TikTok profile API        | Social Media Extraction Agent | tiktok_database_saver         | 📱 TIKTOK EXTRACTION PIPELINE: Scrapes TikTok profiles by username/handle.                                                                                                                                                                                                                                                                                                                            |
| tiktok_database_saver       | Google Sheets Tool               | Saves TikTok profile data to Google Sheets  | tiktok_profile_scraper        | Social Media Extraction Agent | 📱 TIKTOK EXTRACTION PIPELINE: Saves 9 TikTok data fields to “TikTok” tab.                                                                                                                                                                                                                                                                                                                            |
| linkedin_profile_scraper    | HTTP Request Tool                | Calls Dumpling AI LinkedIn profile API      | Social Media Extraction Agent | linkedin_database_saver       |                                                                                                                                                                                                                                                                                                                                                                                                        |
| linkedin_database_saver     | Google Sheets Tool               | Saves LinkedIn profile data to Google Sheets| linkedin_profile_scraper      | Social Media Extraction Agent |                                                                                                                                                                                                                                                                                                                                                                                                        |
| Send Completion Email       | Gmail                           | Sends email notification on completion      | Social Media Extraction Agent | —                            |                                                                                                                                                                                                                                                                                                                                                                                                        |
| Sticky Note                 | Sticky Note                     | Documentation and overview                   | —                            | —                            | 📊 Social Media Profile Scraper with Dumpling AI: Overview, capabilities, usage, setup checklist, and required Google Sheets columns.                                                                                                                                                                                                                                                                  |
| Sticky Note1                | Sticky Note                     | Documentation for input handling             | —                            | —                            | 📥 INPUT HANDLING: Explains voice and text processing in Telegram input.                                                                                                                                                                                                                                                                                                                              |
| Sticky Note2                | Sticky Note                     | Documentation for TikTok extraction pipeline | —                            | —                            | 📱 TIKTOK EXTRACTION PIPELINE: Details scraping and saving TikTok data flow and example input.                                                                                                                                                                                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure to listen for “message” updates.  
   - Enable media download option.  
   - Connect Telegram API credentials.  
   - Use webhook for receiving messages.

2. **Add IF Node to Check for Voice Message:**  
   - Condition: Check if `message.voice` exists (`exists` operator).  
   - Connect from Telegram Trigger node.

3. **Add Telegram Node to Fetch Voice File:**  
   - Type: Telegram (file resource).  
   - Set File ID to `{{$json.message.voice.file_id}}`.  
   - Connect from “true” branch of IF node.  
   - Use Telegram API credentials.

4. **Add OpenAI Whisper Node for Transcription:**  
   - Type: OpenAI Audio (Whisper).  
   - Set operation to “Transcribe”.  
   - Connect from Telegram file node.  
   - Use OpenAI API credentials.

5. **Add Set Node to Format Transcription Text:**  
   - Create a field `Text`.  
   - Set value to `{{$json.text}}` (transcription result).  
   - Connect from Whisper node.

6. **Add Set Node to Format Text Messages:**  
   - Create a field `Text`.  
   - Set value to `{{$json.message.text}}` (direct text message).  
   - Connect from “false” branch of IF node.

7. **Add LangChain OpenAI Chat Model Node:**  
   - Model: GPT-4.1-mini.  
   - Connect to AI agent node as AI language model tool.  
   - Set OpenAI credentials.

8. **Add LangChain Agent Node (Social Media Extraction Agent):**  
   - Input parameter `text`: `User message: {{$json.Text}}`.  
   - Configure system message with detailed instructions for TikTok and LinkedIn scraping and saving logic (as in node’s system prompt).  
   - Add tools: connect to TikTok and LinkedIn scraper and saver nodes, and OpenAI Chat Model node.  
   - Connect input from both formatting nodes.  
   - Connect output to email node.

9. **Add HTTP Request Node for TikTok Profile Scraper:**  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/get-tiktok-profile`  
   - Auth: HTTP Header Auth with Dumpling AI credentials  
   - Body parameter: `handle` from AI agent input  
   - Connect as AI tool to the agent node.

10. **Add Google Sheets Node for TikTok Data Saver:**  
    - Operation: Append  
    - Document ID: Set to Google Sheet ID for “Content Library”  
    - Sheet Name: “TikTok” tab (gid=0)  
    - Map columns: Username, verified, secUid, bioLink, followerCount, followingCount, heartCount, videoCount, friendCount from scraper output  
    - Use Google Sheets OAuth2 credentials  
    - Connect as AI tool to the agent node.

11. **Add HTTP Request Node for LinkedIn Profile Scraper:**  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/linkedin/profile`  
    - Auth: HTTP Header Auth with Dumpling AI credentials  
    - Body parameter: `url` from AI agent input  
    - Connect as AI tool to the agent node.

12. **Add Google Sheets Node for LinkedIn Data Saver:**  
    - Operation: Append  
    - Document ID: Same as TikTok saver node  
    - Sheet Name: “LinkedIn” tab (gid=1956409676)  
    - Map columns: name, about, image, location, followers, recentPosts link from scraper output  
    - Use Google Sheets OAuth2 credentials  
    - Connect as AI tool to the agent node.

13. **Add Gmail Node for Completion Email:**  
    - Recipient: Set to desired email address  
    - Subject: “Extraction complete”  
    - Body: Confirmation text about scraping and saving  
    - Use Gmail OAuth2 credentials  
    - Connect from Social Media Extraction Agent node’s main output.

14. **Set Credentials:**  
    - Telegram API credentials (bot token)  
    - OpenAI API key for Chat and Whisper  
    - Dumpling AI HTTP Header Auth credentials (API key)  
    - Google Sheets OAuth2 credentials with access to designated spreadsheet  
    - Gmail OAuth2 credentials for sending email  

15. **Configure Google Sheets:**  
    - Create or verify a Google Sheet with two tabs:  
      - “TikTok” tab with columns: Username, verified, secUid, bioLink, followerCount, followingCount, heartCount, videoCount, friendCount  
      - “LinkedIn” tab with columns: name, image, location, followers, about, recentPosts link  
    - Update node parameters with correct Sheet IDs and tab identifiers.

16. **Activate Workflow:**  
    - Test by sending messages to Telegram bot: e.g.,  
      - Text: “Scrape @charlidamelio”  
      - Text: “Get LinkedIn data: https://linkedin.com/in/username”  
      - Voice message with similar instructions  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                                |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 📊 Social Media Profile Scraper with Dumpling AI: Automatically extracts TikTok and LinkedIn profile data via Telegram input, supports voice and text, saves to Google Sheets, and sends email notifications on completion. Setup requires Telegram bot creation, OpenAI API, Dumpling AI API, Google Sheets with proper columns, Gmail for notifications, and credential configuration.                                                                                                        | Sticky Note content in workflow overview node                                                                                  |
| 📥 INPUT HANDLING: Supports voice messages via Telegram, transcribes with Whisper, and processes text messages. Allows natural language commands like “Get TikTok stats for @username” or “Extract this LinkedIn profile: [URL]”.                                                                                                                                                                                                                                                             | Sticky Note near input nodes                                                                                                   |
| 📱 TIKTOK EXTRACTION PIPELINE: Scrapes TikTok profiles using Dumpling AI, maps 9 key data fields, and appends to a dedicated Google Sheets tab. Example inputs include “@charlidamelio” or “charlidamelio”.                                                                                                                                                                                                                                                     | Sticky Note near TikTok scraper and saver nodes                                                                                 |
| OpenAI GPT-4.1-mini model is used for AI agent processing, enabling sophisticated routing and extraction logic.                                                                                                                                                                                                                                                                                                                                                                                      | OpenAI Chat Model and Agent nodes                                                                                                |
| Dumpling AI APIs require proper authentication via HTTP header credentials for both TikTok and LinkedIn scrapers.                                                                                                                                                                                                                                                                                                                                                                                     | HTTP Request nodes                                                                                                              |
| Google Sheets nodes use OAuth2 credentials and require pre-configured sheets with exact column names for data mapping.                                                                                                                                                                                                                                                                                                                                                                                | Google Sheets nodes                                                                                                             |
| Email notifications rely on Gmail OAuth2 credentials; ensure API access and quota limits are correctly configured.                                                                                                                                                                                                                                                                                                                                                                                    | Gmail node                                                                                                                     |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated process created with n8n, respecting current content policies and containing no illegal, offensive, or protected elements. All handled data is legal and public.