Nano Banana AI Image Editor via Telegram

https://n8nworkflows.xyz/workflows/nano-banana-ai-image-editor-via-telegram-8082


# Nano Banana AI Image Editor via Telegram

### 1. Workflow Overview

This workflow, titled **Nano Banana AI Image Editor via Telegram**, is designed to receive photos sent to a Telegram bot, process these images using an AI vision model (Nano Banana based on Gemini 2.5 Flash via OpenRouter), and send back the AI-analyzed or enhanced image to the user via Telegram. It targets use cases involving AI-driven image editing or analysis directly integrated within Telegram messaging.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming photo messages from Telegram.
- **1.2 Photo Preparation:** Downloads the photo, converts it to Base64, and formats it as a data URL for AI processing.
- **1.3 AI Processing:** Sends the formatted image and optional caption to the AI model for analysis or editing.
- **1.4 Response Handling:** Parses the AI’s response, converts the returned image from Base64 back to binary, and sends it to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming Telegram messages that contain photos and triggers the workflow.
- **Nodes Involved:** 
  - Photo Message Receiver
- **Node Details:**

  - **Photo Message Receiver**
    - Type: Telegram Trigger
    - Role: Entry point for the workflow, monitors Telegram updates specifically for messages with photos.
    - Configuration: Listens for the "message" update type; no additional filters.
    - Expressions: Extracts the first photo’s file ID from the incoming message payload.
    - Inputs: None (trigger node).
    - Outputs: Photo message data to next node.
    - Edge Cases: No photo in message — workflow will not trigger; Telegram API downtime or invalid bot token can cause failures.
    - Version: 1.2

#### 1.2 Photo Preparation

- **Overview:** Downloads the photo from Telegram, converts the binary photo file to Base64 string, and formats it as a data URL, preparing it for the AI API.
- **Nodes Involved:**
  - Download Telegram Photo
  - Convert Photo to Base64
  - Format Image Data URL
- **Node Details:**

  - **Download Telegram Photo**
    - Type: Telegram node (API client)
    - Role: Fetches the photo file from Telegram servers using the file_id.
    - Configuration: Uses the file ID from the Photo Message Receiver node; resource set to "file."
    - Inputs: Photo file ID from Telegram trigger.
    - Outputs: Binary photo data.
    - Edge Cases: File ID invalid or expired; Telegram API rate limits; network errors.
    - Version: 1.2

  - **Convert Photo to Base64**
    - Type: Extract From File
    - Role: Converts binary photo data to Base64 string representation.
    - Configuration: Operation set to "binaryToProperty," extracting Base64 to JSON property `data`.
    - Inputs: Binary photo file.
    - Outputs: JSON with Base64 encoded image.
    - Edge Cases: Binary data missing or corrupt.
    - Version: 1

  - **Format Image Data URL**
    - Type: Code (JavaScript)
    - Role: Transforms the Base64 string into a proper data URL format (`data:image/png;base64,<base64>`).
    - Configuration: Custom JS code mapping all incoming items to data URL objects with property `url`.
    - Inputs: JSON containing Base64 string.
    - Outputs: JSON with `url` property containing data URL.
    - Edge Cases: Missing or malformed Base64 string causing invalid URL.
    - Version: 2

#### 1.3 AI Processing

- **Overview:** Sends the formatted image data URL along with an optional caption text to the Nano Banana AI model hosted via OpenRouter API for image analysis or editing.
- **Nodes Involved:**
  - Nano Banana Image Processor
- **Node Details:**

  - **Nano Banana Image Processor**
    - Type: HTTP Request
    - Role: Calls the OpenRouter API endpoint for Gemini 2.5 Flash Image Preview model.
    - Configuration:
      - Method: POST
      - URL: `https://openrouter.ai/api/v1/chat/completions`
      - Body: JSON containing the model name and messages array with user role content including the caption text and image URL.
      - Auth: Uses predefined OpenRouter API credentials with Bearer token header.
    - Expressions:
      - Incorporates Telegram photo caption via `$('Photo Message Receiver').item.json.message.caption`.
      - Uses formatted image URL from previous node.
    - Inputs: JSON with `url` property.
    - Outputs: AI JSON response containing processed image URL.
    - Edge Cases: API key missing or invalid; network timeout; rate limiting; response errors; empty or missing caption.
    - Version: 4.2

#### 1.4 Response Handling

- **Overview:** Extracts the Base64 image URL from the AI response, converts it back into binary file format, and sends the processed photo back to the Telegram chat.
- **Nodes Involved:**
  - Parse AI Response Data
  - Base64 to Binary File
  - Send Processed Photo
- **Node Details:**

  - **Parse AI Response Data**
    - Type: Set Node
    - Role: Extracts the Base64 encoded processed image from the nested AI response JSON.
    - Configuration:
      - Assigns a new string field `base` by splitting the AI image URL on comma and taking the second part (the Base64 data).
    - Inputs: AI JSON response.
    - Outputs: JSON with `base` property.
    - Edge Cases: AI response structure changes; missing image URL; malformed Base64 data.
    - Version: 3.4

  - **Base64 to Binary File**
    - Type: Convert To File
    - Role: Converts the Base64 string back to binary format suitable for sending as a photo.
    - Configuration: Operation set to "toBinary" from the `base` property.
    - Inputs: Base64 string from previous node.
    - Outputs: Binary file data.
    - Edge Cases: Invalid Base64 string causing conversion failure.
    - Version: 1.1

  - **Send Processed Photo**
    - Type: Telegram node (API client)
    - Role: Sends the processed photo back to the Telegram chat.
    - Configuration:
      - Chat ID: Must be set to the target chat (currently placeholder "YOUR_CHAT_ID_HERE").
      - Operation: sendPhoto with binary data enabled.
    - Inputs: Binary photo file.
    - Outputs: Confirmation of message sent.
    - Edge Cases: Incorrect chat ID; Telegram API errors; binary data missing.
    - Version: 1.2

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                  | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                      |
|-------------------------|-------------------------|---------------------------------|-----------------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| Photo Message Receiver   | Telegram Trigger        | Receives incoming photo messages | None                        | Download Telegram Photo       | # 🤖 Nano Banana AI Image Editor - Overview and process flow detailed in sticky note             |
| Download Telegram Photo  | Telegram API            | Downloads photo from Telegram    | Photo Message Receiver      | Convert Photo to Base64       | # 🤖 Nano Banana AI Image Editor - Overview and process flow detailed in sticky note             |
| Convert Photo to Base64  | Extract From File       | Converts binary to Base64 string | Download Telegram Photo     | Format Image Data URL         | # 🤖 Nano Banana AI Image Editor - Overview and process flow detailed in sticky note             |
| Format Image Data URL    | Code (JavaScript)       | Formats Base64 as data URL       | Convert Photo to Base64     | Nano Banana Image Processor   | # 🤖 Nano Banana AI Image Editor - Overview and process flow detailed in sticky note             |
| Nano Banana Image Processor | HTTP Request          | Sends image + caption to AI      | Format Image Data URL       | Parse AI Response Data        | # 🤖 Nano Banana AI Image Editor - Overview and process flow detailed in sticky note             |
| Parse AI Response Data   | Set                     | Extracts Base64 image from AI response | Nano Banana Image Processor | Base64 to Binary File         | # 🤖 Nano Banana AI Image Editor - Overview and process flow detailed in sticky note             |
| Base64 to Binary File    | Convert To File         | Converts Base64 back to binary   | Parse AI Response Data      | Send Processed Photo          | # 🤖 Nano Banana AI Image Editor - Overview and process flow detailed in sticky note             |
| Send Processed Photo     | Telegram API            | Sends processed photo to Telegram | Base64 to Binary File       | None                         | # 🤖 Nano Banana AI Image Editor - Overview and process flow detailed in sticky note             |
| Sticky Note             | Sticky Note             | Documentation and overview       | None                        | None                         | # 🤖 Nano Banana AI Image Editor - Full workflow overview and process flow description           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Name: Photo Message Receiver
   - Type: Telegram Trigger
   - Parameters: Listen for `message` updates with photos
   - Credential: Connect your Telegram Bot API credentials
   - Position: Start node

2. **Add Telegram API Node to Download Photo**
   - Name: Download Telegram Photo
   - Type: Telegram node
   - Parameters:
     - Resource: File
     - File ID: Use expression `{{$node["Photo Message Receiver"].item.json.message.photo[0].file_id}}`
   - Credential: Use same Telegram credentials
   - Connect output of Photo Message Receiver to this node

3. **Add Extract From File Node to Convert Binary to Base64**
   - Name: Convert Photo to Base64
   - Type: Extract From File
   - Parameters:
     - Operation: binaryToProperty
     - Property Name: (default `data`)
   - Connect Download Telegram Photo output here

4. **Add Code Node to Format Image Data URL**
   - Name: Format Image Data URL
   - Type: Code (JavaScript)
   - Parameters: Use this JS code:
     ```js
     const items = $input.all();
     const updatedItems = items.map((item) => {
       const base64Url = item?.json?.data;
       const url = `data:image/png;base64,${base64Url}`;
       return { url };
     });
     return updatedItems;
     ```
   - Connect Convert Photo to Base64 output here

5. **Add HTTP Request Node for AI Processing**
   - Name: Nano Banana Image Processor
   - Type: HTTP Request
   - Parameters:
     - URL: `https://openrouter.ai/api/v1/chat/completions`
     - Method: POST
     - Authentication: Predefined credential (OpenRouter API)
     - Headers: Authorization Bearer token from credentials
     - Body Content Type: JSON
     - JSON Body:
       ```json
       {
         "model": "google/gemini-2.5-flash-image-preview:free",
         "messages": [
           {
             "role": "user",
             "content": [
               {
                 "type": "text",
                 "text": "{{ $('Photo Message Receiver').item.json.message.caption }}"
               },
               {
                 "type": "image_url",
                 "image_url": {
                   "url": "{{ $json.url }}"
                 }
               }
             ]
           }
         ]
       }
       ```
   - Connect Format Image Data URL output here

6. **Add Set Node to Parse AI Response**
   - Name: Parse AI Response Data
   - Type: Set
   - Parameters:
     - Add a string field named `base`
     - Value expression: `={{ $json.choices[0].message.images[0].image_url.url.split(',')[1] }}`
   - Connect Nano Banana Image Processor output here

7. **Add Convert To File Node to Convert Base64 to Binary**
   - Name: Base64 to Binary File
   - Type: Convert To File
   - Parameters:
     - Operation: toBinary
     - Source Property: `base`
   - Connect Parse AI Response Data output here

8. **Add Telegram Node to Send Processed Photo**
   - Name: Send Processed Photo
   - Type: Telegram node
   - Parameters:
     - Operation: sendPhoto
     - Chat ID: Set your target Telegram chat ID (replace placeholder `YOUR_CHAT_ID_HERE`)
     - Binary Data: enabled (send binary photo)
   - Credential: Telegram Bot credentials
   - Connect Base64 to Binary File output here

9. **Add Sticky Note for Documentation (Optional)**
   - Add a sticky note node with workflow overview and process flow for future reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Uses the Nano Banana AI model based on Google Gemini 2.5 Flash Image Preview via OpenRouter API (free tier).           | OpenRouter API Docs: https://openrouter.ai/docs                                                  |
| Telegram Bot must be configured with appropriate permissions to receive and send photos.                              | Telegram Bot API Docs: https://core.telegram.org/bots/api                                        |
| Replace `"YOUR_CHAT_ID_HERE"` in the Send Processed Photo node with actual chat ID to enable message delivery.        | How to get Chat ID: https://stackoverflow.com/questions/32423837/telegram-bot-how-to-get-a-chat-id |
| The AI response parsing assumes a consistent JSON structure from OpenRouter Gemini API, which may change over time.    | Monitor API response changes to update parsing logic accordingly.                                |
| The workflow currently processes only the first photo in a Telegram message; multi-photo messages are not supported.  | To extend, modify the logic to iterate over all photo items in the message payload.              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all current content policies and containing no illegal or offensive material. All data processed is legal and publicly accessible.