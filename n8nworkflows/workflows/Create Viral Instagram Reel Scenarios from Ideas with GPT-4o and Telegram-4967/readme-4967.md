Create Viral Instagram Reel Scenarios from Ideas with GPT-4o and Telegram

https://n8nworkflows.xyz/workflows/create-viral-instagram-reel-scenarios-from-ideas-with-gpt-4o-and-telegram-4967


# Create Viral Instagram Reel Scenarios from Ideas with GPT-4o and Telegram

### 1. Workflow Overview

This workflow automates the transformation of user-submitted ideas into viral Instagram Reel scenarios using AI (GPT-4o) and Telegram as the communication interface. It is designed for content creators, marketers, or social media managers who want to quickly generate engaging short video scripts from text or voice input. The workflow includes:

- **1.1 Input Reception:** Listens for incoming messages (text or voice) from Telegram users.
- **1.2 Input Processing:** Differentiates between text and voice inputs; transcribes voice to text if needed.
- **1.3 AI Content Generation:** Uses GPT-4o with a marketing expert prompt to generate a detailed viral Instagram Reel scenario, including script, hooks, caption, and visual ideas.
- **1.4 Output Delivery:** Sends the generated scenario back to the user on Telegram.
- **1.5 Optional Logging:** Saves generated ideas to Google Sheets (disabled by default).
- **1.6 Error Handling:** Catches and reports errors to users via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures incoming Telegram messages from users, including both text and voice notes, to trigger the workflow.

**Nodes Involved:**  
- Start: Receive Message on Telegram

**Node Details:**  
- **Start: Receive Message on Telegram**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for new messages from Telegram users. Triggers workflow on any message event.  
  - *Configuration:*  
    - Listens to "message" update type.  
    - Uses Telegram Bot credentials for authentication.  
  - *Input/Output:* No inputs; outputs message JSON with details of incoming message.  
  - *Edge Cases:*  
    - Bot token misconfiguration leads to authentication failure.  
    - Telegram API downtime or message delivery delays may cause missed triggers.

#### 2.2 Input Processing and Routing

**Overview:**  
This block routes the incoming message based on its content type: audio voice note, text message, or error. Voice notes are downloaded and transcribed to text.

**Nodes Involved:**  
- Route by Input Type  
- Get Voice Message  
- Transcribe Voice to Text  
- Set User Input  
- Set Error Message  
- Send Error Message to Telegram

**Node Details:**  
- **Route by Input Type**  
  - *Type:* Switch  
  - *Role:* Determines if the message contains voice, text, or error.  
  - *Configuration:*  
    - Checks for existence of `message.voice.file_id` → routes to Audio output.  
    - Checks for existence of `message.text` → routes to Text output.  
    - Checks for existence of `error` → routes to Error output.  
  - *Edge Cases:*  
    - Messages without text or voice trigger error branch.  
    - Unexpected message formats could cause routing failures.

- **Get Voice Message**  
  - *Type:* Telegram  
  - *Role:* Downloads the audio file from Telegram servers using the voice message file ID.  
  - *Configuration:*  
    - Uses `message.voice.file_id` to fetch the audio.  
  - *Edge Cases:*  
    - File ID invalid or expired.  
    - Telegram API rate limits or errors.

- **Transcribe Voice to Text**  
  - *Type:* Langchain OpenAI (Audio Transcription)  
  - *Role:* Converts downloaded voice audio into text via OpenAI transcription.  
  - *Configuration:*  
    - Operation set to "transcribe" on audio resource.  
    - Uses OpenAI API credentials.  
  - *Edge Cases:*  
    - Audio quality poor, leading to inaccurate transcription.  
    - API errors or rate limits.

- **Set User Input**  
  - *Type:* Set  
  - *Role:* Prepares text input from Telegram text message for AI processing.  
  - *Configuration:*  
    - Assigns `text` field from `message.text`.  
  - *Edge Cases:*  
    - Empty or malformed text inputs.

- **Set Error Message**  
  - *Type:* Set  
  - *Role:* Creates a generic error message for user notification.  
  - *Configuration:*  
    - Sets `Error` string value "An error has occurred".  
  - *Edge Cases:* None.

- **Send Error Message to Telegram**  
  - *Type:* Telegram  
  - *Role:* Sends error messages back to Telegram user.  
  - *Configuration:*  
    - Sends text from incoming error JSON.  
    - Uses same chat ID from original user message.  
  - *Edge Cases:*  
    - Telegram API errors.

#### 2.3 AI Content Generation

**Overview:**  
This core block uses GPT-4o and Langchain to generate viral Instagram Reel scripts, hooks, captions, and visual ideas based on the user’s input idea.

**Nodes Involved:**  
- AI Model: GPT-4o  
- Memory for Chat Context  
- Generate Reels Scenario with AI  
- Optional: Log Ideas to Google Sheets (disabled by default)

**Node Details:**  
- **AI Model: GPT-4o**  
  - *Type:* Langchain OpenAI Chat Language Model  
  - *Role:* Provides access to the GPT-4o model for text generation.  
  - *Configuration:*  
    - Model set to "gpt-4o".  
    - Uses OpenAI API credentials.  
  - *Edge Cases:*  
    - API limits or errors.  
    - Version-specific compatibility with Langchain node version 1.2.

- **Memory for Chat Context**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Manages conversational memory context for the AI to maintain session continuity.  
  - *Configuration:*  
    - Session key set dynamically from Telegram chat ID to separate user sessions.  
    - Context window length limited to 10 messages for efficiency.  
  - *Edge Cases:*  
    - Missing or corrupted session keys; context loss.

- **Generate Reels Scenario with AI**  
  - *Type:* Langchain Agent  
  - *Role:* Implements the AI prompt and generates the full Instagram Reel scenario text output.  
  - *Configuration:*  
    - Receives input text from either transcription or text message.  
    - Contains an elaborate system prompt defining the marketing expert persona and detailed instructions for output formatting including script sections, hook variants, caption, and visual ideas.  
    - Returns structured formatted text output for Telegram delivery.  
  - *Key Expressions:*  
    - Uses `{{$json.text}}` as input text.  
    - Prompt includes placeholders and strict formatting instructions.  
  - *Edge Cases:*  
    - Prompt interpretation errors.  
    - AI hallucinations or irrelevant output.  
    - Input text too short or ambiguous.

- **Optional: Log Ideas to Google Sheets** (Disabled)  
  - *Type:* Google Sheets Tool  
  - *Role:* Appends generated scenarios and metadata to a Google Sheet for archival.  
  - *Configuration:*  
    - Maps Date, Script, Status, Description columns with dynamic values including AI-generated text overrides.  
    - Requires Google OAuth2 credentials.  
  - *Edge Cases:*  
    - Google API auth errors.  
    - Permission or quota issues.  
    - Workflow disabled by default.

#### 2.4 Output Delivery

**Overview:**  
Sends the AI-generated Instagram Reel scenario text back to the Telegram user who submitted the original idea.

**Nodes Involved:**  
- Send Scenario to Telegram

**Node Details:**  
- **Send Scenario to Telegram**  
  - *Type:* Telegram  
  - *Role:* Delivers the AI output scenario text message to the user's Telegram chat.  
  - *Configuration:*  
    - Sends text from AI agent output (`$json.output`).  
    - Uses chat ID dynamically from the original message.  
    - Attribution appending disabled to keep message clean.  
    - Uses Telegram Bot credentials.  
  - *Edge Cases:*  
    - Telegram API errors or downtime.  
    - Message length limits in Telegram (may truncate or fail).

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                          | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                                  |
|-------------------------------|----------------------------------|----------------------------------------|----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Start: Receive Message on Telegram | Telegram Trigger                 | Entry point; listens for Telegram messages | None                             | Route by Input Type               | 🚀 Quick Start Guide: Connect Telegram, OpenAI, optionally Google Sheets, then activate and send ideas via bot |
| Route by Input Type            | Switch                          | Routes messages by content type (voice/text/error) | Start: Receive Message on Telegram | Get Voice Message, Set User Input, Set Error Message |                                                                                                              |
| Get Voice Message              | Telegram                        | Downloads voice message audio file      | Route by Input Type (Audio)       | Transcribe Voice to Text          |                                                                                                              |
| Transcribe Voice to Text       | Langchain OpenAI (Audio)        | Transcribes voice to text via OpenAI    | Get Voice Message                 | Generate Reels Scenario with AI   |                                                                                                              |
| Set User Input                | Set                             | Prepares text input for AI processing   | Route by Input Type (Text)        | Generate Reels Scenario with AI   |                                                                                                              |
| Set Error Message             | Set                             | Creates a generic error message          | Route by Input Type (Error)       | Send Error Message to Telegram    |                                                                                                              |
| Send Error Message to Telegram | Telegram                        | Sends error messages back to user       | Set Error Message                 | None                             |                                                                                                              |
| AI Model: GPT-4o              | Langchain OpenAI Chat Model      | Provides GPT-4o model for text generation | Generate Reels Scenario with AI    | Generate Reels Scenario with AI   |                                                                                                              |
| Memory for Chat Context       | Langchain Memory Buffer Window   | Manages conversational memory context   | AI Model: GPT-4o                  | Generate Reels Scenario with AI   |                                                                                                              |
| Generate Reels Scenario with AI | Langchain Agent                  | Generates Reel scenario, hooks, captions, visuals | Set User Input, Transcribe Voice to Text, AI Model: GPT-4o, Memory for Chat Context, Optional: Log Ideas to Google Sheets | Send Scenario to Telegram        |                                                                                                              |
| Send Scenario to Telegram     | Telegram                        | Sends generated scenario back to Telegram user | Generate Reels Scenario with AI    | None                             | Optional: Log generated scenarios in Google Sheets for archival                                              |
| Optional: Log Ideas to Google Sheets | Google Sheets Tool               | (Optional) Logs ideas and metadata to Google Sheets | Generate Reels Scenario with AI   | None                             | Optional: Ask your bot to store the generated scenario in your Google Sheet                                  |
| Sticky Note                   | Sticky Note                    | Quick start guide for setup and usage    | None                             | None                             | 🚀 Quick Start Guide: Connect Telegram, OpenAI, optionally Google Sheets, then activate and send ideas via bot |
| Sticky Note1                  | Sticky Note                    | Reminder about optional logging feature  | None                             | None                             | Optional: Ask your bot to store the generated scenario in your google sheet*                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Set "Updates" parameter to listen for "message".  
   - Connect your Telegram Bot credentials.  
   - Position this node as the workflow start.

2. **Add Switch Node to Route Input**  
   - Type: Switch  
   - Create three outputs with conditions:  
     - Audio: Check if `message.voice.file_id` exists.  
     - Text: Check if `message.text` exists.  
     - Error: Check if `error` exists.  
   - Connect Start Telegram Trigger output to this Switch node.

3. **Add Telegram Node to Download Voice Messages**  
   - Type: Telegram  
   - Resource: File  
   - Operation: Get file by ID using `message.voice.file_id` expression.  
   - Connect Audio output from Switch to this node.

4. **Add Langchain OpenAI Node to Transcribe Audio**  
   - Type: Langchain OpenAI (Audio resource)  
   - Operation: Transcribe  
   - Connect output of Get Voice Message node here.  
   - Connect your OpenAI API credentials.

5. **Add Set Node to Prepare Text Input**  
   - Type: Set  
   - Assign a new field `text` with value from `message.text`.  
   - Connect Text output from Switch to this node.

6. **Create Set Node for Error Message**  
   - Type: Set  
   - Assign a field `Error` with value "An error has occurred".  
   - Connect Error output from Switch to this node.

7. **Create Telegram Node to Send Error Messages**  
   - Type: Telegram  
   - Text: Use expression to send error message from previous node.  
   - Chat ID: Extract from original message chat ID.  
   - Connect Set Error Message node to this node.

8. **Add Langchain OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: Set to "gpt-4o".  
   - Connect OpenAI API credentials.

9. **Add Langchain Memory Buffer Node**  
   - Type: Langchain Memory Buffer Window  
   - Session Key: Use expression to extract Telegram chat ID for session isolation.  
   - Context Window Length: 10.  
   - Connect AI Model node output to this memory node.

10. **Add Langchain Agent Node for Scenario Generation**  
   - Type: Langchain Agent  
   - Input Parameter: Set to use field `text` from either transcription or Set User Input node.  
   - System Prompt: Insert a detailed marketing expert prompt specifying output format (script, hooks, captions, visuals).  
   - Connect outputs from Memory node, AI Model node, and both Transcribe and Set User Input nodes as inputs.  
   - Optionally connect Google Sheets node as a tool input (disabled by default).

11. **Add Telegram Node to Send Generated Scenario**  
   - Type: Telegram  
   - Text: Use output from Langchain Agent node (the generated scenario).  
   - Chat ID: Use Telegram chat ID from initial message.  
   - Connect Langchain Agent output to this node.

12. **(Optional) Add Google Sheets Node to Log Ideas**  
   - Type: Google Sheets Tool (Append operation)  
   - Configure sheet with columns: Date, Script, Status, Description.  
   - Map AI-generated fields accordingly.  
   - Connect this node as an AI tool input to the Langchain Agent node.  
   - Authenticate with Google Sheets OAuth2 credentials.  
   - Leave disabled if not needed.

13. **Configure Credentials**  
   - Telegram Bot: Create bot with BotFather, get token, and configure in n8n credentials.  
   - OpenAI API: Generate API key, set in n8n credentials.  
   - Google Sheets OAuth2 (optional): Configure OAuth2 for Google Sheets API.

14. **Activate Workflow**  
   - Save and activate the workflow.  
   - Test by sending text or voice message to the Telegram bot.  
   - Verify that the bot returns a detailed Instagram Reel scenario.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                           |
|-------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| 🚀 Quick Start Guide: Connect Telegram, OpenAI, optionally Google Sheets. Activate workflow. Send ideas via Telegram bot. | Sticky Note at workflow start node                        |
| Optional: Ask your bot to store the generated scenario in your Google Sheet (disabled by default).                      | Sticky Note near Google Sheets node                       |
| Telegram Bot creation link: https://telegram.me/BotFather                                                                | Instruction for bot token setup                            |
| Prompt design inspired by top marketing legends (Russell Brunson, Dan Kennedy, Gary Halbert, etc.) and viral Reel tactics | Embedded in AI Agent system prompt                         |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated n8n workflow. All content complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.