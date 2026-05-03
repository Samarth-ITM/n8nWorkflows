Generate & Publish AI Videos with Sora 2, Veo 3.1, Gemini & Blotato

https://n8nworkflows.xyz/workflows/generate---publish-ai-videos-with-sora-2--veo-3-1--gemini---blotato-11276


# Generate & Publish AI Videos with Sora 2, Veo 3.1, Gemini & Blotato

### 1. Workflow Overview

This workflow, titled **"Advanced Hybrid AI Video Generator with Sora, Veo & Blotato Multi-Publisher"**, automates the creation and multi-platform publishing of AI-generated videos from simple user prompts. It targets creators, brands, and user-generated content (UGC) teams who want to generate videos using cutting-edge AI models and distribute them efficiently.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Prompt Enhancement**  
  Receives chat messages as video ideas and enhances them into detailed video generation prompts using Google Gemini AI.

- **1.2 Configurable Branching & Toggles**  
  Allows enabling/disabling of video generation via Sora 2 or Veo 3.1, and toggles publishing destinations and logging.

- **1.3 Sora 2 Pro Video Generation & Publishing**  
  Generates videos via OpenAI’s Sora 2 Pro, downloads the resulting video, uploads it to Blotato media, and publishes across YouTube, TikTok, and Instagram via Blotato. It also sends an email confirmation and logs the video info to Google Sheets.

- **1.4 Veo 3.1 Video Generation, Result Handling & Optional Publishing**  
  Generates videos via Wavespeed’s Veo 3.1, waits for processing completion, emails the result link, uploads the video to Blotato, and optionally publishes it to YouTube.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Prompt Enhancement

**Overview:**  
This block listens for incoming chat messages, then uses Google Gemini to enhance the user’s raw input into a rich, detailed video prompt suitable for AI video generation models.

**Nodes Involved:**  
- When chat message received  
- Google Gemini Chat Model  
- Gemini Prompt Enhancer  

**Node Details:**

- **When chat message received**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Entry point listening for chat input (video idea).  
  - *Config:* Default options, webhook ID set for external trigger.  
  - *Inputs:* External chat messages.  
  - *Outputs:* Raw user message as JSON.  
  - *Failures:* Webhook connectivity issues, malformed input.  

- **Google Gemini Chat Model**  
  - *Type:* Langchain Google Gemini Chat Model  
  - *Role:* Provides AI-powered language model capabilities for prompt enhancement.  
  - *Config:* Uses Google PaLM API credentials. No extra parameters specified.  
  - *Inputs:* Receives raw chat input.  
  - *Outputs:* Generates AI-assisted text to improve prompt.  
  - *Failures:* API authentication errors, rate limits, network timeouts.  

- **Gemini Prompt Enhancer**  
  - *Type:* Langchain Agent Node  
  - *Role:* Takes user input and rewrites it into a detailed video generation prompt.  
  - *Config:*  
    - Input text: user chat input expression `{{$json.chatInput}}`  
    - System message rules: expand prompt with action, scene, lighting, mood, camera movement, style; safe for all audiences; output plain text only.  
  - *Inputs:* Output of Google Gemini Chat Model.  
  - *Outputs:* Enhanced prompt text for video generation.  
  - *Failures:* Expression evaluation errors, incorrect prompt formatting, AI response issues.  

---

#### 2.2 Configurable Branching & Toggles

**Overview:**  
This block uses a configuration node with boolean toggles to control whether to use Sora and/or Veo for video generation and which social platforms to publish on. It conditionally routes the flow based on these toggles.

**Nodes Involved:**  
- Config – Toggles  
- If Use Sora  
- If Use Veo  

**Node Details:**

- **Config – Toggles**  
  - *Type:* Set Node  
  - *Role:* Defines configuration flags as booleans to enable/disable Sora, Veo, YouTube, TikTok, Instagram publishing, and Google Sheets logging.  
  - *Config:*  
    - `useSora` = true  
    - `useVeo` = true  
    - `publishToYouTube` = true  
    - `publishToTikTok` = false (not set)  
    - `publishToInstagram` = false (not set)  
    - `logToGoogleSheets` = true  
  - *Inputs:* Enhanced prompt from Gemini Prompt Enhancer.  
  - *Outputs:* JSON with toggle flags.  
  - *Failures:* None typical; misconfiguration possible.  

- **If Use Sora**  
  - *Type:* If Node  
  - *Role:* Routes flow to Sora video generation if `useSora` is true.  
  - *Config:* Checks boolean expression `{{$json.useSora}}`.  
  - *Inputs:* Output of Config – Toggles.  
  - *Outputs:* Main output if true, else no output.  
  - *Failures:* Expression evaluation errors.  

- **If Use Veo**  
  - *Type:* If Node  
  - *Role:* Routes flow to Veo video generation if `useVeo` is true.  
  - *Config:* Checks boolean expression `{{$json.useVeo}}`.  
  - *Inputs:* Output of Config – Toggles.  
  - *Outputs:* Main output if true, else no output.  
  - *Failures:* Expression evaluation errors.  

---

#### 2.3 Sora 2 Pro Video Generation & Multi-Platform Publishing

**Overview:**  
This block sends the enhanced prompt to OpenAI Sora 2 Pro to create an 8-second video, downloads the video, uploads it to Blotato media hosting, then publishes across YouTube, TikTok, and Instagram using Blotato. It also sends an email confirmation and logs the event to Google Sheets.

**Nodes Involved:**  
- Create Sora 2 Pro Video  
- Download Sora Video  
- Upload Sora Video to Blotato  
- Publish Sora to YouTube  
- Publish Sora to TikTok  
- Publish Sora to Instagram  
- Email Sora Upload Confirmation  
- Log Sora Video to Google Sheets  

**Node Details:**

- **Create Sora 2 Pro Video**  
  - *Type:* HTTP Request  
  - *Role:* POST request to OpenAI video API to generate an 8-second 1280x720 video using `sora-2-pro` model.  
  - *Config:*  
    - URL: `https://api.openai.com/v1/videos`  
    - Method: POST  
    - Body (multipart-form-data): prompt (enhanced prompt), model, size, seconds  
    - Authentication: HTTP header with OpenAI API key credential.  
  - *Inputs:* Enhanced prompt or raw chat input fallback.  
  - *Outputs:* JSON with video generation response (includes video ID).  
  - *Failures:* API auth errors, quota limits, invalid prompt, network failures.  

- **Download Sora Video**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the generated MP4 video file from OpenAI using video ID.  
  - *Config:*  
    - URL constructed dynamically: `https://api.openai.com/v1/videos/{{ $json.id }}/content`  
    - Response format: file (video)  
    - Authentication: OpenAI API key.  
  - *Inputs:* Video ID from Create Sora 2 Pro Video.  
  - *Outputs:* Binary video file in output property.  
  - *Failures:* Video not ready, auth errors, network timeouts.  

- **Upload Sora Video to Blotato**  
  - *Type:* Blotato Node  
  - *Role:* Uploads downloaded video file to Blotato media hosting.  
  - *Config:* Resource set to "media".  
  - *Inputs:* Binary video file from Download Sora Video.  
  - *Outputs:* JSON with uploaded media URL.  
  - *Failures:* API connectivity, auth issues, upload size limits.  

- **Publish Sora to YouTube / TikTok / Instagram**  
  - *Type:* Blotato Nodes (3 separate nodes)  
  - *Role:* Publish uploaded video with enhanced prompt as post text to respective social platforms.  
  - *Config:*  
    - Platform specified per node (YouTube, TikTok, Instagram)  
    - Account ID configured (placeholder `YOUR_BLOTATO_ACCOUNT_ID`)  
    - Post text uses Gemini Prompt Enhancer output fallback to chat input.  
    - Media URLs use uploaded video URL.  
    - YouTube options include synthetic media flag and title set to prompt.  
  - *Inputs:* Media URL from Blotato upload.  
  - *Outputs:* Publish confirmation data.  
  - *Failures:* Platform API limits, credential expiry, invalid media format, quota limits.  

- **Email Sora Upload Confirmation**  
  - *Type:* Gmail Node  
  - *Role:* Sends email confirmation about successful Sora video generation and publishing.  
  - *Config:*  
    - Recipient: configured email (placeholder).  
    - Subject: fixed text.  
    - Message: fixed text explaining where to find published content.  
  - *Inputs:* After publishing.  
  - *Outputs:* Email send confirmation.  
  - *Failures:* SMTP auth errors, network failures.  

- **Log Sora Video to Google Sheets**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends a row with video info for tracking.  
  - *Config:*  
    - Operation: Append  
    - Sheet name and Document ID placeholders (must be set).  
  - *Inputs:* After email confirmation.  
  - *Outputs:* Append confirmation.  
  - *Failures:* Sheet ID/sheet name misconfiguration, API quota, auth errors.  

---

#### 2.4 Veo 3.1 Video Generation, Result Handling & Optional Publishing

**Overview:**  
Generates a video with Wavespeed Veo 3.1 from the enhanced prompt, waits 60 seconds for processing, fetches the result link, emails it to the user, uploads the video to Blotato, and optionally publishes it to YouTube.

**Nodes Involved:**  
- Create Veo 3.1 Video  
- Wait for Veo Result  
- Get Veo Result  
- Email Veo Video Link  
- Upload Veo Video to Blotato  
- Publish Veo to YouTube  

**Node Details:**

- **Create Veo 3.1 Video**  
  - *Type:* HTTP Request  
  - *Role:* Sends prompt to Wavespeed Veo 3.1 API for 8-second 720p vertical video with audio.  
  - *Config:*  
    - URL: `https://api.wavespeed.ai/api/v3/google/veo3.1/text-to-video`  
    - Method: POST  
    - Body parameters: aspect_ratio=9:16, duration=8, generate_audio=true, prompt, resolution=720p  
    - Authentication: HTTP header with Wavespeed API key.  
  - *Inputs:* Enhanced prompt or raw chat input fallback.  
  - *Outputs:* JSON with prediction job ID.  
  - *Failures:* API auth, invalid parameter, network errors.  

- **Wait for Veo Result**  
  - *Type:* Wait Node  
  - *Role:* Delays workflow for 60 seconds to allow Veo job to complete.  
  - *Config:* Fixed 60 seconds delay.  
  - *Inputs:* After Create Veo 3.1 Video.  
  - *Outputs:* Delayed pass-through.  
  - *Failures:* None typical.  

- **Get Veo Result**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves processed video URL using prediction job ID from Veo API.  
  - *Config:*  
    - URL constructed dynamically: `https://api.wavespeed.ai/api/v3/predictions/{{ $json.data.id }}/result`  
    - Authentication: Wavespeed API key.  
  - *Inputs:* Prediction job ID from Create Veo 3.1 Video.  
  - *Outputs:* Result JSON including video output link(s).  
  - *Failures:* Job not completed, invalid ID, auth errors.  

- **Email Veo Video Link**  
  - *Type:* Gmail Node  
  - *Role:* Sends an email with Veo video result link or fallback message.  
  - *Config:*  
    - Recipient: configured email (placeholder).  
    - Subject: fixed text.  
    - Message: uses expression to send first output link or fallback text.  
  - *Inputs:* Result JSON from Get Veo Result.  
  - *Outputs:* Email send confirmation.  
  - *Failures:* SMTP issues, invalid email.  

- **Upload Veo Video to Blotato**  
  - *Type:* Blotato Node  
  - *Role:* Uploads the Veo video URL to Blotato media hosting.  
  - *Config:*  
    - Media URL from Veo result outputs.  
    - Resource: media.  
  - *Inputs:* Video URL from Get Veo Result.  
  - *Outputs:* Uploaded media info.  
  - *Failures:* Invalid URL, Blotato auth, upload failure.  

- **Publish Veo to YouTube**  
  - *Type:* Blotato Node  
  - *Role:* Publishes Veo video via Blotato YouTube account with prompt as post text.  
  - *Config:*  
    - Platform: YouTube.  
    - Account ID placeholder.  
    - Post content text uses Gemini Prompt Enhancer output or fallback.  
    - Media URLs from uploaded Veo video.  
    - Synthetic media flag enabled.  
  - *Inputs:* From Upload Veo Video to Blotato.  
  - *Outputs:* Publish confirmation.  
  - *Failures:* Platform API errors, auth issues.  

---

### 3. Summary Table

| Node Name                    | Node Type                               | Functional Role                                  | Input Node(s)                       | Output Node(s)                                  | Sticky Note                                                                                                                                                                                                                  |
|------------------------------|---------------------------------------|-------------------------------------------------|-----------------------------------|------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received    | Langchain Chat Trigger                 | Entry point for receiving user video ideas      | —                                 | Google Gemini Chat Model, Gemini Prompt Enhancer | ## Input & Prompt Enhancement<br>This section receives your idea from the Chat Trigger and rewrites it into a detailed, structured video prompt using Google Gemini. The enhanced prompt is then passed to the Sora and Veo branches for video generation. |
| Google Gemini Chat Model      | Langchain Google Gemini Chat Model     | AI language model for prompt enhancement         | When chat message received          | Gemini Prompt Enhancer                          | See above                                                                                                                                                                                                                   |
| Gemini Prompt Enhancer        | Langchain Agent                       | Rewrites user input into detailed video prompt   | Google Gemini Chat Model, When chat message received | Config – Toggles                                  | See above                                                                                                                                                                                                                   |
| Config – Toggles             | Set Node                             | Enables/disables video generation and publishing | Gemini Prompt Enhancer              | If Use Veo, If Use Sora                        | ## Advanced Hybrid AI Video Generator (Sora + Veo + Gemini + Blotato)<br>This workflow turns a simple idea into AI-generated videos using OpenAI Sora 2 and Google Veo 3.1. Your message is enhanced by Google Gemini, then sent to both video models in parallel.<br>The Sora branch downloads the MP4 file and publishes it to multiple platforms through Blotato (YouTube, TikTok, Instagram). The Veo branch retrieves the Wavespeed result link, emails it to you for comparison, and can optionally upload and publish the Veo output through Blotato.<br>A Config – Toggles node lets you enable or disable Sora, Veo, each publishing platform, and optional Google Sheets logging.<br>This template is ideal for creators, brands, UGC teams, and anyone wanting a flexible, multi-model AI video pipeline with automated cross-platform publishing. |
| If Use Sora                  | If Node                             | Branches flow to Sora video generation           | Config – Toggles                   | Create Sora 2 Pro Video                        | See above                                                                                                                                                                                                                   |
| Create Sora 2 Pro Video       | HTTP Request                        | Requests video generation from OpenAI Sora 2    | If Use Sora                       | Download Sora Video                            | ## Sora Generation & Multi-Platform Publishing<br>This section generates a Sora 2 Pro video, downloads the MP4, uploads it to Blotato, and publishes it to YouTube, TikTok, and Instagram. It can also log each result to Google Sheets for tracking. |
| Download Sora Video           | HTTP Request                        | Downloads generated Sora video MP4                | Create Sora 2 Pro Video           | Upload Sora Video to Blotato                    | See above                                                                                                                                                                                                                   |
| Upload Sora Video to Blotato  | Blotato Media Upload                | Uploads video to Blotato media hosting             | Download Sora Video               | Publish Sora to YouTube, Publish Sora to TikTok, Publish Sora to Instagram | See above                                                                                                                                                                                                                   |
| Publish Sora to YouTube       | Blotato Publisher                  | Publishes video to YouTube via Blotato           | Upload Sora Video to Blotato       | Email Sora Upload Confirmation                  | See above                                                                                                                                                                                                                   |
| Publish Sora to TikTok        | Blotato Publisher                  | Publishes video to TikTok via Blotato            | Upload Sora Video to Blotato       | —                                              | See above                                                                                                                                                                                                                   |
| Publish Sora to Instagram     | Blotato Publisher                  | Publishes video to Instagram via Blotato         | Upload Sora Video to Blotato       | —                                              | See above                                                                                                                                                                                                                   |
| Email Sora Upload Confirmation| Gmail                             | Sends confirmation email after Sora publishing   | Publish Sora to YouTube           | Log Sora Video to Google Sheets                  | See above                                                                                                                                                                                                                   |
| Log Sora Video to Google Sheets | Google Sheets                   | Logs video info for tracking                       | Email Sora Upload Confirmation    | —                                              | See above                                                                                                                                                                                                                   |
| If Use Veo                   | If Node                             | Branches flow to Veo video generation             | Config – Toggles                   | Create Veo 3.1 Video                           | ## Veo Comparison & Optional Publishing<br>This section generates a Veo 3.1 video via Wavespeed, emails you the result link, and can upload the output to Blotato for optional publishing to YouTube.                      |
| Create Veo 3.1 Video          | HTTP Request                        | Requests video generation from Wavespeed Veo 3.1 | If Use Veo                       | Wait for Veo Result                            | See above                                                                                                                                                                                                                   |
| Wait for Veo Result           | Wait Node                         | Waits 60 seconds for Veo video processing         | Create Veo 3.1 Video              | Get Veo Result                                 | See above                                                                                                                                                                                                                   |
| Get Veo Result                | HTTP Request                        | Fetches Veo video result link                       | Wait for Veo Result              | Email Veo Video Link                            | See above                                                                                                                                                                                                                   |
| Email Veo Video Link          | Gmail                             | Sends email with Veo video result link             | Get Veo Result                   | Upload Veo Video to Blotato                      | See above                                                                                                                                                                                                                   |
| Upload Veo Video to Blotato    | Blotato Media Upload                | Uploads Veo video to Blotato media hosting          | Email Veo Video Link             | Publish Veo to YouTube                          | See above                                                                                                                                                                                                                   |
| Publish Veo to YouTube        | Blotato Publisher                  | Publishes Veo video to YouTube via Blotato         | Upload Veo Video to Blotato       | —                                              | See above                                                                                                                                                                                                                   |
| Sticky Note                  | Sticky Note                       | Workflow description and usage notes               | —                                 | —                                              | See content in node details section                                                                                                                                                                                        |
| Sticky Note1                 | Sticky Note                       | Input and prompt enhancement explanation            | —                                 | —                                              | See content in node details section                                                                                                                                                                                        |
| Sticky Note2                 | Sticky Note                       | Sora generation and publishing explanation          | —                                 | —                                              | See content in node details section                                                                                                                                                                                        |
| Sticky Note3                 | Sticky Note                       | Veo generation and publishing explanation           | —                                 | —                                              | See content in node details section                                                                                                                                                                                        |
| Sticky Note4                 | Sticky Note                       | Setup instructions before running                     | —                                 | —                                              | See content in node details section                                                                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: Langchain Chat Trigger (v1.3)  
   - Webhook ID: `chat-trigger-advanced-hybrid`  
   - Purpose: Entry point to receive user input for video idea.

2. **Add "Google Gemini Chat Model" node**  
   - Type: Langchain Google Gemini Chat Model (v1)  
   - Credentials: Google PaLM API account configured with valid key.  
   - Connect input from "When chat message received" node.  
   - Purpose: Provide AI model interface for prompt enhancement.

3. **Add "Gemini Prompt Enhancer" node**  
   - Type: Langchain Agent (v3)  
   - Parameters:  
     - Text expression: `={{ $json.chatInput }}`  
     - System message (copy exactly):  
       ```
       You are an AI assistant that rewrites user ideas into highly detailed AI video generation prompts.

       Rules:
       - Take the user input and expand it into a single improved prompt suitable for models like OpenAI Sora 2 and Google Veo 3.1.
       - Focus on describing action, scene, lighting, mood, camera movement and style.
       - Make it safe for all audiences.
       - Output ONLY the improved prompt as plain text (no JSON, no keys).
       ```  
   - Connect input from "Google Gemini Chat Model" node.  
   - Purpose: Enhance user input into detailed video prompt.

4. **Add "Config – Toggles" node**  
   - Type: Set Node (v2)  
   - Parameters:  
     - Boolean values:  
       - `useSora`: true  
       - `useVeo`: true  
       - `publishToYouTube`: true  
       - `publishToTikTok`: false (unset)  
       - `publishToInstagram`: false (unset)  
       - `logToGoogleSheets`: true  
   - Connect input from "Gemini Prompt Enhancer" node.  
   - Purpose: Control workflow branches and publishing options.

5. **Add "If Use Veo" node**  
   - Type: If Node (v2)  
   - Condition: Boolean expression `{{$json.useVeo}} is True`  
   - Connect input from "Config – Toggles" node.

6. **Add "If Use Sora" node**  
   - Type: If Node (v2)  
   - Condition: Boolean expression `{{$json.useSora}} is True`  
   - Connect input from "Config – Toggles" node.

---

**Sora Branch:**

7. **Create "Create Sora 2 Pro Video" node**  
   - Type: HTTP Request (v4.2)  
   - Method: POST  
   - URL: `https://api.openai.com/v1/videos`  
   - Authentication: HTTP Header Auth with OpenAI API key credential.  
   - Body (multipart-form-data):  
     - prompt: `={{ $json.output || $json.chatInput }}`  
     - model: `sora-2-pro`  
     - size: `1280x720`  
     - seconds: `8`  
   - Connect input from "If Use Sora" node.

8. **Create "Download Sora Video" node**  
   - Type: HTTP Request (v4.2)  
   - Method: GET  
   - URL: `=https://api.openai.com/v1/videos/{{ $json.id }}/content`  
   - Response format: File (binary)  
   - Authentication: HTTP Header Auth with OpenAI API key credential.  
   - Connect input from "Create Sora 2 Pro Video" node.

9. **Create "Upload Sora Video to Blotato" node**  
   - Type: Blotato node (media resource) (v2)  
   - Credentials: Blotato API account configured.  
   - Connect input from "Download Sora Video" node.

10. **Create "Publish Sora to YouTube" node**  
    - Type: Blotato node (publish resource) (v2)  
    - Parameters:  
      - platform: YouTube  
      - accountId: your Blotato YouTube account ID  
      - postContentText: `={{ $('Gemini Prompt Enhancer').item.json.output || $json.chatInput }}`  
      - postContentMediaUrls: `={{ $json.url }}`  
      - postCreateYoutubeOptionTitle: same as postContentText  
      - postCreateYoutubeOptionContainsSyntheticMedia: true  
    - Credentials: Blotato API account  
    - Connect input from "Upload Sora Video to Blotato" node.

11. **Create "Publish Sora to TikTok" node**  
    - Similar setup as YouTube node but platform: TikTok  
    - Connect input from "Upload Sora Video to Blotato" node.

12. **Create "Publish Sora to Instagram" node**  
    - Similar setup, platform: Instagram  
    - Connect input from "Upload Sora Video to Blotato" node.

13. **Create "Email Sora Upload Confirmation" node**  
    - Type: Gmail (v2.1)  
    - Recipient: your email address  
    - Subject: `Sora Video Published via Blotato`  
    - Message: `Your Sora 2 Pro video has been generated and uploaded via Blotato. Check your connected social channels (YouTube, TikTok, Instagram) or Blotato dashboard for the published content.`  
    - Connect input from "Publish Sora to YouTube" node.

14. **Create "Log Sora Video to Google Sheets" node**  
    - Type: Google Sheets (v4)  
    - Operation: Append  
    - Document ID and Sheet Name: configure with your Google Sheets details  
    - Connect input from "Email Sora Upload Confirmation" node.

---

**Veo Branch:**

15. **Create "Create Veo 3.1 Video" node**  
    - Type: HTTP Request (v4.2)  
    - Method: POST  
    - URL: `https://api.wavespeed.ai/api/v3/google/veo3.1/text-to-video`  
    - Authentication: HTTP Header Auth with Wavespeed (Weav Veo) API key  
    - Body Parameters:  
      - aspect_ratio: `9:16`  
      - duration: `8`  
      - generate_audio: `true`  
      - prompt: `={{ $json.output || $json.chatInput }}`  
      - resolution: `720p`  
    - Connect input from "If Use Veo" node.

16. **Create "Wait for Veo Result" node**  
    - Type: Wait Node (v1.1)  
    - Wait time: 60 seconds  
    - Connect input from "Create Veo 3.1 Video" node.

17. **Create "Get Veo Result" node**  
    - Type: HTTP Request (v4.2)  
    - Method: GET  
    - URL: `=https://api.wavespeed.ai/api/v3/predictions/{{ $json.data.id }}/result`  
    - Authentication: Wavespeed API key  
    - Connect input from "Wait for Veo Result" node.

18. **Create "Email Veo Video Link" node**  
    - Type: Gmail (v2.1)  
    - Recipient: your email address  
    - Subject: `Veo 3.1 Video Result`  
    - Message:  
      ```
      ={{ $json.data && $json.data.outputs && $json.data.outputs[0] ? $json.data.outputs[0] : 'Your Veo 3.1 video has finished. Check Wavespeed dashboard for details.' }}
      ```  
    - Connect input from "Get Veo Result" node.

19. **Create "Upload Veo Video to Blotato" node**  
    - Type: Blotato node (media resource) (v2)  
    - Media URL: `={{ $json.data && $json.data.outputs && $json.data.outputs[0] ? $json.data.outputs[0] : '' }}`  
    - Credentials: Blotato API account  
    - Connect input from "Email Veo Video Link" node.

20. **Create "Publish Veo to YouTube" node**  
    - Type: Blotato node (publish resource) (v2)  
    - Platform: YouTube  
    - Account ID: your Blotato YouTube account ID  
    - Post content text: `={{ $('Gemini Prompt Enhancer').item.json.output || $json.chatInput }}`  
    - Media URLs: `={{ $json.url }}`  
    - Synthetic media flag: true  
    - Credentials: Blotato API account  
    - Connect input from "Upload Veo Video to Blotato" node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow turns a simple idea into AI-generated videos using OpenAI Sora 2 and Google Veo 3.1. Your message is enhanced by Google Gemini, then sent to both video models in parallel. The Sora branch downloads MP4 and publishes it to multiple platforms through Blotato (YouTube, TikTok, Instagram). The Veo branch emails the Wavespeed result link and optionally uploads and publishes the Veo output through Blotato. | Sticky Note in workflow overview section.                    |
| Input & Prompt Enhancement: receives idea from chat trigger and rewrites it into a detailed video prompt using Google Gemini.                                                                                                         | Sticky Note1                                                 |
| Sora Generation & Multi-Platform Publishing: generates Sora 2 Pro video, downloads MP4, uploads to Blotato, publishes to YouTube, TikTok, Instagram, and logs results to Google Sheets.                                               | Sticky Note2                                                 |
| Veo Comparison & Optional Publishing: generates Veo 3.1 video, emails result link, optionally uploads and publishes via Blotato YouTube.                                                                                               | Sticky Note3                                                 |
| Before running: set API keys, connect Blotato accounts, adjust toggles in Config node. Optional platforms can be enabled or disabled as needed.                                                                                        | Sticky Note4                                                 |
| Blotato platform links and API docs: https://blotato.com/  
| Google PaLM API docs: https://developers.generativeai.google  
| OpenAI Video API docs: https://platform.openai.com/docs/api-reference/videos  
| Wavespeed Veo API docs: https://api.wavespeed.ai/docs  

---

**Disclaimer:**  
The provided text is exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.