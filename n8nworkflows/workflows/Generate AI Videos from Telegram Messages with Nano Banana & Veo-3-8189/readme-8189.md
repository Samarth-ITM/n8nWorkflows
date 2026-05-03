Generate AI Videos from Telegram Messages with Nano Banana & Veo-3

https://n8nworkflows.xyz/workflows/generate-ai-videos-from-telegram-messages-with-nano-banana---veo-3-8189


# Generate AI Videos from Telegram Messages with Nano Banana & Veo-3

### 1. Workflow Overview

This workflow automates the generation of AI-powered videos and images from Telegram messages, supporting both text and audio inputs. Upon receiving a Telegram message (text or voice), it transcribes audio if needed, generates descriptive prompts via an AI agent, and produces an image thumbnail and an 8-second video based on these prompts. The final media files are then sent back to the Telegram chat.

Logical blocks:

- **1.1 Input Reception and Message Type Handling:** Listening for Telegram messages, distinguishing between text and audio.
- **1.2 Audio Transcription:** Downloading voice messages and transcribing them to text.
- **1.3 Prompt Generation:** Using an AI agent to create structured prompts for image and video generation.
- **1.4 Image Generation and Delivery:** Generating a thumbnail image using Nano Banana API and sending it on Telegram.
- **1.5 Video Generation and Delivery:** Creating a video via Veo-3 API, polling for completion, downloading, and sending it on Telegram.
- **1.6 Wait and Polling Mechanisms:** Implementing delays to wait for asynchronous video/image processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Message Type Handling

- **Overview:**  
  This block triggers on new Telegram messages and routes the flow based on whether the message is an audio voice note or a text message.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Switch

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point, listens for new Telegram messages (updates type: "message").  
    - *Config:* Downloads disabled (download: false), uses configured Telegram credentials.  
    - *Input/Output:* No input; outputs message data.  
    - *Edge Cases:* Fails if Telegram API credentials expire or network issues arise.  
    - *Sticky Note:* "Trigger when the Bot gets a Text message or audio message"

  - **Switch**  
    - *Type:* Switch  
    - *Role:* Branches flow into two paths — audio voice messages or text messages.  
    - *Config:*  
      - Output "audio" if `message.voice.mime_type` contains "audio".  
      - Output "text" if `message.text` exists.  
    - *Input/Output:* Input from Telegram Trigger; outputs to either "Download Audio1" node (audio path) or "Text Message" node (text path).  
    - *Edge Cases:* Message may have neither text nor audio, which is not handled explicitly and may cause the workflow to stop.  
    - *Sticky Note:* "Get the Text Message or transcribe the audio"

#### 1.2 Audio Transcription

- **Overview:**  
  This block downloads voice messages from Telegram and transcribes them into text using OpenAI's audio transcription.

- **Nodes Involved:**  
  - Download Audio1  
  - Transcribe a recording  
  - Transcribed Audio

- **Node Details:**

  - **Download Audio1**  
    - *Type:* Telegram node (file resource)  
    - *Role:* Downloads the voice message file using Telegram file ID.  
    - *Config:* Uses Telegram credentials.  
    - *Input/Output:* Input from Switch (audio path); outputs audio file binary data.  
    - *Edge Cases:* File ID invalid or expired; Telegram API errors.

  - **Transcribe a recording**  
    - *Type:* OpenAI audio transcription node (@n8n/n8n-nodes-langchain.openAi)  
    - *Role:* Transcribes audio to text.  
    - *Config:* Operation set to "transcribe" on audio resource, using OpenAI credentials.  
    - *Input/Output:* Input from Download Audio1; output is transcription text.  
    - *Edge Cases:* Audio format unsupported, transcription API rate limits/errors.

  - **Transcribed Audio**  
    - *Type:* Set node  
    - *Role:* Sets the transcription text into `text` property for downstream use.  
    - *Config:* Assigns `text = $json.text` from transcription output.  
    - *Input/Output:* Input from Transcribe a recording; outputs with standardized `text` field.

#### 1.3 Prompt Generation

- **Overview:**  
  Generates structured prompts for image and video creation from the received or transcribed text using an AI agent with a system prompt tailored for this purpose.

- **Nodes Involved:**  
  - Text Message  
  - AI Agent  
  - Structured Output Parser  
  - OpenAI Chat Model

- **Node Details:**

  - **Text Message**  
    - *Type:* Set node  
    - *Role:* For text messages, initializes `text` property with the message text. If audio was transcribed, this node is bypassed.  
    - *Config:* Sets `text` to empty string initially; actual text comes from Switch "text" output path.  
    - *Input/Output:* Input from Switch (text path); output `text` for AI Agent.

  - **AI Agent**  
    - *Type:* Langchain Agent node (@n8n/n8n-nodes-langchain.agent)  
    - *Role:* Receives text, prompts the AI to generate JSON output with two keys: `promptImage` and `promptVideo`.  
    - *Config:*  
      - System message instructs the agent to create prompts for an image and an 8-second video based on user input.  
      - Prompt type set to "define" with output parser enabled.  
      - Input expression: `text` property.  
    - *Input/Output:* Input from Text Message or Transcribed Audio; outputs structured JSON prompts.  
    - *Edge Cases:* AI model errors, malformed response, or missing fields.  
    - *Sub-workflow:* Uses OpenAI Chat Model and Structured Output Parser internally.

  - **Structured Output Parser**  
    - *Type:* Langchain Output Parser node (@n8n/n8n-nodes-langchain.outputParserStructured)  
    - *Role:* Parses AI agent output ensuring JSON structure matches example schema.  
    - *Config:* Example schema requires keys `promptImage` and `promptVideo` with string values.  
    - *Input/Output:* Connected internally to AI Agent node’s output parser slot.

  - **OpenAI Chat Model**  
    - *Type:* Langchain Chat Model node (@n8n/n8n-nodes-langchain.lmChatOpenAi)  
    - *Role:* Underlying language model used by AI Agent; model set to "gpt-4.1-mini".  
    - *Config:* Uses OpenAI credentials.  
    - *Input/Output:* Connected internally by AI Agent node.

  - *Sticky Note:* "Prompt Generation for Video and Image"

#### 1.4 Image Generation and Delivery

- **Overview:**  
  Generates a thumbnail image using the Nano Banana API, waits for image creation, downloads it, and sends it as a photo back on Telegram.

- **Nodes Involved:**  
  - Image Gen (Nano Banana)  
  - Wait1  
  - Get Image  
  - Image created (If node)  
  - Download Image  
  - Telegram: Send Photo

- **Node Details:**

  - **Image Gen (Nano Banana)**  
    - *Type:* HTTP Request node  
    - *Role:* Sends POST request to Nano Banana API to create an image based on `promptImage` from AI Agent output.  
    - *Config:*  
      - URL: https://api.kie.ai/api/v1/playground/createTask  
      - Method: POST  
      - JSON body includes model "google/nano-banana", prompt text, and output format "png".  
      - Authorization Bearer token required (redacted).  
      - Timeout set to 30 seconds.  
    - *Input/Output:* Input from AI Agent; outputs taskId for image creation.  
    - *Edge Cases:* API errors, auth failure, prompt empty or invalid.

  - **Wait1**  
    - *Type:* Wait node  
    - *Role:* Pauses workflow for 12 seconds to allow image generation to complete asynchronously.  
    - *Input/Output:* Input from Image Gen; output to Get Image.

  - **Get Image**  
    - *Type:* HTTP Request node  
    - *Role:* Queries image generation status/result using taskId from previous image generation response.  
    - *Config:*  
      - URL: https://api.kie.ai/api/v1/playground/recordInfo  
      - Uses query parameter taskId.  
      - Includes Authorization header.  
    - *Input/Output:* Input from Wait1; outputs status code and result URL.

  - **Image created (If node)**  
    - *Type:* If node  
    - *Role:* Checks if image generation succeeded by testing if HTTP response code equals 200.  
    - *Input/Output:* Input from Get Image; on true, continues to Download Image and Wait1; on false, stops or errors.

  - **Download Image**  
    - *Type:* HTTP Request node  
    - *Role:* Downloads the generated image using the result URL.  
    - *Input/Output:* Input from Image created (true branch); outputs binary image data to Telegram.

  - **Telegram: Send Photo**  
    - *Type:* Telegram node  
    - *Role:* Sends the downloaded image as a photo to Telegram chat with caption "✅ Thumbnail generated. Video is rendering...".  
    - *Config:*  
      - chatId hardcoded to 8388894079 (likely the intended recipient).  
      - Binary data enabled.  
      - Uses Telegram API credentials.  
    - *Input/Output:* Input from Download Image; outputs confirmation.

  - *Sticky Note:* "Image Generation"

#### 1.5 Video Generation and Delivery

- **Overview:**  
  Generates an 8-second video from the AI-generated video prompt using the Veo-3 API, polls for completion, downloads the video, and sends it back on Telegram.

- **Nodes Involved:**  
  - Generate Video  
  - Wait  
  - Get Video  
  - Video created (If node)  
  - Download Video  
  - Telegram: Send Video

- **Node Details:**

  - **Generate Video**  
    - *Type:* HTTP Request node  
    - *Role:* Sends POST request to Veo-3 API to start video generation with prompt from AI Agent.  
    - *Config:*  
      - URL: https://api.kie.ai/api/v1/veo/generate  
      - Method: POST  
      - JSON body includes prompt, model "veo3_fast", callback URL (localhost webhook), aspect ratio 16:9, seed 12345, fallback disabled.  
      - Authorization Bearer token required.  
      - Note: "Conent-Type" header misspelled, may cause issues.  
    - *Input/Output:* Input from Download Image node; outputs taskId for video generation.  
    - *Edge Cases:* API rejection, auth failure, invalid prompt, callback URL unreachable.

  - **Wait**  
    - *Type:* Wait node  
    - *Role:* Waits 120 seconds for video processing to complete before polling status.  
    - *Input/Output:* Input from Generate Video and from Download Video; outputs to Get Video.

  - **Get Video**  
    - *Type:* HTTP Request node  
    - *Role:* Polls the Veo-3 API for video generation status and result URL using taskId.  
    - *Config:*  
      - URL: https://api.kie.ai/api/v1/veo/get-1080p-video  
      - Query parameter: taskId from Generate Video node.  
      - Authorization header included.  
    - *Input/Output:* Input from Wait node; outputs status code and video result URL.

  - **Video created (If node)**  
    - *Type:* If node  
    - *Role:* Checks if video generation succeeded by testing if HTTP response code equals 200.  
    - *Input/Output:* Input from Get Video; on true, continues to Download Video and Wait node; on false, workflow may stop or retry.

  - **Download Video**  
    - *Type:* HTTP Request node  
    - *Role:* Downloads the completed video file from result URL.  
    - *Input/Output:* Input from Video created (true branch); outputs binary video data.

  - **Telegram: Send Video**  
    - *Type:* Telegram node  
    - *Role:* Sends the downloaded video to the Telegram chat with caption including video duration.  
    - *Config:*  
      - chatId hardcoded to 8388894079.  
      - Uses Telegram API credentials.  
      - Caption dynamically includes video duration from JSON.  
    - *Input/Output:* Input from Download Video; outputs confirmation.

  - *Sticky Note:* "Generate Video"

#### 1.6 Wait and Polling Mechanisms

- **Overview:**  
  Implements waiting periods to allow asynchronous image and video generation tasks to complete before polling their status.

- **Nodes Involved:**  
  - Wait1 (12 seconds)  
  - Wait (120 seconds)

- **Node Details:**

  - **Wait1**  
    - *Type:* Wait node  
    - *Role:* Waits 12 seconds after image generation request before polling image status.  
    - *Input/Output:* From Image Gen (Nano Banana) to Get Image.

  - **Wait**  
    - *Type:* Wait node  
    - *Role:* Waits 120 seconds after video generation request before polling video status.  
    - *Input/Output:* From Generate Video and Download Video to Get Video.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                         | Input Node(s)                 | Output Node(s)                | Sticky Note                                                 |
|---------------------|----------------------------------|---------------------------------------|------------------------------|------------------------------|-------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger                 | Listens for Telegram messages         |                              | Switch                       | Trigger when the Bot gets a Text message or audio message  |
| Switch              | Switch                          | Routes to audio or text processing     | Telegram Trigger             | Download Audio1, Text Message | Get the Text Message or transcribe the audio                |
| Download Audio1      | Telegram (file resource)          | Downloads voice message audio file     | Switch (audio path)          | Transcribe a recording        |                                                             |
| Transcribe a recording | OpenAI Audio Transcription      | Transcribes audio to text               | Download Audio1              | Transcribed Audio             |                                                             |
| Transcribed Audio    | Set                            | Sets transcribed text to `text` field | Transcribe a recording       | AI Agent                     |                                                             |
| Text Message         | Set                            | Sets text message to `text` field      | Switch (text path)           | AI Agent                     |                                                             |
| AI Agent            | Langchain Agent                 | Generates prompts for image and video  | Text Message, Transcribed Audio | Image Gen (Nano Banana)       | Prompt Generation for Video and Image                        |
| Structured Output Parser | Langchain Output Parser       | Parses AI agent JSON output             | (Internal to AI Agent)       | AI Agent                     |                                                             |
| OpenAI Chat Model    | Langchain Chat Model            | Provides language model for AI Agent   | (Internal to AI Agent)       | AI Agent                     |                                                             |
| Image Gen (Nano Banana) | HTTP Request                  | Requests image generation               | AI Agent                    | Wait1                        | Image Generation                                            |
| Wait1                | Wait                           | Waits 12 seconds for image to be ready | Image Gen (Nano Banana)      | Get Image                    |                                                             |
| Get Image            | HTTP Request                   | Polls image generation status           | Wait1                       | Image created (If)           |                                                             |
| Image created        | If                             | Checks if image generation succeeded   | Get Image                   | Download Image, Wait1        |                                                             |
| Download Image       | HTTP Request                   | Downloads generated image               | Image created (true)         | Telegram: Send Photo, Generate Video |                                                             |
| Telegram: Send Photo | Telegram                       | Sends image thumbnail to Telegram       | Download Image              | Generate Video               |                                                             |
| Generate Video       | HTTP Request                   | Requests video generation                | Download Image              | Wait                        | Generate Video                                              |
| Wait                 | Wait                          | Waits 120 seconds for video to be ready | Generate Video, Download Video | Get Video                   |                                                             |
| Get Video            | HTTP Request                  | Polls video generation status            | Wait                       | Video created (If)           |                                                             |
| Video created        | If                            | Checks if video generation succeeded    | Get Video                   | Download Video, Wait         |                                                             |
| Download Video       | HTTP Request                  | Downloads generated video                | Video created (true)         | Telegram: Send Video         |                                                             |
| Telegram: Send Video | Telegram                      | Sends video to Telegram chat             | Download Video              | Wait                        |                                                             |
| Wait                 | Wait                          | See above; also loops back after sending video | Telegram: Send Video, Generate Video | Get Video                   |                                                             |
| Sticky Note          | Sticky Note                   | Notes/Comments                          |                              |                              | Multiple sticky notes covering logical blocks               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates, do not download files automatically.  
   - Credentials: Connect your Telegram account credentials.

2. **Add a Switch node** connected from Telegram Trigger output  
   - Configure two outputs:  
     - "audio": if `message.voice.mime_type` contains "audio"  
     - "text": if `message.text` exists  

3. **Audio Path Setup** (from "audio" output):  
   1. Add Telegram node "Download Audio1"  
      - Download file resource using `message.voice.file_id`  
      - Credentials: Telegram account.  
   2. Connect to OpenAI Audio Transcription node "Transcribe a recording"  
      - Operation: "transcribe" on audio resource  
      - Credentials: OpenAI API.  
   3. Connect to Set node "Transcribed Audio"  
      - Set property `text` = transcription output text.

4. **Text Path Setup** (from "text" output):  
   - Add Set node "Text Message"  
     - Set property `text` = incoming message text.

5. **Merge both paths to AI Agent node**  
   - Add Langchain Agent node "AI Agent"  
   - Parameters:  
     - Input text: `text` from previous node  
     - System message: instruct agent to produce JSON with `promptImage` and `promptVideo` describing image and 8-second video prompts.  
     - Enable output parser with example JSON schema.  
   - Credentials: OpenAI API.

6. **Setup AI Agent internals:**  
   - Connect internal Langchain Chat Model node using model "gpt-4.1-mini" with OpenAI credentials.  
   - Connect internal Structured Output Parser node with example JSON schema.

7. **Image Generation Branch:**  
   1. Add HTTP Request node "Image Gen (Nano Banana)"  
      - POST to `https://api.kie.ai/api/v1/playground/createTask`  
      - JSON body: model "google/nano-banana", prompt `promptImage` from AI Agent output, output format "png"  
      - Headers: Authorization Bearer token, Content-Type application/json  
      - Timeout: 30 seconds  
   2. Connect to Wait node "Wait1" set to wait 12 seconds.  
   3. Connect to HTTP Request node "Get Image"  
      - GET `https://api.kie.ai/api/v1/playground/recordInfo`  
      - Query param: taskId from previous response  
      - Header: Authorization Bearer token  
   4. Add If node "Image created"  
      - Condition: check if response code == 200  
      - True branch:  
        - HTTP Request node "Download Image" to download image from `result_url`  
        - Telegram node "Telegram: Send Photo" to send photo with caption "✅ Thumbnail generated. Video is rendering..." (chatId hardcoded or dynamic)  
      - False branch: handle error or stop.

8. **Video Generation Branch:**  
   1. From "Download Image" node, connect to HTTP Request node "Generate Video"  
      - POST to `https://api.kie.ai/api/v1/veo/generate`  
      - JSON body includes:  
        - prompt: `promptVideo` from AI Agent output  
        - model: "veo3_fast"  
        - callbackUrl: your webhook URL (e.g., localhost webhook)  
        - aspectRatio: "16:9"  
        - seeds: 12345  
        - enableFallback: false  
      - Headers: Authorization Bearer token, Content-Type application/json (fix any typos)  
   2. Connect to Wait node "Wait" set to 120 seconds.  
   3. Connect to HTTP Request node "Get Video"  
      - GET `https://api.kie.ai/api/v1/veo/get-1080p-video`  
      - Query param: taskId from Generate Video node  
      - Header: Authorization Bearer token  
   4. Add If node "Video created"  
      - Condition: response code == 200  
      - True branch:  
        - HTTP Request node "Download Video" downloads video from `result_url`  
        - Telegram node "Telegram: Send Video" sends video with caption including duration (e.g., "🎬 Here is your Veo-3 video ({{ $json.duration }}s)")  
      - False branch: handle retry or stop.

9. **Set up credentials:**  
   - Telegram API OAuth2 or bot token credentials.  
   - OpenAI API key for transcription and chat model.  
   - KIE.ai API tokens for Nano Banana and Veo-3 services.

10. **Webhook and Permissions:**  
    - Ensure webhook URLs for Telegram and local callback URLs are reachable and correctly configured.  
    - Verify all API keys and tokens are valid and have necessary permissions.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                         |
|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| The workflow uses Nano Banana for image generation and Veo-3 for 8-second video generation.       | APIs: https://api.kie.ai                                                  |
| OpenAI GPT-4.1-mini model powers prompt generation and audio transcription.                       | OpenAI API documentation                                                |
| Telegram chatId is hardcoded as `8388894079`; update this to your target chat or make dynamic.    | Telegram Bot API                                                        |
| The video generation node has a typo in header "Conent-Type" which should be "Content-Type".     | May cause HTTP 400 errors; verify header spelling.                     |
| The workflow includes waiting nodes to handle API processing delays; adjust wait times if needed.| Wait times: 12s for image, 120s for video                              |
| Example AI prompt output JSON schema:                                                            | `{ "promptImage": "...", "promptVideo": "..." }`                       |
| Sticky notes in workflow provide contextual guidance on each block's purpose.                     | Visible in the n8n editor for user assistance                          |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.