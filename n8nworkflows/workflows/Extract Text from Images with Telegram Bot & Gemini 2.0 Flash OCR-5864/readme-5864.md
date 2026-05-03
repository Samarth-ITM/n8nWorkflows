Extract Text from Images with Telegram Bot & Gemini 2.0 Flash OCR

https://n8nworkflows.xyz/workflows/extract-text-from-images-with-telegram-bot---gemini-2-0-flash-ocr-5864


# Extract Text from Images with Telegram Bot & Gemini 2.0 Flash OCR

### 1. Workflow Overview

This workflow enables automatic extraction of text content from images sent via Telegram to a bot. Users send images to the Telegram bot, which downloads the image, converts it for processing, and sends it to the Gemini 2.0 Flash OCR API (Google’s generative language API tailored for OCR). The extracted text is then sent back to the user within Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives image messages from Telegram users via a Telegram bot trigger and extracts necessary metadata.
- **1.2 File Retrieval and Preparation:** Downloads the image file from Telegram servers and prepares the binary data for OCR processing.
- **1.3 OCR Processing:** Sends the image data to the Gemini OCR HTTP API to extract text content.
- **1.4 Output Delivery:** Sends the extracted text back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for incoming Telegram messages and extracts chat ID and the highest resolution image file ID from the message payload.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Clean Input Data

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Initiates workflow on receiving a Telegram message update.  
    - Configuration: Watches for `"message"` updates exclusively, with automatic download enabled (for image files).  
    - Credentials: Uses an OAuth2 Telegram API credential linked to the bot `@RuriImageReader_bot`.  
    - Inputs: Webhook receives Telegram updates.  
    - Outputs: Emits message JSON including chat and photo metadata.  
    - Edge Cases/Potential Failures:  
      - Telegram API downtime or invalid bot token could cause auth errors.  
      - Receiving non-image messages results in no valid photo array.  
      - Large images might delay webhook response.

  - **Clean Input Data**  
    - Type: Set node  
    - Role: Extracts and simplifies the incoming data to only relevant fields: `chatID` and the largest image `file_id`.  
    - Configuration:  
      - Assigns `chatID` from `message.chat.id` (number).  
      - Assigns `Image` from last element of `message.photo` array’s `file_id`. This fetches the highest resolution image from multiple sizes.  
    - Inputs: Receives raw Telegram message JSON from Trigger node.  
    - Outputs: JSON with properties `chatID` and `Image` (file_id string).  
    - Edge Cases/Potential Failures:  
      - If no photo is present (e.g., text message), expression may fail or produce undefined.  
      - Photo array indexing assumes at least one photo available.

#### 2.2 File Retrieval and Preparation

- **Overview:**  
  Downloads the image file from Telegram using the extracted file ID and prepares binary data for passing to the OCR API.

- **Nodes Involved:**  
  - get file  
  - Extract from File

- **Node Details:**

  - **get file**  
    - Type: Telegram node (Resource: file)  
    - Role: Downloads the actual image file from Telegram servers using the file ID.  
    - Configuration:  
      - `fileId` parameter set dynamically to the cleaned `Image` property (removing any newline characters).  
      - Uses the same Telegram API credentials as the trigger.  
    - Inputs: Receives JSON with `Image` file_id from Clean Input Data node.  
    - Outputs: Returns binary file data of the image.  
    - Edge Cases/Potential Failures:  
      - Invalid or expired file_id leads to API errors.  
      - Network issues causing download failures.  
      - Large files may cause timeouts.

  - **Extract from File**  
    - Type: Extract From File node  
    - Role: Converts binary file data into a property format accessible in JSON for further processing.  
    - Configuration: Operation set to `binaryToProperty`, which extracts binary data into a JSON property (`data`).  
    - Inputs: Binary data from get file node.  
    - Outputs: JSON with base64-encoded image data in the `data` property.  
    - Edge Cases/Potential Failures:  
      - Binary extraction errors if file data is corrupted or incomplete.  
      - Potential data size limits on base64 encoding.

#### 2.3 OCR Processing

- **Overview:**  
  Sends the prepared image data to the Gemini 2.0 Flash OCR API endpoint for text extraction.

- **Nodes Involved:**  
  - Gemini OCR

- **Node Details:**

  - **Gemini OCR**  
    - Type: HTTP Request node  
    - Role: Calls Google’s Gemini OCR API to extract text from the image.  
    - Configuration:  
      - HTTP Method: POST  
      - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
      - Authentication: Generic Credential Type with Query Auth (using credentials created from Google AI Studio).  
      - Request Body (JSON):  
        ```
        {
          "contents": [
            {
              "role": "user",
              "parts": [
                {"inlineData": {"mimeType": "image/jpeg", "data": "{{ $json.data }}"}},
                {"text": "Extract text"}
              ]
            }
          ]
        }
        ```  
        The `$json.data` variable is the base64 image data extracted earlier.  
      - Sends JSON body with the inline image data and the instruction "Extract text".  
    - Inputs: Receives JSON containing base64 image data from Extract from File node.  
    - Outputs: JSON response containing OCR extracted text (typically in the `.output` property).  
    - Edge Cases/Potential Failures:  
      - API authentication failure or quota exhaustion.  
      - Malformed JSON or incorrect base64 data causes bad request errors.  
      - API response delays or timeouts.  
      - Unexpected schema or empty OCR results.

#### 2.4 Output Delivery

- **Overview:**  
  Sends the extracted text back to the Telegram user via the bot.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - Type: Telegram node (send message)  
    - Role: Sends the OCR result text to the user chat ID that initiated the request.  
    - Configuration:  
      - `text` parameter set dynamically to the `output` property from Gemini OCR API response.  
      - `chatId` parameter dynamically set from `chatID` extracted initially (Clean Input Data node).  
      - `appendAttribution` disabled (no attribution appended).  
      - Uses the same Telegram API credentials as before.  
    - Inputs: Receives OCR text output from Gemini OCR node and chat ID from Clean Input Data node (via expression).  
    - Outputs: Confirmation of message sent to Telegram.  
    - Edge Cases/Potential Failures:  
      - Telegram API errors if chat ID invalid or message content is empty.  
      - Rate limits on Telegram messages.  
      - Large extracted text might be truncated or rejected.

---

### 3. Summary Table

| Node Name       | Node Type                 | Functional Role                  | Input Node(s)       | Output Node(s)     | Sticky Note                                                                                                   |
|-----------------|---------------------------|--------------------------------|---------------------|--------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger| Telegram Trigger           | Receives Telegram image messages| -                   | Clean Input Data    |                                                                                                              |
| Clean Input Data| Set                       | Extracts chat ID and image file ID | Telegram Trigger    | get file           |                                                                                                              |
| get file        | Telegram                  | Downloads image file from Telegram servers | Clean Input Data    | Extract from File   |                                                                                                              |
| Extract from File| Extract From File         | Converts binary image to base64 JSON property | get file            | Gemini OCR          |                                                                                                              |
| Gemini OCR      | HTTP Request              | Sends image to Gemini OCR API and extracts text | Extract from File   | Telegram             | ## HTTP Gemini OCR Setting: Defines Gemini Model URL and auth: Generic Credential Type with Query Auth. JSON body includes base64 image and extraction command. |
| Telegram        | Telegram                  | Sends extracted text back to user | Gemini OCR          | -                  |                                                                                                              |
| Sticky Note1    | Sticky Note               | Decorative/label node            | -                   | -                  | ## Image Reader                                                                                              |
| Sticky Note     | Sticky Note               | Documentation on Gemini OCR HTTP node setup | -                   | -                  | ## HTTP Gemini OCR Setting\n\nDefine Gemini Model on URL, Authentication, and JSON body setup details...     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Set `Updates` to `message` only  
   - Enable `Download` option (for media)  
   - Set credentials with Telegram bot API (e.g., `@RuriImageReader_bot`)  
   - Position it as the first node.

2. **Add Set node named "Clean Input Data"**  
   - Assign two new fields:  
     - `chatID` (number) = expression: `{{ $json.message.chat.id }}`  
     - `Image` (string) = expression: `{{ $json["message"]["photo"][$json["message"]["photo"].length - 1]["file_id"] }}`  
   - Connect output of Telegram Trigger to this node.

3. **Add Telegram node named "get file"**  
   - Resource: `file`  
   - Operation: use default to download a file by ID  
   - Set parameter `fileId` with expression: `{{ $json.Image.replace(/\n/g, '') }}` to ensure no newline characters  
   - Use the same Telegram API credentials as before  
   - Connect output of "Clean Input Data" to this node.

4. **Add Extract From File node named "Extract from File"**  
   - Operation: `binaryToProperty` (convert binary data to JSON property)  
   - Connect output of "get file" to this node.

5. **Add HTTP Request node named "Gemini OCR"**  
   - Set HTTP Method to POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
   - Authentication: Generic Credential Type with Query Auth  
   - Create or use existing credentials from Google AI Studio with valid API key  
   - Under Body Parameters:  
     - Set Body Content Type to JSON  
     - Specify Body as JSON with the following structure, using expression to insert base64 image data:  
       ```json
       {
         "contents": [
           {
             "role": "user",
             "parts": [
               {
                 "inlineData": {
                   "mimeType": "image/jpeg",
                   "data": "{{ $json.data }}"
                 }
               },
               {
                 "text": "Extract text"
               }
             ]
           }
         ]
       }
       ```  
   - Connect output of "Extract from File" to this node.

6. **Add Telegram node named "Telegram" for output**  
   - Operation: Send Message  
   - `text` parameter set with expression: `{{ $json.output }}` (the OCR extracted text from Gemini OCR response)  
   - `chatId` parameter set with expression: `{{ $('Clean Input Data').item.json.chatID }}` (to send message back to the original user)  
   - Disable `appendAttribution`  
   - Use Telegram API credentials as before  
   - Connect output of "Gemini OCR" to this node.

7. **Optional: Add Sticky Notes**  
   - Add descriptive sticky notes for labeling and documenting the workflow and key nodes as in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                            | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| HTTP Gemini OCR Setting details: Use Gemini 2.0 Flash model endpoint with Generic Credential Type and Query Auth from Google AI Studio. JSON body must embed base64 image and extraction instruction text. | See Sticky Note in workflow; also consult https://aistudio.google.com/ for credential setup.       |
| Telegram bot setup requires a valid bot token and proper webhook configuration in n8n to receive updates and download files.                                                                            | Telegram Bot API documentation: https://core.telegram.org/bots/api                                      |
| Gemini OCR API may have quota limits and require billing enabled on Google Cloud Project associated with the API key.                                                                                   | Google Generative Language Models API docs: https://developers.generativeai.google/                  |

---

**Disclaimer:**  
The text provided here is derived exclusively from an automated n8n workflow. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.