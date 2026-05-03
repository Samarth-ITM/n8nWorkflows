Automated WhatsApp Group Weekly Team Reports with Gemini AI Summarization

https://n8nworkflows.xyz/workflows/automated-whatsapp-group-weekly-team-reports-with-gemini-ai-summarization-6528


# Automated WhatsApp Group Weekly Team Reports with Gemini AI Summarization

### 1. Workflow Overview

This workflow automates the generation of weekly summary reports for a WhatsApp group team by leveraging AI-based summarization (Google Gemini AI). It targets teams using WhatsApp as a primary communication channel who want to capture, analyze, and share highlights, wins, and challenges from the past week's conversations. The workflow operates on a weekly schedule (Monday 6 AM), collects all messages, fetches replies, summarizes threads, generates individual team member reports, compiles a team-wide report, and finally posts the report to the team channel.

The workflow is logically organized into the following functional blocks:

- **1.1 Scheduled Trigger & Data Collection:** Automatically triggers every Monday morning and fetches all WhatsApp group messages from the previous week.
- **1.2 Message Grouping & Data Mining:** Groups messages by user and retrieves all replies to those messages to provide full conversation context.
- **1.3 Thread Summarization:** Uses AI to summarize individual message threads including replies.
- **1.4 Individual Team Member Reporting:** Aggregates summarized threads per user and generates a weekly report focusing on wins, challenges, and interactions.
- **1.5 Team-wide Report Generation:** Compiles individual reports into one comprehensive weekly team summary using AI.
- **1.6 Report Delivery:** Posts the final summarized report to the team’s WhatsApp channel.

Sub-workflows are extensively used to handle looping and data processing tasks, simplifying complex item references and ensuring linear data flow.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Data Collection

- **Overview:** Initiates the workflow every Monday at 6 AM and collects all WhatsApp group messages from the last 7 days.
- **Nodes Involved:** 
  - Monday @ 6am (Schedule Trigger)
  - WhapAround.pro (Respond to Webhook)
  - Group By User (Code)
  - Split Out (Split Out)
  - Messages to Items (Code)
  - Simplify Message (Set)
  - Aggregate (Aggregate)
  - Get User Info (Set)

- **Node Details:**

  - **Monday @ 6am**
    - Type: Schedule Trigger
    - Configured to run weekly at 6:00 AM on Mondays (cron expression: `0 6 * * 1`)
    - Triggers workflow execution.
    - Outputs trigger to "WhapAround.pro".

  - **WhapAround.pro**
    - Type: Respond to Webhook
    - Role: Receives incoming webhook data containing WhatsApp group messages.
    - No specific configuration; expects data payload for processing.
    - Connects to "Group By User".

  - **Group By User**
    - Type: Code Node
    - Groups all incoming messages by the user who sent them.
    - Uses JavaScript to create an object indexed by `user` with arrays of messages.
    - Outputs grouped messages as an array under `data`.
    - Input: Raw messages JSON array.
    - Output: Grouped messages by user.

  - **Split Out**
    - Type: Split Out
    - Splits the grouped message data array into individual items for downstream processing.
    - Input: grouped user messages.
    - Output: individual user message groups.

  - **Messages to Items**
    - Type: Code Node
    - Converts the grouped message structure into individual message items.
    - Uses data from the "Switch" node's first output to extract message arrays.
    - Output: individual messages for processing.

  - **Simplify Message**
    - Type: Set
    - Simplifies the message JSON structure to core fields: timestamp, userId, userName, message type, and cleans the text by removing user mentions.
    - Pulls user info from the "Get User" node.
    - Output: simplified message objects.

  - **Aggregate**
    - Type: Aggregate
    - Aggregates all simplified messages into a field named `messages` for further processing.

  - **Get User Info**
    - Type: Set
    - Extracts and organizes user information (`id`, `team_id`, `name`, `is_bot`).
    - Includes other fields implicitly.
    - Used for user context in later steps.

- **Edge Cases & Failures:**
  - Missing or malformed webhook data could cause failures.
  - If user info is incomplete or missing, subsequent node expressions may fail.
  - Timezone or timestamp parsing errors may occur if timestamps are inconsistent.

---

#### 1.2 Message Grouping & Data Mining

- **Overview:** For each user’s messages, fetch all replies to obtain full conversation threads and context.
- **Nodes Involved:** 
  - Switch
  - Map Users to Messages (Execute Workflow)
  - Fetch Message Replies (Execute Workflow)
  - Summarize Message Threads (Execute Workflow)
  - Split Out1 (Split Out)
  - Loop Over Items (Split In Batches)
  - Filter
  - Simplify Thread Comments (Set)
  - Aggregate1 (Aggregate)
  - Message Ref (NoOp)
  - Split Out2 (Split Out)
  - Map Reply UserIds (Set)
  - Has ReplyUsers? (If)
  - Aggregate Reply Users (Aggregate)
  - Reply Users (Set)
  - Messages to Items1 (Code)

- **Node Details:**

  - **Switch**
    - Type: Switch
    - Routes workflow based on the `action` input (`users`, `message_replies`, or `message_summarize`).
    - Controls flow for:
      - Grouping users and messages,
      - Fetching replies,
      - Summarizing threads.

  - **Map Users to Messages**
    - Type: Execute Workflow (Self-reference)
    - Runs the workflow in "each" mode for each user group to fetch message replies.
    - Passes user data and action `users` to the subworkflow.
    - Handles recursion and isolated processing.

  - **Fetch Message Replies**
    - Type: Execute Workflow (Self-reference)
    - For each message, fetches all replies.
    - Passes message data with action `message_replies`.
    - Enables recursive data mining of all threads.

  - **Summarize Message Threads**
    - Type: Execute Workflow (Self-reference)
    - Summarizes each message thread using AI.
    - Action is `message_summarize`.
    - Runs in "each" mode to summarize independently.

  - **Split Out1**
    - Type: Split Out
    - Splits the aggregated messages object for processing.

  - **Loop Over Items**
    - Type: Split In Batches
    - Processes items in batches for efficient handling.
    - Avoids overloading nodes with many items simultaneously.

  - **Filter**
    - Type: Filter
    - Filters out duplicate or already processed messages based on timestamp.

  - **Simplify Thread Comments**
    - Type: Set
    - Simplifies replies in the thread to core fields (timestamp, userId, type, text).

  - **Aggregate1**
    - Type: Aggregate
    - Aggregates simplified replies.

  - **Message Ref**
    - Type: NoOp
    - Placeholder node to maintain item references.

  - **Split Out2**
    - Type: Split Out
    - Splits aggregated replyUsers array for processing.

  - **Map Reply UserIds**
    - Type: Set
    - Extracts unique user IDs from replies for user info enrichment.

  - **Has ReplyUsers?**
    - Type: If
    - Checks if there are any reply users to process.
    - Routes flow accordingly.

  - **Aggregate Reply Users**
    - Type: Aggregate
    - Aggregates all reply user data.

  - **Reply Users**
    - Type: Set
    - Sets the reply users data in an array for downstream usage.

  - **Messages to Items1**
    - Type: Code
    - Converts summarized message threads into items to prepare for AI summarization.

- **Edge Cases & Failures:**
  - Missing or incomplete replies will reduce thread context.
  - Large volumes of messages/replies may cause timeout or memory issues.
  - Filtering logic needs to be carefully maintained to avoid data loss.
  - Sub-workflow recursion requires careful error handling to avoid infinite loops.

---

#### 1.3 Thread Summarization

- **Overview:** Uses Google Gemini AI to summarize each message thread, highlighting important content and replies.
- **Nodes Involved:** 
  - Google Gemini Chat Model
  - Summarise Threads (Chain LLM)
  - Aggregate2 (Aggregate)
  - Messages to Items1 (Code)

- **Node Details:**

  - **Google Gemini Chat Model**
    - Type: LangChain Google Gemini LLM Chat Node
    - Model: `models/gemini-2.0-flash`
    - Processes prompt with messages including user names, timestamps, message text, and replies.
    - Summarizes each thread for content highlights, wins, challenges.

  - **Summarise Threads**
    - Type: Chain LLM
    - Inputs formatted message and reply text for summarization.
    - Prompt instructs AI to summarize topics, decisions, challenges.
    - Uses dynamic expressions to pull user names and timestamps.

  - **Aggregate2**
    - Type: Aggregate
    - Aggregates summarized threads for downstream consumption.

- **Edge Cases & Failures:**
  - AI model failures or rate limits.
  - Input text length limits may truncate long conversations.
  - Expression errors in prompt formatting could distort AI input.

---

#### 1.4 Individual Team Member Reporting

- **Overview:** Aggregates summarized threads per user and generates a weekly report focusing on achievements, challenges, and collaboration.
- **Nodes Involved:** 
  - Aggregate Results (Set)
  - Team Member Weekly Report Agent (Chain LLM)
  - Google Gemini Chat Model1 (Google Gemini LLM Chat)
  - Merge with Results (Set)

- **Node Details:**

  - **Aggregate Results**
    - Type: Set
    - Combines user messages with summaries.
    - Prepares data for AI to generate reports.

  - **Team Member Weekly Report Agent**
    - Type: Chain LLM
    - Uses all summarized threads per user.
    - Prompt asks AI to write mini-reports with motivation, highlights, challenges, welcomes, and farewells.
    - Formatting includes user names, timestamps, and summaries.

  - **Google Gemini Chat Model1**
    - Type: LangChain Google Gemini LLM Chat Node
    - Model: `models/gemini-2.0-flash`
    - Runs AI prompt for weekly report generation.

  - **Merge with Results**
    - Type: Set
    - Merges AI-generated report text with aggregated user data.

- **Edge Cases & Failures:**
  - AI prompt misinterpretation.
  - Missing user data could cause incomplete reports.
  - Long text inputs may hit model size limits.

---

#### 1.5 Team-wide Report Generation

- **Overview:** Combines all individual user reports into one comprehensive team report to motivate and align the whole team.
- **Nodes Involved:** 
  - Team Weekly Report Agent (Chain LLM)
  - Google Gemini Chat Model2 (Google Gemini LLM Chat)
  - Respond to Webhook

- **Node Details:**

  - **Team Weekly Report Agent**
    - Type: Chain LLM
    - Aggregates all user reports.
    - AI prompt instructs to produce a motivational, comprehensive overview.
    - Highlights team-wide wins, challenges, connections, welcomes, and farewells.

  - **Google Gemini Chat Model2**
    - Type: LangChain Google Gemini LLM Chat Node
    - Model: `models/gemini-2.0-flash`
    - Runs the final summarization prompt.

  - **Respond to Webhook**
    - Type: Respond to Webhook
    - Sends the final report as output, ready for posting or further delivery.

- **Edge Cases & Failures:**
  - AI model latency or failures.
  - Extremely large inputs could cause truncation.
  - Response formatting issues.

---

#### 1.6 Report Delivery

- **Overview:** (Implied, no explicit node in JSON for WhatsApp posting)
- The final step would typically involve posting the generated report to the WhatsApp group channel. The workflow ends by responding to the webhook, implying integration with an external system or another node to send the report message.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                              | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                       |
|-----------------------------|----------------------------------|----------------------------------------------|---------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| Monday @ 6am                | Schedule Trigger                 | Starts workflow every Monday at 6 AM         | -                               | WhapAround.pro                    |                                                                                                 |
| WhapAround.pro              | Respond to Webhook               | Receives WhatsApp group messages              | Monday @ 6am                   | Group By User                    | ## 1. Fetch All Activity from Last Week [Learn more about the WhapAround.pro node]               |
| Group By User               | Code                            | Groups messages by user                        | WhapAround.pro                 | Split Out                       |                                                                                                 |
| Split Out                  | Split Out                       | Splits grouped user messages                   | Group By User                  | Map Users to Messages            | ## 2. Summarise Messages Threads & Conversations (with link to Execute Workflow docs)            |
| Map Users to Messages       | Execute Workflow                | Runs subworkflow for each user’s messages    | Split Out                     | Fetch Message Replies            |                                                                                                 |
| Fetch Message Replies       | Execute Workflow                | Fetches replies for each message              | Map Users to Messages          | Summarize Message Threads        |                                                                                                 |
| Summarize Message Threads   | Execute Workflow                | Summarizes each message thread                 | Fetch Message Replies          | Aggregate Results                |                                                                                                 |
| Messages to Items           | Code                            | Converts grouped messages to individual items | whapAround.pro                | Simplify Message                |                                                                                                 |
| Simplify Message            | Set                             | Simplifies message object                       | Messages to Items              | Aggregate                      |                                                                                                 |
| Aggregate                  | Aggregate                      | Aggregates simplified messages                  | Simplify Message              | Get User Info                  |                                                                                                 |
| Get User Info               | Set                             | Extracts user info                              | Aggregate                    | Map Users to Messages           |                                                                                                 |
| Switch                     | Switch                         | Routes workflow based on action input          | When Executed by Another Workflow | whapAround.pro, Split Out1, Map Reply UserIds |                                                                                                 |
| Split Out1                 | Split Out                      | Splits messages for processing                  | Switch (message_replies)       | Loop Over Items                 |                                                                                                 |
| Loop Over Items             | Split In Batches               | Processes items in batches                        | Aggregate1, Split Out1          | Aggregate3, Message Ref         |                                                                                                 |
| Filter                     | Filter                         | Filters duplicate or processed messages         | whapAround.pro-1              | Simplify Thread Comments        |                                                                                                 |
| Simplify Thread Comments    | Set                            | Simplifies thread replies                         | Filter                       | Aggregate1                     |                                                                                                 |
| Aggregate1                 | Aggregate                      | Aggregates simplified thread comments             | Simplify Thread Comments       | Split Out1                    |                                                                                                 |
| Message Ref                | NoOp                           | Maintains item reference                          | Loop Over Items                | whapAround.pro-1               |                                                                                                 |
| Split Out2                 | Split Out                      | Splits reply user IDs                              | Has ReplyUsers?               | whapAround.pro-2               |                                                                                                 |
| Map Reply UserIds           | Set                            | Extracts unique reply user IDs                    | Switch (message_summarize)    | Has ReplyUsers?                |                                                                                                 |
| Has ReplyUsers?             | If                             | Checks presence of reply users                     | Map Reply UserIds             | Split Out2 (true), Reply Users (false) |                                                                                                 |
| Aggregate Reply Users       | Aggregate                      | Aggregates reply user data                          | whapAround.pro-2              | Reply Users                   |                                                                                                 |
| Reply Users                | Set                            | Sets reply users data                               | Aggregate Reply Users         | Messages to Items1             |                                                                                                 |
| Messages to Items1          | Code                           | Converts summarized threads into items             | Reply Users                  | Summarise Threads             |                                                                                                 |
| Summarise Threads           | Chain LLM                      | AI summarization of message threads                | Messages to Items1            | Aggregate2                    |                                                                                                 |
| Aggregate2                 | Aggregate                      | Aggregates summarized threads                       | Summarise Threads             | -                            |                                                                                                 |
| Aggregate Results           | Set                            | Combines user messages and summaries                | Summarize Message Threads     | Team Member Weekly Report Agent | ## 3. Generate Activity Reports for Each Team Member (with link to Basic LLM node)               |
| Team Member Weekly Report Agent | Chain LLM                 | AI generates weekly report per team member          | Aggregate Results             | Merge with Results            |                                                                                                 |
| Google Gemini Chat Model1   | LM Chat Google Gemini          | Runs AI prompt for team member report               | Team Member Weekly Report Agent | Merge with Results            |                                                                                                 |
| Merge with Results          | Set                            | Merges AI report with aggregated data                | Team Member Weekly Report Agent | Team Weekly Report Agent      |                                                                                                 |
| Team Weekly Report Agent    | Chain LLM                      | AI generates final team-wide weekly report           | Merge with Results            | Respond to Webhook            | ## 4. Generate Final Report for Whole Team (with link to Basic LLM node)                        |
| Google Gemini Chat Model2   | LM Chat Google Gemini          | Runs AI prompt for team-wide report                   | Team Weekly Report Agent      | Respond to Webhook            |                                                                                                 |
| Respond to Webhook          | Respond to Webhook             | Outputs the final team report                          | Team Weekly Report Agent      | -                            | ## 5. Post Report on Team Channel (on Monday Morning!) (with link to Slack node docs)           |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point for external workflow execution           | -                           | Switch                      | ## 5. SubWorkflows (with link to Execute Workflow Trigger docs)                                |
| Sticky Note                 | Sticky Note                   | Notes on fetching last week’s WhatsApp group activity | -                           | -                            | ## 1. Fetch All Activity from Last Week [Learn more about the WhapAround.pro node]               |
| Sticky Note1                | Sticky Note                   | Notes on summarizing message threads & conversations  | -                           | -                            | ## 2. Summarise Messages Threads & Conversations (with link to Execute Workflow docs)            |
| Sticky Note2                | Sticky Note                   | Notes on generating individual team member reports     | -                           | -                            | ## 3. Generate Activity Reports for Each Team Member (with link to Basic LLM node)               |
| Sticky Note3                | Sticky Note                   | Notes on generating final team report                   | -                           | -                            | ## 4. Generate Final Report for Whole Team (with link to Basic LLM node)                        |
| Sticky Note4                | Sticky Note                   | Notes on posting report to team channel                 | -                           | -                            | ## 5. Post Report on Team Channel (on Monday Morning!) (with link to Slack node docs)           |
| Sticky Note5                | Sticky Note                   | Notes on use of subworkflows for complex item loops     | -                           | -                            | ## 5. SubWorkflows (with link to Execute Workflow Trigger docs)                                |
| Sticky Note6                | Sticky Note                   | Workflow overview, problem, solution, and setup tips    | -                           | -                            | ## Try It Out! WhatsApp Weekly Summary Template... (full content in node)                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node:**
   - Add `Schedule Trigger` node named "Monday @ 6am".
   - Configure to trigger weekly every Monday at 6:00 AM (cron expression: `0 6 * * 1`).

2. **Create WhatsApp Message Reception Node:**
   - Add `Respond to Webhook` node named "WhapAround.pro".
   - Configure to receive WhatsApp group messages (expects webhook from WhapAround.pro or similar).

3. **Group Messages by User:**
   - Add `Code` node named "Group By User".
   - Use JavaScript code to reduce incoming messages into an object keyed by user containing arrays of messages.
   - Input: Data from "WhapAround.pro".
   - Output: Object with user keys and messages array.

4. **Split Grouped User Messages:**
   - Add `Split Out` node named "Split Out".
   - Split the grouped user messages array into individual items.

5. **Map Users to Messages via Subworkflow:**
   - Add `Execute Workflow` node named "Map Users to Messages".
   - Set mode to "each" to process each user group individually.
   - Pass inputs with action set to "users" and current data.
   - Connect "Split Out" output to this node.

6. **Simplify Messages:**
   - Add `Set` node named "Simplify Message".
   - Map fields: timestamp (`ts`), userId, userName (from "Get User" node), message type, and cleaned text (remove user mentions).
   - Input: individual messages.

7. **Aggregate Simplified Messages:**
   - Add `Aggregate` node named "Aggregate".
   - Aggregate all simplified messages into a field named `messages`.

8. **Get User Info:**
   - Add `Set` node named "Get User Info".
   - Extract user info fields (`id`, `team_id`, `name`, `is_bot`) from the "Get User" node.

9. **Configure Action Routing:**
   - Add `Execute Workflow Trigger` node named "When Executed by Another Workflow".
   - Define inputs: `action` (string), `data` (object).
   - Add `Switch` node named "Switch" to route based on `action` (`users`, `message_replies`, `message_summarize`).

10. **Fetch Message Replies:**
    - Add `Execute Workflow` node named "Fetch Message Replies".
    - Mode: "each".
    - Pass input data with action `message_replies`.
    - Connect from "Map Users to Messages".

11. **Split & Loop over Replies:**
    - Add `Split Out` node "Split Out1".
    - Add `Split In Batches` node "Loop Over Items" for batch processing.
    - Add `Filter` node "Filter" to exclude already processed messages.
    - Add `Set` node "Simplify Thread Comments" to simplify replies.
    - Add `Aggregate` node "Aggregate1" to aggregate simplified replies.
    - Add `NoOp` node "Message Ref" to maintain references.

12. **Map Reply User IDs:**
    - Add `Set` node "Map Reply UserIds" to extract unique userIds from replies.
    - Add `If` node "Has ReplyUsers?" to check if reply users exist.

13. **Aggregate Reply Users:**
    - Add `Aggregate` node "Aggregate Reply Users".
    - Add `Set` node "Reply Users" to set reply user data.

14. **Convert Summarized Threads to Items:**
    - Add `Code` node "Messages to Items1".
    - Extract summarized threads from "Reply Users".

15. **Summarize Threads Using AI:**
    - Add `Chain LLM` node "Summarise Threads".
    - Configure prompt to summarize conversations focusing on highlights, challenges, and accomplishments.
    - Add `Google Gemini Chat Model` node configured with model `models/gemini-2.0-flash`.
    - Connect "Messages to Items1" to "Summarise Threads" to "Google Gemini Chat Model".

16. **Aggregate Summarized Threads:**
    - Add `Aggregate` node "Aggregate2".

17. **Aggregate Results for User Reports:**
    - Add `Set` node "Aggregate Results".
    - Combine messages and summaries for each user.

18. **Generate Team Member Weekly Reports:**
    - Add `Chain LLM` node "Team Member Weekly Report Agent".
    - Prompt configured to generate motivational, personalized reports.
    - Add `Google Gemini Chat Model1` node.
    - Connect "Aggregate Results" → "Team Member Weekly Report Agent" → "Google Gemini Chat Model1".

19. **Merge AI Reports with Data:**
    - Add `Set` node "Merge with Results".

20. **Generate Team-wide Weekly Report:**
    - Add `Chain LLM` node "Team Weekly Report Agent".
    - Prompt configured to produce a comprehensive team summary.
    - Add `Google Gemini Chat Model2` node.
    - Connect "Merge with Results" → "Team Weekly Report Agent" → "Google Gemini Chat Model2".

21. **Respond with Final Report:**
    - Add `Respond to Webhook` node.
    - Output the final team report.

22. **Configure Subworkflows:**
    - Ensure subworkflows invoked by "Execute Workflow" nodes are configured to accept `action` and `data` inputs.
    - Use the same workflow for recursion with clear switch routing on `action`.

23. **Credentials:**
    - Set up OpenAI or Google Gemini AI credentials for LLM nodes.
    - Configure webhook URLs and authentication for WhapAround.pro or WhatsApp integrations.

24. **Validation & Testing:**
    - Test each block individually.
    - Monitor for API rate limits, data completeness, and AI response issues.
    - Use sticky notes as guidance for configuration and best practices.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow automatically collects WhatsApp group messages weekly, summarizes conversations via AI, and posts reports to keep the team aligned and motivated.    | Workflow purpose                                                                                             |
| Use WhapAround.pro for WhatsApp message collection.                                                                                                            | https://whaparound.pro (implied from node names)                                                             |
| Subworkflows simplify looping over nested message threads and replies, ensuring easier item referencing and predictable data flow.                             | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflowtrigger                      |
| Use Google Gemini AI models `models/gemini-2.0-flash` for high-quality summarization and report generation.                                                     | AI model selection                                                                                           |
| For more on executing workflows and sub-workflows see n8n docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow           | Documentation link                                                                                           |
| Customize report tone and style by editing the AI prompt texts in the Chain LLM nodes.                                                                          | Prompt customization                                                                                         |
| Ensure timezones and timestamps are consistent for accurate report chronology.                                                                                   | Timestamp handling note                                                                                      |
| Consider Slack or WhatsApp API nodes for final report delivery; currently, the workflow ends with webhook response ready for integration.                      | Suggested extension for report posting                                                                      |

---

This detailed documentation provides both technical and functional insight into the workflow, enabling developers and AI agents to understand, reproduce, and enhance the WhatsApp Weekly Team Reports automation with clarity and confidence.