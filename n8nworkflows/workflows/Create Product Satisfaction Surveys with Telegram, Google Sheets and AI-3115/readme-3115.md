Create Product Satisfaction Surveys with Telegram, Google Sheets and AI

https://n8nworkflows.xyz/workflows/create-product-satisfaction-surveys-with-telegram--google-sheets-and-ai-3115


# Create Product Satisfaction Surveys with Telegram, Google Sheets and AI

### 1. Workflow Overview

This workflow implements a Product Satisfaction Survey chatbot using Telegram, Google Sheets, Redis, and AI. It targets use cases where structured survey questions are asked sequentially, but with the flexibility to include AI-driven follow-up questions for deeper insights based on user responses.

The workflow is logically divided into these blocks:

- **1.1 Initiate Survey & User Interaction:** Handles incoming Telegram messages, recognizes bot commands, and starts or resets survey sessions.
- **1.2 Survey State Management:** Uses Redis to track the user's current question index and session state.
- **1.3 Question Retrieval & Delivery:** Pulls survey questions from Google Sheets and sends them to the user via Telegram.
- **1.4 Response Handling & Storage:** Captures user answers, updates Google Sheets, and manages session state updates.
- **1.5 AI Follow-Up Logic:** Uses a text classifier to decide if a follow-up question is needed, and if so, engages an AI agent to conduct a mini conversation.
- **1.6 AI Agent Memory Management:** Resets and manages AI chat memory per question to avoid context bleed and hallucinations.
- **1.7 Survey Completion & Restart:** Detects when all questions are answered and informs the user, allowing survey restart.

---

### 2. Block-by-Block Analysis

#### 1.1 Initiate Survey & User Interaction

- **Overview:**  
  This block listens for Telegram messages, extracts commands, and routes the flow based on recognized commands (`/start`, `/next`, `/reset`). It initiates new sessions or handles unrecognized commands gracefully.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Set Variables1  
  - Execution Data2  
  - Message Type1 (Switch)  
  - Get Command1 (Set)  
  - Bot Actions1 (Switch)  
  - Create Record1 (Google Sheets)  
  - Start Session1 (Redis)  
  - Send Start (Telegram)  
  - Send Start1 (Telegram)  

- **Node Details:**  
  - **Telegram Trigger:**  
    - Type: Trigger node for Telegram messages.  
    - Configured to listen for "message" updates only.  
    - Uses Telegram API credentials for the bot.  
    - Input: Incoming Telegram messages.  
    - Output: Message JSON including chat and user info.  
    - Edge cases: Telegram API downtime, webhook misconfiguration.

  - **Set Variables1:**  
    - Type: Set node to define constants like survey title, Google Sheet ID, and Redis cache key based on session ID.  
    - Key expressions: `cacheKey` uses `survey_user_{{ $json.sessionId }}` pattern.  
    - Input: Telegram Trigger output.  
    - Output: Variables for downstream nodes.

  - **Execution Data2:**  
    - Type: Execution Data node to save metadata about the job (e.g., jobType, gsheetId, title, fromId).  
    - Requires Community+ license.  
    - Input: Set Variables1 output.  
    - Output: Passes data forward.

  - **Message Type1 (Switch):**  
    - Type: Switch node to classify incoming messages as bot commands or normal messages.  
    - Conditions check if message text is in ['/start', '/next', '/reset'].  
    - Outputs: `is_bot_command` or `is_normal_message`.  
    - Input: Get State2 output (Redis get).  
    - Output: Routes to Get Command1 or Get Record1.

  - **Get Command1 (Set):**  
    - Type: Set node extracting the command text from the Telegram message entity.  
    - Uses expression to slice command from message text.  
    - Input: Message Type1 output.  
    - Output: Command string for Bot Actions1.

  - **Bot Actions1 (Switch):**  
    - Type: Switch node routing based on command string.  
    - Recognizes `/start` (new_session), `/next` (next_question), else fallback.  
    - Input: Get Command1 output.  
    - Output: Routes to Create Record1, Get State3, or Send Start1.

  - **Create Record1 (Google Sheets):**  
    - Type: Google Sheets node to append or update a user record with ID (Telegram user ID).  
    - Uses mapping mode to match on ID column.  
    - Input: Bot Actions1 new_session output.  
    - Output: Starts session in Redis.

  - **Start Session1 (Redis):**  
    - Type: Redis node to set initial session state (has_session, session_createdAt, current_question_idx=0).  
    - Input: Create Record1 output.  
    - Output: Sends welcome message.

  - **Send Start (Telegram):**  
    - Type: Telegram node sending welcome and instructions message.  
    - Input: Start Session1 output.  
    - Output: Ends this flow branch.

  - **Send Start1 (Telegram):**  
    - Type: Telegram node sending unrecognized command message.  
    - Input: Bot Actions1 fallback output.  
    - Output: Ends this flow branch.

- **Edge Cases:**  
  - User sends unknown command → handled by Send Start1.  
  - Telegram API errors or rate limits.  
  - Redis connection failures.  
  - Google Sheets API quota exceeded.

---

#### 1.2 Survey State Management

- **Overview:**  
  Manages user session state and current question index using Redis. Retrieves and updates state to track survey progress.

- **Nodes Involved:**  
  - Get State2 (Redis)  
  - Get State3 (Redis)  
  - Increment Index1 (Redis)  
  - Is Survey Continue? (If)  

- **Node Details:**  
  - **Get State2:**  
    - Redis get operation using the user's cacheKey.  
    - Retrieves current session state for message type classification.  
    - Input: Execution Data2 output.  
    - Output: JSON with session data.

  - **Get State3:**  
    - Redis get operation to fetch current question index and session data.  
    - Input: Create Record2 or Bot Actions1 output.  
    - Output: Used to calculate next question and survey completion.

  - **Increment Index1:**  
    - Redis set operation to update current_question_idx based on next_question_idx and total questions.  
    - Ensures index does not exceed number of questions.  
    - Input: Get Survey State1 output.  
    - Output: Used by Is Survey Continue? node.

  - **Is Survey Continue? (If):**  
    - Checks if survey is complete by comparing current_question_idx with total questions.  
    - Routes to Reset Agent Memory1 if survey continues, or Completed Survey if done.

- **Edge Cases:**  
  - Redis connection errors or timeouts.  
  - Corrupted or missing session data.  
  - Race conditions if multiple messages arrive simultaneously.

---

#### 1.3 Question Retrieval & Delivery

- **Overview:**  
  Retrieves survey questions from Google Sheets and sends the next question to the user via Telegram.

- **Nodes Involved:**  
  - Get Columns1 (Google Sheets)  
  - Get Survey State1 (Set)  
  - Send Next Question (Telegram)  

- **Node Details:**  
  - **Get Columns1:**  
    - Reads the first row (questions) from the Google Sheet specified by gsheetId.  
    - Input: Get State3 output.  
    - Output: JSON object with question columns.

  - **Get Survey State1:**  
    - Calculates survey metadata: title, number of questions, next question index, and completion status.  
    - Uses expressions to compute next_question_idx and is_survey_complete.  
    - Input: Get Columns1 output.  
    - Output: Used to increment index and send next question.

  - **Send Next Question:**  
    - Sends the next question text to the user via Telegram.  
    - Constructs message with question number and question text from Google Sheets columns.  
    - Input: Reset Agent Memory1 output.  
    - Output: Ends this step.

- **Edge Cases:**  
  - Google Sheets API quota or permission errors.  
  - Missing or malformed questions in sheet.  
  - Telegram API failures.

---

#### 1.4 Response Handling & Storage

- **Overview:**  
  Captures user responses, appends them to the Google Sheet, and updates session state accordingly.

- **Nodes Involved:**  
  - Get Record1 (Google Sheets)  
  - Has No Record? (If)  
  - Create Record2 (Google Sheets)  
  - Get Last Bot Message1 (Redis)  
  - Append Responses1 (Set)  
  - Update Answer3 (HTTP Request)  
  - Update Answer2 (HTTP Request)  

- **Node Details:**  
  - **Get Record1:**  
    - Retrieves the user's existing survey record from Google Sheets by matching Telegram user ID.  
    - Input: Message Type1 output.  
    - Output: Passes record or empty.

  - **Has No Record? (If):**  
    - Checks if the user record exists.  
    - If no record, creates one with Create Record2.  
    - If record exists, proceeds to get last bot message.

  - **Create Record2:**  
    - Appends or updates the user record in Google Sheets with user ID.  
    - Input: Has No Record? output.  
    - Output: Fetches state after creation.

  - **Get Last Bot Message1:**  
    - Retrieves last bot message from Redis chat history to provide context for follow-up classification.  
    - Input: Has No Record? output.  
    - Output: Used by Append Responses1.

  - **Append Responses1:**  
    - Prepares payload combining the question, user answer, and agent response placeholders.  
    - Calculates the cell in Google Sheets to update based on question index.  
    - Input: Get Last Bot Message1 output.  
    - Output: Used by Update Answer nodes.

  - **Update Answer3:**  
    - Updates Google Sheet cell with user response only (used when no follow-up is needed).  
    - HTTP PUT request to Google Sheets API with RAW input option.  
    - Input: Should Follow Up?1 output (no follow-up branch).  
    - Output: Triggers Get State3 to refresh state.

  - **Update Answer2:**  
    - Updates Google Sheet cell with user response plus AI agent's follow-up answer.  
    - HTTP PUT request to Google Sheets API with RAW input option.  
    - Input: Interview Agent1 output (follow-up branch).  
    - Output: Sends response back to user.

- **Edge Cases:**  
  - Google Sheets API rate limits or permission errors.  
  - Redis failures retrieving chat history.  
  - HTTP request failures or malformed payloads.

---

#### 1.5 AI Follow-Up Logic

- **Overview:**  
  Determines whether a follow-up question is needed based on user response and last bot message, then engages an AI agent for deeper conversation if required.

- **Nodes Involved:**  
  - Should Follow Up?1 (Text Classifier)  
  - Interview Agent1 (AI Agent)  
  - Model2 (OpenAI GPT-4o-mini)  
  - Model3 (OpenAI GPT-4o-mini)  

- **Node Details:**  
  - **Should Follow Up?1:**  
    - Text Classifier node analyzing the last bot message and user response to categorize if follow-up is needed.  
    - Categories: `should_ask_followup_questions` and `should_not_ask_followup_questions`.  
    - Input: Append Responses1 output (payload with last bot message and user text).  
    - Output: Routes to Update Answer3 (no follow-up) or Interview Agent1 (follow-up).

  - **Interview Agent1:**  
    - AI Agent node configured to converse with the user for follow-up questions.  
    - Uses Model2 (GPT-4o-mini) as language model.  
    - Input: Telegram Trigger message text.  
    - Output: AI-generated follow-up response.

  - **Model2 & Model3:**  
    - OpenAI GPT-4o-mini models used for generating AI responses and classification.  
    - Model3 is used by Should Follow Up?1 node.  
    - Model2 is used by Interview Agent1 node.

- **Edge Cases:**  
  - OpenAI API rate limits or errors.  
  - AI hallucinations or irrelevant follow-ups.  
  - Network issues causing timeouts.

---

#### 1.6 AI Agent Memory Management

- **Overview:**  
  Resets AI chat memory for each new question to prevent context bleed and hallucinations, ensuring focused conversations.

- **Nodes Involved:**  
  - Memory3 (Redis Chat Memory)  
  - Memory4 (Redis Chat Memory)  
  - Reset Agent Memory1 (Memory Manager)  

- **Node Details:**  
  - **Memory3 & Memory4:**  
    - Redis-based chat memory nodes storing conversation history with context window lengths of 10 and 100 respectively.  
    - Memory3 feeds Reset Agent Memory1; Memory4 feeds Interview Agent1.  
    - Input: Redis credentials required.  
    - Output: Provides context for AI agent.

  - **Reset Agent Memory1:**  
    - Memory Manager node configured in "insert" mode with override to clear previous messages and insert only the current question prompt.  
    - Input: Is Survey Continue? output (survey ongoing).  
    - Output: Triggers Send Next Question.

- **Edge Cases:**  
  - Redis connection failures.  
  - Memory overflow or truncation issues.  
  - AI agent losing context if memory reset fails.

---

#### 1.7 Survey Completion & Restart

- **Overview:**  
  Detects when the survey is complete and sends a completion message with instructions to restart.

- **Nodes Involved:**  
  - Completed Survey (Telegram)  

- **Node Details:**  
  - **Completed Survey:**  
    - Sends a Telegram message thanking the user and instructing to use `/start` to restart the survey.  
    - Input: Is Survey Continue? output (survey complete).  
    - Output: Ends the survey session.

- **Edge Cases:**  
  - Telegram API failures.  
  - User ignoring restart instructions.

---

### 3. Summary Table

| Node Name            | Node Type                          | Functional Role                              | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                  |
|----------------------|----------------------------------|----------------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger      | Telegram Trigger                 | Entry point, listens for Telegram messages   |                            | Set Variables1             | ## 1. Initiate Survey by Inviting User to Chat [Learn more about the chat trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chattrigger/) |
| Set Variables1        | Set                             | Defines survey title, sheet ID, cache key    | Telegram Trigger           | Execution Data2            |                                                                                                              |
| Execution Data2       | Execution Data                  | Saves job metadata                            | Set Variables1             | Get State2                 |                                                                                                              |
| Get State2            | Redis                          | Gets session state for message classification| Execution Data2            | Message Type1              |                                                                                                              |
| Message Type1         | Switch                         | Classifies message as bot command or normal | Get State2                 | Get Command1, Get Record1  |                                                                                                              |
| Get Command1          | Set                            | Extracts command text from message            | Message Type1              | Bot Actions1               |                                                                                                              |
| Bot Actions1          | Switch                         | Routes based on command (/start, /next, etc.)| Get Command1               | Create Record1, Get State3, Send Start1 | ## 2. Handle Bot Commands [Learn more about the switch node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch/) |
| Create Record1        | Google Sheets                  | Creates or updates user record                | Bot Actions1 (new_session) | Start Session1             |                                                                                                              |
| Start Session1        | Redis                          | Initializes session state                      | Create Record1             | Send Start                 |                                                                                                              |
| Send Start            | Telegram                       | Sends welcome and instructions message       | Start Session1             |                            |                                                                                                              |
| Send Start1           | Telegram                       | Sends unrecognized command message            | Bot Actions1 (fallback)    |                            |                                                                                                              |
| Get Record1           | Google Sheets                  | Retrieves user record                          | Message Type1 (normal_msg) | Has No Record?             |                                                                                                              |
| Has No Record?        | If                             | Checks if user record exists                   | Get Record1                | Create Record2, Get Last Bot Message1 |                                                                                                              |
| Create Record2        | Google Sheets                  | Creates user record if none exists             | Has No Record?             | Get State3                 |                                                                                                              |
| Get Last Bot Message1 | Redis                          | Retrieves last bot message from chat history  | Has No Record?             | Append Responses1          |                                                                                                              |
| Append Responses1     | Set                            | Prepares payload and cell reference for update| Get Last Bot Message1      | Should Follow Up?1         |                                                                                                              |
| Should Follow Up?1    | Text Classifier                | Decides if follow-up question is needed       | Append Responses1          | Update Answer3, Interview Agent1 | ## 3. Support to Follow-Up Questions [Learn more about the Text Classifier node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.text-classifier/) |
| Update Answer3        | HTTP Request                  | Updates Google Sheet with user response only  | Should Follow Up?1 (no)    | Get State3                 |                                                                                                              |
| Interview Agent1      | AI Agent                      | Conducts AI-driven follow-up conversation     | Should Follow Up?1 (yes)   | Update Answer2             | ## 4. Deeper Insights with a Conversational AI Agent [Learn more about the AI Agent](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) |
| Update Answer2        | HTTP Request                  | Updates Google Sheet with user + AI response  | Interview Agent1           | Send Response              |                                                                                                              |
| Send Response         | Telegram                       | Sends AI agent's follow-up response to user  | Update Answer2             |                            |                                                                                                              |
| Get State3            | Redis                          | Gets current survey state                       | Create Record2, Bot Actions1 | Get Columns1             |                                                                                                              |
| Get Columns1          | Google Sheets                  | Retrieves survey questions                      | Get State3                 | Get Survey State1          |                                                                                                              |
| Get Survey State1     | Set                            | Calculates next question index and completion | Get Columns1               | Increment Index1           |                                                                                                              |
| Increment Index1      | Redis                          | Updates current question index in Redis        | Get Survey State1          | Is Survey Continue?        | ## 5. Managing Conversational Flow with External State [Learn more about the Redis node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.redis/) |
| Is Survey Continue?   | If                             | Checks if survey is complete                    | Increment Index1           | Reset Agent Memory1, Completed Survey |                                                                                                              |
| Reset Agent Memory1   | Memory Manager                | Clears AI chat memory and inserts current question | Is Survey Continue? (yes) | Send Next Question         | ## 6. Resetting Chat Memory for Every Question [Learn more about the Chat Memory Manager](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorymanager/) |
| Send Next Question    | Telegram                       | Sends next survey question to user              | Reset Agent Memory1        |                            |                                                                                                              |
| Completed Survey      | Telegram                       | Sends survey completion message                  | Is Survey Continue? (no)   |                            |                                                                                                              |
| Memory3               | Redis Chat Memory             | Stores limited chat history for AI agent        |                            | Reset Agent Memory1        |                                                                                                              |
| Memory4               | Redis Chat Memory             | Stores extended chat history for AI agent       |                            | Interview Agent1           |                                                                                                              |
| Model2                | OpenAI GPT-4o-mini            | Language model for AI Agent                      |                            | Interview Agent1           |                                                                                                              |
| Model3                | OpenAI GPT-4o-mini            | Language model for Text Classifier               |                            | Should Follow Up?1         |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates only  
   - Credentials: Connect your Telegram bot API credentials  
   - Position: Entry point

2. **Add Set Variables node (Set Variables1):**  
   - Define variables:  
     - `title`: "Product Satisfaction Survey: DJI Mini 2"  
     - `gsheetId`: Your Google Sheet ID containing survey questions  
     - `cacheKey`: Expression `survey_user_{{ $json.sessionId }}`  
   - Connect Telegram Trigger output to this node

3. **Add Execution Data node (Execution Data2):**  
   - Save metadata: jobType = "state_message", gsheetId, title, fromId (Telegram user ID)  
   - Connect Set Variables1 output to this node

4. **Add Redis node (Get State2):**  
   - Operation: Get  
   - Key: `={{ $json.cacheKey }}`  
   - Property Name: `data`  
   - Credentials: Your Redis instance  
   - Connect Execution Data2 output to this node

5. **Add Switch node (Message Type1):**  
   - Condition: Check if message text is one of `/start`, `/next`, `/reset`  
   - Outputs: `is_bot_command` and `is_normal_message`  
   - Connect Get State2 output to this node

6. **Add Set node (Get Command1):**  
   - Extract command text from Telegram message entities using expression  
   - Connect Message Type1 `is_bot_command` output to this node

7. **Add Switch node (Bot Actions1):**  
   - Conditions:  
     - `/start` → `new_session`  
     - `/next` → `next_question`  
     - fallback → `extra`  
   - Connect Get Command1 output to this node

8. **Add Google Sheets node (Create Record1):**  
   - Operation: Append or update  
   - Sheet: Your survey sheet with ID column and question columns  
   - Mapping: Match on `ID` column with Telegram user ID  
   - Connect Bot Actions1 `new_session` output to this node

9. **Add Redis node (Start Session1):**  
   - Operation: Set  
   - Key: `={{ $('Set Variables1').first().json.cacheKey }}`  
   - Value: JSON with `has_session: true`, `session_createdAt: $now`, `current_question_idx: 0`  
   - Key Type: Hash  
   - Connect Create Record1 output to this node

10. **Add Telegram node (Send Start):**  
    - Text: Welcome message with survey title and instructions  
    - Chat ID: `={{ $('Telegram Trigger').first().json.message.chat.id }}`  
    - Connect Start Session1 output to this node

11. **Add Telegram node (Send Start1):**  
    - Text: "Sorry, that command is unrecognised. The available options are /start or /next."  
    - Chat ID: same as above  
    - Connect Bot Actions1 `extra` output to this node

12. **Add Google Sheets node (Get Record1):**  
    - Operation: Lookup user record by Telegram user ID  
    - Connect Message Type1 `is_normal_message` output to this node

13. **Add If node (Has No Record?):**  
    - Condition: Check if Get Record1 output is empty  
    - True → Create Record2  
    - False → Get Last Bot Message1  
    - Connect Get Record1 output to this node

14. **Add Google Sheets node (Create Record2):**  
    - Same config as Create Record1  
    - Connect Has No Record? true output to this node

15. **Add Redis node (Get Last Bot Message1):**  
    - Operation: Get  
    - Key: `={{ $('Set Variables1').item.json.cacheKey }}_history`  
    - Property Name: `data`  
    - Connect Has No Record? false output to this node

16. **Add Set node (Append Responses1):**  
    - Prepare payload combining question, user answer, and placeholders  
    - Calculate cell reference based on question index and row number  
    - Connect Get Last Bot Message1 output to this node

17. **Add Text Classifier node (Should Follow Up?1):**  
    - Input Text: Combines last bot message and user answer  
    - Categories: `should_ask_followup_questions`, `should_not_ask_followup_questions`  
    - Connect Append Responses1 output to this node

18. **Add HTTP Request node (Update Answer3):**  
    - Method: PUT  
    - URL: Google Sheets API endpoint for updating cell  
    - Body: User response only  
    - Connect Should Follow Up?1 `should_not_ask_followup_questions` output here

19. **Add AI Agent node (Interview Agent1):**  
    - Text input: User message text  
    - Language Model: GPT-4o-mini (Model2)  
    - Connect Should Follow Up?1 `should_ask_followup_questions` output here

20. **Add HTTP Request node (Update Answer2):**  
    - Same as Update Answer3 but includes AI agent's response appended  
    - Connect Interview Agent1 output here

21. **Add Telegram node (Send Response):**  
    - Sends AI agent's follow-up response to user  
    - Connect Update Answer2 output here

22. **Add Redis node (Get State3):**  
    - Operation: Get  
    - Key: User cacheKey  
    - Connect Create Record2 and Bot Actions1 `next_question` output to this node

23. **Add Google Sheets node (Get Columns1):**  
    - Reads first row (questions) from Google Sheet  
    - Connect Get State3 output to this node

24. **Add Set node (Get Survey State1):**  
    - Calculates next question index, total questions, and completion status  
    - Connect Get Columns1 output to this node

25. **Add Redis node (Increment Index1):**  
    - Updates current_question_idx in Redis based on next_question_idx and total questions  
    - Connect Get Survey State1 output to this node

26. **Add If node (Is Survey Continue?):**  
    - Checks if survey is complete  
    - True → Reset Agent Memory1  
    - False → Completed Survey  
    - Connect Increment Index1 output here

27. **Add Memory Manager node (Reset Agent Memory1):**  
    - Mode: Insert with override  
    - Inserts current question prompt only to clear previous context  
    - Connect Is Survey Continue? true output here

28. **Add Telegram node (Send Next Question):**  
    - Sends next question text to user  
    - Connect Reset Agent Memory1 output here

29. **Add Telegram node (Completed Survey):**  
    - Sends survey completion message with restart instructions  
    - Connect Is Survey Continue? false output here

30. **Add Redis Chat Memory nodes (Memory3 and Memory4):**  
    - Memory3: context window length 10, feeds Reset Agent Memory1  
    - Memory4: context window length 100, feeds Interview Agent1  
    - Connect appropriately with Redis credentials

31. **Add OpenAI GPT-4o-mini nodes (Model2 and Model3):**  
    - Model2: used by Interview Agent1  
    - Model3: used by Should Follow Up?1  
    - Configure with OpenAI API credentials

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates a structured chat journey augmented with AI for product satisfaction surveys. It balances scripted question flow with AI-driven follow-ups for richer insights.                                                                                                                                                                                                                                                                                                                            | See Sticky Note7 content in workflow.                                                                           |
| Telegram bot setup is required. See n8n docs for Telegram credentials setup.                                                                                                                                                                                                                                                                                                                                                                                                                                         | https://docs.n8n.io/integrations/builtin/credentials/telegram/                                                  |
| Google Sheet must have an ID column and survey questions as columns. Sample sheet provided in description.                                                                                                                                                                                                                                                                                                                                                                                                           | https://docs.google.com/spreadsheets/d/e/2PACX-1vQWcREg75CzbZd8loVI12s-DzSTj3NE_02cOCpAh7umj0urazzYCfzPpYvvh7jqICWZteDTALzBO46i/pubhtml?gid=0&single=true |
| Redis is used for fast session state and chat memory management. Upstash is recommended for free Redis hosting.                                                                                                                                                                                                                                                                                                                                                                                                      | https://upstash.com?ref=jimleuk                                                                                  |
| Community+ license is required for the Execution Data node; remove if unavailable.                                                                                                                                                                                                                                                                                                                                                                                                                                    |                                                                                                                 |
| The AI agent memory is reset per question to reduce hallucinations and keep conversations focused.                                                                                                                                                                                                                                                                                                                                                                                                                   | See Sticky Note6 content in workflow.                                                                            |
| This pattern can be adapted to other chat platforms like WhatsApp by replacing Telegram nodes accordingly.                                                                                                                                                                                                                                                                                                                                                                                                           |                                                                                                                 |
| For help and community support, join the n8n Discord or Forum.                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Discord: https://discord.com/invite/XPKeKXeB7d, Forum: https://community.n8n.io/                                 |

---

This documentation provides a detailed, stepwise understanding of the workflow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the survey chatbot effectively.