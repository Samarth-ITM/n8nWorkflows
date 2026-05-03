Generate Enhanced AI Images via Telegram with DALL-E and GPT

https://n8nworkflows.xyz/workflows/generate-enhanced-ai-images-via-telegram-with-dall-e-and-gpt-4467


# Generate Enhanced AI Images via Telegram with DALL-E and GPT

### 1. Workflow Overview

This workflow, titled **"Generate Enhanced AI Images via Telegram with DALL-E and GPT"** (named internally as "PromptCraft AI"), automates the process of receiving image prompt requests via Telegram, enhancing those prompts using a GPT-based model, generating images with the DALL·E API, storing the images on Google Drive, logging metadata to Google Sheets, and finally sending the generated images back to the Telegram user. It is aimed at users or services that want to leverage AI-based image generation with improved prompt crafting through AI language models, fully integrated in a seamless chat interface.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Input Reception:** Receives user messages containing initial image prompt text.
- **1.2 AI Prompt Enhancement:** Uses GPT (OpenAI Chat Model) to enhance and engineer a better image prompt.
- **1.3 Image Generation:** Sends the enhanced prompt to the OpenAI DALL·E image generation API.
- **1.4 Image Download and Storage:** Downloads the generated image and stores it in Google Drive.
- **1.5 Logging:** Appends image metadata (title, URL) to a Google Sheet for record-keeping.
- **1.6 Image Response:** Sends the generated image back to the Telegram user.
- **1.7 Documentation/Notes:** Assists users with workflow context and demo links.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

- **Overview:** This block listens for incoming Telegram messages containing image prompt requests.
- **Nodes Involved:**  
  - Telegram Trigger
- **Node Details:**

| Node Name       | Details                                                                                                   |
|-----------------|-----------------------------------------------------------------------------------------------------------|
| Telegram Trigger| - Type: Telegram Trigger node (Webhook) <br> - Listens for 'message' updates from Telegram chat <br> - Credentials: Telegram API OAuth <br> - Output: JSON with message data <br> - Failure cases: Invalid webhook setup, Telegram API downtime, invalid credentials |

#### 1.2 AI Prompt Enhancement

- **Overview:** Enhances the user-provided text prompt by passing it to a GPT-based language model specialized in image prompt engineering.
- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Image Prompt (Langchain Agent)
- **Node Details:**

| Node Name       | Details                                                                                                   |
|-----------------|-----------------------------------------------------------------------------------------------------------|
| OpenAI Chat Model | - Type: Langchain OpenAI Chat Model node <br> - Model: "gpt-4o-mini" <br> - Role: Generates an enhanced prompt for image generation based on the user's input <br> - Credentials: OpenAI API Key <br> - Connects output to Image Prompt node for further processing <br> - Potential failures: API key invalid, rate limits, model unavailability |
| Image Prompt    | - Type: Langchain Agent node <br> - Purpose: Acts as an expert image prompt engineer to refine and define the prompt <br> - Input: Receives text from Telegram Trigger and enhanced language model output <br> - Configuration: System message defines expert role; prompt type is "define" <br> - Output: Text prompt optimized for image generation <br> - Failure points: Expression errors, Langchain node issues |

#### 1.3 Image Generation

- **Overview:** Sends the enhanced prompt to OpenAI’s DALL·E endpoint to generate an image.
- **Nodes Involved:**  
  - Create Image (HTTP Request)  
  - HTTP Request (for downloading image)
- **Node Details:**

| Node Name    | Details                                                                                                   |
|--------------|-----------------------------------------------------------------------------------------------------------|
| Create Image | - Type: HTTP Request <br> - Method: POST <br> - URL: OpenAI Image Generation API endpoint <br> - Body: JSON with prompt from Image Prompt node, size 1024x1024 <br> - Headers: Authorization with Bearer token <br> - Output: JSON containing URL(s) of generated image(s) <br> - Failure points: API key invalid, network issues, API limits, malformed prompt |
| HTTP Request | - Type: HTTP Request <br> - Method: GET (default) <br> - URL: Extracted image URL from Create Image node output <br> - Purpose: Downloads the actual image binary data for storage and sending <br> - Failure points: URL invalid or expired, network timeout |

#### 1.4 Image Download and Storage

- **Overview:** Uploads the downloaded image to Google Drive for permanent storage under a folder named "ai image".
- **Nodes Involved:**  
  - Google Drive
- **Node Details:**

| Node Name    | Details                                                                                                   |
|--------------|-----------------------------------------------------------------------------------------------------------|
| Google Drive | - Type: Google Drive node <br> - Operation: Upload file to "My Drive" root or specific folder "ai image" <br> - Input: Binary image data from HTTP Request node <br> - Credentials: Google Drive OAuth2 <br> - Failure cases: Authentication failure, insufficient permissions, file size limits |

#### 1.5 Logging

- **Overview:** Records metadata about the generated image (title and Google Drive link) into a Google Sheet for tracking and auditing.
- **Nodes Involved:**  
  - Image Log (Google Sheets)
- **Node Details:**

| Node Name | Details                                                                                                   |
|-----------|-----------------------------------------------------------------------------------------------------------|
| Image Log | - Type: Google Sheets node <br> - Operation: Append row <br> - Sheet: "Sheet1" in a specified Google Sheet document <br> - Columns: Title (image name), Heygen video URL (Google Drive link) <br> - Credentials: Google Sheets OAuth2 <br> - Failure cases: Sheet access denied, invalid spreadsheet ID, API quota exceeded |

#### 1.6 Image Response

- **Overview:** Sends the generated image back to the Telegram user who requested it.
- **Nodes Involved:**  
  - Send Photo (Telegram)
- **Node Details:**

| Node Name  | Details                                                                                                   |
|------------|-----------------------------------------------------------------------------------------------------------|
| Send Photo | - Type: Telegram node <br> - Operation: sendPhoto <br> - Uses chatId from original Telegram message <br> - Sends binary image downloaded earlier <br> - Credentials: Telegram API OAuth <br> - Failure points: Invalid chat ID, Telegram API errors, binary data issues |

#### 1.7 Documentation/Notes

- **Overview:** Includes sticky notes for model and process documentation and a Loom demo link for user guidance.
- **Nodes Involved:**  
  - Sticky Note2 (Model block title)  
  - Sticky Note6 (Write to Drive & Sheets)  
  - Sticky Note7 (Send Image)  
  - Sticky Note (Loom Link Demo)
- **Node Details:**

| Node Name   | Details                                                                                                   |
|-------------|-----------------------------------------------------------------------------------------------------------|
| Sticky Note2 | Title note for the AI Model block                                                                        |
| Sticky Note6 | Title note for the "Write to Drive & Sheets" block                                                       |
| Sticky Note7 | Title note for the "Send Image" block                                                                    |
| Sticky Note  | Contains Loom video demo link: https://www.loom.com/share/9d4743b32c204b189a237d8b9446f45d?sid=af5f3a1d-fb78-4bba-ac01-f4f29ac7f14a |

---

### 3. Summary Table

| Node Name        | Node Type                             | Functional Role                      | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|------------------|-------------------------------------|------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger  | Telegram Trigger                    | Receive Telegram messages           | —                      | Image Prompt            |                                                                                                |
| Image Prompt     | Langchain Agent                    | Enhance user image prompt           | Telegram Trigger, OpenAI Chat Model | Create Image            |                                                                                                |
| OpenAI Chat Model | Langchain OpenAI Chat Model        | Generate enhanced prompt via GPT   | —                      | Image Prompt            |                                                                                                |
| Create Image      | HTTP Request                      | Call OpenAI DALL·E API              | Image Prompt            | HTTP Request            |                                                                                                |
| HTTP Request     | HTTP Request                      | Download generated image            | Create Image            | Google Drive, Send Photo |                                                                                                |
| Google Drive      | Google Drive                      | Upload image to Drive folder        | HTTP Request            | Image Log                |                                                                                                |
| Image Log         | Google Sheets                    | Log image metadata                  | Google Drive            | —                       |                                                                                                |
| Send Photo        | Telegram                        | Send image back to user             | HTTP Request            | —                       |                                                                                                |
| Sticky Note2      | Sticky Note                      | "## Model" section title            | —                      | —                       |                                                                                                |
| Sticky Note6      | Sticky Note                      | "# Write to Drive & Sheets" section | —                      | —                       |                                                                                                |
| Sticky Note7      | Sticky Note                      | "# Send Image" section title        | —                      | —                       |                                                                                                |
| Sticky Note       | Sticky Note                      | Loom Link Demo                      | —                      | —                       | **https://www.loom.com/share/9d4743b32c204b189a237d8b9446f45d?sid=af5f3a1d-fb78-4bba-ac01-f4f29ac7f14a** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure credentials with Telegram Bot API OAuth2  
   - Set to listen for message updates  
   - Position: Start of workflow

2. **Add OpenAI Chat Model Node:**  
   - Type: Langchain OpenAI Chat Model  
   - Credentials: OpenAI API key OAuth2  
   - Model: Select "gpt-4o-mini" or equivalent GPT model  
   - Connect input as needed (from Telegram Trigger or separate if applicable)  
   - Purpose: Enhance user prompt text

3. **Add Image Prompt Node (Langchain Agent):**  
   - Type: Langchain Agent node  
   - Configure system message: "Overview\nYou are an expert image prompt engineer."  
   - Prompt type: "define"  
   - Input expression: Use Telegram message text or output from OpenAI Chat Model  
   - Connect OpenAI Chat Model output to this node’s AI language model input  
   - Connect Telegram Trigger main output to this node’s main input

4. **Add Create Image HTTP Request Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.openai.com/v1/images/generations`  
   - Headers: Authorization with Bearer token for OpenAI API (DALL·E)  
   - Body parameters:  
     - prompt: Output from Image Prompt node  
     - size: "1024x1024"  
   - Connect output of Image Prompt node to this node

5. **Add HTTP Request Node to Download Image:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression to extract the image URL from Create Image node output (`{{$json.data[0].url}}`)  
   - Connect output of Create Image node to this node

6. **Add Google Drive Node:**  
   - Type: Google Drive  
   - Credentials: Google Drive OAuth2  
   - Operation: Upload file  
   - Destination: Folder "ai image" or root of My Drive  
   - Input: Binary data from HTTP Request (image download) node  
   - Connect output of HTTP Request (image download) node

7. **Add Google Sheets Node (Image Log):**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2  
   - Operation: Append row  
   - Document ID: Specify Google Sheet for logs  
   - Sheet Name: "Sheet1" (or as per your setup)  
   - Columns to append:  
     - Title: Use the file name or image descriptor from Google Drive node output  
     - Heygen video url: Use Google Drive file URL (`webViewLink`)  
   - Connect output of Google Drive node

8. **Add Telegram Send Photo Node:**  
   - Type: Telegram node  
   - Operation: sendPhoto  
   - Credentials: Telegram API OAuth2  
   - Chat ID: Use expression to extract from original Telegram Trigger message (`{{$json.message.from.id}}`)  
   - Binary data: Use the downloaded image binary from HTTP Request node  
   - Connect output of HTTP Request (image download) node

9. **(Optional) Add Sticky Notes for Documentation:**  
   - Add Sticky Note nodes to describe functional blocks, e.g., “## Model”, “# Write to Drive & Sheets”, “# Send Image”  
   - Add Sticky Note with Loom demo link for user guidance

10. **Connect nodes following the data flow:**  
    - Telegram Trigger → OpenAI Chat Model → Image Prompt → Create Image → HTTP Request (download) → Google Drive → Image Log  
    - HTTP Request (download) → Send Photo

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Loom demo video showcasing the workflow in action                                                                                | https://www.loom.com/share/9d4743b32c204b189a237d8b9446f45d?sid=af5f3a1d-fb78-4bba-ac01-f4f29ac7f14a                |
| Workflow uses OpenAI GPT model "gpt-4o-mini" for prompt enhancement and DALL·E for image generation                              | Requires valid OpenAI API keys for ChatGPT and DALL·E endpoints                                                  |
| Google Drive and Google Sheets nodes require OAuth2 credentials with appropriate file and sheet access permissions               |                                                                                                                  |
| Telegram API nodes require Telegram Bot credentials, with webhook correctly configured                                            |                                                                                                                  |
| Possible failure points include API rate limits, network timeouts, invalid credentials, and expression evaluation errors         | Error handling should be considered especially around external API calls                                         |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.