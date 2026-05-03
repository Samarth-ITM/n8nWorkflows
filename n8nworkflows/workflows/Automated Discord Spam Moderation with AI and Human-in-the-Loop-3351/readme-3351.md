Automated Discord Spam Moderation with AI and Human-in-the-Loop

https://n8nworkflows.xyz/workflows/automated-discord-spam-moderation-with-ai-and-human-in-the-loop-3351


# Automated Discord Spam Moderation with AI and Human-in-the-Loop

### 1. Workflow Overview

This workflow automates spam moderation in a Discord community by combining AI-powered text classification with human-in-the-loop moderation. Its purpose is to efficiently detect spam messages, notify moderators with actionable options, and execute moderation decisions while minimizing false positives and unnecessary moderator workload.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Message Retrieval:** Periodically fetch recent messages from a Discord channel, filtering out duplicates to avoid repeated processing.
- **1.2 User Grouping and Batch Processing:** Group messages by user and iterate over each user’s message batch to optimize notification frequency.
- **1.3 AI Spam Detection:** Use an AI text classifier (OpenAI-based) to determine if messages are spam.
- **1.4 Flagging and Filtering:** Label messages as spam or not, and filter only flagged messages for further action.
- **1.5 Human-in-the-Loop Moderation Notification:** Notify moderators with a detailed message and a dropdown action form, then wait for their input.
- **1.6 Moderation Actions Execution:** Based on moderator input, take actions such as deleting messages, warning users, or doing nothing, including sending direct messages to flagged users.
- **1.7 Workflow Concurrency Handling:** Use subworkflows to allow concurrent processing of multiple users without blocking.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Message Retrieval

- **Overview:**  
  This block triggers the workflow on a schedule (every hour) and fetches recent messages from a specified Discord channel. It uses a deduplication step to prevent reprocessing of already handled messages.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Recent Messages  
  - Only Once (Remove Duplicates)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger  
    - Runs every hour to start message fetching.  
    - No inputs, outputs to "Get Recent Messages".  
    - Edge cases: Missed executions if n8n server down; Interval may need adjustment based on message volume.

  - **Get Recent Messages**  
    - Type: Discord node (Get all messages)  
    - Fetches all recent messages from a configured Discord channel (guildId and channelId set).  
    - Credentials: Discord Bot API.  
    - Outputs raw message data, simplified for easier use.  
    - Possible failures: Discord API rate limits, invalid credentials, channel access issues.

  - **Only Once (Remove Duplicates)**  
    - Type: Remove Duplicates  
    - Deduplicates messages based on message ID to avoid reprocessing.  
    - Maintains a history size of 100 to track seen messages.  
    - Input: messages from "Get Recent Messages".  
    - Output: only new messages.  
    - Edge cases: If history is too small, duplicates may be processed again; if too large, memory use increases.

---

#### 1.2 User Grouping and Batch Processing

- **Overview:**  
  Groups messages by author to reduce the number of notifications sent to moderators and processes each user’s messages in batches to avoid blocking in human-in-the-loop steps.

- **Nodes Involved:**  
  - Group By User (Code)  
  - Split Out (Split Out grouped data)  
  - For Each User... (Split In Batches)  
  - Message to List (Code)

- **Node Details:**

  - **Group By User**  
    - Type: Code node (JavaScript)  
    - Groups input messages by the user ID (author.id).  
    - Outputs an object keyed by user IDs mapping to arrays of messages.  
    - Input: deduplicated messages.  
    - Output: grouped messages object.  
    - Edge cases: Empty input results in empty grouping.

  - **Split Out**  
    - Type: Split Out  
    - Splits the grouped user object into individual user message groups.  
    - Input: grouped messages object.  
    - Output: array of user message groups.

  - **For Each User...**  
    - Type: Split In Batches  
    - Iterates over each user's batch of messages sequentially.  
    - This node manages batch size and processing order.  
    - Input: split user message groups.  
    - Output: single user message batch per iteration.  
    - Edge cases: Large number of users may slow workflow; human-in-the-loop inside loop can cause blocking (mitigated by subworkflow).

  - **Message to List**  
    - Type: Code node  
    - Converts grouped user messages from object format to array for processing.  
    - Input: grouped messages object.  
    - Output: list of message arrays per user.  
    - Used in conjunction with batch processing.

---

#### 1.3 AI Spam Detection

- **Overview:**  
  Uses an AI text classifier node powered by OpenAI to classify each message as spam or not spam, based on predefined categories.

- **Nodes Involved:**  
  - Model (OpenAI Chat Model)  
  - Spam Detection (Text Classifier)

- **Node Details:**

  - **Model**  
    - Type: Langchain LM Chat OpenAI node  
    - Configured to use the "o3-mini" model.  
    - Credentials: OpenAI API.  
    - Input: passes message content for classification.  
    - Output: LLM response for classification.  
    - Edge cases: API rate limits, errors, or latency.

  - **Spam Detection**  
    - Type: Langchain Text Classifier  
    - Input text: message content.  
    - Categories: "is_spam" (promotion, sales pitch, spam) and "is_not_spam".  
    - Output: classification result used to flag messages.  
    - Edge cases: misclassification risk, ambiguous messages.

---

#### 1.4 Flagging and Filtering

- **Overview:**  
  Flags messages as spam or not spam based on classifier output, merges results, and filters only the spam messages to proceed with moderation.

- **Nodes Involved:**  
  - Flag as Spam (Set)  
  - Flag as Not Spam (Set)  
  - Merge  
  - Spam Messages Only (Filter)  
  - Has Flagged Messages? (If)

- **Node Details:**

  - **Flag as Spam**  
    - Type: Set node  
    - Sets field `is_spam=true` on messages classified as spam.  
    - Passes other fields along unchanged.

  - **Flag as Not Spam**  
    - Type: Set node  
    - Sets field `is_spam=false` on messages classified as not spam.  
    - Always outputs data to maintain flow.

  - **Merge**  
    - Type: Merge node  
    - Combines flagged spam and not spam messages into one stream for filtering.

  - **Spam Messages Only**  
    - Type: Filter node  
    - Filters messages where `is_spam` is true.  
    - Allows only spam messages to pass.

  - **Has Flagged Messages?**  
    - Type: If node  
    - Checks if any messages exist after filtering (i.e., flagged spam messages).  
    - If yes, proceed to moderation subworkflow; if no, continue looping.

---

#### 1.5 Human-in-the-Loop Moderation Notification

- **Overview:**  
  Sends a detailed notification to moderators on Discord using the "Send and Wait" mode, which pauses the workflow until a moderator selects a predefined action from a dropdown form.

- **Nodes Involved:**  
  - Moderation Subworkflow (Execute Workflow)  
  - When Executed by Another Workflow (Execute Workflow Trigger)  
  - Notify Moderators with Send & Wait (Discord SendAndWait)  
  - Receive Instructions (Switch)

- **Node Details:**

  - **Moderation Subworkflow**  
    - Type: Execute Workflow  
    - Calls the current workflow as a subworkflow (recursive call) with `waitForSubWorkflow` set to false for concurrency.  
    - Input: flagged messages.  
    - Output: none directly used.  
    - Edge cases: recursive call risks mitigated by concurrency flag.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Entry point for subworkflow executions triggered by the main workflow.  
    - Input: passes through flagged messages.  
    - Output: triggers notification node.

  - **Notify Moderators with Send & Wait**  
    - Type: Discord node (Send and Wait)  
    - Sends a formatted message listing flagged messages by user with timestamps.  
    - Includes a custom form dropdown with three options:  
      - "Delete Message and Warn User"  
      - "Do nothing and Warn User"  
      - "Do nothing"  
    - Waits for moderator response before continuing.  
    - Credentials: Discord Bot API.  
    - Edge cases: moderator delay, message formatting errors.

  - **Receive Instructions**  
    - Type: Switch node  
    - Routes based on moderator's selected action:  
      - Delete & Warn  
      - Warn User Only  
      - Do nothing  
    - Each output triggers subsequent action nodes.

---

#### 1.6 Moderation Actions Execution

- **Overview:**  
  Executes the moderator-selected action on flagged messages, including deleting messages, warning users via direct messages, or no action.

- **Nodes Involved:**  
  - Get Message IDs (Code)  
  - Delete Messages (Discord)  
  - Warn User (Discord)  
  - Warn User Only (Discord)  
  - No Action Taken (NoOp)

- **Node Details:**

  - **Get Message IDs**  
    - Type: Code node  
    - Extracts message IDs and channel IDs from flagged messages for deletion.  
    - Input: flagged messages from moderator response.

  - **Delete Messages**  
    - Type: Discord node  
    - Deletes each flagged message by ID in its respective channel.  
    - On completion, sends a warning DM to the user via "Warn User".  
    - Credentials: Discord Bot API.  
    - Edge cases: permission errors, message already deleted.

  - **Warn User**  
    - Type: Discord node  
    - Sends direct message to user warning about deleted spam message and consequences.  
    - Credentials: Discord Bot API.

  - **Warn User Only**  
    - Type: Discord node  
    - Sends direct message warning user flagged messages exist but no deletion performed.  
    - Credentials: Discord Bot API.

  - **No Action Taken**  
    - Type: NoOp node (no operation)  
    - Used when moderator chooses to do nothing.

---

#### 1.7 Workflow Concurrency Handling

- **Overview:**  
  Uses subworkflows to prevent blocking when waiting for moderator input in a loop over multiple users.

- **Nodes Involved:**  
  - Moderation Subworkflow (Execute Workflow)  
  - When Executed by Another Workflow (Execute Workflow Trigger)

- **Node Details:**

  - **Moderation Subworkflow**  
    - Configured to run asynchronously (without waiting for completion) to allow multiple user batches to be processed concurrently.

  - **When Executed by Another Workflow**  
    - Acts as the subworkflow entry point to continue processing flagged messages as independent executions.

- **Edge Cases:**  
  - Recursive calls may cause stack depth issues if not monitored.  
  - Concurrent executions require careful rate limit and resource management.

---

### 3. Summary Table

| Node Name                         | Node Type                            | Functional Role                                   | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                                 |
|----------------------------------|------------------------------------|-------------------------------------------------|--------------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger                   | Periodic trigger to start message fetching       | -                                    | Get Recent Messages                   | ## 1. Get Channel Messages [Read more about the scheduled Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/) |
| Get Recent Messages             | Discord                           | Fetch recent messages from designated Discord channel | Schedule Trigger                     | Only Once                            |                                                                                                             |
| Only Once                      | Remove Duplicates                 | Remove already processed messages                  | Get Recent Messages                   | Group By User                        |                                                                                                             |
| Group By User                  | Code                             | Group messages by user ID                           | Only Once                            | Split Out                           | ## 2. Group Messages By User [Learn more about the loop node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitinbatches/)         |
| Split Out                     | Split Out                       | Split grouped user messages into separate items    | Group By User                        | For Each User...                    |                                                                                                             |
| For Each User...              | Split In Batches                | Iterate over each user’s messages batch             | Split Out                           | Message to List, (empty batch output) |                                                                                                             |
| Message to List               | Code                           | Convert grouped messages object to array             | For Each User... (empty output)      | Spam Detection                     | ## 3. Spam Detection using AI-powered Text Classification [Learn more about the text classification node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.text-classifier) |
| Model                        | Langchain LM Chat OpenAI        | Provides language model for text classification     | Message to List                     | Spam Detection                     |                                                                                                             |
| Spam Detection               | Langchain Text Classifier       | Classify messages as spam or not spam                | Model                              | Flag as Spam, Flag as Not Spam      |                                                                                                             |
| Flag as Spam                 | Set                            | Flag messages classified as spam                      | Spam Detection                     | Merge                             |                                                                                                             |
| Flag as Not Spam             | Set                            | Flag messages classified as not spam                  | Spam Detection                     | Merge                             |                                                                                                             |
| Merge                       | Merge                          | Combine spam and non-spam flagged messages            | Flag as Spam, Flag as Not Spam      | Spam Messages Only                |                                                                                                             |
| Spam Messages Only           | Filter                         | Filter only messages flagged as spam                  | Merge                             | Has Flagged Messages?              |                                                                                                             |
| Has Flagged Messages?        | If                             | Determine if any spam messages exist                   | Spam Messages Only                 | Moderation Subworkflow, For Each User... |                                                                                                             |
| Moderation Subworkflow       | Execute Workflow               | Run moderation steps as subworkflow for concurrency   | Has Flagged Messages?              | For Each User...                   | ## 4. Concurrent Processing using Subworkflows [Learn more about Subworkflow Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow) |
| When Executed by Another Workflow | Execute Workflow Trigger       | Entry point for subworkflow triggered by main workflow | Moderation Subworkflow             | Notify Moderators with Send & Wait |                                                                                                             |
| Notify Moderators with Send & Wait | Discord (Send and Wait)          | Notify moderators with message and action form, then wait for response | When Executed by Another Workflow  | Receive Instructions              | ## 5. Moderation using Human-in-the-Loop [Read more about n8n's human-fallback functionality](https://docs.n8n.io/advanced-ai/examples/human-fallback/) |
| Receive Instructions         | Switch                         | Route workflow based on moderator's selected action  | Notify Moderators with Send & Wait | Get Message IDs, Warn User Only, No Action Taken |                                                                                                             |
| Get Message IDs              | Code                           | Extract message and channel IDs for deletion          | Receive Instructions (Delete & Warn) | Delete Messages                  |                                                                                                             |
| Delete Messages              | Discord                        | Delete flagged messages from Discord channels          | Get Message IDs                   | Warn User                        | ## 6. Execute Moderation Actions [Learn more about the Discord node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.discord/)           |
| Warn User                   | Discord                        | Send warning DM to user after message deletion         | Delete Messages                  | -                               |                                                                                                             |
| Warn User Only               | Discord                        | Send warning DM without deleting messages              | Receive Instructions             | -                               |                                                                                                             |
| No Action Taken             | NoOp                          | No action taken on flagged messages                      | Receive Instructions             | -                               |                                                                                                             |
| Sticky Notes (Various)      | Sticky Note                   | Provide documentation and explanations within workflow | -                                | -                               | See individual nodes for details and links                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set it to run every 1 hour (or adjust as per community activity).  
   - This node will initiate message fetching.

3. **Add a Discord node "Get Recent Messages":**  
   - Operation: Get All Messages  
   - Resource: Message  
   - Set Guild ID and Channel ID to the target Discord server and channel to moderate.  
   - Enable "Simplify" option.  
   - Provide Discord Bot API credentials with permission to read messages.

4. **Add a Remove Duplicates node "Only Once":**  
   - Operation: Remove items seen in previous executions  
   - Deduplicate based on message ID (use expression `{{$json.id}}`).  
   - Set history size to 100.

5. **Add a Code node "Group By User":**  
   - JavaScript code groups messages by `author.id`.  
   - Output a JSON object keyed by user ID with arrays of messages.

6. **Add a Split Out node "Split Out":**  
   - Split the grouped user messages object into individual user message groups.

7. **Add a Split In Batches node "For Each User...":**  
   - Iterate over each user message batch sequentially.

8. **Add a Code node "Message to List":**  
   - Convert grouped messages object to an array format for AI processing.

9. **Add a Langchain LM Chat OpenAI node "Model":**  
   - Select "o3-mini" model (or another suitable OpenAI model).  
   - Provide OpenAI API credentials.

10. **Add a Langchain Text Classifier node "Spam Detection":**  
    - Input text: set to `{{$json.content}}`.  
    - Define categories:  
      - "is_spam": promotional, sales pitch, or spam message description.  
      - "is_not_spam": not spam.  
    - Connect output from "Model".

11. **Add two Set nodes "Flag as Spam" and "Flag as Not Spam":**  
    - "Flag as Spam": set `is_spam` field to `true`.  
    - "Flag as Not Spam": set `is_spam` field to `false`.  
    - Connect spam classifier outputs accordingly.

12. **Add a Merge node "Merge":**  
    - Merge flagged spam and not spam messages into one stream.

13. **Add a Filter node "Spam Messages Only":**  
    - Filter messages where `is_spam` is `true`.

14. **Add an If node "Has Flagged Messages?":**  
    - Condition: check if any spam messages exist (not empty).

15. **Add an Execute Workflow node "Moderation Subworkflow":**  
    - Set to call the current workflow (or another workflow) with `waitForSubWorkflow` set to false to allow concurrency.  
    - Input: pass flagged messages.  
    - Connect the true branch of "Has Flagged Messages?".

16. **Add an Execute Workflow Trigger node "When Executed by Another Workflow":**  
    - Acts as subworkflow entry point.

17. **Add a Discord node "Notify Moderators with Send & Wait":**  
    - Operation: Send and Wait  
    - Send message to moderation channel with user and message details.  
    - Add a custom form with a dropdown field labeled "Action" with options:  
      - Delete Message and Warn User  
      - Do nothing and Warn User  
      - Do nothing  
    - Credentials: Discord Bot API.

18. **Add a Switch node "Receive Instructions":**  
    - Branch on moderator's selection from dropdown.  
    - Outputs:  
      - Delete & Warn  
      - Warn User Only  
      - Do nothing

19. **Add a Code node "Get Message IDs":**  
    - Extract message IDs and channel IDs from flagged messages for deletion.

20. **Add Discord node "Delete Messages":**  
    - Operation: Delete Message  
    - Use message ID and channel ID from previous step.  
    - Credentials: Discord Bot API.

21. **Add Discord nodes "Warn User" and "Warn User Only":**  
    - Send direct messages (DM) to flagged users warning about spam or pending violations.  
    - Customize message content accordingly.  
    - Credentials: Discord Bot API.

22. **Add a NoOp node "No Action Taken":**  
    - Connect to the "Do nothing" branch for clarity.

23. **Connect nodes as per the workflow logic:**  
    - Schedule Trigger → Get Recent Messages → Only Once → Group By User → Split Out → For Each User...  
    - For Each User... → Message to List → Model → Spam Detection → Flag as Spam / Flag as Not Spam → Merge → Spam Messages Only → Has Flagged Messages?  
    - True branch of Has Flagged Messages? → Moderation Subworkflow → When Executed by Another Workflow → Notify Moderators with Send & Wait → Receive Instructions → branch to moderation actions.

24. **Apply credentials:**  
    - Configure Discord Bot API with appropriate bot token and permissions (read messages, send messages, delete messages, send DMs).  
    - Configure OpenAI API credentials for AI nodes.

25. **Activate the workflow:**  
    - Ensure the workflow is enabled to run on schedule.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| This template balances automated spam detection with human moderation to reduce false positives and moderator workload.                           | Workflow overview and purpose                                                                                                          |
| For busy communities, adjust the schedule trigger interval and batch sizes to optimize performance.                                                | Workflow tuning advice                                                                                                                  |
| The human-in-the-loop step uses Discord’s "Send and Wait" functionality, allowing moderators to select consistent actions from a dropdown form.   | [Human-fallback docs](https://docs.n8n.io/advanced-ai/examples/human-fallback/)                                                        |
| Subworkflows are used to allow concurrent processing of multiple users, avoiding blocking caused by waiting for moderator responses in loops.     | [Subworkflow Trigger docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow)                          |
| Discord Bot API credentials require permissions to read messages, send messages, delete messages, and send direct messages to users.              | Discord bot setup                                                                                                                       |
| OpenAI credentials must be configured for the Langchain LM Chat and Text Classifier nodes to function.                                             | OpenAI API setup                                                                                                                        |
| The AI classifier uses a general spam definition which can be customized for your community’s needs by editing categories and prompts.             | Customization tip                                                                                                                       |
| Additional moderation actions can be added by expanding the switch node and corresponding execution nodes.                                        | Extensibility note                                                                                                                      |
| Supports multiple Discord channels by adding more "Get Recent Messages" nodes or modifying the existing one. Can be adapted to Slack or similar. | Flexibility note                                                                                                                        |
| Join the n8n community Discord or Forum for help and discussion on this workflow and similar automations.                                         | [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)                                                    |

---

This completes the comprehensive structured reference for the "Automated Discord Spam Moderation with AI and Human-in-the-Loop" n8n workflow.