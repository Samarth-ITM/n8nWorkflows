Extract Text from Images with Telegram Bot & OCR

https://n8nworkflows.xyz/workflows/extract-text-from-images-with-telegram-bot---ocr-5413


# Extract Text from Images with Telegram Bot & OCR

### 1. Workflow Overview

This workflow automates the extraction of text from images sent by users in a Telegram chat, using Optical Character Recognition (OCR) and AI text enhancement. It is designed to receive images via a Telegram bot, perform OCR on the images to extract raw text, and then improve the readability and formatting of the extracted text through an AI agent before sending the polished result back to the user on Telegram.

Logical blocks included:

- **1.1 Input Reception via Telegram Bot:** Captures incoming image messages from Telegram users.
- **1.2 Image Retrieval and Conversion:** Downloads the image file from Telegram and converts it to a base64 string suitable for OCR processing.
- **1.3 OCR Processing:** Sends the base64-encoded image to an OCR service to extract raw text.
- **1.4 AI Text Enhancement:** Uses an AI agent (LangChain with Google Gemini chat model) to clean up and clarify the OCR output.
- **1.5 Output Delivery:** Sends the enhanced text back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception via Telegram Bot

- **Overview:**  
  Listens for incoming messages to the Telegram bot, specifically looking for image messages, and extracts relevant data for further processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Clean Input Data

- **Node Details:**

  - **Telegram Trigger**  
    - Type: `telegramTrigger`  
    - Role: Entry point that listens for new Telegram messages.  
    - Configuration: Triggers on "message" updates; uses Telegram API credentials named "Ruri Image Reader".  
    - Key expressions: None at this node; raw incoming JSON stored.  
    - Input: External Telegram webhook events.  
    - Output: Raw Telegram message JSON.  
    - Edge cases: Bot API downtime, webhook misconfiguration, unsupported message types.  
    - Notes: Webhook ID assigned for Telegram integration.

  - **Clean Input Data**  
    - Type: `set` node  
    - Role: Extracts and formats essential data from Telegram message JSON for downstream usage.  
    - Configuration:  
      - Extracts `chatID`: numeric Telegram chat identifier from `message.chat.id`.  
      - Extracts `Image`: file_id of the largest photo in the message (`message.photo` array; last element).  
    - Key expressions:  
      - `={{ $json.message.chat.id }}` for chat ID  
      - `={{ $json["message"]["photo"][$json["message"]["photo"].length - 1]["file_id"] }}` for image file ID  
    - Input: Telegram Trigger output JSON.  
    - Output: JSON with `chatID` and `Image` properties.  
    - Edge cases: Messages without images, empty photo arrays, malformed JSON.  
    - Failure modes: Expression failure if expected fields missing.

---

#### 1.2 Image Retrieval and Conversion

- **Overview:**  
  Downloads the Telegram image file using its file ID and converts the binary file into a base64-encoded string for OCR consumption.

- **Nodes Involved:**  
  - get file  
  - Convert to base64

- **Node Details:**

  - **get file**  
    - Type: `telegram` node  
    - Role: Downloads the image file from Telegram servers.  
    - Configuration:  
      - Uses `fileId` from `Clean Input Data.Image` (with newline characters removed).  
      - Resource set to "file" to fetch the actual file content.  
      - Credentials: Telegram API "Ruri Image Reader".  
    - Key expressions: `={{ $json.Image.replace(/\n/g, '') }}` to sanitize file ID.  
    - Input: Clean Input Data node output.  
    - Output: Binary data of the image file.  
    - Edge cases: Invalid file IDs, network failures, file not found.  
    - Failure modes: Telegram API errors, authentication errors.

  - **Convert to base64**  
    - Type: `extractFromFile`  
    - Role: Converts binary image file into base64 string and stores it in JSON property.  
    - Configuration: Operation set to "binaryToProperty" (binary to JSON property).  
    - Input: Binary output from "get file".  
    - Output: JSON with `data` property containing base64 string of the image.  
    - Edge cases: Corrupted binary data, unexpected binary formats.

---

#### 1.3 OCR Processing

- **Overview:**  
  Sends the base64-encoded image to an OCR service via HTTP POST to extract the raw text content from the image.

- **Nodes Involved:**  
  - OCR

- **Node Details:**

  - **OCR**  
    - Type: `httpRequest`  
    - Role: Calls an external OCR API endpoint to extract text from the base64 image.  
    - Configuration:  
      - Method: POST  
      - URL: Placeholder `"#"` (to be replaced with actual OCR API endpoint).  
      - Body parameters: JSON with `"image"` key set to `={{ $json.data }}` (base64 image).  
      - Sends body as JSON.  
    - Input: JSON from "Convert to base64" node.  
    - Output: JSON containing OCR text in `output` property.  
    - Edge cases: OCR service downtime, invalid API URL, malformed requests, rate limits.  
    - Failure modes: HTTP errors, timeouts, invalid responses.

---

#### 1.4 AI Text Enhancement

- **Overview:**  
  Uses an AI agent powered by LangChain and Google Gemini chat model to clean, clarify, and restructure OCR output for better readability before sending to the user.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model (as AI language model backend)

- **Node Details:**

  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Processes raw OCR text and improves clarity, fixes OCR errors, and reformats content for end users.  
    - Configuration:  
      - Input text: `={{ $json.text }}` (set dynamically).  
      - System message prompt instructs the AI to focus on cleaning OCR output, summarizing tables into bullet points, fixing broken words and characters, and improving formatting.  
      - Uses "define" prompt type for custom instructions.  
    - Input: OCR node output mapped to `text` property (connection implicit).  
    - Output: Enhanced text in `output` property.  
    - Edge cases: AI API errors, unexpected input formats, prompt misinterpretation.  
    - Failure modes: Rate limits, authentication errors, malformed prompt.

  - **Google Gemini Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - Role: Language model backend used by AI Agent for text generation.  
    - Configuration:  
      - Model name: `models/gemini-2.0-flash`  
      - Credentials: Google Palm API account ("Google Gemini(PaLM) Api account").  
    - Connected as AI language model to AI Agent node.  
    - Edge cases: API key invalidation, service downtime, rate limits.

---

#### 1.5 Output Delivery

- **Overview:**  
  Sends the enhanced text back to the originating Telegram chat.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - Type: `telegram` node  
    - Role: Sends a text message reply to the Telegram user with the cleaned OCR text.  
    - Configuration:  
      - Text: `={{ $json.output }}` (enhanced OCR text from AI Agent).  
      - Chat ID: `={{ $('Clean Input Data').item.json.chatID }}` (original sender).  
      - Append attribution disabled.  
      - Credentials: Telegram API "Ruri Image Reader".  
    - Input: Output from AI Agent node.  
    - Output: Telegram message sent confirmation.  
    - Edge cases: Invalid chat ID, Telegram API errors, message length limits.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                          | Input Node(s)       | Output Node(s)         | Sticky Note                                      |
|-----------------------|----------------------------------|----------------------------------------|---------------------|------------------------|-------------------------------------------------|
| Telegram Trigger       | telegramTrigger                  | Receives Telegram messages (images)    | External Telegram    | Clean Input Data       |                                                 |
| Clean Input Data       | set                             | Extracts chat ID and image file ID      | Telegram Trigger    | get file               |                                                 |
| get file              | telegram                        | Downloads image file from Telegram      | Clean Input Data     | Convert to base64       |                                                 |
| Convert to base64      | extractFromFile                 | Converts image binary to base64 string  | get file             | OCR                    |                                                 |
| OCR                   | httpRequest                    | Sends base64 image to OCR API           | Convert to base64    | AI Agent               |                                                 |
| AI Agent              | langchain.agent                | Enhances OCR text with AI                | OCR                  | Telegram                |                                                 |
| Google Gemini Chat Model | langchain.lmChatGoogleGemini | AI language model backend for AI Agent | None (used by AI Agent) | AI Agent (ai_languageModel) |                                                 |
| Telegram              | telegram                       | Sends enhanced text back to user         | AI Agent             | None                   |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `telegramTrigger`  
   - Set update types to listen for: `message`  
   - Configure credentials with your Telegram Bot API token ("Ruri Image Reader" or your own).  
   - Position it as the workflow entry point.

2. **Add Set Node ("Clean Input Data")**  
   - Type: `set` node  
   - Add two fields:  
     - `chatID` (number) with expression: `{{$json.message.chat.id}}`  
     - `Image` (string) with expression: `{{$json["message"]["photo"][$json["message"]["photo"].length - 1]["file_id"]}}`  
   - Connect Telegram Trigger → Clean Input Data.

3. **Add Telegram Node ("get file")**  
   - Type: `telegram`  
   - Resource: `file`  
   - Set `fileId` parameter with expression: `={{ $json.Image.replace(/\n/g, '') }}`  
   - Use same Telegram API credentials.  
   - Connect Clean Input Data → get file.

4. **Add Extract From File Node ("Convert to base64")**  
   - Type: `extractFromFile`  
   - Operation: `binaryToProperty` (convert binary to base64 string property)  
   - Connect get file → Convert to base64.

5. **Add HTTP Request Node ("OCR")**  
   - Type: `httpRequest`  
   - Method: POST  
   - URL: Set to your OCR API endpoint URL (replace placeholder "#").  
   - Body Parameters: JSON with key `image` and value expression `={{ $json.data }}` (base64 image).  
   - Send body as JSON.  
   - Connect Convert to base64 → OCR.

6. **Add LangChain AI Agent Node ("AI Agent")**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Text parameter: `={{ $json.text }}` expecting OCR output text mapped to `text` property.  
   - System Message: Paste the detailed prompt instructing the AI to clean and clarify OCR text, fix common OCR errors, convert tables to bullet points, etc.  
   - Prompt Type: `define`  
   - Connect OCR → AI Agent.

7. **Add Google Gemini Chat Model Node ("Google Gemini Chat Model")**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Model Name: `models/gemini-2.0-flash`  
   - Credentials: Configure with Google Palm API account.  
   - Connect as AI language model input to AI Agent node (ai_languageModel connection).

8. **Add Telegram Node ("Telegram")**  
   - Type: `telegram`  
   - Text: `={{ $json.output }}` (final AI-processed text)  
   - Chat ID: `={{ $('Clean Input Data').item.json.chatID }}`  
   - Append Attribution: set to false  
   - Credentials: same Telegram API credentials.  
   - Connect AI Agent → Telegram.

9. **Activate Webhook URLs**  
   - Ensure Telegram webhook URLs are properly configured in Telegram Bot settings to point to your n8n instance webhook URL for Telegram Trigger.

10. **Test End-to-End**  
    - Send an image to your Telegram bot.  
    - Confirm the bot replies with the cleaned and enhanced OCR text.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The OCR node’s HTTP URL is set as "#" and must be replaced with a valid OCR API endpoint accepting base64 images.   | OCR API integration requires proper endpoint setup.                                               |
| The AI Agent prompt focuses on fixing common OCR errors such as broken words, character misrecognition, and formatting. | Custom system prompt in AI Agent node defines workflow's text enhancement behavior.                |
| Google Gemini (PaLM) API credentials are required for the AI language model node to function correctly.               | [Google PaLM API documentation](https://developers.generativeai.google/api/gemini)                 |
| Telegram Bot credentials must be generated through BotFather and properly configured for webhook integration.        | Telegram Bot API documentation: https://core.telegram.org/bots/api                                  |

---

**Disclaimer:** The text provided is extracted solely from an n8n automated workflow. All content complies with current content policies and contains no illegal or offensive elements. All data handled is legal and public.