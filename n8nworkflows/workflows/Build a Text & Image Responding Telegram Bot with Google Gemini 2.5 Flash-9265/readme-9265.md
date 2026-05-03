Build a Text & Image Responding Telegram Bot with Google Gemini 2.5 Flash

https://n8nworkflows.xyz/workflows/build-a-text---image-responding-telegram-bot-with-google-gemini-2-5-flash-9265


# Build a Text & Image Responding Telegram Bot with Google Gemini 2.5 Flash

### 1. Workflow Overview

This workflow implements an intelligent Telegram bot that interacts with users by responding to both text and image messages using Google Gemini 2.5 Flash AI capabilities. It is designed to:

- Receive user inputs via Telegram (text or images).
- For text messages, process queries with AI considering conversation context.
- For images, download the image, analyze it with Google Gemini Vision, generate descriptive text, and then process with AI.
- Maintain conversation memory for context-aware and coherent responses.
- Format responses attractively for Telegram chat.

The workflow is logically divided into these blocks:

- **1.1 Trigger Reception:** Receives Telegram messages via webhook.
- **1.2 Message Routing:** Determines if the message is text or image, routing accordingly.
- **1.3 Image Processing:** Downloads and analyzes images, then prepares descriptive text.
- **1.4 Text Processing:** Prepares text prompts for AI.
- **1.5 AI Agent Processing:** Uses Google Gemini for contextual conversation with memory.
- **1.6 Response Sending:** Sends the AI-generated response back to the user in Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Reception

- **Overview:**  
Starts the workflow when a user sends a message to the Telegram bot, capturing both text and images.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Sticky Note1 (description)

- **Node Details:**

  - **Telegram Trigger**  
    - *Type & Role:* Telegram Trigger node; listens for incoming Telegram messages.  
    - *Configuration:* Listening for "message" updates only. No additional filters. Webhook ID set.  
    - *Expressions:* None used.  
    - *Connections:* Outputs to "Route Types".  
    - *Failure Modes:* Telegram API downtime, invalid webhook setup, or expired token.  
    - *Version:* v1.2.

  - **Sticky Note1**  
    - Informational node describing the trigger block.

#### 1.2 Message Routing

- **Overview:**  
Determines the type of incoming message (text or image) and routes it to appropriate handling nodes.

- **Nodes Involved:**  
  - Route Types (Switch node)  
  - Sticky Note2

- **Node Details:**

  - **Route Types**  
    - *Type & Role:* Switch node to differentiate message types.  
    - *Configuration:*  
      - Checks if `$json.message.text` exists → route to "Text".  
      - Checks if `$json.message.photo[2]` exists → route to "Image".  
    - *Expressions:* Uses strict existence checks with expressions evaluated from incoming JSON.  
    - *Connections:*  
      - Text output → "Map text prompt".  
      - Image output → "Get a file".  
    - *Failure Modes:* Unexpected message types (e.g., stickers, voice notes) are not handled and drop out.  
    - *Version:* v3.2.

  - **Sticky Note2**  
    - Describes routing logic.

#### 1.3 Image Processing

- **Overview:**  
Downloads the image from Telegram, analyzes it with Google Gemini Vision, and then creates a descriptive prompt combining analysis and the original caption.

- **Nodes Involved:**  
  - Get a file  
  - Analyze image  
  - Map image prompt  
  - Sticky Note3

- **Node Details:**

  - **Get a file**  
    - *Type & Role:* Telegram node to download the image file.  
    - *Configuration:* Uses `file_id` extracted from `$json.message.photo[2].file_id` to fetch the image binary.  
    - *Credentials:* Telegram OAuth2 credentials configured.  
    - *Connections:* Outputs binary data to "Analyze image".  
    - *Failure Modes:* File not found, API errors, invalid `file_id`.  
    - *Version:* v1.2.

  - **Analyze image**  
    - *Type & Role:* Google Gemini Vision node for image analysis.  
    - *Configuration:*  
      - Model: "models/gemini-2.5-flash".  
      - Operation: Analyze binary image input.  
    - *Credentials:* Google Gemini (PaLM) API credentials.  
    - *Connections:* Outputs analysis results to "Map image prompt".  
    - *Failure Modes:* API key issues, quota exceeded, image format errors.  
    - *Version:* v1.

  - **Map image prompt**  
    - *Type & Role:* Set node to create formatted text prompt combining Gemini's description and Telegram's image caption.  
    - *Configuration:*  
      - Constructs `text` field:  
        ```
        User image description:

        {{ $json.content.parts[0].text }}

        User image caption: {{ $('Telegram Trigger').item.json.message.caption }}
        ```  
    - *Connections:* Outputs to "Knowledge Base Agent".  
    - *Failure Modes:* Missing caption or analysis results could lead to incomplete prompts.  
    - *Version:* v3.4.

  - **Sticky Note3**  
    - Describes image processing steps.

#### 1.4 Text Processing

- **Overview:**  
Prepares the raw text message into a prompt format suitable for AI processing.

- **Nodes Involved:**  
  - Map text prompt  
  - Sticky Note2 (routing covers this)

- **Node Details:**

  - **Map text prompt**  
    - *Type & Role:* Set node to extract the user's text message for AI input.  
    - *Configuration:*  
      - Sets `text` to the body of the first message: `={{ $json.messages[0].text.body }}`.  
    - *Connections:* Outputs to "Knowledge Base Agent".  
    - *Failure Modes:* If message structure changes or missing `text.body`, prompt fails.  
    - *Version:* v3.4.

#### 1.5 AI Agent Processing

- **Overview:**  
Processes the prepared prompt (text or image description) with Google Gemini Chat Model, maintaining conversation memory for context-aware replies and formatting responses for Telegram.

- **Nodes Involved:**  
  - Knowledge Base Agent  
  - Simple Memory  
  - Google Gemini Chat Model  
  - Sticky Note4

- **Node Details:**

  - **Simple Memory**  
    - *Type & Role:* Langchain memory buffer storing last 20 messages per session keyed by Telegram message ID.  
    - *Configuration:*  
      - Session key: `memory_{{ $('Telegram Trigger').item.json.message.message_id }}`  
      - Context window length: 20 messages.  
    - *Connections:* Memory input to "Knowledge Base Agent".  
    - *Failure Modes:* Memory overflow or key misalignment could cause loss of conversation context.  
    - *Version:* v1.3.

  - **Knowledge Base Agent**  
    - *Type & Role:* Langchain Agent node that formulates responses based on input prompt and memory.  
    - *Configuration:*  
      - Uses `text` from previous Set nodes as input.  
      - System message is empty (default).  
      - Output formatting described as well spaced and attractive for Telegram.  
    - *Connections:*  
      - Main output to "Send a text message".  
      - Memory input from "Simple Memory".  
      - AI Language Model input from "Google Gemini Chat Model".  
    - *Failure Modes:* AI response generation failure, formatting errors.  
    - *Version:* v1.9.

  - **Google Gemini Chat Model**  
    - *Type & Role:* Langchain Google Gemini Chat Model node for AI language processing.  
    - *Configuration:* Default options.  
    - *Credentials:* Google Gemini (PaLM) API credentials.  
    - *Connections:* Outputs to "Knowledge Base Agent" AI Language Model input.  
    - *Failure Modes:* API limits, authentication errors, latency.  
    - *Version:* v1.

  - **Sticky Note4**  
    - Describes AI agent capabilities.

#### 1.6 Response Sending

- **Overview:**  
Sends the AI-generated response back to the user on Telegram.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**

  - **Send a text message**  
    - *Type & Role:* Telegram node to send a text message.  
    - *Configuration:*  
      - Sends text from `{{$json.output}}` (AI response).  
      - Chat ID extracted from original Telegram message: `{{$('Telegram Trigger').item.json.message.chat.id}}`.  
      - Disables attribution append to message.  
    - *Credentials:* Telegram OAuth2 credentials.  
    - *Connections:* Final node, no outputs.  
    - *Failure Modes:* Chat ID missing, Telegram API errors, message too long.  
    - *Version:* v1.2.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                   | Input Node(s)           | Output Node(s)         | Sticky Note                          |
|-------------------------|----------------------------------|---------------------------------|------------------------|------------------------|------------------------------------|
| Sticky Note             | Sticky Note                      | Workflow overview and setup guide| -                      | -                      | 📋 TELEGRAM AI BOT - SETUP GUIDE   |
| Sticky Note1            | Sticky Note                      | Describes Trigger block          | -                      | -                      | 🚀 TRIGGER                        |
| Sticky Note2            | Sticky Note                      | Describes Routing block          | -                      | -                      | 🔀 ROUTING                       |
| Sticky Note3            | Sticky Note                      | Describes Image Processing       | -                      | -                      | 🖼️ IMAGE PROCESSING              |
| Sticky Note4            | Sticky Note                      | Describes AI Agent               | -                      | -                      | 🤖 AI AGENT                     |
| Telegram Trigger        | Telegram Trigger                 | Receives Telegram messages       | -                      | Route Types             | 🚀 TRIGGER                        |
| Route Types             | Switch                          | Routes messages by type          | Telegram Trigger        | Map text prompt, Get a file | 🔀 ROUTING                    |
| Get a file              | Telegram                        | Downloads Telegram image file    | Route Types (Image)     | Analyze image           | 🖼️ IMAGE PROCESSING              |
| Analyze image           | Google Gemini Vision            | Analyzes image content           | Get a file              | Map image prompt        | 🖼️ IMAGE PROCESSING              |
| Map image prompt        | Set                            | Formats image description prompt | Analyze image           | Knowledge Base Agent    | 🖼️ IMAGE PROCESSING              |
| Map text prompt         | Set                            | Formats text prompt              | Route Types (Text)      | Knowledge Base Agent    | 🔀 ROUTING                       |
| Simple Memory           | Langchain Memory Buffer         | Stores conversation memory       | -                      | Knowledge Base Agent    | 🤖 AI AGENT                     |
| Knowledge Base Agent    | Langchain Agent                | Processes prompts with memory    | Map image prompt, Map text prompt, Simple Memory, Google Gemini Chat Model | Send a text message | 🤖 AI AGENT               |
| Google Gemini Chat Model| Langchain Google Gemini Model | Provides AI language model       | -                      | Knowledge Base Agent    | 🤖 AI AGENT                     |
| Send a text message     | Telegram                        | Sends AI response to Telegram    | Knowledge Base Agent    | -                      |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: Select "message" only.  
     - Leave other fields default.  
   - Connect this node as the workflow start.  
   - Configure Telegram credentials (bot token from @BotFather).

2. **Add Switch Node "Route Types"**  
   - Type: Switch  
   - Rules:  
     - Output "Text": Condition is existence of `$json.message.text`.  
     - Output "Image": Condition is existence of `$json.message.photo[2]`.  
   - Connect Telegram Trigger main output to Route Types input.

3. **Text Message Processing Branch:**

   - Add a Set node "Map text prompt"  
     - Set field `text` to expression: `={{ $json.messages[0].text.body }}`  
   - Connect Route Types "Text" output to "Map text prompt".

4. **Image Message Processing Branch:**

   - Add Telegram node "Get a file"  
     - Resource: file  
     - File ID: `={{ $json.message.photo[2].file_id }}`  
   - Connect Route Types "Image" output to "Get a file".

   - Add Langchain Google Gemini node "Analyze image"  
     - Model ID: "models/gemini-2.5-flash"  
     - Resource: image  
     - Operation: analyze  
     - Input type: binary  
   - Connect "Get a file" output to "Analyze image".

   - Add Set node "Map image prompt"  
     - Set field `text` with this template:  
       ```
       User image description:

       {{ $json.content.parts[0].text }}

       User image caption: {{ $('Telegram Trigger').item.json.message.caption }}
       ```  
   - Connect "Analyze image" output to "Map image prompt".

5. **AI Processing Setup:**

   - Add Langchain memory node "Simple Memory"  
     - Session key expression: `memory_{{ $('Telegram Trigger').item.json.message.message_id }}`  
     - Context window length: 20  
   - This node does not have direct input connections but will be linked to the agent.

   - Add Langchain Google Gemini Chat Model node "Google Gemini Chat Model"  
     - Use default parameters.  
     - Assign Google Gemini API credentials.

   - Add Langchain Agent node "Knowledge Base Agent"  
     - Text input: use `text` from either "Map text prompt" or "Map image prompt".  
     - System message left empty.  
     - Configure AI memory input connection from "Simple Memory".  
     - Configure AI Language Model input from "Google Gemini Chat Model".  
   - Connect outputs of "Map text prompt" and "Map image prompt" to "Knowledge Base Agent".

6. **Send Response Back to Telegram:**

   - Add Telegram node "Send a text message"  
     - Text: `={{ $json.output }}` (output from AI agent)  
     - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
     - Disable "appendAttribution".  
   - Connect "Knowledge Base Agent" main output to this node.

7. **Credentials Setup:**

   - Telegram API credentials with bot token from @BotFather.  
   - Google Gemini (PaLM) API credentials with valid API key from Google AI Studio.

8. **Activate Workflow and Configure Webhook:**

   - Activate the workflow to get the webhook URL.  
   - Configure Telegram webhook for your bot to point to this URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                             |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| This bot uses Google Gemini 2.5 Flash for image analysis and chat completion.                                                            | https://ai.google.dev                       |
| Conversation memory is limited to 20 messages per user message ID to maintain context without excessive memory use.                      | Workflow design                             |
| Telegram messages are formatted for readability and attractiveness, enhancing user experience.                                           | Workflow design                             |
| To create Telegram Bot Token: use @BotFather on Telegram.                                                                                | Telegram official bot creation               |
| To create Google Gemini API Key: register and generate via Google AI Studio (https://ai.google.dev).                                     | Google AI Studio                            |
| Webhook URL is generated automatically when activating the workflow.                                                                     | n8n workflow activation                      |

---

This document provides a thorough understanding of the Telegram AI bot workflow using Google Gemini 2.5 Flash, enabling users and automation agents to interpret, reproduce, and extend it confidently.