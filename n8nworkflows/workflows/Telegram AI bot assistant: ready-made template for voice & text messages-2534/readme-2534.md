Telegram AI bot assistant: ready-made template for voice & text messages

https://n8nworkflows.xyz/workflows/workflow-2534-1747220449864.png


# Telegram AI bot assistant: ready-made template for voice & text messages

### 1. Workflow Overview

This n8n workflow implements a Telegram AI chatbot capable of handling both voice and text messages, enriched with short-term conversational memory. It is designed as a ready-to-use template requiring minimal setup, suitable for personal assistants, customer support bots, or integration hubs.

The workflow is logically divided into these blocks:

- **1.1 Receive and Pre-process Messages**  
  Listens for incoming Telegram events, distinguishes message types (voice, text, or unsupported), and handles initial message preparation.

- **1.2 Audio Transcription**  
  For voice messages, downloads the audio file from Telegram and transcribes it into text using OpenAI Whisper.

- **1.3 Message Combination and Classification**  
  Combines either transcribed voice or direct text messages into a unified format, annotates message metadata such as source type and message type.

- **1.4 AI Processing with Memory**  
  Sends the processed message to an AI agent powered by GPT-4, utilizing a window buffer memory to retain context from the last 10 messages in the same chat session.

- **1.5 Response Delivery and Error Handling**  
  Sends the AI-generated reply back to the user in Telegram with proper HTML formatting, including fallback error messages for unsupported commands or processing failures.

---

### 2. Block-by-Block Analysis

#### 1.1 Receive and Pre-process Messages

- **Overview:**  
  This block listens for all incoming Telegram updates, identifies whether the message is text, voice, or unsupported, and triggers corresponding downstream processing.

- **Nodes Involved:**  
  - Listen for incoming events  
  - Determine content type  
  - Send Typing action  
  - Download voice file  
  - Send error message

- **Node Details:**

  - **Listen for incoming events**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point for all Telegram updates (messages, commands, etc.)  
    - *Config:* Listens to all update types (`updates: ["*"]`) using Telegram Bot API credentials  
    - *Input:* External Telegram webhook events  
    - *Output:* JSON of incoming message update  
    - *Potential Failures:* Webhook misconfiguration, Telegram API outages

  - **Determine content type**  
    - *Type:* Switch  
    - *Role:* Routes flow based on message content type: text query, voice message, or unsupported  
    - *Config:*  
      - Output to "Text" if message.text exists and does not start with '/' (indicating a command)  
      - Output to "Voice" if message.voice exists  
      - Fallback output "extra" for unsupported commands or content  
    - *Input:* Incoming message JSON from Telegram Trigger  
    - *Output:* Routes to either text processing, voice file download, or error message node  
    - *Edge Cases:* Commands starting with '/' are currently unsupported and lead to error node

  - **Send Typing action**  
    - *Type:* Telegram  
    - *Role:* Sends "typing" chat action to user, improving UX by indicating the bot is processing  
    - *Config:* Uses chat ID from incoming message to target user  
    - *Input:* Triggered in parallel with content determination  
    - *Output:* None (fire-and-forget)  
    - *Failures:* Telegram API connectivity, rate limits

  - **Download voice file**  
    - *Type:* Telegram  
    - *Role:* Downloads voice message audio file using Telegram file ID  
    - *Config:* Extracts voice.file_id from message; fetches file resource  
    - *Input:* Triggered only for voice messages  
    - *Output:* Binary audio file data passed downstream  
    - *Failures:* File not found, Telegram API errors, network issues

  - **Send error message**  
    - *Type:* Telegram  
    - *Role:* Sends a friendly error message to user when unsupported commands or message types are received  
    - *Config:* Uses first name from message to personalize response; Markdown formatting; no attribution  
    - *Input:* Triggered on unsupported message types  
    - *Output:* None  
    - *Failures:* Telegram API errors, personalization data missing

---

#### 1.2 Audio Transcription

- **Overview:**  
  Converts downloaded voice audio files into text using OpenAI Whisper model.

- **Nodes Involved:**  
  - Convert audio to text

- **Node Details:**

  - **Convert audio to text**  
    - *Type:* OpenAI node (Audio resource, Transcribe operation)  
    - *Role:* Transcribes voice audio to text using OpenAI Whisper  
    - *Config:* Language autodetection (empty), temperature 0.7  
    - *Credentials:* OpenAI API with access to Whisper and GPT-4  
    - *Input:* Binary audio file from Telegram download  
    - *Output:* Transcribed text in JSON field  
    - *Failures:* API auth errors, audio format issues, transcription latency

---

#### 1.3 Message Combination and Classification

- **Overview:**  
  Normalizes and combines message content into one field with metadata about message type and source.

- **Nodes Involved:**  
  - Combine content and set properties

- **Node Details:**

  - **Combine content and set properties**  
    - *Type:* Set node  
    - *Role:* Creates unified message text field and annotates message type ("text query" or "voice message") and source type (e.g., forwarded)  
    - *Config:*  
      - `CombinedMessage`: uses message.text if present, otherwise uses transcribed text  
      - `Message Type`: "text query" if message.text exists and no transcribed text, else "voice message"  
      - `Source Type`: adds " forwarded" if message was forwarded  
    - *Input:* Either transcribed text or direct text messages  
    - *Output:* JSON with combined message and metadata  
    - *Failures:* Missing expected fields, expression evaluation errors

---

#### 1.4 AI Processing with Memory

- **Overview:**  
  Sends the combined message to an AI agent using GPT-4, leveraging a sliding window memory buffer to maintain conversational context.

- **Nodes Involved:**  
  - Window Buffer Memory  
  - OpenAI Chat Model  
  - AI Agent

- **Node Details:**

  - **Window Buffer Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Maintains last 10 messages per chat session for context retention  
    - *Config:* Session key based on Telegram chat ID; context window length of 10 messages  
    - *Input:* Chat messages from AI Agent node output  
    - *Output:* Provides conversational memory context to AI Agent  
    - *Failures:* Memory overflow, missing session key evaluation

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model (GPT-4)  
    - *Role:* Language model that generates AI responses  
    - *Config:* Model set to GPT-4o variant, temperature 0.7, frequency penalty 0.2 for response diversity  
    - *Input:* Human message from AI Agent node  
    - *Output:* Raw AI-generated text reply  
    - *Failures:* API rate limits, auth errors, model availability

  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Coordinates AI interactions including tool usage and message formatting  
    - *Config:*  
      - Text input: Combined message from previous node  
      - System message: Personalized greeting with user’s first name and current date/time, instructs formatting in Telegram HTML  
      - Human message: Instructions for possible use of tools, expects JSON action code snippet from AI  
    - *Input:* Combined message with memory context, OpenAI Chat Model output, Window Buffer Memory  
    - *Output:* Structured AI response for sending to user  
    - *Failures:* Expression resolution errors, API timeouts, incorrect tool instructions

---

#### 1.5 Response Delivery and Error Handling

- **Overview:**  
  Sends the AI-generated reply back to the Telegram user, handling formatting and potential error corrections.

- **Nodes Involved:**  
  - Send final reply  
  - Correct errors

- **Node Details:**

  - **Send final reply**  
    - *Type:* Telegram  
    - *Role:* Sends final AI response text to user with HTML formatting and appended gratitude message  
    - *Config:*  
      - Text: AI output with HTML escaping for special characters  
      - Chat ID: User's Telegram ID  
      - Parse mode: HTML  
      - Append attribution disabled  
    - *Input:* AI Agent response  
    - *Output:* Message sent confirmation or error  
    - *On Error:* Continues to "Correct errors" node  
    - *Failures:* HTML formatting errors, Telegram API errors, network issues

  - **Correct errors**  
    - *Type:* Telegram  
    - *Role:* Attempts to send AI output again with corrected HTML escaping if first attempt fails  
    - *Config:* Escapes &, <, >, and " characters in AI output before sending  
    - *Input:* Error output from Send final reply  
    - *Output:* Final message delivery confirmation  
    - *Failures:* Same as Send final reply; fallback mechanism

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                         | Input Node(s)                    | Output Node(s)                | Sticky Note                                                    |
|----------------------------|-------------------------------------|---------------------------------------|---------------------------------|------------------------------|---------------------------------------------------------------|
| Listen for incoming events  | Telegram Trigger                    | Entry point for Telegram updates      | -                               | Determine content type, Send Typing action | ## Receive and pre-process messages                            |
| Determine content type      | Switch                             | Routes by message type: text, voice, unsupported | Listen for incoming events       | Combine content..., Download voice file, Send error message |                                                               |
| Send Typing action          | Telegram                           | Sends "typing" indicator to user      | Listen for incoming events       | -                            |                                                               |
| Download voice file         | Telegram                           | Downloads voice audio file             | Determine content type (Voice)   | Convert audio to text         | ## Transcribe audio                                           |
| Convert audio to text       | OpenAI (Audio Transcribe)          | Transcribes voice audio to text       | Download voice file              | Combine content and set properties |                                                               |
| Combine content and set properties | Set                         | Combines text or transcribed message and annotates metadata | Determine content type (Text) and Convert audio to text | AI Agent                     |                                                               |
| Window Buffer Memory        | LangChain Memory Buffer Window     | Maintains conversational context      | AI Agent                        | AI Agent                     | ## 1. Send incoming message to the AI Agent                   |
| OpenAI Chat Model           | LangChain OpenAI Chat Model        | Generates AI responses (GPT-4)         | AI Agent                        | AI Agent                     |                                                               |
| AI Agent                   | LangChain Agent                    | Coordinates AI processing with memory | Combine content and set properties, Window Buffer Memory, OpenAI Chat Model | Send final reply             |                                                               |
| Send final reply            | Telegram                           | Sends AI response to Telegram user    | AI Agent                       | Correct errors (on error)     | ## 2. Deliver agent reply to the user                         |
| Correct errors             | Telegram                           | Resends AI response with corrected HTML escaping | Send final reply (error output) | -                            |                                                               |
| Send error message          | Telegram                           | Sends error for unsupported commands  | Determine content type (extra)  | -                            |                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure credentials with Telegram Bot API  
   - Select updates as `*` (all) to listen for all inbound messages and events  
   - Position: start of workflow  

2. **Add Switch Node "Determine content type"**  
   - Type: Switch  
   - Add rule for "Text" output: message.text is not empty AND does not start with "/"  
   - Add rule for "Voice" output: message.voice exists  
   - Add fallback output "extra" for unsupported commands or message types  

3. **Add Telegram Node "Send Typing action"**  
   - Type: Telegram  
   - Operation: sendChatAction  
   - Chat ID: expression from incoming message's from.id  

4. **Add Telegram Node "Download voice file"**  
   - Type: Telegram  
   - Operation: get file resource  
   - File ID: expression from message.voice.file_id  
   - Connect from "Voice" output of switch  

5. **Add OpenAI Node "Convert audio to text"**  
   - Type: OpenAI (Audio resource, Transcribe operation)  
   - Language: empty (auto)  
   - Temperature: 0.7  
   - Credentials: OpenAI API with Whisper access  
   - Connect input from "Download voice file" output  

6. **Add Set Node "Combine content and set properties"**  
   - Create fields:  
     - CombinedMessage = message.text if exists, else transcribed text  
     - Message Type = "text query" if message.text exists but no transcribed text, else "voice message"  
     - Source Type = " forwarded" if message.forward_origin exists, else empty  
   - Connect input from both "Text" output of switch and from "Convert audio to text"  

7. **Add LangChain Memory Node "Window Buffer Memory"**  
   - Type: Memory Buffer Window  
   - Session Key: expression using chat ID from incoming message (e.g., `chat_with_{{chat.id}}`)  
   - Context Window Length: 10 messages  

8. **Add LangChain OpenAI Chat Model Node "OpenAI Chat Model"**  
   - Model: GPT-4o or GPT-4 variant  
   - Temperature: 0.7  
   - Frequency Penalty: 0.2  
   - Credentials: OpenAI API  

9. **Add LangChain Agent Node "AI Agent"**  
   - Text: expression from CombinedMessage field  
   - System Message: personalized greeting using user's first name and current datetime, instruct to format reply in Telegram HTML  
   - Human Message: instructions for tool usage and formatting (JSON code snippet expected)  
   - Connect inputs from:  
     - Set node for message text  
     - Window Buffer Memory for memory context  
     - OpenAI Chat Model for language model output  

10. **Add Telegram Node "Send final reply"**  
    - Text: AI Agent output, HTML escaped for special characters  
    - Chat ID: from incoming message user ID  
    - Parse Mode: HTML  
    - Append Attribution: false  
    - Configure error handling to continue on error  

11. **Add Telegram Node "Correct errors"**  
    - Text: AI Agent output with additional HTML escaping for &, <, >, and "  
    - Chat ID: from incoming message user ID  
    - Parse Mode: HTML  
    - Append Attribution: false  
    - Connect error output from "Send final reply" to this node  

12. **Add Telegram Node "Send error message"**  
    - Text: Friendly message to user for unsupported commands including user's first name  
    - Chat ID: from incoming message user ID  
    - Parse Mode: Markdown  
    - Append Attribution: false  
    - Connect fallback output "extra" from switch node  

13. **Connect all nodes following the logical flow:**  
    - Telegram Trigger → Determine content type  
    - Determine content type → Text → Combine content and set properties  
    - Determine content type → Voice → Download voice file → Convert audio to text → Combine content and set properties  
    - Determine content type → Extra → Send error message  
    - Combine content and set properties → AI Agent  
    - Window Buffer Memory and OpenAI Chat Model connected as inputs to AI Agent  
    - AI Agent → Send final reply → Correct errors (on error)  

14. **Credential Setup:**  
    - Telegram API credential with bot token for all Telegram nodes  
    - OpenAI API credential with access for GPT-4 and Whisper models  

15. **Set Webhook URL in Telegram Trigger and deploy workflow**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Free template supports short-term memory with 10-message context window by default.                                               | Workflow feature                                                                                                          |
| Easily swap GPT-4 and Whisper for other language and speech-to-text models.                                                       | Extensibility                                                                                                              |
| Extend with LangChain tools like HTTP Request Tool and Workflow Tool for API integration and workflow triggering.                 | https://community.n8n.io/t/review-node-as-tools-is-finally-here/57539                                                      |
| Step-by-step guide for Telegram bots with n8n and OpenAI access available.                                                        | https://blog.n8n.io/create-telegram-bot/#how-to-build-a-telegram-bot-with-n8n                                              |
| Supports Telegram HTML formatting with explicit escaping of &, <, >, and " characters for message safety.                       | Implementation detail                                                                                                      |
| Use of "typing" chat action node improves user experience by indicating bot is processing.                                        | UX best practice                                                                                                           |
| Handles unsupported commands gracefully with personalized error messages.                                                        | User experience                                                                                                           |

---

This detailed documentation enables advanced users and AI agents to understand, reproduce, and customize the Telegram AI multi-format chatbot workflow effectively.