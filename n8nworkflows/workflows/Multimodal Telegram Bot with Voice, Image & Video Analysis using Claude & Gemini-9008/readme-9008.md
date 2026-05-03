Multimodal Telegram Bot with Voice, Image & Video Analysis using Claude & Gemini

https://n8nworkflows.xyz/workflows/multimodal-telegram-bot-with-voice--image---video-analysis-using-claude---gemini-9008


# Multimodal Telegram Bot with Voice, Image & Video Analysis using Claude & Gemini

### 1. Workflow Overview

This workflow implements a **Multimodal Telegram Bot** capable of processing various types of Telegram messages including text, voice recordings, images, and videos. It uses advanced AI models, specifically Anthropic Claude and Google Gemini, to analyze and respond to user inputs. The core purpose is to receive a message, identify its type, optionally transcribe or analyze media content, unify the input, and send an AI-generated reply back to the user on Telegram.

The workflow is organized into these logical blocks:

- **1.1 Input Reception and Classification:** Waits for incoming Telegram messages and determines their type (text, voice, image, video, or start command).
- **1.2 Media Retrieval and Processing:** Downloads media files from Telegram (voice, photo, video) and processes them (transcription for voice, analysis for images and videos) using AI models.
- **1.3 Input Unification:** Consolidates different input formats into a single text input for AI processing.
- **1.4 AI Reasoning:** Uses a configurable AI agent powered by Anthropic Claude and optionally Google Gemini to generate responses based on user input and conversation memory.
- **1.5 Output Delivery:** Sends the AI-generated textual response back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Classification

- **Overview:**  
  This block listens for Telegram messages and categorizes them into types for further processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Switch  
  - No Operation, do nothing  
  - Sticky Note1 (instructional)  
  - Sticky Note2 (instructional)

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Trigger node for Telegram messages.  
    - *Config:* Listens for "message" update type only. Uses Telegram API credentials.  
    - *Input/Output:* No input; outputs incoming Telegram message JSON.  
    - *Potential Failures:* Invalid webhook setup, expired credentials, Telegram API downtime.  
    - *Sticky Note1:* "Waits for telegram message to start workflow."

  - **Switch**  
    - *Type:* Routing node to classify message type.  
    - *Config:* Checks message content to route to one of: "Do Nothing" (command /start), "voice", "picture", "video", or "text". Uses strict type validation and case-sensitive matching.  
    - *Key Expressions:* Checks existence of `message.voice`, `message.photo`, `message.video` or message text content for routing.  
    - *Input:* Telegram Trigger output.  
    - *Output:* Routes to different branches depending on message type.  
    - *Sticky Note2:* "Determines what type of message was sent."  
    - *Edge Cases:* Unexpected message types, missing fields, malformed JSON could cause misrouting.

  - **No Operation, do nothing**  
    - *Type:* No-op node to handle /start command or unprocessable messages gracefully.  
    - *Input:* Switch output "Do Nothing".  
    - *Output:* None.  
    - *Edge Cases:* None (safe fallback).

---

#### 2.2 Media Retrieval and Processing

- **Overview:**  
  Downloads media files (voice, photo, video) from Telegram and processes them with AI services—OpenAI Whisper for transcription of voice, Google Gemini for image and video analysis.

- **Nodes Involved:**  
  - Get a Audio File  
  - Transcribe a recording  
  - Get a Photo  
  - Analyze an image  
  - Get a Video File  
  - Analyze video  
  - Sticky Notes 3,4,10,13,14 (instructional)

- **Node Details:**

  - **Get a Audio File**  
    - *Type:* Telegram node to download voice file.  
    - *Config:* Uses `message.voice.file_id` to fetch binary file.  
    - *Credentials:* Telegram API.  
    - *Input:* Switch output "voice".  
    - *Output:* Binary audio file.  
    - *Sticky Note3:* "Get Audio File."  
    - *Failures:* Invalid file_id, Telegram API errors, network issues.

  - **Transcribe a recording**  
    - *Type:* OpenAI Whisper transcription node.  
    - *Config:* Transcribes audio resource to text.  
    - *Credentials:* OpenAI API key.  
    - *Input:* Binary audio file from "Get a Audio File".  
    - *Output:* JSON with transcription text.  
    - *Sticky Note4:* "Use OpenAI to Transcribe Recording."  
    - *Failures:* API quota exceeded, audio format unsupported, transcription errors.

  - **Get a Photo**  
    - *Type:* Telegram node to download photo file.  
    - *Config:* Uses `message.photo[0].file_id` to fetch the highest resolution photo binary.  
    - *Credentials:* Telegram API.  
    - *Input:* Switch output "picture".  
    - *Output:* Binary image file.  
    - *Sticky Note10:* "Get Photo File."  
    - *Failures:* Missing photo array, API errors.

  - **Analyze an image**  
    - *Type:* Google Gemini AI image analysis node.  
    - *Config:* Analyzes binary image with Gemini model "models/gemini-2.5-flash". Passes message caption or text as prompt context.  
    - *Credentials:* Google Palm API key.  
    - *Input:* Binary image from "Get a Photo".  
    - *Output:* JSON with analysis results.  
    - *Sticky Note11:* "Use Gemini to analyze photo."  
    - *Failures:* API errors, unsupported image formats.

  - **Get a Video File**  
    - *Type:* Telegram node to download video file.  
    - *Config:* Uses `message.video.file_id` to fetch binary video.  
    - *Credentials:* Telegram API.  
    - *Input:* Switch output "video".  
    - *Output:* Binary video file.  
    - *Sticky Note13:* "Get Video File."  
    - *Failures:* Invalid file_id, large file size, API errors.

  - **Analyze video**  
    - *Type:* Google Gemini AI video analysis node.  
    - *Config:* Analyzes binary video with Gemini model "models/gemini-2.5-flash". Uses message caption or text as prompt context.  
    - *Credentials:* Google Palm API key.  
    - *Input:* Binary video from "Get a Video File".  
    - *Output:* JSON with analysis results.  
    - *Sticky Note14:* "Use Gemini to analyze Video."  
    - *Failures:* API timeouts for large videos, unsupported formats.

---

#### 2.3 Input Unification

- **Overview:**  
  Merges all processed inputs (transcribed text, image/video analysis, or direct text) into a single normalized text field for AI reasoning.

- **Nodes Involved:**  
  - Merge  
  - Edit Fields  
  - Sticky Note12 (instructional)

- **Node Details:**

  - **Merge**  
    - *Type:* Merge node with 4 inputs.  
    - *Config:* Collects outputs from different branches (voice transcription, image analysis, video analysis, and text messages) into a unified stream.  
    - *Input:* Outputs from Transcribe a recording, Analyze an image, Analyze video, and Switch text branch.  
    - *Output:* Combined JSON data for further processing.  
    - *Failures:* Missing inputs if any branch fails, ordering issues.

  - **Edit Fields**  
    - *Type:* Set node to create unified user input field.  
    - *Config:* Sets a string field "User Input" based on available fields in JSON: tries `text`, `message.text`, `content`, `data`, `result`, `transcription`, or empty string fallback.  
    - *Input:* From Merge.  
    - *Output:* JSON with normalized `User Input` property.  
    - *Sticky Note12:* "Creates a unified input to pass to AI Agent."  
    - *Failures:* Expression failures if fields are missing or malformed.

---

#### 2.4 AI Reasoning

- **Overview:**  
  Uses a customizable AI agent to generate responses based on the unified user input and conversational memory.

- **Nodes Involved:**  
  - Simple Memory  
  - Anthropic Chat Model  
  - AI Agent  
  - Date & Time  
  - Sticky Notes5,6,8,9 (instructional)

- **Node Details:**

  - **Simple Memory**  
    - *Type:* Langchain memory buffer window node.  
    - *Config:* Stores recent chat context with a window length of 50 messages, keyed by Telegram chat id.  
    - *Input:* Telegram chat id from Trigger node.  
    - *Output:* Provides memory context for AI Agent.  
    - *Sticky Note8:* "Choose # of messages the LLM can read back."  
    - *Failures:* Memory overflow, session key errors.

  - **Anthropic Chat Model**  
    - *Type:* Langchain LLM node using Anthropic Claude Sonnet 4.  
    - *Config:* Uses the specified Claude model with max token sampling set high (10000).  
    - *Credentials:* Anthropic API key.  
    - *Input:* From AI Agent node’s language model input.  
    - *Output:* AI-generated text response.  
    - *Sticky Note5:* "Choose any LLM you want here."  
    - *Failures:* API quota, latency, or rate limits.

  - **AI Agent**  
    - *Type:* Langchain agent node.  
    - *Config:* Receives "User Input" text and system prompt that instructs the agent to act as a helpful assistant and output plain UTF-8 strings without JSON or code fences.  
    - *Input:* Unified user input from Edit Fields, plus memory and LLM inputs.  
    - *Output:* AI-generated textual output.  
    - *Sticky Note6:* "Base AI Agent. Adjust 'System Prompt' to personalize this agent to your needs."  
    - *Failures:* Expression errors, agent misconfiguration.

  - **Date & Time**  
    - *Type:* DateTime utility node.  
    - *Config:* Provides current date/time info to AI Agent if needed.  
    - *Input/Output:* Feeds into AI Agent as an AI tool input.  
    - *Failures:* Minimal.

---

#### 2.5 Output Delivery

- **Overview:**  
  Sends the AI agent’s response back to the user on Telegram.

- **Nodes Involved:**  
  - Send Regular Message  
  - Sticky Note7 (instructional)

- **Node Details:**

  - **Send Regular Message**  
    - *Type:* Telegram message node.  
    - *Config:* Sends text from AI Agent output to chat ID from Telegram Trigger node. Does not append attribution.  
    - *Credentials:* Telegram API.  
    - *Input:* AI Agent output.  
    - *Output:* Message sent confirmation.  
    - *Sticky Note7:* "Sends AI Agent output to you in Telegram."  
    - *Failures:* Telegram API errors, chat ID invalid.

---

### 3. Summary Table

| Node Name              | Node Type                            | Functional Role                         | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                         |
|------------------------|------------------------------------|---------------------------------------|--------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger       | telegramTrigger                    | Receive Telegram messages              | -                        | Switch                   | Waits for telegram message to start workflow                                                                        |
| Switch                 | switch                            | Determine message type                 | Telegram Trigger         | No Operation, Get a Audio File, Get a Photo, Get a Video File, Merge | Determines what type of message was sent                                                                            |
| No Operation, do nothing | noOp                             | Handle /start command (do nothing)    | Switch                   | -                        |                                                                                                                     |
| Get a Audio File       | telegram                         | Download voice file                    | Switch                   | Transcribe a recording    | Get Audio File                                                                                                       |
| Transcribe a recording | openAi (Whisper)                 | Transcribe audio to text               | Get a Audio File         | Merge                    | Use OpenAI to Transcribe Recording                                                                                  |
| Get a Photo            | telegram                         | Download photo file                    | Switch                   | Analyze an image          | Get Photo File                                                                                                       |
| Analyze an image       | googleGemini                     | Analyze image content                  | Get a Photo              | Merge                    | Use Gemini to analyze photo                                                                                          |
| Get a Video File       | telegram                         | Download video file                    | Switch                   | Analyze video             | Get Video File                                                                                                       |
| Analyze video          | googleGemini                     | Analyze video content                  | Get a Video File         | Merge                    | Use Gemini to analyze Video                                                                                          |
| Merge                  | merge                           | Combine all processed inputs           | Transcribe a recording, Analyze an image, Analyze video, Switch(text) | Edit Fields              |                                                                                                                     |
| Edit Fields            | set                             | Create unified "User Input" field      | Merge                    | AI Agent                 | Creates a unified input to pass to AI Agent                                                                          |
| Simple Memory          | memoryBufferWindow              | Maintain conversation memory           | Telegram Trigger         | AI Agent                 | Choose # of messages the LLM can read back                                                                           |
| Anthropic Chat Model   | lmChatAnthropic                | LLM backend for AI Agent               | AI Agent                 | AI Agent                 | Choose any LLM you want here                                                                                        |
| AI Agent               | langchain.agent                | Core AI reasoning and response         | Edit Fields, Simple Memory, Anthropic Chat Model, Date & Time | Send Regular Message      | Base AI Agent. Adjust "System Prompt" to personalize this agent to your needs                                       |
| Date & Time            | dateTimeTool                   | Provide current date/time info         | -                        | AI Agent                 |                                                                                                                     |
| Send Regular Message   | telegram                       | Send AI-generated reply on Telegram   | AI Agent                 | -                        | Sends AI Agent output to you in Telegram                                                                             |
| Sticky Notes 1 to 14   | stickyNote                     | Instructional comments                  | -                        | -                        | Various notes as described in node analysis sections above                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Use BotFather on Telegram to create a bot and obtain the API token.  
   - In n8n, create Telegram API credentials with this token.

2. **Create Telegram Trigger Node**  
   - Add `telegramTrigger` node.  
   - Set "Updates" to `"message"`.  
   - Attach Telegram API credentials.  
   - Position at workflow start.

3. **Add Switch Node to Classify Message Types**  
   - Add `switch` node connected from Telegram Trigger.  
   - Create rules to detect:  
     - `/start` command → "Do Nothing" output  
     - Existence of `message.voice` → "voice" output  
     - Existence of `message.photo` → "picture" output  
     - Existence of `message.video` → "video" output  
     - Otherwise → "text" output.

4. **Handle /start Command with No Operation**  
   - Add `noOp` node connected from Switch "Do Nothing" output.

5. **Set Up Voice Message Processing Branch**  
   - Add `telegram` node ("Get a Audio File") connected from Switch "voice".  
     - Configure to download file using `message.voice.file_id`.  
     - Use Telegram credentials.  
   - Add OpenAI node ("Transcribe a recording") connected next.  
     - Set resource to "audio", operation to "transcribe".  
     - Use OpenAI API credentials.

6. **Set Up Image Processing Branch**  
   - Add `telegram` node ("Get a Photo") connected from Switch "picture".  
     - Download photo using `message.photo[0].file_id`.  
     - Use Telegram credentials.  
   - Add Google Gemini node ("Analyze an image").  
     - Set resource to "image", modelId to "models/gemini-2.5-flash".  
     - Use Google Palm API credentials.

7. **Set Up Video Processing Branch**  
   - Add `telegram` node ("Get a Video File") connected from Switch "video".  
     - Download video using `message.video.file_id`.  
     - Use Telegram credentials.  
   - Add Google Gemini node ("Analyze video").  
     - Set resource to "video", modelId to "models/gemini-2.5-flash".  
     - Use Google Palm API credentials.

8. **Set Up Text Message Branch**  
   - Connect Switch "text" output directly to the Merge node in step 9.

9. **Merge All Processing Branches**  
   - Add `merge` node configured for 4 inputs.  
   - Connect outputs of:  
     - Transcribe a recording  
     - Analyze an image  
     - Analyze video  
     - Switch "text" output branch  
   - This node consolidates all data.

10. **Create Unified User Input Field**  
    - Add `set` node ("Edit Fields") connected after Merge.  
    - Add assignment:  
      - Name: "User Input"  
      - Value: Expression to check multiple fields in priority:  
        ```
        {{$json.text || $json.message?.text || $json.content || $json["data"] || $json.result || $json.transcription || ""}}
        ```

11. **Configure AI Agent Components**  
    - Add `memoryBufferWindow` node ("Simple Memory").  
      - Set session key to:  
        ```
        {{$('Telegram Trigger').item.json.message.chat.id}}
        ```  
      - Context window length: 50  
    - Add `lmChatAnthropic` node ("Anthropic Chat Model").  
      - Model: "claude-sonnet-4-20250514"  
      - Max tokens to sample: 10000  
      - Use Anthropic API credentials.  
    - Add `dateTimeTool` node ("Date & Time") for current date/time.  
    - Add `langchain.agent` node ("AI Agent").  
      - Text input: `{{$json["User Input"]}}`  
      - System message:  
        ```
        You are a helpful, intelligent assistant.

        Output ONLY a plain UTF-8 string for Telegram. 
        No JSON. No code fences. No prefaces. 
        If content is empty, output "…".
        ```  
      - Connect AI Agent inputs:  
        - `ai_memory` from Simple Memory  
        - `ai_languageModel` from Anthropic Chat Model  
        - `ai_tool` from Date & Time  
      - Connect unified input from Edit Fields as main input.

12. **Send AI Response Back to Telegram**  
    - Add `telegram` node ("Send Regular Message").  
    - Configure text: `{{$('AI Agent').item.json.output}}`.  
    - Set chatId: `{{$('Telegram Trigger').item.json.message.chat.id}}`.  
    - Use Telegram credentials.  
    - Connect from AI Agent output.

13. **Add Instructional Sticky Notes**  
    - Add sticky notes near each logical block as per the content in the original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| Create your Telegram bot using BotFather, generate access token, and add it to n8n credentials.                                                                                 | Instructions in Sticky Note at workflow start.                                                                                                |
| Generate API tokens for your chosen LLMs (Anthropic Claude, OpenAI Whisper, Google Gemini). Purchase credits if needed.                                                        | LLM setup instructions in Sticky Note.                                                                                                       |
| Adjust the AI Agent’s system prompt to customize assistant personality and output format.                                                                                       | Sticky Note6 in workflow.                                                                                                                     |
| The workflow supports multimodal inputs: voice (transcribed by OpenAI Whisper), images/videos analyzed by Google Gemini.                                                      | Useful for building versatile Telegram bots with rich media support.                                                                         |
| For detailed LLM API documentation, visit Anthropic: https://www.anthropic.com/index/ or Google Palm API docs.                                                                  | External LLM provider docs.                                                                                                                   |
| The workflow uses Langchain nodes within n8n for advanced AI orchestration, enabling memory and agent capabilities.                                                             | See Langchain integration docs at https://docs.n8n.io/integrations/agents/                                                                     |
| This workflow is a solid foundation for building intelligent, multimodal conversational agents on Telegram.                                                                    | Can be extended with additional tools or customized LLM prompts.                                                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.