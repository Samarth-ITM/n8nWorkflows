Prioritize Todoist Tasks with OpenRouter AI and Send Daily Summaries to Slack

https://n8nworkflows.xyz/workflows/prioritize-todoist-tasks-with-openrouter-ai-and-send-daily-summaries-to-slack-9564


# Prioritize Todoist Tasks with OpenRouter AI and Send Daily Summaries to Slack

---

### 1. Workflow Overview

This workflow automates the prioritization of daily tasks from Todoist using AI and delivers a structured daily summary to a Slack channel every morning. It is designed for users who want to optimize their daily productivity by leveraging AI to rank tasks based on urgency, importance, dependencies, and effort, and then receive actionable summaries via Slack.

**Logical blocks:**

- **1.1 Morning Trigger:** Initiates the workflow at 8 AM daily.
- **1.2 Task Retrieval:** Fetches all incomplete tasks from Todoist.
- **1.3 AI Prioritization:** Uses an AI agent (LangChain with OpenRouter GPT-4) to analyze tasks and generate a prioritized list with reasoning and warnings.
- **1.4 Parsing AI Output:** Structures the AI response into defined JSON objects for further processing.
- **1.5 Formatting Summary:** Converts the structured AI output into a readable Markdown message.
- **1.6 Delivery:** Sends the formatted message to a designated Slack channel using OAuth2.

---

### 2. Block-by-Block Analysis

#### 2.1 Morning Trigger

- **Overview:**  
  Automatically triggers the workflow at 8 AM every day to start the task prioritization process.

- **Nodes Involved:**  
  - Morning Schedule Trigger  
  - Sticky Note - Trigger

- **Node Details:**  
  - **Morning Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow run daily at a fixed time.  
    - Configuration: Set to trigger at hour 8 (8 AM).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Get Todo List" node.  
    - Edge Cases: Misconfiguration of time zone or interval could cause missed or multiple triggers.

  - **Sticky Note - Trigger**  
    - Type: Sticky Note  
    - Role: Documentation block explaining the trigger purpose and schedule adjustability.  
    - Inputs/Outputs: None.

#### 2.2 Task Retrieval

- **Overview:**  
  Connects to the Todoist API to fetch all incomplete tasks to be analyzed.

- **Nodes Involved:**  
  - Get Todo List  
  - Sticky Note - Tasks

- **Node Details:**  
  - **Get Todo List**  
    - Type: Todoist Node  
    - Role: Retrieves all tasks marked incomplete from Todoist.  
    - Configuration: Operation set to "getAll" with no filters, returning all tasks.  
    - Credentials: OAuth2 authenticated with Todoist account.  
    - Inputs: Receives trigger from "Morning Schedule Trigger".  
    - Outputs: Feeds raw task list JSON to "AI Task Analyzer".  
    - Edge Cases: OAuth token expiration, API rate limits, no tasks returned, or connection errors.  
    - Version: Version 2 of Todoist node.

  - **Sticky Note - Tasks**  
    - Type: Sticky Note  
    - Role: Provides setup instructions and notes on possible task management alternatives.

#### 2.3 AI Prioritization

- **Overview:**  
  An AI agent processes the raw task data, evaluating each task’s urgency, importance, dependencies, and effort to produce a ranked and annotated task list.

- **Nodes Involved:**  
  - AI Task Analyzer  
  - OpenRouter Chat Model  
  - Task Priority Parser  
  - Sticky Note - AI

- **Node Details:**  
  - **AI Task Analyzer**  
    - Type: LangChain AI Agent  
    - Role: Sends task data to an AI model for analysis and receives structured prioritization and reasoning.  
    - Configuration:  
      - Input: Raw JSON of tasks as text input.  
      - System Message: Detailed prompt instructing the AI on how to prioritize tasks.  
      - Output: Structured JSON with prioritized tasks, daily summary, and warnings.  
      - Output parser: Enabled.  
    - Inputs: Receives tasks from "Get Todo List".  
    - Outputs: Passes AI response to "Format AI Summary".  
    - Edge Cases: AI API failures, malformed input data, unexpected AI output format.  
    - Version: Version 2.  
    - Sub-Workflow: None.

  - **OpenRouter Chat Model**  
    - Type: LangChain OpenRouter Chat Model  
    - Role: Provides GPT-4 based language model backend for the AI agent.  
    - Credentials: OpenRouter API key.  
    - Inputs/Outputs: Connected as the language model backend for "AI Task Analyzer".  
    - Edge Cases: API key invalid, rate limiting, network errors.  
    - Version: 1.

  - **Task Priority Parser**  
    - Type: LangChain Output Parser (Structured)  
    - Role: Parses AI agent’s raw text output into a structured JSON schema with fields like prioritizedTasks array, dailySummary string, and warnings array.  
    - Configuration: Manual schema defining expected properties and types.  
    - Inputs: Connected as output parser for "AI Task Analyzer".  
    - Outputs: Sends structured data back to "AI Task Analyzer" for downstream consumption.  
    - Edge Cases: Schema mismatch, parsing errors if AI output deviates.

  - **Sticky Note - AI**  
    - Type: Sticky Note  
    - Role: Explains AI functionality and configuration hints for credentials and prompt customization.

#### 2.4 Formatting Summary

- **Overview:**  
  Formats the structured AI output into a readable Markdown message suitable for Slack posting.

- **Nodes Involved:**  
  - Format AI Summary  
  - Sticky Note - Format

- **Node Details:**  
  - **Format AI Summary**  
    - Type: Set Node  
    - Role: Constructs a formatted string message containing:  
      - Current date header  
      - Daily summary from AI  
      - Prioritized task list with rank, reasoning, suggested time, and urgency  
      - Conditional warnings section if any exist  
    - Configuration: Uses JavaScript expressions to map over tasks and build multi-line Markdown text.  
    - Inputs: Takes structured AI output JSON.  
    - Outputs: Provides a single string field `formattedMessage` for Slack.  
    - Edge Cases: Empty task list, missing fields in AI output, expression evaluation errors.  
    - Version: 3.4.

  - **Sticky Note - Format**  
    - Type: Sticky Note  
    - Role: Documentation on the formatting logic and output structure.

#### 2.5 Delivery

- **Overview:**  
  Sends the formatted daily summary message to a Slack channel via OAuth2 authentication.

- **Nodes Involved:**  
  - Send to Slack  
  - Sticky Note - Delivery

- **Node Details:**  
  - **Send to Slack**  
    - Type: Slack Node  
    - Role: Posts the formatted priority message to a configured Slack channel.  
    - Configuration:  
      - Text: Uses the `formattedMessage` from the previous node.  
      - Channel: OAuth2 authenticated channel selector; requires user setup.  
      - Authentication: OAuth2 with Slack credentials.  
    - Inputs: Receives formatted message from "Format AI Summary".  
    - Outputs: None (terminal node).  
    - Edge Cases: Invalid Slack OAuth token, wrong channel ID, network issues, message length limits.  
    - Version: 2.2.

  - **Sticky Note - Delivery**  
    - Type: Sticky Note  
    - Role: Provides setup instructions and notes on alternative delivery options such as Microsoft Teams, Discord, SMS, or push notifications.

#### 2.6 Template Overview

- **Overview:**  
  Provides a high-level explanation of the workflow structure, intended usage, and setup notes.

- **Nodes Involved:**  
  - Sticky Note - Template Overview

- **Node Details:**  
  - **Sticky Note - Template Overview**  
    - Type: Sticky Note  
    - Content: Explains the 4-step process, self-hosted requirement for LangChain nodes, credential setup, and schedule customization.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                 | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|-------------------------|-------------------------------------|--------------------------------|-------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Morning Schedule Trigger | Schedule Trigger                    | Starts workflow daily at 8 AM  | —                       | Get Todo List           | ## 🌅 Morning Trigger This workflow runs every morning at 8 AM to analyze and prioritize your daily tasks. Adjust the schedule time in the trigger node to match your preferred morning routine. |
| Get Todo List           | Todoist                            | Fetches incomplete tasks       | Morning Schedule Trigger | AI Task Analyzer        | ## 📋 Task Retrieval Fetches all incomplete tasks from your Todoist account. Setup Required: Connect your Todoist credentials You can replace this with Google Tasks, Notion, or any other task management tool |
| AI Task Analyzer        | LangChain AI Agent                 | Analyzes and prioritizes tasks | Get Todo List            | Format AI Summary        | ## 🤖 AI Analysis The AI Agent analyzes your tasks using GPT-4 to create intelligent prioritization. Configuration needed: Add your OpenAI API credentials Adjust the system prompt if you want different prioritization criteria The output parser ensures structured data for downstream processing |
| OpenRouter Chat Model   | LangChain OpenRouter Chat Model    | Provides GPT-4 language model  | — (used by AI Task Analyzer) | AI Task Analyzer (backend) |                                                                                                |
| Task Priority Parser    | LangChain Output Parser Structured | Parses AI output to JSON       | AI Task Analyzer (ai_outputParser) | AI Task Analyzer (ai_outputParser) |                                                                                                |
| Format AI Summary       | Set                               | Formats AI output to Markdown  | AI Task Analyzer         | Send to Slack           | ## 📝 Message Formatting Formats the AI's prioritized task list into a readable message format. The formatted output includes: Date header Daily summary Prioritized task list with details Any critical warnings |
| Send to Slack           | Slack                             | Sends message to Slack channel | Format AI Summary        | —                       | ## 📬 Delivery Options Choose how you want to receive your daily priorities: Slack Setup: Add Slack credentials Configure the channel or DM Other Options: You can add nodes for: Microsoft Teams Discord SMS (Twilio) Push notifications |
| Sticky Note - Trigger   | Sticky Note                       | Documentation block            | —                       | —                       | ## 🌅 Morning Trigger This workflow runs every morning at 8 AM to analyze and prioritize your daily tasks. Adjust the schedule time in the trigger node to match your preferred morning routine. |
| Sticky Note - Tasks     | Sticky Note                       | Documentation block            | —                       | —                       | ## 📋 Task Retrieval Fetches all incomplete tasks from your Todoist account. Setup Required: Connect your Todoist credentials You can replace this with Google Tasks, Notion, or any other task management tool |
| Sticky Note - AI        | Sticky Note                       | Documentation block            | —                       | —                       | ## 🤖 AI Analysis The AI Agent analyzes your tasks using GPT-4 to create intelligent prioritization. Configuration needed: Add your OpenAI API credentials Adjust the system prompt if you want different prioritization criteria The output parser ensures structured data for downstream processing |
| Sticky Note - Format    | Sticky Note                       | Documentation block            | —                       | —                       | ## 📝 Message Formatting Formats the AI's prioritized task list into a readable message format. The formatted output includes: Date header Daily summary Prioritized task list with details Any critical warnings |
| Sticky Note - Delivery  | Sticky Note                       | Documentation block            | —                       | —                       | ## 📬 Delivery Options Choose how you want to receive your daily priorities: Slack Setup: Add Slack credentials Configure the channel or DM Other Options: You can add nodes for: Microsoft Teams Discord SMS (Twilio) Push notifications |
| Sticky Note - Template Overview | Sticky Note               | Documentation block            | —                       | —                       | ## Template Overview (Self-hosted only) This template uses a community node (LangChain) and is intended for self-hosted n8n. How it works 1. 8 AM trigger 2. Fetch Todoist tasks 3. AI prioritization 4. Format & send to Slack Setup - Connect OAuth2 for Todoist/Slack - Set `slackChannelId` in User Config - Adjust schedule if needed |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 8:00 AM.

2. **Add a Todoist node:**  
   - Operation: Get All Tasks  
   - Filters: None (fetch all incomplete)  
   - Authentication: OAuth2 with your Todoist account credentials.  
   - Connect the output of the Schedule Trigger to this node.

3. **Add an OpenRouter Chat Model node:**  
   - Type: LangChain OpenRouter Chat Model  
   - Credentials: Provide OpenRouter API key (or suitable GPT-4 compatible API key).  
   - No direct input/output connections except to AI Agent.

4. **Add a LangChain AI Agent node:**  
   - Input: Set input to the raw JSON from the Todoist node (`{{$json}}`).  
   - System Message:  
     ```
     You are an expert task prioritization assistant. Analyze the provided tasks and create a prioritized list based on:

     1. Urgency: Deadlines and time-sensitive items
     2. Importance: Impact on goals and objectives
     3. Dependencies: Tasks that unlock other work
     4. Effort: Quick wins vs. complex projects

     For each task, provide:
     - Priority rank (1-10, where 1 is highest priority)
     - Reasoning for the priority
     - Suggested time block for completion
     - Any warnings about overdue or critical items

     Be concise but thorough in your analysis.
     ```  
   - Enable Output Parser.  
   - Assign the OpenRouter Chat Model as the language model in the AI Agent settings.  
   - Connect output of Todoist node to this AI Agent node.

5. **Add a LangChain Output Parser (Structured) node:**  
   - Define manual JSON schema with properties:  
     - `prioritizedTasks`: array of objects with `rank` (number), `taskName` (string), `reasoning` (string), `suggestedTimeBlock` (string), `urgencyLevel` (string).  
     - `dailySummary`: string.  
     - `warnings`: array of strings (optional).  
   - Connect its output parser input to the AI Agent output parser port.

6. **Add a Set node for formatting:**  
   - Create a new string field `formattedMessage`.  
   - Use JavaScript expression to build a Markdown message including:  
     - Current date header (e.g., "## 🗓️ Today's Priority Tasks - April 27, 2024")  
     - Daily summary text from AI output  
     - Loop over `prioritizedTasks` to list rank, task name, reasoning, suggested time, urgency  
     - If any warnings exist, add a warnings section.  
   - Connect AI Agent output to this node.

7. **Add a Slack node:**  
   - Authentication: OAuth2 with Slack credentials.  
   - Channel: Select or set the default Slack channel ID for posting.  
   - Text: Use the `formattedMessage` field from the Set node.  
   - Connect output of the Set node to this Slack node.

8. **Connect nodes to form the flow:**  
   - Schedule Trigger → Get Todo List → AI Task Analyzer → Format AI Summary → Send to Slack.

9. **Add Sticky Notes for documentation (optional):**  
   - Add explanatory notes beside each logical block for maintainability.

10. **Configure credentials:**  
    - Setup OAuth2 credentials for Todoist and Slack in n8n credentials manager.  
    - Add OpenRouter API key credentials for LangChain nodes.

11. **Test workflow:**  
    - Run manually or wait for scheduled trigger.  
    - Verify task retrieval, AI prioritization, message formatting, and Slack delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This template uses LangChain community nodes and requires self-hosted n8n (not compatible with n8n.cloud).                               | Sticky Note - Template Overview                                                                     |
| For Slack delivery, ensure OAuth2 credentials have permission to post messages to the target channel.                                     | Sticky Note - Delivery                                                                              |
| You can customize the AI system prompt to reflect your specific prioritization criteria or task management style.                        | Sticky Note - AI                                                                                    |
| Alternative task sources like Google Tasks or Notion can replace Todoist by swapping the "Get Todo List" node with appropriate API nodes. | Sticky Note - Tasks                                                                                  |
| Message formatting uses Markdown compatible with Slack for rich text display.                                                             | Sticky Note - Format                                                                                |
| Adjust the schedule trigger time to fit your personal morning routine or timezone.                                                        | Sticky Note - Trigger                                                                               |

---

**Disclaimer:**  
The provided content is extracted exclusively from an n8n workflow automation and adheres strictly to applicable content policies. It contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.