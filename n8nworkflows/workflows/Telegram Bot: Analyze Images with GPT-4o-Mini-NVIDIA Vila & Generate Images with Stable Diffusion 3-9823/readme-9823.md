Telegram Bot: Analyze Images with GPT-4o-Mini/NVIDIA Vila & Generate Images with Stable Diffusion 3

https://n8nworkflows.xyz/workflows/telegram-bot--analyze-images-with-gpt-4o-mini-nvidia-vila---generate-images-with-stable-diffusion-3-9823


# Telegram Bot: Analyze Images with GPT-4o-Mini/NVIDIA Vila & Generate Images with Stable Diffusion 3

### 1. Workflow Overview

This workflow implements a **Smart Telegram Bot** that integrates advanced AI vision and image generation models to interact with users via Telegram messages. Its primary purpose is to:

- **Analyze images sent by users** using a combination of NVIDIA Vila (a vision-language model) and GPT-4o-Mini (an OpenAI vision-capable model), then email the textual analysis results.
- **Generate images from user text prompts** using NVIDIA's Stable Diffusion 3 model, then email the generated images.

The workflow is designed for content moderators, researchers, developers, or any user needing fast, AI-powered image understanding and generation directly from Telegram.

The workflow logic is divided into these blocks:

- **1.1 Input Reception:** Telegram Trigger node listens for new messages.
- **1.2 Message Parsing:** Extract message details, detect content type (image, text, etc.).
- **1.3 Content Routing:** Switch node routes processing based on content type.
- **1.4 Image Analysis Path:** Download image → analyze with NVIDIA Vila and GPT-4o-Mini → merge results → send email.
- **1.5 Text-to-Image Generation Path:** Use text prompt → generate image with NVIDIA Stable Diffusion 3 → convert to binary → send email.
- **1.6 Notifications:** Telegram replies to user confirming processing status.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens to Telegram messages of all types (text, images, documents, audio).
- **Nodes Involved:**  
  - 📱 Telegram Trigger
- **Node Details:**

  - **📱 Telegram Trigger**  
    - **Type:** Telegram Trigger  
    - **Role:** Entry point webhook listening for all new Telegram messages (update type: "message").  
    - **Configuration:** Uses a Telegram credential configured with a bot token. No additional filters; accepts all messages.  
    - **Input/Output:** Receives incoming Telegram updates, outputs full update JSON.  
    - **Edge Cases:** Telegram API connection errors, webhook failures, unsupported message types.  
    - **Version:** 1.2  

#### 1.2 Message Parsing

- **Overview:** Extracts relevant message properties, detects content type (image, text, audio, voice, document), and normalizes data for routing.
- **Nodes Involved:**  
  - 📋 Extract Message Data
- **Node Details:**

  - **📋 Extract Message Data**  
    - **Type:** Code (JavaScript)  
    - **Role:** Parses Telegram message JSON to extract text, usernames, chat IDs, message IDs, timestamps, and identifies media type presence. Sets a `contentType` field based on media presence (image, audio, voice, document, text).  
    - **Key Expressions:** Uses optional chaining and logical checks to safely extract fields.  
    - **Input:** Telegram update JSON from Trigger.  
    - **Output:** Structured JSON with extracted fields and contentType.  
    - **Potential Failures:** Unexpected message formats causing undefined values or missing fields.  
    - **Version:** 2  

#### 1.3 Content Routing

- **Overview:** Decides processing path based on content type: "image" messages go to image analysis, "text" messages to image generation.
- **Nodes Involved:**  
  - 🔀 Route by Content Type - Image or Text
- **Node Details:**

  - **🔀 Route by Content Type - Image or Text**  
    - **Type:** Switch  
    - **Role:** Conditional routing node checking `contentType` field for "image" or "text".  
    - **Configuration:** Two explicit outputs matching string equality on `contentType`.  
    - **Input:** Parsed message data from Extract Message Data.  
    - **Outputs:**  
      - Output 1: messages with contentType = "image"  
      - Output 2: messages with contentType = "text"  
    - **Fallback:** Unmatched content types go to fallback output (index 2, not connected here).  
    - **Edge Cases:** Unexpected content types or missing `contentType`.  
    - **Version:** 3.2  

#### 1.4 Image Analysis Path

- **Overview:** For image messages: download the image file from Telegram, send to NVIDIA Vila for description, analyze further with GPT-4o-Mini, merge results, and email the textual analysis.
- **Nodes Involved:**  
  - 📸 Get Image file from Telegram  
  - HTTP Nvidia Vila  
  - Analyze image using GPT-4O-Mini  
  - 🔀 Merge AI Results  
  - Send a message of image-to-text results  
- **Node Details:**

  - **📸 Get Image file from Telegram**  
    - **Type:** Telegram node (file resource)  
    - **Role:** Downloads the highest-resolution photo file from Telegram using the file_id extracted previously.  
    - **Config:** Uses Telegram API credentials; fileId set dynamically from message data.  
    - **Input:** Message data for contentType=image.  
    - **Output:** Binary file data of the image.  
    - **Edge Cases:** File not found, Telegram API errors, invalid file_id.  
    - **Version:** 1.2  

  - **HTTP Nvidia Vila**  
    - **Type:** HTTP Request  
    - **Role:** Sends the image file ID to NVIDIA Vila API to get a textual description of the image.  
    - **Config:** POST JSON body with message prompting description; uses Bearer token authentication for Nvidia API.  
    - **Input:** File ID from Telegram file node.  
    - **Output:** JSON response including textual description.  
    - **Edge Cases:** API authentication errors, rate limiting, malformed requests.  
    - **Version:** 4.2  

  - **Analyze image using GPT-4O-Mini**  
    - **Type:** OpenAI LangChain Node  
    - **Role:** Further analyzes the image, supplied as base64, via GPT-4o-Mini vision model.  
    - **Config:** Model set to "gpt-4o-mini"; input type is base64 image. Requires OpenAI API credentials.  
    - **Input:** Binary image from Telegram file node.  
    - **Output:** Text analysis JSON.  
    - **Edge Cases:** Model request failures, API quota exceeded, invalid base64 input.  
    - **Version:** 1.8  

  - **🔀 Merge AI Results**  
    - **Type:** Merge node  
    - **Role:** Combines output from NVIDIA Vila and GPT-4o-Mini analysis into a single JSON object.  
    - **Config:** Mode = combine, join mode = keep everything, matches on text content fields.  
    - **Input:** Two inputs from HTTP Nvidia Vila and GPT-4o-Mini nodes.  
    - **Output:** Combined AI textual results.  
    - **Version:** 3  

  - **Send a message of image-to-text results**  
    - **Type:** Gmail node  
    - **Role:** Sends an email containing the combined textual analysis results to a configured recipient.  
    - **Config:** Uses Gmail OAuth2 credentials; email subject "Results"; message body dynamically includes AI output.  
    - **Input:** Merged AI results.  
    - **Output:** Confirmation of email sent.  
    - **Edge Cases:** Email send failure, OAuth2 reauthorization required.  
    - **Version:** 2.1  

#### 1.5 Text-to-Image Generation Path

- **Overview:** For text messages: sends the text prompt to NVIDIA Stable Diffusion 3 API for image generation, converts returned base64 image to binary, then emails the generated image.
- **Nodes Involved:**  
  - Get Text from Telegram  
  - NVIDIA API NVIDIA Stable Diffusion 3  
  - Convert to Binary File  
  - Send a message of text-to-image results  
- **Node Details:**

  - **Get Text from Telegram**  
    - **Type:** Telegram node (send message)  
    - **Role:** Sends a Telegram reply notifying the user that their text message was received but advanced processing requires an image.  
    - **Config:** Message includes the original text and instructions; uses Telegram API credentials; chatId dynamic from message data.  
    - **Input:** Text content messages.  
    - **Output:** Confirmation of sent Telegram message.  
    - **Version:** 1.2  

  - **NVIDIA API NVIDIA Stable Diffusion 3**  
    - **Type:** HTTP Request  
    - **Role:** Calls NVIDIA Stable Diffusion 3 API with the user text prompt to generate an image.  
    - **Config:** POST request with parameters like prompt (from user text), cfg_scale=5, aspect ratio 16:9, steps=50; Bearer token authentication.  
    - **Input:** Text prompt from Telegram message.  
    - **Output:** JSON containing base64-encoded image data.  
    - **Edge Cases:** API timeouts, invalid prompt, auth failures.  
    - **Version:** 4.2  

  - **Convert to Binary File**  
    - **Type:** Code (JavaScript)  
    - **Role:** Parses API response to extract base64 image, cleans data URI prefixes, converts to binary buffer for attachment.  
    - **Config:** Custom JS code handling different possible response formats; creates binary file object with correct mime type and filename.  
    - **Input:** NVIDIA Stable Diffusion API JSON response.  
    - **Output:** Binary file data suitable for email attachment.  
    - **Edge Cases:** Missing or malformed base64 fields, decoding errors.  
    - **Version:** 2  

  - **Send a message of text-to-image results**  
    - **Type:** Gmail node  
    - **Role:** Sends an email to recipient with the generated image attached.  
    - **Config:** Gmail OAuth2 credentials; subject "Results"; message references attached image; attachment passed as binary.  
    - **Input:** Binary image from conversion node.  
    - **Output:** Email send confirmation.  
    - **Edge Cases:** Attachment size limits, email send failure.  
    - **Version:** 2.1  

#### 1.6 Notifications

- **Overview:** Sends Telegram messages back to user, informing them of message receipt and processing instructions.
- **Nodes Involved:**  
  - Get Text from Telegram (already described in 1.5)
- **Note:** The workflow sends an informative Telegram message only for text inputs, encouraging users to send images for advanced processing.

---

### 3. Summary Table

| Node Name                             | Node Type                     | Functional Role                                            | Input Node(s)                 | Output Node(s)                              | Sticky Note                                                                                                                       |
|-------------------------------------|-------------------------------|------------------------------------------------------------|------------------------------|---------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| 📱 Telegram Trigger                 | Telegram Trigger               | Entry point, listens for Telegram messages                  | None                         | 📋 Extract Message Data                      |                                                                                                                                |
| 📋 Extract Message Data             | Code                          | Extracts message fields, determines content type            | 📱 Telegram Trigger           | 🔀 Route by Content Type - Image or Text    |                                                                                                                                |
| 🔀 Route by Content Type - Image or Text | Switch                        | Routes workflow based on message contentType (image/text)  | 📋 Extract Message Data       | 📸 Get Image file from Telegram, Get Text from Telegram |                                                                                                                                |
| 📸 Get Image file from Telegram     | Telegram                      | Downloads image file from Telegram                           | 🔀 Route by Content Type      | Analyze image using GPT-4O-Mini, HTTP Nvidia Vila |                                                                                                                                |
| HTTP Nvidia Vila                   | HTTP Request                  | Sends image ID to NVIDIA Vila API for image description     | 📸 Get Image file from Telegram | 🔀 Merge AI Results                         |                                                                                                                                |
| Analyze image using GPT-4O-Mini    | OpenAI LangChain              | Analyzes image with GPT-4o-Mini vision model                | 📸 Get Image file from Telegram | 🔀 Merge AI Results                         |                                                                                                                                |
| 🔀 Merge AI Results                 | Merge                         | Combines text results from NVIDIA Vila and GPT-4o-Mini      | HTTP Nvidia Vila, Analyze image using GPT-4O-Mini | Send a message of image-to-text results   |                                                                                                                                |
| Send a message of image-to-text results | Gmail                         | Emails combined AI textual description to user              | 🔀 Merge AI Results           | None                                        |                                                                                                                                |
| Get Text from Telegram              | Telegram                      | Replies to user text messages with info about image requests| 🔀 Route by Content Type      | NVIDIA API NVIDIA Stable Diffusion 3        |                                                                                                                                |
| NVIDIA API NVIDIA Stable Diffusion 3 | HTTP Request                  | Generates image from text prompt using NVIDIA Stable Diffusion 3 | Get Text from Telegram        | Convert to Binary File                       |                                                                                                                                |
| Convert to Binary File              | Code                          | Converts base64 image from API to binary for email attachment| NVIDIA API NVIDIA Stable Diffusion 3 | Send a message of text-to-image results     |                                                                                                                                |
| Send a message of text-to-image results | Gmail                         | Emails generated image to user                               | Convert to Binary File         | None                                        |                                                                                                                                |
| Sticky Note1                       | Sticky Note                   | Documentation note with detailed workflow introduction      | None                         | None                                        | ## Introduction Transform your Telegram bot into an AI vision system using GPT-4o-Mini and NVIDIA Stable Diffusion 3. Perfect for content moderators, researchers, and developers. (Full note content above) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Set webhook to listen for `"message"` updates.  
   - Configure Telegram API credentials (Telegram bot token from @BotFather).  
   - Position: start of the workflow.

2. **Add Code node "📋 Extract Message Data"**  
   - Type: Code (JavaScript)  
   - Paste code to extract message text, chatId, user info, media presence, and determine contentType (`image`, `text`, etc.).  
   - Connect output of Telegram Trigger to this node.

3. **Add Switch node "🔀 Route by Content Type - Image or Text"**  
   - Type: Switch  
   - Configure two outputs based on expression: `{{$json.contentType}}` equals `"image"` or `"text"`.  
   - Connect from Extract Message Data node.

4. **Image Analysis Path:**

   4.1. **Add Telegram node "📸 Get Image file from Telegram"**  
   - Type: Telegram  
   - Resource: file  
   - FileId set dynamically: `{{$json.photoFileId}}`  
   - Use same Telegram API credentials.  
   - Connect first output (image) from Switch node to this node.

   4.2. **Add HTTP Request node "HTTP Nvidia Vila"**  
   - URL: `https://ai.api.nvidia.com/v1/vlm/nvidia/vila`  
   - Method: POST  
   - Body: JSON with messages including the Telegram photo file id; set model to `nvidia/vila`.  
   - Authentication: Bearer token with NVIDIA API key credential.  
   - Connect output of "📸 Get Image file from Telegram" to this node.

   4.3. **Add OpenAI LangChain node "Analyze image using GPT-4O-Mini"**  
   - Model: `gpt-4o-mini`  
   - Resource: image  
   - Operation: analyze  
   - Input type: base64 (provide image binary from Telegram file node)  
   - Configure OpenAI API credentials.  
   - Connect output of "📸 Get Image file from Telegram" to this node.

   4.4. **Add Merge node "🔀 Merge AI Results"**  
   - Mode: combine  
   - Join mode: keep everything  
   - Connect outputs of both "HTTP Nvidia Vila" and "Analyze image using GPT-4O-Mini" nodes to this merge node.

   4.5. **Add Gmail node "Send a message of image-to-text results"**  
   - Send To: recipient email  
   - Subject: "Results"  
   - Message body: includes merged AI text results (use expressions referencing merge node output).  
   - Use Gmail OAuth2 credentials.  
   - Connect output of Merge node to this node.

5. **Text-to-Image Generation Path:**

   5.1. **Add Telegram node "Get Text from Telegram"**  
   - Sends confirmation message to user that text message received and image is preferred.  
   - Use Telegram API credentials.  
   - Connect second output (text) of Switch node here.

   5.2. **Add HTTP Request node "NVIDIA API NVIDIA Stable Diffusion 3"**  
   - URL: `https://ai.api.nvidia.com/v1/genai/stabilityai/stable-diffusion-3-medium`  
   - Method: POST  
   - Body parameters: `prompt` from user text, plus cfg_scale, aspect_ratio, steps, seed as specified.  
   - Authentication: Bearer token with NVIDIA API key credential.  
   - Connect output of "Get Text from Telegram" to this node.

   5.3. **Add Code node "Convert to Binary File"**  
   - JavaScript code to parse base64 image from API response, clean data URI, convert to binary buffer for email attachment.  
   - Connect output of Stable Diffusion HTTP Request node to this node.

   5.4. **Add Gmail node "Send a message of text-to-image results"**  
   - Send To: recipient email  
   - Subject: "Results"  
   - Message body: text indicating image attached.  
   - Attach binary data from previous node.  
   - Use Gmail OAuth2 credentials.  
   - Connect output of "Convert to Binary File" node to this node.

6. **Optional: Add Sticky Note node**  
   - For documentation and explanatory purposes, add a sticky note summarizing purpose, prerequisites, setup, and benefits.

7. **Set workflow execution settings**  
   - Ensure webhook URLs are correctly configured and active for Telegram Trigger.  
   - Test each API credential (Telegram, OpenAI, NVIDIA, Gmail).  
   - Activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Transform your Telegram bot into an AI vision system using GPT-4o-Mini and NVIDIA Stable Diffusion 3. Ideal for content moderators, researchers, and developers. Workflow handles image analysis and text-to-image generation automatically.                                                                                                                   | Sticky Note content inside workflow.                                                                             |
| Prerequisites include Telegram Bot token from @BotFather, OpenAI API key for GPT-4 Vision, NVIDIA API key, and Gmail OAuth2 credentials for emailing results.                                                                                                                                                                                              | Setup instructions in Sticky Note.                                                                               |
| Customize with more AI models (Anthropic, Gemini), route audio/documents to transcription/OCR, replace Gmail with Slack/Discord, or connect to databases for storage.                                                                                                                                                                                         | Suggestions in Sticky Note.                                                                                       |
| NVIDIA Stable Diffusion 3 API documentation: https://docs.nvidia.com/ai-api/genai/stable-diffusion/                                                                                                                                                                                                                                                          | Official NVIDIA API docs.                                                                                        |
| OpenAI GPT-4o-Mini model requires OpenAI API access with vision capabilities enabled.                                                                                                                                                                                                                                                                       | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4o-mini                                                  |
| Gmail OAuth2 setup requires creating Google Cloud credentials with Gmail API enabled and OAuth2 client configured for n8n usage.                                                                                                                                                                                                                            | Google Gmail API docs: https://developers.google.com/gmail/api/quickstart/js                                         |

---

This structured reference document provides a comprehensive understanding of the Telegram Bot workflow, enabling users or AI agents to reproduce, maintain, or extend it confidently.