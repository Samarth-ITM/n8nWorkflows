Multi-User Telegram Bot to Summarize & Repurpose YouTube Videos with GPT-4o

https://n8nworkflows.xyz/workflows/multi-user-telegram-bot-to-summarize---repurpose-youtube-videos-with-gpt-4o-5468


# Multi-User Telegram Bot to Summarize & Repurpose YouTube Videos with GPT-4o

---

### 1. Workflow Overview

This workflow implements a **multi-user Telegram bot** designed to summarize and repurpose YouTube videos using GPT-4o, an advanced OpenAI language model variant. It targets users who want quick video summaries or chatbot interactions via Telegram, supporting both YouTube link-based queries and general chat messages. The workflow:

- Listens to Telegram messages from multiple users concurrently.
- Detects if a message contains a YouTube video link.
- If yes, extracts the video ID and fetches the transcript from YouTube via RapidAPI.
- Cleans and processes the transcript text.
- Sends the transcript or normal text queries to OpenAI GPT-4o for summarization or conversational replies.
- Maintains conversational context per user/session.
- Cleans the AI response formatting before sending it back to Telegram.
- Supports follow-up commands for editing summaries or content repurposing (e.g., shorter summaries, LinkedIn posts).

The workflow is logically divided into these blocks:

- **1.1 Trigger & Input Reception:** Receives Telegram messages and decides if they contain YouTube links.
- **1.2 YouTube Transcript Processing:** Extracts video ID, fetches transcript, cleans it.
- **1.3 AI Chat & Summarization:** Sends either transcript or normal chat message to OpenAI GPT-4o.
- **1.4 Output Formatting & Response:** Cleans AI output and sends the reply back to Telegram.
- **1.5 Memory Management:** Stores conversational context per chat session for coherent interactions.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

- **Overview:**  
  This block starts the workflow by listening to Telegram messages and determines whether the incoming message contains a YouTube link. It routes the flow accordingly.

- **Nodes Involved:**  
  - Trigger: Telegram Bot Message  
  - Check: Is YouTube Link?

- **Node Details:**

  - **Trigger: Telegram Bot Message**  
    - Type: Telegram Trigger  
    - Role: Listens for new messages sent to the bot on Telegram.  
    - Configuration: Watches for "message" updates only. Uses Telegram Bot OAuth2 credentials.  
    - Input: N/A (Webhook trigger)  
    - Output: Message JSON structure containing chat ID, message text, etc.  
    - Edge Cases: Telegram API downtime, invalid bot token, webhook misconfiguration.  
    - Notes: Requires valid Telegram Bot Token from @BotFather.

  - **Check: Is YouTube Link?**  
    - Type: IF node  
    - Role: Checks if the message text contains "https" indicating a link (simple heuristic for YouTube URLs).  
    - Configuration: Condition checks if the message text string contains substring "https". Case sensitive.  
    - Input: Output from Telegram Trigger  
    - Output: Two branches:  
      - True: Message contains a link → proceed to YouTube processing  
      - False: No link → proceed to normal chat AI processing  
    - Edge Cases: Messages with links other than YouTube; links without "https"; empty messages.

---

#### 1.2 YouTube Transcript Processing

- **Overview:**  
  When a YouTube link is detected, this block extracts the video ID from the URL, fetches the transcript via RapidAPI, and cleans up the transcript text for further processing.

- **Nodes Involved:**  
  - Extract Chat & Video ID  
  - Fetch YouTube Transcript (via RapidAPI)  
  - Clean Transcript Symbols

- **Node Details:**

  - **Extract Chat & Video ID**  
    - Type: Set node  
    - Role: Uses regex to extract YouTube video ID from the message text; also extracts Telegram chat ID for session tracking.  
    - Configuration:  
      - `videoId` extracted by matching regex `/(?:v=|youtu\.be\/)([a-zA-Z0-9_-]+)/` on the message text.  
      - `chatId` extracted from message JSON path.  
    - Input: Output of IF node (true branch)  
    - Output: JSON with `videoId` and `chatId` properties.  
    - Edge Cases: Malformed YouTube URLs, missing video ID, regex failures.

  - **Fetch YouTube Transcript (via RapidAPI)**  
    - Type: HTTP Request  
    - Role: Calls RapidAPI endpoint to get transcript for the extracted video ID.  
    - Configuration:  
      - URL constructed dynamically as `https://youtube-transcript3.p.rapidapi.com/api/transcript?videoId={{ $json.videoId }}`  
      - Headers include `X-RapidAPI-Key` and `X-RapidAPI-Host` (must be set with user credentials).  
    - Input: From "Extract Chat & Video ID"  
    - Output: JSON transcript array with text segments.  
    - Edge Cases: API key invalid or missing, rate limiting, video transcript unavailable or private, network timeouts.

  - **Clean Transcript Symbols**  
    - Type: Set node  
    - Role: Concatenates transcript text segments into a single string and replaces HTML entities with readable characters.  
    - Configuration:  
      - Joins transcript array texts with space.  
      - Replaces `&#39;` with `'`, `&quot;` with `"`, and `&amp;` with `&`.  
      - Stores result in `fullTranscript`.  
    - Input: Output of HTTP Request node  
    - Output: JSON with cleaned full transcript string.  
    - Edge Cases: Empty transcripts, unexpected HTML entities, malformed data.

---

#### 1.3 AI Chat & Summarization

- **Overview:**  
  This block sends either the cleaned transcript (for YouTube links) or the raw message text (for normal chats) to OpenAI GPT-4o via LangChain agent. It handles summarization, normal chat replies, and follow-up commands with session memory context.

- **Nodes Involved:**  
  - Simple Memory  
  - OpenAI Chat Model  
  - AI Chat & Summarizer Agent

- **Node Details:**

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational context per Telegram chat session ID, holding the last 10 interactions.  
    - Configuration:  
      - `sessionKey` set dynamically as the Telegram chat ID (`$('Check: Is YouTube Link?').item.json.message.chat.id`).  
      - Uses custom key sessionIdType.  
      - Context window length is 10 messages.  
    - Input: Connected as AI memory for the Agent.  
    - Output: Context data to AI agent.  
    - Edge Cases: Session key missing or invalid, memory overflow, concurrency issues.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4o model access for generating completions based on prompts and memory.  
    - Configuration:  
      - Model: "gpt-4o-mini" (a GPT-4o variant optimized for cost/performance).  
      - Credentials: OpenAI API key required.  
    - Input: Connected as language model for the Agent.  
    - Output: AI-generated chat or summary completions.  
    - Edge Cases: API rate limits, invalid API key, model availability.

  - **AI Chat & Summarizer Agent**  
    - Type: LangChain Agent  
    - Role: Central agent that receives either the user message or the full transcript, sends it to OpenAI for response generation, managing chat or summary context.  
    - Configuration:  
      - Text input is conditional: uses `message.text` if present; otherwise uses `fullTranscript`.  
      - System message instructs agent to:  
        - Chat normally or summarize YouTube videos.  
        - Not require transcript or link if not provided.  
        - Adjust existing summaries on follow-up requests instead of restarting.  
      - Prompt type: "define".  
    - Input: Receives from "Check: Is YouTube Link?" (no link branch) or "Clean Transcript Symbols" (link branch).  
    - Output: AI response with `output` field.  
    - Edge Cases: Missing inputs, prompt failures, API errors, long inputs exceeding model limits.

---

#### 1.4 Output Formatting & Response

- **Overview:**  
  Cleans the raw AI output formatting to remove unsupported markdown or special characters and sends the cleaned message back to the Telegram user.

- **Nodes Involved:**  
  - Clean AI Output Formatting  
  - Send Reply to Telegram

- **Node Details:**

  - **Clean AI Output Formatting**  
    - Type: Code (JavaScript)  
    - Role: Post-processes the AI response string to remove bold markers, markdown headers, bullet symbols, backticks, blockquotes, underscores, and normalize spacing/newlines.  
    - Configuration:  
      - Uses regex replacements to strip or replace markdown symbols.  
      - Collapses multiple newlines and spaces.  
      - Returns cleaned message as `cleanedMessage`.  
    - Input: AI Chat & Summarizer Agent output.  
    - Output: JSON with `cleanedMessage`.  
    - Edge Cases: Unexpected formatting, empty messages, encoding issues.

  - **Send Reply to Telegram**  
    - Type: Telegram node  
    - Role: Sends the cleaned reply text back to the Telegram chat from which the original message came.  
    - Configuration:  
      - Text: Uses `cleanedMessage` from previous node.  
      - Chat ID: Extracted from Telegram trigger node's message chat ID.  
      - Parse mode: Markdown (Telegram formatting).  
      - Credentials: Telegram Bot OAuth2 token required.  
    - Input: From "Clean AI Output Formatting".  
    - Output: Confirmation of message sent.  
    - Edge Cases: Invalid chat ID, Telegram API failures, message length limits.

---

#### 1.5 Memory Management

- **Overview:**  
  Manages session-based memory to maintain context for multi-turn conversations with users, enhancing the chatbot’s coherence over time.

- **Nodes Involved:**  
  - Simple Memory (also connected as memory for AI Chat & Summarizer Agent)

- **Node Details:**

  - Details are same as in 1.3. The memory buffer stores recent messages per chat session, allowing the AI agent to reference past inputs and outputs, enabling follow-up commands like "Make it shorter" or "Turn this into a LinkedIn post."

---

### 3. Summary Table

| Node Name                     | Node Type                                 | Functional Role                              | Input Node(s)                 | Output Node(s)                             | Sticky Note                                                                                                   |
|-------------------------------|-------------------------------------------|----------------------------------------------|------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Trigger: Telegram Bot Message  | Telegram Trigger                         | Incoming Telegram messages trigger            | N/A                          | Check: Is YouTube Link?                     |                                                                                                               |
| Check: Is YouTube Link?        | If Node                                  | Detects if message contains YouTube link     | Trigger: Telegram Bot Message | Extract Chat & Video ID / AI Chat & Summarizer Agent |                                                                                                               |
| Extract Chat & Video ID        | Set Node                                 | Extracts video ID and chat ID                  | Check: Is YouTube Link? (true) | Fetch YouTube Transcript (via RapidAPI)    |                                                                                                               |
| Fetch YouTube Transcript (via RapidAPI) | HTTP Request                    | Fetches transcript text from RapidAPI          | Extract Chat & Video ID       | Clean Transcript Symbols                   |                                                                                                               |
| Clean Transcript Symbols       | Set Node                                 | Cleans transcript text, replaces HTML entities | Fetch YouTube Transcript      | AI Chat & Summarizer Agent                 |                                                                                                               |
| AI Chat & Summarizer Agent     | LangChain Agent                          | Sends transcript or text to OpenAI for response | Clean Transcript Symbols / Check: Is YouTube Link? (false) | Clean AI Output Formatting              |                                                                                                               |
| Simple Memory                 | LangChain Memory Buffer Window           | Stores conversational context per chat session | Connected as memory to Agent  | Provides context to AI Agent                |                                                                                                               |
| OpenAI Chat Model              | LangChain OpenAI Chat Model               | Provides GPT-4o language model for Agent     | Connected as LM to Agent      | Provides completions to Agent               |                                                                                                               |
| Clean AI Output Formatting     | Code Node (JavaScript)                    | Cleans AI output formatting                    | AI Chat & Summarizer Agent    | Send Reply to Telegram                      |                                                                                                               |
| Send Reply to Telegram         | Telegram Node                            | Sends final cleaned message back to user      | Clean AI Output Formatting    | N/A                                        |                                                                                                               |
| Sticky Note                   | Sticky Note                              | Explains YouTube Link Handling flow           | N/A                          | N/A                                        | ## 🔗 YouTube Link Handling Flow: Extracts video ID, fetches transcript, cleans it, sends to OpenAI for summary |
| Sticky Note1                  | Sticky Note                              | Explains AI Chatbot + Summarizer block        | N/A                          | N/A                                        | ## 💬 AI Chatbot + Summarizer: Handles chat queries and summaries, maintains context, cleans output            |
| Sticky Note2                  | Sticky Note                              | Explains required API keys and setup           | N/A                          | N/A                                        | ## 🔐 Required API Keys & Setup: Telegram, OpenAI, RapidAPI keys and links                                      |
| Sticky Note3                  | Sticky Note                              | Summarizes overall workflow logic              | N/A                          | N/A                                        | ## 🧠 Workflow Logic Overview: Describes trigger, YouTube link handling, AI processing, and output             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Name: "Trigger: Telegram Bot Message"  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates only.  
   - Connect Telegram Bot OAuth2 credentials obtained from @BotFather.

2. **Add IF Node to Detect YouTube Link**  
   - Name: "Check: Is YouTube Link?"  
   - Type: IF Node  
   - Condition: Check if `{{$json.message.text}}` contains substring "https" (case sensitive).  
   - Connect input from Telegram Trigger node.

3. **Create Set Node to Extract Video and Chat ID**  
   - Name: "Extract Chat & Video ID"  
   - Type: Set Node  
   - Assignments:  
     - `videoId` = regex extract from `{{$json.message.text}}` using `/(?:v=|youtu\.be\/)([a-zA-Z0-9_-]+)/`  
     - `chatId` = `{{$json.message.chat.id}}`  
   - Connect from "Check: Is YouTube Link?" node, true branch.

4. **Add HTTP Request Node to Fetch Transcript**  
   - Name: "Fetch YouTube Transcript (via RapidAPI)"  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://youtube-transcript3.p.rapidapi.com/api/transcript?videoId={{ $json.videoId }}`  
   - Add Headers:  
     - `X-RapidAPI-Key`: your RapidAPI key  
     - `X-RapidAPI-Host`: your RapidAPI host, e.g., `youtube-transcript3.p.rapidapi.com`  
   - Connect from "Extract Chat & Video ID".

5. **Add Set Node to Clean Transcript Symbols**  
   - Name: "Clean Transcript Symbols"  
   - Type: Set Node  
   - Assignments:  
     - `fullTranscript` = join all transcript segment texts with spaces, then replace HTML entities: `&#39;` → `'`, `&quot;` → `"`, `&amp;` → `&`.  
   - Connect from "Fetch YouTube Transcript".

6. **Add LangChain Memory Node**  
   - Name: "Simple Memory"  
   - Type: LangChain Memory Buffer Window  
   - Set `sessionKey` to Telegram Chat ID: `={{ $('Check: Is YouTube Link?').item.json.message.chat.id }}`  
   - Context window length: 10 messages.

7. **Add LangChain OpenAI Chat Model Node**  
   - Name: "OpenAI Chat Model"  
   - Type: LangChain OpenAI Chat Model  
   - Model: Select "gpt-4o-mini".  
   - Attach OpenAI API credentials.

8. **Add LangChain Agent Node**  
   - Name: "AI Chat & Summarizer Agent"  
   - Type: LangChain Agent  
   - Text input expression: use `{{$json.message?.text ? $json.message.text : $json.fullTranscript}}`  
   - System message:  
     ```
     You are an assistant who chats with users and also summarizes YouTube videos when requested.

     IMPORTANT:
     1) Don't force the user to provide the transcript or youtube video link.

     If the user sends a YouTube link and asks for a summary, generate a concise summary of that video.

     Otherwise, reply normally and help with any other requests.
     Keep the conversation context.
     If the user requests modifications to a previous summary, adjust the existing summary rather than starting from scratch.
     ```  
   - Prompt type: define  
   - Connect OpenAI Chat Model as language model.  
   - Connect Simple Memory as memory.

   - Connect inputs:  
     - From "Clean Transcript Symbols" (YouTube link branch)  
     - From "Check: Is YouTube Link?" (No link branch, direct to agent for normal chat)

9. **Add Code Node to Clean AI Output Formatting**  
   - Name: "Clean AI Output Formatting"  
   - Type: Code (JavaScript)  
   - Paste the cleaning JavaScript code to remove markdown symbols, normalize spacing, and newlines as described in the workflow.  
   - Input: From "AI Chat & Summarizer Agent".

10. **Add Telegram Node to Send Reply**  
    - Name: "Send Reply to Telegram"  
    - Type: Telegram Node  
    - Text: Use `{{$json.cleanedMessage}}` from previous node.  
    - Chat ID: Use `{{$json.message.chat.id}}` from Telegram Trigger node.  
    - Parse mode: Markdown  
    - Attach Telegram Bot credentials.

11. **Connect Workflow Edges**  
    - Telegram Trigger → Check YouTube Link  
    - Check YouTube Link (true) → Extract Chat & Video ID → Fetch Transcript → Clean Transcript → AI Agent  
    - Check YouTube Link (false) → AI Agent  
    - AI Agent → Clean AI Output → Send Reply to Telegram  

12. **Set Sticky Notes**  
    - Add sticky notes with explanations for:  
      - YouTube link handling flow.  
      - AI chat + summarizer process.  
      - Required API keys and setup instructions.  
      - Overall workflow logic overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                     | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Telegram Bot Token required. Get your bot token from [@BotFather](https://t.me/BotFather).                                                       | Telegram Bot setup                                                                                      |
| OpenAI API Key required. Generate your key from https://platform.openai.com/account/api-keys                                                     | OpenAI GPT-4o API access                                                                                |
| Groq API can be used as a free alternative to OpenAI: https://console.groq.com                                                                   | Alternative LLM provider                                                                                |
| RapidAPI credentials needed to access YouTube transcript API: `X-RapidAPI-Key` and `X-RapidAPI-Host`, typically `youtube-transcript3.p.rapidapi.com` | YouTube Transcript fetching                                                                             |
| The workflow supports follow-up commands like "Make it shorter", "Turn this into a LinkedIn post", or "Create a tweet thread"                   | Advanced user interactions                                                                              |
| The memory buffer window holds last 10 messages per user to maintain context across conversations                                               | Conversation context management                                                                         |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow. This processing strictly respects all current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---