Create AI News Avatar Videos with Dumpling AI, GPT-4o and HeyGen

https://n8nworkflows.xyz/workflows/create-ai-news-avatar-videos-with-dumpling-ai--gpt-4o-and-heygen-5796


# Create AI News Avatar Videos with Dumpling AI, GPT-4o and HeyGen

### 1. Workflow Overview

This workflow automates the creation of short avatar-style videos summarizing the latest news related to AI agents. It is designed to run on an hourly schedule and produces engaging video content by extracting news, generating scripts via AI language models, and synthesizing avatar videos with text-to-speech capabilities.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger:** Initiates the workflow periodically (every hour).
- **1.2 News Retrieval and Processing:** Searches for recent AI news articles, extracts individual items, limits to the top 4, scrapes full article content, and merges these into a single text block.
- **1.3 Script Generation Using AI:** Uses GPT-4o OpenAI model to convert aggregated news content into a concise, natural language video script.
- **1.4 Avatar Video Generation:** Sends the script to HeyGen API to generate an avatar video, waits for processing, polls for completion status, and handles retries if needed.
- **1.5 Logging and Tracking:** Once the video is ready, logs the video URL into a Google Sheet for tracking purposes.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This node triggers the entire workflow execution on a fixed hourly interval, ensuring the process runs automatically.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Role:** Periodic workflow starter  
  - **Configuration:** Runs every hour (default interval with empty object in interval array)  
  - **Input:** None (trigger node)  
  - **Output:** Starts the workflow chain by passing empty data to the next node  
  - **Edge Cases:** Misconfiguration of interval could lead to no or too frequent triggers  

#### 2.2 News Retrieval and Processing

- **Overview:**  
  Searches for recent AI-related news using Dumpling AI’s search API, splits the response into individual news items, limits to top 4 results, scrapes the full article content for each item, and merges the content into a single text for downstream processing.

- **Nodes Involved:**  
  - Dumpling AI: Search AI News  
  - Split: Individual News Items  
  - Limit: Top 4 News Results  
  - Dumpling AI: Scrape Article Content  
  - Combine: Merge Scraped News Content

- **Node Details:**

  1. **Dumpling AI: Search AI News**  
     - **Type:** HTTP Request  
     - **Role:** Queries Dumpling AI API to search news articles on the topic "AI Agent" within the past hour.  
     - **Configuration:**  
       - POST request to `https://app.dumplingai.com/api/v1/search-news`  
       - JSON body contains query parameters including `"query": "AI Agent"`, `"language": "en"`, `"dateRange": "pastHour"`, `"page": "1"`  
       - Authenticated with HTTP header-based credential (Dumpling AI API key)  
     - **Inputs:** Trigger from Schedule Trigger  
     - **Outputs:** JSON with a "news" array  
     - **Edge Cases:** API rate limits, network errors, empty search results  

  2. **Split: Individual News Items**  
     - **Type:** Split Out  
     - **Role:** Splits the array of news articles into individual items for sequential processing  
     - **Configuration:** Splits on the field "news" from prior node output  
     - **Inputs:** Output from Dumpling AI: Search AI News  
     - **Outputs:** One item per news article  
     - **Edge Cases:** Empty array results in no downstream execution  

  3. **Limit: Top 4 News Results**  
     - **Type:** Limit  
     - **Role:** Limits the number of news articles processed to the top 4  
     - **Configuration:** Max items set to 4  
     - **Inputs:** Individual news items from Split node  
     - **Outputs:** Maximum 4 news items  
     - **Edge Cases:** Less than 4 news items available - processes whatever is present  

  4. **Dumpling AI: Scrape Article Content**  
     - **Type:** HTTP Request  
     - **Role:** Scrapes full article content from each news article’s link  
     - **Configuration:**  
       - POST to `https://app.dumplingai.com/api/v1/scrape`  
       - JSON body includes `"url": "{{ $json.link }}"` dynamically set per news item, `"cleaned": "true"` for cleaned content  
       - Authenticated with Dumpling AI API key via HTTP header  
     - **Inputs:** Top 4 individual news items  
     - **Outputs:** Scraped content JSON for each article  
     - **Edge Cases:** Broken URLs, scraping failures, empty content  

  5. **Combine: Merge Scraped News Content**  
     - **Type:** Aggregate  
     - **Role:** Aggregates all scraped article contents into a single concatenated text field  
     - **Configuration:** Aggregates the field "content" from all inputs  
     - **Inputs:** Scraped content items from previous node  
     - **Outputs:** One item with combined content text  
     - **Edge Cases:** Empty content fields, aggregation errors  

#### 2.3 Script Generation Using AI

- **Overview:**  
  Uses the LangChain GPT-4o agent to generate a short, engaging video script from the combined news article content. The script is concise, conversational, and formatted as a single line for easy speech synthesis.

- **Nodes Involved:**  
  - GPT-4o Agent: Write Video Script  
  - GPT-4o Model

- **Node Details:**

  1. **GPT-4o Agent: Write Video Script**  
     - **Type:** LangChain Agent Node  
     - **Role:** Defines the prompt and system message for script generation  
     - **Configuration:**  
       - Input text includes combined news content and the original search query ("AI Agent")  
       - System message instructs generation of a short, clear, conversational video script suitable for 30-60 seconds, without new lines or formatting  
       - Prompt type set to "define" for fixed prompt structure  
     - **Inputs:** Combined article content  
     - **Outputs:** Prompt passed to GPT-4o model node  
     - **Edge Cases:** Expression errors, prompt formatting issues  

  2. **GPT-4o Model**  
     - **Type:** LangChain OpenAI Chat Model  
     - **Role:** Executes the language model call using GPT-4o-mini for chat completion  
     - **Configuration:**  
       - Model selected: "gpt-4o-mini"  
       - Connected to OpenAI API via OAuth2 credential  
     - **Inputs:** Prompt from GPT-4o Agent node  
     - **Outputs:** Generated video script text in a single line  
     - **Edge Cases:** API rate limits, timeouts, auth errors  

#### 2.4 Avatar Video Generation

- **Overview:**  
  Sends the generated script to HeyGen API to create an avatar video with specified dimensions and voice parameters. Waits for the video rendering, polls for completion, and retries if necessary.

- **Nodes Involved:**  
  - HeyGen: Generate Avatar Video  
  - Wait: For HeyGen to Process  
  - HeyGen: Check Video Status  
  - IF: Video Completed?  
  - Wait: Retry if Not Complete

- **Node Details:**

  1. **HeyGen: Generate Avatar Video**  
     - **Type:** HTTP Request  
     - **Role:** Initiates avatar video generation with script text and avatar/voice settings  
     - **Configuration:**  
       - POST to `https://api.heygen.com/v2/video/generate`  
       - JSON body includes:  
         - Video inputs with avatar type, style (normal), and voice parameters (text input from GPT-4o script, voice_id empty, speed 1.1)  
         - Video dimension 1280x720  
       - Authenticated via HTTP header (Bearer token)  
       - Accepts JSON response  
     - **Inputs:** Script text from GPT-4o Model node  
     - **Outputs:** Video generation job data including video_id  
     - **Edge Cases:** API errors, invalid avatar or voice IDs, network failures  

  2. **Wait: For HeyGen to Process**  
     - **Type:** Wait Node (Webhook-enabled)  
     - **Role:** Pauses execution to allow HeyGen video rendering  
     - **Configuration:** No fixed time, waits for webhook/webhook event to proceed  
     - **Inputs:** Video generation job data  
     - **Outputs:** Triggers check video status node  
     - **Edge Cases:** Timeout if video takes too long or webhook not received  

  3. **HeyGen: Check Video Status**  
     - **Type:** HTTP Request  
     - **Role:** Polls HeyGen API to check if video processing is complete  
     - **Configuration:**  
       - GET request to `https://api.heygen.com/v1/video_status.get`  
       - Query parameter `video_id` set dynamically from prior response  
       - Authenticated with HTTP header  
     - **Inputs:** From Wait node or retry wait  
     - **Outputs:** Video status JSON with "status" and video URL if completed  
     - **Edge Cases:** API errors, invalid video_id, network issues  

  4. **IF: Video Completed?**  
     - **Type:** If Condition  
     - **Role:** Checks whether video status equals "completed"  
     - **Configuration:** Condition compares `$json.data.status` to string "completed" (case-sensitive, strict type)  
     - **Inputs:** Video status JSON  
     - **Outputs:**  
       - True branch: video ready → log URL  
       - False branch: not ready → retry wait  
     - **Edge Cases:** Status field missing or unexpected values  

  5. **Wait: Retry if Not Complete**  
     - **Type:** Wait Node (Webhook-enabled)  
     - **Role:** Waits 20 seconds before retrying video status check  
     - **Configuration:** Wait time 20 seconds  
     - **Inputs:** False branch from IF node  
     - **Outputs:** Loops back to HeyGen: Check Video Status node for polling  
     - **Edge Cases:** Infinite loop if video never completes, risk of API rate limiting  

#### 2.5 Logging and Tracking

- **Overview:**  
  Logs the final generated avatar video URL into a Google Sheets spreadsheet to track all generated videos.

- **Nodes Involved:**  
  - Google Sheets: Log Video URL

- **Node Details:**  
  - **Type:** Google Sheets node  
  - **Role:** Appends a new row with the video URL to a specified sheet  
  - **Configuration:**  
    - Operation: Append  
    - Document ID: Google Sheets spreadsheet ID (configured in credentials)  
    - Sheet Name: gid=0 (Sheet1)  
    - Columns mapped: "Video link" set to `{{ $json.data.video_url }}`  
    - Credentials: Google Sheets OAuth2  
  - **Inputs:** True branch output from IF node (video completed)  
  - **Outputs:** None (terminal logging)  
  - **Edge Cases:** Google API authentication issues, quota limits, spreadsheet missing or renamed  

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                       | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                       |
|-------------------------------|--------------------------------|-------------------------------------|---------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger               | Starts workflow every hour           | None                      | Dumpling AI: Search AI News    | 🎥 Workflow Overview: Auto-Generate AI News Avatar Videos. Runs hourly to create avatar videos. |
| Dumpling AI: Search AI News   | HTTP Request                  | Searches for recent AI news          | Schedule Trigger          | Split: Individual News Items   | See above                                                                                       |
| Split: Individual News Items  | Split Out                    | Splits news array into single items | Dumpling AI: Search AI News | Limit: Top 4 News Results       |                                                                                                 |
| Limit: Top 4 News Results      | Limit                        | Limits to top 4 news articles        | Split: Individual News Items | Dumpling AI: Scrape Article Content |                                                                                                 |
| Dumpling AI: Scrape Article Content | HTTP Request              | Scrapes full article content         | Limit: Top 4 News Results | Combine: Merge Scraped News Content |                                                                                                 |
| Combine: Merge Scraped News Content | Aggregate                 | Merges scraped content into one text | Dumpling AI: Scrape Article Content | GPT-4o Agent: Write Video Script |                                                                                                 |
| GPT-4o Agent: Write Video Script | LangChain Agent            | Creates video script prompt          | Combine: Merge Scraped News Content | HeyGen: Generate Avatar Video   |                                                                                                 |
| GPT-4o Model                  | LangChain OpenAI Chat Model    | Generates video script text          | GPT-4o Agent: Write Video Script | HeyGen: Generate Avatar Video   |                                                                                                 |
| HeyGen: Generate Avatar Video | HTTP Request                  | Sends script to HeyGen for video     | GPT-4o Model             | Wait: For HeyGen to Process     |                                                                                                 |
| Wait: For HeyGen to Process   | Wait                         | Waits for video rendering            | HeyGen: Generate Avatar Video | HeyGen: Check Video Status       |                                                                                                 |
| HeyGen: Check Video Status    | HTTP Request                  | Polls HeyGen video status            | Wait: For HeyGen to Process, Wait: Retry if Not Complete | IF: Video Completed?            |                                                                                                 |
| IF: Video Completed?          | If                           | Checks if video is completed         | HeyGen: Check Video Status | Google Sheets: Log Video URL (true branch), Wait: Retry if Not Complete (false branch) |                                                                                                 |
| Wait: Retry if Not Complete   | Wait                         | Waits 20 seconds before retrying     | IF: Video Completed? (false branch) | HeyGen: Check Video Status        |                                                                                                 |
| Google Sheets: Log Video URL  | Google Sheets                 | Logs video URL into spreadsheet      | IF: Video Completed? (true branch) | None                          |                                                                                                 |
| Sticky Note2                  | Sticky Note                  | Overview and instructions            | None                      | None                          | See Workflow Overview section above                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Add Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run every hour (default interval with empty object)  
   - No credentials needed  
   - Connect output to next node  

2. **Add HTTP Request Node: Dumpling AI Search**  
   - Name: Dumpling AI: Search AI News  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/search-news`  
   - Authentication: HTTP Header Auth with Dumpling AI API key credential  
   - Body (JSON):  
     ```json
     {
       "query": "AI Agent",
       "language": "en",
       "dateRange": "pastHour",
       "page": "1"
     }
     ```  
   - Connect Schedule Trigger output to this node  

3. **Add Split Out Node**  
   - Name: Split: Individual News Items  
   - Field to split out: `news`  
   - Connect output of Dumpling AI Search to this node  

4. **Add Limit Node**  
   - Name: Limit: Top 4 News Results  
   - Max items: 4  
   - Connect output of Split node to this node  

5. **Add HTTP Request Node: Dumpling AI Scrape Content**  
   - Name: Dumpling AI: Scrape Article Content  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/scrape`  
   - Authentication: HTTP Header Auth (same Dumpling AI API key)  
   - Body (JSON):  
     ```json
     {
       "url": "{{ $json.link }}",
       "cleaned": "true"
     }
     ```  
   - Connect output of Limit node here  

6. **Add Aggregate Node**  
   - Name: Combine: Merge Scraped News Content  
   - Aggregate field: `content` (concatenate all scraped contents)  
   - Connect output of Scrape Article Content node here  

7. **Add LangChain Agent Node: GPT-4o Agent**  
   - Name: GPT-4o Agent: Write Video Script  
   - Set prompt text to:  
     ```
     Here is the topic:{{ $json.content }}

     Here is the news article:{{ $('Dumpling AI: Search AI News').item.json.searchParameters.q }}
     ```  
   - System message:  
     ```
     You are a creative content writer. I will give you a news article and the intended topic or angle I want to focus on. Your job is to turn that into a short, engaging script suitable for a 30 to 60-second video. Write in a natural, conversational tone that sounds like someone talking to a general audience. Keep it simple, clear, and focused on the most interesting or important angle based on the topic I provide. Avoid technical jargon. The goal is to grab attention and make the message easy to understand and relatable. Very important: Output the final script as a single line only — no new lines, no paragraph breaks, no titles or formatting. Just plain text in one continuous sentence.
     ```  
   - Connect output of Combine node here  

8. **Add LangChain OpenAI Chat Model Node**  
   - Name: GPT-4o Model  
   - Model: `gpt-4o-mini`  
   - Credentials: OpenAI API OAuth2 credential  
   - Connect output of GPT-4o Agent node here  

9. **Add HTTP Request Node: HeyGen Generate Avatar Video**  
   - Name: HeyGen: Generate Avatar Video  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Authentication: HTTP Header Auth with HeyGen API key  
   - Headers: Accept: application/json  
   - Body (JSON):  
     ```json
     {
       "video_inputs": [
         {
           "character": {
             "type": "avatar",
             "avatar_id": "",
             "avatar_style": "normal"
           },
           "voice": {
             "type": "text",
             "input_text": "{{ $json.output }}",
             "voice_id": "",
             "speed": 1.1
           }
         }
       ],
       "dimension": {
         "width": 1280,
         "height": 720
       }
     }
     ```  
   - Connect output of GPT-4o Model node here  

10. **Add Wait Node: For HeyGen to Process**  
    - Name: Wait: For HeyGen to Process  
    - Enable Webhook to pause until triggered  
    - Connect output of HeyGen Generate Avatar Video node here  

11. **Add HTTP Request Node: HeyGen Check Video Status**  
    - Name: HeyGen: Check Video Status  
    - Method: GET  
    - URL: `https://api.heygen.com/v1/video_status.get`  
    - Authentication: HTTP Header Auth with HeyGen API key  
    - Query Parameters: `video_id={{ $json.data.video_id }}`  
    - Connect output of Wait node here  

12. **Add If Node: Video Completed?**  
    - Name: IF: Video Completed?  
    - Condition: `$json.data.status` equals `completed` (case-sensitive, strict)  
    - Connect output of HeyGen Check Video Status node here  

13. **Add Wait Node: Retry if Not Complete**  
    - Name: Wait: Retry if Not Complete  
    - Wait time: 20 seconds  
    - Connect false branch of IF node here  
    - Connect output of this node back to HeyGen Check Video Status node (loop)  

14. **Add Google Sheets Node**  
    - Name: Google Sheets: Log Video URL  
    - Operation: Append  
    - Document ID: your target Google Sheet ID  
    - Sheet Name: gid=0 (Sheet1)  
    - Columns:  
      - Video link → `{{ $json.data.video_url }}`  
    - Credentials: Google Sheets OAuth2  
    - Connect true branch of IF node here  

15. **Optional: Add Sticky Note**  
    - Add a sticky note with workflow overview and usage instructions for future reference  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow integrates Dumpling AI, OpenAI GPT-4o, HeyGen, and Google Sheets to automate avatar video generation from AI news.                                                                                             | Workflow description                                                                                |
| Customize the search query in the Dumpling AI Search node to target different topics beyond "AI Agent."                                                                                                                       | Node configuration                                                                                  |
| Ensure all API credentials (Dumpling AI, HeyGen, OpenAI, Google Sheets) are properly configured with required scopes and permissions to avoid authentication errors.                                                           | Credentials setup                                                                                   |
| Use the HeyGen API’s avatar_id and voice_id fields to customize the avatar and voice styles to fit branding needs.                                                                                                            | HeyGen: Generate Avatar Video node                                                                 |
| The workflow includes retry logic for video generation polling to handle asynchronous video rendering by HeyGen. Adjust wait times or add max retry limits to optimize performance and avoid infinite loops.                  | Wait and IF nodes                                                                                   |
| Google Sheets is used for simple tracking; consider extending for error logging or workflow monitoring.                                                                                                                       | Logging and tracking                                                                                |
| Dumpling AI API documentation: https://app.dumplingai.com/docs                                                                                                                                                                | Dumpling AI API docs                                                                                |
| OpenAI GPT-4o API details: https://platform.openai.com/docs/models/gpt-4o                                                                                                                                                      | OpenAI API docs                                                                                     |
| HeyGen API reference: https://heygen.com/developer-api                                                                                                                                                                        | HeyGen API docs                                                                                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow, fully compliant with content policies, and contains no illegal or protected content. All data processed is legal and publicly available.