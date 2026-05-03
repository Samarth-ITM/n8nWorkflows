Bulk YouTube Channel Content Analysis with Apify & DeepSeek AI to Google Sheets

https://n8nworkflows.xyz/workflows/bulk-youtube-channel-content-analysis-with-apify---deepseek-ai-to-google-sheets-9869


# Bulk YouTube Channel Content Analysis with Apify & DeepSeek AI to Google Sheets

---
### 1. Workflow Overview

This workflow automates bulk content analysis of a specified YouTube channel's videos by combining data scraping, AI-powered transcript summarization, and structured output delivery to Google Sheets, with backups to Google Drive and notifications via Gmail. It is designed for content creators, researchers, or analysts who want to extract and summarize video content at scale from YouTube channels.

The workflow is logically divided into the following blocks:

- **1.1 Input & Initialization:** Collect user inputs via a form (channel URL, number of videos, API tokens, etc.) and prepare a Google Sheets tab to store results.
- **1.2 Channel Crawl & Status Loop:** Launch a YouTube scraper actor on Apify, poll for completion, then retrieve the raw video data.
- **1.3 Raw Backup to Google Drive:** Convert the raw JSON data to a file and save it in Google Drive for archival and reprocessing.
- **1.4 Reload & Structured Extraction:** Download the backup file, extract individual video data items, and prepare for batch processing.
- **1.5 Batch Processing & Subtitle Selection:** Split videos into batches and select the first English subtitle track available for each video.
- **1.6 AI Summarization Pipeline:** Use DeepSeek AI agent to analyze subtitles and metadata, generating structured summaries with schema validation and auto-fixing.
- **1.7 Sheet Write & Throttling:** Append the summarized data to Google Sheets with gentle delays to avoid API throttling.
- **1.8 Completion Notification:** Send an email notification to the user when processing finishes successfully.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Initialization

- **Overview:**  
  This block collects all necessary parameters via a web form and initializes the Google Sheets environment by creating or preparing a sheet tab named after the user’s chosen storing name.

- **Nodes Involved:**  
  - `Parameters` (Form Trigger)  
  - `Create_Sheet`

- **Node Details:**

  - **Parameters**  
    - Type: Form Trigger  
    - Role: Accepts user inputs for channel URL, number of videos, storing name, Apify API token, and notification email.  
    - Config: Required fields include `Youtuber_MainPage_URL`, `Total_number_video`, `Apify_API`. Optional `Email` for notification.  
    - Inputs: None (trigger)  
    - Outputs: JSON with form data  
    - Failures: Missing required fields, invalid URL format, or API token errors if used prematurely.

  - **Create_Sheet**  
    - Type: Google Sheets node (Create operation)  
    - Role: Creates a new tab in the specified Google Sheets document, named by `Storing_Name`.  
    - Config: Uses Google Sheets OAuth2 credentials, document ID is fixed, tab name dynamic from form input.  
    - Inputs: From `Parameters`  
    - Outputs: Confirmation of sheet creation  
    - Failures: Authentication errors, sheet name conflicts or quota limits.

---

#### 2.2 Channel Crawl & Status Loop

- **Overview:**  
  Starts the YouTube scraper actor on Apify with provided parameters, waits and polls the actor’s run status until successful completion, then retrieves the scraped video data.

- **Nodes Involved:**  
  - `Start_YouTube_Scraper`  
  - `Wait_15_Seconds`  
  - `Get_Scraper_Status`  
  - `Check_Scraper_Status` (IF)  
  - `Retrieve_Scraped_Data`

- **Node Details:**

  - **Start_YouTube_Scraper**  
    - Type: HTTP Request (POST)  
    - Role: Triggers Apify actor run with JSON body specifying video and subtitles options, channel URL, and max number of videos to fetch.  
    - Config: Uses Apify API token from parameters; requests subtitles download; video filters set to false; maxResults dynamic.  
    - Inputs: From `Create_Sheet`  
    - Outputs: Run initiation response  
    - Failures: API token invalid, rate limiting, malformed request.

  - **Wait_15_Seconds**  
    - Type: Wait node  
    - Role: Pause for 15 seconds between status checks to prevent excessive polling.  
    - Inputs: From `Start_YouTube_Scraper` or `Check_Scraper_Status` (loop)  
    - Outputs: Delayed continuation  
    - Failures: Minimal, but can delay workflow unnecessarily if polling interval is too short or too long.

  - **Get_Scraper_Status**  
    - Type: HTTP Request (GET)  
    - Role: Queries Apify API for the latest run status of the scraper actor.  
    - Config: Uses Apify token, dynamic URL.  
    - Inputs: From `Wait_15_Seconds`  
    - Outputs: JSON with run status  
    - Failures: Network errors, invalid token, API downtime.

  - **Check_Scraper_Status**  
    - Type: IF node  
    - Role: Checks if the run status is "SUCCEEDED" to proceed or loop again.  
    - Config: Condition equals `data.status == "SUCCEEDED"`  
    - Inputs: From `Get_Scraper_Status`  
    - Outputs: If true → `Retrieve_Scraped_Data`, else → `Wait_15_Seconds` (loop)  
    - Failures: Expression errors if input format changes.

  - **Retrieve_Scraped_Data**  
    - Type: HTTP Request (GET)  
    - Role: Retrieves the final dataset JSON from the Apify actor run when completed.  
    - Config: Uses Apify token, requests JSON format, filters for successful status.  
    - Inputs: From `Check_Scraper_Status` (true branch)  
    - Outputs: Raw scraped video data JSON array  
    - Failures: Token errors, empty dataset if scrape failed silently.

---

#### 2.3 Raw Backup to Google Drive

- **Overview:**  
  Converts the raw JSON data to a file and uploads it to Google Drive for archival, enabling audit and possible reprocessing.

- **Nodes Involved:**  
  - `Convert_To_File`  
  - `Upload_To_Google_Drive`

- **Node Details:**

  - **Convert_To_File**  
    - Type: Convert to File node  
    - Role: Converts JSON data into a file format suitable for upload (e.g., JSON file).  
    - Inputs: From `Retrieve_Scraped_Data`  
    - Outputs: File object with metadata  
    - Failures: Data conversion errors if input is malformed.

  - **Upload_To_Google_Drive**  
    - Type: Google Drive node (Upload operation)  
    - Role: Uploads the converted file under a name given by `Storing_Name` to Google Drive root folder.  
    - Config: Uses Google Drive OAuth2 credentials.  
    - Inputs: From `Convert_To_File`  
    - Outputs: Uploaded file metadata including ID  
    - Failures: Auth errors, quota exceeded, upload interruptions.

---

#### 2.4 Reload & Structured Extraction

- **Overview:**  
  Downloads the backup file from Google Drive and extracts its content into individual JSON items for further processing.

- **Nodes Involved:**  
  - `Download_From_Google_Drive`  
  - `Extract_From_File`  
  - `start from?` (disabled code node)

- **Node Details:**

  - **Download_From_Google_Drive**  
    - Type: Google Drive node (Download operation)  
    - Role: Downloads the file just uploaded to Drive using its file ID.  
    - Config: Uses OAuth2 credentials; file ID dynamically from upload node.  
    - Inputs: From `Upload_To_Google_Drive`  
    - Outputs: Binary file data  
    - Failures: File not found, permission errors.

  - **Extract_From_File**  
    - Type: Extract from File node  
    - Role: Parses the downloaded file content and outputs individual JSON objects per video.  
    - Inputs: From `Download_From_Google_Drive`  
    - Outputs: JSON items stream  
    - Failures: Parsing errors if file content corrupted.

  - **start from?**  
    - Type: Code node (disabled)  
    - Role: Allows resuming processing from a certain index in the items array (for advanced partial reruns).  
    - Inputs: From `Extract_From_File`  
    - Outputs: Sliced items array based on `START` variable  
    - Failures: Disabled by default; errors if enabled without proper configuration.

---

#### 2.5 Batch Processing & Subtitle Selection

- **Overview:**  
  Splits the extracted video items into batches for manageable processing and selects the first English subtitle track available from the subtitles metadata.

- **Nodes Involved:**  
  - `Process_Each_Video` (SplitInBatches)  
  - `Select_Subtitle_Language` (Code node)

- **Node Details:**

  - **Process_Each_Video**  
    - Type: SplitInBatches node  
    - Role: Processes the video items in smaller chunks (default batch size) to control load.  
    - Inputs: From `start from?` or directly from extraction  
    - Outputs: Single item per batch for downstream nodes  
    - Failures: Batch size misconfiguration may cause delays or overloads.

  - **Select_Subtitle_Language**  
    - Type: Code node  
    - Role: Iterates subtitle tracks to select the first English (`en`, `english`, or variants) subtitle track’s SRT content and URL.  
    - Inputs: From `Process_Each_Video`  
    - Outputs: JSON with subtitle index, language, SRT text, and URL or null values if none found  
    - Failures: No English subtitles found - AI summary quality may degrade.

---

#### 2.6 AI Summarization Pipeline (Structured)

- **Overview:**  
  This pipeline uses DeepSeek AI to generate structured summaries from video transcript data, enforcing JSON format with auto-fixing and validation.

- **Nodes Involved:**  
  - `DeepSeek Chat Model` (LLM node)  
  - `Content_Analysis_Agent` (LangChain Agent)  
  - `Output_Parser_Auto_Fix` (Auto-fixing output parser)  
  - `Structured_Output_Parser` (Structured output parser)  
  - `Data_Processing_Script` (Code node)

- **Node Details:**

  - **DeepSeek Chat Model**  
    - Type: LangChain LLM node (DeepSeek API)  
    - Role: Provides the language model backend for AI processing.  
    - Credentials: DeepSeek API token  
    - Inputs: From `Content_Analysis_Agent` (as language model)  
    - Outputs: Raw AI completions  
    - Failures: API limits, connectivity, or quota.

  - **Content_Analysis_Agent**  
    - Type: LangChain agent node  
    - Role: Constructs prompt to analyze video metadata and subtitle, instructs AI to generate structured JSON summaries.  
    - Config: System prompt defines role as Video Content Summarization Specialist with strict rules on output format and content.  
    - Inputs: Receives video metadata and subtitles from `Select_Subtitle_Language` and `Process_Each_Video`  
    - Outputs: AI text completions  
    - Failures: Prompt formatting errors, missing subtitles, or AI model errors.

  - **Output_Parser_Auto_Fix**  
    - Type: Auto-fixing output parser  
    - Role: Automatically retries AI output if it does not satisfy JSON schema constraints, improving robustness.  
    - Inputs: From `Content_Analysis_Agent` output  
    - Outputs: Validated and fixed AI output  
    - Failures: Persistent parsing failures if output is malformed.

  - **Structured_Output_Parser**  
    - Type: Structured output parser  
    - Role: Validates AI output against the defined JSON schema example (VideoID, Field, Output).  
    - Inputs: From `Output_Parser_Auto_Fix`  
    - Outputs: Parsed structured JSON  
    - Failures: Schema mismatch or unexpected AI response.

  - **Data_Processing_Script**  
    - Type: Code node  
    - Role: Extracts relevant fields (`VideoID`, `Field`, `Summary`) from AI output for downstream use.  
    - Inputs: From `Content_Analysis_Agent` (after parsing)  
    - Outputs: Cleaned JSON objects per video  
    - Failures: Null or missing fields if AI output incomplete.

---

#### 2.7 Sheet Write & Gentle Throttling

- **Overview:**  
  Gathers all processed data, appends or updates rows in the Google Sheets tab, and inserts small delays to avoid API rate limits.

- **Nodes Involved:**  
  - `Extract_Alldata` (Set node)  
  - `Write_Contentdata_To_Sheet1` (Google Sheets Append/Update)  
  - `Wait_2_Seconds` (Wait node)

- **Node Details:**

  - **Extract_Alldata**  
    - Type: Set node  
    - Role: Collects metadata and AI summary fields into a single JSON object per video for sheet writing.  
    - Inputs: From `Data_Processing_Script` and `Process_Each_Video`  
    - Outputs: Structured rows ready for sheet insertion  
    - Failures: Mapping errors if fields missing.

  - **Write_Contentdata_To_Sheet1**  
    - Type: Google Sheets node (AppendOrUpdate operation)  
    - Role: Adds or updates rows in the specified sheet tab using `VideoID` as key.  
    - Config: Uses Google Sheets OAuth2 credentials, dynamic sheet tab from `Storing_Name`.  
    - Inputs: From `Extract_Alldata`  
    - Outputs: Confirmation of row writes  
    - Failures: API quota exceeded, auth errors, invalid data types.

  - **Wait_2_Seconds**  
    - Type: Wait node  
    - Role: Pauses 2 seconds between writing batches to reduce API throttling risk.  
    - Inputs: From `Write_Contentdata_To_Sheet1`  
    - Outputs: Delayed continuation to next batch  
    - Failures: Minimal.

---

#### 2.8 Completion Notification (Email)

- **Overview:**  
  Sends a notification email to the user upon workflow completion.

- **Nodes Involved:**  
  - `Send a message` (Gmail node)

- **Node Details:**

  - **Send a message**  
    - Type: Gmail node (Send email)  
    - Role: Sends a completion notice to the email provided in form parameters.  
    - Config: Uses Gmail OAuth2 credentials; dynamic recipient and subject including workflow name and timestamp.  
    - Inputs: From `Process_Each_Video` (runs once after processing completes)  
    - Outputs: Email sent confirmation  
    - Failures: Authentication errors, invalid email format, SMTP issues.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                     | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                                     |
|-------------------------|----------------------------------|----------------------------------------------------|-------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------|
| Parameters              | Form Trigger                     | Collect user inputs for channel, tokens, and email | None                          | Create_Sheet                    | ## 1) Input & Initialization - Purpose and form details                                                        |
| Create_Sheet            | Google Sheets (Create)           | Prepare Google Sheets tab for storing results       | Parameters                    | Start_YouTube_Scraper           | ## 1) Input & Initialization                                                                                   |
| Start_YouTube_Scraper   | HTTP Request (POST)              | Start Apify YouTube scraper actor                    | Create_Sheet                  | Wait_15_Seconds                | ## 2) Channel Crawl & Status Loop                                                                               |
| Wait_15_Seconds         | Wait                            | Pause 15 seconds between status polls                | Start_YouTube_Scraper, Check_Scraper_Status | Get_Scraper_Status             | ## 2) Channel Crawl & Status Loop                                                                               |
| Get_Scraper_Status      | HTTP Request (GET)               | Poll Apify actor run status                           | Wait_15_Seconds               | Check_Scraper_Status            | ## 2) Channel Crawl & Status Loop                                                                               |
| Check_Scraper_Status    | IF                              | Check if scraper run finished successfully           | Get_Scraper_Status            | Retrieve_Scraped_Data, Wait_15_Seconds | ## 2) Channel Crawl & Status Loop                                                                               |
| Retrieve_Scraped_Data   | HTTP Request (GET)               | Get raw scraped JSON data                             | Check_Scraper_Status          | Convert_To_File                | ## 2) Channel Crawl & Status Loop                                                                               |
| Convert_To_File         | Convert to File                  | Convert raw JSON to file artifact                     | Retrieve_Scraped_Data         | Upload_To_Google_Drive         | ## 3A) Raw Backup to Google Drive                                                                               |
| Upload_To_Google_Drive  | Google Drive (Upload)            | Upload raw data file to Google Drive                  | Convert_To_File               | Download_From_Google_Drive      | ## 3A) Raw Backup to Google Drive                                                                               |
| Download_From_Google_Drive | Google Drive (Download)        | Download backup file from Drive                       | Upload_To_Google_Drive        | Extract_From_File              | ## 4) Reload & Structured Extraction                                                                            |
| Extract_From_File       | Extract from File                | Extract JSON items from downloaded file               | Download_From_Google_Drive    | start from?                   | ## 4) Reload & Structured Extraction                                                                            |
| start from?             | Code (disabled)                 | Optional resume from specific item index              | Extract_From_File             | Process_Each_Video             | ## (Advanced) Resume from Drive CSV when workflow fails                                                        |
| Process_Each_Video      | SplitInBatches                  | Batch processing of videos                             | start from?                  | Send a message, Select_Subtitle_Language | ## 5) Batch Processing & English Subtitle Selection                                                            |
| Select_Subtitle_Language | Code                           | Select first English subtitle track                   | Process_Each_Video            | Content_Analysis_Agent         | ## 5) Batch Processing & English Subtitle Selection                                                            |
| DeepSeek Chat Model     | LangChain LLM (DeepSeek API)    | AI language model backend                              | Content_Analysis_Agent (ai_languageModel) | Content_Analysis_Agent, Output_Parser_Auto_Fix | ## 6) AI Summarization Pipeline                                                                                  |
| Content_Analysis_Agent  | LangChain Agent                 | Generate structured AI summaries                       | Select_Subtitle_Language, Process_Each_Video | Data_Processing_Script         | ## 6) AI Summarization Pipeline                                                                                  |
| Output_Parser_Auto_Fix  | LangChain Output Parser (AutoFix) | Auto-fix AI output JSON if invalid                     | Content_Analysis_Agent (ai_outputParser) | Content_Analysis_Agent         | ## 6) AI Summarization Pipeline                                                                                  |
| Structured_Output_Parser| LangChain Output Parser (Structured) | Validate AI output schema                              | Output_Parser_Auto_Fix (ai_outputParser) | Output_Parser_Auto_Fix         | ## 6) AI Summarization Pipeline                                                                                  |
| Data_Processing_Script  | Code                           | Extract and clean AI output fields                     | Content_Analysis_Agent        | Extract_Alldata                | ## 6) AI Summarization Pipeline                                                                                  |
| Extract_Alldata         | Set                            | Prepare combined metadata + AI summary for sheets     | Data_Processing_Script, Process_Each_Video | Write_Contentdata_To_Sheet1     | ## 7) Sheet Write & Gentle Throttling                                                                           |
| Write_Contentdata_To_Sheet1 | Google Sheets (AppendOrUpdate) | Write summaries and metadata to Google Sheets         | Extract_Alldata              | Wait_2_Seconds               | ## 7) Sheet Write & Gentle Throttling                                                                           |
| Wait_2_Seconds          | Wait                           | Pause 2 seconds between sheet writes                   | Write_Contentdata_To_Sheet1   | Process_Each_Video            | ## 7) Sheet Write & Gentle Throttling                                                                           |
| Send a message          | Gmail (Send email)              | Notify user of workflow completion                      | Process_Each_Video            | None                        | ## 9) Completion Notification (Email)                                                                           |
| Sticky Note             | Sticky Note                    | Explains Input & Initialization                         | None                         | None                         | ## 1) Input & Initialization                                                                                      |
| Sticky Note1            | Sticky Note                    | Quick start guide and workflow overview                 | None                         | None                         | # 🚀 YouTuber Crawler — Quick Start Guide                                                                        |
| Sticky Note2            | Sticky Note                    | Explains Channel Crawl & Status Loop                    | None                         | None                         | ## 2) Channel Crawl & Status Loop                                                                                 |
| Sticky Note3            | Sticky Note                    | Explains Raw Backup to Google Drive                      | None                         | None                         | ## 3A) Raw Backup to Google Drive                                                                                 |
| Sticky Note4            | Sticky Note                    | Explains Reload & Structured Extraction                  | None                         | None                         | ## 4) Reload & Structured Extraction                                                                              |
| Sticky Note6            | Sticky Note                    | Explains AI Summarization Pipeline                        | None                         | None                         | ## 6) AI Summarization Pipeline                                                                                   |
| Sticky Note7            | Sticky Note                    | Explains Sheet Write & Throttling                         | None                         | None                         | ## 7) Sheet Write & Gentle Throttling                                                                             |
| Sticky Note8            | Sticky Note                    | Explains Completion Notification                          | None                         | None                         | ## 9) Completion Notification (Email)                                                                             |
| Sticky Note9            | Sticky Note                    | Explains Reload & Structured Extraction (continued)      | None                         | None                         | ## 4) Reload & Structured Extraction                                                                              |
| Sticky Note10           | Sticky Note                    | Advanced partial-run resume instructions                   | None                         | None                         | ## (Advanced) Resume from Drive CSV when workflow fails                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node (`Parameters`):**  
   - Type: Form Trigger  
   - Configure form title matching workflow name.  
   - Add required fields:  
     - `Youtuber_MainPage_URL` (string, required)  
     - `Total_number_video` (number, required)  
     - `Storing_Name` (string)  
     - `Apify_API` (string, required)  
     - `Email` (string, optional)  

2. **Add Google Sheets Node (`Create_Sheet`):**  
   - Operation: Create sheet tab  
   - Document: Set with your target Google Sheets document ID  
   - Sheet name: Expression from `Storing_Name` parameter  
   - Credentials: Google Sheets OAuth2  

3. **Connect `Parameters` → `Create_Sheet`.**

4. **Add HTTP Request Node (`Start_YouTube_Scraper`):**  
   - Method: POST  
   - URL: Apify actor run start URL with token from parameters  
   - Body (JSON): Configure to pass video filters (mostly false), download subtitles true, maxResults from parameters, startUrls with target channel URL.  
   - Retry enabled.

5. **Connect `Create_Sheet` → `Start_YouTube_Scraper`.**

6. **Add Wait Node (`Wait_15_Seconds`):**  
   - Wait time: 15 seconds.

7. **Connect `Start_YouTube_Scraper` → `Wait_15_Seconds`.**

8. **Add HTTP Request Node (`Get_Scraper_Status`):**  
   - Method: GET  
   - URL: Apify actor latest run status endpoint with token from parameters  
   - Retry enabled.

9. **Connect `Wait_15_Seconds` → `Get_Scraper_Status`.**

10. **Add IF Node (`Check_Scraper_Status`):**  
    - Condition: Check if `data.status == "SUCCEEDED"` (case sensitive, strict).  

11. **Connect `Get_Scraper_Status` → `Check_Scraper_Status`.**

12. **Add HTTP Request Node (`Retrieve_Scraped_Data`):**  
    - Method: GET  
    - URL: Apify actor latest run dataset items endpoint with token from parameters, filter status SUCCEEDED, format JSON.  
    - Retry enabled.

13. **Connect `Check_Scraper_Status` (true) → `Retrieve_Scraped_Data`.**

14. **Connect `Check_Scraper_Status` (false) → `Wait_15_Seconds` (loop).**

15. **Add Convert To File Node (`Convert_To_File`):**  
    - No special options needed; converts JSON to file.  

16. **Connect `Retrieve_Scraped_Data` → `Convert_To_File`.**

17. **Add Google Drive Node (`Upload_To_Google_Drive`):**  
    - Operation: Upload  
    - File name: From `Storing_Name` parameter  
    - Folder: Root or desired folder in Drive  
    - Credentials: Google Drive OAuth2  

18. **Connect `Convert_To_File` → `Upload_To_Google_Drive`.**

19. **Add Google Drive Node (`Download_From_Google_Drive`):**  
    - Operation: Download  
    - File ID: Output file ID from `Upload_To_Google_Drive`  
    - Credentials: Google Drive OAuth2  

20. **Connect `Upload_To_Google_Drive` → `Download_From_Google_Drive`.**

21. **Add Extract From File Node (`Extract_From_File`):**  
    - Default options to parse JSON array.  

22. **Connect `Download_From_Google_Drive` → `Extract_From_File`.**

23. **(Optional) Add Code Node (`start from?`):**  
    - JS to slice items for partial run resume, disabled by default.

24. **Connect `Extract_From_File` → `start from?`.**

25. **Add SplitInBatches Node (`Process_Each_Video`):**  
    - Default batch size (e.g., 1 or small number).  

26. **Connect `start from?` (or `Extract_From_File` if not using resume) → `Process_Each_Video`.**

27. **Add Code Node (`Select_Subtitle_Language`):**  
    - JS code to select first English subtitle track from subtitles array per item.  

28. **Connect `Process_Each_Video` → `Select_Subtitle_Language`.**

29. **Add LangChain Agent Node (`Content_Analysis_Agent`):**  
    - Configure system prompt to role as Video Content Summarization Specialist with instructions for JSON output schema.  
    - Input: Compose text input from video ID, title, subtitle SRT, description.  
    - Output parser: Enable structured output parsing and auto-fixing.  
    - Retry on fail enabled.

30. **Connect `Select_Subtitle_Language` → `Content_Analysis_Agent`.**

31. **Add LangChain LLM Node (`DeepSeek Chat Model`):**  
    - Credentials: DeepSeek API token  
    - Connect as language model input to `Content_Analysis_Agent`.  

32. **Add LangChain Output Parser Node (`Output_Parser_Auto_Fix`):**  
    - Configure to retry on parsing errors with instructions prompt.  

33. **Add LangChain Output Parser Node (`Structured_Output_Parser`):**  
    - Configure JSON schema example for output: VideoID, Field, Output.  

34. **Connect LLM and output parsers appropriately to enable auto-fixing and validation in sequence with the agent.**

35. **Add Code Node (`Data_Processing_Script`):**  
    - JS code extracts `VideoID`, `Field`, and `Summary` from AI output.  

36. **Connect `Content_Analysis_Agent` → `Data_Processing_Script`.**

37. **Add Set Node (`Extract_Alldata`):**  
    - Map metadata fields from current video and AI summary fields for sheet writing.  

38. **Connect `Data_Processing_Script` and `Process_Each_Video` → `Extract_Alldata`.**

39. **Add Google Sheets Node (`Write_Contentdata_To_Sheet1`):**  
    - Operation: Append or Update rows  
    - Sheet name: From `Storing_Name` parameter  
    - Document ID: Your Google Sheet ID  
    - Key column: `VideoID`  
    - Credentials: Google Sheets OAuth2  

40. **Connect `Extract_Alldata` → `Write_Contentdata_To_Sheet1`.**

41. **Add Wait Node (`Wait_2_Seconds`):**  
    - Wait duration: 2 seconds between writes.  

42. **Connect `Write_Contentdata_To_Sheet1` → `Wait_2_Seconds`.**

43. **Connect `Wait_2_Seconds` → `Process_Each_Video` (loop for batch processing).**

44. **Add Gmail Node (`Send a message`):**  
    - Recipient: Email from form parameters  
    - Subject: Workflow completion notice with timestamp  
    - Credentials: Gmail OAuth2  

45. **Connect one completion branch from `Process_Each_Video` to `Send a message`** (execute once when all batches complete).

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| For first runs, use a small `Total_number_video` (10-20) to validate end-to-end workflow operation.  | Sticky Note #1                                                                                      |
| The YouTube Scraper actor is from Apify: https://apify.com/streamers/youtube-scraper                   | Relevant for API usage and token generation                                                        |
| DeepSeek API is used as the AI language model backend for structured video content summarization.     | Credential setup required for AI summarization nodes                                                |
| Google Sheets, Google Drive, and Gmail OAuth2 credentials must be pre-configured in n8n.              | Workflow integration and authentication                                                            |
| Advanced users can resume processing from Google Drive CSV if AI summarization fails mid-run by enabling `start from?` and reconfiguring connections. | Sticky Note #10                                                                                     |
| The workflow uses a fixed Google Sheets document ID for storage; ensure you update this to your own.  | Node `Create_Sheet` and `Write_Contentdata_To_Sheet1` contain the fixed document ID                 |
| The system prompt for the AI agent is detailed and enforces strict JSON output format for consistency and automation compatibility. | Node `Content_Analysis_Agent`                                                                        |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.