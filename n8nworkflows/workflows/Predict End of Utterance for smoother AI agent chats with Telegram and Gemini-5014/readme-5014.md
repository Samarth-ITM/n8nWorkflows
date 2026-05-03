Predict End of Utterance for smoother AI agent chats with Telegram and Gemini

https://n8nworkflows.xyz/workflows/predict-end-of-utterance-for-smoother-ai-agent-chats-with-telegram-and-gemini-5014


# Predict End of Utterance for smoother AI agent chats with Telegram and Gemini

### 1. Workflow Overview

This workflow aims to improve conversational AI interactions on chat platforms, specifically Telegram, by intelligently predicting when a user has finished their utterance before the AI agent responds. This prevents premature replies and creates a more natural, human-like chat experience.

The workflow achieves this by buffering incoming messages from a user, using AI-powered classification to predict whether the user’s message is complete, and only responding once the end of the utterance is detected. It uses Redis for session state management to track ongoing conversations and pauses/resumes execution intelligently based on user activity.

**Logical blocks:**

- **1.1 Input Reception**: Capture incoming Telegram messages and extract relevant fields.
- **1.2 Session & State Management**: Use Redis to track whether a user’s message is part of a new or ongoing utterance.
- **1.3 End of Utterance Prediction Loop**: Employ an AI text classifier (powered by Google Gemini) to decide if the user is done speaking or typing, with wait nodes to pause and resume processing.
- **1.4 AI Agent Response Generation**: Once end is detected, combine buffered messages and send to an AI agent for reply generation.
- **1.5 Response Delivery & Cleanup**: Send the AI-generated reply back to the user via Telegram and clear session state to prepare for the next utterance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming Telegram messages, filters out bot commands, and prepares message data for processing.

**Nodes Involved:**  
- Telegram Trigger  
- Is Bot command?  
- Get Values  
- Call Prediction Subworkflow  

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger node  
  - *Role:* Listens for new Telegram messages (`updates: message`)  
  - *Credentials:* Uses Telegram API credentials for the bot  
  - *Input/Output:* No input; outputs incoming message JSON  
  - *Edge Cases:* Telegram API rate limits or dropped webhooks may disrupt message reception  

- **Is Bot command?**  
  - *Type:* If node  
  - *Role:* Checks if the message text starts with "/" to filter out bot commands (which are ignored in this workflow)  
  - *Expression:* Checks `$json.message.text` starts with "/"  
  - *Connections:* If true (bot command), no further processing; else passes message to next node  
  - *Edge Cases:* Empty or malformed messages could cause expression failures  

- **Get Values**  
  - *Type:* Set node  
  - *Role:* Extracts and stores key message properties: `from` (chat id), `message` (text), `createdAt` (current timestamp)  
  - *Expressions:*  
    - `from` = `$json.message.chat.id`  
    - `message` = `$json.message.text`  
    - `createdAt` = current ISO timestamp `$now.toISO()`  
  - *Output:* Structured JSON with these fields for downstream use  

- **Call Prediction Subworkflow**  
  - *Type:* Execute Workflow node (calls itself as subworkflow)  
  - *Role:* Passes extracted message data to the prediction logic via subworkflow trigger  
  - *Parameters:* Inputs: `from`, `message`, `createdAt`  
  - *Execution:* Asynchronous (does not wait for subworkflow to finish), allowing parallel handling of messages  
  - *Edge Cases:* Recursive calls must be carefully managed to avoid infinite loops  

---

#### 1.2 Session & State Management

**Overview:**  
Manages session data in Redis to track conversation state and message history for each user, determining if a message starts a new utterance or continues an existing one.

**Nodes Involved:**  
- Prediction Subworkflow (sub-workflow trigger)  
- Get Session  
- Update Session  
- Get Updated Session Values  
- is New Session?  
- Update Session1  
- Update Session2  
- Session Ref  

**Node Details:**

- **Prediction Subworkflow**  
  - *Type:* Execute Workflow Trigger node (subworkflow entry point)  
  - *Role:* Receives message data, initiates session check process  

- **Get Session**  
  - *Type:* Redis node (get)  
  - *Role:* Retrieves existing session data for user key: `session/{{ from }}`  
  - *Output:* Session data JSON or empty if none found  
  - *Edge Cases:* Redis connectivity issues, missing keys  

- **is New Session?**  
  - *Type:* If node  
  - *Role:* Determines if session is new by checking absence of a `state` field in session data  
  - *Expression:* `!$json.data?.state`  
  - *Branches:*  
    - Yes: start new session flow  
    - No: continue existing session  

- **Update Session1**  
  - *Type:* Redis node (set)  
  - *Role:* Initializes a new session with state `waiting` and stores `resumeUrl` for wait node resumption  
  - *Key:* `session/{{ from }}`  
  - *Value:* JSON with `state: 'waiting'` and `resumeUrl` from execution context  
  - *Edge Cases:* Redis write errors, invalid resume URLs  

- **Update Session**  
  - *Type:* Redis node (set)  
  - *Role:* Updates the session data with new message appended to the `messages` array  
  - *Key:* `session/{{ from }}`  
  - *Value:* Merges existing messages with new message, stores as JSON string  
  - *Edge Cases:* JSON parsing errors, data corruption  

- **Get Updated Session Values**  
  - *Type:* Set node  
  - *Role:* Prepares updated session data object with appended message for Redis update  
  - *Expression:* Combines old messages + new message into JSON string under `messages`  

- **Update Session2**  
  - *Type:* Redis node (delete)  
  - *Role:* Deletes session key after utterance is complete and response sent  
  - *Edge Cases:* Redis delete failures  

- **Session Ref**  
  - *Type:* NoOp node  
  - *Role:* Helper node to pass session reference data downstream  

---

#### 1.3 End of Utterance Prediction Loop

**Overview:**  
Uses an AI text classifier to predict if the user’s message is complete or if more input is expected. If incomplete, waits for further messages; if complete, triggers response generation.

**Nodes Involved:**  
- Predict End of Utterance  
- Wait For Webhook  
- Resume Wait Node  
- Is Waiting?  
- Session Ended?  

**Node Details:**

- **Predict End of Utterance**  
  - *Type:* LangChain Text Classifier node  
  - *Role:* Classifies combined buffered messages to predict “end_of_utterance” or “has_more_to_follow”  
  - *Input:* Concatenated messages with timestamps from session, formatted as plain text for classifier  
  - *Prompt:* Custom system prompt explaining heuristics and examples for utterance end prediction  
  - *Model:* Google Gemini Chat Model1 (Gemini-2.5-flash-lite-preview-06-17) as AI language model  
  - *Output:* Prediction category  
  - *Branches:*  
    - End detected: passes to AI Agent for response  
    - More to follow: triggers Wait For Webhook node to pause workflow  
  - *Edge Cases:* Model latency, classification errors, input message formatting issues  

- **Wait For Webhook**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow execution until resumed by webhook or timeout (20 seconds)  
  - *Resume Trigger:* HTTP request to stored `resumeUrl` in Redis session  
  - *Edge Cases:* Timeout before resume, lost webhook calls  

- **Resume Wait Node**  
  - *Type:* HTTP Request node  
  - *Role:* Sends HTTP request to resume paused wait node using stored `resumeUrl`  
  - *Input:* URL from session data  
  - *Edge Cases:* HTTP failures, invalid URLs  

- **Is Waiting?**  
  - *Type:* If node  
  - *Role:* Checks if session state is `waiting` to decide whether to resume the wait node or start new prediction  
  - *Expression:* `$json.data?.state === "waiting"`  

- **Session Ended?**  
  - *Type:* If node  
  - *Role:* Checks if session data is empty, signaling no ongoing session and that agent response should be triggered  
  - *Expression:* Checks if session data object is empty (`{}`)  

---

#### 1.4 AI Agent Response Generation

**Overview:**  
After detecting the end of the user’s utterance, this block sends the buffered conversation messages to an AI agent, which generates a reply.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Redis Chat Memory  

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Receives combined conversation messages, generates AI reply in plain text  
  - *Input:* Merged messages from session, extracted and joined as plain text  
  - *System Message:* Instructs agent to respond without markdown formatting  
  - *Model:* Google Gemini Chat Model (Gemini-2.5-flash-preview-05-20)  
  - *Memory:* Uses Redis Chat Memory node to maintain chat context window of last 10 messages  
  - *Output:* Text reply for user  
  - *Edge Cases:* Model latency, API quota limits, memory syncing issues  

- **Google Gemini Chat Model**  
  - *Type:* LangChain LM Chat node  
  - *Role:* Underlying large language model used by AI Agent for response generation  

- **Redis Chat Memory**  
  - *Type:* LangChain Redis Memory node  
  - *Role:* Manages conversational context with Redis, storing last 10 messages per user session  

---

#### 1.5 Response Delivery & Cleanup

**Overview:**  
Sends the AI-generated response back to the user via Telegram and clears the Redis session to prepare for a new utterance.

**Nodes Involved:**  
- Respond to User  
- Update Session2  

**Node Details:**

- **Respond to User**  
  - *Type:* Telegram node  
  - *Role:* Sends the AI agent’s response text to the user’s Telegram chat ID  
  - *Parameters:*  
    - `chatId`: From session `from`  
    - `text`: AI agent’s output  
    - `parse_mode`: HTML (for formatting)  
  - *Credentials:* Telegram API credentials  
  - *Edge Cases:* Telegram API errors, user blocked bot  

- **Update Session2**  
  - *Type:* Redis node (delete)  
  - *Role:* Deletes the session key, clearing buffered messages and state for next utterance  

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                            | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                              |
|-------------------------|--------------------------------------------|-------------------------------------------|-------------------------|--------------------------|---------------------------------------------------------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger                           | Receive incoming Telegram messages        | —                       | Is Bot command?          | ## 1. Accept Telegram messages [Read more](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger) |
| Is Bot command?         | If                                         | Filter bot commands                        | Telegram Trigger         | Get Values               |                                                                                                         |
| Get Values              | Set                                        | Extract message fields                     | Is Bot command?          | Call Prediction Subworkflow |                                                                                                         |
| Call Prediction Subworkflow | Execute Workflow                      | Call subworkflow with message data        | Get Values               | —                        |                                                                                                         |
| Prediction Subworkflow  | Execute Workflow Trigger                    | Entry point for prediction logic subworkflow | Call Prediction Subworkflow | Get Session             | ## 2. Determine If New Message [Learn more](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflowtrigger) |
| Get Session             | Redis (get)                                | Retrieve session data for user             | Prediction Subworkflow   | Update Session           |                                                                                                         |
| Update Session          | Redis (set)                                | Append new message to session              | Get Session              | Get Updated Session Values |                                                                                                         |
| Get Updated Session Values | Set                                      | Prepare updated session data               | Update Session           | is New Session?           |                                                                                                         |
| is New Session?         | If                                         | Check if session is new                     | Get Updated Session Values | Session Ref / Is Waiting? |                                                                                                         |
| Session Ref             | NoOp                                       | Pass session reference                      | is New Session? / Session Ended? | Update Session1 / Update Session2 |                                                                                                         |
| Update Session1         | Redis (set)                                | Initialize new session with waiting state | Session Ref              | Predict End of Utterance  |                                                                                                         |
| Predict End of Utterance | LangChain Text Classifier                  | Predict if user message is complete        | Update Session1          | AI Agent / Wait For Webhook | ## 2. Start of a New Utterance? Start the Prediction Wait Loop. [Learn more](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.text-classifier) |
| Wait For Webhook        | Wait                                       | Pause workflow waiting for new input or resume | Predict End of Utterance | Get Session2              |                                                                                                         |
| Get Session2            | Redis (get)                                | Retrieve session after wait                 | Wait For Webhook         | Session Ended?            |                                                                                                         |
| Session Ended?          | If                                         | Check if session ended                       | Get Session2             | AI Agent / Session Ref    |                                                                                                         |
| AI Agent                | LangChain Agent                            | Generate AI response based on buffered messages | Session Ended?          | Update Session2           | ## 4. Send Reply Only When End of Utterance is Detected [Read more](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) |
| Update Session2         | Redis (delete)                             | Clear session after response sent           | AI Agent                 | Respond to User           |                                                                                                         |
| Respond to User         | Telegram                                   | Send AI response text to user                | Update Session2          | —                        |                                                                                                         |
| Resume Wait Node        | HTTP Request                               | Resume paused wait node                      | Is Waiting?              | —                        | ## 3. Prediction Loop Already Running? Let's Resume! [Read more](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/) |
| Is Waiting?             | If                                         | Check if session is waiting state            | is New Session?          | Resume Wait Node / Predict End of Utterance |                                                                                                         |
| Google Gemini Chat Model | LangChain LM Chat                         | LLM for AI Agent                            | —                       | AI Agent                 |                                                                                                         |
| Google Gemini Chat Model1 | LangChain LM Chat                        | LLM for Text Classifier                      | —                       | Predict End of Utterance  |                                                                                                         |
| Redis Chat Memory       | LangChain Redis Memory                     | Maintains chat context                       | —                       | AI Agent                 |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Set to listen for `message` updates.  
   - Connect Telegram API credentials (bot token).  
   - Position at workflow start.

2. **Add If node “Is Bot command?”**  
   - Condition: `$json.message.text` starts with "/" (bot commands).  
   - True branch: no further processing; False branch: proceed.

3. **Add Set node “Get Values”**  
   - Extract fields:  
     - `from` = `$json.message.chat.id`  
     - `message` = `$json.message.text`  
     - `createdAt` = current ISO timestamp `$now.toISO()`  
   - Connect from “Is Bot command?” false branch.

4. **Add Execute Workflow node “Call Prediction Subworkflow”**  
   - Set to call the same workflow by ID (self-call) asynchronously (no wait).  
   - Pass inputs: `from`, `message`, `createdAt` from “Get Values”.  
   - Connect from “Get Values”.

5. **Add Execute Workflow Trigger node “Prediction Subworkflow”**  
   - Configure inputs: `from`, `message`, `createdAt`.  
   - This node is the subworkflow entry point for prediction logic.

6. **Add Redis node “Get Session” (operation: get)**  
   - Key: `session/{{ $json.from }}`  
   - Retrieves existing session data.  
   - Connect from “Prediction Subworkflow”.

7. **Add If node “is New Session?”**  
   - Condition: check if session data has no `state` property.  
   - True: new session flow; False: existing session flow.

8. **Add Redis node “Update Session1” (operation: set)**  
   - Key: `session/{{ $json.from }}`  
   - Value: JSON with `state: "waiting"` and `resumeUrl` from execution context.  
   - Connect from “Session Ref” (which gets data from true branch of “is New Session?”).

9. **Add LangChain LM Chat node “Google Gemini Chat Model1”**  
   - Model: `models/gemini-2.5-flash-lite-preview-06-17`  
   - Connect as AI language model of “Predict End of Utterance”.

10. **Add LangChain Text Classifier node “Predict End of Utterance”**  
    - Input text: combined buffered messages with timestamps from session.  
    - System prompt: detailed instructions for end-of-utterance prediction.  
    - Categories: “end_of_utterance” and “has_more_to_follow”.  
    - Connect from “Update Session1” for new sessions and from “Is Waiting?” for ongoing sessions.

11. **Add Wait node “Wait For Webhook”**  
    - Resume mode: webhook  
    - Timeout: 20 seconds limit  
    - Connect from “Predict End of Utterance” branch for “has_more_to_follow”.

12. **Add Redis node “Get Session2” (operation: get)**  
    - Key: `session/{{ $json.from }}`  
    - Connect from “Wait For Webhook”.

13. **Add If node “Session Ended?”**  
    - Condition: session data is empty (no messages).  
    - True branch: proceed to AI Agent.  
    - False branch: loop back to “Session Ref”.

14. **Add LangChain Redis Memory node “Redis Chat Memory”**  
    - Session key: `session/{{ $json.from }}/agent`  
    - Context window length: 10 messages  
    - Connect as AI memory for “AI Agent”.

15. **Add LangChain LM Chat node “Google Gemini Chat Model”**  
    - Model: `models/gemini-2.5-flash-preview-05-20`  
    - Connect as AI language model for “AI Agent”.

16. **Add LangChain Agent node “AI Agent”**  
    - Input text: concatenated buffered messages (plain text).  
    - System message: instruct no markdown formatting.  
    - Connect from “Session Ended?” true branch.

17. **Add Redis node “Update Session” (operation: set)**  
    - Append new message to messages array in session.  
    - Connect from “Get Session”.

18. **Add Set node “Get Updated Session Values”**  
    - Prepare updated session data with appended new message.  
    - Connect from “Update Session”.

19. **Add Redis node “Update Session2” (operation: delete)**  
    - Delete session key after utterance completion.  
    - Connect from “AI Agent”.

20. **Add Telegram node “Respond to User”**  
    - Chat ID: from session `from`  
    - Text: AI agent output  
    - Parse mode: HTML  
    - Connect from “Update Session2”.

21. **Add HTTP Request node “Resume Wait Node”**  
    - URL: from session `resumeUrl`  
    - Connect from “Is Waiting?” true branch.

22. **Add If node “Is Waiting?”**  
    - Condition: session state == “waiting”  
    - True: connect to “Resume Wait Node”  
    - False: connect to “Predict End of Utterance”.

23. **Add NoOp node “Session Ref”**  
    - Helper to pass session data downstream where needed.

24. **Establish all connections as per logical flow described in the above nodes.**

25. **Configure Credentials:**  
    - Telegram API for Telegram Trigger and Respond to User nodes.  
    - Redis credentials for session management nodes.  
    - Google Palm API credentials for LangChain Google Gemini LM nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This template works only with chat systems where not every incoming message requires an immediate reply. Telegram is used here for demonstration purposes, capturing many messages from a single user without blocking execution.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | [Telegram Trigger Documentation](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger) |
| The workflow buffers presumed partial messages until the AI classifier predicts the user has finished their message. Redis is used for session/state management to track progress.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | [Subworkflow Trigger Documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflowtrigger) |
| The prediction wait loop can resume if already running using an HTTP request to the stored resume URL, avoiding multiple simultaneous prediction loops.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | [HTTP Request Node Documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/) |
| The AI agent only sends a reply when the end of utterance is detected, resulting in more natural conversations without artificial delays.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | [AI Agent Node Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) |
| The approach demonstrated uses Google Gemini LLMs for prediction and response generation, Redis for session management, and Telegram as the chat platform. It aims to reduce interruptions by predicting utterance completion using AI classification rather than fixed timeouts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | —                                                                                                           |
| For improved prediction accuracy, additional heuristic or code-based checks can supplement the AI classifier. Fast and accurate LLMs (like Gemini-2.5-flash-lite) are recommended to minimize user wait time.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | —                                                                                                           |
| Need help or want to discuss this workflow? Join the [Discord](https://discord.com/invite/XPKeKXeB7d) or [n8n Community Forum](https://community.n8n.io/)!                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)                          |
| Previous similar techniques include buffering messages by fixed time intervals, but this AI-driven method is more adaptive and contextual.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | [Example buffering workflow](https://n8n.io/workflows/2346-enhance-customer-chat-by-buffering-messages-with-twilio-and-redis/) |

---

**Disclaimer:**  
The content analyzed and documented is derived solely from an automated n8n workflow. All data handled is legal and public. The workflow complies strictly with content policies and does not include illegal, offensive, or protected material.