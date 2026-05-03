Create a Witty Telegram Bot with AI-Powered Humor, Roasts & Stats using OpenRouter

https://n8nworkflows.xyz/workflows/create-a-witty-telegram-bot-with-ai-powered-humor--roasts---stats-using-openrouter-7655


# Create a Witty Telegram Bot with AI-Powered Humor, Roasts & Stats using OpenRouter

---

### 1. Workflow Overview

**Purpose:**  
This workflow implements **GiggleGPTBot**, a witty Telegram bot that interacts with users using AI-powered humor, motivational lines, sarcastic roasts, and scheduled witty posts. It supports various commands, tracks user activity, and dynamically responds to mentions. The bot leverages OpenRouter’s AI models and persists all interactions and statistics in a Postgres database.

**Target Use Cases:**  
- Delivering entertaining and motivational content to Telegram chats.  
- Providing user engagement analytics and leaderboards.  
- Automated scheduled posting of jokes, motivation, and wisdom.  
- Responding contextually when mentioned in chat.  

**Logical Blocks:**  
1.1 **Input Reception and Initialization**  
- Telegram webhook trigger to receive messages.  
- Database initialization to create required tables.

1.2 **Message Logging and Statistics Tracking**  
- Persist user messages and update message/command counts.  

1.3 **Command Routing and Analysis**  
- Switch node routes different commands and mentions.  
- Context fetching of recent chat messages.  
- Analysis of mentions, commands, and content type.

1.4 **AI-Powered Response Generation**  
- AI agents generate witty replies to commands and mentions.  
- AI generates scheduled posts (jokes, motivation, wisdom).

1.5 **Information Command Handling**  
- Static replies for `/help`, `/stats`, and `/top` commands.  
- Fetch user and top user statistics from the database.

1.6 **Response Dispatch**  
- Send replies back to Telegram chats.  
- Save bot responses in the database for tracking.

1.7 **Scheduled Posting**  
- Hourly schedule trigger checks for active scheduled posts.  
- Posts generated text automatically to chats.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Initialization

**Overview:**  
Receives Telegram updates via webhook and ensures the database schema is prepared for persistence.

**Nodes Involved:**  
- Webhook Telegram  
- Init Database  

**Node Details:**  

- **Webhook Telegram**  
  - Type: Telegram Trigger  
  - Role: Entry point, receives all Telegram updates (messages, commands, mentions).  
  - Config: Listens for all update types (`*`). Uses Telegram API credentials linked to the bot.  
  - Output: Emits incoming Telegram message JSON.  
  - Potential failures: Credential errors, webhook connectivity issues.

- **Init Database**  
  - Type: Postgres node  
  - Role: Creates necessary tables and indexes if they do not exist.  
  - Config: Executes a multi-table `CREATE TABLE IF NOT EXISTS` query defining tables for user messages, bot responses, commands, reactions, scheduled posts, and user stats. Also creates indexes for efficient querying.  
  - Input: None (manual or initial run recommended).  
  - Output: Success or error on DB schema initialization.  
  - Failures: DB connection/auth issues, SQL syntax errors.

---

#### 1.2 Message Logging and Statistics Tracking

**Overview:**  
Stores incoming user messages and updates per-user chat statistics.

**Nodes Involved:**  
- Log message + statistics  

**Node Details:**  

- **Log message + statistics**  
  - Type: Postgres node  
  - Role: Inserts the user’s message into `user_messages` and increments message count in `user_stats` with conflict handling.  
  - Config: Parameterized SQL inserting user info and message text; safely escapes single quotes to prevent SQL injection. Updates or inserts message count and last activity timestamp.  
  - Inputs: From Webhook Telegram node (message JSON).  
  - Output: Confirmation of DB write.  
  - Failures: DB connectivity, malformed input, concurrency conflicts (handled with upsert).  
  - On error: Continues workflow without stopping.

---

#### 1.3 Command Routing and Analysis

**Overview:**  
Determines the nature of the incoming message: commands (`/joke`, `/help`, etc.), bot mentions, or scheduled posts. Prepares context and metadata for AI processing or static replies.

**Nodes Involved:**  
- Switch  
- Chat history  
- Mention Analysis  
- If1 (conditional branching on mentions)  
- Response type (conditional branching on response type)  
- Generating an information response  
- Get user statistics  
- Get top users  
- Log command  

**Node Details:**  

- **Switch**  
  - Type: Switch node  
  - Role: Routes messages based on prefix (commands or mention).  
  - Config: Checks if message text starts with `/joke`, `/inspire`, `/random`, `/roast`, `/help`, `/stats`, `/top` or contains mention `@GiggleGPTBot`.  
  - Input: Telegram message text.  
  - Output: Multiple branches for each command or mention.

- **Chat history**  
  - Type: Postgres node  
  - Role: Retrieves last 15 messages from the chat, both user and bot messages, for context.  
  - Config: SQL query combines user and bot messages ordered by timestamp desc.  
  - Input: Chat id from Telegram message.  
  - Output: Chat context for AI prompt.  
  - Failures: DB read errors, empty chat history.

- **Mention Analysis**  
  - Type: Code node (JavaScript)  
  - Role: Parses message and chat history; extracts user info, command, content type, time context, and prepares prompt variables.  
  - Key variables: `userId`, `userName`, `userMessage`, `command`, `contentType` (funny, inspiring, mixed), `timeContext` (morning, evening, etc.), `recentMessages`.  
  - Input: Telegram message JSON and chat history.  
  - Output: Structured JSON with parsed info for downstream AI nodes.  
  - Edge cases: Empty messages after mention, no command; handled by random content type selection.

- **If1**  
  - Type: If node  
  - Role: Checks if the original message starts with `@GiggleGPTBot` to decide between AI command or mention response.  
  - Input: Parsed message from Mention Analysis.  
  - Output: Branches to AI response nodes.

- **Response type**  
  - Type: If node  
  - Role: Determines if the response is informational (static text) or AI-generated text based on flags like `isInfoCommand` or presence of `responseText`.  
  - Output: Routes to info reply or AI reply sending nodes.

- **Generating an information response**  
  - Type: Code node  
  - Role: Builds replies for info commands `/help`, `/stats`, `/top` based on database queries.  
  - Uses: User stats and top users data from DB nodes.  
  - Output: Preformatted text to send back to Telegram.  
  - Handles unknown commands gracefully.

- **Get user statistics**  
  - Type: Postgres node  
  - Role: Fetches message, command counts, responses count, last activity for the user in the chat.  
  - Input: Chat and user IDs.  
  - Output: User statistics for info command replies.

- **Get top users**  
  - Type: Postgres node  
  - Role: Selects top 10 active users by messages + command weight, formats usernames.  
  - Input: Chat ID.  
  - Output: Leaderboard data for info command replies.

- **Log command**  
  - Type: Postgres node  
  - Role: Logs every bot command execution, increments command count for the user.  
  - Input: User and chat info, command text.  
  - Output: Confirmation of DB writes.  
  - Failures: DB issues.

---

#### 1.4 AI-Powered Response Generation

**Overview:**  
Uses OpenRouter AI models to generate witty replies to commands, mentions, and scheduled posts with style instructions.

**Nodes Involved:**  
- OpenRouter Commands (LM Chat node)  
- AI response to command  
- AI response to mention  
- AI post generation  

**Node Details:**  

- **OpenRouter Commands**  
  - Type: Langchain LM Chat node (OpenRouter)  
  - Role: Provides access to OpenRouter AI models, configured with the `openai/gpt-oss-120b` model.  
  - Credentials: Requires OpenRouter API key.  
  - Exposes AI model to multiple nodes.

- **AI response to command**  
  - Type: Langchain agent node  
  - Role: Generates replies for recognized commands (`/joke`, `/inspire`, `/random`, `/roast`) with prompt design enforcing witty, concise style.  
  - Prompt uses variables: command, userName, recentMessages, timeContext.  
  - Includes system instructions restricting verbosity, style, and profanity.  
  - Retries on failures (max 2).  
  - Output: AI-generated witty text.

- **AI response to mention**  
  - Type: Langchain agent node  
  - Role: Creates a witty reply when the bot is mentioned without a command, using chat context and content type.  
  - Prompt instructs tactful, concise, humorous style with emojis and metaphors sparingly.  
  - Retries on failures (max 2).  
  - Output: AI-generated witty reply.

- **AI post generation**  
  - Type: Langchain agent node  
  - Role: Generates scheduled posts: morning jokes, daily motivation, or random wisdom.  
  - Prompt instructions emphasize plain text output, short and friendly style.  
  - Output: Text posted to Telegram and saved.  
  - Retries on failures (max 2).

---

#### 1.5 Information Command Handling

**Overview:**  
Constructs static informational replies for commands like `/help`, `/stats`, and `/top`.

**Nodes Involved:**  
- Generating an information response  
- Get user statistics  
- Get top users  
- Log command  
- Response type  
- Send info reply  

**Node Details:**  

- See details in Block 1.3 for data retrieval nodes.  
- **Send info reply**  
  - Type: Telegram node  
  - Role: Sends the informational text reply to the chat.  
  - Config: Uses HTML parse mode to format message.  
  - Input: Text from Generating an information response node.  
  - Failures: Telegram API issues, chat ID errors.

---

#### 1.6 Response Dispatch

**Overview:**  
Sends messages back to users or chats and logs bot responses.

**Nodes Involved:**  
- Send AI response  
- Reply to Mention  
- Save Bot Response  
- Save Bot Response2  

**Node Details:**  

- **Send AI response**  
  - Type: Telegram node  
  - Role: Sends AI-generated text replies for commands to Telegram chat.  
  - Input: AI-generated text and chat ID.  
  - Config: HTML parse mode enabled.  
  - Credentials: Telegram bot API.  
  - Failures: Network, chat ID not found.

- **Reply to Mention**  
  - Type: Telegram node  
  - Role: Replies to user messages where bot was mentioned, replying directly to the message.  
  - Config: Uses reply_to_message_id to link reply.  
  - Input: AI-generated mention response, chat ID, message ID.  
  - Failures: Same as above.

- **Save Bot Response / Save Bot Response2**  
  - Type: Postgres nodes  
  - Role: Persist bot replies to `bot_responses` table, logging user/chat/message context and response type.  
  - Input: Bot response text, user and chat ids, original message.  
  - Failures: DB write errors.

---

#### 1.7 Scheduled Posting

**Overview:**  
Automatically posts scheduled content on an hourly basis using AI-generated posts.

**Nodes Involved:**  
- Schedule (Schedule Trigger)  
- Get scheduled posts  
- If (conditional on scheduled posts existence)  
- AI post generation  
- Submit scheduled post  
- Save Bot Response2  

**Node Details:**  

- **Schedule**  
  - Type: Schedule Trigger  
  - Role: Runs every hour (CRON `0 * * * *`).  
  - Output: Triggers downstream logic to check scheduled posts.

- **Get scheduled posts**  
  - Type: Postgres node  
  - Role: Selects active scheduled posts where the scheduled hour matches current hour.  
  - Output: List of scheduled posts to send.

- **If**  
  - Type: If node  
  - Role: Checks if any scheduled posts exist for the current hour.  
  - Output: Proceeds if there are posts.

- **AI post generation**  
  - See above (Block 1.4). Generates post content based on `post_type`.

- **Submit scheduled post**  
  - Type: Telegram node  
  - Role: Sends the generated post to the target chat.  
  - Config: Uses chat ID from scheduled post, HTML parse mode enabled.

- **Save Bot Response2**  
  - Same as Save Bot Response but for scheduled posts. Stores bot-generated scheduled content.

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                               | Input Node(s)                | Output Node(s)                      | Sticky Note                                                                                             |
|--------------------------|----------------------------------|-----------------------------------------------|------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------|
| Webhook Telegram         | Telegram Trigger                 | Receives Telegram updates                       | None                         | Log message + statistics, Switch   |                                                                                                       |
| Init Database            | Postgres                        | Creates DB tables and indexes                   | None                         | None                              |                                                                                                       |
| Log message + statistics | Postgres                        | Logs user messages and updates stats           | Webhook Telegram             | None                              |                                                                                                       |
| Adding a schedule        | Postgres                        | Inserts initial scheduled posts                 | None                         | None                              |                                                                                                       |
| Schedule                 | Schedule Trigger                | Triggers hourly scheduled post checks          | None                         | Get scheduled posts                |                                                                                                       |
| Get scheduled posts      | Postgres                       | Fetches scheduled posts active for current hour | Schedule                     | If                               |                                                                                                       |
| If                       | If                             | Checks if scheduled posts exist                  | Get scheduled posts          | AI post generation                |                                                                                                       |
| AI post generation       | Langchain Agent (OpenRouter)   | Generates scheduled post content                 | If                           | Submit scheduled post, Save Bot Response2 |                                                                                                       |
| Submit scheduled post    | Telegram                       | Sends scheduled posts to Telegram chats          | AI post generation           | None                              |                                                                                                       |
| Save Bot Response2       | Postgres                      | Saves scheduled post responses                    | AI post generation           | None                              |                                                                                                       |
| Switch                   | Switch                        | Routes messages based on commands or mentions    | Log message + statistics     | Chat history (multiple branches)  |                                                                                                       |
| Chat history             | Postgres                      | Retrieves chat history for context                | Switch                      | Mention Analysis                  |                                                                                                       |
| Mention Analysis         | Code                         | Parses message and chat context                   | Chat history                 | If1                             |                                                                                                       |
| If1                      | If                           | Detects if bot is mentioned                       | Mention Analysis             | AI response to mention / AI response to command |                                                                                                       |
| AI response to mention   | Langchain Agent (OpenRouter)  | Generates witty reply to mentions                 | If1                         | Reply to Mention, Save Bot Response |                                                                                                       |
| Reply to Mention         | Telegram                     | Sends mention reply to Telegram                    | AI response to mention       | None                              |                                                                                                       |
| AI response to command   | Langchain Agent (OpenRouter)  | Generates witty reply to commands                  | If1                         | Response type, Save Bot Response  |                                                                                                       |
| Response type            | If                           | Chooses between info reply or AI reply             | AI response to command / Generating an information response | Send info reply / Send AI response |                                                                                                       |
| Send AI response         | Telegram                     | Sends AI-generated replies to Telegram             | Response type                | None                              |                                                                                                       |
| Send info reply          | Telegram                     | Sends info command replies                         | Response type                | None                              |                                                                                                       |
| Save Bot Response        | Postgres                      | Saves bot replies to DB                             | AI response to mention / AI response to command | None                              |                                                                                                       |
| Generating an information response | Code                  | Creates static replies for `/help`, `/stats`, `/top` | Get user statistics, Get top users | Log command, Response type         |                                                                                                       |
| Get user statistics      | Postgres                      | Fetches user stats for info commands                | Switch                      | Generating an information response |                                                                                                       |
| Get top users            | Postgres                      | Fetches leaderboard data                            | Switch                      | Generating an information response |                                                                                                       |
| Log command              | Postgres                      | Logs command usage and updates stats                | Generating an information response | Response type                     |                                                                                                       |
| Adding a schedule        | Postgres                      | Seeds scheduled posts (morning joke, motivation, wisdom) | None                         | None                              |                                                                                                       |
| Sticky Note2             | Sticky Note                   | Documentation overview and setup instructions      | None                         | None                              | # GiggleGPTBot — Witty Telegram Bot with AI & Postgres (full workflow overview and instructions)      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot**  
   - Use [@BotFather](https://t.me/BotFather) to create a Telegram bot and obtain API token.

2. **Setup Credentials in n8n**  
   - Add Telegram API credentials with the bot token.  
   - Add OpenRouter API credentials with your API key from https://openrouter.ai.  
   - Add Postgres credentials pointing to your database (Supabase recommended).

3. **Create `Init Database` Node (Postgres)**  
   - Create a Postgres node named `Init Database`.  
   - Paste the multi-table `CREATE TABLE IF NOT EXISTS` SQL for tables: `user_messages`, `bot_responses`, `bot_commands`, `message_reactions`, `scheduled_posts`, `user_stats`.  
   - Create necessary indexes as in the original query.  
   - Run once to initialize schema.

4. **Create `Webhook Telegram` Node**  
   - Type: Telegram Trigger.  
   - Set updates to `*` to capture all messages and commands.  
   - Set credentials to Telegram bot API.  
   - Position as input trigger.

5. **Create `Log message + statistics` Node**  
   - Postgres node.  
   - Insert incoming message data into `user_messages`.  
   - Upsert `user_stats` to increment `messages_count` and update `last_activity`.  
   - Connect `Webhook Telegram` output to this node.

6. **Create `Switch` Node**  
   - Add rules for commands `/joke`, `/inspire`, `/random`, `/roast`, `/help`, `/stats`, `/top`.  
   - Add rule for messages starting with `@GiggleGPTBot`.  
   - Connect `Webhook Telegram` —> `Switch`.

7. **Create `Chat history` Node (Postgres)**  
   - Query last 15 messages from `user_messages` and `bot_responses` for the chat.  
   - Connect all `Switch` outputs to this node.

8. **Create `Mention Analysis` Node (Code)**  
   - JavaScript code that:  
     - Extracts user info, command, message text, content type, time context.  
     - Creates a summary of recent messages as chat context.  
   - Input: `Chat history`.  
   - Output: Parsed message data.

9. **Create `If1` Node (If)**  
   - Condition: Check if original message starts with `@GiggleGPTBot`.  
   - True branch to `AI response to mention`.  
   - False branch to `AI response to command`.

10. **Create `OpenRouter Commands` Node (Langchain LM Chat)**  
    - Model: `openai/gpt-oss-120b`.  
    - Credentials: OpenRouter API key.

11. **Create `AI response to command` Node (Langchain Agent)**  
    - Prompt: Generates witty replies for commands with style instructions.  
    - Input: Parsed message data from `If1` false branch.  
    - Connect to `OpenRouter Commands` node.

12. **Create `AI response to mention` Node (Langchain Agent)**  
    - Prompt: Replies tactfully to mentions with humor.  
    - Input: Parsed message data from `If1` true branch.  
    - Connect to `OpenRouter Commands` node.

13. **Create `Generating an information response` Node (Code)**  
    - Handles `/help`, `/stats`, `/top` commands with text responses.  
    - Input: `Get user statistics` and `Get top users` nodes outputs.

14. **Create `Get user statistics` Node (Postgres)**  
    - Query user stats by user and chat ID.

15. **Create `Get top users` Node (Postgres)**  
    - Query top 10 users in chat by activity.

16. **Create `Log command` Node (Postgres)**  
    - Logs command usage and updates command count.

17. **Connect `Generating an information response` to `Log command` and then `Response type` Node (If)**  
    - `Response type` node routes between info replies and AI replies.

18. **Create `Send AI response` Node (Telegram)**  
    - Sends AI-generated text replies to chat.

19. **Create `Send info reply` Node (Telegram)**  
    - Sends info command replies using HTML parse mode.

20. **Create `Reply to Mention` Node (Telegram)**  
    - Sends AI mention replies, replying directly to the user message.

21. **Create `Save Bot Response` and `Save Bot Response2` Nodes (Postgres)**  
    - Insert bot responses into `bot_responses` table with user/chat/message context.

22. **Create Scheduled Posting Flow:**  
    - `Schedule` node: CRON trigger every hour.  
    - `Get scheduled posts` node: Query active posts scheduled for current hour.  
    - `If` node: Checks if posts exist.  
    - Connect true branch to `AI post generation` node (Langchain Agent) with prompt for post type.  
    - Connect `AI post generation` output to `Submit scheduled post` (Telegram) and `Save Bot Response2` nodes.

23. **Create `Adding a schedule` Node (Postgres)**  
    - Insert initial scheduled posts with chat ID, post type, scheduled time.

24. **Link all nodes as per original workflow connections.**

25. **Test the workflow thoroughly:**  
    - Test commands `/joke`, `/help`, `/stats`, `/top`, `/roast`, `/inspire`.  
    - Test mention replies by tagging `@GiggleGPTBot`.  
    - Test scheduled posts by adjusting system time or scheduled times.  
    - Monitor logs and DB for data integrity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| GiggleGPTBot is a witty Telegram Bot built with n8n, OpenRouter AI, and Postgres for persistence. It supports AI-powered humor, motivational lines, roasts, scheduled posts, and user statistics.                             | Overview in Sticky Note node                      |
| Setup instructions include creating a Telegram bot with @BotFather, adding credentials (Telegram API, OpenRouter API, Postgres), initializing the database schema, and optionally seeding scheduled posts.                     | Sticky Note content                              |
| Commands supported: `/joke`, `/inspire`, `/random`, `/roast`, `/stats`, `/top`, `/help`, and bot mention replies with `@GiggleGPTBot`.                                                                                      | Command list in Sticky Note                      |
| OpenRouter AI prompts enforce style rules: concise, witty, light irony, sparing emojis, no crude or offensive content, and no explanations or lengthy replies.                                                                | Prompt instructions in AI nodes                  |
| The database schema includes tables for user messages, bot responses, bot commands, reactions, scheduled posts, and aggregated user stats with indexes for performance.                                                       | DB schema in Init Database node                   |
| Scheduled posts support morning jokes (6:00), daily motivation (9:00), and random wisdom (17:00) by default; scheduling can be customized.                                                                                   | Adding a schedule node                            |
| For advanced customization, users can add new commands, extend analytics, localize prompts, or adjust scheduled posting CRON expressions.                                                                                   | Suggested customization ideas in Sticky Note    |
| Telegram API credentials require bot token; OpenRouter requires API key from https://openrouter.ai; Postgres DB (Supabase recommended) must be accessible with proper permissions.                                           | Credential setup notes                            |
| The workflow handles errors gracefully, continuing execution on DB write failure for logging, and retries AI calls up to twice for robustness.                                                                              | Error handling notes                              |

---

**Disclaimer:** The content originates exclusively from an n8n automated workflow. It complies fully with content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.

---