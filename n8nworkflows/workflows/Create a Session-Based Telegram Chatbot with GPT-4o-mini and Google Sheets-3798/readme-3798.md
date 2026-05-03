Create a Session-Based Telegram Chatbot with GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/create-a-session-based-telegram-chatbot-with-gpt-4o-mini-and-google-sheets-3798


# Create a Session-Based Telegram Chatbot with GPT-4o-mini and Google Sheets

### 1. Workflow Overview

This workflow implements a **session-based AI-powered Telegram chatbot** using GPT-4o-mini and Google Sheets for persistent session and conversation management. It enables users to interact with the bot through Telegram commands to start new sessions, check or resume existing sessions, get conversation summaries, and ask questions related to past chats.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Command Parsing**  
  Receives Telegram messages and routes them based on recognized commands (`/new`, `/current`, `/resume`, `/summary`, `/question`).

- **1.2 Session Management**  
  Manages session lifecycle using Google Sheets: creating new sessions, disabling old ones, resuming sessions, and tracking current active sessions.

- **1.3 AI Chatbot Interaction**  
  Processes user messages with GPT-4o-mini, maintaining contextual memory per session, and generating AI responses.

- **1.4 Summarization**  
  Summarizes entire conversation histories on demand using OpenAI summarization chains.

- **1.5 Question Answering**  
  Answers user questions about past conversations by analyzing stored chat logs with OpenAI language models.

- **1.6 Data Logging**  
  Logs all prompts and AI responses into Google Sheets for auditability and persistent memory.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Command Parsing

- **Overview:**  
  This block listens for incoming Telegram messages and determines which command or text the user sent, routing the workflow accordingly.

- **Nodes Involved:**  
  - Get message (Telegram Trigger)  
  - Command or text? (Switch)

- **Node Details:**

  - **Get message**  
    - Type: Telegram Trigger  
    - Role: Listens for Telegram messages (updates of type "message").  
    - Configuration: Uses Telegram Bot credentials; webhook enabled.  
    - Input: Telegram webhook event.  
    - Output: Message JSON with text and user info.  
    - Edge cases: Telegram API downtime, webhook misconfiguration.

  - **Command or text?**  
    - Type: Switch  
    - Role: Routes messages based on command prefix (`/new`, `/current`, `/resume`, `/summary`, `/question`).  
    - Configuration: Uses string "startsWith" conditions on message text to identify commands; fallback output for unrecognized inputs.  
    - Input: Output from Get message node.  
    - Output: Six branches corresponding to commands or fallback.  
    - Edge cases: Case sensitivity, malformed commands, empty messages.

---

#### 2.2 Session Management

- **Overview:**  
  Handles creation, expiration, resumption, and retrieval of sessions stored in Google Sheets.

- **Nodes Involved:**  
  - Get session (Google Sheets)  
  - Disable previous session (Google Sheets)  
  - Set new session (Google Sheets)  
  - Session activated (Telegram Send)  
  - Set to expire (Google Sheets)  
  - Set new current session (Google Sheets)  
  - Exist? (If)  
  - OK (Telegram Send)  
  - KO (Telegram Send)  
  - Trim resume (Code)

- **Node Details:**

  - **Get session**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves the current active session (`STATE = current`) from the `Session` sheet.  
    - Configuration: Returns first match where `STATE` equals "current".  
    - Input: Triggered after receiving a message.  
    - Output: Session data including `SESSION` ID and `STATE`.  
    - Edge cases: No active session found, Google Sheets API errors.

  - **Disable previous session**  
    - Type: Google Sheets (Update)  
    - Role: Marks the current session as expired (`STATE = expire`) before creating a new one.  
    - Configuration: Updates row matching current `SESSION`.  
    - Input: Session data from Get session.  
    - Output: Confirmation of update.  
    - Edge cases: Session not found, update conflicts.

  - **Set new session**  
    - Type: Google Sheets (Append)  
    - Role: Creates a new session entry with `STATE = current` and `SESSION` set to Telegram update ID.  
    - Configuration: Appends new row in `Session` sheet.  
    - Input: After disabling previous session.  
    - Output: New session record.  
    - Edge cases: Append failures, duplicate sessions.

  - **Session activated**  
    - Type: Telegram Send  
    - Role: Notifies user that a new session has been activated.  
    - Configuration: Sends fixed text "New session activated" to user's chat ID.  
    - Input: After new session creation.  
    - Output: Telegram message sent.  
    - Edge cases: Telegram API errors, invalid chat ID.

  - **Set to expire**  
    - Type: Google Sheets (Update)  
    - Role: Marks a session as expired during resumption or switching.  
    - Configuration: Updates `STATE` to "expire" for given `SESSION`.  
    - Input: Session ID from Trim resume node.  
    - Output: Update confirmation.  
    - Edge cases: Session not found, update conflicts.

  - **Set new current session**  
    - Type: Google Sheets (Update)  
    - Role: Sets a resumed session's `STATE` to "current".  
    - Configuration: Updates row matching resumed `SESSION`.  
    - Input: Session ID extracted from user command.  
    - Output: Confirmation of update.  
    - Edge cases: Session does not exist, update failure.

  - **Exist?**  
    - Type: If  
    - Role: Checks if a session exists after attempting to set it current.  
    - Configuration: Checks if input JSON is empty (session not found).  
    - Input: Output of Set new current session.  
    - Output: Routes to OK or KO nodes accordingly.  
    - Edge cases: False positives if data malformed.

  - **OK**  
    - Type: Telegram Send  
    - Role: Sends confirmation that the current session is active.  
    - Configuration: Sends text with session ID to user.  
    - Input: If session exists.  
    - Output: Telegram message sent.  
    - Edge cases: Telegram API errors.

  - **KO**  
    - Type: Telegram Send  
    - Role: Sends error message if session does not exist.  
    - Configuration: Sends "This session doesn't exist" to user.  
    - Input: If session does not exist.  
    - Output: Telegram message sent.  
    - Edge cases: Telegram API errors.

  - **Trim resume**  
    - Type: Code  
    - Role: Extracts session ID from `/resume [session_id]` command text.  
    - Configuration: Uses regex to parse command argument.  
    - Input: Message text from Get message1 node.  
    - Output: JSON with extracted `resume` session ID or null.  
    - Edge cases: Missing or malformed session ID.

---

#### 2.3 AI Chatbot Interaction

- **Overview:**  
  Processes user messages through GPT-4o-mini with session-based memory to generate contextual AI responses.

- **Nodes Involved:**  
  - Telegram Chatbot (Langchain Agent)  
  - OpenAI Chat Model  
  - Simple Memory  
  - Send response (Telegram Send)  
  - Update database (Google Sheets)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4o-mini language model for generating AI responses.  
    - Configuration: Model set to "gpt-4o-mini", linked with OpenAI API credentials.  
    - Input: Text prompt from Telegram Chatbot node.  
    - Output: AI-generated text.  
    - Edge cases: API quota exceeded, network timeouts, invalid API key.

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains conversational context per session using a sliding window of last 100 messages.  
    - Configuration: Session key derived from current session ID; context window length set to 100.  
    - Input: Session ID from Get session node; messages from Telegram Chatbot.  
    - Output: Contextual memory for AI prompt.  
    - Edge cases: Missing session ID, memory overflow.

  - **Telegram Chatbot**  
    - Type: Langchain Agent  
    - Role: Orchestrates AI prompt construction and response generation using OpenAI Chat Model and Simple Memory.  
    - Configuration: Uses system message with current date/time; input text from Telegram message.  
    - Input: User message text, session memory.  
    - Output: AI response text.  
    - Edge cases: Model failures, memory sync issues.

  - **Send response**  
    - Type: Telegram Send  
    - Role: Sends AI-generated response back to the user on Telegram.  
    - Configuration: Sends text from Telegram Chatbot output to user's chat ID.  
    - Input: AI response text.  
    - Output: Telegram message sent.  
    - Edge cases: Telegram API errors.

  - **Update database**  
    - Type: Google Sheets (Append)  
    - Role: Logs prompt and AI response with timestamp and session ID into `Database` sheet.  
    - Configuration: Appends row with columns DATE, PROMPT, SESSION, RESPONSE.  
    - Input: User prompt and AI response.  
    - Output: Confirmation of append.  
    - Edge cases: Google Sheets API errors, data format issues.

---

#### 2.4 Summarization

- **Overview:**  
  Generates concise summaries of entire conversation histories upon user request.

- **Nodes Involved:**  
  - Get session1 (Google Sheets)  
  - Prompt + Resume (Code)  
  - Summarization Chain (Langchain Summarization Chain)  
  - OpenAI Chat Model1  
  - Send summary (Telegram Send)

- **Node Details:**

  - **Get session1**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves all chat logs for the current session from `Database` sheet.  
    - Configuration: Filters rows where `SESSION` matches current session ID.  
    - Input: Session ID from Get session node.  
    - Output: Array of chat log entries.  
    - Edge cases: No logs found, API errors.

  - **Prompt + Resume**  
    - Type: Code  
    - Role: Concatenates all PROMPT and RESPONSE pairs into a single text block for summarization.  
    - Configuration: Iterates over input items, formats text as "PROMPT: ... RESPONSE: ...".  
    - Input: Chat logs from Get session1.  
    - Output: JSON with `fullText` containing conversation history and `chat_id` for Telegram.  
    - Edge cases: Empty logs, malformed data.

  - **OpenAI Chat Model1**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4o-mini model for summarization chain.  
    - Configuration: Same as main OpenAI Chat Model node.  
    - Input: Summarization prompt.  
    - Output: Summary text.  
    - Edge cases: API errors, rate limits.

  - **Summarization Chain**  
    - Type: Langchain Summarization Chain  
    - Role: Uses a prompt template to generate a concise summary of the conversation text.  
    - Configuration: Prompt instructs to "Write a concise summary of the following...".  
    - Input: Full conversation text from Prompt + Resume.  
    - Output: Summarized text.  
    - Edge cases: Long input exceeding model limits.

  - **Send summary**  
    - Type: Telegram Send  
    - Role: Sends the generated summary back to the user.  
    - Configuration: Sends summary text to user's chat ID.  
    - Input: Summary from Summarization Chain.  
    - Output: Telegram message sent.  
    - Edge cases: Telegram API errors.

---

#### 2.5 Question Answering

- **Overview:**  
  Allows users to ask questions about past conversations, answered by analyzing stored chat logs.

- **Nodes Involved:**  
  - Get session1 (Google Sheets)  
  - Response + Text (Google Sheets)  
  - fullText (Code)  
  - Trim question (Code)  
  - Basic LLM Chain (Langchain LLM Chain)  
  - OpenAI Chat Model2  
  - Send answer (Telegram Send)

- **Node Details:**

  - **Get session1**  
    - Same as in summarization block; retrieves chat logs for the session.

  - **Response + Text**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves chat logs filtered by session ID (redundant with Get session1, possibly for data consistency).  
    - Configuration: Filters by `SESSION`.  
    - Input: Session ID.  
    - Output: Chat logs.  
    - Edge cases: API errors.

  - **fullText**  
    - Type: Code  
    - Role: Concatenates PROMPT and RESPONSE pairs into a single text block and extracts the question from user input.  
    - Configuration: Iterates over chat logs, formats text, and includes question extracted by Trim question.  
    - Input: Chat logs and trimmed question.  
    - Output: JSON with `fullText`, `chat_id`, and `question`.  
    - Edge cases: Empty logs, missing question.

  - **Trim question**  
    - Type: Code  
    - Role: Extracts the question text from `/question [query]` command.  
    - Configuration: Regex match to extract question after `/question`.  
    - Input: User message text.  
    - Output: JSON with `question` field.  
    - Edge cases: Missing or malformed question.

  - **OpenAI Chat Model2**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-4o-mini model for answering questions.  
    - Configuration: Same as other OpenAI Chat Model nodes.  
    - Input: Prompt from Basic LLM Chain.  
    - Output: Answer text.  
    - Edge cases: API errors.

  - **Basic LLM Chain**  
    - Type: Langchain LLM Chain  
    - Role: Defines prompt template to answer questions by analyzing conversation text.  
    - Configuration: Prompt includes "You have to answer the questions that are asked by analyzing the following text: ...".  
    - Input: Full conversation text and question.  
    - Output: AI-generated answer.  
    - Edge cases: Model misunderstanding, prompt injection.

  - **Send answer**  
    - Type: Telegram Send  
    - Role: Sends the AI-generated answer back to the user.  
    - Configuration: Sends text to user's chat ID.  
    - Input: Answer text.  
    - Output: Telegram message sent.  
    - Edge cases: Telegram API errors.

---

#### 2.6 Data Logging

- **Overview:**  
  Logs all user prompts and AI responses with timestamps and session IDs into Google Sheets for persistence and audit.

- **Nodes Involved:**  
  - Update database (Google Sheets)

- **Node Details:**

  - **Update database**  
    - Type: Google Sheets (Append)  
    - Role: Appends a new row to the `Database` sheet with DATE, PROMPT, SESSION, RESPONSE.  
    - Configuration: Uses current timestamp, user message text, session ID, and AI response.  
    - Input: User message and AI response.  
    - Output: Confirmation of append.  
    - Edge cases: API errors, data format issues.

---

### 3. Summary Table

| Node Name             | Node Type                              | Functional Role                          | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                   |
|-----------------------|--------------------------------------|----------------------------------------|-------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Get message           | Telegram Trigger                     | Receive Telegram messages               | -                       | Command or text?             | # Telegram ChatBot with multiple sessions - Clone [this sheet](https://docs.google.com/spreadsheets/d/1MCJLAqKP0Y7Qr68ZYoSSBeEVyKI1QgAAZnlEiyqkzXo/edit?usp=sharing) |
| Command or text?      | Switch                              | Route messages by command               | Get message             | Disable previous session, Send current session, Get message1, Get session1, Get message2, Telegram Chatbot |                                                                                               |
| Get session           | Google Sheets (Read)                | Retrieve current active session         | Get message             | Command or text?             |                                                                                               |
| Disable previous session | Google Sheets (Update)             | Expire old session before new one       | Command or text? (New session) | Set new session              |                                                                                               |
| Set new session       | Google Sheets (Append)              | Create new session                      | Disable previous session | Session activated            |                                                                                               |
| Session activated     | Telegram Send                      | Notify user of new session activation  | Set new session          | -                           |                                                                                               |
| Send current session  | Telegram Send                      | Send current session info to user      | Command or text? (Current session) | -                           |                                                                                               |
| Get message1          | Set                                | Extract message text                    | Command or text? (Resume) | Trim resume                  |                                                                                               |
| Trim resume           | Code                               | Extract session ID from /resume command | Get message1             | Set to expire                |                                                                                               |
| Set to expire         | Google Sheets (Update)              | Expire session before resuming          | Trim resume              | Set new current session      |                                                                                               |
| Set new current session | Google Sheets (Update)             | Mark resumed session as current         | Set to expire            | Exist?                      |                                                                                               |
| Exist?                | If                                 | Check if resumed session exists          | Set new current session  | KO, OK                      |                                                                                               |
| KO                    | Telegram Send                      | Notify user session does not exist      | Exist? (false)           | -                           |                                                                                               |
| OK                    | Telegram Send                      | Confirm current resumed session         | Exist? (true)            | -                           |                                                                                               |
| Get session1          | Google Sheets (Read)                | Retrieve chat logs for session           | Command or text? (Summary, Question) | Prompt + Resume, Response + Text |                                                                                               |
| Prompt + Resume       | Code                               | Concatenate chat logs for summarization | Get session1             | Summarization Chain          |                                                                                               |
| Summarization Chain   | Langchain Summarization Chain      | Generate concise summary                 | Prompt + Resume          | Send summary                 |                                                                                               |
| OpenAI Chat Model1    | Langchain OpenAI Chat Model        | GPT-4o-mini model for summarization     | Summarization Chain (ai_languageModel) | Summarization Chain          |                                                                                               |
| Send summary          | Telegram Send                      | Send summary to user                     | Summarization Chain      | -                           |                                                                                               |
| Get message2          | Set                                | Extract message text                    | Command or text? (Question) | Trim question                |                                                                                               |
| Trim question         | Code                               | Extract question from /question command | Get message2             | Response + Text              |                                                                                               |
| Response + Text       | Google Sheets (Read)                | Retrieve chat logs for question answering | Trim question            | fullText                    |                                                                                               |
| fullText              | Code                               | Concatenate chat logs and extract question | Response + Text, Trim question | Basic LLM Chain             |                                                                                               |
| Basic LLM Chain       | Langchain LLM Chain                | Generate answer to user question         | fullText                 | Send answer                 |                                                                                               |
| OpenAI Chat Model2    | Langchain OpenAI Chat Model        | GPT-4o-mini model for question answering | Basic LLM Chain (ai_languageModel) | Basic LLM Chain             |                                                                                               |
| Send answer           | Telegram Send                      | Send AI answer to user                   | Basic LLM Chain           | -                           |                                                                                               |
| Telegram Chatbot      | Langchain Agent                   | Generate AI response with session memory | Command or text? (Question branch) | Send response               |                                                                                               |
| OpenAI Chat Model     | Langchain OpenAI Chat Model        | GPT-4o-mini model for chatbot responses  | Telegram Chatbot (ai_languageModel) | Telegram Chatbot            |                                                                                               |
| Simple Memory         | Langchain Memory Buffer Window    | Maintain session-based conversation memory | Get session, Telegram Chatbot | Telegram Chatbot            |                                                                                               |
| Send response         | Telegram Send                      | Send AI chatbot response to user         | Telegram Chatbot          | Update database             |                                                                                               |
| Update database       | Google Sheets (Append)             | Log prompt and response                   | Send response             | -                           |                                                                                               |
| Sticky Note           | Sticky Note                       | Workflow description and sheet link      | -                       | -                           | # Telegram ChatBot with multiple sessions - Clone [this sheet](https://docs.google.com/spreadsheets/d/1MCJLAqKP0Y7Qr68ZYoSSBeEVyKI1QgAAZnlEiyqkzXo/edit?usp=sharing) |
| Sticky Note1          | Sticky Note                       | Label: NEW SESSION block                  | -                       | -                           |                                                                                               |
| Sticky Note2          | Sticky Note                       | Label: GET CURRENT SESSION block          | -                       | -                           |                                                                                               |
| Sticky Note3          | Sticky Note                       | Label: RESUME SESSION block                | -                       | -                           |                                                                                               |
| Sticky Note4          | Sticky Note                       | Label: GET SUMMARY block                   | -                       | -                           |                                                                                               |
| Sticky Note5          | Sticky Note                       | Label: SEND QUESTION block                  | -                       | -                           |                                                                                               |
| Sticky Note6          | Sticky Note                       | Label: CHATBOT block                        | -                       | -                           |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("Get message")**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot Token credentials.  
   - Set to listen for "message" updates.  
   - Enable webhook.

2. **Add Switch Node ("Command or text?")**  
   - Connect input from "Get message".  
   - Add rules to detect commands by checking if message text starts with:  
     - `/new` → Output: "New session"  
     - `/current` → Output: "Current session"  
     - `/resume` → Output: "Resume session"  
     - `/summary` → Output: "Summary"  
     - `/question` → Output: "Question"  
   - Set fallback output for unrecognized inputs.

3. **Session Management Block**

   - **Get session (Google Sheets Read)**  
     - Connect from "Get message".  
     - Configure to read from `Session` sheet, filter rows where `STATE` = "current".  
     - Use Google Sheets OAuth2 credentials.

   - **Disable previous session (Google Sheets Update)**  
     - Connect from "Command or text?" node's "New session" output.  
     - Update `Session` sheet row matching current `SESSION` to set `STATE` = "expire".

   - **Set new session (Google Sheets Append)**  
     - Connect from "Disable previous session".  
     - Append new row with `SESSION` = Telegram update ID, `STATE` = "current".

   - **Session activated (Telegram Send)**  
     - Connect from "Set new session".  
     - Send text "New session activated" to user's chat ID.

   - **Send current session (Telegram Send)**  
     - Connect from "Command or text?" node's "Current session" output.  
     - Send text "The current session is {{SESSION}}" to user.

   - **Get message1 (Set)**  
     - Connect from "Command or text?" node's "Resume session" output.  
     - Assign variable `text` = incoming message text.

   - **Trim resume (Code)**  
     - Connect from "Get message1".  
     - Use regex to extract session ID after `/resume`.  
     - Output JSON with `resume` field.

   - **Set to expire (Google Sheets Update)**  
     - Connect from "Trim resume".  
     - Update `Session` sheet row matching `SESSION` = extracted resume ID to set `STATE` = "expire".

   - **Set new current session (Google Sheets Update)**  
     - Connect from "Set to expire".  
     - Update `Session` sheet row matching `SESSION` = extracted resume ID to set `STATE` = "current".

   - **Exist? (If)**  
     - Connect from "Set new current session".  
     - Condition: Check if output is empty (session does not exist).

   - **KO (Telegram Send)**  
     - Connect from "Exist?" false branch.  
     - Send "This session doesn't exist" to user.

   - **OK (Telegram Send)**  
     - Connect from "Exist?" true branch.  
     - Send "The current session is {{SESSION}}" to user.

4. **Summarization Block**

   - **Get session1 (Google Sheets Read)**  
     - Connect from "Command or text?" node's "Summary" output.  
     - Read all chat logs from `Database` sheet where `SESSION` matches current session.

   - **Prompt + Resume (Code)**  
     - Connect from "Get session1".  
     - Concatenate all PROMPT and RESPONSE pairs into one string `fullText`.

   - **OpenAI Chat Model1 (Langchain OpenAI Chat Model)**  
     - Configure with GPT-4o-mini and OpenAI API credentials.

   - **Summarization Chain (Langchain Summarization Chain)**  
     - Connect from "Prompt + Resume".  
     - Use prompt: "Write a concise summary of the following: \"{{ fullText }}\" CONCISE SUMMARY:".

   - **Send summary (Telegram Send)**  
     - Connect from "Summarization Chain".  
     - Send summary text to user's chat ID.

5. **Question Answering Block**

   - **Get message2 (Set)**  
     - Connect from "Command or text?" node's "Question" output.  
     - Assign variable `text` = incoming message text.

   - **Trim question (Code)**  
     - Connect from "Get message2".  
     - Extract question text after `/question`.

   - **Response + Text (Google Sheets Read)**  
     - Connect from "Trim question".  
     - Read chat logs from `Database` sheet filtered by current session.

   - **fullText (Code)**  
     - Connect from "Response + Text" and "Trim question".  
     - Concatenate PROMPT and RESPONSE pairs; include extracted question.

   - **OpenAI Chat Model2 (Langchain OpenAI Chat Model)**  
     - Configure with GPT-4o-mini and OpenAI API credentials.

   - **Basic LLM Chain (Langchain LLM Chain)**  
     - Connect from "fullText".  
     - Prompt template:  
       ```
       You have to answer the questions that are asked by analyzing the following text:

       {{ fullText }}

       Question:
       {{ question }}
       ```
   - **Send answer (Telegram Send)**  
     - Connect from "Basic LLM Chain".  
     - Send AI-generated answer to user's chat ID.

6. **AI Chatbot Interaction Block**

   - **Simple Memory (Langchain Memory Buffer Window)**  
     - Configure with session key from current session ID.  
     - Set context window length to 100 messages.

   - **OpenAI Chat Model (Langchain OpenAI Chat Model)**  
     - Configure with GPT-4o-mini and OpenAI API credentials.

   - **Telegram Chatbot (Langchain Agent)**  
     - Connect from "Command or text?" node's "Question" branch.  
     - Use system message with current date/time.  
     - Use Simple Memory and OpenAI Chat Model for response generation.

   - **Send response (Telegram Send)**  
     - Connect from "Telegram Chatbot".  
     - Send AI response to user.

   - **Update database (Google Sheets Append)**  
     - Connect from "Send response".  
     - Append new row to `Database` sheet with DATE, PROMPT, SESSION, RESPONSE.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Telegram Bot Token must be created via [BotFather](https://core.telegram.org/bots#botfather).       | Telegram Bot setup                                                                                           |
| Google Sheets template with `Session` and `Database` sheets: [Template Sheet](https://docs.google.com/spreadsheets/d/1MCJLAqKP0Y7Qr68ZYoSSBeEVyKI1QgAAZnlEiyqkzXo/edit) | Required for session and chat log storage                                                                   |
| OpenAI API Key required for GPT-4o-mini model usage.                                                | OpenAI API setup                                                                                            |
| Workflow supports commands: `/new`, `/current`, `/resume [session_id]`, `/summary`, `/question [query]` | User interaction commands                                                                                   |
| Contact for customization and support: [info@n3w.it](mailto:info@n3w.it), LinkedIn: [Davide Boizza](https://www.linkedin.com/in/davideboizza/) | Consulting and support                                                                                       |
| Uses Langchain nodes for memory and chain management, requiring n8n version supporting Langchain nodes | Version compatibility                                                                                       |
| Google Sheets OAuth2 credentials must be configured with access to the specified spreadsheet.       | Credential setup                                                                                            |
| Telegram API credentials must be configured with the bot token.                                     | Credential setup                                                                                            |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and extending the Telegram Chatbot with multiple session management using GPT-4o-mini and Google Sheets.