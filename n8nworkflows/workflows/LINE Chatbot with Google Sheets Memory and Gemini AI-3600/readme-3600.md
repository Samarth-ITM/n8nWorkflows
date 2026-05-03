LINE Chatbot with Google Sheets Memory and Gemini AI

https://n8nworkflows.xyz/workflows/line-chatbot-with-google-sheets-memory-and-gemini-ai-3600


# LINE Chatbot with Google Sheets Memory and Gemini AI

### 1. Workflow Overview

This workflow automates AI-powered conversational replies for a LINE Official Account by integrating Google Sheets as a memory store and Google Gemini AI for generating context-aware responses. It is designed for businesses or support teams seeking personalized, continuous interactions with users by maintaining chat history and leveraging AI to understand and respond appropriately.

The workflow is logically divided into the following blocks:

- **1.1 Connect to LINE Official Account API:** Receives incoming messages via a webhook.
- **1.2 Prepare Incoming Data:** Extracts and structures relevant user and message data.
- **1.3 Retrieve Chat History:** Fetches previous conversation history from Google Sheets.
- **1.4 Prepare AI Prompt:** Combines historical chat data with the new message to create a prompt.
- **1.5 AI Processing with Google Gemini:** Sends the prompt to Google Gemini AI to generate a reply.
- **1.6 Split & Clean History:** Processes and chunks chat history to fit Google Sheets limits.
- **1.7 Save Updated History:** Updates Google Sheets with the latest conversation data.
- **1.8 Send Reply to LINE:** Sends the AI-generated response back to the user via LINE Messaging API.

---

### 2. Block-by-Block Analysis

#### 1.1 Connect to LINE Official Account API

- **Overview:** Listens for incoming POST requests from LINE when a user sends a message.
- **Nodes Involved:** Webhook, Sticky Note (comment)
- **Node Details:**
  - **Webhook**
    - Type: Webhook (HTTP POST listener)
    - Configuration: Listens on path `/guitarpa` for POST requests.
    - Inputs: External HTTP POST from LINE platform.
    - Outputs: JSON payload containing user message and metadata.
    - Edge Cases: Invalid or malformed webhook requests, missing required fields.
  - **Sticky Note**
    - Content: "### Connect to Line Official Account's API"
    - Purpose: Documentation aid.

#### 1.2 Prepare Incoming Data

- **Overview:** Extracts user ID, message text, and reply token from the webhook payload for downstream use.
- **Nodes Involved:** Edit Fields, Sticky Note (comment)
- **Node Details:**
  - **Edit Fields**
    - Type: Set node (field extraction and assignment)
    - Configuration: Extracts `body.events[0].message.text`, `body.events[0].replyToken`, and `body.events[0].source.userId` from the webhook JSON.
    - Inputs: Output from Webhook node.
    - Outputs: Cleaned JSON with essential fields for processing.
    - Edge Cases: Missing fields in webhook payload, empty messages.
  - **Sticky Note**
    - Content: "Prepare the data"

#### 1.3 Retrieve Chat History

- **Overview:** Queries Google Sheets to fetch the user's previous conversation history to provide context for AI responses.
- **Nodes Involved:** Get History (Google Sheets Read), Sticky Note (comment)
- **Node Details:**
  - **Get History**
    - Type: Google Sheets node (read operation)
    - Configuration:
      - Reads from a specific Google Sheet document and sheet (gid=0).
      - Filters rows by matching `UserID` column with the extracted user ID.
      - Returns the first matching row containing chat history fields (`History`, `History_Archive_1` to `History_Archive_4`).
    - Inputs: Output from Edit Fields.
    - Outputs: JSON containing chat history segments.
    - Credentials: Google Sheets OAuth2.
    - Edge Cases: No history found for user, Google Sheets API errors, rate limits.
  - **Sticky Note**
    - Content: "Retrieve chat history"

#### 1.4 Prepare AI Prompt

- **Overview:** Constructs a prompt string combining multiple archived history fields and the new user message to feed into the AI agent.
- **Nodes Involved:** Prepare Prompt (Set node), Sticky Note (comment)
- **Node Details:**
  - **Prepare Prompt**
    - Type: Set node
    - Configuration:
      - Concatenates up to five history fields (`History_Archive_1` to `History_Archive_4` and `History`) with line breaks.
      - Adds a prefix in Thai instructing the AI to act as a polite and friendly chatbot named "ลลิตา".
      - Appends the current user message and a prompt for the AI reply.
    - Inputs: Output from Get History and Edit Fields.
    - Outputs: JSON with a single field `Prompt` containing the full prompt string.
    - Edge Cases: Missing or empty history fields, malformed concatenation.
  - **Sticky Note**
    - Content: "Give our AI previous chat history"

#### 1.5 AI Processing with Google Gemini

- **Overview:** Uses Google Gemini Chat Model via an AI Agent node to generate a context-aware reply based on the prepared prompt.
- **Nodes Involved:** AI Agent, Google Gemini Chat Model, Sticky Note (comment)
- **Node Details:**
  - **Google Gemini Chat Model**
    - Type: Google Gemini Language Model node
    - Configuration:
      - Model: `models/gemini-2.0-flash-001`
      - Credentials: Google Palm API (Google Gemini)
    - Inputs: Connected as language model for AI Agent.
    - Outputs: AI-generated text response.
    - Edge Cases: API authentication errors, rate limits, model unavailability.
  - **AI Agent**
    - Type: Langchain AI Agent node
    - Configuration:
      - Receives prompt from `Prepare Prompt`.
      - System message sets assistant name "ลลิตา", language, timezone (Asia/Bangkok), and current date.
      - Uses output parser for structured response.
    - Inputs: Prompt from Prepare Prompt, language model from Google Gemini.
    - Outputs: AI-generated reply text.
    - Edge Cases: Parsing errors, empty AI response.
  - **Sticky Note**
    - Content: "Get input with this command.   \"{{ $json.Prompt }}\""

#### 1.6 Split & Clean History

- **Overview:** Updates and manages chat history by appending the latest exchange and splitting the history into chunks to avoid Google Sheets cell size limits.
- **Nodes Involved:** Split History (Code node), Sticky Note (comment)
- **Node Details:**
  - **Split History**
    - Type: Code node (JavaScript)
    - Configuration:
      - Retrieves current history and archives.
      - Appends new user message and AI reply as a conversation exchange.
      - Checks if updated history exceeds 35,000 characters (70% of Google Sheets cell limit).
      - If exceeded, splits and distributes excess text into archive fields (`History_Archive_1` to `History_Archive_4`), each capped at 35,000 characters.
      - Returns updated history and archives for saving.
    - Inputs: Output from AI Agent and Get History.
    - Outputs: JSON with updated `historyToSave` and archives.
    - Edge Cases: Extremely long conversations, string manipulation errors.
  - **Sticky Note**
    - Content: "Split history into small chunks (data cleaning)"

#### 1.7 Save Updated History

- **Overview:** Writes the updated chat history and archives back to Google Sheets, either appending or updating the user's row.
- **Nodes Involved:** Save History (Google Sheets Write), Sticky Note (comment)
- **Node Details:**
  - **Save History**
    - Type: Google Sheets node (append or update operation)
    - Configuration:
      - Writes fields: `UserID`, `History`, `LastUpdated`, `History_Archive_1` to `History_Archive_4`.
      - Matches rows by `UserID` to update existing or append new.
      - Uses ISO timestamp for `LastUpdated`.
    - Inputs: Output from Split History and Edit Fields.
    - Outputs: Confirmation of write operation.
    - Credentials: Google Sheets OAuth2.
    - Edge Cases: Write conflicts, API errors, quota limits.
  - **Sticky Note**
    - Content: "Save to Google Sheets"

#### 1.8 Send Reply to LINE

- **Overview:** Sends the AI-generated reply back to the user via LINE Messaging API using the reply token.
- **Nodes Involved:** HTTP Request, Sticky Note (comment)
- **Node Details:**
  - **HTTP Request**
    - Type: HTTP Request node
    - Configuration:
      - POST to `https://api.line.me/v2/bot/message/reply`
      - JSON body includes `replyToken` and message text from AI Agent output.
      - Headers include Authorization Bearer token and Content-Type.
      - Cleans AI response text by replacing tabs, quotes, and newlines for JSON safety.
    - Inputs: Output from Save History (which follows Split History).
    - Outputs: LINE API response.
    - Edge Cases: Authorization failures, invalid reply tokens, network timeouts.
  - **Sticky Note**
    - Content: "Send it back to Line"

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                          | Input Node(s)       | Output Node(s)       | Sticky Note                                  |
|-----------------------|----------------------------------|----------------------------------------|---------------------|----------------------|----------------------------------------------|
| Webhook               | Webhook                          | Receive incoming LINE messages          | External HTTP POST   | Edit Fields          | ### Connect to Line Official Account's API  |
| Edit Fields           | Set                              | Extract user ID, message, reply token  | Webhook             | Get History          | Prepare the data                             |
| Get History           | Google Sheets (Read)             | Fetch user chat history from Google Sheets | Edit Fields         | Prepare Prompt       | Retrieve chat history                        |
| Prepare Prompt        | Set                              | Compose AI prompt with history + message | Get History, Edit Fields | AI Agent          | Give our AI previous chat history           |
| Google Gemini Chat Model | Google Gemini Language Model    | Provide AI language model for agent    | AI Agent (as LM)    | AI Agent             | Get input with this command. "{{ $json.Prompt }}" |
| AI Agent              | Langchain AI Agent               | Generate AI reply based on prompt      | Prepare Prompt, Google Gemini Chat Model | Split History | Get input with this command. "{{ $json.Prompt }}" |
| Split History         | Code                             | Append new exchange, split history for storage | AI Agent, Get History | Save History        | Split history into small chunks (data cleaning) |
| Save History          | Google Sheets (Append/Update)   | Save updated chat history back to Google Sheets | Split History, Edit Fields | HTTP Request     | Save to Google Sheets                        |
| HTTP Request          | HTTP Request                    | Send AI reply back to LINE user        | Save History        |                      | Send it back to Line                         |
| Sticky Note           | Sticky Note                     | Documentation comments                  | -                   | -                    | Various (see above)                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook
   - HTTP Method: POST
   - Path: `guitarpa`
   - Purpose: Receive incoming messages from LINE Official Account.
   - No credentials required.
   - Connect output to Edit Fields node.

2. **Create Edit Fields Node**
   - Type: Set
   - Extract and assign:
     - `body.events[0].message.text` from webhook JSON to same path.
     - `body.events[0].replyToken` from webhook JSON.
     - `body.events[0].source.userId` from webhook JSON.
   - Connect input from Webhook.
   - Connect output to Get History node.

3. **Create Get History Node**
   - Type: Google Sheets (Read)
   - Credentials: Google Sheets OAuth2 with access to your chat history sheet.
   - Document ID: Your Google Sheet ID.
   - Sheet Name: `Sheet1` (or your sheet).
   - Filter: Lookup `UserID` column matching `body.events[0].source.userId` from Edit Fields.
   - Return first matching row.
   - Connect input from Edit Fields.
   - Connect output to Prepare Prompt node.

4. **Create Prepare Prompt Node**
   - Type: Set
   - Assign a string field `Prompt` with the following logic:
     - Concatenate the following fields from Get History: `History_Archive_1`, `History_Archive_2`, `History_Archive_3`, `History_Archive_4`, and `History`.
     - Add a Thai-language system message prefix instructing the AI to be polite and friendly as chatbot "ลลิตา".
     - Append the current user message from Edit Fields.
     - Format example:
       ```
       คุณคือลลิตา แชทบอทภาษาไทยที่สุภาพและเป็นมิตร ตอบตามบริบทของการสนทนา:
       [History_Archive_1]
       [History_Archive_2]
       ...
       ผู้ใช้: [Current message]
       ลลิตา:
       ```
   - Connect input from Get History.
   - Connect output to AI Agent node.

5. **Create Google Gemini Chat Model Node**
   - Type: Google Gemini Language Model
   - Credentials: Google Palm API with access to Gemini.
   - Model Name: `models/gemini-2.0-flash-001`
   - No additional parameters required.
   - Connect output to AI Agent node as language model.

6. **Create AI Agent Node**
   - Type: Langchain AI Agent
   - Parameters:
     - Text input: `={{ $json.Prompt }}`
     - System message: `"You are a helpful assistant. Your name is \"ลลิตา\". You will help me in everything I need. You will answer based on user language. You are an AI Agent operating in the Thailand time zone (Asia/Bangkok, UTC+7). Today is {{ $now }}."`
     - Prompt type: Define
     - Enable output parser.
   - Connect input from Prepare Prompt.
   - Connect language model input from Google Gemini Chat Model.
   - Connect output to Split History node.

7. **Create Split History Node**
   - Type: Code (JavaScript)
   - Code logic:
     - Retrieve current history and archives from Get History.
     - Append new user message and AI reply.
     - If combined history exceeds 35,000 characters, split excess into archive fields.
     - Return updated `historyToSave` and archives.
   - Connect input from AI Agent and Get History.
   - Connect output to Save History node.

8. **Create Save History Node**
   - Type: Google Sheets (Append or Update)
   - Credentials: Google Sheets OAuth2.
   - Document ID and Sheet Name same as Get History.
   - Mapping:
     - `UserID` from Edit Fields.
     - `History` from Split History `historyToSave`.
     - `LastUpdated` current ISO timestamp.
     - `History_Archive_1` to `History_Archive_4` from Split History.
   - Matching column: `UserID` to update existing row or append new.
   - Connect input from Split History and Edit Fields.
   - Connect output to HTTP Request node.

9. **Create HTTP Request Node**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://api.line.me/v2/bot/message/reply`
   - Headers:
     - `Authorization: Bearer [Your LINE Channel Access Token]`
     - `Content-Type: application/json`
   - Body (JSON):
     ```json
     {
       "replyToken": "{{ $('Edit Fields').item.json.body.events[0].replyToken }}",
       "messages": [
         {
           "type": "text",
           "text": "{{ $('AI Agent').item.json.output.replaceAll('\\t', ' ').replaceAll('\"', '\\\\\"').replaceAll('\\n', '\\n').trim() || 'No response available.' }}"
         }
       ]
     }
     ```
   - Connect input from Save History.

10. **Add Sticky Notes (Optional)**
    - Add sticky notes near each logical block for documentation and clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow maintains conversational context by storing chat history in Google Sheets.             | Core design principle for personalized AI replies.                                             |
| Google Sheets cell limit is 50,000 characters; history is chunked at 35,000 characters to avoid overflow. | Important for data integrity and storage management.                                           |
| AI Agent is named "ลลิตา" and operates in Thai language and Thailand timezone (Asia/Bangkok).    | Ensures culturally and linguistically appropriate responses.                                   |
| LINE Messaging API requires a valid channel access token for authorization in HTTP Request node. | See https://developers.line.biz/en/docs/messaging-api/overview/                                |
| Google Gemini API credentials must be set up with Google Palm API access.                        | See https://developers.generativeai.google/api/gemini/                                         |
| For webhook setup, ensure LINE Official Account webhook URL points to the automation platform URL with path `/guitarpa`. | LINE Developer Console configuration step.                                                    |
| Regularly monitor Google Sheets for storage limits and API quotas to maintain workflow stability.| Prevents data loss and service interruptions.                                                  |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the LINE Chatbot workflow integrating Google Sheets memory and Google Gemini AI.