Track Personal Finances in Google Sheets with AI Agent via Slack

https://n8nworkflows.xyz/workflows/track-personal-finances-in-google-sheets-with-ai-agent-via-slack-10056


# Track Personal Finances in Google Sheets with AI Agent via Slack

### 1. Workflow Overview

This workflow automates personal finance tracking using Google Sheets as the data store, Slack as the user interface, and AI for natural language processing and decision-making. It is designed to facilitate daily financial check-ins and allow users to log transactions, debts, and balances through conversational Slack messages. The AI agent handles parsing, validation, calculation, and updates, ensuring accurate and seamless finance management.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Daily Check-in:** Automatically retrieves current balances and active debts from Google Sheets, formats a daily summary message, and sends it to a Slack channel as a prompt for user input.
  
- **1.2 Slack Message Trigger & AI Processing:** Listens for Slack messages that mention the bot (usually containing user transactions), then processes these messages using an AI agent that parses transactions, validates data, calculates updated balances, and prepares responses.

- **1.3 AI Agent Tools Integration:** The AI agent integrates multiple specialized tools such as language models, memory storage, calculators, and Google Sheets read/write operations to process data and maintain conversational context.

- **1.4 Data Reading Nodes:** Dedicated nodes fetch current financial data (balances, transactions, debts) from Google Sheets to feed the AI agent’s calculations and decision-making.

- **1.5 Data Writing Nodes:** After user approval, the AI agent updates Google Sheets with new or modified transaction, debt, and balance records.

- **1.6 Slack Messaging Nodes:** Nodes responsible for sending formatted messages and confirmations back to users on Slack.

- **1.7 Utility Nodes:** Supporting nodes including code for message formatting and merging data streams.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Daily Check-in

**Overview:**  
This block triggers every day at 11 PM to fetch the latest balances and active debts, formats a summary message, and sends it to Slack to prompt the user for daily transactions.

**Nodes Involved:**  
- Daily checkin trigger  
- Get Current Balance  
- Get Active Debts  
- Merge  
- Format Daily Message  
- Send Slack Message  

**Node Details:**

- **Daily checkin trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the daily check-in at 23:00 (11 PM) daily.  
  - Configuration: Scheduled for hour 23 every day.  
  - Inputs: None  
  - Outputs: Triggers downstream nodes to fetch balance and debts.  
  - Failure cases: Scheduler misconfiguration or disabled node.  

- **Get Current Balance**  
  - Type: Google Sheets Read  
  - Role: Reads balance data from Google Sheets, specifically columns A-D as defined in a range.  
  - Configuration: Reads specified range from the "Balances" sheet (sheetName and documentId configured externally).  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Trigger from schedule  
  - Outputs: JSON data array of balances  
  - Failure cases: Authentication errors, sheet not found, empty or malformed data.

- **Get Active Debts**  
  - Type: Google Sheets Read  
  - Role: Reads data about active debts from the "Debts" sheet (columns A-F).  
  - Configuration: Similar to Get Current Balance, but for debts.  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Trigger from schedule  
  - Outputs: JSON data array of debts  
  - Failure cases: Authentication errors, sheet missing, data format issues.

- **Merge**  
  - Type: Merge  
  - Role: Combines data from balances and debts into a single stream for processing.  
  - Configuration: Default merge mode (append).  
  - Inputs: From Get Current Balance and Get Active Debts  
  - Outputs: Single merged JSON array  
  - Failure cases: None specific, but must ensure both inputs complete.

- **Format Daily Message**  
  - Type: Code (JavaScript)  
  - Role: Processes merged data to create a user-friendly daily summary message.  
  - Configuration: Custom JS code that filters balances and debts, calculates total debts, and builds a formatted message with instructions/examples for user input.  
  - Inputs: Merged data stream from Merge node  
  - Outputs: JSON with formatted message string and extracted finance data  
  - Failure cases: JS runtime errors, unexpected data shapes.

- **Send Slack Message**  
  - Type: Slack Node (Send Message)  
  - Role: Sends the formatted daily summary message to a Slack channel to prompt user input.  
  - Configuration: Sends message to configured Slack channel using Slack API credentials; message text is expression-evaluated from previous node output.  
  - Inputs: Message from Format Daily Message  
  - Outputs: Confirmation of message sent  
  - Failure cases: Slack API errors, invalid channel, rate limits.

---

#### 1.2 Slack Message Trigger & AI Processing

**Overview:**  
This block listens for Slack app mentions (user messages directed at the bot), and sends message text to the AI Agent for parsing, validation, balance calculation, and response formulation.

**Nodes Involved:**  
- Bot Mention trigger  
- AI Agent  
- replying to the user  

**Node Details:**

- **Bot Mention trigger**  
  - Type: Slack Trigger  
  - Role: Listens for Slack messages mentioning the bot, which contain user input transactions or commands.  
  - Configuration: Trigger on "app_mention" event in Slack channel, channel ID configured externally.  
  - Credentials: Slack API  
  - Inputs: Slack app mention events  
  - Outputs: Message JSON including text to AI Agent  
  - Failure cases: Slack API errors, missing permissions.

- **AI Agent**  
  - Type: LangChain AI Agent Node  
  - Role: Core processing node that interprets user messages, parses transactions, interacts with memory and calculation tools, and prepares responses or updates.  
  - Configuration:  
    - Uses Google Gemini Chat Model as language model.  
    - System message defines the agent role, database schema, workflow phases, parsing logic, emojis, and error handling.  
    - Connects to multiple tools: Think (reasoning), Calculator, Postgres Chat Memory (conversation state), Google Sheets tools (Get Balances, Get Transactions, Get Debts, Append/Update rows), and Slack messaging for replies.  
  - Inputs: User message text from Slack trigger  
  - Outputs: JSON responses to send to Slack or further processing  
  - Version-specific: Requires n8n LangChain nodes and Google Gemini API credentials.  
  - Failure cases: API errors, parsing errors, missing data, database connectivity issues, invalid user input.

- **replying to the user**  
  - Type: Slack Node (Send Message)  
  - Role: Sends AI agent’s response back to the user in Slack channel.  
  - Configuration: Uses Slack API with configured channel, sends message text from AI Agent output.  
  - Inputs: AI Agent output  
  - Outputs: Slack message confirmation  
  - Failure cases: Slack API errors, invalid channel, rate limits.

---

#### 1.3 AI Agent Tools Integration

**Overview:**  
This block includes all specialized tools integrated into the AI Agent for natural language understanding, memory management, balance calculations, and Google Sheets data operations.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Think  
- Calculator  
- Postgres Chat Memory  
- Get Balances  
- Get Transactions  
- Get Debts  
- Adding/updating Transactions  
- Adding/updating debts  
- Adding/updating balances  

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Language Model (Google Gemini)  
  - Role: Processes language understanding and generation for the AI Agent.  
  - Configuration: Uses Google Palm API credentials.  
  - Inputs/Outputs: Receives text prompts, returns AI-generated text.  
  - Failure cases: API errors, rate limits, invalid credentials.

- **Think**  
  - Type: LangChain Tool (Reasoning)  
  - Role: Performs internal reasoning tasks such as parsing messages, decision-making, and validation.  
  - Inputs/Outputs: Feeds into and from the AI Agent.  
  - Failure cases: Runtime errors, malformed input.

- **Calculator**  
  - Type: LangChain Tool  
  - Role: Performs arithmetic calculations on balances and transactions.  
  - Inputs/Outputs: Numeric data from AI Agent, returns calculated results.  
  - Failure cases: Calculation errors, invalid input.

- **Postgres Chat Memory**  
  - Type: LangChain Memory Node  
  - Role: Stores conversation context and parsed data in PostgreSQL for session continuity.  
  - Configuration: Uses a custom session key and table for finance tracker conversations.  
  - Credentials: PostgreSQL connection required.  
  - Inputs/Outputs: Connected to AI Agent memory interface.  
  - Failure cases: Database connection errors, query failures.

- **Get Balances, Get Transactions, Get Debts**  
  - Type: Google Sheets Tool (Read operations)  
  - Role: Fetches current data from respective sheets for AI processing.  
  - Configuration: Uses configured Google Sheets document and sheet names.  
  - Credentials: Google Sheets OAuth2  
  - Inputs/Outputs: Provide data for AI calculations.  
  - Failure cases: Authentication errors, missing sheets, empty data.

- **Adding/updating Transactions, debts, balances**  
  - Type: Google Sheets Tool (Append or Update)  
  - Role: Writes or updates records in respective sheets based on AI agent’s approved transactions.  
  - Configuration: Maps parsed fields like Date, Amount, Person_Name, Status, etc., to sheet columns; uses keys like Transaction_ID or Person_Name for matching.  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Data from AI Agent after approval  
  - Outputs: Confirmation of successful update  
  - Failure cases: Write permission errors, matching conflicts, API limits.

---

#### 1.4 Utility Nodes

**Overview:**  
Supporting nodes that handle data formatting and provide user guidance or notes.

**Nodes Involved:**  
- Sticky Note (multiple)  
- Merge (already covered)  
- Format Daily Message (covered in 1.1)  

**Node Details:**

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Provide documentation and warnings directly within the workflow editor.  
  - Content covers: Agent usage notes, scheduled check-in overview, data retrieval, and modification notes, general project overview and prerequisites.  
  - No inputs or outputs.  
  - Failure cases: None (non-executable).

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                           | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                      |
|-----------------------------|----------------------------------|-----------------------------------------|---------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Daily checkin trigger        | Schedule Trigger                 | Triggers daily balance/debt check-in    | None                            | Get Current Balance, Get Active Debts | ## Scheduled checkin                                                                            |
| Get Current Balance          | Google Sheets                   | Reads current balances from sheet       | Daily checkin trigger           | Merge                          |                                                                                                 |
| Get Active Debts             | Google Sheets                   | Reads active debts from sheet            | Daily checkin trigger           | Merge                          |                                                                                                 |
| Merge                       | Merge                           | Merges balances and debts data           | Get Current Balance, Get Active Debts | Format Daily Message           |                                                                                                 |
| Format Daily Message         | Code (JS)                      | Formats daily summary message             | Merge                          | Send Slack Message             |                                                                                                 |
| Send Slack Message           | Slack Node (Send Message)       | Sends daily summary message to Slack     | Format Daily Message            | None                          |                                                                                                 |
| Bot Mention trigger          | Slack Trigger                   | Listens for Slack bot mentions           | None                           | AI Agent                      |                                                                                                 |
| AI Agent                    | LangChain AI Agent              | Parses messages, manages logic & updates | Bot Mention trigger, Think, Calculator, Postgres Chat Memory, Google Gemini Chat Model, Get Balances, Get Transactions, Get Debts, Adding/updating Transactions, Adding/updating debts, Adding/updating balances | replying to the user           | ## Agent Note: Never trigger flow with normal message to avoid infinite loop. Claude recommended. |
| replying to the user         | Slack Node (Send Message)       | Replies to user with AI agent output     | AI Agent                      | None                          |                                                                                                 |
| Google Gemini Chat Model     | LangChain LM Chat Model          | Provides language understanding          | AI Agent                      | AI Agent                      |                                                                                                 |
| Think                       | LangChain Tool (Reasoning)      | AI reasoning and parsing                  | AI Agent                      | AI Agent                      |                                                                                                 |
| Calculator                  | LangChain Tool (Calculator)     | Performs balance calculations             | AI Agent                      | AI Agent                      |                                                                                                 |
| Postgres Chat Memory        | LangChain Memory (Postgres)     | Stores conversation context               | AI Agent                      | AI Agent                      |                                                                                                 |
| Get Balances                | Google Sheets Tool (Read)       | Reads balances sheet for AI processing   | AI Agent                      | AI Agent                      | ## Retrieving data                                                                             |
| Get Transactions            | Google Sheets Tool (Read)       | Reads transactions sheet for AI           | AI Agent                      | AI Agent                      | ## Retrieving data                                                                             |
| Get Debts                   | Google Sheets Tool (Read)       | Reads debts sheet for AI                   | AI Agent                      | AI Agent                      | ## Retrieving data                                                                             |
| Adding/updating Transactions | Google Sheets Tool (Append/Update) | Writes new/updated transaction records   | AI Agent                      | AI Agent                      | ## Adding/Modifying data                                                                       |
| Adding/updating debts        | Google Sheets Tool (Append/Update) | Writes new/updated debt records           | AI Agent                      | AI Agent                      | ## Adding/Modifying data                                                                       |
| Adding/updating balances     | Google Sheets Tool (Append/Update) | Writes new/updated balances                | AI Agent                      | AI Agent                      | ## Adding/Modifying data                                                                       |
| Sticky Note                 | Sticky Note                     | Documentation and warnings                | None                         | None                          | ## Various notes including overview, agent warnings, and instructions                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Scheduled Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 23:00 (11 PM).

2. **Create Google Sheets Read Nodes for Balances and Debts**  
   - Add two Google Sheets nodes: "Get Current Balance" and "Get Active Debts".  
   - Configure each to read respective sheets ("Balances" and "Debts") specifying the data ranges (A:D and A:F).  
   - Set Google Sheets OAuth2 credentials.  
   - Connect both to the output of the Scheduled Trigger node.

3. **Add Merge Node**  
   - Add a Merge node to merge outputs of "Get Current Balance" and "Get Active Debts".  
   - Connect outputs of both Sheets nodes to Merge node inputs.

4. **Create Code Node "Format Daily Message"**  
   - Add a Code node with custom JavaScript to process merged data:  
     - Parse balances and debts.  
     - Calculate total active debts.  
     - Format a message with current balances, debts, and instructions/examples for user transactions.  
   - Connect output of Merge node to this node.

5. **Add Slack Node "Send Slack Message"**  
   - Configure Slack node to send a message to a predefined channel.  
   - Use expression to set message text from "Format Daily Message" output (`{{$json.message}}`).  
   - Configure Slack API credentials.  
   - Connect "Format Daily Message" output to this node.

6. **Create Slack Trigger Node "Bot Mention trigger"**  
   - Add a Slack Trigger node configured to trigger on "app_mention" events.  
   - Configure Slack API credentials and specify monitored channel(s).

7. **Add AI Agent Node**  
   - Add a LangChain AI Agent node.  
   - Configure the system prompt as per the workflow description, defining roles, tools, workflow phases, emojis, error handling, and data schema.  
   - Connect Slack Trigger output to AI Agent input.

8. **Configure AI Agent Tools**  
   - Add and connect the following nodes as sub-tools to the AI Agent:  
     - Google Gemini Chat Model (linked as language model) with Google Palm API credentials.  
     - Think tool for reasoning.  
     - Calculator tool for computations.  
     - Postgres Chat Memory with PostgreSQL credentials and session parameters.  
     - Google Sheets Tools: Get Balances, Get Transactions, Get Debts (configured to read respective sheets).  
     - Google Sheets Tools: Adding/updating Transactions, debts, balances configured for appendOrUpdate operations with appropriate mapping of columns and keys.  
   - Connect all these tools to the AI Agent node using the respective interfaces (`ai_tool`, `ai_languageModel`, `ai_memory`).

9. **Add Slack Node "replying to the user"**  
   - Configure Slack node to send final responses from AI Agent back to Slack channel.  
   - Connect AI Agent output to this node.

10. **Add Sticky Notes**  
    - Optionally add Sticky Note nodes for documentation and reminders inside the workflow editor.

11. **Connect Data Flow**  
    - Ensure:  
      - Scheduled check-in triggers data fetch → merge → format → Slack message.  
      - Slack mention triggers AI Agent → AI Agent uses tools → final reply to Slack.

12. **Credentials Setup**  
    - Configure Google Sheets OAuth2 with appropriate access to the documents.  
    - Configure Slack API with bot token and required permissions.  
    - Configure Google Palm API credentials for Gemini model.  
    - Configure PostgreSQL credentials for memory storage.

13. **Testing & Validation**  
    - Test scheduled message delivery.  
    - Test Slack mention with sample transaction messages.  
    - Verify AI Agent parses, previews, and updates sheets correctly.  
    - Monitor for error handling and edge cases.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| AI Agent Warning: Never trigger the flow with normal messages to avoid infinite loop from the bot replying to itself. Claude model recommended for best performance.                                                                                                                                                                                               | Sticky Note in workflow near AI Agent node                                  |
| Project Overview: AI-powered Slack bot for personal finance tracking, supporting natural language parsing, balance calculation, debt management, and daily reminders. Requires Slack workspace, Google Sheets with three tabs, Google Gemini API, PostgreSQL, and n8n instance.                                                                                      | Sticky Note near workflow start                                            |
| Daily check-in scheduled at 11 PM sends a summary of current balances and active debts, prompting user for transaction input via Slack.                                                                                                                                                                                                                            | Sticky Note near Daily checkin trigger                                     |
| Retrieving and updating data from Google Sheets uses dedicated nodes with OAuth2 credentials.                                                                                                                                                                                                                                                                       | Sticky Notes near Google Sheets read/write nodes                           |
| Data Structure:  
  - Balances Sheet: Date, Cash_Balance, Online_Balance, Total_Balance  
  - Transactions Sheet: Transaction_ID, Date, Time, Amount, Payment_Type, Category, Transaction_Type, Person_Name, Description, Added_At  
  - Debts Sheet: Person_Name, Amount, Type, Date_Created, Status, Notes                                                                                                                             | Detailed in AI Agent system message                                       |
| Workflow Logic Flow: Message received → Think if transaction → Parse and store → Get current data → Calculate → Preview → Await approval → Update sheets → Confirm message.                                                                                                                                                | Detailed in AI Agent system message                                       |
| Error handling includes messages for unparseable inputs, missing debt persons, and database read/write errors.                                                                                                                                                                                                                                                    | Detailed in AI Agent system message                                       |
| Emojis used for transaction types and categories for enhanced readability in Slack messages.                                                                                                                                                                                                                                                                        | Detailed in AI Agent system message                                       |
| For best AI integration, Google Gemini API and PostgreSQL memory storage recommended.                                                                                                                                                                                                                                                                               | Credential requirements                                                    |

---

**Disclaimer:** This documentation is generated exclusively from an automated n8n workflow. All data and processes comply with legal and ethical guidelines.