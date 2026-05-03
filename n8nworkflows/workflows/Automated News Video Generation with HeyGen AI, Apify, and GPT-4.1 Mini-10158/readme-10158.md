Automated News Video Generation with HeyGen AI, Apify, and GPT-4.1 Mini

https://n8nworkflows.xyz/workflows/automated-news-video-generation-with-heygen-ai--apify--and-gpt-4-1-mini-10158


# Automated News Video Generation with HeyGen AI, Apify, and GPT-4.1 Mini

### 1. Workflow Overview

This workflow automates the generation of news recap videos by integrating HeyGen AI video generation, Apify web content crawling, and GPT-4.1 Mini language model for script writing. The primary goal is to fetch the latest news from Morning Brew, generate a concise spoken-style script summarizing the top stories, and create an AI avatar video reading that script.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Manual trigger to initiate the workflow.
- **1.2 News Content Crawling:** Uses Apify to scrape the latest Morning Brew newsletter content.
- **1.3 Script Generation:** Employs GPT-4.1 Mini and an n8n Langchain agent to convert raw news text into a concise, spoken-style video script.
- **1.4 Video Generation with HeyGen:** Calls HeyGen API to generate a video using a chosen avatar and voice, based on the script.
- **1.5 Video Status Polling:** Periodically checks the video generation status until completion.

Supporting nodes provide retrieval of available avatars and voices from HeyGen to aid configuration and selection.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Provides a manual trigger node to start the workflow execution on demand.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Configuration: No parameters, initiates workflow on manual test click  
    - Input: None  
    - Output: Starts the workflow, connected to the `News` node  
    - Failure Modes: None; manual trigger only  
    - Version: 1

---

#### 2.2 News Content Crawling

- **Overview:**  
  Fetches the latest Morning Brew newsletter content using Apify Web Scraper API. It is configured for adaptive Playwright crawling with various filters and settings to extract readable Markdown text.

- **Nodes Involved:**  
  - News

- **Node Details:**  
  - **News**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: Apify run-sync endpoint for website-content-crawler act  
      - Body: JSON with crawler options (Playwright adaptive mode, pruning disabled, click selectors, proxy enabled, filtering for readable content, exclusion of certain HTML elements, saving Markdown output)  
      - Sends JSON body and headers  
      - Authentication: Uses Apify token credential (configured in n8n credentials store)  
    - Expressions: None dynamic; static JSON body  
    - Input: From manual trigger  
    - Output: JSON with scraped Markdown content, specifically the text of the Morning Brew issue  
    - Failures: Possible network errors, API rate limits, or invalid token  
    - Version: 4.2

---

#### 2.3 Script Generation

- **Overview:**  
  Transforms the raw scraped newsletter text into a concise, engaging script optimized for a short video recap using GPT-4.1 Mini and an n8n Langchain agent.

- **Nodes Involved:**  
  - GPT 4.1 Mini  
  - Script Writer

- **Node Details:**  
  - **GPT 4.1 Mini**  
    - Type: Langchain LM Chat OpenRouter Node  
    - Configuration:  
      - Uses OpenRouter API for GPT-4.1 Mini model  
      - No additional parameters set, default chat model call  
    - Input: Connected from `Script Writer` as language model backend  
    - Output: Language model response (script text)  
    - Failure Modes: API key invalid, rate limits, network failures  
    - Version: 1

  - **Script Writer**  
    - Type: Langchain Agent Node  
    - Configuration:  
      - Text input uses expression: `={{ $json.text }}` to pass the scraped newsletter text  
      - System message prompt instructs the AI to generate a concise 10–20 second spoken-style news recap script covering 2–4 top stories, with style guidelines and sample tone  
      - Output: Single paragraph suitable for voice narration  
    - Input: From `News` node (scraped text) and from `GPT 4.1 Mini` as LM backend  
    - Output: Generated script text for video narration  
    - Failure Modes: Expression errors if input text missing; LM API failures  
    - Version: 1.9

---

#### 2.4 Video Generation with HeyGen

- **Overview:**  
  Sends the generated script text to HeyGen’s video generation API, specifying avatar, voice, dimensions, and speech speed to create an AI-driven video.

- **Nodes Involved:**  
  - Generate Video1  
  - Get Video1

- **Node Details:**  
  - **Generate Video1**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: HeyGen `/v2/video/generate` endpoint  
      - JSON Body: Contains video input array with:  
        - Character object specifying avatar type, avatar_id (configurable), and style  
        - Voice object specifying text input (script), voice_id (configurable), and speed (1.1)  
        - Video dimension 1280x720  
      - Sends body and headers with authentication using HeyGen API key credential  
    - Input: From `Script Writer` node’s output script text, dynamically inserted into JSON body  
    - Output: JSON response with video generation job info including video_id  
    - Failure Modes: Authentication errors, invalid avatar/voice IDs, API errors, network issues  
    - Version: 4.2

  - **Get Video1**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET  
      - URL: HeyGen `/v1/video_status.get` endpoint  
      - Query parameter: video_id from the `Generate Video1` node response  
      - Sends headers with authentication  
    - Input: From wait nodes (polling)  
    - Output: Video status JSON including `status` field  
    - Failure Modes: Video job not found, API errors, auth errors  
    - Version: 4.2

---

#### 2.5 Video Status Polling

- **Overview:**  
  Implements polling to check the status of the video generation until it is completed.

- **Nodes Involved:**  
  - 30 Seconds  
  - If  
  - Wait

- **Node Details:**  
  - **30 Seconds**  
    - Type: Wait node  
    - Configuration: Waits 10 seconds before proceeding  
    - Input: From `Generate Video1`  
    - Output: To `Get Video1` to check status  
    - Failure Modes: None (simple delay)  
    - Version: 1.1

  - **If**  
    - Type: Conditional node  
    - Configuration: Checks if `status` field in `Get Video1` response equals `completed`  
    - Input: From `Get Video1`  
    - Output:  
      - True branch: Ends polling  
      - False branch: Connects to `Wait` node to delay before next poll  
    - Failure Modes: Missing or malformed response data  
    - Version: 2.2

  - **Wait**  
    - Type: Wait node  
    - Configuration: Default wait (amount unspecified, likely short delay) before retry  
    - Input: From `If` false branch  
    - Output: Loops back to `Get Video1` for next status check  
    - Failure Modes: None  
    - Version: 1.1

---

#### 2.6 Supporting Nodes: HeyGen Avatars & Voices Retrieval

- **Overview:**  
  These nodes fetch lists of available avatars and voices from HeyGen to assist in selecting valid IDs for video generation.

- **Nodes Involved:**  
  - Get Avatars  
  - Get Voices

- **Node Details:**  
  - **Get Avatars**  
    - Type: HTTP Request  
    - Configuration:  
      - GET request to HeyGen `/v2/avatars` endpoint  
      - Authenticated with HeyGen API key  
      - Returns list of avatar objects including IDs, styles  
    - Input: None (manual or test trigger)  
    - Output: Avatar data for user reference  
    - Failures: Auth errors, network issues  
    - Version: 4.2

  - **Get Voices**  
    - Type: HTTP Request  
    - Configuration:  
      - GET request to HeyGen `/v2/voices` endpoint  
      - Authenticated with HeyGen API key  
      - Returns list of voice profiles including voice IDs  
    - Input: None  
    - Output: Voice data for user reference  
    - Failures: Auth errors, network issues  
    - Version: 4.2

---

#### 2.7 Documentation and Sticky Notes

- **Overview:**  
  Sticky notes provide inline documentation and setup guidance within the workflow canvas for user clarity.

- **Nodes Involved:**  
  - Sticky Note (Get Avatar ID)  
  - Sticky Note1 (Get Voice ID)  
  - Sticky Note2 (Create Video)  
  - Sticky Note3 (Create Video w/ Polling)  
  - Sticky Note4 (Get Video)  
  - Sticky Note5 (Setup Guide)

- **Node Details:**  
  - Sticky notes contain textual instructions such as:  
    - How to obtain avatar and voice IDs from HeyGen  
    - Overview of creating videos and polling for completion  
    - Setup steps including credential configuration links:  
      - HeyGen API key  
      - OpenRouter API key for GPT-4.1 Mini  
      - Apify token for content crawling  
    - Author credit and video tutorial link: [Jadai Kongolo YouTube](https://www.youtube.com/@jadaikongolo)  
  - Positioning is near relevant functional nodes for ease of understanding  
  - No inputs or outputs; purely informational

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role              | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                               |
|-------------------------|--------------------------------|-----------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger               | Workflow start trigger      | None                         | News                        |                                                                                                          |
| News                    | HTTP Request                   | Scrape Morning Brew content | When clicking ‘Test workflow’| Script Writer               |                                                                                                          |
| GPT 4.1 Mini            | Langchain LM Chat OpenRouter  | Language model backend      | Script Writer                | Script Writer               |                                                                                                          |
| Script Writer           | Langchain Agent               | Generate concise video script| News, GPT 4.1 Mini           | Generate Video1             |                                                                                                          |
| Generate Video1         | HTTP Request                   | Request HeyGen video generation| Script Writer               | 30 Seconds                  |                                                                                                          |
| 30 Seconds              | Wait                          | Delay before polling        | Generate Video1              | Get Video1                  |                                                                                                          |
| Get Video1              | HTTP Request                   | Check video generation status| 30 Seconds, Wait            | If                         | Sticky Note4: # Get Video                                                                                 |
| If                      | If                            | Check if video is complete  | Get Video1                   | Wait (if not completed)     |                                                                                                          |
| Wait                    | Wait                          | Pause before next poll      | If (false branch)            | Get Video1                  |                                                                                                          |
| Get Avatars             | HTTP Request                   | Retrieve available avatars  | None                        | None                       | Sticky Note: # Get Avatar ID                                                                              |
| Get Voices              | HTTP Request                   | Retrieve available voices   | None                        | None                       | Sticky Note1: # Get Voice ID                                                                              |
| Generate Video          | HTTP Request                   | Initial video generation (unused in main flow) | None             | Get Video                  | Sticky Note2: # Create Video                                                                              |
| Get Video               | HTTP Request                   | Check video generation status (unused in main flow) | Generate Video       | None                       |                                                                                                          |
| Sticky Note             | Sticky Note                   | Document Get Avatar ID      | None                        | None                       | # Get Avatar ID                                                                                           |
| Sticky Note1            | Sticky Note                   | Document Get Voice ID       | None                        | None                       | # Get Voice ID                                                                                            |
| Sticky Note2            | Sticky Note                   | Document Create Video       | None                        | None                       | # Create Video                                                                                            |
| Sticky Note3            | Sticky Note                   | Document Create Video w/ Polling | None                     | None                       | # Create Video w/ Polling                                                                                  |
| Sticky Note4            | Sticky Note                   | Document Get Video          | None                        | None                       | # Get Video                                                                                                |
| Sticky Note5            | Sticky Note                   | Setup and author guide      | None                        | None                       | # 🛠️Setup Guide  \n## **Author: [Jadai Kongolo](https://www.youtube.com/@jadaikongolo)** \n\nSetup instructions and links |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add **Manual Trigger** node named `When clicking ‘Test workflow’`.

2. **Configure Apify Scraper:**  
   - Add **HTTP Request** node named `News`.  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/apify~website-content-crawler/run-sync-get-dataset-items`  
   - Body Type: JSON  
   - Body: Copy the JSON configuration for Playwright adaptive crawling targeting `https://www.morningbrew.com/issues/latest` with readable Markdown output and proxy enabled.  
   - Connect output of manual trigger to `News` node.  
   - Configure Apify API token credential in n8n and assign to this node.

3. **Add GPT 4.1 Mini Language Model Node:**  
   - Add **Langchain LM Chat OpenRouter** node named `GPT 4.1 Mini`.  
   - Connect output of `Script Writer` node (to be created next) to this node’s AI language model input.  
   - Configure OpenRouter API key credential.

4. **Add Script Writer Node:**  
   - Add **Langchain Agent** node named `Script Writer`.  
   - Set parameter: `text` = `={{ $json.text }}` (this uses the Markdown text from `News`)  
   - Set system prompt (copy from workflow) instructing concise 10–20 second news recap script generation in engaging spoken style.  
   - Connect output of `News` node to this node’s input.  
   - Connect this node’s AI language model input to `GPT 4.1 Mini` node.  
   - Connect output of `Script Writer` to `Generate Video1`.

5. **Configure HeyGen Video Generation Node:**  
   - Add **HTTP Request** node named `Generate Video1`.  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Body Type: JSON  
   - Body: JSON structure specifying:  
     - `video_inputs` with `character` (type `"avatar"`, `avatar_id`, and style `"normal"`)  
     - `voice` (type `"text"`, `input_text` from script, `voice_id`, `speed` 1.1)  
     - `dimension` 1280x720  
   - Use expressions to insert dynamic values for `input_text` from `Script Writer` output and set `avatar_id` and `voice_id` as per your chosen HeyGen assets.  
   - Add HeyGen API key credential with HTTP Header authentication.  
   - Connect output of `Script Writer` to this node.

6. **Add Delay Node for Polling:**  
   - Add **Wait** node named `30 Seconds`.  
   - Set wait time to 10 seconds.  
   - Connect output of `Generate Video1` to `30 Seconds`.

7. **Add Video Status Check Node:**  
   - Add **HTTP Request** node named `Get Video1`.  
   - Method: GET  
   - URL: `https://api.heygen.com/v1/video_status.get`  
   - Add Query Parameter `video_id` set dynamically from `Generate Video1` response: `={{ $('Generate Video1').item.json.data.video_id }}`  
   - Add HeyGen API key credential.  
   - Connect output of `30 Seconds` to `Get Video1`.

8. **Add Conditional Node to Check Video Completion:**  
   - Add **If** node named `If`.  
   - Condition: Check if `{{$json.data.status}}` equals `completed` (string).  
   - Connect output of `Get Video1` to `If`.

9. **Add Wait Node for Re-polling:**  
   - Add **Wait** node named `Wait`.  
   - Use default delay or short delay (e.g., 10 seconds).  
   - Connect false branch of `If` to `Wait`.  
   - Connect output of `Wait` back to `Get Video1` to create polling loop.

10. **Finishing:**  
    - True branch of `If` ends polling and can be connected to further nodes or leave unconnected.

11. **Supporting Nodes (Optional):**  
    - Add HTTP Request nodes `Get Avatars` and `Get Voices` with GET method to HeyGen endpoints `/v2/avatars` and `/v2/voices` respectively, authorized with HeyGen API key, to retrieve available avatar and voice IDs for configuration.

12. **Sticky Notes (Optional):**  
    - Add sticky notes with setup instructions, API key requirements, and author credit as per user preference for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                      | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Setup instructions detail creation of HeyGen avatar and voice, importing ElevenLabs voice clone, and connecting API keys for HeyGen, OpenRouter (GPT), and Apify tokens.                                                                                          | Sticky Note5 and workflow description                                                          |
| Author: Jadai Kongolo YouTube Channel — [https://www.youtube.com/@jadaikongolo](https://www.youtube.com/@jadaikongolo)                                                                                                                                             | Sticky Note5                                                                                   |
| HeyGen API documentation: https://docs.heygen.com/                                                                                                                                                                                                              | For avatar, voice, and video API usage                                                        |
| Apify website content crawler documentation: https://apify.com/website-content-crawler                                                                                                                                                                            | For customizing crawling parameters                                                           |
| OpenRouter AI GPT-4.1 Mini model endpoint: https://openrouter.ai/                                                                                                                                                                                                | For configuring Langchain OpenRouter node                                                     |

---

This documentation provides a thorough, stepwise understanding of the workflow’s structure, nodes, configuration, and practical instructions to replicate or modify it. Potential errors mainly relate to API authentication, network reliability, invalid dynamic expressions, and polling timeouts. Proper credential storage and testing of API connectivity before running the workflow are recommended.