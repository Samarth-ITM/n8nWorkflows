Chinese Translator via Line x OpenRouter

https://n8nworkflows.xyz/workflows/chinese-translator-via-line-x-openrouter-3207


# Chinese Translator via Line x OpenRouter

### 1. Workflow Overview

This workflow, titled **Chinese Translator via Line x OpenRouter**, automates the translation of user-inputted text into Chinese characters, pinyin, and English translations through integration with the Line Messaging API and OpenRouter.ai’s advanced language models (notably the Qwen model). It is designed to provide real-time, comprehensive translations while maintaining an engaging user experience by sending loading animations during processing.

**Target Use Cases:**
- Language learners seeking instant translations and pronunciation guides.
- Travelers needing quick communication assistance in Chinese.
- Businesses providing multilingual support via Line messaging.

**Logical Blocks:**

- **1.1 Input Reception:** Captures incoming messages from Line users via webhook.
- **1.2 Loading Animation:** Sends a loading animation to inform users that translation is in progress.
- **1.3 Translation Processing:** Calls OpenRouter.ai’s API with the user’s text to obtain Chinese characters, pinyin, and English translations.
- **1.4 Sending Response:** Formats and sends the translated text back to the user through Line’s Reply API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming messages from Line users. It extracts the text content and reply token necessary for subsequent processing and replying.

- **Nodes Involved:**  
  - Line Webhook

- **Node Details:**

  - **Line Webhook**  
    - *Type & Role:* Webhook node; entry point for receiving POST requests from Line Messaging API.  
    - *Configuration:*  
      - HTTP Method: POST  
      - Path: `cn` (endpoint path for webhook)  
      - No additional options configured.  
    - *Key Expressions:*  
      - Extracts user message text from `body.events[0].message.text`  
      - Extracts reply token from `body.events[0].replyToken`  
      - Extracts userId from `body.events[0].source.userId` (used later)  
    - *Input/Output:*  
      - Input: HTTP POST request from Line platform.  
      - Output: JSON payload containing event data forwarded downstream.  
    - *Edge Cases / Potential Failures:*  
      - Missing or malformed webhook payloads.  
      - Incorrect webhook URL setup in Line Developer Console.  
      - HTTP method mismatch or unauthorized requests.  
    - *Notes:*  
      - Requires webhook URL registration in Line Developer Console.  
      - Must remove any test suffix from URL before production use.  
      - See [Line Receiving Messages Docs](https://developers.line.biz/en/docs/messaging-api/receiving-messages/).

#### 1.2 Loading Animation

- **Overview:**  
  Sends a loading animation message back to the user to indicate the workflow is processing their request, improving user experience by providing immediate feedback.

- **Nodes Involved:**  
  - Line Loading Animation

- **Node Details:**

  - **Line Loading Animation**  
    - *Type & Role:* HTTP Request node; sends a POST request to Line’s loading indicator API.  
    - *Configuration:*  
      - URL: `https://api.line.me/v2/bot/chat/loading/start`  
      - Method: POST  
      - JSON Body:  
        ```json
        {
          "chatId": "{{ $json.body.events[0].source.userId }}",
          "loadingSeconds": 60
        }
        ```  
      - Authentication: HTTP Header Auth with Line channel access token.  
    - *Key Expressions:*  
      - Uses userId from webhook payload to target the correct chat.  
    - *Input/Output:*  
      - Input: JSON from Line Webhook node.  
      - Output: Response from Line API (not used downstream).  
    - *Edge Cases / Potential Failures:*  
      - Invalid or expired Line channel access token.  
      - Incorrect userId extraction leading to failed API call.  
      - API rate limits or network timeouts.  
    - *Notes:*  
      - This node’s purpose is purely UX; failure here does not block translation.  
      - See [Line Loading Indicator Docs](https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/).

#### 1.3 Translation Processing

- **Overview:**  
  Sends the user’s input text to OpenRouter.ai’s API using the Qwen language model to obtain a detailed translation including Chinese characters, pinyin, and English meanings.

- **Nodes Involved:**  
  - Use OpenRouter

- **Node Details:**

  - **Use OpenRouter**  
    - *Type & Role:* HTTP Request node; calls OpenRouter.ai’s chat completions API.  
    - *Configuration:*  
      - URL: `https://openrouter.ai/api/v1/chat/completions`  
      - Method: POST  
      - JSON Body (constructed dynamically):  
        ```json
        {
          "model": "qwen/qwen-max",
          "messages": [
            {
              "role": "system",
              "content": "please provide chinese character, pinyin and translation in english. if the input is in english, you will translate and also provide chinese character, pinyin and translation in english for each word"
            },
            {
              "role": "user",
              "content": "{{ $('Line Webhook').item.json.body.events[0].message.text }}"
            }
          ]
        }
        ```  
      - Authentication: HTTP Header Auth with OpenRouter.ai API key.  
    - *Key Expressions:*  
      - Injects user message text dynamically from Line Webhook node.  
    - *Input/Output:*  
      - Input: JSON from Line Loading Animation node (which receives from Line Webhook).  
      - Output: JSON response containing translation results in `choices[0].message.content`.  
    - *Edge Cases / Potential Failures:*  
      - API key invalid or quota exceeded.  
      - Network timeouts or API errors.  
      - Unexpected response format or missing fields.  
      - Input text too long or unsupported characters.  
    - *Notes:*  
      - Supports multiple LLMs via OpenRouter.ai but configured here specifically for Qwen.  
      - See [OpenRouter Quickstart](https://openrouter.ai/docs/quickstart).

#### 1.4 Sending Response

- **Overview:**  
  Formats the translation output into a plain text message and sends it back to the user via Line’s Reply API.

- **Nodes Involved:**  
  - Line Reply

- **Node Details:**

  - **Line Reply**  
    - *Type & Role:* HTTP Request node; sends reply messages to Line users.  
    - *Configuration:*  
      - URL: `https://api.line.me/v2/bot/message/reply`  
      - Method: POST  
      - JSON Body (dynamically constructed):  
        ```json
        {
          "replyToken": "{{ $('Line Webhook').item.json.body.events[0].replyToken }}",
          "messages": [
            {
              "type": "text",
              "text": "{{ $json.choices[0].message.content.replaceAll(\"\\n\",\"\\\\n\").replaceAll(\"\\n\",\"\").removeMarkdown().removeTags().replaceAll('\"',\"\") }}"
            }
          ]
        }
        ```  
      - Authentication: HTTP Header Auth with Line channel access token.  
    - *Key Expressions:*  
      - Uses replyToken from webhook to target correct conversation.  
      - Cleans translation text by removing newlines, markdown, HTML tags, and quotes to ensure message formatting compatibility.  
    - *Input/Output:*  
      - Input: JSON from Use OpenRouter node containing translation.  
      - Output: Response from Line API (not used downstream).  
    - *Edge Cases / Potential Failures:*  
      - Invalid or expired Line channel token.  
      - Missing or expired replyToken (reply tokens expire quickly).  
      - Message size limits exceeded.  
      - Formatting errors causing message rejection.  
    - *Notes:*  
      - Critical for delivering final user-visible output.  
      - See [Line Sending Messages Docs](https://developers.line.biz/en/docs/messaging-api/sending-messages/).

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                 | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                      |
|-----------------------|---------------------|--------------------------------|---------------------|------------------------|-------------------------------------------------------------------------------------------------|
| Line Webhook          | Webhook             | Receive incoming Line messages | (HTTP POST request) | Line Loading Animation  | You need to set-up this webhook at Line Manager or Line Developer Console. See link: https://developers.line.biz/en/docs/messaging-api/receiving-messages/ |
| Line Loading Animation | HTTP Request        | Send loading animation to user | Line Webhook        | Use OpenRouter         | This node sends loading animation to inform user the workflow is running. See link: https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/ |
| Use OpenRouter         | HTTP Request        | Call OpenRouter.ai for translation | Line Loading Animation | Line Reply            | Calls LLMs at OpenRouter.ai (Qwen model here). See link: https://openrouter.ai/docs/quickstart   |
| Line Reply             | HTTP Request        | Send translated reply to user  | Use OpenRouter      | (End)                  | Sends reply message via Line API. See link: https://developers.line.biz/en/docs/messaging-api/sending-messages/ |
| Sticky Note1           | Sticky Note         | Instruction for webhook setup  |                     |                        | You need to set-up this webhook at Line Manager or Line Developer Console. See link above.      |
| Sticky Note2           | Sticky Note         | Explanation of loading animation node |                 |                        | This node is to only give loading animation back in Line. See link above.                        |
| Sticky Note            | Sticky Note         | Explanation of OpenRouter node |                     |                        | This is to call LLMs model at Openrouter.ai. See link above.                                    |
| Sticky Note3           | Sticky Note         | Explanation of Line Reply node |                     |                        | This node is to send reply message via Line. See link above.                                   |
| Sticky Note4           | Sticky Note         | Testing instruction            |                     |                        | You can test this workflow by adding Line @405jtfqs                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Line Webhook Node**  
   - Node Type: Webhook  
   - HTTP Method: POST  
   - Path: `cn`  
   - Purpose: Receive incoming messages from Line users.  
   - No authentication needed here.  
   - Save and copy the generated webhook URL.

2. **Register Webhook URL in Line Developer Console**  
   - Go to your Line channel settings.  
   - Paste the webhook URL from step 1, removing any test suffix.  
   - Enable webhook and verify connection.

3. **Create the Line Loading Animation Node**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/chat/loading/start`  
   - Authentication: HTTP Header Auth with your Line channel access token.  
   - JSON Body:  
     ```json
     {
       "chatId": "{{ $json.body.events[0].source.userId }}",
       "loadingSeconds": 60
     }
     ```  
   - Connect input from Line Webhook node.

4. **Create the Use OpenRouter Node**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Authentication: HTTP Header Auth with OpenRouter.ai API key.  
   - JSON Body (raw JSON mode):  
     ```json
     {
       "model": "qwen/qwen-max",
       "messages": [
         {
           "role": "system",
           "content": "please provide chinese character, pinyin and translation in english. if the input is in english, you will translate and also provide chinese character, pinyin and translation in english for each word"
         },
         {
           "role": "user",
           "content": "{{ $('Line Webhook').item.json.body.events[0].message.text }}"
         }
       ]
     }
     ```  
   - Connect input from Line Loading Animation node.

5. **Create the Line Reply Node**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/message/reply`  
   - Authentication: HTTP Header Auth with your Line channel access token.  
   - JSON Body (raw JSON mode):  
     ```json
     {
       "replyToken": "{{ $('Line Webhook').item.json.body.events[0].replyToken }}",
       "messages": [
         {
           "type": "text",
           "text": "{{ $json.choices[0].message.content.replaceAll(\"\\n\",\"\\\\n\").replaceAll(\"\\n\",\"\").removeMarkdown().removeTags().replaceAll('\"',\"\") }}"
         }
       ]
     }
     ```  
   - Connect input from Use OpenRouter node.

6. **Set Workflow Settings**  
   - Timezone: Asia/Bangkok  
   - Execution Order: Sequential (default)  
   - Error Workflow: Optional, configure if desired.

7. **Credentials Setup**  
   - Create HTTP Header Auth credentials for Line channel with your channel access token.  
   - Create HTTP Header Auth credentials for OpenRouter.ai with your API key.

8. **Test the Workflow**  
   - Add the Line official account `@405jtfqs` or your own Line channel.  
   - Send a message to the Line bot.  
   - Observe loading animation, then receive translated text with Chinese characters, pinyin, and English.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| You need to set-up this webhook at Line Manager or Line Developer Console. Copy webhook URL from node. | https://developers.line.biz/en/docs/messaging-api/receiving-messages/                              |
| Loading animation node improves user experience by showing processing status.                           | https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/                           |
| OpenRouter.ai allows calling multiple LLMs with a standardized API and single top-up.                   | https://openrouter.ai/docs/quickstart                                                             |
| Line Reply node sends messages back to users via Line Messaging API.                                    | https://developers.line.biz/en/docs/messaging-api/sending-messages/                               |
| You can test this workflow by adding Line @405jtfqs                                                    |                                                                                                   |

---

This documentation fully describes the **Chinese Translator via Line x OpenRouter** workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents.