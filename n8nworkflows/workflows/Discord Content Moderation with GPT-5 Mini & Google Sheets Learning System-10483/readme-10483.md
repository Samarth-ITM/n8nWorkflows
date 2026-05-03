Discord Content Moderation with GPT-5 Mini & Google Sheets Learning System

https://n8nworkflows.xyz/workflows/discord-content-moderation-with-gpt-5-mini---google-sheets-learning-system-10483


# Discord Content Moderation with GPT-5 Mini & Google Sheets Learning System

### 1. Workflow Overview

This workflow automates content moderation for a Discord community (specifically a forex trading academy) by leveraging AI to analyze recent messages and identify those violating community standards. It integrates Google Sheets as a learning knowledge base to continuously improve moderation accuracy based on curated examples. Flagged messages are automatically deleted on Discord, and a notification with details is sent to an admin channel for transparency and oversight.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Initialization:** Periodically triggers the workflow and sets core configuration parameters (Discord server and channel IDs).
- **1.2 Training Data Retrieval:** Loads moderation examples from a Google Sheet that distinguish messages that should be deleted from those that should be kept.
- **1.3 Message Retrieval & Preparation:** Fetches recent messages from the monitored Discord channel and formats them along with training data for AI consumption.
- **1.4 AI Moderation Processing:** Uses a GPT-5 Mini language model via LangChain to classify messages that violate rules, guided by training examples and a detailed system prompt.
- **1.5 Post-Processing & Cleanup:** Parses AI results, deduplicates flagged messages, deletes them from Discord, and sends admin notifications about the deletions.
- **1.6 Flow Control & Rate Limiting:** Manages deletion request pacing with wait nodes and batch processing to avoid API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Initialization

- **Overview:**  
Triggers the workflow every few minutes to perform continuous monitoring and sets the core Discord IDs required for operations.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set Credentials Here  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Periodically starts the workflow every minute (configured for minute-based intervals)  
    - Configuration: Interval set to run every 1 minute (default, can be adjusted)  
    - Input: None  
    - Output: Triggers next node  
    - Edge Cases: Missed triggers if n8n instance down or time drift; ensure accurate timezone if relevant.

  - **Set Credentials Here**  
    - Type: Set  
    - Role: Stores essential Discord IDs as workflow variables for reuse  
    - Configuration:  
      - `discord server ID` (object): placeholder "1234567" — should be replaced by actual server ID  
      - `discord moderated channel ID` (object): placeholder "1234567" — should be replaced by actual moderated channel ID  
      - `discord admin channel ID` (object): placeholder "1234567" — should be replaced by actual admin notification channel ID  
    - Input: Trigger from Schedule Trigger  
    - Output: Passes IDs downstream  
    - Edge Cases: Missing or incorrect IDs will cause API calls to fail or operate on wrong channels.

---

#### 1.2 Training Data Retrieval

- **Overview:**  
Fetches training examples from Google Sheets which define clear examples of messages that should always or never be deleted to guide AI moderation.

- **Nodes Involved:**  
  - Get sheet knowledgebase  
  - Training Data Section (Sticky Note)  

- **Node Details:**

  - **Get sheet knowledgebase**  
    - Type: Google Sheets  
    - Role: Reads rows from a specified Google Sheet containing training examples  
    - Configuration:  
      - Document ID: points to a specific Google Sheet (URL provided in cachedResultUrl)  
      - Sheet Name: defaults to first sheet (gid=0)  
    - Credentials: Google Sheets OAuth2  
    - Input: From Set Credentials Here  
    - Output: Array of training example rows containing columns like `message_content`, `should_delete`, `reason`  
    - Edge Cases: Authentication failure, sheet not found, malformed data rows.

  - **Training Data Section (Sticky Note)**  
    - Role: Documentation only, explains purpose of training data (no functional impact).

---

#### 1.3 Message Retrieval & Preparation

- **Overview:**  
Fetches the most recent messages from the Discord moderated channel and prepares a formatted dataset for AI, combining these with the training examples loaded earlier.

- **Nodes Involved:**  
  - Get recent messages  
  - prep messages for AI  
  - Analysis Section (Sticky Note)  

- **Node Details:**

  - **Get recent messages**  
    - Type: Discord  
    - Role: Retrieves the latest 10 messages from the monitored channel  
    - Configuration:  
      - Operation: getAll messages  
      - Limit: 10 messages (can be adjusted)  
      - Guild ID: from `discord server ID` parameter set earlier  
      - Channel ID: from `discord moderated channel ID`  
    - Credentials: Discord OAuth2  
    - Input: From Get sheet knowledgebase  
    - Output: JSON array of message objects including id, content, author  
    - Edge Cases: API rate limits, empty channels, permissions errors.

  - **prep messages for AI**  
    - Type: Code (JavaScript)  
    - Role:  
      - Reads training examples from Google Sheets node output  
      - Formats these into "Messages to ALWAYS DELETE" and "Messages to NEVER DELETE" lists  
      - Formats recent messages with indices, author, content for AI prompt  
      - Passes all data and config parameters downstream  
    - Key Expressions:  
      - Builds training text snippets dynamically based on `should_delete` field  
      - Constructs a numbered message list for AI analysis  
      - Injects config object with Discord IDs for downstream use  
    - Input: From Get recent messages  
    - Output: Single JSON object containing `messageList`, `originalMessages`, `trainingExamples`, and `config`  
    - Edge Cases: Missing or malformed training data, empty message arrays, JSON parsing errors.

  - **Analysis Section (Sticky Note)**  
    - Role: Documentation explaining this block's purpose.

---

#### 1.4 AI Moderation Processing

- **Overview:**  
Uses a GPT-5 Mini language model to analyze messages in context of training data and community rules, returning indices of messages that should be deleted.

- **Nodes Involved:**  
  - GPT5 mini  
  - AI Agent  

- **Node Details:**

  - **GPT5 mini**  
    - Type: LangChain OpenAI Chat  
    - Role: Runs GPT-5 Mini model to generate AI output based on prompt text  
    - Configuration:  
      - Model: `gpt-5-mini-2025-08-07` (cached for performance)  
      - No additional options explicitly set  
    - Credentials: OpenAI API key  
    - Input: Receives prompt text including training examples and formatted messages  
    - Output: AI-generated JSON array of indices to delete or empty array  
    - Edge Cases: API key limits, response malformed or not JSON, network timeout.

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Builds a complex prompt combining training examples and messages to analyze, instructing the AI to flag messages for deletion based on detailed community rules  
    - Configuration:  
      - Prompt includes:  
        - Always delete messages examples  
        - Never delete messages examples  
        - System message with detailed moderation rules focusing on intent/context, allowing profanity in positive contexts  
        - Training examples and message list dynamically injected from inputs  
      - Returns: JSON array of message indices for deletion (e.g. `[0,2,5]`) or empty array if none  
    - Input: From prep messages for AI  
    - Output: AI response passed to next node  
    - Edge Cases: Parsing errors if AI returns invalid JSON, misunderstanding of prompt leading to false positives or negatives.

---

#### 1.5 Post-Processing & Cleanup

- **Overview:**  
Processes the AI output by extracting messages to delete, deduplicates them, deletes flagged messages on Discord, and posts notifications in the admin channel.

- **Nodes Involved:**  
  - Get only the bad messages (Code)  
  - Loop Over Items (SplitInBatches)  
  - delete bad content (Discord)  
  - wait  
  - update admin channel about moderation (Discord)  
  - Processing Section (Sticky Note)  

- **Node Details:**

  - **Get only the bad messages**  
    - Type: Code (JavaScript)  
    - Role:  
      - Parses the AI agent's output to extract indices of bad messages  
      - Validates indices and removes duplicates  
      - Maps valid indices to message objects with ID, author, content, and config  
      - Returns array of messages to delete or empty array if none  
    - Input: From AI Agent node  
    - Output: Array of JSON objects, each representing one flagged message  
    - Edge Cases: AI output parse errors, empty or invalid indices, duplicate messages.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each flagged message sequentially to safely process deletions and notifications  
    - Configuration: Default batch size (1) to handle one message at a time  
    - Input: From Get only the bad messages  
    - Output: Single item per iteration to delete and notify nodes  
    - Edge Cases: Empty input array leads to no iterations; batch size can be adjusted to manage rate limits.

  - **delete bad content**  
    - Type: Discord  
    - Role: Deletes a specified message from the moderated channel on Discord  
    - Configuration:  
      - Operation: deleteMessage  
      - Guild ID: from message config  
      - Channel ID: from message config (moderated channel)  
      - Message ID: from current batch item  
    - Credentials: Discord OAuth2  
    - Input: From Loop Over Items  
    - Output: Triggers wait node for rate limiting  
    - On Error: Configured to continue regular output to avoid halting workflow if deletion fails  
    - Edge Cases: Message already deleted, permissions denied, rate limits.

  - **wait**  
    - Type: Wait  
    - Role: Adds a 1.5-second delay between deletions to avoid hitting Discord API rate limits  
    - Configuration: 1.5 seconds delay  
    - Input: From delete bad content  
    - Output: Triggers update admin channel about moderation node  
    - Edge Cases: Delay might slow workflow if many messages; adjust as needed.

  - **update admin channel about moderation**  
    - Type: Discord  
    - Role: Sends a notification message to the admin channel with details about the deleted message (author, ID, content)  
    - Configuration:  
      - Guild ID: from message config  
      - Channel ID: from admin channel config  
      - Content: Text with placeholders for author username, message ID, and content from the deleted message  
    - Credentials: Discord OAuth2  
    - Input: From wait node  
    - Output: Triggers Loop Over Items continuation to process next message  
    - Edge Cases: Permission to post in admin channel, message formatting errors.

  - **Processing Section (Sticky Note)**  
    - Role: Documentation explaining purpose of this block.

---

### 3. Summary Table

| Node Name                     | Node Type                       | Functional Role                                     | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                                                             |
|-------------------------------|--------------------------------|----------------------------------------------------|--------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                | Periodically triggers workflow                      | None                     | Set Credentials Here           |                                                                                                                                         |
| Set Credentials Here           | Set                            | Stores Discord server and channel IDs              | Schedule Trigger          | Get sheet knowledgebase        |                                                                                                                                         |
| Get sheet knowledgebase        | Google Sheets                  | Loads training examples for AI                      | Set Credentials Here      | Get recent messages            |                                                                                                                                         |
| Training Data Section          | Sticky Note                    | Explains purpose of training data                   | None                     | None                          | ## Training Data\nLoads examples from Google Sheets (message_content, should_delete, reason) to teach the AI your moderation standards. |
| Get recent messages            | Discord                        | Fetches recent messages from monitored channel     | Get sheet knowledgebase   | prep messages for AI           |                                                                                                                                         |
| prep messages for AI           | Code                           | Formats messages and training data for AI prompt   | Get recent messages       | AI Agent                      |                                                                                                                                         |
| Analysis Section              | Sticky Note                    | Explains message analysis block                      | None                     | None                          | ## Message Analysis\nFetches recent messages, formats them with training data, and sends to GPT-5 for context-aware moderation.          |
| GPT5 mini                     | LangChain OpenAI Chat          | Runs GPT-5 Mini model to analyze messages           | prep messages for AI      | AI Agent                      |                                                                                                                                         |
| AI Agent                     | LangChain Agent                | Builds prompt and classifies messages for deletion | prep messages for AI      | Get only the bad messages      |                                                                                                                                         |
| Get only the bad messages      | Code                           | Parses AI output to identify messages to delete     | AI Agent                 | Loop Over Items                |                                                                                                                                         |
| Loop Over Items               | SplitInBatches                 | Iterates over each flagged message                   | Get only the bad messages | delete bad content, update admin channel about moderation |                                                                                                                                         |
| delete bad content            | Discord                        | Deletes flagged messages from Discord                | Loop Over Items           | wait                         |                                                                                                                                         |
| wait                         | Wait                           | Adds delay between deletions to avoid rate limits   | delete bad content        | update admin channel about moderation |                                                                                                                                         |
| update admin channel about moderation | Discord                        | Sends notification to admin channel about deletions | wait                     | Loop Over Items               |                                                                                                                                         |
| Main Overview                | Sticky Note                    | Overview of workflow purpose and setup instructions | None                     | None                          | # AI-Driven Discord Moderation with Google Sheets Learning & Admin Insights\n\n## How it works:... (see detailed note content)        |
| Processing Section           | Sticky Note                    | Explains post-processing and cleanup                 | None                     | None                          | ## Processing & Cleanup\nParses AI results, deduplicates, then loops through each flagged message to delete and notifies admin channel.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set to run every 1 minute (or your preferred interval).

3. **Add a Set node named “Set Credentials Here”:**  
   - Add three variables:  
     - `discord server ID` (string or object) with your actual Discord server ID  
     - `discord moderated channel ID` with the channel ID to monitor  
     - `discord admin channel ID` with the admin notification channel ID  
   - Connect Schedule Trigger → Set Credentials Here.

4. **Add a Google Sheets node named “Get sheet knowledgebase”:**  
   - Operation: Read rows  
   - Document ID: your Google Sheet ID containing training examples  
   - Sheet Name: the sheet tab name or gid (e.g., “Sheet1” or “gid=0”)  
   - Connect Set Credentials Here → Get sheet knowledgebase.  
   - Configure Google Sheets OAuth2 credentials.

5. **Add a Discord node named “Get recent messages”:**  
   - Resource: Message  
   - Operation: getAll  
   - Limit: 10 (or desired number)  
   - Guild ID: set expression to reference `discord server ID` from Set Credentials Here  
   - Channel ID: set expression to reference `discord moderated channel ID` from Set Credentials Here  
   - Authentication: OAuth2 with your Discord credentials  
   - Connect Get sheet knowledgebase → Get recent messages.

6. **Add a Code node named “prep messages for AI”:**  
   - Paste the provided JavaScript code that:  
     - Reads training data from Google Sheets node  
     - Formats “Messages to ALWAYS DELETE” and “Messages to NEVER DELETE” lists  
     - Formats recent messages as a numbered list with IDs, authors, and content  
     - Passes a config object with Discord IDs downstream  
   - Connect Get recent messages → prep messages for AI.

7. **Add a LangChain GPT Chat node named “GPT5 mini”:**  
   - Model: select or enter “gpt-5-mini-2025-08-07” or your preferred GPT-5 Mini model  
   - Connect prep messages for AI → GPT5 mini.  
   - Set OpenAI API credentials.

8. **Add a LangChain Agent node named “AI Agent”:**  
   - Configure prompt with:  
     - System message detailing moderation rules, emphasizing intent/context, examples of what to delete or keep (from sticky notes)  
     - Use the `messageList` and `trainingExamples` from input JSON to build prompt dynamically  
     - Instruct AI to return JSON array of indices of messages to delete  
   - Connect GPT5 mini (ai_languageModel output) → AI Agent (ai_languageModel input).

9. **Add a Code node named “Get only the bad messages”:**  
   - Paste the JavaScript code to:  
     - Parse AI Agent’s output JSON array of indices  
     - Validate and deduplicate message indices  
     - Map indices to original messages with IDs, content, author, and config  
   - Connect AI Agent → Get only the bad messages.

10. **Add a SplitInBatches node named “Loop Over Items”:**  
    - Default batch size 1  
    - Connect Get only the bad messages → Loop Over Items.

11. **Add a Discord node named “delete bad content”:**  
    - Resource: Message  
    - Operation: deleteMessage  
    - Guild ID: from `config.discordServerId` in current item JSON  
    - Channel ID: from `config.moderatedChannelId`  
    - Message ID: from `messageId` field  
    - Authentication: OAuth2 with Discord credentials  
    - On Error: set to continue regular output (to not stop workflow if deletion fails)  
    - Connect Loop Over Items (main output 1) → delete bad content.

12. **Add a Wait node named “wait”:**  
    - Time: 1.5 seconds  
    - Connect delete bad content → wait.

13. **Add a Discord node named “update admin channel about moderation”:**  
    - Resource: Message  
    - Operation: create message (send)  
    - Guild ID: from `config.discordServerId`  
    - Channel ID: from `config.adminChannelId`  
    - Content: template with message author username, message ID, and content from current item JSON  
    - Authentication: OAuth2 with Discord credentials  
    - Connect wait → update admin channel about moderation.

14. **Connect update admin channel about moderation main output → Loop Over Items (main input 2) to continue batch processing.**

15. **Add Sticky Notes for documentation:**  
    - “Main Overview” with detailed project overview and instructions  
    - “Training Data Section” explaining Google Sheets role  
    - “Analysis Section” explaining message retrieval and AI processing  
    - “Processing Section” explaining deletion and notifications.

16. **Ensure all credentials for Discord OAuth2, OpenAI API, and Google Sheets OAuth2 are set up and linked correctly.**

17. **Test workflow manually before enabling schedule trigger: run once, observe logs, check deletion and admin notifications on Discord.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Copy the training sheet to customize examples: add your own messages with `message_content`, `should_delete` (YES/NO), and `reason` columns to teach the AI your community standards.                                         | https://docs.google.com/spreadsheets/d/1xodthGg8RpQJB62mB6fziuwblG9nn3udZvQvyRquCXM/edit?usp=sharing          |
| The AI prompt emphasizes understanding context and intent over keyword matching, allowing profanity in positive/supportive messages.                                                                                        | Workflow AI Agent node system prompt content.                                                                   |
| Rate limiting is managed via the Wait node with 1.5 seconds delay between deletions to avoid Discord API restrictions.                                                                                                       | See Wait node configuration.                                                                                    |
| Use the sticky notes inside the workflow for detailed explanations of each block and setup instructions.                                                                                                                     | Within workflow editor UI.                                                                                       |
| Workflow tested with Discord OAuth2 authentication and OpenAI API keys; ensure tokens have appropriate scopes and permissions on Discord (reading messages, deleting messages, posting in admin channel).                      | Credential configuration in n8n.                                                                                |
| The workflow is designed for a forex trading Discord community but can be adapted by modifying the AI prompt and training examples for other communities or themes.                                                           | AI Agent system message and Google Sheets training data.                                                        |

---

**Disclaimer:** The provided text is solely derived from a workflow automated using n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly available.