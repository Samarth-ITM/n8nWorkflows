Chat with your event schedule from Google Sheets in Telegram

https://n8nworkflows.xyz/workflows/chat-with-your-event-schedule-from-google-sheets-in-telegram-2937


# Chat with your event schedule from Google Sheets in Telegram

### 1. Workflow Overview

This workflow enables users to interact with a Telegram bot to query an event schedule stored in a Google Spreadsheet using natural language. It is designed for meetup organizers or communities who want to provide easy access to event information such as dates, presenters, and attendance via chat. The workflow integrates Telegram messaging, Google Sheets data retrieval, markdown formatting, and AI language model processing to generate contextual, human-readable responses.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures chat messages either from Telegram or an internal n8n chat trigger.
- **1.2 Input Normalization:** Standardizes incoming chat data into a unified format for downstream processing.
- **1.3 Data Retrieval and Formatting:** Fetches the event schedule from Google Sheets and converts it into a markdown table.
- **1.4 AI Processing:** Uses a language model (OpenRouter) with the schedule context to generate a response.
- **1.5 Response Preparation and Routing:** Prepares the AI-generated response and routes it back to the appropriate chat platform (Telegram or internal).
- **1.6 Telegram Interaction:** Sends typing indicators and posts the final response back to Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming chat messages either from Telegram users or from an internal n8n chat interface for testing/debugging.

**Nodes Involved:**  
- `telegramInput` (Telegram Trigger)  
- `n8nInput` (Langchain Chat Trigger)  
- `SendTyping` (Telegram node to send typing action)

**Node Details:**

- **telegramInput**  
  - Type: Telegram Trigger  
  - Role: Listens for new messages sent to the Telegram bot.  
  - Config: Listens to "message" updates only. Uses Telegram API credentials.  
  - Inputs: External Telegram messages  
  - Outputs: Emits message JSON including chat ID and text.  
  - Edge cases: Telegram API downtime, invalid bot token, message format changes.

- **n8nInput**  
  - Type: Langchain Chat Trigger  
  - Role: Allows triggering the workflow internally for testing or alternative chat input.  
  - Config: Default, no special parameters.  
  - Inputs: Internal chat messages.  
  - Outputs: Emits chat input JSON with session ID and chat text.  
  - Edge cases: Misconfigured internal trigger, missing session ID.

- **SendTyping**  
  - Type: Telegram node (sendChatAction)  
  - Role: Sends "typing..." indicator to Telegram chat to improve UX while processing.  
  - Config: Uses chat ID from `telegramInput` node, operation set to "sendChatAction".  
  - Inputs: Message from `telegramInput`.  
  - Outputs: Confirmation of chat action sent.  
  - Edge cases: Telegram API rate limits, invalid chat ID.

---

#### 2.2 Input Normalization

**Overview:**  
Transforms incoming chat data from either Telegram or internal input into a unified set of variables (`inputMessage`, `chatId`, `mode`) for consistent downstream processing.

**Nodes Involved:**  
- `telegramChatSettings` (Set node)  
- `n8nChatSettings` (Set node)  
- `Settings` (Set node)

**Node Details:**

- **telegramChatSettings**  
  - Type: Set  
  - Role: Extracts and sets `inputMessage`, `chatId`, and `mode` ("telegram") from Telegram input JSON.  
  - Config:  
    - `inputMessage` = text from Telegram message  
    - `chatId` = chat ID from Telegram message  
    - `mode` = "telegram" (string literal)  
  - Inputs: Output from `SendTyping` node  
  - Outputs: Unified JSON with above fields.  
  - Edge cases: Missing message text or chat ID in Telegram payload.

- **n8nChatSettings**  
  - Type: Set  
  - Role: Extracts and sets `inputMessage`, `chatId`, and `mode` ("n8n") from internal chat input.  
  - Config:  
    - `inputMessage` = `chatInput` from internal trigger  
    - `chatId` = `sessionId` from internal trigger  
    - `mode` = "n8n" (string literal)  
  - Inputs: Output from `n8nInput` node  
  - Outputs: Unified JSON with above fields.  
  - Edge cases: Missing or malformed internal chat data.

- **Settings**  
  - Type: Set  
  - Role: Stores global settings including the Google Sheets URL and passes normalized input variables downstream.  
  - Config:  
    - `scheduleURL` = URL of Google Spreadsheet containing event schedule  
    - Passes through `inputMessage`, `chatId`, and `mode` from previous nodes.  
  - Inputs: From either `telegramChatSettings` or `n8nChatSettings`  
  - Outputs: JSON with schedule URL and chat variables.  
  - Edge cases: Invalid or inaccessible Google Sheets URL.

---

#### 2.3 Data Retrieval and Formatting

**Overview:**  
Retrieves the event schedule data from the specified Google Spreadsheet and converts it into a markdown table format to provide context for the AI language model.

**Nodes Involved:**  
- `Schedule` (Google Sheets node)  
- `ScheduleToMarkdown` (Code node)

**Node Details:**

- **Schedule**  
  - Type: Google Sheets  
  - Role: Reads all rows from the first sheet (`gid=0`) of the spreadsheet URL specified in `scheduleURL`.  
  - Config:  
    - Document ID extracted from `scheduleURL`  
    - Sheet name set to `gid=0` (first sheet)  
    - Uses Google Sheets OAuth2 credentials  
  - Inputs: From `Settings` node  
  - Outputs: Array of rows as JSON objects representing event data.  
  - Edge cases: OAuth token expiration, spreadsheet access permissions, empty sheet.

- **ScheduleToMarkdown**  
  - Type: Code (JavaScript)  
  - Role: Converts the array of event rows into a markdown-formatted table string.  
  - Config:  
    - Extracts headers from first row keys  
    - Builds markdown table header, separator, and rows  
    - Returns markdown string in JSON property `markdown`  
  - Inputs: Rows from `Schedule` node  
  - Outputs: JSON with `markdown` property containing the schedule table  
  - Edge cases: Empty data (returns "No data available."), missing values in rows.

---

#### 2.4 AI Processing

**Overview:**  
Feeds the markdown schedule and user query into an AI language model to generate a natural language response answering the user's question about the event schedule.

**Nodes Involved:**  
- `ScheduleBot` (Langchain Agent node)  
- `LLM` (OpenRouter language model node)  
- `Memory` (Langchain memory buffer node)

**Node Details:**

- **ScheduleBot**  
  - Type: Langchain Agent  
  - Role: Orchestrates AI processing by combining user input and schedule context, then generating a response.  
  - Config:  
    - `text` input set to `inputMessage` from `Settings` node  
    - System prompt includes a description of the assistant role and injects the markdown schedule as context  
    - Uses `LLM` node as language model and `Memory` node for conversation memory  
  - Inputs: Markdown schedule from `ScheduleToMarkdown`, user input from `Settings`  
  - Outputs: AI-generated text response in `output` property  
  - Edge cases: AI service downtime, prompt formatting errors, memory session key issues.

- **LLM**  
  - Type: Langchain LM Chat OpenRouter  
  - Role: Provides the language model backend for generating responses.  
  - Config: Uses OpenRouter API credentials  
  - Inputs: From `ScheduleBot` node  
  - Outputs: Text completions  
  - Edge cases: API rate limits, authentication failures.

- **Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains conversation context keyed by chat ID to enable multi-turn dialogue.  
  - Config: Session key set to `chatId` from `Settings` node  
  - Inputs: From `ScheduleBot` node  
  - Outputs: Updated memory context  
  - Edge cases: Memory overflow, session key conflicts.

---

#### 2.5 Response Preparation and Routing

**Overview:**  
Prepares the AI-generated response message and routes it to the appropriate output channel depending on the chat mode (Telegram or internal n8n).

**Nodes Involved:**  
- `SetResponse` (Set node)  
- `Switch` (Switch node)

**Node Details:**

- **SetResponse**  
  - Type: Set  
  - Role: Assigns the AI output text to a variable `responseMessage` for downstream use.  
  - Config: Sets `responseMessage` = AI output text from `ScheduleBot` node  
  - Inputs: AI response from `ScheduleBot`  
  - Outputs: JSON with `responseMessage`  
  - Edge cases: Missing or empty AI output.

- **Switch**  
  - Type: Switch  
  - Role: Routes the workflow based on the `mode` variable (`n8n` or `telegram`) to send the response to the correct channel.  
  - Config:  
    - Checks `mode` from `Settings` node  
    - Outputs:  
      - "n8n mode" → `n8nResponse` node  
      - "telegram mode" → `telegramResponse` node  
  - Inputs: From `SetResponse` node  
  - Outputs: Branches to response nodes  
  - Edge cases: Unexpected mode values.

---

#### 2.6 Telegram Interaction

**Overview:**  
Sends the AI-generated response back to the Telegram chat and manages user experience by sending typing indicators.

**Nodes Involved:**  
- `telegramResponse` (Telegram node)  
- `n8nResponse` (NoOp node)

**Node Details:**

- **telegramResponse**  
  - Type: Telegram  
  - Role: Sends the final response message text back to the Telegram chat identified by `chatId`.  
  - Config:  
    - Text set to `responseMessage` from `SetResponse`  
    - Chat ID from `Settings`  
    - Uses Telegram API credentials  
  - Inputs: From `Switch` node (telegram mode branch)  
  - Outputs: Confirmation of message sent  
  - Edge cases: Telegram API errors, invalid chat ID, message length limits.

- **n8nResponse**  
  - Type: NoOp  
  - Role: Placeholder node for internal chat mode response (no external output).  
  - Inputs: From `Switch` node (n8n mode branch)  
  - Outputs: None  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                                 | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                   |
|---------------------|----------------------------------|------------------------------------------------|------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| telegramInput       | Telegram Trigger                  | Receives Telegram chat messages                 | —                      | SendTyping               | ## Chat input triggered by Telegram<br>Used for live chat within Telegram                    |
| n8nInput            | Langchain Chat Trigger           | Receives internal chat messages                  | —                      | n8nChatSettings          | ## Chat input triggered inside n8n<br>Used for testing and debugging                         |
| SendTyping          | Telegram                         | Sends "typing..." action to Telegram chat       | telegramInput          | telegramChatSettings     |                                                                                              |
| telegramChatSettings| Set                             | Normalizes Telegram input into unified variables| SendTyping             | Settings                 |                                                                                              |
| n8nChatSettings     | Set                             | Normalizes internal chat input                   | n8nInput               | Settings                 |                                                                                              |
| Settings            | Set                             | Stores global settings and passes variables     | telegramChatSettings, n8nChatSettings | Schedule               |                                                                                              |
| Schedule            | Google Sheets                   | Retrieves event schedule from Google Sheets     | Settings               | ScheduleToMarkdown       | ## Retrieve Data<br>Get schedule from Google Spreadsheet and convert it to a Markdown-Table as context for the LLM |
| ScheduleToMarkdown  | Code                            | Converts schedule rows to markdown table         | Schedule               | ScheduleBot              |                                                                                              |
| ScheduleBot         | Langchain Agent                 | AI processing: generates response using LLM     | ScheduleToMarkdown, Settings | SetResponse           | ## AI Processing<br>Chat input → Chat output                                                |
| LLM                 | Langchain LM Chat OpenRouter   | Language model backend                            | ScheduleBot            | ScheduleBot              |                                                                                              |
| Memory              | Langchain Memory Buffer Window | Maintains conversation context                    | ScheduleBot            | ScheduleBot              |                                                                                              |
| SetResponse         | Set                             | Prepares AI response message                      | ScheduleBot            | Switch                   | ## Prepare response<br>Decide to which chat the response will go.                           |
| Switch              | Switch                         | Routes response to Telegram or internal chat     | SetResponse            | n8nResponse, telegramResponse |                                                                                              |
| telegramResponse    | Telegram                       | Sends AI response back to Telegram chat          | Switch (telegram mode) | —                        | ## Chat response to Telegram                                                                |
| n8nResponse         | NoOp                           | Placeholder for internal chat response            | Switch (n8n mode)      | —                        | ## Chat response inside n8n                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot API credentials.  
   - Set to listen for "message" updates only.

2. **Create Langchain Chat Trigger Node**  
   - Type: Langchain Chat Trigger  
   - No special parameters needed. Used for internal testing.

3. **Create Telegram Node to Send Typing Action**  
   - Type: Telegram  
   - Operation: sendChatAction  
   - Chat ID: Expression `={{ $('telegramInput').item.json.message.chat.id }}`  
   - Use Telegram API credentials.

4. **Create Set Node for Telegram Chat Settings**  
   - Type: Set  
   - Assign variables:  
     - `inputMessage` = `={{ $('telegramInput').item.json.message.text }}`  
     - `chatId` = `={{ $('telegramInput').item.json.message.chat.id }}`  
     - `mode` = `"telegram"` (string literal)

5. **Create Set Node for Internal Chat Settings**  
   - Type: Set  
   - Assign variables:  
     - `inputMessage` = `={{ $json.chatInput }}`  
     - `chatId` = `={{ $json.sessionId }}`  
     - `mode` = `"n8n"` (string literal)

6. **Create Set Node for Global Settings**  
   - Type: Set  
   - Assign variables:  
     - `scheduleURL` = `"https://docs.google.com/spreadsheets/d/1BJFS9feEy94_WgIgzWZttBwzjp09siOw1xuUgq4yuI4"` (replace with your own)  
     - Pass through `inputMessage`, `chatId`, `mode` from previous nodes.

7. **Create Google Sheets Node to Retrieve Schedule**  
   - Type: Google Sheets  
   - Document ID: Extract from `scheduleURL` (use expression)  
   - Sheet Name: `gid=0` (first sheet)  
   - Use Google Sheets OAuth2 credentials.

8. **Create Code Node to Convert Schedule to Markdown**  
   - Type: Code (JavaScript)  
   - Paste the provided code that converts rows to markdown table (see node details).  
   - Input: Rows from Google Sheets node.

9. **Create Langchain Agent Node (ScheduleBot)**  
   - Type: Langchain Agent  
   - Text input: `={{ $('Settings').first().json.inputMessage }}`  
   - System message: Include assistant role description and inject `{{ $json.markdown }}` from markdown node.  
   - Set prompt type to "define".  
   - Connect to LLM and Memory nodes.

10. **Create Langchain LLM Node (OpenRouter)**  
    - Type: Langchain LM Chat OpenRouter  
    - Configure with OpenRouter API credentials.

11. **Create Langchain Memory Node**  
    - Type: Langchain Memory Buffer Window  
    - Session key: `={{ $('Settings').first().json.chatId }}`  
    - Session ID type: customKey.

12. **Create Set Node to Prepare Response**  
    - Type: Set  
    - Assign `responseMessage` = `={{ $json.output }}` from ScheduleBot.

13. **Create Switch Node to Route Response**  
    - Type: Switch  
    - Condition: Check `mode` from `Settings` node.  
    - If `mode` == "n8n", route to `n8nResponse`.  
    - If `mode` == "telegram", route to `telegramResponse`.

14. **Create Telegram Node to Send Response**  
    - Type: Telegram  
    - Operation: sendMessage (default)  
    - Text: `={{ $json.responseMessage }}`  
    - Chat ID: `={{ $('Settings').first().json.chatId }}`  
    - Use Telegram API credentials.

15. **Create NoOp Node for Internal Response**  
    - Type: NoOp  
    - Used as placeholder for internal chat mode response.

16. **Connect Nodes According to Logic:**  
    - `telegramInput` → `SendTyping` → `telegramChatSettings` → `Settings`  
    - `n8nInput` → `n8nChatSettings` → `Settings`  
    - `Settings` → `Schedule` → `ScheduleToMarkdown` → `ScheduleBot`  
    - `ScheduleBot` → `SetResponse` → `Switch`  
    - `Switch` → `telegramResponse` (telegram mode)  
    - `Switch` → `n8nResponse` (n8n mode)

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow was created as a proof-of-concept demo for an AI & Developer Meetup hackathon in Da Nang, Vietnam. | Context of original development.                                                                    |
| To test the Telegram bot live, use https://t.me/AiDaNangBot and ask questions like "When is the next meetup?" | Live demo link for reviewers.                                                                       |
| Telegram bot creation guide: https://core.telegram.org/bots#how-do-i-create-a-bot                              | Instructions for creating your own Telegram bot.                                                   |
| Event Schedule Template Sheet: https://docs.google.com/spreadsheets/d/1fAxGcdOZzQZpsvhiB-ULEDndb4AVKOA_B-sKoTfKvyE/edit?usp=sharing | Template for event schedule data structure.                                                        |
| The structure of the spreadsheet is flexible but should have clearly labeled columns in the header row.       | Important for correct markdown conversion and AI context.                                         |

---

This document fully describes the workflow structure, node configurations, data flow, and setup instructions to enable reproduction, modification, and troubleshooting.