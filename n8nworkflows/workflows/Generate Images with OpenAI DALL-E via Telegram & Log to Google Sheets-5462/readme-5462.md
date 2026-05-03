Generate Images with OpenAI DALL-E via Telegram & Log to Google Sheets

https://n8nworkflows.xyz/workflows/generate-images-with-openai-dall-e-via-telegram---log-to-google-sheets-5462


# Generate Images with OpenAI DALL-E via Telegram & Log to Google Sheets

### 1. Workflow Overview

This workflow automates the generation of images based on user prompts received via Telegram, using OpenAI’s DALL·E model, and logs the prompt and resulting image URL into a Google Sheets document. After the image is generated and logged, the workflow sends the image back to the original Telegram user. The workflow is logically divided into the following blocks:

- **1.1 Input Reception (Telegram Trigger):** Receives user messages with image generation requests from Telegram.
- **1.2 AI Image Generation (OpenAI Node):** Processes the user’s prompt to generate an image using OpenAI’s DALL·E.
- **1.3 Data Logging (Google Sheets Node):** Records the prompt and the generated image URL into a Google Sheet for tracking and history.
- **1.4 Output Delivery (Telegram Node):** Sends the generated image back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures incoming messages from Telegram users that contain image generation requests. It acts as the entry point for the workflow.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - **Type:** Telegram Trigger node (Base)  
    - **Role:** Listens for new incoming Telegram messages to start the workflow.  
    - **Configuration:**  
      - Watches for "message" update types exclusively.  
      - Uses a specific webhook ID to receive Telegram updates.  
      - Credentials configured for a Telegram account named "Telegram account 2".  
    - **Key Expressions:** None directly, but output JSON includes the full Telegram message object.  
    - **Input:** External webhook (Telegram updates).  
    - **Output:** JSON containing message text, chat ID, and other Telegram metadata.  
    - **Version:** 1.2  
    - **Edge Cases / Failures:**  
      - Telegram API connectivity issues or invalid webhook setup.  
      - Unrecognized or unsupported message types (non-text messages may not trigger image generation).  
      - Rate limits or authorization errors with Telegram API.

#### 2.2 AI Image Generation

- **Overview:**  
  Generates an image based on the text prompt received from the user via Telegram using OpenAI’s image generation capabilities.

- **Nodes Involved:**  
  - OpenAI (Langchain OpenAI node)

- **Node Details:**

  - **OpenAI**  
    - **Type:** Langchain OpenAI node (Image resource)  
    - **Role:** Sends the user's prompt to OpenAI’s DALL·E model to generate an image.  
    - **Configuration:**  
      - Prompt is dynamically sourced from the Telegram message text (`{{$json.message.text}}`).  
      - Options specify image size as 1024x1024 pixels.  
      - Image quality set to "standard".  
      - Configured to return image URLs (not just image binary data).  
      - Resource set to "image" to trigger image generation endpoint.  
    - **Credentials:** OpenAI API key associated with "OpenAi account Dave".  
    - **Input:** JSON with Telegram message text as prompt.  
    - **Output:** JSON containing the generated image URL(s) and revised prompt.  
    - **Version:** 1.8  
    - **Edge Cases / Failures:**  
      - API rate limits or quota exceeded errors.  
      - Invalid or empty prompt text leading to failed generation.  
      - Network issues or timeouts contacting OpenAI API.  
      - Unexpected response formats or missing image URLs.

#### 2.3 Data Logging

- **Overview:**  
  Logs the generated image URL and the prompt text into a Google Sheets spreadsheet for record-keeping.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - **Type:** Google Sheets node (Base)  
    - **Role:** Appends a new row to a defined sheet containing the image URL and its description.  
    - **Configuration:**  
      - Operation set to "append".  
      - Target sheet is "Sheet1" identified by the gid=0.  
      - Document ID points to a Google Sheet named "image database".  
      - Mapping mode defines two columns:  
        - "Image title" set to the generated image URL (`{{$json.url}}`)  
        - "Image description" set to the revised prompt from OpenAI (`{{$json.revised_prompt}}`)  
    - **Credentials:** Google Sheets OAuth2 credentials named "Google Sheets account 2".  
    - **Input:** JSON output from OpenAI node with image URL and prompt.  
    - **Output:** Confirmation of append operation to the sheet.  
    - **Version:** 4.6  
    - **Edge Cases / Failures:**  
      - OAuth credential expiration or revocation.  
      - Incorrect spreadsheet ID or sheet name causing append failures.  
      - API quota limits or connectivity issues.  
      - Mismatched column schema or data type errors.

#### 2.4 Output Delivery

- **Overview:**  
  Sends the generated image back to the user on Telegram as a photo message, completing the user interaction loop.

- **Nodes Involved:**  
  - Telegram (Send Photo node)

- **Node Details:**

  - **Telegram**  
    - **Type:** Telegram node (Base)  
    - **Role:** Sends the generated image as a photo back to the Telegram chat.  
    - **Configuration:**  
      - Operation set to "sendPhoto".  
      - Photo file parameter dynamically set from the Google Sheets logged "Image title" field (which holds the image URL): `{{$json["Image title"]}}`.  
      - Chat ID set dynamically from the original Telegram trigger message chat ID (`{{$('Telegram Trigger').item.json.message.chat.id}}`).  
    - **Credentials:** Same Telegram API credentials as the Telegram Trigger node ("Telegram account 2").  
    - **Input:** JSON containing image URL from Google Sheets and chat metadata from Telegram Trigger.  
    - **Output:** Confirmation of the photo message sent.  
    - **Version:** 1.2  
    - **Edge Cases / Failures:**  
      - Invalid or expired Telegram API credentials.  
      - Malformed or inaccessible image URLs causing send failure.  
      - Chat ID not found or user blocked the bot.  
      - Telegram API rate limits or connectivity errors.

---

### 3. Summary Table

| Node Name       | Node Type                      | Functional Role                 | Input Node(s)      | Output Node(s)   | Sticky Note                                                                                                                         |
|-----------------|--------------------------------|--------------------------------|--------------------|------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger| Telegram Trigger                | Input Reception from Telegram  | -                  | OpenAI           |                                                                                                                                     |
| OpenAI          | Langchain OpenAI (Image)       | Generate image via OpenAI DALL·E| Telegram Trigger   | Google Sheets    |                                                                                                                                     |
| Google Sheets   | Google Sheets                  | Log prompt and image URL       | OpenAI             | Telegram         |                                                                                                                                     |
| Telegram        | Telegram node                  | Send generated image to user   | Google Sheets      | -                |                                                                                                                                     |
| Sticky Note     | Sticky Note                   | Describes workflow overview    | -                  | -                | ## Telegram AI Image Generator + Google Sheets Logger 1. Telegram Trigger Receives image generation requests from users directly via Telegram. 2. OpenAI Node Processes the request by generating the required image based on the user’s prompt. 3. Saves the prompt and generated image link to Google Sheets. 4. Telegram Send Message Sends the generated image back to the user in Telegram as a seamless automated response. Loom Demo : [Link](https://www.loom.com/share/1c5e645442f6441baf9efd12a334eef0) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" update type only.  
   - Set up webhook ID (auto-generated or custom).  
   - Connect Telegram API credentials for the bot ("Telegram account 2").  
   - Position at the start of the workflow.

2. **Create OpenAI Node**  
   - Type: Langchain OpenAI node  
   - Resource: Image  
   - Prompt: Set expression to `{{$json.message.text}}` to use Telegram message text.  
   - Options:  
     - Size: 1024x1024  
     - Dalle Quality: standard  
     - Return Image URLs: enabled  
   - Connect OpenAI API credentials ("OpenAi account Dave").  
   - Connect output of Telegram Trigger node to input of OpenAI node.

3. **Create Google Sheets Node**  
   - Type: Google Sheets node  
   - Operation: Append  
   - Document ID: Use your Google Sheet ID (e.g., "1iABxBKBY9ERctigW1bYvvRTIAf-m-InTSvSKnqANUhg")  
   - Sheet Name: "Sheet1" (gid=0)  
   - Mapping Columns:  
     - Image title → map to `{{$json.url}}` (image URL from OpenAI output)  
     - Image description → map to `{{$json.revised_prompt}}` (prompt text or revised prompt from OpenAI output)  
   - Connect Google Sheets OAuth2 credentials ("Google Sheets account 2").  
   - Connect output of OpenAI node to input of Google Sheets node.

4. **Create Telegram Node (Send Photo)**  
   - Type: Telegram node  
   - Operation: Send Photo  
   - File: Set expression to `{{$json["Image title"]}}` to use the logged image URL from Google Sheets output.  
   - Chat ID: Set expression to `{{$('Telegram Trigger').item.json.message.chat.id}}` to send back to the original chat.  
   - Connect the same Telegram API credentials as the trigger node.  
   - Connect output of Google Sheets node to input of Telegram node.

5. **Connect Nodes Sequentially:**  
   - Telegram Trigger → OpenAI → Google Sheets → Telegram (Send Photo)

6. **Test Workflow:**  
   - Deploy and activate the workflow.  
   - Send a text message to your Telegram bot with an image generation prompt.  
   - Verify image generation, logging in Google Sheets, and image delivery back on Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automates AI image generation on Telegram and logs results for audit and reference.                                               | Workflow purpose                                                                                   |
| Loom demo video walkthrough available for visual guidance.                                                                                      | [https://www.loom.com/share/1c5e645442f6441baf9efd12a334eef0](https://www.loom.com/share/1c5e645442f6441baf9efd12a334eef0) |
| Ensure OpenAI and Google Sheets credentials have appropriate API scopes and permissions.                                                        | Credential setup best practices                                                                    |
| Telegram bot must have permissions to receive messages and send media files.                                                                    | Telegram bot configuration                                                                         |
| Handle potential failures gracefully by adding error workflows or node-level error handling if scaling or production use is intended.          | Error handling recommendation                                                                      |

---

*Disclaimer:* The text provided is extracted exclusively from an n8n automated workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.