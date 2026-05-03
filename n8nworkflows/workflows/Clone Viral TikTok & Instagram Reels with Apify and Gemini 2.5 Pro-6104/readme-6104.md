Clone Viral TikTok & Instagram Reels with Apify and Gemini 2.5 Pro

https://n8nworkflows.xyz/workflows/clone-viral-tiktok---instagram-reels-with-apify-and-gemini-2-5-pro-6104


# Clone Viral TikTok & Instagram Reels with Apify and Gemini 2.5 Pro

### 1. Workflow Overview

This workflow, titled **"Clone Viral TikTok & Instagram Reels with Apify and Gemini 2.5 Pro"**, is designed to reverse engineer short-form videos from Instagram Reels or TikTok. It accepts a video URL, downloads the video using Apify scraping services, converts the video to a base64-encoded string, then sends it to the Google Gemini 2.5 Pro API for deep forensic video analysis. The goal is to produce a highly detailed "Generative Manifest" prompt that can be used by AI video generation platforms (e.g., Google Veo, Sora) to create an exact digital twin of the original video.

The workflow is logically divided into the following blocks:

- **1.1 URL Input & Base Prompt Setup**: Receives the video URL and sets up the base prompt for video analysis.
- **1.2 Platform Detection (Switching)**: Determines whether the input URL is for Instagram or TikTok.
- **2.1 Instagram Reel Processing**: Scrapes the Instagram Reel video, downloads it, converts to base64, builds the prompt, and sends it to Gemini for analysis.
- **2.2 TikTok Video Processing**: Scrapes the TikTok video, downloads it, converts to base64, builds the prompt, and sends it to Gemini for analysis.
- **3. Share Final Prompt**: Converts the generated AI analysis prompt into a text file, uploads it to Slack, and posts a notification message with the prompt link.

---

### 2. Block-by-Block Analysis

#### 1.1 URL Input & Base Prompt Setup

- **Overview:**  
  This block captures the input video URL via a web form and initializes the foundational "Generative Manifest" prompt text that will be appended with video-specific metadata later.

- **Nodes Involved:**  
  - `form_trigger`  
  - `set_base_prompt`

- **Node Details:**

  - **form_trigger**  
    - Type: Form Trigger (Webhook)  
    - Role: Receives user input via a web form containing a required "Url" field for Instagram Reels or TikTok videos.  
    - Configuration: Form titled "Reverse Engineer Short Form Video" with a single required URL field.  
    - Input: HTTP webhook call with form data  
    - Output: JSON with field `Url` containing the video link  
    - Edge Cases: Missing or malformed URL input will fail; no URL validation beyond required field.

  - **set_base_prompt**  
    - Type: Set  
    - Role: Initializes a complex string prompt defining the AI analyst persona and instructions for forensic video deconstruction.  
    - Configuration: Sets a `prompt` string property with a detailed multi-layer textual description instructing the AI on how to analyze and transcribe video characteristics precisely.  
    - Input: JSON from `form_trigger`  
    - Output: JSON with `prompt` property initialized  
    - Edge Cases: None significant; static prompt setup.

  - **Sticky Note (positioned nearby):**  
    - Explains the high-level purpose of this first phase: input URL, download, base64 conversion, send to Gemini API, reverse engineer and generate a perfect AI video generation prompt.

---

#### 1.2 Platform Detection (Switching)

- **Overview:**  
  This block determines whether the input URL is from Instagram or TikTok and routes the workflow accordingly.

- **Nodes Involved:**  
  - `switch_platform`

- **Node Details:**

  - **switch_platform**  
    - Type: Switch  
    - Role: Examines the input URL string to detect if it contains "instagram.com" or "tiktok.com" and routes the execution flow accordingly.  
    - Configuration: Two cases:  
      1. URL contains "instagram.com" → route to Instagram scraping  
      2. URL contains "tiktok.com" → route to TikTok scraping  
    - Input: JSON with `Url` from `set_base_prompt` output  
    - Output: Directs to either `scrape_ig_reel` or `scrape_tiktok`  
    - Edge Cases: URLs without these domains will result in no output branch, effectively dropping the workflow (no fallback or error node).

---

#### 2.1 Instagram Reel Processing

- **Overview:**  
  This block scrapes the Instagram Reel video using Apify, downloads the video file, converts it to base64, builds a detailed prompt, sends it to Google Gemini API for analysis, sets the AI response, and finally converts the prompt result to a file for Slack upload.

- **Nodes Involved:**  
  - `scrape_ig_reel`  
  - `build_final_ig_prompt`  
  - `download_ig_reel`  
  - `ig_to_base64`  
  - `analyze_ig_reel`  
  - `set_ig_result`  
  - `Convert to File`  

- **Node Details:**

  - **scrape_ig_reel**  
    - Type: HTTP Request  
    - Role: Calls Apify's Instagram API scraper to retrieve metadata and video URL for the given Instagram Reel URL.  
    - Configuration: POST to Apify API endpoint with JSON body containing the `directUrls` array set to the input URL, limits results to 1 post. Uses Apify API credentials (HTTP header auth).  
    - Input: URL from `switch_platform`  
    - Output: JSON with video metadata including `videoUrl`  
    - Edge Cases: API request failure, invalid URL, rate limits, or no video returned.

  - **build_final_ig_prompt**  
    - Type: Set  
    - Role: Constructs the final prompt string by appending Instagram video caption and hashtags to the base prompt.  
    - Configuration: Uses expressions to insert `caption` and `hashtags` from scrape results into the prompt string.  
    - Input: JSON from `scrape_ig_reel`  
    - Output: JSON with updated `prompt` string  
    - Edge Cases: Missing caption or hashtags fields.

  - **download_ig_reel**  
    - Type: HTTP Request  
    - Role: Downloads the Instagram Reel video binary file from the scraped video URL.  
    - Configuration: GET request to the `videoUrl` field; response expected as a file binary.  
    - Input: URL from `build_final_ig_prompt`  
    - Output: Binary video file  
    - Edge Cases: Download failure, invalid URL, network errors.

  - **ig_to_base64**  
    - Type: Extract From File  
    - Role: Converts the downloaded video binary into a base64-encoded data string property on the JSON.  
    - Configuration: Operation set to "binaryToProperty" to extract binary data as base64 string into JSON property `data`.  
    - Input: Binary video file from `download_ig_reel`  
    - Output: JSON with base64 `data` property  
    - Edge Cases: Large file size may cause memory or performance issues.

  - **analyze_ig_reel**  
    - Type: HTTP Request  
    - Role: Sends the base64 video data and constructed prompt to the Google Gemini 2.5 Pro API for content generation (video analysis).  
    - Configuration: POST request with JSON body containing:  
      - `contents` array with two parts:  
        1. `inline_data` with mime type "video/mp4" and base64 data  
        2. Text prompt string from `build_final_ig_prompt`  
      Uses Google Gemini HTTP header auth credentials.  
    - Input: JSON with base64 video data and prompt from `ig_to_base64` and `build_final_ig_prompt`  
    - Output: JSON with AI-generated text analysis in `candidates[0].content.parts[0].text`  
    - Edge Cases: API quota limits, authentication errors, request timeouts, malformed input data.

  - **set_ig_result**  
    - Type: Set  
    - Role: Extracts and stores the AI-generated analysis text into a `result` property for downstream processing.  
    - Configuration: Sets `result` string to the first candidate text from Gemini API response.  
    - Input: JSON from `analyze_ig_reel`  
    - Output: JSON with `result` property  
    - Edge Cases: Missing or empty response from Gemini API.

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts the textual `result` into a text file named `prompt.md` (Markdown format) stored as binary data.  
    - Configuration: Converts `result` property to binary file with name `prompt.md`.  
    - Input: JSON with `result` from `set_ig_result`  
    - Output: Binary file for upload  
    - Edge Cases: Large text size, encoding issues.

  - **Sticky Note:**  
    - "2.1 Download Instagram Reel + Analyze Video" — describes this block’s purpose.

---

#### 2.2 TikTok Video Processing

- **Overview:**  
  Similar to the Instagram block but tailored for TikTok videos: scrape video metadata and URL via Apify, download, convert to base64, build prompt including title and hashtags, send to Gemini API, receive analysis, and prepare for Slack sharing.

- **Nodes Involved:**  
  - `scrape_tiktok`  
  - `build_final_tiktok_prompt`  
  - `download_tiktok`  
  - `tiktok_to_base64`  
  - `analyze_tiktok`  
  - `set_tiktok_result`  
  - `Convert to File`  

- **Node Details:**

  - **scrape_tiktok**  
    - Type: HTTP Request  
    - Role: Calls Apify’s TikTok scraper API to get video metadata and URL from the TikTok input URL.  
    - Configuration: POST to Apify TikTok scraper endpoint with JSON body containing `startUrls` array with the input URL and `maxItems` set to 1. Uses Apify API credentials.  
    - Input: URL from `switch_platform`  
    - Output: JSON with video metadata including video URL, title, hashtags  
    - Edge Cases: API request failure, rate limits, invalid URL.

  - **build_final_tiktok_prompt**  
    - Type: Set  
    - Role: Constructs a prompt string by appending the TikTok video title and hashtags to the base prompt.  
    - Configuration: Uses expressions to insert `title` and joined `hashtags` into the prompt string.  
    - Input: JSON from `scrape_tiktok`  
    - Output: JSON with updated `prompt` string  
    - Edge Cases: Missing title or hashtags.

  - **download_tiktok**  
    - Type: HTTP Request  
    - Role: Downloads the TikTok video binary file from the scraped video URL.  
    - Configuration: GET request to the `video.url` field; response as binary file.  
    - Input: URL from `build_final_tiktok_prompt`  
    - Output: Binary video file  
    - Edge Cases: Download failure, invalid URL, network issues.

  - **tiktok_to_base64**  
    - Type: Extract From File  
    - Role: Converts the downloaded TikTok video binary into a base64 string property.  
    - Configuration: Binary to property operation converting binary to base64 string in JSON `data`.  
    - Input: Binary from `download_tiktok`  
    - Output: JSON with base64 `data`  
    - Edge Cases: Large file size.

  - **analyze_tiktok**  
    - Type: HTTP Request  
    - Role: Sends the base64-encoded video plus prompt to Google Gemini API for analysis.  
    - Configuration: Similar to `analyze_ig_reel` but uses the TikTok prompt from `build_final_tiktok_prompt`.  
    - Input: JSON with base64 video data and prompt  
    - Output: Gemini API response with AI-generated text analysis  
    - Edge Cases: API errors, timeouts, auth failures.

  - **set_tiktok_result**  
    - Type: Set  
    - Role: Extracts AI analysis text into `result` property.  
    - Input: Gemini API response  
    - Output: JSON with `result`  
    - Edge Cases: Missing response content.

  - **Convert to File**  
    - Same as Instagram block – converts text `result` to `prompt.md` file.  
    - Input: JSON with `result` from `set_tiktok_result`.

  - **Sticky Note:**  
    - "2.2 Download TikTok + Analyze Video" — describes this block’s purpose.

---

#### 3. Share Final Prompt in Slack

- **Overview:**  
  This block uploads the generated prompt file to Slack and sends a formatted message with the prompt link to a specific Slack channel for collaborative access.

- **Nodes Involved:**  
  - `Convert to File` (from previous blocks)  
  - `upload_file`  
  - `send_prompt_msg`

- **Node Details:**

  - **Convert to File**  
    - Already described above; converts AI analysis text to a Markdown file.

  - **upload_file**  
    - Type: Slack Node (File Upload)  
    - Role: Uploads the generated `prompt.md` file to Slack.  
    - Configuration: Uses Slack OAuth2 credentials; uploads binary `prompt.md` to Slack files.  
    - Input: Binary file from `Convert to File`  
    - Output: Slack file upload response  
    - Edge Cases: Slack API rate limits, auth expiration, file size limits.

  - **send_prompt_msg**  
    - Type: Slack Node (Message)  
    - Role: Sends a Slack message to a specified channel containing the AI video generation prompt permalink.  
    - Configuration: Message text includes the permalink from Slack file upload; posts to channel `C08KC39K8DR` (named `ai-tools-content`). Uses Slack OAuth2 credentials.  
    - Input: Slack file upload response  
    - Output: Slack message sent confirmation  
    - Edge Cases: Slack API errors, invalid channel, auth issues.

  - **Sticky Note:**  
    - "3. Share Final Prompt In Slack" — summarizes this block.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                              | Input Node(s)           | Output Node(s)          | Sticky Note                                               |
|---------------------|----------------------|----------------------------------------------|------------------------|-------------------------|-----------------------------------------------------------|
| form_trigger        | Form Trigger         | Receive video URL input                       | -                      | set_base_prompt          | "1. Url Input + Build Video Reverse Engineering Prompt"   |
| set_base_prompt      | Set                  | Initialize base AI prompt                      | form_trigger           | switch_platform          | "1. Url Input + Build Video Reverse Engineering Prompt"   |
| switch_platform      | Switch               | Detect platform (Instagram or TikTok)         | set_base_prompt        | scrape_ig_reel, scrape_tiktok | "1. Url Input + Build Video Reverse Engineering Prompt"   |
| scrape_ig_reel       | HTTP Request         | Scrape Instagram Reel metadata & video URL    | switch_platform        | build_final_ig_prompt    | "2.1 Download Instagram Reel + Analyze Video"              |
| build_final_ig_prompt| Set                  | Build final AI prompt for Instagram video     | scrape_ig_reel         | download_ig_reel         | "2.1 Download Instagram Reel + Analyze Video"              |
| download_ig_reel     | HTTP Request         | Download Instagram video binary                | build_final_ig_prompt  | ig_to_base64             | "2.1 Download Instagram Reel + Analyze Video"              |
| ig_to_base64         | Extract From File    | Convert IG video binary to base64 string       | download_ig_reel       | analyze_ig_reel          | "2.1 Download Instagram Reel + Analyze Video"              |
| analyze_ig_reel      | HTTP Request         | Send IG video + prompt to Google Gemini API    | ig_to_base64           | set_ig_result            | "2.1 Download Instagram Reel + Analyze Video"              |
| set_ig_result        | Set                  | Extract AI analysis result text from Gemini    | analyze_ig_reel        | Convert to File          | "2.1 Download Instagram Reel + Analyze Video"              |
| scrape_tiktok        | HTTP Request         | Scrape TikTok video metadata & URL             | switch_platform        | build_final_tiktok_prompt| "2.2 Download TikTok + Analyze Video"                      |
| build_final_tiktok_prompt | Set              | Build final AI prompt for TikTok video          | scrape_tiktok          | download_tiktok          | "2.2 Download TikTok + Analyze Video"                      |
| download_tiktok      | HTTP Request         | Download TikTok video binary                     | build_final_tiktok_prompt | tiktok_to_base64       | "2.2 Download TikTok + Analyze Video"                      |
| tiktok_to_base64     | Extract From File    | Convert TikTok video binary to base64 string    | download_tiktok        | analyze_tiktok           | "2.2 Download TikTok + Analyze Video"                      |
| analyze_tiktok       | HTTP Request         | Send TikTok video + prompt to Google Gemini API | tiktok_to_base64       | set_tiktok_result        | "2.2 Download TikTok + Analyze Video"                      |
| set_tiktok_result    | Set                  | Extract AI analysis result text from Gemini     | analyze_tiktok         | Convert to File          | "2.2 Download TikTok + Analyze Video"                      |
| Convert to File      | Convert To File      | Convert AI result text to markdown file          | set_ig_result, set_tiktok_result | upload_file     | "3. Share Final Prompt In Slack"                           |
| upload_file          | Slack (File Upload)  | Upload markdown prompt file to Slack             | Convert to File        | send_prompt_msg          | "3. Share Final Prompt In Slack"                           |
| send_prompt_msg      | Slack (Message)      | Post Slack message with prompt permalink         | upload_file            | -                        | "3. Share Final Prompt In Slack"                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Title: "Reverse Engineer Short Form Video"  
   - Add one required field: "Url" (string, placeholder example: https://www.instagram.com/reel/DKzbAvyoCMJ/)  
   - No additional options needed.

2. **Create Set Node ("set_base_prompt")**  
   - Connect from Form Trigger  
   - Add string field "prompt" with the detailed AI persona and forensic analysis instructions as shown in the overview (multi-paragraph text).  
   - This prompt sets the foundational context for the AI.

3. **Create Switch Node ("switch_platform")**  
   - Connect from "set_base_prompt"  
   - Add two rules:  
     - If `Url` contains "instagram.com" → output 1  
     - If `Url` contains "tiktok.com" → output 2  
   - No default route.

4. **Instagram Branch:**

   a. **HTTP Request Node ("scrape_ig_reel")**  
      - POST to Apify Instagram scraper: `https://api.apify.com/v2/acts/apify~instagram-api-scraper/run-sync-get-dataset-items`  
      - JSON Body:  
        ```json
        {
          "directUrls": ["={{ $json.Url }}"],
          "resultsLimit": 1,
          "resultsType": "posts"
        }
        ```  
      - Authentication: Apify API via HTTP header auth credentials  
      - Connect from Instagram output of "switch_platform"

   b. **Set Node ("build_final_ig_prompt")**  
      - Connect from "scrape_ig_reel"  
      - Set string `prompt` to:  
        ```
        {{$node["set_base_prompt"].json["prompt"]}}
        ---
        Video Caption: {{$json.caption}}
        Video Hashtags: {{$json.hashtags.join(",")}}
        ```

   c. **HTTP Request Node ("download_ig_reel")**  
      - GET request to `{{$json.videoUrl}}`  
      - Response format: File (binary)  
      - Connect from "build_final_ig_prompt"

   d. **Extract From File Node ("ig_to_base64")**  
      - Operation: Binary to Property  
      - Convert binary video data to base64 string in JSON property `data`  
      - Connect from "download_ig_reel"

   e. **HTTP Request Node ("analyze_ig_reel")**  
      - POST to Google Gemini API endpoint:  
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent`  
      - JSON Body:  
        ```json
        {
          "contents": [
            {
              "parts": [
                {
                  "inline_data": {
                    "mime_type": "video/mp4",
                    "data": "{{$json.data}}"
                  }
                },
                {
                  "text": {{$node["build_final_ig_prompt"].json["prompt"] | json_encode}}
                }
              ]
            }
          ]
        }
        ```  
      - Authentication: Google Gemini API HTTP header auth credentials  
      - Connect from "ig_to_base64"

   f. **Set Node ("set_ig_result")**  
      - Extract Gemini API response:  
        Set `result` to `{{$json.candidates[0].content.parts[0].text}}`  
      - Connect from "analyze_ig_reel"

5. **TikTok Branch:**

   a. **HTTP Request Node ("scrape_tiktok")**  
      - POST to Apify TikTok scraper:  
        `https://api.apify.com/v2/acts/apidojo~tiktok-scraper/run-sync-get-dataset-items`  
      - JSON Body:  
        ```json
        {
          "maxItems": 1,
          "startUrls": ["={{ $json.Url }}"]
        }
        ```  
      - Authentication: Apify API HTTP header auth credentials  
      - Connect from TikTok output of "switch_platform"

   b. **Set Node ("build_final_tiktok_prompt")**  
      - Connect from "scrape_tiktok"  
      - Set string `prompt` to:  
        ```
        {{$node["set_base_prompt"].json["prompt"]}}
        ---
        Video Title: {{$json.title}}
        Video Hashtags: {{$json.hashtags.join(",")}}
        ```

   c. **HTTP Request Node ("download_tiktok")**  
      - GET request to `{{$json.video.url}}`  
      - Response format: File (binary)  
      - Connect from "build_final_tiktok_prompt"

   d. **Extract From File Node ("tiktok_to_base64")**  
      - Operation: Binary to Property  
      - Convert binary video to base64 string property `data`  
      - Connect from "download_tiktok"

   e. **HTTP Request Node ("analyze_tiktok")**  
      - POST to Google Gemini API (same as IG)  
      - JSON Body as in IG branch, replacing prompt with TikTok prompt  
      - Connect from "tiktok_to_base64"

   f. **Set Node ("set_tiktok_result")**  
      - Set `result` to Gemini API response text:  
        `{{$json.candidates[0].content.parts[0].text}}`  
      - Connect from "analyze_tiktok"

6. **Final Steps (Shared for Both Platforms):**

   a. **Convert To File Node ("Convert to File")**  
      - Convert the string `result` property into a Markdown text file named `prompt.md`  
      - Connect from both `set_ig_result` and `set_tiktok_result`

   b. **Slack Upload Node ("upload_file")**  
      - Upload binary file `prompt.md` to Slack  
      - Use Slack OAuth2 credentials  
      - Connect from "Convert to File"

   c. **Slack Message Node ("send_prompt_msg")**  
      - Post a message to Slack channel `ai-tools-content` (channel ID: `C08KC39K8DR`)  
      - Message text includes the permalink URL of the uploaded prompt file:  
        ```
        =*AI Video Generator Prompt:*
        
        ```
        {{$json.permalink}}
        ```
        ```  
      - Connect from "upload_file"

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow is designed to produce a *shot-for-shot, motion-for-motion, photon-for-photon digital twin* prompt, emphasizing precision over creativity. | Branding and workflow description                                                              |
| Apify API credentials must be configured properly with valid API keys for Instagram and TikTok scrapers.        | Apify documentation: https://apify.com/docs                                                         |
| Google Gemini API requires proper authentication with an API key configured for HTTP header authentication.     | Gemini API Docs: https://developers.generativeai.google/api/generative-language/reference/rest  |
| Slack OAuth2 credentials must be set up with scopes allowing file upload and message posting in the target workspace. | Slack API docs: https://api.slack.com/authentication/oauth-v2                                   |
| The base prompt contains in-depth forensic video analysis instructions intended for generative AI models like Google Veo or Sora. | Workflow internal prompt text                                                                    |
| Sticky notes within the workflow clarify blocks and are crucial for understanding node grouping and purpose.   | Visible in the n8n editor for contextual guidance                                                |

---

**Disclaimer:**  
The text and data described originate exclusively from an automated n8n workflow. All content complies fully with applicable content policies and contains no illegal, offensive, or protected material. All processed data is public and lawful.