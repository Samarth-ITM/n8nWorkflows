Transform Instagram Reels into Viral Content with Mistral AI, AssemblyAI & Sheets

https://n8nworkflows.xyz/workflows/transform-instagram-reels-into-viral-content-with-mistral-ai--assemblyai---sheets-6729


# Transform Instagram Reels into Viral Content with Mistral AI, AssemblyAI & Sheets

### 1. Workflow Overview

This workflow automates the transformation of Instagram Reels into viral short-form video content ideas using AI and cloud services. It is designed to receive Instagram Reel URLs via Telegram, download and process the video to extract its audio, transcribe the audio to text, analyze the transcription with AI to generate viral content hooks and scripts, and finally log all data into a Google Sheet for management and review.

The workflow is logically divided into six main functional blocks:

- **1.1 Telegram Input & URL Extraction:** Receives user input from Telegram, extracts and validates the Instagram Reel URL.

- **1.2 Reel Metadata Scraping & Validation:** Uses Apify to fetch reel metadata and verify availability.

- **1.3 File Processing (Import & Conversion):** Downloads the reel video, imports it into FreeConvert, and converts it to an MP3 audio file.

- **1.4 Transcription & Validation:** Uploads audio to AssemblyAI for transcription, polls for completion, and validates the presence of speech.

- **1.5 AI Content Analysis & Generation:** Uses Mistral AI and LangChain to generate viral hooks and video scripts from the transcript.

- **1.6 Data Storage & User Notification:** Stores the results in Google Sheets and notifies the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input & URL Extraction

**Overview:**  
This block captures incoming Telegram messages, extracts Instagram Reel URLs using regex, validates the URL presence, and responds to the user if invalid.

**Nodes Involved:**  
- Telegram Trigger  
- Extract::Instagram Reel URL (Code)  
- Validate::URL Extracted (If)  
- Respond::Invalid URL (Telegram)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point for Telegram messages, listens for new messages.  
  - Config: Listens to "message" updates with Telegram API credentials.  
  - Connections: Output to Extract::Instagram Reel URL.  
  - Edge Cases: Telegram API auth failure or message format issues.

- **Extract::Instagram Reel URL**  
  - Type: Code (JavaScript)  
  - Role: Parses Telegram text to find Instagram Reel URLs via regex.  
  - Config: Extracts URL matching pattern `instagram.com/reel/` from message text or related fields; cleans query params and trailing slashes.  
  - Key Variables: `reel_url` in output JSON.  
  - Input: Telegram message JSON.  
  - Output: JSON with original input and extracted `reel_url`.  
  - Edge Cases: No URL present, malformed input, JSON stringify failures.

- **Validate::URL Extracted**  
  - Type: If  
  - Role: Checks if extracted URL is non-empty.  
  - Config: Condition that `reel_url` is not empty string.  
  - Outputs: Pass to Apify scraping if valid; else Respond::Invalid URL.  
  - Edge Cases: Empty or invalid URL triggers error response.

- **Respond::Invalid URL**  
  - Type: Telegram  
  - Role: Sends error message to user if no valid Reel URL found.  
  - Config: Fixed message: "Couldn't find any reel's URL. Please try again" to fixed chat ID.  
  - Edge Cases: Telegram send failure, invalid chat ID.

---

#### 1.2 Reel Metadata Scraping & Validation

**Overview:**  
Fetches Instagram Reel metadata and download URL using Apify API, validates successful retrieval, and handles private or unavailable videos.

**Nodes Involved:**  
- Apify::Scrape Reel Metadata (HTTP Request)  
- Validate::Reel Downloadable (If)  
- Respond::Reel Download Failed1 (Telegram)

**Node Details:**

- **Apify::Scrape Reel Metadata**  
  - Type: HTTP Request  
  - Role: POSTs to Apify Universal Content Extractor with the reel URL to get metadata & video URLs.  
  - Config: Sends JSON body with URLs array containing the extracted reel URL; headers include JSON content type and Apify API bearer token.  
  - Output: JSON including metadata, success flag, and download URLs.  
  - Edge Cases: API key invalid, network errors, API limits.

- **Validate::Reel Downloadable**  
  - Type: If  
  - Role: Checks if Apify response's success flag is true (string equality check).  
  - Output: Pass to download if true; else Respond::Reel Download Failed1.  
  - Edge Cases: False success means private/unavailable video.

- **Respond::Reel Download Failed1**  
  - Type: Telegram  
  - Role: Notifies user that video is private or unavailable.  
  - Config: Fixed message explaining possible private content and suggesting cookies or different video.  
  - Edge Cases: Telegram delivery failures.

---

#### 1.3 File Processing — Import & Conversion

**Overview:**  
Downloads the video file, imports it into FreeConvert for processing, waits for import completion, then submits a job to convert video to MP3 audio, polling until conversion finishes.

**Nodes Involved:**  
- Download::Reel File (HTTP Request)  
- FreeConvert::Import Reel (HTTP Request)  
- Wait (Wait node)  
- Check::Import Status (HTTP Request)  
- Validate::Import Successful (If)  
- FreeConvert::Submit MP3 Conversion (HTTP Request)  
- Wait1 (Wait node)  
- Check::Conversion Status (HTTP Request)  
- Validate::Conversion Complete (If)  
- Download::MP3 File (HTTP Request)

**Node Details:**

- **Download::Reel File**  
  - Type: HTTP Request  
  - Role: Downloads the reel video file using the URL from Apify metadata.  
  - Config: Response type set to file download.  
  - Input: downloadUrl field from Apify node.  
  - Edge Cases: Download failure, invalid URL.

- **FreeConvert::Import Reel**  
  - Type: HTTP Request  
  - Role: Submits video URL to FreeConvert to import for processing.  
  - Config: POST with JSON body specifying import operation and URL; no authentication required.  
  - Output: Job ID and status.  
  - Edge Cases: API limits, invalid URL.

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow by 10 seconds to allow FreeConvert import processing.  
  - Config: 10 seconds fixed delay.

- **Check::Import Status**  
  - Type: HTTP Request  
  - Role: Polls FreeConvert job status using job ID.  
  - Config: GET request to job status endpoint with job ID.  
  - Edge Cases: API timeout, job failures.

- **Validate::Import Successful**  
  - Type: If  
  - Role: Checks if import job status is "completed".  
  - Output: If yes, proceeds to MP3 conversion; else, loops back to Wait (retry).  
  - Edge Cases: Job failure or timeout.

- **FreeConvert::Submit MP3 Conversion**  
  - Type: HTTP Request  
  - Role: Submits conversion job from imported video to MP3 audio.  
  - Config: POST with JSON specifying import and convert tasks, audio codec auto, volume filters, fade-in enabled.  
  - Edge Cases: Conversion errors.

- **Wait1**  
  - Type: Wait  
  - Role: Delays 10 seconds to allow conversion processing.  

- **Check::Conversion Status**  
  - Type: HTTP Request  
  - Role: Polls conversion job status by job ID.  
  - Edge Cases: Network, API, or job errors.

- **Validate::Conversion Complete**  
  - Type: If  
  - Role: Checks if conversion status is "completed".  
  - Output: If yes, proceeds to download MP3; else retries.  

- **Download::MP3 File**  
  - Type: HTTP Request  
  - Role: Downloads the resulting MP3 audio file from FreeConvert.  
  - Edge Cases: Download failures.

---

#### 1.4 Transcription & Validation

**Overview:**  
Uploads the MP3 audio to AssemblyAI for transcription, waits, polls for transcription completion, verifies transcription presence, and handles missing speech cases.

**Nodes Involved:**  
- AssemblyAI::Submit Transcription (HTTP Request)  
- Wait2 (Wait node)  
- Check::Transcript Status (HTTP Request)  
- Validate::Transcript Present (Switch)  
- Validate::Speech Detected (If)  
- Respond::No Speech Found (Telegram)

**Node Details:**

- **AssemblyAI::Submit Transcription**  
  - Type: HTTP Request  
  - Role: POSTs audio URL to AssemblyAI API to start transcription job.  
  - Config: Authorization header with API key; body contains audio_url from MP3 download URL.  
  - Edge Cases: API key invalid, rate limits.

- **Wait2**  
  - Type: Wait  
  - Role: 10 seconds delay for transcription processing.  

- **Check::Transcript Status**  
  - Type: HTTP Request  
  - Role: Polls AssemblyAI transcript job status using transcript ID.  
  - Edge Cases: API or network failures.

- **Validate::Transcript Present**  
  - Type: Switch  
  - Role: Routes based on transcription job status: "completed" proceeds; else loops back to Wait2.  

- **Validate::Speech Detected**  
  - Type: If  
  - Role: Checks if transcription text is non-empty.  
  - Output: If speech detected, continues AI analysis; else Respond::No Speech Found.  

- **Respond::No Speech Found**  
  - Type: Telegram  
  - Role: Notifies user if transcript has no speech detected.  
  - Fixed message advising user to try a different reel.  

---

#### 1.5 AI Content Analysis & Generation

**Overview:**  
Uses Mistral Cloud AI via LangChain to analyze the transcript, identify niche and core message, then generate 3 viral hook titles with scripts. Validates AI output completeness.

**Nodes Involved:**  
- Mistral Cloud Chat Model (AI language model)  
- Structured Output Parser (LangChain Output Parser)  
- AI::Generate Hook Ideas & Scripts (LangChain Agent)  
- Validate::AI Output Fields (If)

**Node Details:**

- **Mistral Cloud Chat Model**  
  - Type: AI Language Model (LangChain)  
  - Role: Provides conversational AI model "mistral-large-pixtral-2411" for content generation.  
  - Credentials: Mistral Cloud API key required.  
  - Output: Model's raw response to prompt.

- **Structured Output Parser**  
  - Type: LangChain Output Parser  
  - Role: Parses AI model's response into structured JSON with fields: niche, core_message, ideas (array with title and script).  
  - Config: Uses example JSON schema to enforce structure.

- **AI::Generate Hook Ideas & Scripts**  
  - Type: LangChain Agent  
  - Role: Core agent sending transcript text to AI with system prompt instructing to identify niche, core message, and generate 3 viral hooks with scripts.  
  - Prompt contains detailed instructions and formatting requirements.  
  - Output: Structured JSON of content strategy.

- **Validate::AI Output Fields**  
  - Type: If  
  - Role: Ensures all required fields (niche, core_message, all 3 ideas with titles and scripts) are non-empty.  
  - Output: If valid, proceeds to data storage; else loops back to AI generation (retry logic implied).  
  - Edge Cases: AI response missing fields or empty content.

---

#### 1.6 Data Storage & User Notification

**Overview:**  
Appends the collected metadata, transcript, AI-generated hooks/scripts, and URLs to a Google Sheet and sends a confirmation message via Telegram.

**Nodes Involved:**  
- Storage::Log to Google Sheet (Google Sheets node)  
- Send a text message (Telegram)

**Node Details:**

- **Storage::Log to Google Sheet**  
  - Type: Google Sheets (OAuth2)  
  - Role: Appends or updates a row in a specific Google Sheet with all relevant data fields including reel metadata, AI output, transcript, and URLs.  
  - Config: Maps fields such as Title, Reel URL, video/audio download URLs, uploader, description, niche, scripts, and transcript text.  
  - Sheet ID and name are fixed to a known spreadsheet.  
  - Edge Cases: Google API auth errors, quota limits.

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends a detailed success message to the user including reel metadata, niche, description, and a link to the Google Sheet.  
  - Chat ID fixed for user.  
  - Edge Cases: Message delivery failures.

---

### 3. Summary Table

| Node Name                    | Node Type                        | Functional Role                             | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                          |
|------------------------------|---------------------------------|--------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger                 | Entry point, receives Telegram messages    |                                  | Extract::Instagram Reel URL       | ## 🔹 TELEGRAM INPUT & URL EXTRACTION: Trigger::Telegram Message is the main entry point           |
| Extract::Instagram Reel URL  | Code                            | Extracts Instagram Reel URL from message   | Telegram Trigger                 | Validate::URL Extracted           |                                                                                                |
| Validate::URL Extracted      | If                              | Validates presence of reel URL              | Extract::Instagram Reel URL      | Apify::Scrape Reel Metadata, Respond::Invalid URL |                                                                                                |
| Respond::Invalid URL         | Telegram                        | Sends error if no valid URL                 | Validate::URL Extracted          |                                  |                                                                                                |
| Apify::Scrape Reel Metadata  | HTTP Request                   | Fetches reel metadata and download URL     | Validate::URL Extracted          | Validate::Reel Downloadable       | ## 🔹 REEL DOWNLOAD & VALIDATION: Uses Apify to fetch metadata + download URL                      |
| Validate::Reel Downloadable  | If                              | Checks if reel is downloadable              | Apify::Scrape Reel Metadata      | Download::Reel File, Respond::Reel Download Failed1 |                                                                                                |
| Respond::Reel Download Failed1 | Telegram                      | Notifies user if reel is private/unavailable | Validate::Reel Downloadable      |                                  |                                                                                                |
| Download::Reel File          | HTTP Request                   | Downloads reel video file                    | Validate::Reel Downloadable      | FreeConvert::Import Reel          | ## 🔹FILE PROCESSING — PHASE 1 (Import): Download reel, submit import job                          |
| FreeConvert::Import Reel     | HTTP Request                   | Imports reel video into FreeConvert         | Download::Reel File              | Wait                            |                                                                                                |
| Wait                        | Wait                            | Waits 10 seconds for import processing      | FreeConvert::Import Reel         | Check::Import Status              |                                                                                                |
| Check::Import Status         | HTTP Request                   | Polls FreeConvert import job status         | Wait                           | Validate::Import Successful       |                                                                                                |
| Validate::Import Successful  | If                              | Validates import completion                  | Check::Import Status             | FreeConvert::Submit MP3 Conversion, Wait (retry) |                                                                                                |
| FreeConvert::Submit MP3 Conversion | HTTP Request             | Submits video-to-MP3 conversion job         | Validate::Import Successful      | Wait1                          | ## 🔹 FILE PROCESSING — PHASE 2 (Convert to Audio): Submit conversion, poll completion             |
| Wait1                       | Wait                            | Waits 10 seconds for conversion              | FreeConvert::Submit MP3 Conversion | Check::Conversion Status         |                                                                                                |
| Check::Conversion Status     | HTTP Request                   | Polls conversion job status                   | Wait1                          | Validate::Conversion Complete    |                                                                                                |
| Validate::Conversion Complete | If                             | Validates audio conversion completion        | Check::Conversion Status         | Download::MP3 File, Wait1 (retry) |                                                                                                |
| Download::MP3 File           | HTTP Request                   | Downloads converted MP3 audio file            | Validate::Conversion Complete    | AssemblyAI::Submit Transcription  | ## 🔹 TRANSCRIPTION & VERIFICATION: Submit audio for transcription, validate speech presence      |
| AssemblyAI::Submit Transcription | HTTP Request               | Starts transcription job on AssemblyAI       | Download::MP3 File              | Wait2                          |                                                                                                |
| Wait2                       | Wait                            | Waits 10 seconds for transcription processing | AssemblyAI::Submit Transcription | Check::Transcript Status          |                                                                                                |
| Check::Transcript Status     | HTTP Request                   | Polls transcription job status                | Wait2                          | Validate::Transcript Present      |                                                                                                |
| Validate::Transcript Present | Switch                         | Routes based on transcript completion status | Check::Transcript Status         | Validate::Speech Detected, Wait2 (retry) |                                                                                                |
| Validate::Speech Detected    | If                              | Checks if transcript text is non-empty        | Validate::Transcript Present     | AI::Generate Hook Ideas & Scripts, Respond::No Speech Found |                                                                                                |
| Respond::No Speech Found     | Telegram                        | Notifies user if no speech detected            | Validate::Speech Detected        |                                  |                                                                                                |
| Mistral Cloud Chat Model     | AI Language Model (LangChain)  | Provides AI model for content generation      |                                | AI::Generate Hook Ideas & Scripts | ## 🔹 AI ANALYSIS, STRATEGY GENERATION & DATA STORAGE: Core AI agent for viral ideas              |
| Structured Output Parser     | LangChain Output Parser        | Parses AI output into structured JSON          |                                | AI::Generate Hook Ideas & Scripts |                                                                                                |
| AI::Generate Hook Ideas & Scripts | LangChain Agent            | Generates 3 viral hooks and scripts from text | Validate::Speech Detected, Mistral Cloud Chat Model, Structured Output Parser | Validate::AI Output Fields          |                                                                                                |
| Validate::AI Output Fields   | If                              | Validates AI output completeness               | AI::Generate Hook Ideas & Scripts | Storage::Log to Google Sheet, AI::Generate Hook Ideas & Scripts (retry) |                                                                                                |
| Storage::Log to Google Sheet | Google Sheets (OAuth2)         | Logs reel metadata, transcript, and AI output | Validate::AI Output Fields      | Send a text message              |                                                                                                |
| Send a text message          | Telegram                        | Notifies user of successful processing         | Storage::Log to Google Sheet    |                                  |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure with Telegram API credentials.  
   - Set update types to listen for "message".  

2. **Create Extract::Instagram Reel URL Node:**  
   - Type: Code  
   - JavaScript to extract Instagram Reel URL from incoming Telegram message text using regex.  
   - Input connected from Telegram Trigger.  

3. **Create Validate::URL Extracted Node:**  
   - Type: If  
   - Condition: `$json.reel_url` is not empty.  
   - Connect success output to Apify node; failure output to Respond::Invalid URL.  

4. **Create Respond::Invalid URL Node:**  
   - Type: Telegram  
   - Fixed message: "Couldn't find any reel's URL. Please try again."  
   - Connect failure branch of Validate::URL Extracted here.  

5. **Create Apify::Scrape Reel Metadata Node:**  
   - Type: HTTP Request  
   - POST to Apify universal-content-extractor API endpoint.  
   - Body JSON with `"urls": [extracted reel_url]`, quality "best", format "mp4", no audioOnly, no watermark removal.  
   - Add "Content-Type: application/json" and "Authorization: Bearer <YOUR_API_KEY>" headers.  
   - Connect success output of Validate::URL Extracted here.  

6. **Create Validate::Reel Downloadable Node:**  
   - Type: If  
   - Condition: `$json.success.toString()` equals "true".  
   - On true, connect to Download::Reel File; on false, connect to Respond::Reel Download Failed1.  

7. **Create Respond::Reel Download Failed1 Node:**  
   - Type: Telegram  
   - Message: "This video is private or not available..."  
   - Connect failure branch of Validate::Reel Downloadable here.  

8. **Create Download::Reel File Node:**  
   - Type: HTTP Request  
   - URL from `$json.downloadUrl` in Apify metadata.  
   - Set response format to file.  
   - Connect success output of Validate::Reel Downloadable here.  

9. **Create FreeConvert::Import Reel Node:**  
   - Type: HTTP Request  
   - POST to FreeConvert jobs endpoint for import/url operation.  
   - JSON body with task import operation and video URL from Download::Reel File node.  
   - Connect from Download::Reel File.  

10. **Create Wait Node:**  
    - Type: Wait  
    - Wait 10 seconds.  
    - Connect from FreeConvert::Import Reel.  

11. **Create Check::Import Status Node:**  
    - Type: HTTP Request  
    - GET FreeConvert job status URL with job ID from import response.  
    - Connect from Wait node.  

12. **Create Validate::Import Successful Node:**  
    - Type: If  
    - Condition: `$json.status` equals "completed".  
    - On true, connect to FreeConvert::Submit MP3 Conversion.  
    - On false, connect back to Wait node for retry.  

13. **Create FreeConvert::Submit MP3 Conversion Node:**  
    - Type: HTTP Request  
    - POST to FreeConvert jobs endpoint with JSON for conversion task from imported file to MP3, with audio filters.  
    - Connect success output of Validate::Import Successful here.  

14. **Create Wait1 Node:**  
    - Type: Wait  
    - Wait 10 seconds.  
    - Connect from FreeConvert::Submit MP3 Conversion.  

15. **Create Check::Conversion Status Node:**  
    - Type: HTTP Request  
    - GET FreeConvert job status URL with conversion job ID.  
    - Connect from Wait1.  

16. **Create Validate::Conversion Complete Node:**  
    - Type: If  
    - Condition: `$json.status` equals "completed".  
    - On true, connect to Download::MP3 File.  
    - On false, connect back to Wait1 for retry.  

17. **Create Download::MP3 File Node:**  
    - Type: HTTP Request  
    - URL from conversion result MP3 file URL.  
    - Connect from Validate::Conversion Complete.  

18. **Create AssemblyAI::Submit Transcription Node:**  
    - Type: HTTP Request  
    - POST to AssemblyAI transcript API with audio_url from MP3 download URL.  
    - Include authorization header with AssemblyAI API key.  
    - Connect from Download::MP3 File.  

19. **Create Wait2 Node:**  
    - Type: Wait  
    - Wait 10 seconds.  
    - Connect from AssemblyAI::Submit Transcription.  

20. **Create Check::Transcript Status Node:**  
    - Type: HTTP Request  
    - GET AssemblyAI transcript status URL with transcript ID.  
    - Include authorization header.  
    - Connect from Wait2.  

21. **Create Validate::Transcript Present Node:**  
    - Type: Switch  
    - Route on `$json.status`: "completed" → Validate::Speech Detected; else → Wait2 (retry).  

22. **Create Validate::Speech Detected Node:**  
    - Type: If  
    - Condition: transcription text `$json.text` is not empty.  
    - On true, connect to Mistral Cloud Chat Model; on false, connect to Respond::No Speech Found.  

23. **Create Respond::No Speech Found Node:**  
    - Type: Telegram  
    - Message: "The Agent couldn't detect any speech in the reel..."  
    - Connect failure branch of Validate::Speech Detected.  

24. **Create Mistral Cloud Chat Model Node:**  
    - Type: AI Language Model (LangChain)  
    - Select model "mistral-large-pixtral-2411" with Mistral Cloud API credentials.  
    - Connect from Validate::Speech Detected success.  

25. **Create Structured Output Parser Node:**  
    - Type: LangChain Output Parser  
    - Configure JSON schema example matching expected AI output.  
    - Connect from Mistral Cloud Chat Model `ai_outputParser` output.  

26. **Create AI::Generate Hook Ideas & Scripts Node:**  
    - Type: LangChain Agent  
    - Configure system prompt to analyze transcript text, identify niche/core message, generate 3 viral hooks and scripts with tone and format instructions.  
    - Connect from Mistral Cloud Chat Model and Structured Output Parser.  

27. **Create Validate::AI Output Fields Node:**  
    - Type: If  
    - Validate all required AI output fields (niche, core_message, 3 idea titles and scripts) are not empty.  
    - On true, connect to Storage::Log to Google Sheet.  
    - On false, connect back to AI::Generate Hook Ideas & Scripts (retry logic).  

28. **Create Storage::Log to Google Sheet Node:**  
    - Type: Google Sheets (OAuth2)  
    - Configure to append or update rows with all relevant fields: reel metadata, URLs, transcript, AI outputs.  
    - Use Google Sheets credentials with access to target spreadsheet ID and sheet name.  
    - Connect from Validate::AI Output Fields success.  

29. **Create Send a text message Node:**  
    - Type: Telegram  
    - Sends success message with reel info and Google Sheet link to user.  
    - Connect from Storage::Log to Google Sheet.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses Apify universal-content-extractor actor for scraping Instagram Reel metadata and downloadable video URLs.        | Apify API documentation: https://docs.apify.com/api/v2/acts/red.cars~universal-content-extractor/run-sync-get-dataset-items           |
| FreeConvert API is used for video import and conversion to MP3 audio.                                                             | FreeConvert API docs: https://freeconvert.com/api                                                                              |
| AssemblyAI provides speech-to-text transcription with polling to detect completion.                                                | AssemblyAI API docs: https://www.assemblyai.com/docs                                                                              |
| Mistral Cloud AI integrates as a LangChain language model for generating viral content hooks and scripts.                          | Mistral Cloud API: https://mistral.ai/                                                                                             |
| Google Sheets OAuth2 credentials are needed for appending data to the spreadsheet.                                                  | Google Sheets API: https://developers.google.com/sheets/api                                                                       |
| Telegram Bot API credentials are required for both receiving messages and sending responses/notifications.                         | Telegram Bot API: https://core.telegram.org/bots/api                                                                               |
| The workflow includes retry loops with Wait nodes for asynchronous job polling to handle eventual consistency and API latency.    |                                                                                                                                    |
| The AI prompt is designed to generate bold, emotional, and natural-sounding scripts ideal for short-form video platforms.          |                                                                                                                                    |
| Spreadsheet link in user message: https://docs.google.com/spreadsheets/d/1UOAzmnYS8Wm8ngkgfap88u6D5koi0t1Eavm9q-ftgho/edit#gid=0   |                                                                                                                                    |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. This process adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.