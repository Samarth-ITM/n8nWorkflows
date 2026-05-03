Text-to-Image Generation with Google Gemini & Enhanced Prompts via Telegram Bot

https://n8nworkflows.xyz/workflows/text-to-image-generation-with-google-gemini---enhanced-prompts-via-telegram-bot-5777


# Text-to-Image Generation with Google Gemini & Enhanced Prompts via Telegram Bot

### 1. Workflow Overview

This workflow is a Telegram bot that converts user text descriptions into high-quality AI-generated images using Google Gemini models. It is designed for instant, interactive text-to-image generation through a Telegram chat interface. The workflow consists of several logical blocks:

- **1.1 Input Reception:** Listens for incoming Telegram messages containing user image requests.
- **1.2 Prompt Enhancement:** Uses Google Gemini 2.5 Pro AI to transform the user's basic text into a detailed, richly descriptive prompt optimized for image generation.
- **1.3 Image Generation:** Calls Google Gemini 2.0 Flash Preview image generation API to produce an image based on the enhanced prompt.
- **1.4 Image Conversion:** Converts the API’s base64 image data into a binary format compatible with Telegram messaging.
- **1.5 Delivery:** Sends the generated image back to the user on Telegram as a photo message.

This structure ensures a smooth flow from receiving user input, enhancing it with AI prompt engineering, generating an image, preparing the file, and delivering it back seamlessly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block captures incoming Telegram messages and triggers the workflow.
- **Nodes Involved:** Telegram Trigger
- **Node Details:**

  - **Telegram Trigger**
    - Type: Telegram Trigger node
    - Role: Entry point; listens for new Telegram messages.
    - Configuration:
      - Listens to "message" updates only.
      - Outputs the full message JSON, including chat ID and message text.
      - Credential: Telegram API OAuth2 credential.
    - Inputs: None (trigger node)
    - Outputs: User message JSON, including `message.text` and `message.chat.id`.
    - Potential Failures:
      - Telegram API connectivity/authentication issues.
      - Missing or invalid bot token.
      - Unexpected message formats.
    - Sticky Note: Describes its role as entry point capturing user image requests.

#### 2.2 Prompt Enhancement

- **Overview:** Enhances the user's basic text input into a detailed, artistically rich prompt suitable for image generation.
- **Nodes Involved:** Gemini 2.5 Pro, Generate prompt, Structured Prompt
- **Node Details:**

  - **Gemini 2.5 Pro**
    - Type: LangChain Google Gemini LLM Chat node
    - Role: Calls Google Gemini 2.5 Pro to generate enhanced prompt text.
    - Configuration:
      - Uses model `models/gemini-2.5-pro`.
      - Credential: Google Palm API key.
      - No custom options set.
    - Inputs: Receives the user message text from "Generate prompt" node.
    - Outputs: AI-generated expanded prompt.
    - Potential Failures:
      - API key invalid or expired.
      - Network timeouts.
      - Model quota limits.
    - No sub-workflow.

  - **Generate prompt**
    - Type: LangChain Chain LLM node
    - Role: Defines the prompt engineering logic for expanding user input.
    - Configuration:
      - Uses a detailed instruction template.
      - Input: Raw user text from Telegram.
      - Output: Expected structured enhanced prompt.
      - Has output parser enabled to structure response.
    - Inputs: User message from Telegram Trigger.
    - Outputs: Structured prompt JSON.
    - Expressions:
      - Uses `{{ $json.message.text }}` to access user input.
    - Potential Failures:
      - Expression parsing errors.
      - AI response malformed or missing fields.
    - Connected downstream to "Generate Image".

  - **Structured Prompt**
    - Type: LangChain Output Parser Structured node
    - Role: Parses AI output into JSON with expected schema `{ "enhanced_prompt": "..." }`.
    - Configuration:
      - Defines JSON schema example to ensure structured output.
    - Inputs: AI raw output from Gemini 2.5 Pro.
    - Outputs: Parsed enhanced prompt JSON.
    - Potential Failures:
      - Parser failures if AI output does not match schema.
    - Connected upstream to "Generate prompt".

#### 2.3 Image Generation

- **Overview:** Sends the enhanced prompt to Google Gemini’s image generation API to create an image.
- **Nodes Involved:** Generate Image
- **Node Details:**

  - **Generate Image**
    - Type: HTTP Request node
    - Role: POST request to Google Gemini 2.0 Flash Preview image generation endpoint.
    - Configuration:
      - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-preview-image-generation:generateContent`
      - Method: POST
      - Body: JSON including the enhanced prompt text as `"contents"` with nested parts.
      - Query parameter: `key` for API authentication (placeholder "Your API Key" to be replaced).
      - Request expects JSON response with text and base64 image.
    - Inputs: Enhanced prompt JSON (`.output.enhanced_prompt`).
    - Outputs: JSON containing image data base64 encoded in `candidates[0].content.parts[1].inlineData.data`.
    - Potential Failures:
      - API key invalid or missing.
      - API rate limits or quota exceeded.
      - Malformed request body.
      - Network or timeout errors.
    - Sticky Note: Explains it calls Gemini 2.0 Flash API to generate images.

#### 2.4 Image Conversion

- **Overview:** Converts base64 image data from API response into binary data for Telegram.
- **Nodes Involved:** Convert to Image
- **Node Details:**

  - **Convert to Image**
    - Type: Convert To File node
    - Role: Converts base64 inline image data into binary file format.
    - Configuration:
      - Operation: toBinary
      - Source Property: `candidates[0].content.parts[1].inlineData.data` (base64 string of the image)
    - Inputs: JSON response from "Generate Image".
    - Outputs: Binary data representing the image file.
    - Potential Failures:
      - Missing or corrupted base64 data.
      - Conversion failures due to unexpected data format.
    - Sticky Note: Notes this prepares image for Telegram delivery.

#### 2.5 Delivery

- **Overview:** Sends the generated image as a photo message back to the user on Telegram.
- **Nodes Involved:** Send a photo message
- **Node Details:**

  - **Send a photo message**
    - Type: Telegram node
    - Role: Sends binary image data as a photo message to the user.
    - Configuration:
      - Operation: sendPhoto
      - Chat ID: Extracted dynamically from original Telegram message `$('Telegram Trigger').item.json.message.chat.id`
      - Sends binary data (image) attached from "Convert to Image".
      - Credential: Same Telegram API credential.
    - Inputs: Binary image data from "Convert to Image".
    - Outputs: Confirmation of message sent.
    - Potential Failures:
      - Telegram API errors (invalid chat ID, rate limits).
      - Binary data format issues.
      - Authentication failures.
    - Sticky Note: Describes final delivery step completing the interaction.

---

### 3. Summary Table

| Node Name          | Node Type                           | Functional Role                     | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                             |
|--------------------|-----------------------------------|-----------------------------------|---------------------|----------------------|-------------------------------------------------------------------------------------------------------|
| Telegram Trigger    | Telegram Trigger                  | Listens for user messages          | None                | Generate prompt      | Entry point for the workflow; captures incoming text messages; triggers entire process                 |
| Generate prompt     | LangChain Chain LLM               | Enhances user input with AI prompt | Telegram Trigger    | Generate Image       | Enhances user input with AI; adds artistic style, lighting, composition; powered by Gemini 2.5 Pro     |
| Structured Prompt   | LangChain Output Parser Structured| Parses AI output into structured JSON | Gemini 2.5 Pro    | Generate prompt      | Parses AI output for structured enhanced prompt                                                       |
| Gemini 2.5 Pro      | LangChain Google Gemini LLM Chat | AI model generating enhanced prompt| Generate prompt     | Structured Prompt    | AI engine providing detailed prompt expansion                                                        |
| Generate Image      | HTTP Request                     | Calls Google Gemini image generation API | Generate prompt   | Convert to Image     | Calls Gemini 2.0 Flash API for image creation; returns base64 format                                  |
| Convert to Image    | Convert To File                  | Converts base64 image to binary    | Generate Image      | Send a photo message | Converts base64 to binary for Telegram compatibility                                                 |
| Send a photo message| Telegram node                   | Sends image back to user           | Convert to Image    | None                 | Sends generated image to Telegram user                                                               |
| Sticky Note         | Sticky Note                     | Documentation and comments         | None                | None                 | Describes Telegram Trigger node                                                                        |
| Sticky Note1        | Sticky Note                     | Documentation and comments         | None                | None                 | Describes Generate prompt node                                                                         |
| Sticky Note2        | Sticky Note                     | Documentation and comments         | None                | None                 | Describes Generate Image node                                                                          |
| Sticky Note3        | Sticky Note                     | Documentation and comments         | None                | None                 | Describes Convert to Image node                                                                        |
| Sticky Note4        | Sticky Note                     | Documentation and comments         | None                | None                 | Describes Send a photo message node                                                                   |
| Sticky Note5        | Sticky Note                     | Documentation and comments         | None                | None                 | Provides overall workflow summary and flow description                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**
   - Type: Telegram Trigger
   - Configure to listen for `message` updates only.
   - Select or create Telegram API credential.
   - Position: far left to represent entry.
   - No inputs; this node triggers workflow on user messages.

2. **Create LangChain Google Gemini LLM Chat node ("Gemini 2.5 Pro")**
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`
   - Model: `models/gemini-2.5-pro`
   - Configure credential: Google Palm API key (Google Gemini).
   - No additional options.
   - Position near center-left.

3. **Create LangChain Chain LLM node ("Generate prompt")**
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`
   - Paste prompt template that expands user text into detailed prompt, including instructions for artistic style, lighting, composition, etc.
   - Enable output parser.
   - Connect input from Telegram Trigger.
   - Set output to feed Gemini 2.5 Pro node.

4. **Create LangChain Output Parser Structured node ("Structured Prompt")**
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`
   - Provide JSON schema example: `{ "enhanced_prompt": "..." }`
   - Connect input from Gemini 2.5 Pro.
   - Output feeds back into Generate prompt node (as ai_outputParser).

5. **Create HTTP Request node ("Generate Image")**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-preview-image-generation:generateContent`
   - Authentication: API key as query parameter (`key=Your API Key`)
   - Body: JSON with `contents` array containing the enhanced prompt text from structured prompt output.
   - Ensure request body is JSON and sent as such.
   - Connect input from Generate prompt's output (enhanced prompt).
   - Position center.

6. **Create Convert To File node ("Convert to Image")**
   - Type: Convert To File
   - Operation: toBinary
   - Source property: `candidates[0].content.parts[1].inlineData.data` (base64 image data)
   - Connect input from Generate Image.
   - Position center-right.

7. **Create Telegram node ("Send a photo message")**
   - Type: Telegram node
   - Operation: sendPhoto
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
   - Enable sending binary data.
   - Connect input from Convert to Image.
   - Configure with Telegram API credential.
   - Position rightmost.

8. **Connect nodes in order:**
   - Telegram Trigger → Generate prompt → Gemini 2.5 Pro → Structured Prompt → Generate prompt (loop for output parsing)
   - Generate prompt → Generate Image → Convert to Image → Send a photo message

9. **Credentials Setup:**
   - Telegram API OAuth2 credential with bot token.
   - Google Palm API key credential for Gemini nodes.
   - HTTP Request node requires Google API key as query parameter.

10. **Defaults & Constraints:**
    - Ensure Telegram bot has permission to send messages and receive updates.
    - Google Gemini API keys must be valid and enabled for respective models (2.5 Pro for text, 2.0 for image).
    - Handle API rate limits gracefully or add error handling if desired.
    - Confirm base64 data path in HTTP response matches actual API response schema.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow transforms simple Telegram text inputs into stunning AI-generated images using Google Gemini| Overall workflow description                                                                              |
| Prompt engineering instructions are detailed to maximize image quality and artistic effect           | "Generate prompt" node prompt template                                                                    |
| Uses Google Gemini 2.5 Pro for advanced text prompt enhancement and Gemini 2.0 Flash Preview for images| Google Gemini AI models integration                                                                        |
| Telegram bot interface provides interactive user experience with instant feedback                    | Telegram Trigger and Telegram Send Photo nodes                                                            |
| Sticky notes embedded in workflow provide inline documentation of each logical block                 | Sticky Notes nodes                                                                                            |
| Replace `"Your API Key"` in the HTTP Request node with a valid Google API Key for image generation   | Required API key replacement for production                                                                |
| Workflow uses LangChain nodes requiring n8n version supporting LangChain integration (v1.7+ suggested) | Version-specific support for LangChain nodes                                                                |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.