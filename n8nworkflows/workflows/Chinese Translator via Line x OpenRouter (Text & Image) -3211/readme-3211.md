Chinese Translator via Line x OpenRouter (Text & Image) 

https://n8nworkflows.xyz/workflows/chinese-translator-via-line-x-openrouter--text---image---3211


# Chinese Translator via Line x OpenRouter (Text & Image) 

### 1. Workflow Overview

This workflow, titled **"Chinese Translator via Line x OpenRouter (Text & Image)"**, is designed to provide real-time Chinese translation services within the LINE messaging platform. It supports both text and image inputs, translating Chinese characters into pinyin and English, leveraging OpenRouter.ai’s Qwen language models. The workflow is ideal for language learners, travelers, educators, businesses, and automation enthusiasts who need instant, accurate translations integrated into LINE chat interactions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming messages from LINE users via a webhook.
- **1.2 User Feedback:** Sends a loading animation to inform users that their request is being processed.
- **1.3 Input Routing:** Determines the type of incoming message (text, image, audio, or unsupported) and routes accordingly.
- **1.4 Content Processing and Translation:**
  - For text: Sends the text to OpenRouter.ai’s Qwen model for translation.
  - For images: Retrieves the image content, converts it to base64, and sends it to a vision-capable Qwen model for translation.
- **1.5 Response Delivery:** Sends the translated output back to the user via LINE’s Reply API.
- **1.6 Unsupported Input Handling:** Replies politely when the message type is not supported.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming POST requests from LINE’s webhook, capturing user messages to trigger the workflow.

- **Nodes Involved:**  
  - Line Webhook

- **Node Details:**  
  - **Line Webhook**  
    - Type: Webhook (HTTP POST listener)  
    - Configuration: Listens on path `/cn` for POST requests from LINE platform.  
    - Key expressions: Extracts message data from `body.events[0]` in the incoming JSON payload.  
    - Input: HTTP POST request from LINE platform.  
    - Output: JSON containing event data including message type, user ID, reply token, and message content.  
    - Edge Cases:  
      - Invalid or missing webhook configuration on LINE Developer Console will prevent triggering.  
      - Malformed payloads or unexpected message structures may cause failures.  
    - Notes: Requires webhook URL to be registered in LINE Developer Console with production URL (remove any "test" suffix).  
    - Sticky Note1 provides setup instructions and link: https://developers.line.biz/en/docs/messaging-api/receiving-messages/

#### 2.2 User Feedback (Loading Animation)

- **Overview:**  
  Sends a loading animation to the user’s LINE chat to indicate the workflow is processing the request, improving user experience.

- **Nodes Involved:**  
  - Line Loading Animation

- **Node Details:**  
  - **Line Loading Animation**  
    - Type: HTTP Request  
    - Configuration: POST to `https://api.line.me/v2/bot/chat/loading/start` with JSON body specifying `chatId` (userId from webhook) and `loadingSeconds` (60 seconds).  
    - Authentication: Uses HTTP header authentication with LINE channel access token.  
    - Input: Triggered immediately after webhook reception.  
    - Output: Passes original message data downstream.  
    - Edge Cases:  
      - Authorization failure if token is invalid or expired.  
      - Network timeout or API rate limits.  
    - Sticky Note2 explains purpose and authorization: https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/

#### 2.3 Input Routing

- **Overview:**  
  Determines the type of incoming message (text, image, audio, or unsupported) and routes the workflow accordingly.

- **Nodes Involved:**  
  - Switch

- **Node Details:**  
  - **Switch**  
    - Type: Switch (conditional routing)  
    - Configuration: Checks `body.events[0].message.type` for values:  
      - "text" → routes to text translation  
      - "image" → routes to image processing  
      - "audio" → routes to unsupported reply  
      - else → routes to unsupported reply  
    - Input: Receives message data from loading animation node.  
    - Output: Routes to four different paths based on message type.  
    - Edge Cases:  
      - Unexpected or new message types not accounted for will be treated as unsupported.  
      - Case sensitivity and strict type validation enforced.  
    - Sticky Note5 describes routing logic.

#### 2.4 Content Processing and Translation

##### 2.4.1 Text Input Translation

- **Overview:**  
  Sends received text messages to OpenRouter.ai’s Qwen model to obtain Chinese characters, pinyin, and English translations.

- **Nodes Involved:**  
  - OpenRouter: qwen/qwen-2.5-72b-instruct:free  
  - Line Reply (Text)

- **Node Details:**  
  - **OpenRouter: qwen/qwen-2.5-72b-instruct:free**  
    - Type: HTTP Request  
    - Configuration: POST to OpenRouter.ai API endpoint `/api/v1/chat/completions`  
    - Payload: System prompt instructs model to provide Chinese characters, pinyin, and English translation for input text. User content is the text from webhook.  
    - Authentication: Uses OpenRouter.ai API key via HTTP header auth.  
    - Input: Text message from Switch node.  
    - Output: JSON response containing translation in `choices[0].message.content`.  
    - Edge Cases:  
      - API key invalid or quota exceeded.  
      - Model response delay or errors.  
      - Input text too long or malformed.  
    - Sticky Note explains OpenRouter.ai usage: https://openrouter.ai/docs/quickstart

  - **Line Reply (Text)**  
    - Type: HTTP Request  
    - Configuration: POST to LINE Reply API endpoint `/v2/bot/message/reply`  
    - Payload: Uses reply token from webhook and sends translated text message.  
    - Text content is sanitized to remove newlines, markdown, and quotes for LINE compatibility.  
    - Authentication: Uses LINE channel access token.  
    - Input: Translation response from OpenRouter node.  
    - Output: Sends reply to user.  
    - Edge Cases:  
      - Invalid reply token if message expired or already replied.  
      - Authorization failures.  
    - Sticky Note3 provides LINE Reply API reference: https://developers.line.biz/en/docs/messaging-api/sending-messages/

##### 2.4.2 Image Input Translation

- **Overview:**  
  Retrieves image content from LINE, converts it to base64, and sends it to an image-capable Qwen model for translation.

- **Nodes Involved:**  
  - Get Image  
  - Extract from File  
  - OpenRouter : qwen/qwen2.5-vl-72b-instruct:free  
  - Line Reply (image)

- **Node Details:**  
  - **Get Image**  
    - Type: HTTP Request  
    - Configuration: GET from LINE API `/v2/bot/message/{messageId}/content` to fetch image binary data.  
    - Authentication: Uses LINE channel access token.  
    - Input: Routed from Switch node on image type.  
    - Output: Binary image data.  
    - Edge Cases:  
      - Authorization failure or invalid message ID.  
      - Network errors or large image sizes.  
    - Sticky Note8 explains pre-processing.

  - **Extract from File**  
    - Type: Extract From File  
    - Configuration: Converts binary image data to base64 string stored in JSON property `img_prompt`.  
    - Input: Binary data from Get Image node.  
    - Output: JSON with base64 encoded image string.  
    - Edge Cases:  
      - Binary conversion errors if data corrupted.

  - **OpenRouter : qwen/qwen2.5-vl-72b-instruct:free**  
    - Type: HTTP Request  
    - Configuration: POST to OpenRouter.ai with system prompt requesting Chinese characters, pinyin, and English translation for text in image.  
    - User content includes an image URL with base64 data embedded.  
    - Authentication: OpenRouter.ai API key.  
    - Input: Base64 image string from Extract from File node.  
    - Output: Translation response JSON.  
    - Edge Cases:  
      - Large base64 payloads may cause timeout or API limits.  
      - Model may fail to extract text if image quality is poor.  
    - Sticky Note7 describes image prompt usage.

  - **Line Reply (image)**  
    - Type: HTTP Request  
    - Configuration: POST to LINE Reply API sending translated text extracted from image.  
    - Text content sanitized similarly to text reply node.  
    - Authentication: LINE channel access token.  
    - Input: Translation response from OpenRouter image model.  
    - Output: Reply message to user.  
    - Edge Cases: Same as text reply.

#### 2.5 Unsupported Input Handling

- **Overview:**  
  For audio or other unsupported message types, sends a polite reply indicating the message type is not supported.

- **Nodes Involved:**  
  - Line Reply (Not Supported 1)  
  - Line Reply (Not Supported 2)

- **Node Details:**  
  - **Line Reply (Not Supported 1)** and **Line Reply (Not Supported 2)**  
    - Type: HTTP Request  
    - Configuration: POST to LINE Reply API with a fixed text message: "Please try again. Message type is not supported"  
    - Authentication: LINE channel access token.  
    - Input: Routed from Switch node for audio or other unsupported types.  
    - Output: Reply message to user.  
    - Edge Cases: Same as other reply nodes.  
    - Sticky Note9 notes this is for unsupported message replies.

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)                     | Sticky Note                                                                                   |
|-------------------------------------|---------------------|----------------------------------------|-------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Line Webhook                        | Webhook             | Receive incoming LINE messages          | —                       | Line Loading Animation            | Setup webhook in LINE Console; copy URL; remove "test" for production; https://developers.line.biz/en/docs/messaging-api/receiving-messages/ |
| Line Loading Animation              | HTTP Request        | Send loading animation to user          | Line Webhook            | Switch                          | Shows loading indicator; requires LINE token; https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/ |
| Switch                            | Switch              | Route input by message type              | Line Loading Animation   | OpenRouter (text), Get Image, Line Reply (Not Supported 1), Line Reply (Not Supported 2) | Routes text, image, audio, else; see Sticky Note5                                            |
| Get Image                         | HTTP Request        | Retrieve image content from LINE         | Switch (image route)     | Extract from File               | Pre-processing: get media API and convert to base64                                         |
| Extract from File                 | Extract From File    | Convert binary image to base64 string    | Get Image                | OpenRouter (image)              | Converts image to base64 for prompt                                                        |
| OpenRouter: qwen/qwen-2.5-72b-instruct:free | HTTP Request        | Translate text input via OpenRouter.ai   | Switch (text route)      | Line Reply (Text)               | Calls Qwen text model; https://openrouter.ai/docs/quickstart                                |
| OpenRouter : qwen/qwen2.5-vl-72b-instruct:free | HTTP Request        | Translate image input via OpenRouter.ai  | Extract from File        | Line Reply (image)              | Calls Qwen vision-language model; supports image prompt                                    |
| Line Reply (Text)                 | HTTP Request        | Send translated text reply to user       | OpenRouter (text)        | —                              | Sends reply via LINE API; https://developers.line.biz/en/docs/messaging-api/sending-messages/ |
| Line Reply (image)                | HTTP Request        | Send translated image reply to user      | OpenRouter (image)       | —                              | Sends reply via LINE API                                                                   |
| Line Reply (Not Supported 1)      | HTTP Request        | Reply for unsupported audio message      | Switch (audio route)     | —                              | Reply "message type not supported"                                                        |
| Line Reply (Not Supported 2)      | HTTP Request        | Reply for unsupported other message types| Switch (else route)      | —                              | Reply "message type not supported"                                                        |
| Sticky Note1                     | Sticky Note         | Webhook setup instructions                | —                       | —                              | See above                                                                                  |
| Sticky Note2                     | Sticky Note         | Loading animation explanation             | —                       | —                              | See above                                                                                  |
| Sticky Note3                     | Sticky Note         | LINE Reply API info                       | —                       | —                              | See above                                                                                  |
| Sticky Note5                     | Sticky Note         | Switch routing explanation                | —                       | —                              | See above                                                                                  |
| Sticky Note7                     | Sticky Note         | OpenRouter image prompt explanation       | —                       | —                              | See above                                                                                  |
| Sticky Note8                     | Sticky Note         | Pre-processing image explanation          | —                       | —                              | See above                                                                                  |
| Sticky Note9                     | Sticky Note         | Unsupported message reply explanation     | —                       | —                              | See above                                                                                  |
| Sticky Note4                     | Sticky Note         | Test info for LINE bot                     | —                       | —                              | "You can test this workflow by adding Line @405jtfqs"                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `cn`  
   - Purpose: Receive incoming LINE messages.  
   - Save and copy the webhook URL.

2. **Configure LINE Developer Console**  
   - Register the webhook URL from step 1.  
   - Enable webhook and remove any "test" suffix for production.

3. **Create HTTP Request Node for Loading Animation**  
   - Name: Line Loading Animation  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/chat/loading/start`  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "chatId": "={{ $json.body.events[0].source.userId }}",
       "loadingSeconds": 60
     }
     ```  
   - Authentication: HTTP Header Auth with LINE channel access token credentials.

4. **Connect Webhook → Loading Animation**

5. **Create Switch Node for Input Routing**  
   - Name: Switch  
   - Condition on `{{$json.body.events[0].message.type}}` with cases:  
     - Equals "text" → Output "text"  
     - Equals "image" → Output "img"  
     - Equals "audio" → Output "audio"  
     - Else → Output "else"

6. **Connect Loading Animation → Switch**

7. **Text Input Path:**  
   - Create HTTP Request Node:  
     - Name: OpenRouter: qwen/qwen-2.5-72b-instruct:free  
     - Method: POST  
     - URL: `https://openrouter.ai/api/v1/chat/completions`  
     - Body Type: JSON  
     - JSON Body:  
       ```json
       {
         "model": "qwen/qwen-2.5-72b-instruct:free",
         "messages": [
           {
             "role": "system",
             "content": "please provide chinese character, pinyin and translation in english. if the input is in english, you will translate and also provide chinese character, pinyin and translation in english for each word"
           },
           {
             "role": "user",
             "content": "={{ $json.body.events[0].message.text }}"
           }
         ]
       }
       ```  
     - Authentication: HTTP Header Auth with OpenRouter.ai API key.

   - Create HTTP Request Node:  
     - Name: Line Reply (Text)  
     - Method: POST  
     - URL: `https://api.line.me/v2/bot/message/reply`  
     - Body Type: JSON  
     - JSON Body:  
       ```json
       {
         "replyToken": "={{ $json.body.events[0].replyToken }}",
         "messages": [
           {
             "type": "text",
             "text": "={{ $json.choices[0].message.content.replaceAll('\\n','\\\\n').replaceAll('\\n','').removeMarkdown().removeTags().replaceAll('\"','') }}"
           }
         ]
       }
       ```  
     - Authentication: HTTP Header Auth with LINE channel access token.

   - Connect Switch "text" output → OpenRouter text node → Line Reply (Text)

8. **Image Input Path:**  
   - Create HTTP Request Node:  
     - Name: Get Image  
     - Method: GET  
     - URL: `https://api-data.line.me/v2/bot/message/{{ $json.body.events[0].message.id }}/content`  
     - Authentication: HTTP Header Auth with LINE channel access token.

   - Create Extract From File Node:  
     - Name: Extract from File  
     - Operation: Binary to Property  
     - Destination Key: `img_prompt`

   - Create HTTP Request Node:  
     - Name: OpenRouter : qwen/qwen2.5-vl-72b-instruct:free  
     - Method: POST  
     - URL: `https://openrouter.ai/api/v1/chat/completions`  
     - Body Type: JSON  
     - JSON Body:  
       ```json
       {
         "model": "qwen/qwen2.5-vl-72b-instruct:free",
         "messages": [
           {
             "role": "system",
             "content": "please provide chinese character, pinyin and translation in english for all the text in the image"
           },
           {
             "role": "user",
             "content": [
               {
                 "type": "image_url",
                 "image_url": {
                   "url": "data:image/jpeg;base64,{{ $json.img_prompt }}"
                 }
               }
             ]
           }
         ]
       }
       ```  
     - Authentication: HTTP Header Auth with OpenRouter.ai API key.

   - Create HTTP Request Node:  
     - Name: Line Reply (image)  
     - Method: POST  
     - URL: `https://api.line.me/v2/bot/message/reply`  
     - Body Type: JSON  
     - JSON Body:  
       ```json
       {
         "replyToken": "={{ $json.body.events[0].replyToken }}",
         "messages": [
           {
             "type": "text",
             "text": "={{ $json.choices[0].message.content.replaceAll('\\n','\\\\n').replaceAll('\\n','').removeMarkdown().removeTags().replaceAll('\"','') }}"
           }
         ]
       }
       ```  
     - Authentication: HTTP Header Auth with LINE channel access token.

   - Connect Switch "img" output → Get Image → Extract from File → OpenRouter image node → Line Reply (image)

9. **Unsupported Input Paths:**  
   - Create two HTTP Request Nodes for unsupported replies:  
     - Name: Line Reply (Not Supported 1)  
     - Name: Line Reply (Not Supported 2)  
     - Both configured as:  
       - Method: POST  
       - URL: `https://api.line.me/v2/bot/message/reply`  
       - Body Type: JSON  
       - JSON Body:  
         ```json
         {
           "replyToken": "={{ $json.body.events[0].replyToken }}",
           "messages": [
             {
               "type": "text",
               "text": "Please try again. Message type is not supported"
             }
           ]
         }
         ```  
       - Authentication: HTTP Header Auth with LINE channel access token.

   - Connect Switch "audio" output → Line Reply (Not Supported 1)  
   - Connect Switch "else" output → Line Reply (Not Supported 2)

10. **Credentials Setup:**  
    - Create HTTP Header Auth credentials for LINE channel access token (used in all LINE API calls).  
    - Create HTTP Header Auth credentials for OpenRouter.ai API key (used in OpenRouter nodes).

11. **Test Workflow:**  
    - Add LINE bot @405jtfqs or your own configured bot.  
    - Send text and image messages to verify translation and replies.  
    - Confirm loading animation appears during processing.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook setup requires registering the URL in LINE Developer Console and removing "test" suffix | https://developers.line.biz/en/docs/messaging-api/receiving-messages/                               |
| Loading animation improves user experience by showing processing status                          | https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/                            |
| OpenRouter.ai allows calling multiple LLMs including Qwen with a unified API                     | https://openrouter.ai/docs/quickstart                                                              |
| LINE Reply API used to send messages back to users                                              | https://developers.line.biz/en/docs/messaging-api/sending-messages/                                 |
| You can test this workflow by adding LINE bot @405jtfqs                                         | Mentioned in Sticky Note4                                                                           |

---

This documentation provides a complete, detailed understanding of the "Chinese Translator via Line x OpenRouter (Text & Image)" workflow, enabling advanced users and AI agents to analyze, reproduce, or customize it effectively.