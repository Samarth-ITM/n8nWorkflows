Telegram Multi-language Translation Bot with GPT-3.5

https://n8nworkflows.xyz/workflows/telegram-multi-language-translation-bot-with-gpt-3-5-10385


# Telegram Multi-language Translation Bot with GPT-3.5

---

## 1. Workflow Overview

This workflow implements a **Telegram Multi-language Translation Bot** leveraging OpenAI's GPT-3.5 to automatically detect the source language of incoming messages, translate them into multiple target languages according to group members' preferences, and relay the translated messages back to each recipient individually. It targets multilingual group chat scenarios where messages need to be understood by participants speaking different languages.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming chat messages via webhook from Telegram or similar platforms.
- **1.2 Data Extraction:** Parses the webhook payload to extract message text, sender ID, group ID, and sender name.
- **1.3 Language Detection:** Uses OpenAI GPT-3.5 to detect the language of the original message.
- **1.4 Group Member Lookup:** Retrieves group members and their preferred languages to determine translation targets.
- **1.5 Translation Processing:** Sends translation requests to OpenAI GPT-3.5 for each target language and waits for completion.
- **1.6 Formatting and Splitting:** Prepares translated content and splits it into individual messages per recipient.
- **1.7 Message Delivery:** Sends translated messages to each recipient through the chat platform API.
- **1.8 Delivery Logging and Response:** Logs delivery status for monitoring and sends a success response to the webhook caller.

Supporting sticky notes document the purpose and context of each block for clarity.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception

- **Overview:**  
  Receives incoming messages from chat platforms via an HTTP webhook endpoint and initiates the workflow.

- **Nodes Involved:**  
  - Webhook - Incoming Message  
  - Sticky Note (📥 WEBHOOK ENTRY)

- **Node Details:**  
  - **Webhook - Incoming Message**  
    - Type: Webhook  
    - Role: Entry point node that listens for POST requests at `/translate-relay`.  
    - Configuration: HTTP POST method, responds using a response node downstream.  
    - Inputs: External HTTP POST calls from chat platforms (Telegram, Slack, Discord, etc.).  
    - Outputs: Passes raw webhook JSON payload.  
    - Edge Cases: Invalid payload formats, no message text, or unsupported platforms may cause empty or failed processing.

  - **Sticky Note:**  
    - Describes incoming data fields expected (message text, sender ID, group ID).

---

### 1.2 Data Extraction

- **Overview:**  
  Parses the webhook payload to extract key data fields: message text, sender ID, group ID, and sender name. Normalizes data for downstream nodes.

- **Nodes Involved:**  
  - Extract Message Data  
  - Sticky Note1 (🔍 DATA EXTRACTION)

- **Node Details:**  
  - **Extract Message Data**  
    - Type: Code  
    - Role: Extracts relevant fields from incoming JSON, handling variant field names for compatibility.  
    - Key Expressions:  
      - `messageText` from `body.message` or `body.text`  
      - `senderId` from `body.sender_id` or `body.user_id`  
      - `groupId` from `body.group_id` or `body.chat_id`  
      - `senderName` with fallback to "User"  
    - Outputs: JSON containing extracted fields with an ISO timestamp.  
    - Edge Cases: Missing fields default to fallback values; absence of message text would halt meaningful processing.

  - **Sticky Note:** Highlights the extraction purpose.

---

### 1.3 Language Detection

- **Overview:**  
  Calls OpenAI GPT-3.5 to detect the ISO 639-1 language code of the original message text.

- **Nodes Involved:**  
  - Detect Language (OpenAI) HTTP Request  
  - Parse Detected Language Code (Code)  
  - Sticky Note2 (🌐 LANGUAGE DETECTION)  
  - Sticky Note3 (📋 LANGUAGE PARSING)

- **Node Details:**  
  - **Detect Language (OpenAI)**  
    - Type: HTTP Request  
    - Role: Sends a chat completion request to OpenAI API with system instructions to detect language and respond with only the two-letter ISO code.  
    - Configuration:  
      - Model: `gpt-3.5-turbo`  
      - Temperature: 0.3 (low randomness)  
      - Max tokens: 10 (response constrained to short code)  
      - Headers: `Content-Type: application/json`  
    - Inputs: Extracted original message text.  
    - Outputs: OpenAI response JSON with language code.  
    - Edge Cases: API errors (auth failure, rate limits), incorrect or unexpected responses if message is empty or ambiguous.

  - **Parse Detected Language**  
    - Type: Code  
    - Role: Extracts and normalizes the language code from OpenAI’s response.  
    - Key Expression: Trims and lowers response content from `choices[0].message.content`.  
    - Outputs: Extends previous JSON with `detectedLanguage`.  
    - Edge Cases: Malformed or missing response content from OpenAI.

  - **Sticky Notes:** Explain detection and parsing purpose.

---

### 1.4 Group Member Lookup

- **Overview:**  
  Retrieves a predefined list of group members with their language preferences, filters out the sender, and determines target languages to translate into.

- **Nodes Involved:**  
  - Get Group Members & Languages (Code)  
  - Sticky Note4 (👥 GROUP MEMBERS)

- **Node Details:**  
  - **Get Group Members & Languages**  
    - Type: Code  
    - Role: Simulates a database of group members per `groupId` with their user IDs, names, and preferred languages.  
    - Logic:  
      - Retrieves all members of the group (hardcoded for `group_123`).  
      - Excludes the sender from recipients.  
      - Extracts unique target languages excluding the sender’s detected language.  
      - Outputs an array of JSON objects, each representing a translation task for a target language with corresponding recipients.  
    - Edge Cases: Group ID absent or unknown leads to empty member list; no recipients means no translations.

  - **Sticky Note:** Describes member fetching and language determination.

---

### 1.5 Translation Processing

- **Overview:**  
  For each target language, sends the original message to OpenAI GPT-3.5 to get a professional translation. Waits for the AI to respond before continuing.

- **Nodes Involved:**  
  - Translate Message (OpenAI) HTTP Request  
  - Wait For Answer (Wait)  
  - Sticky Note5 (🔄 TRANSLATION ENGINE)

- **Node Details:**  
  - **Translate Message (OpenAI)**  
    - Type: HTTP Request  
    - Role: Sends chat completion requests to OpenAI with instructions to translate the original message to the specified target language, providing only the translated text.  
    - Configuration:  
      - Model: `gpt-3.5-turbo`  
      - Temperature: 0.3  
      - Max tokens: 500 (accommodating longer translations)  
      - Headers: `Content-Type: application/json`  
    - Inputs: Translation task JSON including `targetLanguage` and `originalMessage`.  
    - Outputs: OpenAI translation response.  
    - Edge Cases: API errors, partial translations, or empty results.

  - **Wait For Answer**  
    - Type: Wait  
    - Role: Ensures workflow pauses until translation is received before formatting results.  
    - Inputs: Output of translation HTTP request.  
    - Outputs: Passes translated content forward.  
    - Edge Cases: Timeout or interruptions in waiting may cause incomplete processing.

  - **Sticky Note:** Explains translation and response waiting.

---

### 1.6 Formatting and Splitting

- **Overview:**  
  Extracts the translated text, packages it with metadata, then splits the message into individual messages for each recipient in the target language.

- **Nodes Involved:**  
  - Format Translation Result (Code)  
  - Split to Individual Recipients (Code)  
  - Sticky Note6 (📝 FORMAT RESULTS)  
  - Sticky Note7 (📤 RECIPIENT SPLIT)

- **Node Details:**  
  - **Format Translation Result**  
    - Type: Code  
    - Role: Extracts translated text from OpenAI response and combines it with original and recipient metadata for downstream splitting.  
    - Key Variables:  
      - `translatedMessage` from `choices[0].message.content`  
      - Includes `targetLanguage`, `originalMessage`, `senderName`, `recipients`, and `groupId` from earlier nodes.  
    - Outputs: Single JSON with all relevant translation data.  
    - Edge Cases: Missing translation content or recipient info causes incomplete messages.

  - **Split to Individual Recipients**  
    - Type: Code  
    - Role: Iterates over recipients array to create separate message items, each containing recipient-specific info and the translated message prefixed with sender name.  
    - Outputs: Multiple JSON items, one per recipient, ready for delivery.  
    - Edge Cases: Empty recipients array, string concatenation errors.

  - **Sticky Notes:** Describe formatting and splitting roles.

---

### 1.7 Message Delivery

- **Overview:**  
  Sends each translated message to the respective recipient via the chat platform’s API.

- **Nodes Involved:**  
  - Send Translated Message (HTTP Request)  
  - Sticky Note8 (📨 MESSAGE DELIVERY)

- **Node Details:**  
  - **Send Translated Message**  
    - Type: HTTP Request  
    - Role: Posts translated message to chat platform API endpoint for delivery to individual recipients.  
    - Configuration:  
      - URL: `https://api.your-chat-platform.com/send` (placeholder; must be replaced with actual API)  
      - Body parameters: recipient ID, message text, recipient language  
      - Headers: `Content-Type: application/json`  
    - Inputs: Recipient-specific JSON from split node.  
    - Outputs: Response from chat platform API.  
    - Edge Cases: API authentication failures, invalid recipient IDs, message length limits, rate limits.

  - **Sticky Note:** Explains message delivery purpose.

---

### 1.8 Delivery Logging and Response

- **Overview:**  
  Logs the successful delivery status for monitoring and analytics, then sends a confirmation response back to the webhook caller.

- **Nodes Involved:**  
  - Log Delivery Status (Code)  
  - Webhook Response (Respond to Webhook)  
  - Sticky Note9 (📊 DELIVERY LOGGING)  
  - Sticky Note10 (✅ RESPONSE)

- **Node Details:**  
  - **Log Delivery Status**  
    - Type: Code  
    - Role: Creates a structured log entry with timestamp, recipient info, language, delivery status, and message length.  
    - Outputs: JSON log object for potential storage or analytics.  
    - Edge Cases: Missing recipient or translation data may cause incomplete logs.

  - **Webhook Response**  
    - Type: Respond to Webhook  
    - Role: Returns JSON response confirming success and the number of delivery messages processed.  
    - Response Body: `{ "status": "success", "message": "Translation relay completed", "deliveries": <count> }`  
    - Edge Cases: Response failure or premature termination.

  - **Sticky Notes:** Provide context on delivery tracking and final response.

---

### Additional Notes

- **Sticky Note11 (🤖 TRANSLATION RELAY BOT)** summarizes the entire workflow’s features and setup instructions, including prerequisites such as OpenAI API credentials, chat platform API configuration, group member database setup, and webhook testing.

---

## 3. Summary Table

| Node Name                  | Node Type            | Functional Role                               | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                  |
|----------------------------|----------------------|-----------------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------|
| Webhook - Incoming Message | Webhook              | Receives incoming messages                     | (External HTTP POST)           | Extract Message Data           | 📥 WEBHOOK ENTRY: Receives incoming messages with message text, sender ID, group ID |
| Extract Message Data        | Code                 | Extracts key fields from webhook payload       | Webhook - Incoming Message     | Detect Language (OpenAI)       | 🔍 DATA EXTRACTION: Extracts key fields from incoming payload |
| Detect Language (OpenAI)    | HTTP Request         | Detects source language using OpenAI            | Extract Message Data           | Parse Detected Language        | 🌐 LANGUAGE DETECTION: Identifies source language using AI   |
| Parse Detected Language     | Code                 | Parses and stores detected language code        | Detect Language (OpenAI)       | Get Group Members & Languages  | 📋 LANGUAGE PARSING: Stores detected language code           |
| Get Group Members & Languages | Code               | Retrieves group members & their languages       | Parse Detected Language        | Translate Message (OpenAI)     | 👥 GROUP MEMBERS: Fetches recipient list and preferred languages |
| Translate Message (OpenAI)  | HTTP Request         | Translates message to target languages          | Get Group Members & Languages  | Wait For Answer                | 🔄 TRANSLATION ENGINE: Translates message and waits for answer |
| Wait For Answer             | Wait                 | Waits for translation completion                | Translate Message (OpenAI)     | Format Translation Result      | 🔄 TRANSLATION ENGINE (continued)                            |
| Format Translation Result   | Code                 | Formats translation results with metadata       | Wait For Answer                | Split to Individual Recipients | 📝 FORMAT RESULTS: Prepares translated messages with metadata |
| Split to Individual Recipients | Code              | Splits messages for individual recipients       | Format Translation Result      | Send Translated Message        | 📤 RECIPIENT SPLIT: Creates individual delivery items        |
| Send Translated Message     | HTTP Request         | Sends translated messages to recipients          | Split to Individual Recipients| Log Delivery Status            | 📨 MESSAGE DELIVERY: Sends translated messages to recipients |
| Log Delivery Status         | Code                 | Logs delivery info for tracking                   | Send Translated Message        | Webhook Response              | 📊 DELIVERY LOGGING: Tracks successful message deliveries    |
| Webhook Response            | Respond to Webhook   | Sends success response to webhook caller         | Log Delivery Status            | (Workflow End)                | ✅ RESPONSE: Confirms successful translation relay           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook - Incoming Message`  
   - HTTP Method: POST  
   - Path: `translate-relay`  
   - Response Mode: Respond with a downstream node  
   - Purpose: Entry point for incoming chat messages.

2. **Add Code Node to Extract Message Data**  
   - Name: `Extract Message Data`  
   - Type: Code (JavaScript)  
   - Code: Extract `messageText`, `senderId`, `groupId`, `senderName` from webhook payload with fallbacks.  
   - Output: JSON with these fields plus current ISO timestamp.  
   - Connect input from `Webhook - Incoming Message`.

3. **Add HTTP Request Node for Language Detection**  
   - Name: `Detect Language (OpenAI)`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.openai.com/v1/chat/completions`  
   - Headers: `Content-Type: application/json`, Authorization with OpenAI API credential (configured separately)  
   - Body (JSON):  
     ```json
     {
       "model": "gpt-3.5-turbo",
       "messages": [
         {"role": "system", "content": "Detect the language of the following text and respond with only the ISO 639-1 language code (e.g., 'en'). Respond with only the two-letter code."},
         {"role": "user", "content": "{{ $json.originalMessage }}"}
       ],
       "temperature": 0.3,
       "max_tokens": 10
     }
     ```  
   - Connect input from `Extract Message Data`.

4. **Add Code Node to Parse Detected Language**  
   - Name: `Parse Detected Language`  
   - Type: Code  
   - Code: Extract and normalize detected language code from OpenAI response at `choices[0].message.content`.  
   - Include original message data for continuity.  
   - Connect input from `Detect Language (OpenAI)`.

5. **Add Code Node to Lookup Group Members and Languages**  
   - Name: `Get Group Members & Languages`  
   - Type: Code  
   - Logic: Hardcode group members with userId, name, language; filter out sender; determine target languages excluding source language; output multiple items by target language.  
   - Connect input from `Parse Detected Language`.

6. **Add HTTP Request Node to Translate Message**  
   - Name: `Translate Message (OpenAI)`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.openai.com/v1/chat/completions`  
   - Headers: `Content-Type: application/json`, OpenAI API auth  
   - Body (JSON):  
     ```json
     {
       "model": "gpt-3.5-turbo",
       "messages": [
         {"role": "system", "content": "You are a professional translator. Translate the following text to {{ $json.targetLanguage }}. Provide only the translation without any additional text or explanations."},
         {"role": "user", "content": "{{ $json.originalMessage }}"}
       ],
       "temperature": 0.3,
       "max_tokens": 500
     }
     ```  
   - Connect input from `Get Group Members & Languages`.

7. **Add Wait Node**  
   - Name: `Wait For Answer`  
   - Type: Wait  
   - Connect input from `Translate Message (OpenAI)`.

8. **Add Code Node to Format Translation Result**  
   - Name: `Format Translation Result`  
   - Type: Code  
   - Code: Extract translated text from OpenAI response; package with target language, original message, sender name, recipients, group ID.  
   - Connect input from `Wait For Answer`.

9. **Add Code Node to Split Messages per Recipient**  
   - Name: `Split to Individual Recipients`  
   - Type: Code  
   - Code: Iterate over recipients array; create one message JSON per recipient with recipient ID, name, language, and message prefixed with sender name.  
   - Connect input from `Format Translation Result`.

10. **Add HTTP Request Node to Send Translated Message**  
    - Name: `Send Translated Message`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Set to your chat platform’s message sending endpoint (replace placeholder).  
    - Headers: `Content-Type: application/json` plus any required auth.  
    - Body parameters:  
      - `recipient_id` = `{{$json.recipientId}}`  
      - `message` = `{{$json.message}}`  
      - `language` = `{{$json.recipientLanguage}}`  
    - Connect input from `Split to Individual Recipients`.

11. **Add Code Node to Log Delivery Status**  
    - Name: `Log Delivery Status`  
    - Type: Code  
    - Code: Create log object with timestamp, recipient info, language, status 'delivered', message length.  
    - Connect input from `Send Translated Message`.

12. **Add Respond to Webhook Node**  
    - Name: `Webhook Response`  
    - Type: Respond to Webhook  
    - Respond with JSON body:  
      ```json
      {
        "status": "success",
        "message": "Translation relay completed",
        "deliveries": {{$items().length}}
      }
      ```  
    - Connect input from `Log Delivery Status`.

13. **Configure Credentials**  
    - Add OpenAI API credentials with your API key.  
    - Setup authentication for your chat platform API in `Send Translated Message` node.

14. **Test Workflow**  
    - Deploy and test webhook endpoint with sample Telegram messages containing varied languages.  
    - Verify translations and deliveries.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| Workflow features automatic language detection, multi-language translation, individual recipient delivery, and delivery tracking.                             | Workflow description and sticky note11 summarizing capabilities                       |
| Setup requires adding OpenAI API credentials, configuring chat platform API credentials, updating group member data, and testing the webhook endpoint.         | Sticky note11 setup instructions                                                      |
| The translation relies on OpenAI GPT-3.5 chat completions with low temperature to ensure consistent translations and ISO language code detection.              | OpenAI API usage nodes                                                                 |
| The chat platform API endpoint URL in `Send Translated Message` is a placeholder and must be replaced with the actual messaging API of the platform used.      | Node configuration notes                                                              |
| Error handling is minimal; production use should add error catching on API calls and webhook validations to improve robustness and reliability.                 | General recommendation                                                                |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---