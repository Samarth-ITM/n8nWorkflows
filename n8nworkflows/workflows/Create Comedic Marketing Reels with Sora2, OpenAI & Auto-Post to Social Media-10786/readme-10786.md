Create Comedic Marketing Reels with Sora2, OpenAI & Auto-Post to Social Media

https://n8nworkflows.xyz/workflows/create-comedic-marketing-reels-with-sora2--openai---auto-post-to-social-media-10786


# Create Comedic Marketing Reels with Sora2, OpenAI & Auto-Post to Social Media

### 1. Workflow Overview

This workflow automates the creation and publishing of comedic marketing reels to social media platforms using AI and automation tools. It targets local businesses like coffee shops aiming to produce engaging short videos with humor and natural dialogue, enhancing brand presence on Instagram and TikTok without manual video scripting or posting.

The workflow is logically divided into these blocks:

- **1.1 Schedule Trigger:** Initiates the workflow on a daily schedule at a fixed time.
- **1.2 AI Video Prompt Generation:** Uses an AI agent to generate a cinematic, ultra-realistic 12-second video prompt tailored to promote a coffee shop.
- **1.3 Prompt Logging:** Stores each generated video prompt into a data table for tracking and reuse.
- **1.4 Video Generation with Sora2:** Sends the prompt to the Wavespeed Sora2 API to create a vertical video, then polls the API until the video is ready.
- **1.5 Media Upload to Blotato:** Uploads the completed video to Blotato, a social media media management platform.
- **1.6 Caption Generation:** Uses OpenAI to create a social media caption with relevant hashtags for Instagram or TikTok.
- **1.7 Social Media Post Creation:** Publishes the video and caption automatically to the connected social accounts via Blotato.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
Triggers the workflow daily at 7 PM, automatically starting the content creation cycle.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `n8n-nodes-base.scheduleTrigger`  
    - Configuration: Fires daily at 19:00 hours (7 PM)  
    - Inputs: None (entry trigger)  
    - Outputs: Initiates the "Video Prompt Agent" node  
    - Edge Cases: Workflow won't trigger if n8n is offline or scheduler misconfigured; time zone considerations may affect firing time.

#### 1.2 AI Video Prompt Generation

- **Overview:**  
Generates a vivid, realistic, and humorous text prompt for a 12-second video promoting Sally’s Coffee, using an OpenAI-powered LangChain agent.

- **Nodes Involved:**  
  - Video Prompt Agent  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**  
  - **Video Prompt Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Text-to-Video prompt generator using LangChain agent architecture  
    - Configuration:  
      - System message instructs the agent to create realistic, cinematic, humorous, and natural dialogue-based video prompts (12 seconds length) promoting Sally's Coffee.  
      - Output: Final text prompt only, no annotations or formatting.  
    - Inputs: Triggered by Schedule Trigger; uses AI memory and OpenAI chat model nodes as subcomponents.  
    - Outputs: A single text string representing the video prompt.  
    - Edge Cases: API failures, timeout, or unexpected output formatting; prompt may require tuning to avoid forced slogans or surreal content.

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides the language model backend (GPT-5) for the Video Prompt Agent.  
    - Configuration: Model set to "gpt-5" with extended timeout (6000000 ms).  
    - Connections: Supplies AI language model input to Video Prompt Agent.  
    - Edge Cases: Authentication errors, rate limits, or model unavailability.

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains session context for the LangChain agent using execution ID as session key.  
    - Connections: Connected to Video Prompt Agent’s AI memory input.  
    - Edge Cases: Memory overflow or incorrect session key usage.

#### 1.3 Prompt Logging

- **Overview:**  
Logs every generated video prompt into an n8n Data Table for record-keeping and future reference.

- **Nodes Involved:**  
  - Insert row

- **Node Details:**  
  - **Insert row**  
    - Type: `n8n-nodes-base.dataTable`  
    - Role: Inserts each generated video prompt text into a predefined data table named "Demo Sally's Coffee".  
    - Configuration: Defines a single column "Video_Prompt" mapped to the Video Prompt Agent’s output.  
    - Inputs: From Video Prompt Agent node  
    - Outputs: Triggers the Sora2 POST Request node  
    - Edge Cases: DataTable connectivity or permission errors; data type mismatches.

#### 1.4 Video Generation with Sora2

- **Overview:**  
Sends the prompt to Sora2 API to generate a 12-second vertical video, then waits and polls the API until the video creation is complete.

- **Nodes Involved:**  
  - Sora2 POST Request  
  - Wait 30 Secs  
  - GET Sora2 Result  
  - If  
  - Wait Another 30 Secs

- **Node Details:**  
  - **Sora2 POST Request**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Sends HTTP POST request to Wavespeed Sora2 API endpoint to start video generation.  
    - Configuration:  
      - URL: `https://api.wavespeed.ai/api/v3/openai/sora-2/text-to-video`  
      - Body includes duration (12 seconds), prompt text, and video size (720*1280)  
      - Uses HTTP header authentication via generic credential  
    - Inputs: From Insert row node  
    - Outputs: To Wait 30 Secs node  
    - Edge Cases: API auth failures, invalid prompt text, network timeouts.

  - **Wait 30 Secs**  
    - Type: `n8n-nodes-base.wait`  
    - Role: Pauses workflow for 30 seconds before polling for results  
    - Inputs: From Sora2 POST Request  
    - Outputs: To GET Sora2 Result node  

  - **GET Sora2 Result**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Polls the Sora2 API prediction endpoint to check video status.  
    - Configuration: Uses dynamic URL with prediction ID obtained from prior POST response.  
    - Inputs: From Wait 30 Secs or Wait Another 30 Secs nodes  
    - Outputs: To If node  
    - Edge Cases: API errors, wrong prediction ID, network issues.

  - **If**  
    - Type: `n8n-nodes-base.if`  
    - Role: Checks if video generation status equals "completed".  
    - Inputs: From GET Sora2 Result node  
    - Outputs:  
      - True: proceeds to Upload media node  
      - False: loops back to Wait Another 30 Secs node  
    - Edge Cases: Status field missing or unexpected values.

  - **Wait Another 30 Secs**  
    - Type: `n8n-nodes-base.wait`  
    - Role: Waits an additional 30 seconds before retrying GET Sora2 Result.  
    - Inputs: From If node (when status not completed)  
    - Outputs: Back to GET Sora2 Result

#### 1.5 Media Upload to Blotato

- **Overview:**  
Uploads the finalized video to Blotato for managing and posting to social media accounts.

- **Nodes Involved:**  
  - Upload media

- **Node Details:**  
  - **Upload media**  
    - Type: `@blotato/n8n-nodes-blotato.blotato`  
    - Role: Uploads media file using the URL from Sora2 completed video output.  
    - Configuration: Uses "media" resource type in Blotato.  
    - Inputs: From If node’s true branch (video ready)  
    - Outputs: To Caption Generator node  
    - Edge Cases: Upload failures, invalid media URL, API authentication errors.

#### 1.6 Caption Generation

- **Overview:**  
Generates a catchy and hashtag-rich caption for Instagram or TikTok posts promoting Sally’s Coffee.

- **Nodes Involved:**  
  - Caption Generator

- **Node Details:**  
  - **Caption Generator**  
    - Type: `@n8n/n8n-nodes-langchain.openAi`  
    - Role: Uses OpenAI GPT-4 to create social media captions.  
    - Configuration:  
      - Model: GPT-4  
      - System prompt: Instagram Caption Writer Agent for Sally's Coffee  
      - User prompt: "Generate an Instagram/Tik Tok Caption for Sally's Coffee"  
    - Inputs: From Upload media node  
    - Outputs: To Create post node  
    - Edge Cases: API limits, inappropriate content filtering, caption length constraints.

#### 1.7 Social Media Post Creation

- **Overview:**  
Creates and publishes the social media post on the selected account(s) using Blotato, combining video and caption.

- **Nodes Involved:**  
  - Create post

- **Node Details:**  
  - **Create post**  
    - Type: `@blotato/n8n-nodes-blotato.blotato`  
    - Role: Creates a new social media post with video and caption on selected Blotato account.  
    - Configuration:  
      - Account ID: 7680 (automatewithmarc)  
      - Post content text: Caption from Caption Generator output  
      - Media URLs: Video URL from Upload media node  
    - Inputs: From Caption Generator node  
    - Outputs: None (final step)  
    - Edge Cases: Posting failures, account authorization, media format issues.

---

### 3. Summary Table

| Node Name           | Node Type                                      | Functional Role                          | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|------------------------------------------------|----------------------------------------|-------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger     | n8n-nodes-base.scheduleTrigger                  | Starts workflow on schedule             | None                    | Video Prompt Agent      | Schdeule Trigger                                                                            |
| Video Prompt Agent   | @n8n/n8n-nodes-langchain.agent                  | Generates text-to-video prompt          | Schedule Trigger, Simple Memory, OpenAI Chat Model | Insert row             | Video Generation                                                                            |
| OpenAI Chat Model    | @n8n/n8n-nodes-langchain.lmChatOpenAi           | Provides GPT-5 model for prompt agent   | None                    | Video Prompt Agent (AI language model input) | Video Generation                                                                            |
| Simple Memory       | @n8n/n8n-nodes-langchain.memoryBufferWindow      | Maintains AI agent session context      | None                    | Video Prompt Agent (AI memory) | Video Generation                                                                            |
| Insert row          | n8n-nodes-base.dataTable                          | Logs generated prompts                   | Video Prompt Agent      | Sora2 POST Request      | Logging                                                                                    |
| Sora2 POST Request  | n8n-nodes-base.httpRequest                        | Sends prompt to Sora2 API to generate video | Insert row              | Wait 30 Secs            | POST and GET from Sora2                                                                    |
| Wait 30 Secs        | n8n-nodes-base.wait                              | Waits 30 seconds before polling result  | Sora2 POST Request      | GET Sora2 Result        | POST and GET from Sora2                                                                    |
| GET Sora2 Result    | n8n-nodes-base.httpRequest                        | Polls Sora2 API for video status        | Wait 30 Secs, Wait Another 30 Secs | If                      | POST and GET from Sora2                                                                    |
| If                  | n8n-nodes-base.if                                | Checks if video generation completed    | GET Sora2 Result        | Upload media, Wait Another 30 Secs | POST and GET from Sora2                                                                    |
| Wait Another 30 Secs| n8n-nodes-base.wait                              | Additional wait for video generation     | If (false branch)       | GET Sora2 Result        | POST and GET from Sora2                                                                    |
| Upload media        | @blotato/n8n-nodes-blotato.blotato               | Uploads video to Blotato                 | If (true branch)        | Caption Generator       | Upload to Bloatato (Instagram & Tik Tok)                                                  |
| Caption Generator   | @n8n/n8n-nodes-langchain.openAi                   | Creates social media caption             | Upload media            | Create post             | Upload to Bloatato (Instagram & Tik Tok)                                                  |
| Create post         | @blotato/n8n-nodes-blotato.blotato                | Publishes post to social media           | Caption Generator       | None                    | Upload to Bloatato (Instagram & Tik Tok)                                                  |
| Sticky Note         | n8n-nodes-base.stickyNote                         | Various notes                           | None                    | None                    | Multiple sticky notes with detailed project overview and YouTube tutorial link               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Schedule Trigger” node:**  
   - Type: `Schedule Trigger`  
   - Set to trigger daily at 19:00 (7 PM).

2. **Create “OpenAI Chat Model” node:**  
   - Type: `LangChain OpenAI Chat Model`  
   - Model: GPT-5  
   - Set timeout to 6000000 ms (100 minutes).  
   - No inputs, used as AI language model backend.

3. **Create “Simple Memory” node:**  
   - Type: `LangChain Memory Buffer Window`  
   - Session key: `={{ $execution.id }}`  
   - SessionIdType: customKey.

4. **Create “Video Prompt Agent” node:**  
   - Type: `LangChain Agent`  
   - Text prompt: “Generate an effective video prompt for a comedy reel promoting Sally’s Coffee”  
   - System message: Use the detailed prompt from the workflow explaining style, tone, and requirements for the video prompt (12-second cinematic reel with natural humor).  
   - Connect Schedule Trigger to “Video Prompt Agent” main input.  
   - Connect “OpenAI Chat Model” to agent’s AI language model input.  
   - Connect “Simple Memory” to agent’s AI memory input.

5. **Create “Insert row” node:**  
   - Type: `Data Table`  
   - Select or create a Data Table with schema having a string column “Video_Prompt”.  
   - Map “Video_Prompt” to `={{ $json.output }}` from “Video Prompt Agent”.  
   - Connect “Video Prompt Agent” output to this node.

6. **Create “Sora2 POST Request” node:**  
   - Type: `HTTP Request`  
   - Method: POST  
   - URL: `https://api.wavespeed.ai/api/v3/openai/sora-2/text-to-video`  
   - Body parameters:  
     - duration: 12  
     - prompt: `={{ $json.Video_Prompt }}`  
     - size: `720*1280`  
   - Authentication: HTTP Header Auth (configure Wavespeed/Sora2 credentials).  
   - Connect “Insert row” node output to this node.

7. **Create “Wait 30 Secs” node:**  
   - Type: `Wait`  
   - Amount: 30 seconds  
   - Connect “Sora2 POST Request” output to this node.

8. **Create “GET Sora2 Result” node:**  
   - Type: `HTTP Request`  
   - Method: GET  
   - URL: `https://api.wavespeed.ai/api/v3/predictions/{{ $json.data.id }}/result` (dynamic with prediction ID)  
   - Authentication: HTTP Header Auth (same as POST).  
   - Connect “Wait 30 Secs” output to this node.

9. **Create “If” node:**  
   - Type: `If`  
   - Condition: `$json.data.status` equals “completed”  
   - Connect “GET Sora2 Result” output to this node.

10. **Create “Wait Another 30 Secs” node:**  
    - Type: `Wait`  
    - Amount: 30 seconds  
    - Connect “If” node’s false branch output back to “GET Sora2 Result” node (loop).

11. **Create “Upload media” node:**  
    - Type: `Blotato`  
    - Operation: Upload media  
    - Media URL: `={{ $json.data.outputs[0] }}` (video URL from Sora2)  
    - Connect “If” node’s true branch output to this node.  
    - Configure Blotato credentials.

12. **Create “Caption Generator” node:**  
    - Type: `LangChain OpenAI`  
    - Model: GPT-4  
    - System prompt: "You're an Instagram Caption Writer Agent. Your job is to create effective Instagram or Tik Tok captions for Sally's Coffee, a local neighborhood brand. Include relevant hashtags."  
    - User prompt: "Generate an Instagram/Tik Tok Caption for Sally's Coffee"  
    - Connect “Upload media” output to this node.

13. **Create “Create post” node:**  
    - Type: `Blotato`  
    - Operation: Create post  
    - Account ID: Select your social media account ID (e.g., 7680)  
    - Post content text: `={{ $json.output[0].content[0].text }}` (caption from Caption Generator)  
    - Post content media URLs: `={{ $('Upload media').item.json.url }}`  
    - Connect “Caption Generator” output to this node.

14. **Add Sticky Notes for clarity:**  
    - Add notes for each block such as “Schedule Trigger”, “Video Generation”, “Logging”, “POST and GET from Sora2”, “Upload to Blotato (Instagram & TikTok)”, and a project overview note with the YouTube tutorial link.

15. **Credential Setup:**  
    - Add OpenAI API credentials in n8n.  
    - Add Wavespeed/Sora2 API credentials with HTTP header authentication.  
    - Add Blotato API credentials and connect social accounts.

16. **Testing and Validation:**  
    - Run the workflow manually once to test end-to-end.  
    - Monitor logs for errors such as API failures, timeout, or invalid data.  
    - Adjust prompts or retry intervals if necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                   | Context or Link                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Funny Marketing Reels Autopilot with Sora2 and Blotato: Automate comedic short-form marketing videos with AI-generated scripts, Sora2 video creation, caption writing, and auto-posting to social media.                                                                                                                         | Workflow description sticky note                    |
| Watch Step by Step Tutorial of this build: https://www.youtube.com/watch?v=lKZknEzhivo                                                                                                                                                                                                                                           | YouTube tutorial link                               |
| Typical use cases include local cafes or stores, creators/agencies prototyping AI-first social content, and marketing teams testing comedic formats without manual effort.                                                                                                                                                     | Workflow description sticky note                    |
| Key integrations: n8n Schedule Trigger, OpenAI Chat / LangChain, Wavespeed + Sora2, n8n Data Table, OpenAI for captions, Blotato for social posting.                                                                                                                                                                          | Workflow description sticky note                    |
| Before starting, configure your OpenAI, Wavespeed/Sora2, and Blotato credentials in n8n, and optionally update the Data Table ID for prompt logging.                                                                                                                                                                          | Workflow description sticky note                    |
| The workflow uses a custom OpenAI prompt to generate natural, relatable humor embedded in a realistic scene, avoiding forced slogans or surreal imagery, optimized for 12-second vertical reels.                                                                                                                               | Video Prompt Agent node system message              |
| Blotato node requires setting up accounts and credentials in advance for media upload and post creation, supporting Instagram and TikTok.                                                                                                                                                                                      | Upload and Create post nodes                         |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.