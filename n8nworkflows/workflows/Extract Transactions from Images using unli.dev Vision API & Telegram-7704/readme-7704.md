Extract Transactions from Images using unli.dev Vision API & Telegram

https://n8nworkflows.xyz/workflows/extract-transactions-from-images-using-unli-dev-vision-api---telegram-7704


# Extract Transactions from Images using unli.dev Vision API & Telegram

---

### 1. Workflow Overview

This workflow is designed to extract transaction details from images sent via Telegram by leveraging the unli.dev Vision API. It targets use cases where users submit photos containing transaction information (e.g., receipts, invoices) and receive a formatted Markdown response summarizing the extracted data. The workflow consists of four logical blocks:

- **1.1 Telegram Input Reception:** Captures incoming Telegram messages containing images.
- **1.2 Image Processing Preparation:** Downloads the image, converts it to Base64, and prepares the request parameters.
- **1.3 AI Vision API Call:** Sends the image and prompt to unli.dev Vision API to analyze and extract transaction details.
- **1.4 Response Delivery:** Formats and sends the AI-generated response back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

**Overview:**  
This block listens for new Telegram messages, specifically expecting images, and triggers the workflow upon receiving such messages.

**Nodes Involved:**  
- Telegram Trigger

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Trigger node for Telegram messages  
  - *Configuration:* Listens for "message" updates on a Telegram bot identified by the "Khaisa Dev Bot" credentials.  
  - *Expressions:* None directly; outputs the full message JSON, including image metadata and chat information.  
  - *Input/Output:* No input; outputs message JSON. Connected to "📥 Download Image".  
  - *Edge cases:* Messages without images will not trigger the subsequent nodes properly; only messages containing photos (array of images) are expected. Potential issues if image has fewer than 4 sizes (uses the 4th size).  
  - *Credentials:* Telegram API with OAuth2 credentials for the bot.  

---

#### 1.2 Image Processing Preparation

**Overview:**  
This block downloads the image from Telegram, converts it into a Base64 string, and sets up the parameters needed for the vision API call.

**Nodes Involved:**  
- 📥 Download Image  
- Convert to Base  
- Set Request  

**Node Details:**

- **📥 Download Image**  
  - *Type:* Telegram node to download files  
  - *Configuration:* Downloads the image file using the file_id from the 4th photo size in the Telegram message array.  
  - *Input:* Output from "Telegram Trigger".  
  - *Output:* Binary data representing the image file.  
  - *Edge cases:* If the photo array has fewer than 4 entries, the file_id reference will fail. Also, if Telegram API limits or network errors occur, download may fail.  
  - *Credentials:* Uses same Telegram bot credentials.  

- **Convert to Base**  
  - *Type:* Code node (JavaScript)  
  - *Configuration:* Converts the binary image data to a Base64-encoded string. Extracts caption from Telegram message (or defaults to "What is in this image?") to use as the prompt. Also extracts chat ID and message ID for later use.  
  - *Key expressions:* Accesses binary data buffer; extracts chat ID and caption via expression referencing "Telegram Trigger" node outputs.  
  - *Input:* Receives binary image from "📥 Download Image".  
  - *Output:* JSON with `base64Image`, `prompt`, `chatId`, and `messageId` fields.  
  - *Edge cases:* If binary data is missing or corrupted, conversion fails. Caption may be undefined.  
  - *Version:* Uses n8n Code node version 2 (supporting latest JavaScript features).  

- **Set Request**  
  - *Type:* Set node  
  - *Configuration:* Sets three fields:  
    - `model`: fixed string "auto" to let the API choose model automatically.  
    - `base64Image`: from previous node JSON.  
    - `userPrompt`: fixed string "extract the trancastion on this image. output in md format" (note typo in "transaction").  
  - *Input:* JSON from "Convert to Base".  
  - *Output:* Structured JSON for the API request.  
  - *Edge cases:* Typo in prompt may affect AI understanding; hardcoded prompt limits flexibility.  

---

#### 1.3 AI Vision API Call

**Overview:**  
This block sends the prepared data to the unli.dev Vision API to analyze the image and extract transaction details in Markdown format.

**Nodes Involved:**  
- Call Vision API

**Node Details:**

- **Call Vision API**  
  - *Type:* HTTP Request node  
  - *Configuration:*  
    - POST request to `https://api.unli.dev/v1/chat/completions`  
    - Timeout set to 120 seconds to accommodate processing delay.  
    - Request body is a JSON object containing:  
      - `model`: from input JSON ("auto")  
      - `messages`: an array with one object containing two parts:  
        - An image object embedding the Base64 image as a data URI  
        - A text object with the user prompt  
    - Headers include `Content-Type: application/json`.  
    - Authentication uses HTTP Header Auth with credentials named "Unli.dev - Jon".  
  - *Input:* JSON from "Set Request".  
  - *Output:* JSON response from unli.dev containing AI analysis, tokens usage, and model info.  
  - *Edge cases:*  
    - API key invalidity or quota exceeded leads to auth errors.  
    - Timeout if processing is too slow.  
    - Malformed base64 or payload may cause rejection.  
  - *Version:* HTTP Request node v4.2, supports advanced JSON body and auth options.  

---

#### 1.4 Response Delivery

**Overview:**  
This final block formats the AI response and sends it back to the Telegram user who submitted the image.

**Nodes Involved:**  
- 📤 Send Response

**Node Details:**

- **📤 Send Response**  
  - *Type:* Telegram node (send message)  
  - *Configuration:*  
    - Sends a Markdown formatted message including:  
      - The content from `choices[0].message.content` (AI generated text).  
      - Model name used.  
      - Token usage statistics (prompt and completion tokens).  
    - Sends to the chat ID extracted from the original Telegram trigger message.  
    - Markdown parse mode enabled for rich formatting.  
  - *Input:* AI response JSON from "Call Vision API".  
  - *Output:* Message sent confirmation.  
  - *Edge cases:* If chat ID is missing or incorrect, message sending fails. If AI response is empty or malformed, output message may be unclear.  
  - *Credentials:* Telegram API with OAuth2 credentials for the bot.  

---

### 3. Summary Table

| Node Name          | Node Type               | Functional Role            | Input Node(s)       | Output Node(s)     | Sticky Note                                 |
|--------------------|-------------------------|----------------------------|---------------------|--------------------|---------------------------------------------|
| Telegram Trigger    | Telegram Trigger         | Receive Telegram messages   | -                   | 📥 Download Image  |                                            |
| 📥 Download Image   | Telegram (file download) | Download image from Telegram| Telegram Trigger     | Convert to Base    |                                            |
| Convert to Base     | Code                    | Convert image to Base64     | 📥 Download Image    | Set Request        |                                            |
| Set Request        | Set                     | Prepare request parameters  | Convert to Base      | Call Vision API    |                                            |
| Call Vision API    | HTTP Request            | Call unli.dev Vision API    | Set Request         | 📤 Send Response   |                                            |
| 📤 Send Response   | Telegram (send message)  | Send AI response to user    | Call Vision API     | -                  |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure credentials for your Telegram bot (OAuth2).  
   - Set to listen for "message" updates.  
   - Save.

2. **Create 📥 Download Image Node:**  
   - Type: Telegram (Download File) node  
   - Connect input from Telegram Trigger.  
   - Set `fileId` to expression: `{{$json.message.photo[3].file_id}}` (4th photo size).  
   - Use same Telegram bot credentials.  
   - Save.

3. **Create Convert to Base Node:**  
   - Type: Code node (JavaScript)  
   - Connect input from 📥 Download Image.  
   - Use the following JavaScript snippet:

   ```javascript
   const binaryData = $input.first().binary.data;
   const base64Image = binaryData.data.toString('base64');
   const caption = $('Telegram Trigger').item.json.message.caption || 'What is in this image?';

   return [{ 
     json: { 
       base64Image: base64Image,
       prompt: caption,
       chatId: $('Telegram Trigger').item.json.message.chat.id,
       messageId: $('Telegram Trigger').item.json.message.message_id
     } 
   }];
   ```

   - Save.

4. **Create Set Request Node:**  
   - Type: Set node  
   - Connect input from Convert to Base.  
   - Add fields:  
     - `model` = "auto" (string)  
     - `base64Image` = Expression: `{{$json.base64Image}}`  
     - `userPrompt` = "extract the trancastion on this image. output in md format" (string)  
   - Save.

5. **Create Call Vision API Node:**  
   - Type: HTTP Request  
   - Connect input from Set Request.  
   - Set method to POST.  
   - URL: `https://api.unli.dev/v1/chat/completions`  
   - Set timeout to 120000 ms.  
   - Set body type to JSON.  
   - Use the following JSON body template (with expressions):

   ```json
   {
     "model": "{{ $json.model }}",
     "messages": [
       {
         "role": "user",
         "content": [
           {
             "type": "image_url",
             "image_url": {
               "url": "data:image/jpeg;base64,{{ $json.base64Image }}"
             }
           },
           {
             "type": "text",
             "text": "{{ $json.userPrompt }}"
           }
         ]
       }
     ]
   }
   ```

   - Add HTTP header: `Content-Type: application/json`  
   - Authentication: HTTP Header Auth with your unli.dev API key credentials ("Unli.dev - Jon").  
   - Save.

6. **Create 📤 Send Response Node:**  
   - Type: Telegram (Send Message)  
   - Connect input from Call Vision API.  
   - Configure message text with expression:

   ```
   📸 **Image Analysis Result**

   {{ $json.choices[0].message.content }}

   Model: {{ $json.model }}
   prompt tokens: {{ $json.usage.prompt_tokens }}
   completion tokens: {{ $json.usage.completion_tokens }}
   ```

   - Set chatId to expression: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Enable Markdown parse mode.  
   - Use Telegram bot credentials.  
   - Save.

7. **Connect all nodes in sequence:**  
   Telegram Trigger → 📥 Download Image → Convert to Base → Set Request → Call Vision API → 📤 Send Response

8. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The prompt contains a typo: "trancastion" instead of "transaction". Correcting may improve results. | Prompt in "Set Request" node.                                                                   |
| Telegram photo array indexing assumes at least 4 sizes; could cause errors if fewer sizes sent.    | Photo size used is the 4th element `[3]` in the photo array from Telegram messages.             |
| unli.dev Vision API expects Base64 image embedded as a Data URI in JSON request.                   | API documentation: https://unli.dev                                                              |
| Telegram bot credentials must be OAuth2 authorized and correctly configured for both Trigger and send nodes. | Ensure bot token permissions include reading messages and sending messages.                      |
| Timeout set to 120 seconds to accommodate long processing times of AI Vision API.                  | HTTP Request node configuration.                                                                |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, respecting content policies and containing no illegal or protected data. All processed data is legal and public.

---