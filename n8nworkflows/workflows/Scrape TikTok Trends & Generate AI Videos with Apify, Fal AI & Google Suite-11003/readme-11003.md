Scrape TikTok Trends & Generate AI Videos with Apify, Fal AI & Google Suite

https://n8nworkflows.xyz/workflows/scrape-tiktok-trends---generate-ai-videos-with-apify--fal-ai---google-suite-11003


# Scrape TikTok Trends & Generate AI Videos with Apify, Fal AI & Google Suite

### 1. Workflow Overview

This n8n workflow automates the end-to-end pipeline for researching TikTok trends related to job change and employment topics and generating AI-driven short vertical videos based on the findings. It integrates TikTok data scraping, AI prompt generation, video creation via a specialized API, and cloud storage.

The workflow is logically divided into these main blocks:

- **1.1 Initial Trigger and TikTok Data Scraping:** Start execution manually, trigger Apify’s TikTok scraper to collect trending videos for specified search queries.
- **1.2 Raw Data Storage and Throttling:** Store scraped TikTok metadata in Google Sheets and limit throughput to control processing pace.
- **1.3 AI Prompt Generation:** Use an OpenRouter language model and an AI Agent to analyze TikTok content and produce detailed video creation prompts.
- **1.4 Prompt Storage:** Append AI-generated prompts to a dedicated Google Sheet tab.
- **1.5 Video Generation via Fal AI:** Submit prompts to Fal AI’s Veo3 model to create 8-second vertical videos with audio.
- **1.6 Wait & Video Retrieval:** Pause workflow to allow video generation, then retrieve the generated video link.
- **1.7 Upload to Google Drive:** Upload the generated video file to a specified Google Drive folder for storage and sharing.

---

### 2. Block-by-Block Analysis

#### 2.1 Initial Trigger and TikTok Data Scraping

- **Overview:** This block initiates the workflow manually and triggers the Apify TikTok scraper actor to collect TikTok videos related to “転職” (job change) and “就職” (employment). It sends a POST request to start scraping, then waits for the scraping to complete before requesting the latest dataset of scraped items.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (ManualTrigger)  
  - HTTP Request (POST to start scraper)  
  - Wait (10 seconds delay)  
  - HTTP Request1 (GET latest dataset of scraped TikTok videos)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually.  
    - Inputs: None  
    - Outputs: Triggers next node HTTP Request.

  - **HTTP Request**  
    - Type: HTTP POST Request  
    - Role: Starts Apify TikTok scraper with search queries "転職" and "就職".  
    - Config: JSON body with scrape parameters (max profiles, results per page, no downloads), Apify token passed as URL query param.  
    - Input: Trigger node  
    - Output: Triggers Wait node.  
    - Potential Failures: Invalid token, network timeout, malformed JSON, API limits.

  - **Wait**  
    - Type: Wait node (delay)  
    - Role: Pauses workflow 10 seconds to allow scraper run.  
    - Input: HTTP Request output  
    - Output: Triggers HTTP Request1.

  - **HTTP Request1**  
    - Type: HTTP GET Request  
    - Role: Fetches the latest scraped TikTok dataset items from Apify.  
    - Config: URL with Apify token, no body.  
    - Input: Wait node  
    - Output: Triggers Append or update row in sheet node.  
    - Failures: Token issues, data unavailability, network errors.

#### 2.2 Raw Data Storage and Throttling

- **Overview:** Appends or updates TikTok raw metadata into a Google Sheets tab, then throttles the workflow to control processing speed before proceeding to AI prompt generation.

- **Nodes Involved:**  
  - Append or update row in sheet  
  - Limit  

- **Node Details:**

  - **Append or update row in sheet**  
    - Type: Google Sheets node  
    - Role: Stores detailed TikTok video metadata (id, text, author, stats, hashtags, etc.) into a designated spreadsheet and sheet ("シート1").  
    - Config: Auto-mapping input JSON to sheet columns. Requires Google Sheets OAuth2 credentials.  
    - Input: HTTP Request1 output  
    - Output: Limit node  
    - Failures: Authentication errors, sheet or document not found, mapping mismatches.

  - **Limit**  
    - Type: Limit node (rate control)  
    - Role: Controls number of items proceeding to AI generation at a time to avoid overload.  
    - Input: Append or update row node  
    - Output: AI Agent node  
    - Failures: Misconfiguration may cause throughput bottleneck or stalled runs.

#### 2.3 AI Prompt Generation

- **Overview:** Uses OpenRouter’s language model via the Langchain integration and an AI Agent node to analyze TikTok video metadata and generate a detailed video creation prompt suitable for Fal AI.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - AI Agent  

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: Langchain LLM Chat Node with OpenRouter backend  
    - Role: Provides LLM capabilities for text generation.  
    - Config: Connected to AI Agent node as language model. Requires OpenRouter API credentials.  
    - Input: None (triggered internally by AI Agent)  
    - Output: AI Agent node (llm response).  
    - Failures: API quota/exceed, invalid credentials.

  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Receives TikTok metadata, constructs a prompt for video generation with detailed visual and textual instructions in Japanese, excluding greetings or explanations, focusing only on the prompt.  
    - Config: Uses template expression referencing current item’s `text` and `authorMeta.id` fields.  
    - Input: Limit node  
    - Output: Append or update row in sheet1 node (stores prompt).  
    - Failures: Expression errors, LLM response issues.

#### 2.4 Prompt Storage

- **Overview:** Saves the AI-generated video prompt into a separate Google Sheets tab ("生成済み") for record-keeping and further reference.

- **Nodes Involved:**  
  - Append or update row in sheet1  

- **Node Details:**

  - **Append or update row in sheet1**  
    - Type: Google Sheets node  
    - Role: Appends or updates the AI-generated prompt string into a specified sheet.  
    - Config: Maps the `output` field (the prompt) to the sheet column named "output".  
    - Input: AI Agent output  
    - Output: Fal Submit node (video generation).  
    - Failures: Authentication, sheet access, mapping errors.

#### 2.5 Video Generation via Fal AI

- **Overview:** Sends the generated prompt to Fal AI’s Veo3 model to generate an 8-second vertical video with audio, specifying aspect ratio and duration.

- **Nodes Involved:**  
  - Fal Submit  

- **Node Details:**

  - **Fal Submit**  
    - Type: HTTP Request  
    - Role: POSTs prompt and video generation parameters (aspect ratio 9:16, duration 8s, audio enabled) to Fal AI API endpoint.  
    - Config: Uses HTTP Header Authentication with Fal AI credentials passed via header.  
    - Input: Append or update row in sheet1  
    - Output: Wait1 node (to wait for generation to complete).  
    - Failures: API request failures, auth errors, rate limits.

#### 2.6 Wait & Video Retrieval

- **Overview:** Waits a configurable amount of time (minutes) for Fal AI to complete video generation, then fetches the generated video URL for upload.

- **Nodes Involved:**  
  - Wait1  
  - HTTP Request3  

- **Node Details:**

  - **Wait1**  
    - Type: Wait node  
    - Role: Delays workflow to allow video generation to finish (time unit: minutes).  
    - Input: Fal Submit output  
    - Output: HTTP Request3 node  
    - Failures: Misconfigured wait time may cause premature or delayed retrieval.

  - **HTTP Request3**  
    - Type: HTTP GET Request  
    - Role: Retrieves video generation result from the URL returned by Fal Submit (`response_url`).  
    - Config: Uses Fal AI header authentication.  
    - Input: Wait1 output  
    - Output: Upload file node  
    - Failures: Network errors, invalid URL, auth failures.

#### 2.7 Upload to Google Drive

- **Overview:** Uploads the generated video file to a specified folder in Google Drive for storage and distribution.

- **Nodes Involved:**  
  - Upload file  

- **Node Details:**

  - **Upload file**  
    - Type: Google Drive node  
    - Role: Uploads the file received from Fal AI to the Google Drive folder identified by `folderId`.  
    - Config: Requires Google Drive OAuth2 credentials, drive set to "My Drive", folder ID configured for destination folder.  
    - Input: HTTP Request3 output  
    - Output: (End of workflow)  
    - Failures: Authentication errors, folder not found, file upload issues.

---

### 3. Summary Table

| Node Name                   | Node Type                    | Functional Role                             | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                             |
|-----------------------------|------------------------------|---------------------------------------------|----------------------------|------------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Initiates the workflow manually             | -                          | HTTP Request                 |                                                                                                       |
| HTTP Request                | HTTP Request (POST)           | Starts Apify TikTok scraper                 | When clicking ‘Execute workflow’ | Wait                        |                                                                                                       |
| Wait                       | Wait node                     | Delays 10 seconds for scraper to start      | HTTP Request               | HTTP Request1                |                                                                                                       |
| HTTP Request1              | HTTP Request (GET)            | Fetches latest TikTok scraped data          | Wait                       | Append or update row in sheet|                                                                                                       |
| Append or update row in sheet | Google Sheets                 | Stores raw TikTok metadata                   | HTTP Request1              | Limit                       |                                                                                                       |
| Limit                      | Limit node                   | Controls processing throughput               | Append or update row in sheet | AI Agent                   |                                                                                                       |
| AI Agent                   | Langchain Agent              | Generates video prompt from TikTok data     | Limit                      | Append or update row in sheet1 |                                                                                                       |
| OpenRouter Chat Model      | Langchain LLM Chat Model     | Provides LLM for AI Agent                    | (Internal to AI Agent)      | AI Agent                    |                                                                                                       |
| Append or update row in sheet1 | Google Sheets                 | Stores AI-generated video prompt             | AI Agent                   | Fal Submit                  |                                                                                                       |
| Fal Submit                 | HTTP Request (POST)           | Submits prompt to Fal AI for video creation | Append or update row in sheet1 | Wait1                      |                                                                                                       |
| Wait1                      | Wait node                   | Waits for video generation to complete      | Fal Submit                 | HTTP Request3               |                                                                                                       |
| HTTP Request3              | HTTP Request (GET)            | Retrieves generated video URL                | Wait1                      | Upload file                 |                                                                                                       |
| Upload file                | Google Drive                 | Uploads generated video to Google Drive     | HTTP Request3              | -                          |                                                                                                       |
| Sticky Note                | Sticky Note                  | Setup instructions                           | -                          | -                          | See detailed setup instructions and credentials info.                                                 |
| Sticky Note1               | Sticky Note                  | Workflow description and logic summary      | -                          | -                          | Explains the entire workflow and its automated pipeline.                                              |
| Sticky Note2               | Sticky Note                  | Requirements and prerequisites               | -                          | -                          | Lists required accounts, credentials, and n8n version.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters.

2. **Create HTTP Request Node (Start TikTok Scraper)**  
   - Type: HTTP Request (POST)  
   - URL: `https://api.apify.com/v2/acts/clockworks~tiktok-scraper/runs?token=YOUR_APIFY_TOKEN`  
   - Method: POST  
   - Body (JSON):  
     ```json
     {
       "excludePinnedPosts": false,
       "maxProfilesPerQuery": 10,
       "proxyCountryCode": "None",
       "resultsPerPage": 10,
       "scrapeRelatedVideos": false,
       "searchQueries": ["転職", "就職"],
       "shouldDownloadAvatars": false,
       "shouldDownloadCovers": false,
       "shouldDownloadMusicCovers": false,
       "shouldDownloadSlideshowImages": false,
       "shouldDownloadSubtitles": false,
       "shouldDownloadVideos": false
     }
     ```
   - Connect Manual Trigger → HTTP Request.

3. **Create Wait Node (10 seconds)**  
   - Type: Wait  
   - Amount: 10 seconds  
   - Connect HTTP Request → Wait.

4. **Create HTTP Request Node (Get Latest Dataset)**  
   - Type: HTTP Request (GET)  
   - URL: `https://api.apify.com/v2/acts/clockworks~tiktok-scraper/runs/last/dataset/items?token=YOUR_APIFY_TOKEN`  
   - Connect Wait → HTTP Request1.

5. **Create Google Sheets Node (Store Raw TikTok Data)**  
   - Type: Google Sheets  
   - Operation: Append or update  
   - Document ID: Your spreadsheet ID  
   - Sheet Name: "シート1" (or your sheet)  
   - Map incoming JSON fields (id, text, authorMeta, etc.) to sheet columns automatically.  
   - Connect HTTP Request1 → Append or update row in sheet.

6. **Create Limit Node**  
   - Type: Limit  
   - Use defaults (to throttle workflow)  
   - Connect Append or update row in sheet → Limit.

7. **Create OpenRouter Chat Model Node**  
   - Type: Langchain LLM Chat Model  
   - Configure with OpenRouter credentials.  
   - No direct input connections; it connects internally to the AI Agent node.

8. **Create AI Agent Node**  
   - Type: Langchain Agent  
   - Text parameter: Use template with Japanese instructions referencing TikTok video data, e.g.:  
     ```
     =下記の内容を参照して、動画生成用のプロンプトを作成してください
     挨拶や説明文は省いてください
     プロンプトのみを出力してください

     #参考動画 
     タイトル： {{ $json.text }}
     動画参考URL：{{ $json.authorMeta.id }}

     #出力形式の例
     （続きはノード設定参照）
     ```
   - Connect Limit → AI Agent.  
   - Set AI Agent’s language model to OpenRouter Chat Model node.

9. **Create Google Sheets Node (Store AI Prompt)**  
   - Type: Google Sheets  
   - Operation: Append or update  
   - Document ID: Your spreadsheet ID  
   - Sheet Name: "生成済み"  
   - Map AI Agent output field `output` to the sheet column `output`.  
   - Connect AI Agent → Append or update row in sheet1.

10. **Create HTTP Request Node (Fal AI Submit)**  
    - Type: HTTP Request (POST)  
    - URL: `https://queue.fal.run/fal-ai/veo3`  
    - Authentication: HTTP Header Auth with Fal AI credentials.  
    - Body Parameters:  
      - prompt: `={{$json["output"]}}`  
      - aspect_ratio: "9:16"  
      - duration: "8s"  
      - generate_audio: "true"  
    - Connect Append or update row in sheet1 → Fal Submit.

11. **Create Wait Node (minutes)**  
    - Type: Wait  
    - Unit: minutes (adjust as needed for video generation time)  
    - Connect Fal Submit → Wait1.

12. **Create HTTP Request Node (Retrieve Video)**  
    - Type: HTTP Request (GET)  
    - URL: `={{ $node["Fal Submit"].json["response_url"] }}`  
    - Authentication: Fal AI Header Auth.  
    - Connect Wait1 → HTTP Request3.

13. **Create Google Drive Node (Upload Video)**  
    - Type: Google Drive  
    - Operation: Upload file  
    - Drive: My Drive  
    - Folder ID: Your Google Drive folder ID for storing videos  
    - Connect HTTP Request3 → Upload file.

14. **Credential Setup:**  
    - Apify API token (used in HTTP Request URLs)  
    - OpenRouter API credentials (for Langchain nodes)  
    - Fal AI HTTP Header Auth credentials  
    - Google Sheets OAuth2 credentials  
    - Google Drive OAuth2 credentials

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                                                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Setup Instructions:** Configure all API credentials properly before running. Update spreadsheet and folder IDs in Google Sheets and Drive nodes. Consider switching Apify token usage from URL to header auth for better security.                                            | Sticky Note content in the workflow.                                                                                                                                                                                                     |
| **Workflow Logic Summary:** Automates TikTok trend scraping → metadata storage → AI prompt generation → video creation → upload pipeline. Useful for content creators targeting job/employment niches with automated video production.                                             | Sticky Note1 content in the workflow.                                                                                                                                                                                                   |
| **Requirements:** Requires n8n v1.x or higher, Apify, Fal AI, OpenRouter accounts with credits, and Google Workspace access.                                                                                                                                                 | Sticky Note2 content in the workflow.                                                                                                                                                                                                   |
| Apify TikTok Scraper Actor: https://apify.com/clockworks/tiktok-scraper                                                                                                                                                                                                      | Official Apify actor for TikTok scraping.                                                                                                                                                                                               |
| Fal AI Video Generation: https://fal.ai/                                                                                                                                                                                                                                    | Fal AI platform for AI-based video creation.                                                                                                                                                                                            |
| OpenRouter: https://openrouter.ai/                                                                                                                                                                                                                                          | OpenRouter API for LLM access.                                                                                                                                                                                                           |
| Google Sheets: https://docs.google.com/spreadsheets/                                                                                                                                                                                                                        | For storing raw and processed data.                                                                                                                                                                                                      |
| Google Drive: https://drive.google.com/drive/my-drive                                                                                                                                                                                                                        | For storing generated videos.                                                                                                                                                                                                             |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. All data processed is legal and public. No illegal, offensive, or protected content is included.