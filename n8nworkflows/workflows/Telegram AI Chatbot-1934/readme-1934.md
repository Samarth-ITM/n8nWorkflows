Telegram AI Chatbot

https://n8nworkflows.xyz/workflows/telegram-ai-chatbot-1934


# Telegram AI Chatbot

### 1. Workflow Overview

The **Telegram AI Chatbot** workflow enables interactive communication between Telegram users and an AI-powered chatbot leveraging OpenAI's services. It listens for incoming Telegram messages, then dynamically handles them based on their content:

- **Chat Messages**: Replies with AI-generated text responses in the user's language.
- **Image Generation Commands**: Creates and sends AI-generated images based on user prompts.
- **Unsupported Commands**: Sends an error message guiding the user on valid commands.

The workflow also simulates typing or upload actions in Telegram for a more natural user experience and includes multiple sticky notes for documentation and guidance.

The workflow logic can be divided into these main blocks:

- **1.1 Input Reception and Preprocessing**: Captures Telegram messages and prepares data.
- **1.2 Typing Action Simulation**: Sends typing or upload indicators to Telegram.
- **1.3 Command Analysis and Routing**: Determines message type and routes to appropriate handler.
- **1.4 Chatbot Text Response Handling**: Generates and sends AI chat replies.
- **1.5 Image Generation Handling**: Creates and sends AI-generated images.
- **1.6 Unsupported Command Handling**: Sends an error message for unsupported commands.
- **1.7 Documentation and Notes**: Sticky notes providing explanations and user guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preprocessing

**Overview:**  
This block listens to incoming Telegram messages and prepares the message text for further processing.

**Nodes Involved:**  
- Telegram Trigger  
- PreProcessing  
- Settings

**Node Details:**  

- **Telegram Trigger**  
  - Type: Trigger  
  - Role: Listens for all Telegram updates/messages to the bot.  
  - Configuration: Listens to all update types (`updates: ["*"]`), uses Telegram bot credentials.  
  - Inputs: External Telegram webhook  
  - Outputs: Emits incoming message JSON objects.  
  - Failures: Webhook connectivity issues, credential errors.  
  - Version: 1

- **PreProcessing**  
  - Type: Set  
  - Role: Extracts and normalizes the message text into a dot notation JSON field `message.text`.  
  - Configuration: Sets `message.text` to the text of the Telegram message, or empty string if none.  
  - Inputs: From Telegram Trigger  
  - Outputs: JSON with normalized message text.  
  - Edge Cases: Messages without text (e.g., stickers, images) are handled by setting empty string.  
  - Version: 2

- **Settings**  
  - Type: Set  
  - Role: Defines key parameters for AI model behavior and chat flow such as model temperature, max tokens, system prompt, and typing action.  
  - Configuration:  
    - `model_temperature` = 0.8 (controls AI creativity)  
    - `token_length` = 500 (max token length for AI response)  
    - `system_command` = System prompt instructing AI to behave as a friendly chatbot, detect user language, reply in the same language, and use emojis.  
    - `bot_typing` = Conditional: if message starts with "/image" then "upload_photo" else "typing" (Telegram chat action).  
  - Inputs: From PreProcessing  
  - Outputs: Enhanced JSON including AI parameters and typing action.  
  - Version: 2  
  - Edge Cases: If message text is missing, `bot_typing` defaults to "typing".

---

#### 1.2 Typing Action Simulation

**Overview:**  
Simulates a typing or upload action in Telegram to improve user experience by signaling the bot is processing the user’s request.

**Nodes Involved:**  
- Send Typing action  
- Merge

**Node Details:**  

- **Send Typing action**  
  - Type: Telegram  
  - Role: Sends a chat action to Telegram (typing indicator or upload photo indicator) based on the `bot_typing` parameter.  
  - Configuration:  
    - `action`: Dynamic expression based on message type (`typing` or `upload_photo`).  
    - `chatId`: User's chat ID from incoming message.  
    - Operation: `sendChatAction`.  
  - Inputs: From Settings  
  - Outputs: Passes data downstream after sending the action.  
  - Failures: Telegram API errors, invalid chat ID.  
  - Version: 1  

- **Merge**  
  - Type: Merge (chooseBranch mode)  
  - Role: Combines outputs from Send Typing action and routes to command analysis.  
  - Configuration: `chooseBranch` mode to select between multiple branches.  
  - Inputs: From Send Typing action and Settings  
  - Outputs: To CheckCommand node for command routing.  
  - Version: 2.1  

---

#### 1.3 Command Analysis and Routing

**Overview:**  
Determines the nature of the user message: greeting, chat message, image creation command, or unsupported command, then routes accordingly.

**Nodes Involved:**  
- CheckCommand

**Node Details:**  

- **CheckCommand**  
  - Type: Switch  
  - Role: Analyzes the message text to decide the processing branch.  
  - Configuration:  
    - Input: `{{$json.message?.text}}` (user message text)  
    - Rules:  
      - Output 1: If text does NOT start with "/" → regular chat message (chat mode).  
      - Output 2: If text starts with "/start" → greeting mode.  
      - Output 3: If text starts with "=/image " → image generation mode.  
      - Fallback output (3): unsupported commands.  
  - Inputs: From Merge  
  - Outputs: Branches to Chat_mode, Greeting, Create an image, or Send error message nodes.  
  - Edge Cases: Text with leading spaces or mixed case may not match; command parsing is literal and case-sensitive.  
  - Version: 1  

---

#### 1.4 Chatbot Text Response Handling

**Overview:**  
Generates a textual AI response using OpenAI's GPT-4 model and sends it back to the user via Telegram.

**Nodes Involved:**  
- Chat_mode  
- Text reply

**Node Details:**  

- **Chat_mode**  
  - Type: OpenAI (Chat Completion)  
  - Role: Sends prompt to OpenAI GPT-4 chat completion API to generate chatbot reply.  
  - Configuration:  
    - Model: `gpt-4`  
    - Prompt: System message from `system_command` parameter, user message text as content.  
    - Options: Max tokens and temperature dynamically set from parameters.  
  - Inputs: From CheckCommand (chat message branch)  
  - Outputs: AI-generated text in response.  
  - Credentials: OpenAI API credentials needed.  
  - Failures: Network errors, API rate limits, invalid credentials.  
  - Version: 1  

- **Text reply**  
  - Type: Telegram  
  - Role: Sends the AI-generated text response to the Telegram user.  
  - Configuration:  
    - Text: AI message content from Chat_mode output.  
    - Chat ID: Extracted dynamically from original user message.  
    - Parse mode: Markdown for formatted text.  
  - Inputs: From Chat_mode (and Greeting)  
  - Outputs: To Telegram user.  
  - Failures: Telegram API errors, invalid chat ID.  
  - Version: 1  

---

#### 1.5 Image Generation Handling

**Overview:**  
Handles image generation requests by creating AI images using OpenAI’s image API and sending them to users.

**Nodes Involved:**  
- Create an image  
- Send image

**Node Details:**  

- **Create an image**  
  - Type: OpenAI (Image Generation)  
  - Role: Calls OpenAI's image creation API with user prompt (message text after "/image" command).  
  - Configuration:  
    - Prompt: Extracted by splitting message text and joining words after the "/image" command.  
    - Options: Generates 1 image, 512x512 pixels, response format is image URL.  
  - Inputs: From CheckCommand (image command branch)  
  - Outputs: Image URL in response.  
  - Credentials: OpenAI API credentials needed.  
  - Failures: API errors, invalid prompts, rate limits.  
  - Version: 1  

- **Send image**  
  - Type: Telegram  
  - Role: Sends the generated image to the Telegram user as a photo.  
  - Configuration:  
    - File: URL of generated image from Create an image node.  
    - Chat ID: User's chat ID from Settings node.  
    - Operation: `sendPhoto`.  
  - Inputs: From Create an image  
  - Outputs: To Telegram user.  
  - Failures: Telegram API errors, invalid URLs, chat ID errors.  
  - Version: 1  

---

#### 1.6 Unsupported Command Handling

**Overview:**  
Sends an error message to the user if an unsupported command is detected.

**Nodes Involved:**  
- Send error message  
- Sticky Note (Error fallback)

**Node Details:**  

- **Send error message**  
  - Type: Telegram  
  - Role: Sends a friendly error message guiding the user on supported commands.  
  - Configuration:  
    - Text: Personalized message with instructions on supported commands (`/image [prompt]`) with markdown formatting.  
    - Chat ID: From user message.  
    - Parse mode: Markdown.  
  - Inputs: From CheckCommand fallback branch  
  - Outputs: To Telegram user.  
  - Failures: Telegram API errors, invalid chat ID.  
  - Version: 1  

- **Sticky Note (Error fallback)**  
  - Type: Sticky Note node  
  - Role: Provides a visible note in workflow editor describing error fallback behavior.  
  - Content: "Error fallback for unsupported commands"  
  - Position: Near error handling nodes.  
  - Inputs: None  
  - Outputs: None  

---

#### 1.7 Documentation and Notes

**Overview:**  
Sticky note nodes document various parts of the workflow for clarity and maintenance purposes.

**Nodes Involved:**  
- Sticky Note1 (Chatbot mode)  
- Sticky Note2 (Welcome message)  
- Sticky Note3 (Create an image)

**Node Details:**  

- **Sticky Note1**  
  - Content: "Chatbot mode by default\n(when no command is provided)"  
  - Positioned near Chat_mode node.

- **Sticky Note2**  
  - Content: "Welcome message\n/start"  
  - Positioned near Greeting node.

- **Sticky Note3**  
  - Content: "Create an image\n/image + request"  
  - Positioned near Create an image node.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                      | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                      |
|--------------------|---------------------|------------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Trigger    | Telegram Trigger    | Receive Telegram messages           | External webhook         | PreProcessing           |                                                                                                 |
| PreProcessing      | Set                 | Normalize message text              | Telegram Trigger          | Settings                |                                                                                                 |
| Settings           | Set                 | Define AI parameters & typing action| PreProcessing             | Send Typing action, Merge|                                                                                                 |
| Send Typing action | Telegram             | Send typing/upload action to user  | Settings                  | Merge                   |                                                                                                 |
| Merge              | Merge                | Combine typing action and continue  | Send Typing action, Settings| CheckCommand           |                                                                                                 |
| CheckCommand       | Switch               | Route message by command type       | Merge                     | Chat_mode, Greeting, Create an image, Send error message |                                                                                                 |
| Chat_mode          | OpenAI (Chat)        | Generate AI chat text response      | CheckCommand              | Text reply              | Sticky Note1: "Chatbot mode by default (when no command is provided)"                           |
| Greeting           | OpenAI (Chat)        | Generate greeting message           | CheckCommand              | Text reply              | Sticky Note2: "Welcome message /start"                                                        |
| Text reply         | Telegram             | Send AI-generated text to user      | Chat_mode, Greeting       |                         |                                                                                                 |
| Create an image    | OpenAI (Image)       | Generate image from prompt           | CheckCommand              | Send image              | Sticky Note3: "Create an image /image + request"                                               |
| Send image         | Telegram             | Send generated image to user        | Create an image           |                         |                                                                                                 |
| Send error message | Telegram             | Send unsupported command error      | CheckCommand              |                         | Sticky Note: "Error fallback for unsupported commands"                                         |
| Sticky Note        | Sticky Note          | Document error fallback              |                          |                         | "Error fallback for unsupported commands"                                                     |
| Sticky Note1       | Sticky Note          | Document chatbot mode default        |                          |                         | "Chatbot mode by default (when no command is provided)"                                       |
| Sticky Note2       | Sticky Note          | Document welcome message             |                          |                         | "Welcome message /start"                                                                       |
| Sticky Note3       | Sticky Note          | Document image creation command      |                          |                         | "Create an image /image + request"                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for all updates (`updates: ["*"]`)  
   - Use Telegram bot credentials with bot token  
   - Position accordingly (e.g., x=20, y=340)  

2. **Create PreProcessing Node**  
   - Type: Set  
   - Enable dot notation  
   - Set field `message.text` to `{{$json?.message?.text || ""}}`  
   - Connect output of Telegram Trigger → PreProcessing  

3. **Create Settings Node**  
   - Type: Set  
   - Add parameters:  
     - number: `model_temperature` = 0.8  
     - number: `token_length` = 500  
     - string: `system_command` = `"You are a friendly chatbot. User name is {{ $json?.message?.from?.first_name }}. User system language is {{ $json?.message?.from?.language_code }}. First, detect user text language. Next, provide your reply in the same language. Include several suitable emojis in your answer."`  
     - string: `bot_typing` = `{{$json?.message?.text.startsWith('/image') ? "upload_photo" : "typing"}}`  
   - Connect PreProcessing → Settings  

4. **Create Send Typing action Node**  
   - Type: Telegram  
   - Operation: `sendChatAction`  
   - Action: `{{$json.bot_typing}}` (dynamic)  
   - Chat ID: `{{$json.message.from.id}}`  
   - Connect Settings → Send Typing action  

5. **Create Merge Node**  
   - Type: Merge  
   - Mode: `chooseBranch`  
   - Connect Send Typing action → Merge (branch 0)  
   - Connect Settings → Merge (branch 1)  

6. **Create CheckCommand Node**  
   - Type: Switch  
   - Value1: `{{$json.message?.text}}` (string)  
   - Rules:  
     - Output 1: NOT starts with "/" (regular chat)  
     - Output 2: starts with "/start" (greeting)  
     - Output 3: starts with "=/image " (image creation)  
     - Fallback: unsupported command  
   - Connect Merge → CheckCommand  

7. **Create Chat_mode Node**  
   - Type: OpenAI (Chat Completion)  
   - Model: `gpt-4`  
   - Prompt messages:  
     - System: from `system_command` parameter  
     - User: `{{$json.message.text}}`  
   - Options: maxTokens from `token_length`, temperature from `model_temperature`  
   - Connect CheckCommand output 1 → Chat_mode  

8. **Create Greeting Node**  
   - Type: OpenAI (Chat Completion)  
   - Model: `gpt-4`  
   - Prompt messages:  
     - System: from `system_command` parameter  
     - User: `"This is the first message from a user. Please welcome a new user in '{{ $json.message.from.language_code }}' language"`  
   - Options: maxTokens from `token_length`, temperature from `model_temperature`  
   - Connect CheckCommand output 2 → Greeting  

9. **Create Text reply Node**  
   - Type: Telegram  
   - Text: `{{$json.message.content}}` (output from Chat_mode or Greeting)  
   - Chat ID: `{{$('Settings').first().json.message.from.id}}`  
   - Parse mode: Markdown  
   - Connect Chat_mode → Text reply  
   - Connect Greeting → Text reply  

10. **Create Create an image Node**  
    - Type: OpenAI (Image Generation)  
    - Prompt: `{{$json.message.text.split(' ').slice(1).join(' ')}}`  
    - Options: n=1, size=512x512, responseFormat=imageUrl  
    - Connect CheckCommand output 3 → Create an image  

11. **Create Send image Node**  
    - Type: Telegram  
    - Operation: sendPhoto  
    - File: `{{$json.url}}` (image URL from Create an image)  
    - Chat ID: `{{$('Settings').first().json.message.from.id}}`  
    - Connect Create an image → Send image  

12. **Create Send error message Node**  
    - Type: Telegram  
    - Text:  
      ```
      Sorry, {{$json.message.from.first_name}}! This command is not supported yet. Please type some text to a chat bot or try this command:
      
      /image [your prompt]
      
      Enter the command, then space and provide your request. Example:
      
      `/image a picture or a cute little kitten with big eyes. Miyazaki studio ghibli style`
      ```  
    - Chat ID: `{{$json.message.from.id}}`  
    - Parse mode: Markdown  
    - Connect CheckCommand fallback output → Send error message  

13. **Add Sticky Notes** (optional but recommended for maintenance)  
    - Near Chat_mode: "Chatbot mode by default (when no command is provided)"  
    - Near Greeting: "Welcome message /start"  
    - Near Create an image: "Create an image /image + request"  
    - Near Send error message: "Error fallback for unsupported commands"  

14. **Activate workflow** and test with various Telegram messages including:  
    - Plain text chat messages  
    - `/start` command  
    - `/image cat with sunglasses`  
    - Unsupported commands (e.g., `/foo`)

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                |
|-------------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow demonstrates integration of Telegram bot with OpenAI GPT-4 for chat and DALL·E for image generation. | Core use case for conversational AI and creative image generation. |
| Use of Telegram chat actions (`typing`, `upload_photo`) improves UX by indicating bot activity. | Telegram Bot API documentation: https://core.telegram.org/bots/api#sendchataction |
| Command parsing is case-sensitive and literal; consider extending with regex or normalization for robustness. | Potential improvement for production use.     |
| Sticky notes provide clear documentation inside n8n editor for easier maintenance and onboarding. | Best practice for complex workflows.           |
| OpenAI API credentials must have chat and image generation permissions enabled.                  | OpenAI API docs: https://platform.openai.com/docs/ |
| Telegram bot must be configured with webhook URL matching n8n instance for triggers to work.    | Telegram Bot API webhook setup instructions.   |

---

This document provides a comprehensive understanding of the Telegram AI Chatbot workflow, enabling reproduction, extension, and troubleshooting with ease.