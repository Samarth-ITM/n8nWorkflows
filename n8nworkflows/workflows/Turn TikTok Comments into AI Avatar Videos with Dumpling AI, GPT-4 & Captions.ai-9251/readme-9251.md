Turn TikTok Comments into AI Avatar Videos with Dumpling AI, GPT-4 & Captions.ai

https://n8nworkflows.xyz/workflows/turn-tiktok-comments-into-ai-avatar-videos-with-dumpling-ai--gpt-4---captions-ai-9251


# Turn TikTok Comments into AI Avatar Videos with Dumpling AI, GPT-4 & Captions.ai

### 1. Workflow Overview

This workflow automates the creation of engaging AI avatar videos from TikTok comments and video transcripts by combining multiple AI and video processing services. It is designed for content creators and social media marketers seeking to transform TikTok interactions into polished, shareable video content with minimal manual effort.

The workflow is logically divided into three main blocks:

- **1.1 Scheduled Input and Data Retrieval**: Periodically triggers the workflow, retrieves TikTok videos and their top comments from a DataTable.
- **1.2 AI Content Generation and Video Creation**: Uses Dumpling AI to transcribe videos, GPT-4 to generate engaging TikTok scripts from transcripts and comments, then submits scripts to Captions.ai to create avatar videos, with subsequent status polling.
- **1.3 Post-Processing and Finalization**: Upon successful video creation, sends the video to Submagic for auto-editing enhancements, and via webhook receives the final video details to store them in Airtable for tracking and review.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Input and Data Retrieval

**Overview:**  
This block runs on a schedule to pull TikTok video URLs and top comments from a configured DataTable. It then obtains a transcript for the video using Dumpling AI, preparing data for script generation.

**Nodes Involved:**  
- Trigger on Schedule  
- Get TikTok Video & Comments from DataTable  
- Get Transcript from Dumpling AI  
- Generate TikTok Script with GPT-4  
- Clean Up Script Formatting  
- Sticky Note (for documentation)

**Node Details:**

- **Trigger on Schedule**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow periodically (default interval every minute, can be customized).  
  - Parameters: Uses default scheduling with an empty interval (runs every minute).  
  - Connections: Outputs to "Get TikTok Video & Comments from DataTable".  
  - Edge Cases: Scheduling misconfiguration or downtime could halt workflow triggering.

- **Get TikTok Video & Comments from DataTable**  
  - Type: DataTable Node  
  - Role: Retrieves TikTok video URLs and associated top comments (under the "Keywords" field) from a specific DataTable filtered by non-empty keywords.  
  - Parameters: Filters rows where the "Keywords" field is not empty; DataTable ID points to "Tik Tok Keywords".  
  - Connections: Outputs to "Get Transcript from Dumpling AI".  
  - Edge Cases: Empty results if no keywords present; DataTable connectivity or permission issues.

- **Get Transcript from Dumpling AI**  
  - Type: HTTP Request  
  - Role: Sends the TikTok video URL to Dumpling AI API to fetch the transcript.  
  - Parameters: POST request with body containing "videoUrl" from input JSON and "requestSource" set to "API".  
  - Authentication: HTTP Header Auth using Dumpling AI credentials.  
  - Connections: Outputs to "Generate TikTok Script with GPT-4".  
  - Edge Cases: API rate limits, invalid video URLs, authentication errors, network timeouts.

- **Generate TikTok Script with GPT-4**  
  - Type: OpenAI (LangChain) Node  
  - Role: Uses GPT-4.1 to create an engaging TikTok video script from the Dumpling AI transcript and the top TikTok comment.  
  - Configuration:  
    - Model: GPT-4.1  
    - Messages: System prompt instructs the model to write an engaging, conversational script based on the transcript and comment.  
    - Inputs: Transcript text and top comment keyword, dynamically inserted via expressions.  
  - Authentication: OpenAI API credentials.  
  - Connections: Outputs to "Clean Up Script Formatting".  
  - Edge Cases: API usage limits, malformed input data, expression evaluation errors, model response delays.

- **Clean Up Script Formatting**  
  - Type: Code Node (JavaScript)  
  - Role: Removes new lines and extraneous spaces from the AI-generated script for clean formatting.  
  - Code Logic: Replaces all newline characters with spaces, collapses multiple spaces, trims leading/trailing spaces.  
  - Connections: Outputs to "Captions: Generate Avatar Video".  
  - Edge Cases: Unexpected input data structure; script content missing or malformed.

- **Sticky Note**  
  - Purpose: Documents the process steps and intent of this block for human users.

---

#### 2.2 AI Content Generation and Video Creation

**Overview:**  
This block creates an AI avatar video based on the cleaned TikTok script using Captions.ai, waits for processing completion, polls for video creation status, and handles retries or continuation once the video is ready.

**Nodes Involved:**  
- Captions: Generate Avatar Video  
- Wait: For HeyGen to Process  
- Caption: Check Video Status  
- Was the Captions.ai Video Created Successfully? (If node)  
- Retry Delay Before Rechecking Status  
- Send Final Video to Submagic for Enhancements  
- Sticky Note1 (for documentation)

**Node Details:**

- **Captions: Generate Avatar Video**  
  - Type: HTTP Request  
  - Role: Submits the cleaned TikTok script to Captions.ai API to create an avatar video.  
  - Parameters:  
    - URL: `https://api.captions.ai/api/creator/submit`  
    - Method: POST  
    - Body: JSON with fields "script" (from cleaned script) and "creatorName" set to "twin-3-Blessed".  
  - Authentication: HTTP Header Auth with Captions.ai credentials.  
  - Connections: Outputs to "Wait: For HeyGen to Process".  
  - Edge Cases: API errors, invalid script content, authentication failure.

- **Wait: For HeyGen to Process**  
  - Type: Wait Node  
  - Role: Pauses workflow for 80 seconds to allow Captions.ai processing.  
  - Parameters: Wait for 80 seconds.  
  - Connections: Outputs to "Caption: Check Video Status".  
  - Edge Cases: Insufficient wait time causing premature status check.

- **Caption: Check Video Status**  
  - Type: HTTP Request  
  - Role: Polls Captions.ai API to check the status of the video creation operation.  
  - Parameters: POST to `https://api.captions.ai/api/creator/poll` with JSON body containing the operationId from the previous step.  
  - Authentication: HTTP Header Auth with Captions.ai credentials.  
  - Connections: Outputs to "Was the Captions.ai Video Created Successfully?".  
  - Edge Cases: API errors, missing operationId, network issues.

- **Was the Captions.ai Video Created Successfully?**  
  - Type: If Node  
  - Role: Branches workflow based on whether Captions.ai reports the video state as "COMPLETE".  
  - Condition: Checks if `$json.state === "COMPLETE"`.  
  - Connections:  
    - If true: to "Send Final Video to Submagic for Enhancements".  
    - If false: to "Retry Delay Before Rechecking Status".  
  - Edge Cases: Unexpected state values, missing state field.

- **Retry Delay Before Rechecking Status**  
  - Type: Wait Node  
  - Role: Delays for 3 minutes before retrying video status check.  
  - Parameters: Wait for 3 minutes.  
  - Connections: Outputs back to "Caption: Check Video Status".  
  - Edge Cases: Prolonged delays, possible infinite loops if video never completes.

- **Send Final Video to Submagic for Enhancements**  
  - Type: HTTP Request  
  - Role: Sends the completed Captions.ai video URL to Submagic API for auto-editing enhancements like subtitles and B-roll.  
  - Parameters: POST to `https://api.submagic.co/v1/projects` with JSON body containing the video URL, title (current date), language, webhook URL for callback, and enhancement options (magicZooms, magicBrolls with 75% intensity).  
  - Authentication: HTTP Header Auth with Captions.ai credentials (reused).  
  - Connections: No direct output; Submagic calls webhook when processing is done.  
  - Edge Cases: API authentication errors, invalid video URLs, webhook URL misconfigurations.

- **Sticky Note1**  
  - Purpose: Documents the steps and logic of this block for clarity.

---

#### 2.3 Post-Processing and Finalization

**Overview:**  
The final block receives the enhanced video details from Submagic via webhook, then records these details in Airtable for storage and future reference.

**Nodes Involved:**  
- Submagic Webhook (Video Ready)  
- Save Final Video Details to Airtable  
- Sticky Note2 (for documentation)

**Node Details:**

- **Submagic Webhook (Video Ready)**  
  - Type: Webhook Node  
  - Role: Receives POST callbacks from Submagic when the final enhanced video is ready.  
  - Parameters: Path set to a unique webhook ID string; listens for POST requests.  
  - Connections: Outputs to "Save Final Video Details to Airtable".  
  - Edge Cases: Webhook URL exposure risks, missed webhook calls if n8n is down.

- **Save Final Video Details to Airtable**  
  - Type: Airtable Node  
  - Role: Creates a new record in Airtable with the final video's URL and ID.  
  - Parameters:  
    - Base: "AI videos" Airtable base.  
    - Table: "Videos" table.  
    - Fields mapped: "Video URL" set to `{{$json.body.downloadUrl}}`, "Caption Video ID" set to `{{$json.body.projectId}}`.  
    - Includes option field "Approved?" with "Approved"/"Not Approved" options, default not set here.  
  - Authentication: Airtable Personal Access Token credentials.  
  - Connections: None (end of branch).  
  - Edge Cases: Airtable API limits, field mapping errors, missing data in webhook payload.

- **Sticky Note2**  
  - Purpose: Describes the webhook's role and Airtable saving actions.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                               | Input Node(s)                      | Output Node(s)                                | Sticky Note                           |
|-----------------------------------|----------------------------|-----------------------------------------------|----------------------------------|-----------------------------------------------|-------------------------------------|
| Trigger on Schedule                | Schedule Trigger           | Initiates workflow periodically                 | -                                | Get TikTok Video & Comments from DataTable    | ## 🎬 Create TikTok Script from Comment & Transcript ... |
| Get TikTok Video & Comments from DataTable | DataTable                 | Retrieves TikTok videos and top comments        | Trigger on Schedule               | Get Transcript from Dumpling AI                | ## 🎬 Create TikTok Script from Comment & Transcript ... |
| Get Transcript from Dumpling AI   | HTTP Request               | Gets video transcript via Dumpling AI API       | Get TikTok Video & Comments from DataTable | Generate TikTok Script with GPT-4               | ## 🎬 Create TikTok Script from Comment & Transcript ... |
| Generate TikTok Script with GPT-4 | OpenAI (LangChain)         | Generates engaging TikTok script from transcript and comment | Get Transcript from Dumpling AI  | Clean Up Script Formatting                      | ## 🎬 Create TikTok Script from Comment & Transcript ... |
| Clean Up Script Formatting        | Code (JavaScript)          | Cleans GPT-4 script formatting                   | Generate TikTok Script with GPT-4 | Captions: Generate Avatar Video                 | ## 🎬 Create TikTok Script from Comment & Transcript ... |
| Captions: Generate Avatar Video   | HTTP Request               | Sends script to Captions.ai to create avatar video | Clean Up Script Formatting       | Wait: For HeyGen to Process                     | ## 🤖 Create and Save AI Avatar Video with Captions.ai & Submagic ... |
| Wait: For HeyGen to Process       | Wait                       | Waits 80 seconds for Captions.ai video processing | Captions: Generate Avatar Video  | Caption: Check Video Status                      | ## 🤖 Create and Save AI Avatar Video with Captions.ai & Submagic ... |
| Caption: Check Video Status       | HTTP Request               | Polls Captions.ai to check video creation status | Wait: For HeyGen to Process, Retry Delay Before Rechecking Status | Was the Captions.ai Video Created Successfully? | ## 🤖 Create and Save AI Avatar Video with Captions.ai & Submagic ... |
| Was the Captions.ai Video Created Successfully? | If                         | Branches based on video creation success         | Caption: Check Video Status       | Send Final Video to Submagic / Retry Delay Before Rechecking Status | ## 🤖 Create and Save AI Avatar Video with Captions.ai & Submagic ... |
| Retry Delay Before Rechecking Status | Wait                     | Waits 3 minutes before retrying status check      | Was the Captions.ai Video Created Successfully? (False branch) | Caption: Check Video Status                      | ## 🤖 Create and Save AI Avatar Video with Captions.ai & Submagic ... |
| Send Final Video to Submagic for Enhancements | HTTP Request             | Sends completed video to Submagic for editing    | Was the Captions.ai Video Created Successfully? (True branch) | -                                               | ## 🤖 Create and Save AI Avatar Video with Captions.ai & Submagic ... |
| Submagic Webhook (Video Ready)    | Webhook                    | Receives final enhanced video details from Submagic | -                                | Save Final Video Details to Airtable            | ## 📥 Webhook Branch – Save Final Video from Submagic ... |
| Save Final Video Details to Airtable | Airtable                  | Saves final video URL and ID to Airtable          | Submagic Webhook (Video Ready)   | -                                               | ## 📥 Webhook Branch – Save Final Video from Submagic ... |
| Sticky Note                       | Sticky Note                | Documentation of TikTok script creation block     | -                                | -                                               | ## 🎬 Create TikTok Script from Comment & Transcript ... |
| Sticky Note1                      | Sticky Note                | Documentation of AI avatar video creation block   | -                                | -                                               | ## 🤖 Create and Save AI Avatar Video with Captions.ai & Submagic ... |
| Sticky Note2                      | Sticky Note                | Documentation of webhook and Airtable saving block | -                                | -                                               | ## 📥 Webhook Branch – Save Final Video from Submagic ... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Parameters: Set interval to desired frequency (default every minute)  
   - No credentials needed.

2. **Add a DataTable node ("Get TikTok Video & Comments from DataTable")**  
   - Operation: Get  
   - DataTable ID: Use the TikTok Keywords table ID or your equivalent  
   - Filter: Condition on "Keywords" field to be not empty  
   - Connect Schedule Trigger output to this node input.

3. **Add an HTTP Request node ("Get Transcript from Dumpling AI")**  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/get-tiktok-transcript`  
   - Body Type: JSON (sendBody: true) with parameters:  
     - `videoUrl`: Expression from `{{$json.Videos}}`  
     - `requestSource`: "API"  
   - Authentication: HTTP Header Auth with Dumpling AI API key credentials  
   - Connect DataTable output to this node.

4. **Add OpenAI LangChain node ("Generate TikTok Script with GPT-4")**  
   - Model: GPT-4.1  
   - Messages:  
     - System prompt instructing to write an engaging TikTok script based on transcript and top comment  
     - User prompt with transcript and comment dynamically inserted from prior nodes (`$json.transcript` and DataTable Keywords)  
   - Credentials: OpenAI API Key  
   - Connect Dumpling AI output to this node.

5. **Add Code Node ("Clean Up Script Formatting")**  
   - Language: JavaScript  
   - Paste provided code snippet that removes new lines and excess spaces from the script text  
   - Connect GPT-4 output to this node.

6. **Add HTTP Request node ("Captions: Generate Avatar Video")**  
   - Method: POST  
   - URL: `https://api.captions.ai/api/creator/submit`  
   - Body JSON:  
     ```json
     {
       "script": "{{ $json.message.content }}",
       "creatorName": "twin-3-Blessed"
     }
     ```  
   - Headers: Content-Type `application/json`  
   - Authentication: HTTP Header Auth with Captions.ai credentials  
   - Connect Code node output to this node.

7. **Add Wait node ("Wait: For HeyGen to Process")**  
   - Wait Duration: 80 seconds  
   - Connect Captions.ai submit node output to this node.

8. **Add HTTP Request node ("Caption: Check Video Status")**  
   - Method: POST  
   - URL: `https://api.captions.ai/api/creator/poll`  
   - Body JSON:  
     ```json
     {
       "operationId": "{{ $('Captions: Generate Avatar Video').item.json.operationId }}"
     }
     ```  
   - Authentication: HTTP Header Auth with Captions.ai credentials  
   - Connect Wait node output to this node.

9. **Add If node ("Was the Captions.ai Video Created Successfully?")**  
   - Condition: `$json.state === "COMPLETE"`  
   - Connect Caption Check output to this node.

10. **Add Wait node ("Retry Delay Before Rechecking Status")**  
    - Wait Duration: 3 minutes  
    - Connect False branch of If node to this Wait node.

11. **Link Wait node ("Retry Delay Before Rechecking Status") back to "Caption: Check Video Status"**  
    - To re-poll status after delay.

12. **Add HTTP Request node ("Send Final Video to Submagic for Enhancements")**  
    - Method: POST  
    - URL: `https://api.submagic.co/v1/projects`  
    - Body JSON:  
      ```json
      {
        "title": "AI Avatar video {{$now.format('yyyy-MM-dd')}}",
        "language": "en",
        "videoUrl": "{{ $json.url }}",
        "webhookUrl": "https://[your-n8n-instance]/webhook/[unique-path]",
        "dictionary": ["Submagic", "AI-powered", "captions"],
        "magicZooms": true,
        "magicBrolls": true,
        "magicBrollsPercentage": 75
      }
      ```  
    - Authentication: HTTP Header Auth with Captions.ai credentials  
    - Connect True branch of If node to this node.

13. **Add Webhook node ("Submagic Webhook (Video Ready)")**  
    - HTTP Method: POST  
    - Path: Unique endpoint path (must match webhookUrl in Submagic POST)  
    - No credentials needed  
    - This node waits for Submagic callback.

14. **Add Airtable node ("Save Final Video Details to Airtable")**  
    - Operation: Create  
    - Base: Your Airtable base for AI videos  
    - Table: Videos table  
    - Field mappings:  
      - "Video URL": `{{$json.body.downloadUrl}}`  
      - "Caption Video ID": `{{$json.body.projectId}}`  
    - Credentials: Airtable Personal Access Token  
    - Connect webhook output to this node.

15. **Add Sticky Notes to document each block clearly for workflow maintainers.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses GPT-4.1 via LangChain integration to generate natural, engaging TikTok scripts from comments.| GPT-4 model version "gpt-4.1" specified in OpenAI node.                                         |
| Captions.ai API powers avatar video generation; processing typically takes ~80 seconds before polling status.  | See Captions.ai API docs for operationId usage and limits.                                      |
| Submagic API automates video enhancement with subtitles, B-roll, and other effects; final video is sent back via webhook. | Requires correct webhook URL setup to receive callbacks reliably.                               |
| Airtable acts as a persistent store to track final videos and metadata, enabling review and approvals.         | Airtable Personal Access Token must have appropriate create permissions on the target base/table.|
| The workflow depends on stable API credentials and network connectivity for Dumpling AI, OpenAI, Captions.ai, Submagic, and Airtable. | Ensure credentials are valid and quotas are sufficient to avoid interruptions.                   |
| Sticky notes in the workflow provide detailed documentation of each block, improving maintainability and handoff. | Recommended to keep notes updated with workflow changes.                                       |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. All data and API integrations comply with relevant content policies and contain no illegal or offensive material.