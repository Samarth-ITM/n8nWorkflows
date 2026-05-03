Create Talking Avatar Videos with Elevenlabs & Infinitalk and Auto-Post to TikTok

https://n8nworkflows.xyz/workflows/create-talking-avatar-videos-with-elevenlabs---infinitalk-and-auto-post-to-tiktok-8378


# Create Talking Avatar Videos with Elevenlabs & Infinitalk and Auto-Post to TikTok

---

### 1. Workflow Overview

This workflow automates the creation of a talking AI avatar video from a single image and automatically posts it to TikTok using Postiz. It chains two AI services, Elevenlabs for text-to-speech voice synthesis and Infinitalk for generating the talking avatar video, then processes and uploads the video content before publishing it. The main logical blocks are:

- **1.1 Input Setup:** Define text and voice parameters for speech synthesis.
- **1.2 Voice Creation (Elevenlabs TTS):** Send text to Elevenlabs API to create an audio file and poll for completion.
- **1.3 Video Creation (Infinitalk):** Use the generated audio and source image to create a talking avatar video, polling for completion.
- **1.4 Video Processing and Upload:** Retrieve the completed video, generate a TikTok-optimized title using OpenAI, upload the video to Postiz.
- **1.5 TikTok Posting:** Publish the uploaded video on TikTok via Postiz integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup

**Overview:**  
This block initializes the workflow with user-defined input text and voice parameters used for speech synthesis.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set text input

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow manually for testing  
  - Configuration: No parameters  
  - Inputs: None  
  - Outputs: Connects to "Set text input"  
  - Potential Failures: None (manual trigger)  

- **Set text input**  
  - Type: Set Node  
  - Role: Defines input variables `text` and `voice` for TTS  
  - Configuration:  
    - `text`: "Tomorrow in New York the weather will be clear and partly cloudy with a temperature of 25 degrees"  
    - `voice`: "Alice"  
  - Inputs: From manual trigger  
  - Outputs: To "Create voice"  
  - Expressions: None beyond direct string assignment  
  - Edge Cases: Text length or voice name mismatch with Elevenlabs voice options

---

#### 2.2 Voice Creation (Elevenlabs TTS)

**Overview:**  
This block sends the text and voice data to Elevenlabs via Fal.run API to generate speech audio, then polls the API until audio creation is confirmed complete.

**Nodes Involved:**  
- Create voice  
- Wait 60 sec.2  
- Get status audio  
- Completed audio?  
- Get Url Audio  
- Set Audio Url

**Node Details:**

- **Create voice**  
  - Type: HTTP Request  
  - Role: Initiates TTS request to Elevenlabs via Fal.run API  
  - Configuration:  
    - POST to `https://queue.fal.run/fal-ai/elevenlabs/tts/eleven-v3`  
    - JSON body with `text` and `voice` parameters from input  
    - Auth: HTTP Header Auth with Fal.run API key  
  - Inputs: From "Set text input"  
  - Outputs: To "Wait 60 sec.2"  
  - Edge cases: API authentication failure, network errors, invalid voice name

- **Wait 60 sec.2**  
  - Type: Wait  
  - Role: Delays next API status check for 60 seconds to allow processing  
  - Inputs: From "Create voice" or "Completed audio?" (retry loop)  
  - Outputs: To "Get status audio"  

- **Get status audio**  
  - Type: HTTP Request  
  - Role: Polls Elevenlabs request status using request_id from "Create voice" response  
  - Configuration:  
    - GET `https://queue.fal.run/fal-ai/elevenlabs/requests/{{ request_id }}/status`  
    - Auth: Fal.run API key  
  - Inputs: From "Wait 60 sec.2"  
  - Outputs: To "Completed audio?"  

- **Completed audio?**  
  - Type: If  
  - Role: Checks if Elevenlabs response status is "COMPLETED"  
  - Condition: `$json.status == "COMPLETED"`  
  - Inputs: From "Get status audio"  
  - Outputs:  
    - True: To "Get Url Audio"  
    - False: Back to "Wait 60 sec.2" (retry polling)  
  - Edge cases: Unexpected status codes, infinite loop if never completed

- **Get Url Audio**  
  - Type: HTTP Request  
  - Role: Fetches audio metadata including audio URL once generation is complete  
  - Configuration:  
    - GET `https://queue.fal.run/fal-ai/elevenlabs/requests/{{ request_id }}`  
    - Auth: Fal.run API key  
  - Inputs: From "Completed audio?"  
  - Outputs: To "Set Audio Url"  

- **Set Audio Url**  
  - Type: Set  
  - Role: Extracts and sets the `audio_url` for use in video creation  
  - Configuration: Assign `audio_url` from `json.audio.url`  
  - Inputs: From "Get Url Audio"  
  - Outputs: To "Set Video Params"

---

#### 2.3 Video Creation (Infinitalk)

**Overview:**  
This block combines the generated audio with a predefined image and prompt to create a talking avatar video, polling for completion similarly to voice creation.

**Nodes Involved:**  
- Set Video Params  
- Create Video  
- Wait 60 sec.1  
- Get status  
- Completed?  
- Get Url Video

**Node Details:**

- **Set Video Params**  
  - Type: Set  
  - Role: Defines video input parameters: image URL, audio URL, and prompt for avatar behavior  
  - Configuration:  
    - `image_url`: Static image URL "https://n3wstorage.b-cdn.net/n3witalia/result2.png"  
    - `audio_url`: From previous block's `audio_url`  
    - `prompt`: "You are a girl who makes weather forecasts and needs to be expressive"  
  - Inputs: From "Set Audio Url"  
  - Outputs: To "Create Video"  

- **Create Video**  
  - Type: HTTP Request  
  - Role: Sends a request to Infinitalk API to generate talking avatar video with audio and image  
  - Configuration:  
    - POST to `https://queue.fal.run/fal-ai/infinitalk`  
    - JSON body with `image_url`, `audio_url`, and `prompt`  
    - Auth: Fal.run API key  
  - Inputs: From "Set Video Params"  
  - Outputs: To "Wait 60 sec.1"  

- **Wait 60 sec.1**  
  - Type: Wait  
  - Role: Delay before polling video generation status  
  - Inputs: From "Create Video" or "Completed?" (retry loop)  
  - Outputs: To "Get status"  

- **Get status**  
  - Type: HTTP Request  
  - Role: Polls status of Infinitalk video generation request using `request_id`  
  - Configuration:  
    - GET `https://queue.fal.run/fal-ai/infinitalk/requests/{{ request_id }}/status`  
    - Auth: Fal.run API key  
  - Inputs: From "Wait 60 sec.1"  
  - Outputs: To "Completed?"  

- **Completed?**  
  - Type: If  
  - Role: Checks if video generation status is "COMPLETED"  
  - Condition: `$json.status == "COMPLETED"`  
  - Inputs: From "Get status"  
  - Outputs:  
    - True: To "Get Url Video"  
    - False: Back to "Wait 60 sec.1" (retry polling)  
  - Edge cases: API errors, infinite polling loop if never completed  

- **Get Url Video**  
  - Type: HTTP Request  
  - Role: Fetches final video metadata including downloadable URL  
  - Configuration:  
    - GET `https://queue.fal.run/fal-ai/infinitalk/requests/{{ request_id }}`  
    - Auth: Fal.run API key  
  - Inputs: From "Completed?"  
  - Outputs: To "Generate title"  

---

#### 2.4 Video Processing and Upload

**Overview:**  
Retrieve the video file, generate a catchy TikTok title with OpenAI GPT-4o-mini, upload video to Postiz platform preparing it for TikTok posting.

**Nodes Involved:**  
- Generate title  
- Get File Video  
- Upload Video to Postiz

**Node Details:**

- **Generate title**  
  - Type: OpenAI (LangChain) Node  
  - Role: Generates an SEO-optimized and catchy TikTok video title in the same language as input text  
  - Configuration:  
    - Model: GPT-4o-mini  
    - Prompt includes system message with guidelines and example, input variables: video description text and video prompt  
  - Inputs: From "Get Url Video"  
  - Outputs: To "Get File Video"  
  - Credentials: OpenAI API credentials required  
  - Edge cases: API quota limits, network errors, prompt failures  

- **Get File Video**  
  - Type: HTTP Request  
  - Role: Downloads video file from URL provided by "Get Url Video" node  
  - Configuration:  
    - URL dynamically set from video URL in previous node  
  - Inputs: From "Generate title"  
  - Outputs: To "Upload Video to Postiz"  

- **Upload Video to Postiz**  
  - Type: HTTP Request  
  - Role: Uploads video binary data to Postiz platform for content management and TikTok integration  
  - Configuration:  
    - POST to `https://api.postiz.com/public/v1/upload`  
    - Content-Type: multipart/form-data with file binary data  
    - Auth: HTTP Header Auth with Postiz API key  
  - Inputs: From "Get File Video" (binary data)  
  - Outputs: To "TikTok" node  
  - Edge cases: Upload failures, auth errors, file size limits  

---

#### 2.5 TikTok Posting

**Overview:**  
This final block publishes the uploaded video to TikTok using Postiz's integration, using the generated title as content.

**Nodes Involved:**  
- TikTok

**Node Details:**

- **TikTok**  
  - Type: Postiz Node (custom integration)  
  - Role: Creates a TikTok post with uploaded video file and generated title  
  - Configuration:  
    - Post content includes video image with ID and path from upload result  
    - Content text from "Generate title" node output  
    - Date set to current timestamp  
    - Integration ID: Needs to be set with TikTok channel ID  
    - Option enabled for shortening links  
  - Inputs: From "Upload Video to Postiz"  
  - Outputs: None (terminal node)  
  - Edge cases: Invalid integration ID, API errors, posting limits  

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                               | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                                                   |
|-------------------------|-----------------------------|-----------------------------------------------|---------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger               | Manual start of workflow                       | None                      | Set text input                |                                                                                                                              |
| Set text input          | Set                         | Define initial text and voice parameters       | When clicking ‘Test workflow’ | Create voice                 |                                                                                                                              |
| Create voice            | HTTP Request                | Request text-to-speech audio generation        | Set text input             | Wait 60 sec.2                | Set API Key created in Step 2 (Sticky Note8 and Sticky Note9)                                                                 |
| Wait 60 sec.2           | Wait                        | Delay before polling Elevenlabs status         | Create voice / Completed audio? | Get status audio            |                                                                                                                              |
| Get status audio        | HTTP Request                | Poll Elevenlabs TTS request status             | Wait 60 sec.2              | Completed audio?             |                                                                                                                              |
| Completed audio?        | If                          | Check if audio generation completed            | Get status audio           | Get Url Audio / Wait 60 sec.2 |                                                                                                                              |
| Get Url Audio           | HTTP Request                | Retrieve audio URL and metadata                 | Completed audio?           | Set Audio Url                |                                                                                                                              |
| Set Audio Url           | Set                         | Extract audio URL for video creation            | Get Url Audio              | Set Video Params             |                                                                                                                              |
| Set Video Params        | Set                         | Prepare parameters for video generation         | Set Audio Url              | Create Video                 |                                                                                                                              |
| Create Video            | HTTP Request                | Request creation of talking avatar video        | Set Video Params           | Wait 60 sec.1                | Set API Key created in Step 2 (Sticky Note1 and Sticky Note3)                                                                  |
| Wait 60 sec.1           | Wait                        | Delay before polling video generation status    | Create Video / Completed?  | Get status                  |                                                                                                                              |
| Get status              | HTTP Request                | Poll Infinitalk video generation status         | Wait 60 sec.1              | Completed?                  |                                                                                                                              |
| Completed?              | If                          | Check if video generation completed             | Get status                 | Get Url Video / Wait 60 sec.1 |                                                                                                                              |
| Get Url Video           | HTTP Request                | Retrieve final video URL and metadata            | Completed?                 | Generate title              |                                                                                                                              |
| Generate title          | OpenAI (LangChain)          | Generate TikTok-optimized video title            | Get Url Video              | Get File Video              |                                                                                                                              |
| Get File Video          | HTTP Request                | Download video file                              | Generate title             | Upload Video to Postiz       |                                                                                                                              |
| Upload Video to Postiz  | HTTP Request                | Upload video to Postiz platform                   | Get File Video             | TikTok                     | Set ChannelId Step 3 (Sticky Note2)                                                                                           |
| TikTok                 | Postiz Node                 | Publish video post on TikTok                      | Upload Video to Postiz     | None                        |                                                                                                                              |
| Sticky Note1            | Sticky Note                 | Input image and output video example             | None                      | None                        | Displays input/output images and video download link                                                                          |
| Sticky Note8            | Sticky Note                 | Reminder to set API Key                           | None                      | None                        | Set API Key created in Step 2                                                                                                 |
| Sticky Note9            | Sticky Note                 | Reminder to set API Key                           | None                      | None                        | Set API Key created in Step 2                                                                                                 |
| Sticky Note2            | Sticky Note                 | Reminder to set TikTok ChannelId                  | None                      | None                        | Set ChannelId Step 3                                                                                                          |
| Sticky Note3            | Sticky Note                 | Setup instructions for initial parameters        | None                      | None                        | SETUP STEPS: Set text and voice, image URL and prompt, install Postiz node                                                    |
| Sticky Note             | Sticky Note                 | Workflow purpose and AI services overview         | None                      | None                        | Describes workflow purpose and AI services used (Elevenlabs and Infinitalk)                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Manual start point.

2. **Create Set Node for Text Input**  
   - Name: `Set text input`  
   - Assign fields:  
     - `text`: "Tomorrow in New York the weather will be clear and partly cloudy with a temperature of 25 degrees"  
     - `voice`: "Alice"  
   - Connect output from manual trigger.

3. **Create HTTP Request Node for Voice Creation**  
   - Name: `Create voice`  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/elevenlabs/tts/eleven-v3`  
   - Body (JSON):  
     ```json
     {
       "text": "={{ $json.text }}",
       "voice": "={{ $json.voice }}"
     }
     ```  
   - Authentication: HTTP Header Auth with Fal.run API key (create credential with API key)  
   - Connect output from `Set text input`.

4. **Create Wait Node for 60 seconds**  
   - Name: `Wait 60 sec.2`  
   - Delay: 60 seconds  
   - Connect output from `Create voice`.

5. **Create HTTP Request Node to Get Voice Status**  
   - Name: `Get status audio`  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/elevenlabs/requests/{{ $('Create voice').item.json.request_id }}/status`  
   - Auth: Fal.run API key  
   - Connect output from `Wait 60 sec.2`.

6. **Create If Node to Check Voice Completion**  
   - Name: `Completed audio?`  
   - Condition: `$json.status == "COMPLETED"`  
   - True output: Connect to next node to get audio URL  
   - False output: Connect back to `Wait 60 sec.2` (to retry polling)

7. **Create HTTP Request Node to Get Audio URL**  
   - Name: `Get Url Audio`  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/elevenlabs/requests/{{ $json.request_id }}`  
   - Auth: Fal.run API key  
   - Connect true output from `Completed audio?`.

8. **Create Set Node to Extract Audio URL**  
   - Name: `Set Audio Url`  
   - Assign field: `audio_url` = `{{$json.audio.url}}`  
   - Connect output from `Get Url Audio`.

9. **Create Set Node to Define Video Parameters**  
   - Name: `Set Video Params`  
   - Assign fields:  
     - `image_url`: "https://n3wstorage.b-cdn.net/n3witalia/result2.png"  
     - `audio_url`: `={{$json.audio_url}}`  
     - `prompt`: "You are a girl who makes weather forecasts and needs to be expressive"  
   - Connect output from `Set Audio Url`.

10. **Create HTTP Request Node for Video Creation**  
    - Name: `Create Video`  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/infinitalk`  
    - Body (JSON):  
      ```json
      {
        "image_url": "{{ $json.image_url }}",
        "audio_url": "{{ $json.audio_url }}",
        "prompt": "{{ $json.prompt }}"
      }
      ```  
    - Auth: Fal.run API key  
    - Connect output from `Set Video Params`.

11. **Create Wait Node for 60 seconds**  
    - Name: `Wait 60 sec.1`  
    - Delay: 60 seconds  
    - Connect output from `Create Video`.

12. **Create HTTP Request Node to Get Video Status**  
    - Name: `Get status`  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/infinitalk/requests/{{ $('Create Video').item.json.request_id }}/status`  
    - Auth: Fal.run API key  
    - Connect output from `Wait 60 sec.1`.

13. **Create If Node to Check Video Completion**  
    - Name: `Completed?`  
    - Condition: `$json.status == "COMPLETED"`  
    - True output: Connect to next node to get video URL  
    - False output: Connect back to `Wait 60 sec.1` (polling loop)

14. **Create HTTP Request Node to Get Video URL**  
    - Name: `Get Url Video`  
    - Method: GET  
    - URL: `https://queue.fal.run/fal-ai/infinitalk/requests/{{ $json.request_id }}`  
    - Auth: Fal.run API key  
    - Connect true output from `Completed?`.

15. **Create OpenAI Node to Generate Title**  
    - Name: `Generate title`  
    - Model: GPT-4o-mini  
    - Messages:  
      - User input: "Input: {{ $('Set text input').item.json.text }}\n\nPrompt: {{ $('Set Video Params').item.json.prompt }}"  
      - System instructions as per guidelines in workflow description (see Node 15 details)  
    - Credentials: OpenAI API credentials  
    - Connect output from `Get Url Video`.

16. **Create HTTP Request Node to Download Video File**  
    - Name: `Get File Video`  
    - Method: GET  
    - URL: `={{ $('Get Url Video').item.json.video.url }}`  
    - Connect output from `Generate title`.

17. **Create HTTP Request Node to Upload Video to Postiz**  
    - Name: `Upload Video to Postiz`  
    - Method: POST  
    - URL: `https://api.postiz.com/public/v1/upload`  
    - Content-Type: multipart/form-data  
    - Body Parameters: file binary data from `Get File Video` node  
    - Auth: HTTP Header Auth with Postiz API key  
    - Connect output from `Get File Video`.

18. **Create Postiz Node to Post on TikTok**  
    - Name: `TikTok`  
    - Configure post content with video ID and path from upload response  
    - Content text from `Generate title` output  
    - Set current date/time for post  
    - Set Integration ID (TikTok channel ID)  
    - Enable short link  
    - Auth: Postiz API credentials  
    - Connect output from `Upload Video to Postiz`.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                    |
|--------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| The workflow produces a talking AI avatar video from a single image using Elevenlabs and Infinitalk AI services chained. | Workflow purpose (Sticky Note)                     |
| Setup steps reminder: define text, voice, image URL, prompt, and install Postiz node for TikTok posting.                  | Sticky Note3                                       |
| API keys for Fal.run (Elevenlabs & Infinitalk) and Postiz must be configured in credentials before running.              | Sticky Notes 8, 9, 2                               |
| Example input image: ![image](https://n3wstorage.b-cdn.net/n3witalia/result2.png)                                         | Sticky Note1                                       |
| Example output video (max 5 seconds): [Download the video](https://n3wstorage.b-cdn.net/n3witalia/talking_avatar.mp4)    | Sticky Note1                                       |
| Postiz platform reference: https://postiz.com/?ref=n3witalia                                                             | Workflow purpose note                              |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---