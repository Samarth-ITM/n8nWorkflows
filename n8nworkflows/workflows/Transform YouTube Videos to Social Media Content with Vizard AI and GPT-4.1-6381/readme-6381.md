Transform YouTube Videos to Social Media Content with Vizard AI and GPT-4.1

https://n8nworkflows.xyz/workflows/transform-youtube-videos-to-social-media-content-with-vizard-ai-and-gpt-4-1-6381


# Transform YouTube Videos to Social Media Content with Vizard AI and GPT-4.1

### 1. Workflow Overview

This workflow automates transforming YouTube videos into engaging social media content using Vizard AI and GPT-4.1. The primary use case is to scrape a YouTube channel's video feed, send new videos to Vizard’s AI platform for processing, retrieve detailed project and video data, generate social media captions via GPT-4.1, and log everything into a Google Sheet for easy management and review. An email notification is sent when clips are ready.

The workflow is logically divided into two main functional blocks:

- **1.1 Scrape & Send Videos to Vizard AI**: Trigger starts the workflow, scrapes YouTube channel RSS, limits the number of videos processed, and sends new video URLs to Vizard for AI processing.

- **1.2 Retrieve & Generate Content**: Receives webhook callbacks from Vizard with project data, splits the videos array, loops over each video to generate captions using GPT-4.1, appends data to Google Sheets, waits briefly, and sends an email notification.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Scrape & Send Videos to Vizard AI

**Overview:**  
This block initiates the workflow manually, scrapes the latest videos from a specific YouTube channel using its RSS feed, limits the number of videos processed per run, and sends each new video link to Vizard AI for content creation.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Read youtube RSS feed (RSS Feed Read)  
- Limit (Limit)  
- Send Longform to Vizard (HTTP Request)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow.  
  - Configuration: No parameters required.  
  - Input: None  
  - Output: Triggers next node: Read youtube RSS feed.  
  - Potential failures: None, since manual trigger.  

- **Read youtube RSS feed**  
  - Type: RSS Feed Read  
  - Role: Reads the RSS feed of a YouTube channel to get latest videos.  
  - Configuration: Set to YouTube channel RSS URL `https://www.youtube.com/feeds/videos.xml?channel_id=UCbo-KbSjJDG6JWQ_MTZ_rNA`.  
  - Expressions: None.  
  - Input: Trigger from manual node.  
  - Output: Array of video items. Connects to Limit node.  
  - Edge cases: RSS feed unavailable, network errors, channel ID incorrect.  

- **Limit**  
  - Type: Limit  
  - Role: Restricts the number of videos processed per workflow run.  
  - Configuration: Max items set to 2 (process only first two videos).  
  - Input: Array of videos from RSS feed.  
  - Output: Limited array forwarded to next node.  
  - Edge cases: If fewer than 2 videos available, processes all.  
  - Sticky note context: Advises to deactivate limit when deploying live.  

- **Send Longform to Vizard**  
  - Type: HTTP Request  
  - Role: Sends video URLs to Vizard AI for longform project creation.  
  - Configuration:  
    - POST to `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/create`  
    - JSON body includes language (English), preference length (0), video URL from current item, videoType=2 (likely YouTube).  
    - Header includes `VIZARDAI_API_KEY` (to be set by user).  
  - Expressions: Video URL dynamically set from `$json.link`.  
  - Input: Limited list of videos.  
  - Output: Response from Vizard API (project creation confirmation).  
  - Edge cases: API key invalid, API downtime, malformed response, request timeout.  

---

#### Block 1.2: Retrieve & Generate Content

**Overview:**  
This block handles incoming project data from Vizard via webhook, processes each video item, generates captions using GPT-4.1, logs data to Google Sheets, waits briefly, and sends an email notification.

**Nodes Involved:**  
- Webhook  
- Retrieve Vizard Project (HTTP Request)  
- Split Out (Split Out)  
- Loop Over Items (Split In Batches)  
- Generate captions (OpenAI GPT-4.1)  
- Append row in sheet (Google Sheets)  
- Wait for processing (Wait)  
- Send a message (Gmail)  

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Receives POST requests from Vizard with project data updates.  
  - Configuration: HTTP POST at path `d40c9293-818e-49c9-bcff-a864bd524427`.  
  - Input: Incoming webhook data (JSON with projectId).  
  - Output: Passes data to Retrieve Vizard Project node.  
  - Edge cases: Webhook not reachable, invalid payload, unauthorized requests.  

- **Retrieve Vizard Project**  
  - Type: HTTP Request  
  - Role: Fetches detailed project info from Vizard using projectId from webhook.  
  - Configuration:  
    - GET request to `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/query/{{$json.projectId}}`  
    - Header includes `VIZARDAI_API_KEY`.  
  - Expressions: URL dynamically set using projectId from webhook.  
  - Input: Webhook output with projectId.  
  - Output: JSON containing project with videos array.  
  - Edge cases: API failures, invalid projectId, API key issues.  

- **Split Out**  
  - Type: Split Out  
  - Role: Extracts “videos” array from project data to process each video individually.  
  - Configuration: Split out on field `videos`.  
  - Input: Project JSON with videos array.  
  - Output: One item per video.  
  - Edge cases: Missing or empty videos array.  

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes videos one by one or in batches for downstream nodes.  
  - Configuration: Default batch size (1 item) to sequentially process videos.  
  - Input: Individual video items from Split Out node.  
  - Output: Each video item sent to two parallel branches: Send a message and Generate captions.  
  - Edge cases: Batch processing errors, large video sets causing delays.  

- **Generate captions**  
  - Type: OpenAI GPT-4.1 (LangChain integration)  
  - Role: Generates social media captions based on video transcript.  
  - Configuration:  
    - Model: gpt-4.1  
    - System prompt instructs assistant to generate Instagram/TikTok captions in JSON format with rules on tone, length, style, emojis, reading level.  
    - User prompt feeds `$json.transcript` from video data.  
    - Output: JSON with a caption field.  
  - Input: Video item with transcript.  
  - Output: JSON caption added to data.  
  - Edge cases: OpenAI API rate limits, malformed transcript, timeouts.  

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Logs video metadata and generated captions into a Google Sheet.  
  - Configuration:  
    - Append operation to sheet ID `10CRVnoFOtY8p6mhX4d7wjbKB91RS3SgJ2v4dwmFM8b4` (named "YT LF to tiktok shorts")  
    - Columns mapped include videoId, projectId, videoUrl, transcript, viralScore, viralReason, relatedTopic, clipEditorUrl, video duration, and generatedCaption.  
  - Expressions: Most fields from the video item JSON and generated caption.  
  - Input: Captioned video item from Generate captions.  
  - Output: Passes to Wait for processing.  
  - Edge cases: Google Sheets API quota, incorrect sheet ID, schema mismatch.  

- **Wait for processing**  
  - Type: Wait  
  - Role: Pauses workflow for 2 seconds before continuing looping.  
  - Configuration: 2 seconds wait time.  
  - Input: After appending row.  
  - Output: Loops back to Loop Over Items node, enabling batch processing flow.  
  - Edge cases: Unnecessary wait if no large batches; minimal failure risk.  

- **Send a message**  
  - Type: Gmail  
  - Role: Sends an email notification to a designated user when clips are ready.  
  - Configuration:  
    - Recipient: `nickolassaraev@gmail.com`  
    - Subject: "Hey—your clips are ready to go!"  
    - Body: Text message with a shared Google Sheet link to review clips.  
  - Input: Receives video item from Loop Over Items (parallel to Generate captions).  
  - Output: Final node in one branch.  
  - Edge cases: Gmail authentication failures, email quota exceeded, invalid email address.  

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                               | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                              |
|----------------------------|------------------------|-----------------------------------------------|--------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Entry point to start workflow manually        | None                           | Read youtube RSS feed        | # 1. Scrape & send flow description, config notes on YouTube Channel ID and Vizard Credentials.        |
| Read youtube RSS feed       | RSS Feed Read          | Scrapes YouTube channel videos via RSS        | When clicking ‘Execute workflow’ | Limit                       |                                                                                                        |
| Limit                      | Limit                  | Limits number of videos processed per run     | Read youtube RSS feed           | Send Longform to Vizard      | Advises deactivating limit node for live runs.                                                          |
| Send Longform to Vizard     | HTTP Request           | Sends video URLs to Vizard AI for processing  | Limit                          | (Implicit end or webhook callback) | Requires Vizard API key authentication.                                                                |
| Webhook                    | Webhook                | Receives project data from Vizard post-processing | (External trigger)              | Retrieve Vizard Project      | # 2. Retrieve & generate block info, Google Sheets template link.                                       |
| Retrieve Vizard Project     | HTTP Request           | Fetches detailed project info from Vizard     | Webhook                       | Split Out                   | Requires Vizard API key authentication.                                                                |
| Split Out                  | Split Out              | Splits videos array from project data         | Retrieve Vizard Project         | Loop Over Items             |                                                                                                        |
| Loop Over Items            | Split In Batches       | Iterates over videos one-by-one or in batches | Split Out                     | Generate captions, Send a message |                                                                                                        |
| Generate captions           | OpenAI (LangChain)     | Generates social media captions from transcript | Loop Over Items               | Append row in sheet          | Uses GPT-4.1, system/user prompts for style and tone.                                                  |
| Append row in sheet         | Google Sheets          | Logs video and caption data into Google Sheet | Generate captions             | Wait for processing          | Google Sheet ID and schema mapping must be set correctly.                                              |
| Wait for processing         | Wait                   | Waits 2 seconds to throttle batch processing  | Append row in sheet             | Loop Over Items             |                                                                                                        |
| Send a message              | Gmail                  | Sends email notification when clips are ready | Loop Over Items (parallel)     | None                        | Email contains spreadsheet link and contact info.                                                     |
| Sticky Note                | Sticky Note            | Informational notes                            | None                          | None                        | Covers multiple nodes as described in sections 1.1 and 1.2.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add RSS Feed Read Node**  
   - Name: `Read youtube RSS feed`  
   - Type: RSS Feed Read  
   - Parameter: URL set to `https://www.youtube.com/feeds/videos.xml?channel_id=UCbo-KbSjJDG6JWQ_MTZ_rNA` (replace with desired channel ID).  
   - Connect output from manual trigger to this node.

3. **Add Limit Node**  
   - Name: `Limit`  
   - Type: Limit  
   - Parameter: Max items = 2 (adjust or disable for live runs).  
   - Connect output from RSS Feed Read to Limit.

4. **Add HTTP Request Node to Send to Vizard**  
   - Name: `Send Longform to Vizard`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/create`  
   - Body content type: JSON  
   - JSON body:
     ```
     {
       "lang": "en",
       "preferLength": [0],
       "videoUrl": "={{ $json.link }}",
       "videoType": 2
     }
     ```
   - Headers: Add header `VIZARDAI_API_KEY` with your Vizard API key credential.  
   - Connect output from Limit node here.

5. **Add Webhook Node**  
   - Name: `Webhook`  
   - Type: Webhook (HTTP POST)  
   - Path: `d40c9293-818e-49c9-bcff-a864bd524427` (or custom unique path).  
   - This node listens for incoming project data from Vizard.

6. **Add HTTP Request Node to Retrieve Vizard Project**  
   - Name: `Retrieve Vizard Project`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/query/{{$json.projectId}}` (use expression to insert `projectId` from webhook JSON).  
   - Headers: Include `VIZARDAI_API_KEY` with your Vizard API key.  
   - Connect output from Webhook node.

7. **Add Split Out Node**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Field to split: `videos` (field in project JSON).  
   - Connect output from Retrieve Vizard Project.

8. **Add Split In Batches Node**  
   - Name: `Loop Over Items`  
   - Type: Split In Batches  
   - Default batch size = 1 (process one video at a time).  
   - Connect output from Split Out.

9. **Add OpenAI Node for Caption Generation**  
   - Name: `Generate captions`  
   - Type: OpenAI (LangChain)  
   - Model: GPT-4.1  
   - System prompt:  
     ```
     You're a helpful, intelligent social media assistant. You make captions for Instagram and TikTok.
     ```
   - User prompt:  
     ```
     Your task is to generate high-quality, engaging captions for Instagram and TikTok.

     You'll be fed a transcript.

     Return your captions in JSON using this format:

     {"caption":""}

     Rules:
     - Keep captions to ~100 words.
     - Use a spartan tone of voice, favoring the classic Western style (though still a fit for Instagram and TikTok).
     - Write conversationally, i.e as if I were doing the writing myself (in first person).
     - Use emojis, but sparingly.
     - Ensure each sentence is over 5 words long. Write for a University reading level.
     ```
   - Pass transcript dynamically using: `{{$json.transcript}}`  
   - Connect output from Loop Over Items.

10. **Add Google Sheets Node to Append Row**  
    - Name: `Append row in sheet`  
    - Type: Google Sheets  
    - Operation: Append  
    - Spreadsheet ID: `10CRVnoFOtY8p6mhX4d7wjbKB91RS3SgJ2v4dwmFM8b4` (replace with your Google Sheet ID)  
    - Sheet Name: `Sheet1` or `gid=0`  
    - Map columns: videoId, projectId, videoUrl, transcript, viralScore, viralReason, relatedTopic, clipEditorUrl, videoMsDuration, generatedCaption (map expressions from incoming JSON and generated caption).  
    - Connect output from Generate captions.

11. **Add Wait Node**  
    - Name: `Wait for processing`  
    - Type: Wait  
    - Duration: 2 seconds  
    - Connect output from Append row in sheet.

12. **Loop Wait back to Loop Over Items**  
    - Connect Wait node output back to Loop Over Items node to continue batch processing.

13. **Add Gmail Node to Send Email Notification**  
    - Name: `Send a message`  
    - Type: Gmail (Send Email)  
    - Configuration:  
      - Recipient: `nickolassaraev@gmail.com` (replace with desired recipient)  
      - Subject: "Hey—your clips are ready to go!"  
      - Message:  
        ```
        Hi Nick,

        Your clips are ready to go. Just check the spreadsheet below: https://docs.google.com/spreadsheets/d/1uo3Cq4AoSNhZW8sZup8V5AM55BRua7skVf9gOjxg-Wg/edit?usp=sharing

        Happy clipping!

        Thanks,
        Nick
        ```  
    - Connect output from Loop Over Items (parallel branch).

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Get my Google Sheets template [here](https://docs.google.com/spreadsheets/d/1uo3Cq4AoSNhZW8sZup8V5AM55BRua7skVf9gOjxg-Wg)   | Reference sheet template for logging video and caption data.                                                     |
| Adjust YouTube Channel ID in RSS Feed node to target desired channel.                                                         | To scrape videos from any YouTube channel, replace the channel_id in the RSS feed URL.                           |
| Vizard API key must be securely stored and configured in HTTP Request nodes.                                                  | Necessary for authentication to Vizard AI endpoints.                                                             |
| Deactivate the Limit node when running in production to process all new videos.                                               | Limit node is initially set to 2 for testing/small batches.                                                      |
| Email notification sends a link to review clips on Google Sheets; update recipient in Gmail node accordingly.                | Gmail node requires OAuth2 credentials for sending email via user’s account.                                      |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with prevailing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.