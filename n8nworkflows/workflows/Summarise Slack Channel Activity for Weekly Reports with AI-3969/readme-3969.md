Summarise Slack Channel Activity for Weekly Reports with AI

https://n8nworkflows.xyz/workflows/summarise-slack-channel-activity-for-weekly-reports-with-ai-3969


# Summarise Slack Channel Activity for Weekly Reports with AI

### 1. Workflow Overview

This workflow automates the process of summarizing weekly activity from a specified Slack channel to generate insightful team reports using AI. It targets remote teams who rely heavily on Slack for communication, aiming to extract meaningful information from numerous conversations and threads that often get lost over time.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Data Fetch**: Automatically triggers every Monday at 6 AM to fetch all messages from the last week in a specific Slack channel.
- **1.2 Data Grouping and User Info Retrieval**: Groups messages by user, fetches user details, and prepares data for further processing.
- **1.3 Message Thread Expansion**: Retrieves all replies to each message thread to capture full conversation context.
- **1.4 Thread-level Summarization**: Uses AI to summarize individual message threads, highlighting key points.
- **1.5 Individual User Weekly Report Generation**: Aggregates summarized threads per user and generates personalized weekly reports using AI.
- **1.6 Team-wide Weekly Report Generation**: Combines all individual reports into a comprehensive team weekly report via AI.
- **1.7 Posting Final Report**: Posts the generated team report back to the Slack channel at the start of the week.

Sub-workflows and iterative loops are used extensively to handle nested data, especially for processing messages and replies on a per-user basis.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Data Fetch

- **Overview:**  
  The workflow starts with a scheduled trigger every Monday at 6 AM that fetches all Slack channel messages from the previous week.

- **Nodes Involved:**  
  - Monday @ 6am (Schedule Trigger)  
  - Get Last Week's Messages (Slack node)  
  - Group By User (Code node)  
  - Split Out (Split Out node)

- **Node Details:**  
  - **Monday @ 6am**  
    - Type: Schedule Trigger  
    - Configured to trigger based on a cron expression: every Monday at 6:00 AM.  
    - Outputs no input, triggers the workflow execution.
    - Edge cases: If the workflow or server is down at trigger time, the schedule may be missed.

  - **Get Last Week's Messages**  
    - Type: Slack node  
    - Operation: Fetch channel history with a filter to get messages from the last week using `$now.minus('1', 'week')`.  
    - Channel is fixed (e.g., general channel ID).  
    - Returns all messages without pagination using `returnAll: true`.  
    - Potential issues: Slack API rate limits, auth token expiration.

  - **Group By User**  
    - Type: Code node  
    - Groups fetched messages by the message author's user ID.  
    - Input: all messages from Slack node.  
    - Output: array of message arrays grouped by user.  
    - Failure modes: If message data lacks user fields, grouping may fail.

  - **Split Out**  
    - Type: Split Out node  
    - Splits grouped user message arrays into separate items, one per user.  
    - Input: grouped messages object.  
    - Output: individual user message groups for downstream processing.

---

#### 1.2 Data Grouping and User Info Retrieval

- **Overview:**  
  For each user group, fetch detailed user info from Slack and simplify messages for easier processing.

- **Nodes Involved:**  
  - Map Users to Messages (Execute Workflow node - subworkflow)  
  - Get User (Slack node)  
  - Messages to Items (Code node)  
  - Simplify Message (Set node)  
  - Aggregate (Aggregate node)  
  - Get User Info (Set node)

- **Node Details:**  
  - **Map Users to Messages**  
    - Type: Execute Workflow  
    - Runs the main workflow recursively with action `users` and the user message data.  
    - Handles each user separately in a sub-execution for clarity.  
    - Potential failure: Recursive calls may increase execution time and resource use.

  - **Get User**  
    - Type: Slack node  
    - Fetches user details by user ID from the grouped messages.  
    - Returns user metadata such as name, team ID, and bot status.  
    - Failures: Invalid user ID or Slack API issues.

  - **Messages to Items**  
    - Type: Code node  
    - Converts grouped user messages into individual items for iteration.

  - **Simplify Message**  
    - Type: Set node  
    - Extracts and simplifies message fields: timestamp, userId, userName, type, and cleaned text (removes Slack mentions).  
    - Uses expressions referencing the `Get User` node to insert user details.

  - **Aggregate**  
    - Type: Aggregate node  
    - Aggregates all simplified messages back into a single array under the field `messages`.

  - **Get User Info**  
    - Type: Set node  
    - Creates a user object with selected user info fields for later reporting.

---

#### 1.3 Message Thread Expansion

- **Overview:**  
  This block fetches all replies for each message to capture full conversation threads.

- **Nodes Involved:**  
  - Fetch Message Replies (Execute Workflow node - subworkflow)  
  - Message Ref (NoOp node)  
  - Get Thread (Slack node)  
  - Filter (Filter node)  
  - Simplify Thread Comments (Set node)  
  - Aggregate1 (Aggregate node)  
  - Loop Over Items (Split In Batches node)  
  - Aggregate3 (Set node)  
  - Aggregate4 (Aggregate node)

- **Node Details:**  
  - **Fetch Message Replies**  
    - Type: Execute Workflow  
    - Runs the main workflow recursively with action `message_replies` to process each top-level message for replies.

  - **Message Ref**  
    - Type: NoOp node  
    - Reference node for message data, used to maintain data flow.

  - **Get Thread**  
    - Type: Slack node  
    - Fetches replies to a message using the original message timestamp (`ts`) and channel ID.  
    - Returns all replies in the thread.

  - **Filter**  
    - Type: Filter node  
    - Filters out the original message from replies to avoid duplication by comparing timestamps.

  - **Simplify Thread Comments**  
    - Type: Set node  
    - Simplifies replies extracting fields similar to messages: timestamp, userId, type, and text.

  - **Aggregate1**  
    - Type: Aggregate node  
    - Aggregates all simplified replies into an array under `replies`.

  - **Loop Over Items**  
    - Type: Split In Batches node  
    - Loops over each message with its replies to process them sequentially.

  - **Aggregate3**  
    - Type: Set node  
    - Merges original message data with filtered replies.

  - **Aggregate4**  
    - Type: Aggregate node  
    - Aggregates all processed messages with replies into a single array.

---

#### 1.4 Thread-level Summarization

- **Overview:**  
  Summarizes each message thread, including replies, using the Google Gemini AI model.

- **Nodes Involved:**  
  - Google Gemini Chat Model (LLM node)  
  - Summarise Threads (Chain LLM node)  
  - Aggregate2 (Aggregate node)

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: AI language model node (Google Gemini)  
    - Model used: `models/gemini-2.0-flash`.  
    - Provides LLM capabilities for summarization.

  - **Summarise Threads**  
    - Type: Chain LLM node  
    - Input: Formatted text including message user, timestamp, text, and replies details with corresponding user names.  
    - Prompt: Summarizes the slack message thread, highlighting achievements, challenges, and significant points.  
    - Outputs a summary text per thread.

  - **Aggregate2**  
    - Type: Aggregate node  
    - Aggregates all individual thread summaries.

---

#### 1.5 Individual User Weekly Report Generation

- **Overview:**  
  Aggregates summarized threads per user and generates an AI-powered weekly report for each team member.

- **Nodes Involved:**  
  - Summarize Message Threads (Execute Workflow node - subworkflow)  
  - Aggregate Results (Set node)  
  - Team Member Weekly Report Agent (Chain LLM node)  
  - Google Gemini Chat Model1 (LLM node)  
  - Merge with Results (Set node)

- **Node Details:**  
  - **Summarize Message Threads**  
    - Type: Execute Workflow  
    - Runs recursively with action `message_summarize` for each user’s messages and summaries.

  - **Aggregate Results**  
    - Type: Set node  
    - Combines original messages with their AI-generated summaries.

  - **Team Member Weekly Report Agent**  
    - Type: Chain LLM node  
    - Uses the aggregated summarized threads to create a mini-report per user.  
    - Prompt encourages motivational and insightful tone focusing on wins, challenges, and team dynamics.

  - **Google Gemini Chat Model1**  
    - Type: AI language model node (Google Gemini)  
    - Provides the LLM capability for generating user reports.

  - **Merge with Results**  
    - Type: Set node  
    - Adds the AI-generated report text into the user's data object.

---

#### 1.6 Team-wide Weekly Report Generation

- **Overview:**  
  Combines individual user reports into a consolidated team report using AI.

- **Nodes Involved:**  
  - Team Weekly Report Agent (Chain LLM node)  
  - Google Gemini Chat Model2 (LLM node)  
  - Merge with Results (Set node)

- **Node Details:**  
  - **Team Weekly Report Agent**  
    - Type: Chain LLM node  
    - Input: All individual reports with user info and message counts.  
    - Prompt: Creates a motivational and comprehensive team report summarizing all activities, recognizing patterns, and highlighting wins and challenges.

  - **Google Gemini Chat Model2**  
    - Type: AI language model node (Google Gemini)  
    - Provides LLM capabilities for the team report.

  - **Merge with Results**  
    - This merges the team report text with existing data for posting.

---

#### 1.7 Posting Final Report

- **Overview:**  
  Posts the generated team weekly report back to the Slack channel.

- **Nodes Involved:**  
  - Post Report in Team Channel (Slack node)

- **Node Details:**  
  - **Post Report in Team Channel**  
    - Type: Slack node  
    - Posts a message to the specified Slack channel (same channel as messages fetched).  
    - Text parameter uses the AI-generated report content.  
    - Potential failure: Slack API rate limits or posting permissions.

---

#### Sub-Workflow Handling & Control Nodes

- **When Executed by Another Workflow (Execute Workflow Trigger node)**  
  - Acts as entry point for recursive calls with an `action` parameter directing flow.  
  - Used to handle iterations over users, message replies, and message summarizations in sub-executions.

- **Switch Node**  
  - Routes execution paths based on `action` parameter (`users`, `message_replies`, `message_summarize`).  
  - Ensures appropriate processing logic is applied in sub-workflow calls.

- **Split Out / Split In Batches Nodes**  
  - Used extensively to split arrays into individual items for sequential or parallel processing.  
  - Helps manage large data sets without overwhelming memory or execution contexts.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                         | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                              |
|-----------------------------|-----------------------------------|---------------------------------------------------------|--------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------|
| Monday @ 6am                | Schedule Trigger                  | Triggers workflow every Monday at 6am                   | -                              | Get Last Week's Messages           |                                                                                                        |
| Get Last Week's Messages     | Slack node                       | Fetches messages from Slack channel for past week       | Monday @ 6am                   | Group By User                     | [Learn more about the Slack node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack) |
| Group By User                | Code node                       | Groups messages by user ID                               | Get Last Week's Messages        | Split Out                        | ## 1. Fetch All Activity from Last Week: Fetch and group messages by author                             |
| Split Out                   | Split Out node                  | Splits grouped messages into individual user groups     | Group By User                  | Map Users to Messages             |                                                                                                        |
| Map Users to Messages        | Execute Workflow                | Runs sub-workflow per user for message processing       | Split Out                     | Fetch Message Replies             | ## 5. SubWorkflows: Using subworkflows to handle loops and nested data                                 |
| Get User                    | Slack node                     | Fetches detailed user info from Slack                    | Map Users to Messages          | Messages to Items                 |                                                                                                        |
| Messages to Items            | Code node                     | Converts grouped messages into individual items          | Get User                      | Simplify Message                 |                                                                                                        |
| Simplify Message             | Set node                      | Simplifies message data and cleans text                   | Messages to Items              | Aggregate                       |                                                                                                        |
| Aggregate                   | Aggregate node                | Aggregates simplified messages into array                | Simplify Message              | Get User Info                   |                                                                                                        |
| Get User Info                | Set node                      | Creates user object with selected user properties         | Aggregate                     |                                 |                                                                                                        |
| Fetch Message Replies        | Execute Workflow              | Runs sub-workflow to get replies for each message        | Map Users to Messages          |                               | ## 2. Summarise Messages Threads & Conversations: Fetch all replies for full context                    |
| Message Ref                 | NoOp node                     | Reference node for message data                            | Loop Over Items                | Get Thread                     |                                                                                                        |
| Get Thread                  | Slack node                   | Fetches replies to a message thread                        | Message Ref                   | Filter                         |                                                                                                        |
| Filter                      | Filter node                   | Filters out the original message from replies             | Get Thread                    | Simplify Thread Comments        |                                                                                                        |
| Simplify Thread Comments     | Set node                      | Simplifies replies data                                    | Filter                        | Aggregate1                     |                                                                                                        |
| Aggregate1                  | Aggregate node                | Aggregates simplified replies                              | Simplify Thread Comments      | Loop Over Items                |                                                                                                        |
| Loop Over Items             | Split In Batches              | Loops over messages with replies for sequential processing | Aggregate1                    | Aggregate3 / Message Ref       |                                                                                                        |
| Aggregate3                  | Set node                      | Merges message with filtered replies                       | Loop Over Items               | Aggregate4                    |                                                                                                        |
| Aggregate4                  | Aggregate node                | Aggregates all processed messages with replies            | Aggregate3                   |                               |                                                                                                        |
| Google Gemini Chat Model     | LLM node                     | Provides AI model for summarization                        | Summarise Threads             | Summarise Threads             |                                                                                                        |
| Summarise Threads           | Chain LLM node               | Summarizes individual message threads                      | Messages to Items1            | Aggregate2                    |                                                                                                        |
| Aggregate2                  | Aggregate node                | Aggregates all thread summaries                            | Summarise Threads             |                               |                                                                                                        |
| Summarize Message Threads    | Execute Workflow              | Runs sub-workflow per user for thread summarization       | Fetch Message Replies         | Aggregate Results             |                                                                                                        |
| Aggregate Results           | Set node                      | Combines messages with thread summaries                    | Summarize Message Threads     | Team Member Weekly Report Agent |                                                                                                        |
| Team Member Weekly Report Agent | Chain LLM node               | Generates weekly report per user from summarized threads  | Aggregate Results             | Merge with Results            | ## 3. Generate Activity Reports for Each Team Member: AI generates personalized mini-reports           |
| Google Gemini Chat Model1    | LLM node                     | AI model for individual report generation                  | Team Member Weekly Report Agent | Team Member Weekly Report Agent |                                                                                                        |
| Merge with Results          | Set node                      | Merges AI report text with user data                       | Team Member Weekly Report Agent | Team Weekly Report Agent       |                                                                                                        |
| Team Weekly Report Agent     | Chain LLM node               | Generates combined team report from individual reports    | Merge with Results            | Post Report in Team Channel    | ## 4. Generate Final Report for Whole Team: AI aggregates all individual reports into a team summary   |
| Google Gemini Chat Model2    | LLM node                     | AI model for team report generation                         | Team Weekly Report Agent      | Team Weekly Report Agent       |                                                                                                        |
| Post Report in Team Channel  | Slack node                   | Posts the final team weekly report to Slack channel       | Team Weekly Report Agent      |                               | ## 5. Post Report on Team Channel (on Monday Morning!): Final report delivery                          |
| When Executed by Another Workflow | Execute Workflow Trigger    | Entry point for subworkflow calls, routes by action param  | -                            | Switch                        |                                                                                                        |
| Switch                      | Switch node                  | Routes execution based on action: users, message_replies, or message_summarize | When Executed by Another Workflow | Get User / Split Out1 / Map Reply UserIds |                                                                                                        |
| Map Reply UserIds           | Set node                     | Extracts unique user IDs from replies for lookup           | Switch                       | Has ReplyUsers?               |                                                                                                        |
| Has ReplyUsers?             | If node                      | Checks if reply users exist, branches accordingly          | Map Reply UserIds             | Split Out2 / Reply Users       |                                                                                                        |
| Split Out1                  | Split Out node               | Splits messages with replies for processing                 | Switch                       | Loop Over Items               |                                                                                                        |
| Split Out2                  | Split Out node               | Splits reply users for user info retrieval                  | Has ReplyUsers?               | Get Reply Users               |                                                                                                        |
| Get Reply Users             | Slack node                   | Fetches user info for all reply user IDs                    | Split Out2                   | Aggregate Reply Users         |                                                                                                        |
| Aggregate Reply Users       | Aggregate node               | Aggregates all reply user info                               | Get Reply Users               | Reply Users                  |                                                                                                        |
| Reply Users                | Set node                     | Sets aggregated reply user data                              | Aggregate Reply Users         | Messages to Items1            |                                                                                                        |
| Messages to Items1          | Code node                   | Converts aggregated reply user data into items              | Reply Users                  | Summarise Threads            |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Scheduled Trigger**  
- Add a `Schedule Trigger` node named "Monday @ 6am".  
- Configure to trigger weekly on Monday at 6:00 AM using cron expression `0 6 * * 1`.

**Step 2: Fetch Last Week's Slack Messages**  
- Add a Slack node named "Get Last Week's Messages" with:  
  - Resource: Channel  
  - Operation: history  
  - Channel ID: your target Slack channel (e.g., general)  
  - Filters: oldest = `={{ $now.minus('1', 'week') }}`, inclusive = false  
  - Return all messages (`returnAll: true`)  
- Credential: Slack API with appropriate OAuth2 token.

**Step 3: Group Messages by User**  
- Add a Code node "Group By User" with JavaScript code to group messages by `user` field into an object mapping user ID to array of messages.  
- Output data as array of message arrays (one per user).

**Step 4: Split Grouped Messages**  
- Add a Split Out node "Split Out" to split grouped user messages into separate items.

**Step 5: Create Sub-workflow for User Processing (Map Users to Messages)**  
- Add an Execute Workflow node "Map Users to Messages" pointing to the main workflow itself (self-reference).  
- Pass data: action = "users", data = current user message group.

**Step 6: In Subworkflow, Handle "users" Action**  
- Add an Execute Workflow Trigger node "When Executed by Another Workflow" to receive inputs.  
- Add a Switch node "Switch" to route by `action` input: if `users`, continue with user processing.

**Step 7: Get User Info**  
- Add Slack node "Get User" to fetch user info by user ID from incoming data.  
- Add Code node "Messages to Items" to convert messages array into individual items.  
- Add Set node "Simplify Message" to map message fields: timestamp, userId, userName, type, and cleaned text (strip Slack mentions).  
- Add Aggregate node "Aggregate" to combine simplified messages into an array.  
- Add Set node "Get User Info" to create an object with user details for further use.

**Step 8: Fetch Replies for Each Message (Subworkflow: Fetch Message Replies)**  
- Add an Execute Workflow node "Fetch Message Replies" in the parent workflow calling the main workflow with action `message_replies` and message data.  
- In subworkflow, handle `message_replies` action:  
  - Use a Split Out node to iterate over messages.  
  - For each message, use Slack node "Get Thread" to fetch all replies by timestamp and channel ID.  
  - Use Filter node to exclude the original message from replies.  
  - Use Set node "Simplify Thread Comments" to map replies fields.  
  - Aggregate replies with Aggregate1 node.  
  - Loop over items with Split In Batches node to process each message and its replies.  
  - Merge replies with original message using Set node "Aggregate3".  
  - Aggregate all messages and replies with Aggregate4.

**Step 9: Extract Reply User IDs and Fetch User Info**  
- Use Set node "Map Reply UserIds" to extract unique user IDs from replies.  
- Use If node "Has ReplyUsers?" to branch if reply users exist.  
- If yes, Split Out2 node to split reply users.  
- Slack node "Get Reply Users" to fetch info for each reply user.  
- Aggregate reply users and set in "Reply Users" node.

**Step 10: Summarize Each Message Thread with AI**  
- Use Code node "Messages to Items1" to convert reply user data into items.  
- Use Chain LLM node "Summarise Threads" with Google Gemini model to generate summaries per thread.  
- Aggregate all thread summaries.

**Step 11: Generate Individual Weekly Reports**  
- Use Execute Workflow node "Summarize Message Threads" with action `message_summarize` per user.  
- Aggregate results combining original messages and summaries.  
- Use Chain LLM node "Team Member Weekly Report Agent" with AI prompt to generate mini weekly report for each user.  
- Merge AI-generated report text into user data.

**Step 12: Generate Team-wide Weekly Report**  
- Aggregate all individual user reports.  
- Use Chain LLM node "Team Weekly Report Agent" with AI prompt to combine individual reports into a team report.  
- Merge final report data.

**Step 13: Post Final Report to Slack**  
- Use Slack node "Post Report in Team Channel" to post final team report text to the chosen Slack channel.  
- Ensure Slack credentials have permission to post messages.

**Additional Configuration:**  
- Credentials for Slack and Google Gemini API must be set up with appropriate access tokens.  
- Ensure rate limits are handled or workflow retried if Slack or AI services throttle requests.  
- Customize prompts in AI nodes to tailor tone and content of reports.  
- Adjust scheduled trigger time or channel as needed for your team.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Try It Out! This n8n template lets you summarize individual team member activity on Slack for the past week and generates a report. | Full project description and usage instructions are included in the workflow sticky notes.         |
| Learn more about the Slack node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack                          | Useful for customizing Slack API calls and filters.                                                |
| Learn more about Execute Workflow node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow        | Important for implementing sub-workflow and recursive calls.                                       |
| Learn more about Basic LLM node: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm    | To customize AI summarization and report generation prompts.                                       |
| Join the Discord or ask in the n8n Forum for help:                                                                              | Discord invite: https://discord.com/invite/XPKeKXeB7d ; Forum: https://community.n8n.io/           |
| Consider posting reports to email or integrating project metrics to enrich reports.                                               | Customization advice for adapting workflow to different team needs.                                |
| Use an AI agent to query knowledge-base or tickets relevant to messages for enhanced context.                                    | Suggested advanced extension to this workflow.                                                     |

---

This documentation provides a detailed, structured understanding of the workflow architecture and logic, enabling proficient users or AI systems to reproduce, modify, or troubleshoot the workflow confidently.