Voice-controlled Expense Tracker with Siri, AI and Google Sheets

https://n8nworkflows.xyz/workflows/voice-controlled-expense-tracker-with-siri--ai-and-google-sheets-6033


# Voice-controlled Expense Tracker with Siri, AI and Google Sheets

---

## 1. Workflow Overview

This n8n workflow, titled **"Voice-controlled Expense Tracker with Siri, AI and Google Sheets"**, serves as a personal finance assistant that enables users to manage their expenses and incomes via voice input through Siri. It leverages AI to interpret natural language commands in Hong Kong Chinese, deciding whether to record a new transaction or retrieve historical spending data. The workflow integrates with Google Sheets to store and read financial records, providing users with concise, human-readable feedback.

The workflow logic is organized into the following key blocks:

- **1.1 Input Reception and Formatting:** Receives user voice input via webhook, extracts and formats essential information such as raw text and timestamps.
- **1.2 AI Processing with LangChain Agent:** Uses a LangChain AI agent to analyze user input, determine the intent (record or read), and generate structured payloads for the corresponding tools.
- **1.3 Data Storage and Retrieval:** Interfaces with Google Sheets to append new expense/income records or read historical data based on AI instructions.
- **1.4 Output Formatting and Response:** Cleans AI output for readability and responds back to the user via the webhook.
- **1.5 Memory Management:** Maintains conversational context to improve AI understanding using a sliding window memory buffer.
- **1.6 AI Language Model Configuration:** Uses OpenRouter Gemini model as backend for AI agent’s language understanding.
- **1.7 Documentation and Setup Notes:** Contains a comprehensive sticky note with setup instructions, usage examples, and iOS shortcut integration details.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Formatting

**Overview:**  
This block receives the user's voice command as JSON via a webhook and formats the input for further AI processing by extracting the raw input text and generating timestamp data.

**Nodes Involved:**  
- Recieve (Webhook)  
- FormatInput (Code)

**Node Details:**  

- **Recieve**  
  - Type: Webhook (n8n-nodes-base.webhook)  
  - Role: Entry point accepting POST requests containing user voice input.  
  - Configuration: HTTP POST method, path set to a unique webhook ID. Response mode configured to respond via the Respond node.  
  - Inputs: External HTTP POST requests with JSON payload containing "text".  
  - Outputs: Passes raw request data to FormatInput node.  
  - Edge Cases: Missing or malformed input JSON, HTTP errors, unauthorized access if webhook is public without token protection.

- **FormatInput**  
  - Type: Code (JavaScript)  
  - Role: Extracts "input" field from webhook body, adds current ISO timestamp, date, and time strings.  
  - Configuration: Custom JavaScript code extracting `$json.body.input` and generating ISO timestamps and date/time strings.  
  - Key Variables: `raw_input` (user text), `formatted_time` (ISO timestamp), `date` (YYYY-MM-DD), `time` (HH:mm:ss).  
  - Inputs: Raw webhook JSON data.  
  - Outputs: JSON object with keys: raw_input, formatted_time, date, time.  
  - Edge Cases: Missing `body.input` field, time zone differences affecting timestamp accuracy.

---

### 2.2 AI Processing with LangChain Agent

**Overview:**  
The AI Agent analyzes the user’s raw input to decide whether the user wants to record a new transaction or request a spending summary. It formats the output to instruct the appropriate tool (Append or Read) and ensures responses are in Hong Kong Chinese.

**Nodes Involved:**  
- Simple Memory (LangChain memory buffer)  
- OpenRouter Chat Model (Language Model)  
- AI Agent (LangChain Agent)

**Node Details:**  

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window (memoryBufferWindow)  
  - Role: Maintains a sliding window of the last 3 conversational turns keyed by the raw input string to provide context to the AI Agent.  
  - Configuration: Session key is set to the current raw input, window length 3.  
  - Inputs: Raw user input for session key.  
  - Outputs: Contextual memory data to AI Agent.  
  - Edge Cases: Memory overflow if session keys are not unique or misconfigured.

- **OpenRouter Chat Model**  
  - Type: LangChain Language Model (lmChatOpenRouter)  
  - Role: Provides AI language understanding using the Gemini 2.0 Flash Lite model from OpenRouter.  
  - Configuration: Model set to "google/gemini-2.0-flash-lite-001", max tokens unlimited (-1).  
  - Credentials: OpenRouter API key configured.  
  - Inputs: AI Agent’s prompt messages.  
  - Outputs: AI-generated completions to the AI Agent node.  
  - Edge Cases: API quota limits, network timeouts, invalid credentials.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Core AI logic node that receives the formatted raw input and memory context, runs a system prompt defining its role as finance assistant, and decides the workflow action (Append or Read).  
  - Configuration:  
    - Takes `$json.raw_input` as user input text.  
    - System message instructs the agent to analyze the input, choose tool calls, infer missing fields, and respond in Hong Kong Chinese.  
    - Defines two tools: Append (for recording) and Read (for reading data).  
    - Enforces reply formatting and rules for human-readable messages.  
  - Inputs: Formatted input from FormatInput, memory context from Simple Memory, and language model outputs.  
  - Outputs: JSON payloads for Append or Read tools with transaction details or query parameters.  
  - Edge Cases: Ambiguous input causing wrong tool selection, AI response delays or errors, incomplete inference leading to wrong data.

---

### 2.3 Data Storage and Retrieval

**Overview:**  
This block interacts with Google Sheets to either append new financial records or read past spending data depending on AI Agent instructions.

**Nodes Involved:**  
- Append (Google Sheets Tool)  
- Read (Google Sheets Tool)

**Node Details:**  

- **Append**  
  - Type: Google Sheets Tool (n8n-nodes-base.googleSheetsTool)  
  - Role: Appends a new row to the designated Google Sheet with transaction data (date, type, name, amount, created time, expense/income).  
  - Configuration:  
    - Operation: Append  
    - Document ID and Sheet Name point to a specific Google Sheet and tab for overall expenses.  
    - Columns mapped from AI Agent output fields using `$fromAI()` helper for Date, Name, Type, Amount, expenses/incomes, and current time `$now`.  
  - Credentials: Google OAuth2 configured with "Angus Account".  
  - Inputs: JSON from AI Agent for new transaction.  
  - Outputs: Confirmation or success status passed to the output formatting.  
  - Edge Cases: Google API quota limits, authentication expiration, schema mismatches in sheet columns, empty or invalid values.

- **Read**  
  - Type: Google Sheets Tool  
  - Role: Reads and summarizes spending records from the Google Sheet based on parameters sent by AI Agent (e.g., time range, category).  
  - Configuration:  
    - Operation: Read  
    - Document ID and Sheet Name same as Append node.  
    - Data location set to auto-detect on sheet.  
  - Credentials: Same Google OAuth2.  
  - Inputs: Query parameters from AI Agent requesting spending summaries.  
  - Outputs: Data rows sent back for AI to create summary messages.  
  - Edge Cases: Large data causing timeouts, malformed queries, sheet structure changes.

---

### 2.4 Output Formatting and Response

**Overview:**  
Formats the AI-generated output message to remove unwanted line breaks and characters before sending it back to the user via the webhook response.

**Nodes Involved:**  
- FormatOutput (Code)  
- Respond (Respond to Webhook)

**Node Details:**  

- **FormatOutput**  
  - Type: Code (JavaScript)  
  - Role: Cleans the AI output string by removing all newline characters to ensure compatibility with iOS and Siri speech output.  
  - Configuration:  
    - JavaScript code iterates over all items, replaces `\n` with empty string, and joins outputs.  
    - Returns a JSON object with key "希希" containing the cleaned reply string.  
  - Inputs: AI Agent output JSON with key `output`.  
  - Outputs: Formatted JSON response.  
  - Edge Cases: Missing or empty AI output, encoding issues.

- **Respond**  
  - Type: Respond to Webhook  
  - Role: Sends the final cleaned reply back as HTTP response to the incoming webhook call from Siri Shortcut.  
  - Configuration: Responds with all incoming items, effectively sending the formatted JSON reply.  
  - Inputs: Formatted output from FormatOutput.  
  - Outputs: HTTP response to caller.  
  - Edge Cases: Timeout if response delayed, network errors.

---

### 2.5 Documentation and Setup Notes

**Overview:**  
A large sticky note node containing detailed setup instructions, usage examples, iOS shortcut configuration steps, tips, and links for ease of adoption.

**Nodes Involved:**  
- Sticky Note

**Node Details:**  

- **Sticky Note**  
  - Type: Sticky Note (n8n-nodes-base.stickyNote)  
  - Role: Provides comprehensive documentation within the workflow canvas for users to understand setup and usage.  
  - Content Highlights:  
    - Download links for Siri Shortcut and Google Drive API setup.  
    - Step-by-step instructions for n8n webhook creation, AI Agent setup, Google Sheets integration, and output formatting.  
    - iOS Shortcut setup guide including HTTP POST request configuration.  
    - Example user interactions and expected Siri responses.  
    - Tips for newcomers to handle emojis, logging, and security.  
  - Position: Top-left corner for easy reference.  
  - Edge Cases: None (informational only).

---

## 3. Summary Table

| Node Name           | Node Type                           | Functional Role                 | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                             |
|---------------------|-----------------------------------|--------------------------------|---------------------|---------------------|-------------------------------------------------------------------------------------------------------|
| Recieve             | Webhook                           | Input reception via HTTP POST  | —                   | FormatInput         | Part of initial input reception and formatting flow.                                                  |
| FormatInput         | Code                              | Extracts input & timestamps    | Recieve             | AI Agent            | Parses incoming webhook JSON to structured input for AI.                                             |
| Simple Memory       | LangChain Memory Buffer Window    | Context memory for AI agent    | —                   | AI Agent            | Maintains last 3 inputs context to improve AI interpretation.                                        |
| OpenRouter Chat Model | LangChain Language Model         | Provides AI language model     | AI Agent (ai_languageModel) | AI Agent         | Uses Gemini 2.0 model for text understanding.                                                        |
| AI Agent            | LangChain Agent                   | Core AI logic & decision making| FormatInput, Simple Memory, OpenRouter Chat Model | Append, Read, FormatOutput | Analyzes input to decide action and generate payloads. Outputs replies in Hong Kong Chinese.          |
| Append              | Google Sheets Tool                | Append new expense/income data | AI Agent            | —                   | Adds new financial records to Google Sheets.                                                         |
| Read                | Google Sheets Tool                | Retrieve & summarize expenses  | AI Agent            | —                   | Reads spending data from Google Sheets based on AI query.                                            |
| FormatOutput        | Code                             | Cleans AI output for response  | AI Agent            | Respond             | Removes newlines for iOS compatibility in response.                                                  |
| Respond             | Respond to Webhook                | Sends final reply to user      | FormatOutput        | —                   | Responds back to Siri Shortcut HTTP request with final message.                                      |
| Sticky Note         | Sticky Note                      | Documentation & instructions   | —                   | —                   | Contains detailed setup and usage instructions, iOS Shortcut info, example conversations, and tips. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("Recieve")**  
   - Node Type: Webhook (n8n-nodes-base.webhook)  
   - Set HTTP Method: POST  
   - Set Path: Unique path (e.g., "siri-finance")  
   - Enable "Respond to Webhook" checked  
   - No special credentials needed  
   - Connect this node as the workflow entry point

2. **Add Code Node ("FormatInput")**  
   - Node Type: Code  
   - Paste JavaScript code to extract `body.input` from webhook JSON and add current ISO timestamp, date, and time strings:  
     ```js
     const body = $json.body || {};
     const rawInput = body.input || '';
     const now = new Date();

     return [
       {
         json: {
           raw_input: rawInput,
           formatted_time: now.toISOString(),
           date: now.toISOString().split('T')[0],
           time: now.toTimeString().split(' ')[0],
         }
       }
     ];
     ```  
   - Connect "Recieve" output to this node input

3. **Add LangChain Memory Node ("Simple Memory")**  
   - Node Type: LangChain Memory Buffer Window  
   - Set Session Key: `={{ $json.raw_input }}`  
   - Session ID Type: Custom Key  
   - Context Window Length: 3  
   - Connect from wherever needed for context (input to AI Agent)

4. **Add LangChain Language Model Node ("OpenRouter Chat Model")**  
   - Node Type: LangChain Language Model (lmChatOpenRouter)  
   - Model: "google/gemini-2.0-flash-lite-001"  
   - Options: Max Tokens = -1 (unlimited)  
   - Set credentials: Configure OpenRouter API with valid API key

5. **Add LangChain Agent Node ("AI Agent")**  
   - Node Type: LangChain Agent  
   - Input Text: `={{ $json.raw_input }}`  
   - System Prompt: Use the detailed system message that instructs the AI to decide between Append and Read tools, infer missing info, and respond in Hong Kong Chinese as per the provided system message content.  
   - Tools: Define two tools:  
     - Append: For recording expenses (linked to Append node)  
     - Read: For reading summaries (linked to Read node)  
   - Connect inputs from "FormatInput", "Simple Memory", and "OpenRouter Chat Model" nodes  
   - Connect outputs to "Append", "Read", and "FormatOutput" nodes

6. **Add Google Sheets Append Node ("Append")**  
   - Node Type: Google Sheets Tool  
   - Operation: Append  
   - Document ID: Your Google Sheet ID (e.g., "1uZik4myIt4XHGs5fpv6ZEDdczVyaOpMe3vLmtLCy0Zc")  
   - Sheet Name: Tab ID or name (e.g., "overall")  
   - Columns mapping: Use `$fromAI()` expressions to map AI Agent output fields:  
     - Date, Name, Type, Amount, expenses/incomes, created time (use `$now` for current timestamp)  
   - Credentials: Google OAuth2 with appropriate Google account  
   - Connect AI Agent output to this node

7. **Add Google Sheets Read Node ("Read")**  
   - Node Type: Google Sheets Tool  
   - Operation: Read  
   - Document ID & Sheet Name: Same as Append node  
   - Data location: Automatic detection of range  
   - Credentials: Same Google OAuth2  
   - Connect AI Agent output to this node

8. **Add Code Node ("FormatOutput")**  
   - Node Type: Code  
   - JavaScript code to clean AI response by removing line breaks:  
     ```js
     const outputs = items.map(item => {
       const output = item.json.output || '';
       return output.replace(/\n/g, '');
     });
     return [
       {
         json: {
           希希: outputs.join('')
         }
       }
     ];
     ```  
   - Connect AI Agent output to this node

9. **Add Respond to Webhook Node ("Respond")**  
   - Node Type: Respond to Webhook  
   - Set to respond with all incoming items (default)  
   - Connect from "FormatOutput" node

10. **Connections Summary:**  
    - Recieve → FormatInput → AI Agent → (Append or Read)  
    - Simple Memory and OpenRouter Chat Model connect as inputs to AI Agent  
    - AI Agent → FormatOutput → Respond

11. **Optional: Add Sticky Note for Documentation**  
    - Include setup instructions, iOS shortcut links, usage examples, and tips as per the original sticky note content.

12. **iOS Shortcut Setup (External to n8n):**  
    - Create Shortcut named "記帳助理" or "Finance Bot"  
    - Add Action: Ask for Input (Text) with prompt "請說出你的記帳內容"  
    - Add Action: Get Contents of URL  
      - Method: POST  
      - URL: Your n8n webhook URL (e.g., https://your-n8n-domain/webhook/siri-finance)  
      - Headers: Content-Type: application/json  
      - Request Body: `{ "text": "Provided Input" }` (replace with Magic Variable for input)  
    - Add Action: Show Result with content from Get Contents of URL  
    - Optional: Add Speak Text action to speak the response aloud

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Siri AI 2.0 (Finance Assistant Version) workflow enables voice-operated expense tracking with natural language understanding in Hong Kong Chinese. It integrates Siri Shortcuts with n8n and Google Sheets for seamless user experience.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow description and branding.                                                                                                                                           |
| Shortcut download link: https://www.icloud.com/shortcuts/9848032ea36c434bbdc8cf9631309a81                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | iOS Shortcut to be imported for voice input.                                                                                                                                 |
| Google Sheets document used: https://docs.google.com/spreadsheets/d/1uZik4myIt4XHGs5fpv6ZEDdczVyaOpMe3vLmtLCy0Zc/edit#gid=1478323734                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Google Sheets template to store and manage expense data.                                                                                                                     |
| Tips: Keep webhook public but secure with tokens if necessary. Handle emojis and newline characters carefully for iOS compatibility. Use logging nodes in n8n for debugging Siri messages.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Best practices for deployment and maintenance.                                                                                                                               |
| Example user utterances and AI responses: "我頭先食麥當勞用了52蚊" → Records expense; "幫我查過去一星期的開支" → Returns spending summary.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Usage examples for testing and validation.                                                                                                                                   |

---

*Disclaimer: The provided text is solely derived from an automated workflow created with n8n, respecting all applicable content policies and containing no illegal, offensive, or protected elements. All data processed is legal and public.*