Capture and Organize Ideas via Telegram with GPT-4 & Google Sheets

https://n8nworkflows.xyz/workflows/capture-and-organize-ideas-via-telegram-with-gpt-4---google-sheets-8377


# Capture and Organize Ideas via Telegram with GPT-4 & Google Sheets

### 1. Workflow Overview

This workflow captures and organizes user ideas submitted via Telegram, either as text messages or voice notes, leveraging AI to analyze and categorize each idea before storing it in a structured Google Sheets database. It is designed for users who want a seamless, intelligent ideation capture system that works on the go via Telegram.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Listens for incoming Telegram messages (text or voice) and routes them accordingly.
- **1.2 Voice Transcription:** Converts voice notes to text using ElevenLabs speech-to-text API.
- **1.3 Idea Preparation:** Prepares and formats the idea text for AI processing.
- **1.4 AI Processing:** Uses an AI Agent powered by GPT-4 via Azure OpenAI to analyze, categorize, and structure the idea into predefined fields.
- **1.5 Data Storage:** Saves the structured idea as a new row in a Google Sheets spreadsheet.
- **1.6 Confirmation:** Sends a confirmation message back to the user on Telegram summarizing the captured idea.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures incoming Telegram messages and identifies whether the message contains text or a voice note, directing each to the appropriate processing path.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Switch

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Entry point listening for new Telegram messages (updates of type "message").  
    - Configuration: Monitors all incoming messages to the bot.  
    - Inputs: Telegram user messages (text or voice).  
    - Outputs: Passes message JSON to the Switch node.  
    - Edge Cases: Telegram API rate limits, webhook URL misconfiguration, unsupported message types.  

  - **Switch**  
    - Type: Switch node  
    - Role: Routes messages based on content type ("text" or "audio").  
    - Configuration: Checks if the JSON string of the message contains "text" or "audio".  
    - Inputs: Output from Telegram Trigger.  
    - Outputs:  
      - If text message: routes to "Edit Fields" node.  
      - If audio message: routes to "Telegram1" node for voice processing.  
    - Edge Cases: Messages with neither text nor audio (e.g., stickers, photos) are not handled and thus dropped. Expression failures if message JSON structure changes.

---

#### 2.2 Voice Transcription

- **Overview:**  
  Processes audio messages by downloading the voice file from Telegram and sending it to ElevenLabs API to transcribe speech to text.

- **Nodes Involved:**  
  - Telegram1  
  - HTTP Request  
  - Edit Fields

- **Node Details:**

  - **Telegram1**  
    - Type: Telegram node (file resource)  
    - Role: Downloads the voice message file using the file_id from Telegram message.  
    - Configuration: Uses the voice file_id from incoming audio message JSON.  
    - Inputs: Audio message from Switch.  
    - Outputs: The binary voice file data passed to HTTP Request.  
    - Edge Cases: File might be unavailable if deleted or expired; Telegram API errors.

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Role: Sends the voice file to ElevenLabs speech-to-text API to get a transcription.  
    - Configuration:  
      - POST method to `https://api.elevenlabs.io/v1/speech-to-text`  
      - Multipart/form-data content type  
      - Sends `model_id` as "scribe_v1" and the voice file as form binary data  
      - Authentication via HTTP header credentials (ElevenLabs API key).  
    - Inputs: Binary file from Telegram1.  
    - Outputs: JSON with transcription result.  
    - Edge Cases: API key invalid/missing, network errors, transcription failures, rate limits.

  - **Edit Fields**  
    - Type: Set node  
    - Role: Constructs a unified text prompt combining the transcribed text and any other relevant fields to feed into the AI Agent.  
    - Configuration: Sets a variable named "Prompt" by concatenating transcribed text, original message text (if any), and AI-generated candidate text.  
    - Inputs: Transcription result from HTTP Request or text message from Switch.  
    - Outputs: Prepared prompt JSON to AI Agent node.  
    - Edge Cases: Missing transcription text, expression errors.

---

#### 2.3 AI Processing

- **Overview:**  
  Uses a GPT-4 powered AI Agent to analyze the unified text prompt, extract structured idea data, and prepare it for storage.

- **Nodes Involved:**  
  - Azure OpenAI Chat Model  
  - MCP Client  
  - Agent

- **Node Details:**

  - **Azure OpenAI Chat Model**  
    - Type: LangChain Azure OpenAI Chat Model node  
    - Role: Connects to Azure OpenAI GPT-4.1-2 model to generate AI completions based on prompts.  
    - Configuration: Uses GPT-4.1-2 model without additional options configured.  
    - Inputs: Prompt from Edit Fields.  
    - Outputs: AI-generated completion text.  
    - Edge Cases: API authentication errors, model unavailability, timeout, quota limits.  

  - **MCP Client**  
    - Type: LangChain MCP Client Tool node  
    - Role: Acts as a client interface for the multi-chain processing (MCP) setup, passing AI outputs to the Agent node.  
    - Inputs: AI completions from Azure OpenAI Chat Model.  
    - Outputs: AI Agent node.  
    - Edge Cases: Internal communication failures between MCP client and server.

  - **Agent**  
    - Type: LangChain Agent node  
    - Role: Processes the AI prompt with a detailed system message instructing it to extract and format idea attributes precisely, then uses the Google Sheets tool to save the idea.  
    - Configuration:  
      - Max iterations: 50  
      - System Message: Detailed instructions specifying required fields (Idea, Description, Type, Score, Category, Priority, Status, Complexity) and exact expected values.  
      - Prompt: Uses the "Prompt" variable from Edit Fields.  
    - Inputs: Prepared prompt from MCP Client or Edit Fields.  
    - Outputs: Confirmation text output to Telegram node.  
    - Edge Cases: Misformatted AI responses, exceeding max iterations, failures invoking tools, inconsistent field extraction.

---

#### 2.4 Data Storage

- **Overview:**  
  The AI Agent uses the Google Sheets Tool to append the structured idea as a new row into a preconfigured Google Sheets spreadsheet.

- **Nodes Involved:**  
  - add_row_tool1  
  - MCP Server Trigger

- **Node Details:**

  - **add_row_tool1**  
    - Type: Google Sheets Tool node  
    - Role: Appends a new row with fields extracted by the AI Agent into the "Ideation AI Agent - DB" Google Sheet.  
    - Configuration:  
      - Operation: Append  
      - Document ID and Sheet Name set to specific spreadsheet and sheet ("Sheet1" with gid=0).  
      - Columns mapped according to AI output fields: Idea, Status, Idea Type, Idea Score, Complexity, Priority Level, Category/Domain, Idea Description.  
    - Inputs: AI Agent’s tool call.  
    - Outputs: Sends success confirmation to MCP Server Trigger node.  
    - Edge Cases: Google Sheets API credential errors, quota limits, incorrect mapping, partial data.

  - **MCP Server Trigger**  
    - Type: LangChain MCP Server Trigger node  
    - Role: Receives callbacks from Google Sheets tool invoked by the Agent, allowing continuation of the Agent flow.  
    - Inputs: Success callback from add_row_tool1.  
    - Outputs: Back to Agent node’s ai_tool input.  
    - Edge Cases: Communication failures, webhook misconfigurations.

---

#### 2.5 Confirmation

- **Overview:**  
  Sends a confirmation message back to the Telegram user summarizing the captured idea details.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - Type: Telegram node  
    - Role: Sends a message back to the chat confirming successful idea capture.  
    - Configuration:  
      - Text: Uses AI Agent’s output text (confirmation message).  
      - Chat ID: Extracted dynamically from the original Telegram Trigger message.  
      - Does not append attribution.  
    - Inputs: Output from Agent node.  
    - Outputs: None (end of workflow).  
    - Edge Cases: Telegram API errors, chat ID missing or invalid.

---

### 3. Summary Table

| Node Name            | Node Type                             | Functional Role                       | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                  |
|----------------------|-------------------------------------|------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note          | Sticky Note                         | Documentation                      |                             |                             | ## 🤖 AI Idea Capture Agent: Explains overall workflow purpose and use cases                  |
| Sticky Note4         | Sticky Note                         | Documentation                      |                             |                             | How it works: detailed step-by-step workflow explanation                                    |
| Sticky Note5         | Sticky Note                         | Documentation                      |                             |                             | Requirements and customization notes                                                        |
| Sticky Note1         | Sticky Note                         | Documentation                      |                             |                             | ✅ Listening for Ideas                                                                       |
| Sticky Note2         | Sticky Note                         | Documentation                      |                             |                             | ✅ Prepare the Idea                                                                         |
| Sticky Note3         | Sticky Note                         | Documentation                      |                             |                             | ✅ Save the Idea                                                                           |
| Telegram Trigger     | Telegram Trigger                    | Input Reception (entry point)       |                             | Switch                      |                                                                                              |
| Switch               | Switch                             | Input routing (text vs audio)       | Telegram Trigger            | Edit Fields, Telegram1       |                                                                                              |
| Telegram1            | Telegram (file resource)            | Download voice note file             | Switch                     | HTTP Request                |                                                                                              |
| HTTP Request         | HTTP Request                       | Voice transcription via ElevenLabs | Telegram1                  | Edit Fields                 |                                                                                              |
| Edit Fields          | Set                                | Prepare unified prompt text          | Switch, HTTP Request       | Agent                      |                                                                                              |
| Azure OpenAI Chat Model | LangChain Azure OpenAI Chat Model | Generate AI completion from prompt  | Edit Fields                | MCP Client                 |                                                                                              |
| MCP Client           | LangChain MCP Client Tool           | Pass AI output to Agent node         | Azure OpenAI Chat Model    | Agent                      |                                                                                              |
| Agent                | LangChain Agent                    | Analyze and structure idea; invoke Google Sheets | MCP Client, Edit Fields | Telegram, Google Sheets Tool |                                                                                              |
| add_row_tool1        | Google Sheets Tool                 | Append idea data as new row           | Agent                      | MCP Server Trigger         |                                                                                              |
| MCP Server Trigger   | LangChain MCP Server Trigger       | Receive callbacks from Google Sheets | add_row_tool1              | Agent                      |                                                                                              |
| Telegram             | Telegram node                     | Send confirmation message to user    | Agent                      |                             |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot & Obtain API Token**
   - Use BotFather on Telegram to create a bot.
   - Copy the API token.

2. **Add Telegram Trigger Node**
   - Node Type: Telegram Trigger
   - Configure with your Telegram Bot API token credential.
   - Set "Updates" to listen for "message" events.

3. **Add Switch Node**
   - Node Type: Switch
   - Connect input from Telegram Trigger.
   - Add two rules to check message type:
     - Rule 1: If the message JSON string contains "text" (case sensitive).
     - Rule 2: If the message JSON string contains "audio" (case sensitive).
   - Connect Rule 1 output to "Edit Fields" node.
   - Connect Rule 2 output to "Telegram1" node.

4. **Add Telegram Node (Telegram1) to Download Voice File**
   - Node Type: Telegram
   - Connect input from Switch Rule 2.
   - Set resource to "file".
   - Set fileId to `{{$json.message.voice.file_id}}`.
   - Use Telegram API credentials.

5. **Add HTTP Request Node for ElevenLabs API**
   - Node Type: HTTP Request
   - Connect input from Telegram1.
   - Set method to POST.
   - URL: `https://api.elevenlabs.io/v1/speech-to-text`.
   - Content-Type: multipart/form-data.
   - Body Parameters:
     - model_id = "scribe_v1"
     - file = binary data from Telegram1 node (inputDataFieldName: "data").
   - Authentication: HTTP Header with ElevenLabs API key credential.

6. **Add Set Node (Edit Fields)**
   - Node Type: Set
   - Connect inputs from:
     - Switch Rule 1 (text messages)
     - HTTP Request (voice transcription)
   - Add a field named "Prompt" of type string.
   - Configure the value to concatenate:
     - For text messages: `{{$json.message.text}}`
     - For voice transcription: use transcribed text from ElevenLabs API response.
     - Optionally append AI candidate text if available.
   - This standardizes prompt input for AI.

7. **Add Azure OpenAI Chat Model Node**
   - Node Type: LangChain Azure OpenAI Chat Model
   - Connect input from Edit Fields.
   - Configure credentials for Azure OpenAI.
   - Model: `gpt-4.1-2`.
   - No additional options configured.

8. **Add MCP Client Node**
   - Node Type: LangChain MCP Client Tool
   - Connect input from Azure OpenAI Chat Model node's output.
   - Pass data to Agent node.

9. **Add Agent Node**
   - Node Type: LangChain Agent
   - Connect main input from Edit Fields (text prompt).
   - Connect ai_tool input from MCP Client.
   - Configure with systemMessage to:
     - Instruct the AI to analyze the prompt.
     - Extract fields: Idea, Idea Description, Idea Type, Idea Score, Category/Domain, Priority Level, Status, Complexity Level.
     - Use exact predefined options for categorical fields.
     - After successful data extraction, invoke Google Sheets Tool to append data.
     - Confirm with text message format: "✅ Idea captured: [Brief title] - [Type] - Score: [X/10] - Priority: [Level]".
   - Set maxIterations to 50.

10. **Add Google Sheets Tool Node (add_row_tool1)**
    - Node Type: Google Sheets Tool
    - Connect input from Agent node's ai_tool output.
    - Configure operation as "Append".
    - Set Document ID to your Google Sheets ID (e.g., `1xMuQ0-fT9M3_-LFyvyvuCJnXTrqZsfLKqEeUU9u1v7U`).
    - Set Sheet Name to "Sheet1" or equivalent.
    - Define columns with exact field names matching AI output:
      - Idea, Idea Description, Idea Type, Idea Score, Category/Domain, Priority Level, Status, Complexity.
    - Authenticate with Google Sheets API credentials.

11. **Add MCP Server Trigger Node**
    - Node Type: LangChain MCP Server Trigger
    - Connect input from Google Sheets Tool node's `ai_tool` output.
    - Configure webhook path uniquely.
    - Pass output back to Agent node's ai_tool input.

12. **Add Telegram Node for Confirmation**
    - Node Type: Telegram
    - Connect input from Agent node's main output.
    - Configure to send message text from Agent output.
    - Set chatId dynamically using `{{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}`.
    - Disable "Append Attribution".

13. **Test Workflow**
    - Send text and voice messages to Telegram bot.
    - Verify ideas are transcribed, analyzed, and saved.
    - Confirm Telegram confirmation messages received.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| This workflow creates an intelligent Ideation Agent that captures ideas via Telegram text or voice notes, transcribes audio, analyzes content with GPT-4, and organizes ideas into Google Sheets for easy management.               | Overview Sticky Note                                                                                                          |
| Multi-modal input handling supports both text and voice, enabling versatile idea capture anytime, anywhere.                                                                                                                       | How it works Sticky Note                                                                                                      |
| Requires Telegram Bot API token, Azure OpenAI API key, ElevenLabs API key for voice transcription, and Google Sheets API credentials.                                                                                              | Requirements Sticky Note                                                                                                      |
| Customize idea categories, priority levels, and other fields by editing the Agent node’s system prompt.                                                                                                                            | Customization instructions Sticky Note                                                                                        |
| Swap AI model or backend by replacing the Azure OpenAI Chat Model node with another LLM node if needed.                                                                                                                            | Customization instructions Sticky Note                                                                                        |
| Change data storage by substituting the Google Sheets Tool node with another database or SaaS integration like Notion or Airtable.                                                                                                 | Customization instructions Sticky Note                                                                                        |
| See https://elevenlabs.io/ for API documentation on speech-to-text capabilities.                                                                                                                                                    | ElevenLabs API reference                                                                                                      |
| See https://learn.microsoft.com/en-us/azure/cognitive-services/openai/ for Azure OpenAI usage.                                                                                                                                     | Azure OpenAI API documentation                                                                                                |
| Telegram Bot API documentation: https://core.telegram.org/bots/api                                                                                                                                                                 | Telegram API documentation                                                                                                   |
| Google Sheets API information: https://developers.google.com/sheets/api                                                                                                                                                            | Google Sheets API documentation                                                                                              |

---

**Disclaimer:**  
The provided text and workflow stem exclusively from an automated n8n workflow built with official APIs and tools. It complies fully with content policies, containing no illegal or protected elements. All data processed is legal and public.