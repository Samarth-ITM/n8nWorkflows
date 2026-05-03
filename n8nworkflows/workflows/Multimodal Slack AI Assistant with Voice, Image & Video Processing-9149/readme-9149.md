Multimodal Slack AI Assistant with Voice, Image & Video Processing

https://n8nworkflows.xyz/workflows/multimodal-slack-ai-assistant-with-voice--image---video-processing-9149


# Multimodal Slack AI Assistant with Voice, Image & Video Processing

### 1. Workflow Overview

This workflow implements a **Multimodal Slack AI Assistant** capable of processing voice, images, and video sent as Slack messages, transcribing audio, analyzing images and videos, and responding intelligently using Large Language Models (LLMs). Its primary use case is to act as an interactive AI agent within a Slack channel, interpreting different types of media input, leveraging AI services to understand content, and replying contextually.

The workflow is logically organized into these blocks:

- **1.1 Slack Input Reception:** Captures Slack messages or mentions in a specified Slack channel.
- **1.2 Message Type Determination (Switch):** Identifies the media type of the incoming Slack message (audio, picture, video, or text).
- **1.3 Media Retrieval:** Downloads the media file from Slack for audio, picture, or video inputs.
- **1.4 Media Processing:** Processes the media according to type:
  - Audio: Transcribes using OpenAI Whisper.
  - Image: Analyzes using Google Gemini.
  - Video: Analyzes using Google Gemini.
- **1.5 Input Consolidation:** Merges processed data from media or text into a unified input string.
- **1.6 AI Agent Interaction:** Sends the consolidated input to a customizable AI agent (Langchain-based) that uses Anthropic Claude or other LLMs with memory context.
- **1.7 Output Delivery:** Sends the AI-generated response back to the Slack channel.

Supporting components include session memory management for contextual conversations and date/time tools accessible to the AI.

---

### 2. Block-by-Block Analysis

#### 2.1 Slack Input Reception

- **Overview:**  
  Listens for Slack events (app mentions or messages) specifically in the configured Slack channel to trigger the workflow.

- **Nodes Involved:**  
  - Slack Trigger

- **Node Details:**

  - **Slack Trigger**  
    - Type: Slack Trigger  
    - Role: Entry point; listens for Slack events (app_mention and message) in channel `C09HD4Z93T7` (named ai-agent).  
    - Configuration: Uses Slack API credentials; webhook enabled; filter for channel and event types.  
    - Inputs: Slack event data including text, files, blocks, and metadata.  
    - Outputs: Emits incoming Slack message JSON.  
    - Failures: Slack API auth errors, webhook failures, missing permissions.  

- **Sticky Notes:**  
  - "### Waits for Slack message to start workflow"

---

#### 2.2 Message Type Determination (Switch)

- **Overview:**  
  Determines the type of message received (audio, image, video, or text) based on Slack message metadata to direct processing accordingly.

- **Nodes Involved:**  
  - Switch

- **Node Details:**

  - **Switch**  
    - Type: Switch (n8n core)  
    - Role: Routes workflow based on media type or text type extracted from Slack message JSON.  
    - Configuration:  
      - Checks `$json.files[0].media_display_type === "audio"` → voice output  
      - Checks `$json.files[0].mimetype === "image/png"` → picture output  
      - Checks `$json.files[0].mimetype === "video/mp4"` → video output  
      - Checks `$json.blocks[0].elements[0].elements[1].type === "text"` → text output  
    - Inputs: Slack Trigger output  
    - Outputs: Four distinct outputs for voice, picture, video, and text.  
    - Failures: Undefined or unexpected message structure, missing files array, unsupported media types.

- **Sticky Notes:**  
  - "### Determines what type of message was sent"

---

#### 2.3 Media Retrieval

- **Overview:**  
  Downloads the actual media file from Slack for audio, image, and video inputs, preparing it for AI processing.

- **Nodes Involved:**  
  - Get Audio File  
  - Get a Picture File  
  - Get Video File  
  - HTTP Request (for downloading files)

- **Node Details:**

  - **Get Audio File, Get a Picture File, Get Video File**  
    - Type: Slack node (file get operation)  
    - Role: Retrieves file metadata and binary data from Slack using file ID from the message.  
    - Configuration: Uses Slack API credentials; `fileId` set dynamically to `$json.files[0].id`.  
    - Inputs: Output from Switch node filtered by media type.  
    - Outputs: File metadata with `url_private` for download.  
    - Failures: Slack API auth, missing file ID, file not found.

  - **HTTP Request, HTTP Request1, HTTP Request2**  
    - Type: HTTP Request (n8n core)  
    - Role: Downloads binary file content from the `url_private` URL using Slack bearer token authorization.  
    - Configuration:  
      - URL set dynamically from Slack file metadata (`$json.url_private`).  
      - Authorization header with bearer token placeholder `[Your bot token here]` (must be replaced).  
      - Follows redirects, returns binary content.  
    - Inputs: Output from respective Get File nodes.  
    - Outputs: Binary file data passed to AI processing nodes.  
    - Failures: Invalid or missing authorization token, network/timeouts, URL invalid or expired.

- **Sticky Notes:**  
  - "### Gets Audio File" (Get Audio File)  
  - "### Gets Photo File" (Get a Picture File)  
  - "### Gets Video File" (Get Video File)  
  - "### Download File" (HTTP Request nodes)

---

#### 2.4 Media Processing

- **Overview:**  
  Processes the downloaded media using AI services to extract meaningful text or analysis.

- **Nodes Involved:**  
  - Transcribe a recording (OpenAI Whisper)  
  - Analyze an image (Google Gemini)  
  - Analyze video (Google Gemini)

- **Node Details:**

  - **Transcribe a recording**  
    - Type: Langchain OpenAI node  
    - Role: Transcribes audio to text using OpenAI Whisper API.  
    - Configuration: `resource` set to `audio`, `operation` to `transcribe`, no special options.  
    - Credentials: OpenAI API key required.  
    - Inputs: Binary audio from HTTP Request node.  
    - Outputs: Transcription text in JSON.  
    - Failures: API auth errors, unsupported audio format, transcription timeout.

  - **Analyze an image**  
    - Type: Langchain Google Gemini node  
    - Role: Analyzes image content and generates textual insights.  
    - Configuration:  
      - Text parameter passed from Slack Trigger text.  
      - Model: `models/gemini-2.5-flash`  
      - Resource: `image`  
      - Input type: `binary` (image binary data)  
      - Operation: `analyze`  
    - Credentials: Google Palm API key (Gemini).  
    - Inputs: Binary image data from HTTP Request1.  
    - Outputs: Analysis text.  
    - Failures: API quota exceeded, auth issues, binary data corrupt.

  - **Analyze video**  
    - Type: Langchain Google Gemini node  
    - Role: Analyzes video content and generates textual insights.  
    - Configuration: Similar to image analysis but with resource `video`.  
    - Credentials: Google Palm API key.  
    - Inputs: Binary video data from HTTP Request2.  
    - Outputs: Analysis text.  
    - Failures: Large file size handling, API limits, auth errors.

- **Sticky Notes:**  
  - "### Uses OpenAI to Transcribe Recording" (Transcribe a recording)  
  - "### Uses Gemini to analyze photo" (Analyze an image)  
  - "### Uses Gemini to analyze Video" (Analyze video)  
  - "### Choose any LLM you want here" (Anthropic Chat Model, AI Agent)  

---

#### 2.5 Input Consolidation

- **Overview:**  
  Merges the processed text from transcription or analysis with plain text messages into a single input for the AI agent.

- **Nodes Involved:**  
  - Merge  
  - Set "User Input" for AI Agent

- **Node Details:**

  - **Merge**  
    - Type: Merge (n8n core)  
    - Role: Combines outputs from different media processing branches and plain text into one data stream.  
    - Configuration: `numberInputs` set to 4 corresponding to voice, picture, video, and text paths.  
    - Inputs: Outputs from Transcribe a recording, Analyze an image, Analyze video, and Switch text output.  
    - Outputs: Unified JSON containing the processed input.  
    - Failures: Missing or empty inputs from any branch.

  - **Set "User Input" for AI Agent**  
    - Type: Set (n8n core)  
    - Role: Defines the `User Input` variable for the AI Agent node, selecting the first available text from various fields such as transcription, message text, analysis, or content.  
    - Configuration: Sets field `User Input` to expression:  
      `{{$json.text || $json.message?.text || $json.content || $json["data"] || $json.result || $json.transcription || ""}}`  
    - Inputs: Output from Merge node.  
    - Outputs: JSON with `User Input` string ready for AI processing.  
    - Failures: Expression errors if fields are missing.

- **Sticky Notes:**  
  - "### Creates a unified input to pass to AI Agent"

---

#### 2.6 AI Agent Interaction

- **Overview:**  
  Sends the consolidated message to a configurable AI Agent (Langchain agent) enhanced with memory and optionally Anthropic Claude model, to generate a natural language response.

- **Nodes Involved:**  
  - Simple Memory  
  - Date & Time  
  - Anthropic Chat Model  
  - AI Agent

- **Node Details:**

  - **Simple Memory**  
    - Type: Langchain memoryBufferWindow  
    - Role: Maintains conversation history contextualized by Slack channel (`sessionKey` set to channel ID).  
    - Configuration:  
      - `sessionIdType`: custom key (channel ID from Slack Trigger).  
      - `contextWindowLength`: 50 messages.  
    - Inputs: Slack Trigger node for channel info.  
    - Outputs: Memory context for AI Agent.  
    - Failures: Session key missing, memory overflow.

  - **Date & Time**  
    - Type: Date Time Tool  
    - Role: Provides current date/time context accessible by AI.  
    - Inputs: Connected as AI tool for AI Agent.  
    - Failures: N/A

  - **Anthropic Chat Model**  
    - Type: Langchain LM Chat Anthropic  
    - Role: Optional LLM backend (Claude Sonnet 4) for AI Agent.  
    - Configuration:  
      - Model: `claude-sonnet-4-20250514`  
      - Max tokens: 10000  
    - Credentials: Anthropic API key.  
    - Inputs: AI Agent node as language model.  
    - Failures: API limits, auth errors.

  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Central AI processor generating responses.  
    - Configuration:  
      - Text input from `User Input` field.  
      - System message defines behavior: instructs plain UTF-8 output for Telegram, no JSON or code fences, fallback output `"..."` if empty.  
      - Prompt type: define (custom system prompt).  
      - Connected to Simple Memory (for context), Anthropic Chat Model as language model, Date & Time tool as AI tool.  
    - Outputs: AI response string.  
    - Failures: LLM API errors, memory failures, prompt errors.

- **Sticky Notes:**  
  - "### Base AI Agent. Adjust \"system prompt\" to personalize this agent to your needs"  
  - "### Choose # of messages the LLM can read back" (Simple Memory)  
  - "### Add tools that the LLM has access to here" (Date & Time)  

---

#### 2.7 Output Delivery

- **Overview:**  
  Sends the AI-generated response text back as a message in the Slack channel.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Send a message**  
    - Type: Slack node (message send)  
    - Role: Posts AI Agent output text back to Slack channel `C09HD4Z93T7`.  
    - Configuration:  
      - Text set dynamically to AI Agent output (`$('AI Agent').item.json.output`).  
      - Channel fixed to Slack channel ID.  
      - Uses Slack API credentials.  
    - Inputs: AI Agent output.  
    - Failures: Slack API auth errors, message length limits, rate limits.

- **Sticky Notes:**  
  - "### Sends AI Agent output to you in Slack"

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                      | Input Node(s)               | Output Node(s)                 | Sticky Note                                           |
|---------------------------|----------------------------------------|------------------------------------|-----------------------------|-------------------------------|-------------------------------------------------------|
| Slack Trigger             | Slack Trigger                          | Workflow entry from Slack messages | None                        | Switch                        | ### Waits for Slack message to start workflow          |
| Switch                   | Switch (n8n core)                      | Determines message/media type      | Slack Trigger               | Get Audio File, Get a Picture File, Get Video File, Merge | ### Determines what type of message was sent            |
| Get Audio File            | Slack (file get)                       | Retrieves audio file from Slack    | Switch (voice output)       | HTTP Request                  | ### Gets Audio File                                    |
| HTTP Request             | HTTP Request                          | Downloads audio binary file        | Get Audio File              | Transcribe a recording        | ### Download File                                      |
| Transcribe a recording    | Langchain OpenAI                      | Transcribes audio to text          | HTTP Request                | Merge                        | ### Uses OpenAI to Transcribe Recording                |
| Get a Picture File        | Slack (file get)                      | Retrieves image file from Slack    | Switch (picture output)     | HTTP Request1                 | ### Gets Photo File                                    |
| HTTP Request1            | HTTP Request                          | Downloads image binary file        | Get a Picture File          | Analyze an image              | ### Download File                                      |
| Analyze an image          | Langchain Google Gemini               | Analyzes image content             | HTTP Request1               | Merge                        | ### Uses Gemini to analyze photo                        |
| Get Video File            | Slack (file get)                      | Retrieves video file from Slack    | Switch (video output)       | HTTP Request2                 | ### Gets Video File                                    |
| HTTP Request2            | HTTP Request                          | Downloads video binary file        | Get Video File              | Analyze video                | ### Download File                                      |
| Analyze video             | Langchain Google Gemini               | Analyzes video content             | HTTP Request2               | Merge                        | ### Uses Gemini to analyze Video                        |
| Merge                    | Merge (n8n core)                      | Combines processed inputs          | Switch (text output), Transcribe a recording, Analyze an image, Analyze video | Set "User Input" for AI Agent | ### Creates a unified input to pass to AI Agent         |
| Set "User Input" for AI Agent | Set (n8n core)                    | Prepares user input string for AI  | Merge                       | AI Agent                     |                                                       |
| Simple Memory             | Langchain memoryBufferWindow          | Maintains conversation memory     | Slack Trigger               | AI Agent                     | ### Choose # of messages the LLM can read back          |
| Date & Time               | Date Time Tool                       | Provides date/time tool to AI     | None                       | AI Agent                     | ### Add tools that the LLM has access to here            |
| Anthropic Chat Model      | Langchain LM Chat Anthropic           | Optional LLM backend               | AI Agent                   | AI Agent                     | ### Choose any LLM you want here                        |
| AI Agent                  | Langchain Agent                      | Main AI processor                 | Set "User Input" for AI Agent, Simple Memory, Anthropic Chat Model, Date & Time | Send a message               | ### Base AI Agent. Adjust "system prompt" to personalize this agent to your needs |
| Send a message            | Slack (message send)                   | Sends AI response to Slack channel | AI Agent                   | None                        | ### Sends AI Agent output to you in Slack               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node**  
   - Type: Slack Trigger  
   - Configure credentials with Slack bot token.  
   - Set trigger events: `app_mention`, `message`.  
   - Restrict to Slack channel ID `C09HD4Z93T7` (your target channel).  

2. **Add Switch Node**  
   - Type: Switch  
   - Connect Slack Trigger output to Switch input.  
   - Add 4 outputs with conditions:  
     - Output "voice": `$json.files[0].media_display_type === "audio"`  
     - Output "picture": `$json.files[0].mimetype === "image/png"`  
     - Output "video": `$json.files[0].mimetype === "video/mp4"`  
     - Output "text": `$json.blocks[0].elements[0].elements[1].type === "text"`  

3. **Add Get File Nodes for Each Media Type**  
   - For voice: Add Slack node, operation `get file`, configure `fileId` as `$json.files[0].id`. Connect Switch "voice" output to this node.  
   - For picture: Same as above but connected to Switch "picture" output.  
   - For video: Same as above but connected to Switch "video" output.

4. **Add HTTP Request Nodes to Download Files**  
   - For each Get File node, add HTTP Request node.  
   - Configure URL: `{{$json.url_private}}`.  
   - Add Header: `Authorization: Bearer [Your bot token here]` (replace with actual Slack bot token).  
   - Enable redirect following and binary response. Connect respective Get File node output to this node.

5. **Add Media Processing Nodes**  
   - For audio: Add Langchain OpenAI node, set resource `audio`, operation `transcribe`. Connect HTTP Request (audio) output. Configure OpenAI credentials.  
   - For image: Add Langchain Google Gemini node, model `models/gemini-2.5-flash`, resource `image`, input type `binary`, operation `analyze`. Connect HTTP Request1 output. Configure Google Palm API credentials.  
   - For video: Add Langchain Google Gemini node, same model, resource `video`, input type `binary`, operation `analyze`. Connect HTTP Request2 output. Configure Google Palm API credentials.

6. **Add Merge Node**  
   - Type: Merge  
   - Configure for 4 inputs.  
   - Connect outputs from Transcribe a recording, Analyze an image, Analyze video, and Switch "text" output (for plain text messages).  

7. **Add Set Node to Define User Input**  
   - Add Set node named `Set "User Input" for AI Agent`.  
   - Add field `User Input` (string) with expression:  
     `{{$json.text || $json.message?.text || $json.content || $json["data"] || $json.result || $json.transcription || ""}}`  
   - Connect Merge output to this node.

8. **Add Simple Memory Node**  
   - Type: Langchain memoryBufferWindow  
   - Configure `sessionKey` as Slack channel ID: `{{$('Slack Trigger').item.json.channel}}`  
   - Set `sessionIdType` to `customKey`  
   - Set `contextWindowLength` to 50

9. **Add Date & Time Node**  
   - Add Date & Time tool node (n8n core).  
   - No special config required.

10. **Add Anthropic Chat Model Node (Optional)**  
    - Type: Langchain LM Chat Anthropic  
    - Configure model: `claude-sonnet-4-20250514`  
    - Set max tokens to 10000.  
    - Add Anthropic API credentials.

11. **Add AI Agent Node**  
    - Type: Langchain Agent  
    - Configure text input from `User Input` field.  
    - Set system prompt:  
      ```
      You are a helpful, intelligent assistant.

      Output ONLY a plain UTF-8 string for Telegram. 
      No JSON. No code fences. No prefaces. 
      If content is empty, output "…".
      ```  
    - Connect `Set "User Input"` output to AI Agent.  
    - Connect Simple Memory node as AI memory.  
    - Connect Anthropic Chat Model as AI language model.  
    - Connect Date & Time node as AI tool.

12. **Add Slack Send Message Node**  
    - Type: Slack node (message send)  
    - Configure channel ID to `C09HD4Z93T7`.  
    - Set message text to `{{$('AI Agent').item.json.output}}`.  
    - Connect AI Agent output to this node.

13. **Credentials Setup**  
    - Slack API Credential: Slack bot token with proper scopes for reading messages, files, posting messages.  
    - OpenAI Credential: API key with access to Whisper and GPT models.  
    - Google Palm API Credential: API key for Google Gemini models.  
    - Anthropic API Credential: API key for Claude models if used.

14. **Testing and Validation**  
    - Test with various message types: text, audio (voice), image (PNG), video (MP4).  
    - Confirm correct routing, processing, AI response, and Slack message posting.  
    - Handle errors such as missing files, invalid media types, API limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Instructions for creating Slack bot, setting permissions, installing app, and obtaining access tokens for Slack and LLMs.                            | Provided in first Sticky Note in workflow           |
| Workflow uses Langchain integrations supporting multiple LLMs including Anthropic Claude and Google Gemini.                                           | Useful for customizing AI backend                    |
| Replace all placeholder tokens (e.g., `[Your bot token here]`) with actual Slack bot tokens in HTTP Request nodes for media download authorization.   | Critical for file downloads                           |
| Memory buffer window limits context size to 50 messages to balance performance and context relevance.                                                 | Can be adjusted in Simple Memory node                |
| System prompt in AI Agent node is crafted to produce plain text outputs suitable for Slack and Telegram environments, avoid JSON or code formatting. | Customize prompt to change AI assistant personality |
| Slack channel ID `C09HD4Z93T7` is hardcoded; adapt to your workspace channels as needed.                                                              | Change in Slack Trigger and Send a message nodes     |
| Video and image analysis uses Google Gemini via Google Palm API, requiring valid Google Cloud credentials and API enablement.                        | Ensure API quota and billing are managed             |
| Audio transcription uses OpenAI Whisper endpoint; requires OpenAI API key with transcription access.                                                  |                                                     |

---

This document provides a full technical reference for the "Multimodal Slack AI Assistant with Voice, Image & Video Processing" workflow, enabling advanced users and automation agents to understand, reproduce, and extend it effectively.