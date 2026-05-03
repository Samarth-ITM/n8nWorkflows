Create LinkedIn Content from Telegram Voice/Text using Whisper, GPT-4 & Dumpling AI

https://n8nworkflows.xyz/workflows/create-linkedin-content-from-telegram-voice-text-using-whisper--gpt-4---dumpling-ai-8563


# Create LinkedIn Content from Telegram Voice/Text using Whisper, GPT-4 & Dumpling AI

---
### 1. Workflow Overview

This workflow automates the creation of LinkedIn content from Telegram inputs, supporting both voice messages and text. It leverages AI models and external APIs to transcribe audio, generate ideas and context, create an image prompt, perform web research, generate a visual, and finally save the output to Airtable while confirming completion via Telegram.

Logical blocks of the workflow:

- **1.1 Input Reception & Type Detection:** Receive Telegram messages (voice or text) and determine the message type.
- **1.2 Audio Processing:** Download voice messages and transcribe audio to text using OpenAI Whisper.
- **1.3 Text Preparation:** Extract or prepare text input for AI processing.
- **1.4 AI Content Generation:** Use GPT-4 and an AI agent to research, generate content ideas, context, and an image prompt.
- **1.5 Prompt Cleaning & Image Generation:** Clean the generated image prompt and create an image using Dumpling AI.
- **1.6 Output Formatting & Saving:** Format the AI output and save ideas, context, and image URL to Airtable.
- **1.7 Confirmation Notification:** Send a confirmation message back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Type Detection

- **Overview:** Receives incoming Telegram messages via webhook and identifies if the message is text or a voice/audio message to route processing accordingly.

- **Nodes Involved:**  
  - Receive Telegram Input  
  - Check Message Type  

- **Node Details:**

  - **Receive Telegram Input**  
    - Type: Telegram Trigger  
    - Role: Entry point receiving any Telegram message updates (only "message" update type).  
    - Configuration: Uses a Telegram API credential to listen for new messages.  
    - Input/Output: No input; outputs message JSON to "Check Message Type".  
    - Failure Types: Webhook misconfiguration, credential expiration, Telegram API limits.

  - **Check Message Type**  
    - Type: Switch  
    - Role: Routes flow based on whether the message contains text or an audio voice message.  
    - Configuration:  
      - Checks if message.text exists → routes to text path.  
      - Checks if message.voice.mime_type equals "audio/ogg" → routes to audio path.  
    - Input: Message JSON from Telegram.  
    - Output: Two branches: "text" and "audio".  
    - Edge Cases: Messages without text or voice; unsupported voice formats; malformed JSON.

#### 2.2 Audio Processing

- **Overview:** Downloads the voice message file from Telegram and transcribes it to text using OpenAI Whisper.

- **Nodes Involved:**  
  - Download Telegram Audio  
  - Transcribe Audio to Text  
  - Prepare Transcribed Text  

- **Node Details:**

  - **Download Telegram Audio**  
    - Type: Telegram  
    - Role: Downloads the audio file using the file_id from the voice message.  
    - Configuration: Uses Telegram API credential; fileId taken from message.voice.file_id.  
    - Input: Audio message info from "Check Message Type" audio branch.  
    - Output: Binary audio file for transcription.  
    - Failure Types: File not found, Telegram API errors, network issues.

  - **Transcribe Audio to Text**  
    - Type: OpenAI Audio (Whisper)  
    - Role: Converts audio file to text transcription.  
    - Configuration: Uses OpenAI Whisper transcription with OpenAI API credential.  
    - Input: Binary audio from previous node.  
    - Output: JSON with transcribed text in `.text` field.  
    - Failure Types: Audio format unsupported, OpenAI API rate limits/errors, timeout.

  - **Prepare Transcribed Text**  
    - Type: Set  
    - Role: Sets a new field "text" with the transcribed text for downstream nodes.  
    - Configuration: Assigns `text = {{$json.text}}`.  
    - Input: Transcription JSON.  
    - Output: JSON containing `text` property.  
    - Edge Cases: Empty or inaccurate transcription.

#### 2.3 Text Preparation

- **Overview:** Extracts the text message directly from Telegram when the message is text-based.

- **Nodes Involved:**  
  - Extract Text Message  

- **Node Details:**

  - **Extract Text Message**  
    - Type: Set  
    - Role: Extracts the `message.text` field and assigns it to `text` for downstream AI processing.  
    - Configuration: Assigns `text = {{$json.message.text}}`.  
    - Input: Telegram message JSON from "Check Message Type" text branch.  
    - Output: JSON with a `text` property.  
    - Edge Cases: Empty text messages or messages with unsupported content.

#### 2.4 AI Content Generation

- **Overview:** Uses LangChain agent powered by GPT-4 and Dumpling AI tools to research the input idea, generate supporting context, and create an image prompt. It integrates web search and image generation APIs.

- **Nodes Involved:**  
  - Generate Idea, Context & Image Prompt  
  - Simple Memory  
  - Web_search_tool  
  - GPT-4 Chat Model  
  - Clean Image Prompt  
  - Image_generation_tool  
  - Format Output as JSON  

- **Node Details:**

  - **Generate Idea, Context & Image Prompt**  
    - Type: LangChain Agent  
    - Role: Central AI agent that orchestrates research, idea/context generation, image prompt creation, and calls external tools.  
    - Configuration:  
      - Input: user input from the `text` field.  
      - System prompt guides the agent to: research ideas (via Dumpling AI web search), extract key points, generate a concise idea and context, craft an image prompt, and generate a professional LinkedIn-ready image.  
      - Calls these tools in sequence: Web_search_tool → Clean Image Prompt → Image_generation_tool.  
      - Enforces output format as a JSON object with `ideas`, `context`, and `image` URL.  
    - Input: JSON with `text` from previous blocks.  
    - Output: JSON object with AI-generated content.  
    - Edge Cases: API call failures, empty search results, malformed AI output, prompt cleaning errors.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational context/session memory keyed by Telegram username to provide contextual continuity in AI interactions.  
    - Configuration: Uses Telegram username as session key.  
    - Input: Connected as AI memory in the agent node.  
    - Output: Memory context for agent.  
    - Edge Cases: Missing username, session ID collisions.

  - **Web_search_tool**  
    - Type: HTTP Request Tool  
    - Role: Performs web research by querying Dumpling AI news search API with the refined query from the agent.  
    - Configuration: POST request with JSON body containing `"query"` from AI. Uses Dumpling AI API key header authentication.  
    - Input: Query string from agent.  
    - Output: Search results JSON for agent to parse.  
    - Edge Cases: API rate limits, network errors, empty results.

  - **GPT-4 Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Optional GPT-4 model used in the agent for chat completions.  
    - Configuration: Model set to "gpt-4.1-mini" with OpenAI API credentials.  
    - Input/Output: Integrated within agent node.  
    - Edge Cases: API errors, rate limits.

  - **Clean Image Prompt**  
    - Type: LangChain Tool Code  
    - Role: Cleans the generated image prompt by removing newlines and collapsing spaces to produce a single line prompt for image generation.  
    - Configuration: JavaScript code removes newline and line separator characters, trims spaces.  
    - Input: Raw prompt string from agent.  
    - Output: Cleaned prompt string.  
    - Edge Cases: Empty or malformed prompt.

  - **Image_generation_tool**  
    - Type: HTTP Request Tool  
    - Role: Calls Dumpling AI image generation API with cleaned prompt to produce a LinkedIn-ready image.  
    - Configuration: POST JSON body specifying model "FLUX.1-pro" and image parameters (size 1024x1024, guidance 3, etc.). Uses Dumpling AI API key header.  
    - Input: Cleaned prompt from previous node.  
    - Output: URL of generated image.  
    - Edge Cases: API failures, invalid prompt, rate limiting.

  - **Format Output as JSON**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses and validates the agent’s final output into a strict JSON format as per the defined schema (`ideas`, `context`, `image`).  
    - Configuration: JSON schema example provided.  
    - Input: Raw agent output JSON string.  
    - Output: Parsed and structured JSON object.  
    - Edge Cases: Parsing errors, invalid JSON output.

#### 2.5 Output Formatting & Saving

- **Overview:** Saves the finalized content to Airtable with fields for ideas, context, and image URL.

- **Nodes Involved:**  
  - Save to Airtable  

- **Node Details:**

  - **Save to Airtable**  
    - Type: Airtable  
    - Role: Creates a new record in a specified Airtable base and table with the AI-generated content fields.  
    - Configuration: Uses Airtable personal access token credential; maps fields `ideas`, `context`, and `Image URL` from AI output.  
    - Input: JSON object from AI output parser.  
    - Output: Success confirmation for downstream notification.  
    - Failure Types: API key invalid, base/table permissions, rate limits, field mapping errors.

#### 2.6 Confirmation Notification

- **Overview:** Sends a Telegram message confirming successful content creation and storage.

- **Nodes Involved:**  
  - Send Confirmation to Telegram  

- **Node Details:**

  - **Send Confirmation to Telegram**  
    - Type: Telegram  
    - Role: Sends a feedback message to the Telegram chat confirming content creation completion.  
    - Configuration: Uses Telegram credential; sends a fixed confirmation text mentioning Airtable review; chat ID retrieved dynamically from original Telegram input.  
    - Input: Triggered after Airtable save success.  
    - Output: Telegram message sent confirmation.  
    - Edge Cases: Invalid chat ID, Telegram API failures.

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                                      | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                       |
|-------------------------------|-----------------------------------------|----------------------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Receive Telegram Input         | Telegram Trigger                        | Entry point to receive Telegram messages            | N/A                          | Check Message Type             |                                                                                                                                  |
| Check Message Type             | Switch                                 | Routes messages by type (text or audio)             | Receive Telegram Input        | Extract Text Message, Download Telegram Audio |                                                                                                                                  |
| Extract Text Message           | Set                                    | Extracts text from Telegram message                  | Check Message Type (text)     | Generate Idea, Context & Image Prompt |                                                                                                                                  |
| Download Telegram Audio        | Telegram                               | Downloads voice message file from Telegram           | Check Message Type (audio)    | Transcribe Audio to Text       |                                                                                                                                  |
| Transcribe Audio to Text       | OpenAI Audio (Whisper)                  | Transcribes audio to text                            | Download Telegram Audio       | Prepare Transcribed Text       |                                                                                                                                  |
| Prepare Transcribed Text       | Set                                    | Prepares transcription text for AI processing       | Transcribe Audio to Text      | Generate Idea, Context & Image Prompt |                                                                                                                                  |
| Simple Memory                 | LangChain Memory Buffer Window          | Maintains session memory for contextual AI responses | Connected to Generate Idea... | Connected to Generate Idea...  |                                                                                                                                  |
| Web_search_tool               | HTTP Request Tool                       | Performs web research using Dumpling AI              | Called by Generate Idea...    | Returns search results         |                                                                                                                                  |
| GPT-4 Chat Model              | LangChain OpenAI Chat Model             | Provides GPT-4 chat completions for AI agent         | Integrated in Generate Idea...| Integrated in Generate Idea... |                                                                                                                                  |
| Generate Idea, Context & Image Prompt | LangChain Agent                       | AI agent that generates ideas, context, image prompt | Extract Text Message, Prepare Transcribed Text | Save to Airtable               |                                                                                                                                  |
| Clean Image Prompt            | LangChain Tool Code                      | Cleans image prompt string for image generation      | Called by Generate Idea...    | Returns cleaned prompt         |                                                                                                                                  |
| Image_generation_tool         | HTTP Request Tool                       | Generates an image from prompt via Dumpling AI       | Called by Generate Idea...    | Returns image URL              |                                                                                                                                  |
| Format Output as JSON         | LangChain Output Parser Structured       | Parses AI output into structured JSON                 | Called by Generate Idea...    | Generate Idea, Context & Image Prompt |                                                                                                                                  |
| Save to Airtable              | Airtable                               | Saves generated content to Airtable                   | Generate Idea, Context & Image Prompt | Send Confirmation to Telegram |                                                                                                                                  |
| Send Confirmation to Telegram  | Telegram                               | Sends confirmation message back to Telegram user     | Save to Airtable             | N/A                           |                                                                                                                                  |
| Sticky Note                  | Sticky Note                            | Describes the workflow purpose and steps             | N/A                          | N/A                           | ## 📌 LinkedIn Content Creator Workflow\n\nThis workflow helps you automatically generate polished LinkedIn content using AI. Just send a message (voice or text) to your Telegram bot, and the system will:\n\n1. Transcribe voice (if applicable) using OpenAI Whisper\n2. Use GPT-4 to generate:\n   - A short content idea\n   - Supporting context (2–4 sentences)\n   - An image prompt\n3. Run a Dumpling AI web search for relevant context\n4. Clean and format the image prompt\n5. Generate a visual using Dumpling AI\n6. Save the final result to Airtable\n7. Send a confirmation message back to Telegram\n\n### Output Format (Saved to Airtable):\n```json\n{\n  \"ideas\": \"string\",\n  \"context\": \"string\",\n  \"image\": \"URL\"\n}\n``` |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node: "Receive Telegram Input"**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates  
   - Credential: Connect your Telegram Bot API credentials  
   - Purpose: Entry point for receiving messages from Telegram.

2. **Add Switch Node: "Check Message Type"**  
   - Type: Switch  
   - Rules:  
     - Route to "text" if `{{$json.message.text}}` exists.  
     - Route to "audio" if `{{$json.message.voice.mime_type}} == "audio/ogg"`.  
   - Connect "Receive Telegram Input" output to this node.

3. **Text Path:**  
   a. Add Set Node: "Extract Text Message"  
      - Assign variable: `text = {{$json.message.text}}`  
      - Connect from "Check Message Type" text output.

4. **Audio Path:**  
   a. Add Telegram Node: "Download Telegram Audio"  
      - Resource: file  
      - File ID: `{{$json.message.voice.file_id}}`  
      - Credential: Telegram API credentials  
      - Connect from "Check Message Type" audio output.  
   b. Add OpenAI Node: "Transcribe Audio to Text"  
      - Resource: Audio  
      - Operation: Transcribe  
      - Credential: OpenAI API  
      - Connect from "Download Telegram Audio" output.  
   c. Add Set Node: "Prepare Transcribed Text"  
      - Assign variable: `text = {{$json.text}}`  
      - Connect from "Transcribe Audio to Text" output.

5. **Merge Text Inputs:**  
   - Connect both "Extract Text Message" and "Prepare Transcribed Text" outputs to the next node "Generate Idea, Context & Image Prompt".

6. **Add LangChain Memory Node: "Simple Memory"**  
   - Session Key: `{{$json.message.from.username}}`  
   - Session ID Type: Custom Key  
   - This node will be used by the AI agent for memory.

7. **Add HTTP Request Tool Node: "Web_search_tool"**  
   - Method: POST  
   - URL: https://app.dumplingai.com/api/v1/search-news  
   - Body (JSON): `{ "query": "{{ $fromAI('query') }}" }`  
   - Authentication: HTTP Header Auth with Dumpling AI API key  
   - Purpose: To perform web research.

8. **Add LangChain OpenAI Chat Model Node: "GPT-4 Chat Model"**  
   - Model: gpt-4.1-mini  
   - Credential: OpenAI API

9. **Add LangChain Tool Code Node: "Clean Image Prompt"**  
   - JavaScript code to remove newlines and collapse spaces from prompt string.

10. **Add HTTP Request Tool Node: "Image_generation_tool"**  
    - Method: POST  
    - URL: https://app.dumplingai.com/api/v1/generate-ai-image  
    - Body (JSON):  
      ```json
      {
        "model": "FLUX.1-pro",
        "input": {
          "prompt": "{{ $fromAI('prompt') }}",
          "seed": 42,
          "steps": 25,
          "width": 1024,
          "height": 1024,
          "guidance": 3,
          "aspect_ratio": "1:1",
          "output_format": "png",
          "output_quality": 90,
          "safety_tolerance": 2,
          "prompt_upsampling": false
        }
      }
      ```  
    - Authentication: HTTP Header Auth with Dumpling AI API key

11. **Add LangChain Output Parser Structured Node: "Format Output as JSON"**  
    - JSON Schema Example:  
      ```json
      {
        "ideas": "string",
        "context": "string",
        "image": "URL"
      }
      ```

12. **Add LangChain Agent Node: "Generate Idea, Context & Image Prompt"**  
    - Text Input: `user input: {{ $json.text }}`  
    - System Prompt: Use the detailed multi-step system prompt from the workflow (content creation agent instructions)  
    - Connect tools:  
      - Memory: "Simple Memory"  
      - AI Language Model: "GPT-4 Chat Model"  
      - Tools: "Web_search_tool", "Clean Image Prompt", "Image_generation_tool"  
      - Output Parser: "Format Output as JSON"  
    - Connect inputs from both text paths to this node.

13. **Add Airtable Node: "Save to Airtable"**  
    - Operation: Create  
    - Base & Table: Set to your Airtable base and table IDs  
    - Map fields:  
      - ideas = `{{$json.output.ideas}}`  
      - context = `{{$json.output.context}}`  
      - Image URL = `{{$json.output.image}}`  
    - Credential: Airtable Personal Access Token

14. **Add Telegram Node: "Send Confirmation to Telegram"**  
    - Text:  
      ```
      ✅ Your content has been created and saved to Airtable.
      
      - Ideas and context are ready  
      - Image has been generated and attached  
      
      You can review and approve everything directly in Airtable.
      ```  
    - Chat ID: `{{$json.message.chat.id}}`  
    - Credential: Telegram API  
    - Connect output of Airtable node to this node.

15. **Connections:**  
    - "Receive Telegram Input" → "Check Message Type"  
    - "Check Message Type" text → "Extract Text Message" → "Generate Idea, Context & Image Prompt"  
    - "Check Message Type" audio → "Download Telegram Audio" → "Transcribe Audio to Text" → "Prepare Transcribed Text" → "Generate Idea, Context & Image Prompt"  
    - "Generate Idea, Context & Image Prompt" → "Save to Airtable" → "Send Confirmation to Telegram"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates LinkedIn content creation from Telegram voice or text messages by integrating OpenAI Whisper for transcription, GPT-4 for idea/context generation, and Dumpling AI for web search and image generation. | Sticky note in workflow describing the full process.                                           |
| OpenAI Whisper transcription requires an OpenAI API key with audio resource access.                                                                                             | Credential setup for "Transcribe Audio to Text" node.                                          |
| Dumpling AI API requires HTTP Header authentication with an API key for both news search and image generation endpoints.                                                       | Credentials needed for "Web_search_tool" and "Image_generation_tool".                          |
| Airtable Personal Access Token must have write permissions to the specified base and table to create records.                                                                   | Credential required for "Save to Airtable" node.                                              |
| Telegram Bot API token required for triggering and sending messages back to users.                                                                                              | Credential used in "Receive Telegram Input", "Download Telegram Audio", and "Send Confirmation to Telegram". |
| The AI agent enforces grammar and format rules: no emojis, no hyphens, simple and professional language, single short idea, 2–4 sentence context, and image prompt cleaning before image generation. | System prompt embedded in "Generate Idea, Context & Image Prompt" node.                        |
| The workflow saves the final content as a JSON object with keys: ideas (string), context (string), and image (URL).                                                             | Output format stored in Airtable and sent back to Telegram.                                   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All handled data are legal and public.