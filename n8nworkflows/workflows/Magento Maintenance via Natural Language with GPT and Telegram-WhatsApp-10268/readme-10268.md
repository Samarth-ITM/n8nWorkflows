Magento Maintenance via Natural Language with GPT and Telegram/WhatsApp

https://n8nworkflows.xyz/workflows/magento-maintenance-via-natural-language-with-gpt-and-telegram-whatsapp-10268


# Magento Maintenance via Natural Language with GPT and Telegram/WhatsApp

### 1. Workflow Overview

This workflow enables Magento store maintenance through natural language commands received via Telegram and WhatsApp. It leverages AI (OpenAI GPT models) to interpret user input—both text and audio—and executes corresponding Magento commands over SSH. The workflow manages multi-channel inputs, audio transcription, AI processing, command execution, and message responses.

**Logical Blocks:**

- **1.1 Input Reception:** Captures incoming messages from Telegram and WhatsApp, including both text and audio or media inputs.
- **1.2 Media Download and Audio Transcription:** Downloads voice messages or media files and transcribes audio into text using OpenAI.
- **1.3 AI Processing and Command Interpretation:** Uses OpenAI GPT models and an AI Agent configured for Magento to interpret user intent and generate executable commands.
- **1.4 Command Execution:** Runs the interpreted commands via SSH on the Magento server.
- **1.5 Response Handling:** Sends execution results or error messages back to users on Telegram or WhatsApp.
- **1.6 Routing and Condition Checks:** Directs workflow execution based on message content and AI agent decision outcomes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures incoming messages from Telegram and WhatsApp, handling both text and media types, and routes them for further processing.

**Nodes Involved:**  
- Message Trigger (Telegram)  
- WhatsApp Trigger  
- Route Chat Input (Switch node)

**Node Details:**

- **Message Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages (text/audio).  
  - Config: Default webhook ID dedicated to Telegram messages.  
  - Connections: Outputs to Route Chat Input.  
  - Potential Failures: Webhook misconfiguration, Telegram API limits.  

- **WhatsApp Trigger**  
  - Type: WhatsApp Trigger  
  - Role: Listens for incoming WhatsApp messages (text/media).  
  - Config: Webhook set for WhatsApp messages.  
  - Connections: Outputs to Route Chat Input.  
  - Potential Failures: WhatsApp API quota limits, webhook issues.  

- **Route Chat Input**  
  - Type: Switch  
  - Role: Routes input based on content type (text, audio, media).  
  - Config: Routes to AI processing for text, to audio/media download or transcription for voice inputs.  
  - Connections: Outputs to Magento AI Agent, Get audio file, Download media nodes.  
  - Potential Failures: Incorrect routing logic could misclassify inputs.

#### 2.2 Media Download and Audio Transcription

**Overview:**  
Downloads audio or media files from incoming messages and transcribes audio to text for AI processing.

**Nodes Involved:**  
- Download media (WhatsApp)  
- HTTP Request (for media download)  
- Get audio file (Telegram)  
- Transcribe audio (OpenAI)

**Node Details:**

- **Download media**  
  - Type: WhatsApp node  
  - Role: Downloads media attached to WhatsApp messages.  
  - Config: Uses WhatsApp webhook ID and media URLs from incoming messages.  
  - Connections: Outputs to HTTP Request for actual media content fetch.  
  - Potential Failures: Media URL expiration, download failures, API errors.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Fetches media content from the provided URL.  
  - Config: GET request with media URL, no authentication needed if WhatsApp media URL is public or token-based.  
  - Connections: Outputs to Transcribe audio.  
  - Potential Failures: Timeout, bad URLs, network errors.

- **Get audio file**  
  - Type: Telegram node  
  - Role: Retrieves audio files from Telegram voice messages.  
  - Config: Uses Telegram webhook ID.  
  - Connections: Outputs to Transcribe audio.  
  - Potential Failures: Telegram API limits, file unavailability.

- **Transcribe audio**  
  - Type: OpenAI (Langchain)  
  - Role: Converts audio files into text using OpenAI speech-to-text capabilities.  
  - Config: Uses OpenAI credentials configured via Langchain node.  
  - Connections: Outputs transcribed text to Magento AI Agent.  
  - Potential Failures: Audio format unsupported, transcription errors, API limits.

#### 2.3 AI Processing and Command Interpretation

**Overview:**  
Processes text input (direct or transcribed) using an AI agent specialized for Magento commands to generate executable commands or responses.

**Nodes Involved:**  
- OpenAI Chat Model  
- Magento AI Agent 👩🏻‍🏫  
- If (decision node)

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model node  
  - Role: Provides GPT language model capabilities for natural language understanding.  
  - Config: Default parameters, linked as language model for Magento AI Agent.  
  - Connections: AI languageModel input to Magento AI Agent.  
  - Potential Failures: API key invalidation, rate limits.

- **Magento AI Agent 👩🏻‍🏫**  
  - Type: Langchain Agent node  
  - Role: Specialized AI agent interpreting commands for Magento.  
  - Config: Connected to OpenAI Chat Model; configured with Magento domain knowledge.  
  - Connections: Outputs to If node for decision branching.  
  - Potential Failures: Misinterpretation of commands, model latency, API errors.

- **If**  
  - Type: If node  
  - Role: Branches workflow based on AI agent output (e.g., whether command is valid or requires execution).  
  - Config: Conditions evaluate AI agent results.  
  - Connections: True branch to Execute a command node; False branch to error handling.  
  - Potential Failures: Incorrect condition expressions, empty outputs.

#### 2.4 Command Execution

**Overview:**  
Executes interpreted commands on the Magento server through SSH and routes responses.

**Nodes Involved:**  
- Execute a command (SSH)  
- If1 (decision node)

**Node Details:**

- **Execute a command**  
  - Type: SSH node  
  - Role: Runs shell commands on Magento server.  
  - Config: SSH credentials (host, username, private key or password) preconfigured.  
  - Connections: Outputs to If1 node for success/failure check.  
  - Potential Failures: SSH connection failure, command errors, timeouts.

- **If1**  
  - Type: If node  
  - Role: Checks if command execution succeeded (e.g., exit code or output).  
  - Config: Condition based on SSH node output status.  
  - Connections: True branch to Send message (WhatsApp) or Send Output (Telegram); False branch to Send Error Message.  
  - Potential Failures: Condition misconfiguration, unexpected SSH output format.

#### 2.5 Response Handling

**Overview:**  
Sends the output or error messages back to users on Telegram and WhatsApp channels.

**Nodes Involved:**  
- Send Output (Telegram)  
- Send message (WhatsApp)  
- Send message1 (WhatsApp)  
- Send Error Message (Telegram)

**Node Details:**

- **Send Output**  
  - Type: Telegram node  
  - Role: Sends successful command output back to Telegram user.  
  - Config: Uses Telegram webhook ID and message content from SSH output.  
  - Connections: Terminal node.  
  - Potential Failures: Telegram API errors, message formatting issues.

- **Send message**  
  - Type: WhatsApp node  
  - Role: Sends successful command output back to WhatsApp user.  
  - Config: Uses WhatsApp webhook ID and message from SSH output.  
  - Connections: Terminal node.  
  - Potential Failures: WhatsApp API limits, message delivery failures.

- **Send message1**  
  - Type: WhatsApp node  
  - Role: Sends error messages or notifications to WhatsApp users.  
  - Config: Similar to Send message with error content.  
  - Connections: Terminal node.  
  - Potential Failures: Same as Send message.

- **Send Error Message**  
  - Type: Telegram node  
  - Role: Communicates errors in command execution or AI processing to Telegram users.  
  - Config: Uses Telegram webhook ID and error message content.  
  - Connections: Terminal node.  
  - Potential Failures: Telegram API or formatting errors.

#### 2.6 Routing and Condition Checks

**Overview:**  
Controls workflow branching and ensures correct routing depending on input type and processing outcomes.

**Nodes Involved:**  
- If (decision node after AI Agent)  
- If1 (decision node after SSH Exec)  
- If2 (decision node, connected from If node true branch)

**Node Details:**

- **If**  
  - Already described in AI Processing block.

- **If1**  
  - Already described in Command Execution block.

- **If2**  
  - Type: If node  
  - Role: Additional condition checking on AI Agent output before command execution or error sending.  
  - Config: Evaluates flags or output status from AI Agent or previous nodes.  
  - Connections: True branch routes to Send message1; False branch routes to Send Error Message.  
  - Potential Failures: Logic errors or empty input causing misrouting.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                      | Input Node(s)          | Output Node(s)        | Sticky Note                     |
|-----------------------|----------------------------------|------------------------------------|-----------------------|-----------------------|--------------------------------|
| Message Trigger       | Telegram Trigger                 | Receives Telegram messages          | -                     | Route Chat Input      |                                |
| WhatsApp Trigger      | WhatsApp Trigger                | Receives WhatsApp messages          | -                     | Route Chat Input      |                                |
| Route Chat Input      | Switch                         | Routes input by content type        | Message Trigger, WhatsApp Trigger | Magento AI Agent, Get audio file, Download media |                                |
| Get audio file        | Telegram                        | Retrieves Telegram audio files      | Route Chat Input       | Transcribe audio      |                                |
| Download media        | WhatsApp                       | Downloads WhatsApp media            | Route Chat Input       | HTTP Request          |                                |
| HTTP Request          | HTTP Request                   | Downloads media content             | Download media         | Transcribe audio      |                                |
| Transcribe audio      | OpenAI (Langchain)             | Transcribes audio to text           | Get audio file, HTTP Request | Magento AI Agent      |                                |
| OpenAI Chat Model     | Langchain OpenAI Chat Model    | Provides GPT language model         | -                     | Magento AI Agent (ai_languageModel input) |                                |
| Magento AI Agent 👩🏻‍🏫 | Langchain Agent                | AI agent specialized for Magento   | Route Chat Input, Transcribe audio, OpenAI Chat Model | If                   |                                |
| If                    | If                            | Branch based on AI agent output     | Magento AI Agent       | If2, Execute a command |                                |
| If2                   | If                            | Additional output condition check   | If                    | Send message1, Send Error Message |                                |
| Execute a command     | SSH                           | Executes commands on Magento server | If                    | If1                   |                                |
| If1                   | If                            | Checks command execution success    | Execute a command      | Send message, Send Output |                                |
| Send message          | WhatsApp                      | Sends WhatsApp success messages     | If1                    | -                     |                                |
| Send message1         | WhatsApp                      | Sends WhatsApp error messages       | If2                    | -                     |                                |
| Send Output           | Telegram                      | Sends Telegram success messages     | If1                    | -                     |                                |
| Send Error Message    | Telegram                      | Sends Telegram error messages       | If2                    | -                     |                                |
| Sticky Note6          | Sticky Note                   | Blank                              | -                     | -                     |                                |
| Sticky Note           | Sticky Note                   | Blank                              | -                     | -                     |                                |
| Sticky Note1          | Sticky Note                   | Blank                              | -                     | -                     |                                |
| Sticky Note2          | Sticky Note                   | Blank                              | -                     | -                     |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook to receive messages (ensure bot token and webhook URL setup).  
   - No parameters needed beyond default for incoming messages.  

2. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure webhook with WhatsApp API credentials.  
   - Ensure media receiving permissions are enabled.

3. **Create Route Chat Input (Switch) Node**  
   - Type: Switch  
   - Configure conditions to detect message type:  
     - Text messages → Magento AI Agent  
     - Audio messages (voice notes) → Get audio file (Telegram) or Download media (WhatsApp)  
     - Media messages → Download media (WhatsApp)  
   - Connect Telegram Trigger and WhatsApp Trigger outputs to this node.

4. **Create Get audio file Node (Telegram)**  
   - Type: Telegram  
   - Configure to fetch audio files from incoming Telegram messages.  
   - Connect Route Chat Input audio/audio-file output here.

5. **Create Download media Node (WhatsApp)**  
   - Type: WhatsApp  
   - Configure to download media files from WhatsApp messages.  
   - Connect Route Chat Input media output here.

6. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Configure GET request to download media content using URL from Download media node.  
   - Connect Download media output here.

7. **Create Transcribe audio Node**  
   - Type: OpenAI (Langchain)  
   - Use OpenAI API with speech-to-text capabilities.  
   - Input: Audio files from Get audio file or HTTP Request nodes.  
   - Output: Transcribed text.  
   - Connect both Get audio file and HTTP Request outputs here.

8. **Create OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Configure with OpenAI credentials (API key).  
   - Leave parameters default or adjust temperature as needed.  
   - Connect this node as the language model input for the AI Agent node.

9. **Create Magento AI Agent Node**  
   - Type: Langchain Agent  
   - Configure prompts and context to specialize in Magento maintenance commands.  
   - Input: Text from Route Chat Input (text) or Transcribe audio node.  
   - Link OpenAI Chat Model node as AI language model source.  
   - Output: AI-generated command or response.

10. **Create If Node (decision after AI Agent)**  
    - Type: If  
    - Condition: Check if AI Agent output is valid command or requires execution.  
    - True branch: Connect to Execute a command node.  
    - False branch: Connect to error handling nodes.

11. **Create Execute a command Node (SSH)**  
    - Type: SSH  
    - Configure SSH credentials for Magento server: hostname, username, auth method.  
    - Input: Command from AI Agent node.  
    - Output: Execution result.

12. **Create If1 Node (check execution success)**  
    - Type: If  
    - Condition: Check success status or exit code from SSH node.  
    - True branch: Connect to Send message (WhatsApp) and Send Output (Telegram).  
    - False branch: Connect to Send Error Message node.

13. **Create Send Output Node (Telegram)**  
    - Type: Telegram  
    - Configure to send messages back to Telegram user using webhook ID.  
    - Input: Execution success message.

14. **Create Send message Node (WhatsApp)**  
    - Type: WhatsApp  
    - Configure to send success messages to WhatsApp user.  
    - Input: Execution success message.

15. **Create Send message1 Node (WhatsApp)**  
    - Type: WhatsApp  
    - Configure to send error or failure messages to WhatsApp user.  
    - Input: Error messages from If2 node.

16. **Create Send Error Message Node (Telegram)**  
    - Type: Telegram  
    - Configure to send error messages back to Telegram user.  
    - Input: Error messages from If2 node.

17. **Create If2 Node (additional condition)**  
    - Type: If  
    - Condition: Further checks on AI Agent or execution output to route success or error messages.  
    - Connect True branch to Send message1 (WhatsApp), False branch to Send Error Message (Telegram).

18. **Link all nodes as per connections described in the Summary Table and Workflow Overview.**

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                 |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| The workflow integrates Magento maintenance commands with natural language processing via GPT and multi-channel chat. | Project Description                             |
| Uses Langchain nodes for advanced AI agent orchestration and OpenAI GPT models.                                        | n8n Langchain Documentation                     |
| Telegram and WhatsApp channels are supported for user interaction, including audio message transcription.              | Telegram Bot API and WhatsApp Business API docs |
| SSH node requires secure credential setup to connect to Magento server.                                               | n8n SSH Node Credential Setup                    |
| Audio transcription depends on OpenAI Whisper or similar speech-to-text via Langchain OpenAI node.                     | OpenAI Whisper API                               |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected information. All processed data is legal and public.