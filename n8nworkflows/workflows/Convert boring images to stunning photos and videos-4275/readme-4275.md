Convert boring images to stunning photos and videos

https://n8nworkflows.xyz/workflows/convert-boring-images-to-stunning-photos-and-videos-4275


# Convert boring images to stunning photos and videos

---

### 1. Workflow Overview

This workflow is designed to transform user-submitted images into enhanced, AI-edited photos and optionally generate AI variations or videos based on those images. It is triggered by receiving images via a Telegram bot, uses OpenAI’s image editing API for creative edits based on user captions, and can optionally invoke Replicate’s AI models for further image variation generation. The edited outputs and variations are sent back to the user through Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception via Telegram Bot:** Listens for incoming Telegram messages containing images and captions, downloads images, and prepares data for processing.
- **1.2 AI Image Editing with OpenAI:** Sends the original image and user caption to OpenAI’s image editing API, receiving an edited image in response.
- **1.3 Image Conversion and Return:** Converts OpenAI’s base64 image data to binary format and sends the edited image back to the user on Telegram.
- **1.4 Optional AI Variation Generation with Replicate:** Retrieves the original image file path, sends a request to Replicate’s model to generate AI variations or videos, waits for processing, retrieves the result, and sends it back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception via Telegram Bot

- **Overview:**  
  This block listens for incoming Telegram messages containing images with captions. It downloads the image file to be used as input for AI processing.

- **Nodes Involved:**  
  - Telegram Bot Trigger

- **Node Details:**

  - **Telegram Bot Trigger**  
    - *Type & Role:* Trigger node that listens to Telegram bot messages, specifically those containing images.  
    - *Configuration:* Listens for "message" updates, downloads the image file automatically.  
    - *Key Expressions:* None specific; the node outputs the full Telegram message object including photo binary and caption text.  
    - *Connections:* Output connects to "Edit Image (OpenAI)".  
    - *Version Notes:* Uses webhook-based triggering (requires Telegram webhook setup).  
    - *Edge Cases / Failures:*  
      - Missing caption or image in message (workflow expects both).  
      - Telegram download permission or network errors.  
      - Invalid bot token or revoked permissions.  
    - *Sub-Workflow:* None

- **Comments:**  
  The node requires the bot to have file download permissions and users to send images with captions describing desired edits.

---

#### 2.2 AI Image Editing with OpenAI

- **Overview:**  
  Sends the original image and the user’s caption prompt to OpenAI’s image editing API for AI-powered transformations.

- **Nodes Involved:**  
  - Edit Image (OpenAI)

- **Node Details:**

  - **Edit Image (OpenAI)**  
    - *Type & Role:* HTTP Request node to OpenAI API endpoint for image editing.  
    - *Configuration:*  
      - URL: `https://api.openai.com/v1/images/edits`  
      - Method: POST  
      - Content Type: multipart/form-data  
      - Body Parameters:  
        - `image`: binary data from Telegram (downloaded image).  
        - `prompt`: user caption (`={{ $json.message.caption }}`).  
        - `model`: `gpt-image-1` (OpenAI image editing model).  
        - `n`: 1 (one output image).  
        - `size`: 1024x1024 pixels.  
        - `quality`: low (faster and cheaper).  
      - Authentication: Uses predefined OpenAI API credentials configured in n8n.  
    - *Connections:* Output connects to "Convert to Binary Image".  
    - *Version Notes:* Requires OpenAI API key credential in n8n.  
    - *Edge Cases / Failures:*  
      - Authentication failure (invalid/expired API key).  
      - API rate limits or quota exceeded.  
      - Missing or malformed image data.  
      - Caption prompt empty or invalid.  
      - Network timeouts.  
    - *Sub-Workflow:* None

- **Comments:**  
  This node enables AI-based image editing driven by user prompts, controlling model, size, and output count.

---

#### 2.3 Image Conversion and Return

- **Overview:**  
  Converts the base64-encoded image returned by OpenAI into binary format suitable for sending back via Telegram, then sends the edited image.

- **Nodes Involved:**  
  - Convert to Binary Image  
  - Send Edited Image

- **Node Details:**

  - **Convert to Binary Image**  
    - *Type & Role:* Converts base64 image data to binary file.  
    - *Configuration:*  
      - Source property: `data[0].b64_json` (OpenAI response field containing base64 image).  
      - Output file name: `edited_image.png`.  
      - MIME type: `image/png`.  
    - *Connections:* Output connects to "Send Edited Image".  
    - *Version Notes:* None specific.  
    - *Edge Cases / Failures:*  
      - Missing or malformed base64 data.  
      - Conversion errors.  
    - *Sub-Workflow:* None

  - **Send Edited Image**  
    - *Type & Role:* Sends the edited image to the Telegram chat where the original request came from.  
    - *Configuration:*  
      - Chat ID: dynamically retrieved from `$json.message.chat.id`.  
      - Operation: `sendPhoto`.  
      - Sends binary data from previous node.  
      - Caption: "Here's your edited image! ✨".  
    - *Connections:* Output connects to "Get File Path".  
    - *Version Notes:* Requires Telegram bot token credential.  
    - *Edge Cases / Failures:*  
      - Invalid chat ID or bot permissions.  
      - Binary data missing or corrupt.  
      - Telegram API rate limits or failures.  
    - *Sub-Workflow:* None

- **Comments:**  
  Sending the edited image completes the main user feedback loop of the workflow.

---

#### 2.4 Optional AI Variation Generation with Replicate

- **Overview:**  
  This block uses Replicate’s AI model to create additional variations or videos based on the original image, then sends those results back to the user after asynchronous processing and polling.

- **Nodes Involved:**  
  - Get File Path  
  - Generate Variation (Replicate)  
  - Wait for Processing  
  - Retrieve Generated Image  
  - Send Variation Image

- **Node Details:**

  - **Get File Path**  
    - *Type & Role:* HTTP Request to Telegram Bot API to get the file path of the original image stored on Telegram servers.  
    - *Configuration:*  
      - URL: `https://api.telegram.org/bot{{$credentials.telegramBot.botToken}}/getFile`  
      - Method: POST  
      - Body: JSON with `file_id` from the original photo (`$json.result.photo[0].file_id`).  
    - *Connections:* Output connects to "Generate Variation (Replicate)".  
    - *Version Notes:* Telegram token required.  
    - *Edge Cases / Failures:*  
      - Invalid or expired file_id.  
      - Telegram API errors.  
    - *Sub-Workflow:* None

  - **Generate Variation (Replicate)**  
    - *Type & Role:* HTTP Request node to Replicate API to create AI variations/videos.  
    - *Configuration:*  
      - URL: `https://api.replicate.com/v1/models/pixverse/pixverse-v4/predictions`  
      - Method: POST  
      - JSON Body:  
        - `prompt`: user caption or default "a creative enhancement of this image".  
        - `quality`: "720p".  
        - `image`: direct Telegram file URL constructed dynamically using bot token and file path (`https://api.telegram.org/file/bot{{$credentials.telegramBot.botToken}}/{{ $json.result.file_path }}`).  
      - Headers: Authorization with Bearer token from Replicate API key credential.  
    - *Connections:* Output connects to "Wait for Processing".  
    - *Version Notes:* Requires Replicate API token credential.  
    - *Edge Cases / Failures:*  
      - Authentication failure with Replicate.  
      - Invalid image URL or inaccessible resource.  
      - API rate limits, payload errors.  
    - *Sub-Workflow:* None

  - **Wait for Processing**  
    - *Type & Role:* Waits 45 seconds to allow Replicate model to finish processing asynchronously.  
    - *Configuration:*  
      - Duration: 45 seconds.  
    - *Connections:* Output connects to "Retrieve Generated Image".  
    - *Version Notes:* None.  
    - *Edge Cases / Failures:*  
      - Processing time longer than wait, leading to premature request.  
    - *Sub-Workflow:* None

  - **Retrieve Generated Image**  
    - *Type & Role:* HTTP Request to GET the final AI-generated image/video from Replicate using the returned URL.  
    - *Configuration:*  
      - URL: dynamic from previous response (`={{ $json.urls.get }}`).  
      - Headers: Authorization with Replicate API token.  
      - Response expected in a format suitable for sending (Base64 encoding implied).  
    - *Connections:* Output connects to "Send Variation Image".  
    - *Version Notes:* Requires Replicate API token.  
    - *Edge Cases / Failures:*  
      - URL invalid or processing incomplete.  
      - Authorization failures.  
      - Network errors.  
    - *Sub-Workflow:* None

  - **Send Variation Image**  
    - *Type & Role:* Sends the AI-generated variation (image or video) back to the user on Telegram.  
    - *Configuration:*  
      - Chat ID: dynamically from original message.  
      - Operation: `sendDocument` (supports images/videos).  
      - File: from previous node output.  
      - Caption: "Here's an AI-generated variation of your image! 🚀".  
    - *Connections:* None (end node).  
    - *Version Notes:* Telegram bot token required.  
    - *Edge Cases / Failures:*  
      - File size too large for Telegram.  
      - Invalid chat ID or permissions.  
      - Corrupt or missing file data.  
    - *Sub-Workflow:* None

- **Comments:**  
  This block requires valid Telegram and Replicate credentials and handles asynchronous AI generation with a wait step to avoid premature requests.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                         | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                                    |
|-------------------------|-------------------------|---------------------------------------|---------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| Telegram Bot Trigger     | Telegram Trigger        | Listens for incoming Telegram images  |                           | Edit Image (OpenAI)      | Listens for incoming messages with images from a Telegram bot                                                |
| Workflow Start Documentation | Sticky Note           | Documentation of Telegram trigger     |                           |                         | ## 📱 Telegram Message Trigger<br>Describes Telegram input requirements and output                            |
| Workflow Overview       | Sticky Note              | Overview of entire image processing   |                           |                         | ## 🖼️ Image Processing Pipeline<br>Explains overall workflow and use cases                                    |
| OpenAI API Documentation | Sticky Note              | Describes OpenAI image editing API    |                           |                         | ## 🎨 OpenAI Image Editing API<br>Details API parameters and authentication                                   |
| Conversion Documentation | Sticky Note              | Explains base64 to binary conversion  |                           |                         | ## 📤 File Format Conversion<br>Details converting OpenAI response to binary                                  |
| Edit Image (OpenAI)      | HTTP Request             | Sends image to OpenAI for editing     | Telegram Bot Trigger       | Convert to Binary Image  | Sends image to OpenAI for AI-powered editing based on caption                                                |
| Convert to Binary Image  | Convert To File          | Converts base64 image to binary       | Edit Image (OpenAI)        | Send Edited Image        | Converts base64 encoded image to binary format for sending                                                  |
| Send Edited Image        | Telegram                 | Sends edited image back to user       | Convert to Binary Image    | Get File Path            | Sends the edited image back to the user via Telegram                                                        |
| Replicate Documentation  | Sticky Note              | Describes Replicate AI image generation |                           |                         | ## 🧠 Replicate AI Image Generation<br>Details Replicate model and API usage                                 |
| File Retrieval Documentation | Sticky Note           | Describes Telegram file path retrieval |                           |                         | ## 📡 File Retrieval System<br>Explains retrieving Telegram file path for Replicate                          |
| Get File Path            | HTTP Request             | Retrieves Telegram file path          | Send Edited Image          | Generate Variation (Replicate) | Retrieves the file path from Telegram's servers                                                           |
| Generate Variation (Replicate) | HTTP Request        | Sends image to Replicate for variation | Get File Path             | Wait for Processing      | Sends the original image to Replicate for AI-powered variation                                             |
| Wait Node Documentation  | Sticky Note              | Describes wait step for polling       |                           |                         | ## ⏱️ Polling Mechanism<br>Waits 45 seconds to allow Replicate processing                                    |
| Wait for Processing      | Wait                     | Waits before retrieving Replicate result | Generate Variation (Replicate) | Retrieve Generated Image | Waits for Replicate to complete the image generation                                                        |
| Note: Get Replicate Result | Sticky Note            | Describes retrieving Replicate output |                           |                         | ### Replicate API: Get Result<br>Explains retrieving final video with authorization                         |
| Retrieve Generated Image | HTTP Request             | Gets AI-generated image/video from Replicate | Wait for Processing       | Send Variation Image     | Gets the final generated image from Replicate after processing                                             |
| Note: Send Replicate Output | Sticky Note            | Describes sending Replicate output    |                           |                         | ### Telegram: Send Replicate Output<br>Notes dynamic chat ID for sending                                    |
| Send Variation Image     | Telegram                 | Sends AI-generated variations to user | Retrieve Generated Image   |                         | Sends the AI-generated variation back to the user                                                           |
| Setup Documentation      | Sticky Note              | Setup instructions for credentials    |                           |                         | ## 📝 Setup Instructions<br>Lists credential setup and testing instructions                                 |
| Note: Send Edited Image  | Sticky Note              | Notes sending edited image to Telegram |                           |                         | ### Telegram: Send Edited Image<br>Notes dynamic chat ID for sending edited image                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: message  
     - Additional Fields: enable file download  
   - Connect the output to "Edit Image (OpenAI)".  
   - Configure Telegram credentials with your bot token.

2. **Create HTTP Request node "Edit Image (OpenAI)":**  
   - URL: `https://api.openai.com/v1/images/edits`  
   - Method: POST  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - `image`: binary data from Telegram download (field: `data`)  
     - `prompt`: expression `={{ $json.message.caption }}`  
     - `model`: `gpt-image-1`  
     - `n`: 1  
     - `size`: `1024x1024`  
     - `quality`: `low`  
   - Authentication: Use OpenAI API credentials  
   - Connect output to "Convert to Binary Image".

3. **Create Convert To File node "Convert to Binary Image":**  
   - Operation: toBinary  
   - Source Property: `data[0].b64_json` (from OpenAI response)  
   - File Name: `edited_image.png`  
   - MIME Type: `image/png`  
   - Connect output to "Send Edited Image".

4. **Create Telegram node "Send Edited Image":**  
   - Operation: sendPhoto  
   - Chat ID: `={{ $json.message.chat.id }}`  
   - Binary Data: enable, sending binary from previous node  
   - Caption: "Here's your edited image! ✨"  
   - Connect output to "Get File Path".

5. **Create HTTP Request node "Get File Path":**  
   - URL: `https://api.telegram.org/bot{{$credentials.telegramBot.botToken}}/getFile`  
   - Method: POST  
   - Body (JSON): `{ "file_id": "{{ $json.result.photo[0].file_id }}" }`  
   - Connect output to "Generate Variation (Replicate)".

6. **Create HTTP Request node "Generate Variation (Replicate)":**  
   - URL: `https://api.replicate.com/v1/models/pixverse/pixverse-v4/predictions`  
   - Method: POST  
   - Body (JSON):  
     ```json
     {
       "input": {
         "prompt": "{{ $json.message.caption || 'a creative enhancement of this image' }}",
         "quality": "720p",
         "image": "https://api.telegram.org/file/bot{{$credentials.telegramBot.botToken}}/{{ $json.result.file_path }}"
       }
     }
     ```  
   - Headers: Authorization `Bearer {{$credentials.replicateApi.apiKey}}`  
   - Connect output to "Wait for Processing".

7. **Create Wait node "Wait for Processing":**  
   - Duration: 45 seconds  
   - Connect output to "Retrieve Generated Image".

8. **Create HTTP Request node "Retrieve Generated Image":**  
   - URL: `={{ $json.urls.get }}` (dynamic URL from Replicate response)  
   - Headers: Authorization `Bearer {{$credentials.replicateApi.apiKey}}`  
   - Response format: Base64 or appropriate for Telegram  
   - Connect output to "Send Variation Image".

9. **Create Telegram node "Send Variation Image":**  
   - Operation: sendDocument  
   - Chat ID: `={{ $json.message.chat.id }}`  
   - File: binary data from previous node  
   - Caption: "Here's an AI-generated variation of your image! 🚀"  
   - No further connections (end node).

10. **Credential Setup:**  
    - Configure Telegram Bot credentials with the bot token.  
    - Configure OpenAI API credentials with your OpenAI key.  
    - Configure Replicate API credentials with your Replicate token.

11. **Testing:**  
    - Send an image with a descriptive caption to your Telegram bot.  
    - Observe edited image sent back.  
    - Observe AI-generated variation sent optionally after processing delay.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires your Telegram bot to have permission to download files and send photos/documents.           | Setup instructions and Telegram Bot API documentation                                              |
| OpenAI’s image editing uses the `gpt-image-1` model with low quality for cost-effective and fast processing.       | OpenAI Image Editing API docs                                                                       |
| Replicate’s pixverse-v4 model generates AI variations/videos asynchronously, requiring a wait/polling mechanism.  | Replicate API documentation                                                                         |
| For security, bot tokens are embedded in URLs to access Telegram files; consider environment variables for safety. | Security best practices for Telegram Bot tokens                                                    |
| Chat IDs are dynamically extracted from the triggering Telegram message, enabling responses to the correct user.  | Telegram Bot API usage                                                                              |
| Sending large files to Telegram may fail; ensure file sizes comply with Telegram limits.                            | Telegram API documentation on file size limits                                                    |
| Adjust prompts and parameters in OpenAI and Replicate nodes to customize AI behavior.                              | AI prompt engineering best practices                                                               |
| Useful links:  
- [Telegram Bot API](https://core.telegram.org/bots/api)  
- [OpenAI API docs](https://platform.openai.com/docs/api-reference/images)  
- [Replicate API docs](https://replicate.com/docs/reference/http) | Documentation references for APIs used in the workflow                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---