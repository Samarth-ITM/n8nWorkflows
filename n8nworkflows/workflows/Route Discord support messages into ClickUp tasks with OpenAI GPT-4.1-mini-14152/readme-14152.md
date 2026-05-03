Route Discord support messages into ClickUp tasks with OpenAI GPT-4.1-mini

https://n8nworkflows.xyz/workflows/route-discord-support-messages-into-clickup-tasks-with-openai-gpt-4-1-mini-14152


# Route Discord support messages into ClickUp tasks with OpenAI GPT-4.1-mini

# 1. Workflow Overview

This workflow monitors a Discord support channel, identifies new messages, uses OpenAI GPT-4.1-mini to convert each message into structured support-task metadata, and then creates a corresponding ClickUp task.

Its main use case is lightweight support triage for agencies or technical teams that receive user issues in Discord and want to route them into a task management system automatically, with AI deciding assignee, priority, estimate, and timing.

The workflow is organized into the following logical blocks:

## 1.1 Trigger & Message Retrieval
A scheduled trigger runs every minute and fetches the latest Discord messages from a configured channel.

## 1.2 Per-Message Iteration
Messages are processed one at a time using a batching loop so downstream nodes execute in a controlled sequence.

## 1.3 Duplicate Detection
Each Discord message ID is checked against an n8n Data Table to avoid creating duplicate ClickUp tasks for already-processed messages.

## 1.4 Message Persistence
New messages are stored in the Data Table before AI processing, ensuring idempotency across workflow runs.

## 1.5 AI Task Classification
The Discord message text is sent to OpenAI with a structured system prompt that returns JSON describing the ClickUp task metadata.

## 1.6 Task Formatting
The AI response is parsed and reshaped into fields expected by the ClickUp node.

## 1.7 ClickUp Task Creation
A ClickUp task is created using the generated metadata and original message context, then the workflow loops back to continue processing remaining messages.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Message Retrieval

**Overview:**  
This block starts the workflow on a fixed interval and retrieves recent messages from a Discord support channel. It is the workflow’s only entry point.

**Nodes Involved:**  
- Schedule Trigger
- Get many messages

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based entry node that launches the workflow automatically.
- **Configuration choices:**  
  Configured to run every 1 minute.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Get many messages`
- **Version-specific requirements:**  
  Uses `typeVersion: 1.3`.
- **Edge cases or potential failure types:**  
  - Workflow may run too frequently for Discord/API quotas if the interval is reduced further.
  - Overlapping executions may happen if prior runs take longer than one minute and concurrency is not controlled at the workflow level.
- **Sub-workflow reference:**  
  None.

#### Get many messages
- **Type and technical role:** `n8n-nodes-base.discord`  
  Reads messages from a Discord channel.
- **Configuration choices:**  
  - Resource: `message`
  - Operation: `getAll`
  - Limit: `10`
  - `simplify` disabled, so the node returns fuller Discord payloads instead of reduced fields.
  - Guild ID and Channel ID must be selected/configured.
- **Key expressions or variables used:**  
  None directly, but downstream nodes rely on fields like:
  - `$json.id`
  - `$json.content`
  - `$json.timestamp`
  - `$json.author.username`
  - `$json.author.global_name`
  - `$json.channel_id`
- **Input and output connections:**  
  - Input: `Schedule Trigger`
  - Output: `Loop Over Items`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Discord bot authentication/permission errors
  - Missing guild/channel selection
  - Bot lacking access to read message history
  - Empty message list if the channel has no recent messages
  - Non-simplified payload shape must remain compatible with downstream expressions
- **Sub-workflow reference:**  
  None.

---

## 2.2 Per-Message Iteration

**Overview:**  
This block ensures each Discord message is processed one-by-one rather than in parallel. That controlled iteration is important because the workflow writes to a deduplication table and loops after task creation.

**Nodes Involved:**  
- Loop Over Items

### Node Details

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the fetched Discord messages sequentially.
- **Configuration choices:**  
  Default options are used; no explicit batch size override is shown.
- **Key expressions or variables used:**  
  Downstream nodes reference the current loop item using selectors such as:
  - `$('Loop Over Items').item.json.author.username`
  - `$('Loop Over Items').item.json.author.global_name`
  - `$('Loop Over Items').item.json.content`
- **Input and output connections:**  
  - Input: `Get many messages`
  - Output 1: unused
  - Output 2: `Get row(s)`
  - Receives a loop-back input from:
    - `If` false branch
    - `Create ClickUp Task`
- **Version-specific requirements:**  
  Uses `typeVersion: 3`.
- **Edge cases or potential failure types:**  
  - If no messages are returned, the loop does not process items.
  - Expressions that reference the loop context can fail if used outside valid iteration scope.
  - Misunderstanding split-in-batches routing can break the loop when modifying the workflow.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Duplicate Detection

**Overview:**  
This block checks whether the current Discord message has already been processed by looking up its `message_id` in an n8n Data Table. If a row exists, the workflow skips task creation for that message.

**Nodes Involved:**  
- Get row(s)
- If

### Node Details

#### Get row(s)
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Queries a Data Table for an existing row matching the Discord message ID.
- **Configuration choices:**  
  - Operation: `get`
  - Limit: `1`
  - Match type: `allConditions`
  - Filter condition:
    - `message_id = {{$json.id}}`
  - `alwaysOutputData` is enabled so the workflow continues even when no row is found.
- **Key expressions or variables used:**  
  - `={{$json.id}}`
- **Input and output connections:**  
  - Input: `Loop Over Items`
  - Output: `If`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Incorrect Data Table selected
  - Schema mismatch if `message_id` column is absent or renamed
  - Data type mismatch if message IDs are stored inconsistently
  - With `alwaysOutputData`, downstream logic depends on interpreting empty results correctly
- **Sub-workflow reference:**  
  None.

#### If
- **Type and technical role:** `n8n-nodes-base.if`  
  Determines whether the lookup result is empty, which the workflow interprets as “message not yet processed”.
- **Configuration choices:**  
  - Condition type: string
  - Operation: `empty`
  - Left value: `{{$json.message_id}}`
- **Key expressions or variables used:**  
  - `={{ $json.message_id }}`
- **Input and output connections:**  
  - Input: `Get row(s)`
  - True output: `Insert row`
  - False output: `Loop Over Items`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3` with condition options version 3.
- **Edge cases or potential failure types:**  
  - This logic assumes a missing row results in an empty `message_id`; if Data Table output behavior changes, duplicates may slip through or valid items may be skipped.
  - If a row exists but `message_id` is blank/corrupt, the workflow could incorrectly treat it as new.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Message Persistence

**Overview:**  
This block writes newly discovered Discord messages into the Data Table before any task creation occurs. It provides idempotency by ensuring future workflow runs can detect the message as already processed.

**Nodes Involved:**  
- Insert row

### Node Details

#### Insert row
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Inserts a new record into the selected Data Table.
- **Configuration choices:**  
  Columns mapped explicitly in “define below” mode:
  - `message_id` = current Discord message ID
  - `channel_id` = current Discord channel ID
  - `author` = current Discord author global name
  - `content` = current Discord message content
- **Key expressions or variables used:**  
  - `={{ $('Loop Over Items').item.json.author.global_name }}`
  - `={{ $('Loop Over Items').item.json.content }}`
  - `={{ $('Loop Over Items').item.json.channel_id }}`
  - `={{ $('Loop Over Items').item.json.id }}`
- **Input and output connections:**  
  - Input: `If` true branch
  - Output: `Message a model`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Insert failure if Data Table is missing or unavailable
  - Schema mismatch if required columns differ
  - `author.global_name` may be null for some Discord users; a fallback to username may be safer
  - If insertion succeeds but downstream AI/ClickUp fails, the message will still be marked as processed and may not retry automatically
- **Sub-workflow reference:**  
  None.

---

## 2.5 AI Task Classification

**Overview:**  
This block sends the Discord message content to OpenAI GPT-4.1-mini and asks for a strict one-line JSON object containing assignee, priority, estimate, title, and dates.

**Nodes Involved:**  
- Message a model

### Node Details

#### Message a model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Sends a chat-style request to OpenAI and returns the model response.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
  - Two-message conversation:
    - **System message:** defines routing responsibilities, expected JSON schema, date rules, priority meanings, estimate guidance, and strict output formatting constraints
    - **User message:** Discord message content from `{{$json["content"]}}`
- **Key expressions or variables used:**  
  - `={{$json["content"]}}`
  - Downstream expects output in:
    - `$json.output[0].content[0].text`
- **Input and output connections:**  
  - Input: `Insert row`
  - Output: `Format Task`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`.  
  Requires the LangChain/OpenAI node package available in the n8n instance and valid OpenAI credentials.
- **Edge cases or potential failure types:**  
  - OpenAI authentication or quota errors
  - Model returns malformed JSON despite prompt constraints
  - Response structure may differ across node versions or provider updates
  - Non-English, ambiguous, empty, or attachment-only Discord messages may reduce classification quality
  - Team member IDs hardcoded in the prompt must match real ClickUp assignee IDs
  - The prompt says “current time” for `start_date`, but the model itself does not know exact runtime unless inferred by the provider; this may produce approximate rather than deterministic timestamps
- **Sub-workflow reference:**  
  None.

---

## 2.6 Task Formatting

**Overview:**  
This block parses the AI JSON string and maps it into the fields used by ClickUp, while also building a more human-readable task name and description.

**Nodes Involved:**  
- Format Task

### Node Details

#### Format Task
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates normalized task fields from the model output.
- **Configuration choices:**  
  Produces these fields:
  - `Priority` as number
  - `Estimate` as number
  - `taskName` as string with `Support - ` prefix
  - `taskDescription` including username, raw model text, original timestamp, and stored Discord message ID
  - `assignee` as array-like string expression wrapping the AI assignee ID
  - `Start Date`
  - `Due Date`
- **Key expressions or variables used:**  
  Repeated parsing of the AI response:
  - `JSON.parse($json.output[0].content[0].text).priority`
  - `JSON.parse($json.output[0].content[0].text).estimate_hours`
  - `JSON.parse($json.output[0].content[0].text).task_title`
  - `JSON.parse($json.output[0].content[0].text).assignee`
  - `JSON.parse($json.output[0].content[0].text).start_date`
  - `JSON.parse($json.output[0].content[0].text).due_date`
  
  Additional loop/table references:
  - `$('Loop Over Items').item.json.author.username`
  - `$('Get many messages').item.json.timestamp`
  - `$('Insert row').item.json.message_id`
- **Input and output connections:**  
  - Input: `Message a model`
  - Output: `Create ClickUp Task`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Any malformed AI JSON causes `JSON.parse(...)` failures
  - Parsing the same text repeatedly is inefficient and increases breakage risk if output shape changes
  - `taskDescription` currently includes the full JSON string as “Message:” rather than the original Discord content; this appears unintentional
  - `$('Get many messages').item.json.timestamp` may not always be the safest reference inside a loop; using the loop item timestamp would be more explicit
  - `assignee` formatting should be validated against ClickUp’s expected type in the target n8n version
- **Sub-workflow reference:**  
  None.

---

## 2.7 ClickUp Task Creation

**Overview:**  
This block creates the final task in ClickUp using both the AI-generated metadata and contextual details from the Discord message. After creation, control returns to the loop to process the next message.

**Nodes Involved:**  
- Create ClickUp Task

### Node Details

#### Create ClickUp Task
- **Type and technical role:** `n8n-nodes-base.clickUp`  
  Creates a task in ClickUp.
- **Configuration choices:**  
  - Task name from `taskName`
  - `folderless: true`
  - Additional fields:
    - `status`: blank
    - `content`: original Discord message content
    - `dueDate`: converted from AI ISO date to Unix timestamp in milliseconds
    - `priority`: AI-generated priority
    - `assignees`: AI-generated assignee
    - `startDate`: converted from AI ISO date to Unix timestamp in milliseconds
    - `timeEstimate`: AI-generated estimate
  - Workspace/list/team selection is expected through node UI and credentials
- **Key expressions or variables used:**  
  - `={{$json["taskName"]}}`
  - `={{ $('Loop Over Items').item.json.content }}`
  - `={{ new Date(JSON.parse($json.output[0].content[0].text).due_date).getTime() }}`
  - `={{ $json.Priority }}`
  - `={{ $json.assignee }}`
  - `={{ new Date(JSON.parse($json.output[0].content[0].text).start_date).getTime() }}`
  - `={{ $json.Estimate }}`
- **Input and output connections:**  
  - Input: `Format Task`
  - Output: `Loop Over Items`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.  
  Requires valid ClickUp credentials and selection of the target location for task creation.
- **Edge cases or potential failure types:**  
  - ClickUp authentication or permission errors
  - Invalid assignee IDs
  - Invalid or non-parsable date strings
  - `timeEstimate` unit expectations should be confirmed; some ClickUp APIs use milliseconds rather than hours
  - Because this node reparses `$json.output[0].content[0].text`, it appears to depend on AI output fields still being available after the Set node; this may not hold depending on how Set is configured in the runtime. A safer design would use the formatted fields directly.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow every minute |  | Get many messages | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow every minute |  | Get many messages | ## Step 1 - Trigger & Fetch\nRuns on schedule and fetches recent Discord messages. |
| Get many messages | n8n-nodes-base.discord | Fetches recent messages from a Discord channel | Schedule Trigger | Loop Over Items | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| Get many messages | n8n-nodes-base.discord | Fetches recent messages from a Discord channel | Schedule Trigger | Loop Over Items | ## Step 1 - Trigger & Fetch\nRuns on schedule and fetches recent Discord messages. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Processes messages sequentially | Get many messages; If; Create ClickUp Task | Get row(s) | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| Loop Over Items | n8n-nodes-base.splitInBatches | Processes messages sequentially | Get many messages; If; Create ClickUp Task | Get row(s) | ## Step 2 - Process Messages\nLoops through messages one-by-one for controlled execution. |
| Get row(s) | n8n-nodes-base.dataTable | Checks whether a Discord message was already processed | Loop Over Items | If | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| Get row(s) | n8n-nodes-base.dataTable | Checks whether a Discord message was already processed | Loop Over Items | If | ## Step 3 - Check Duplicates\nLooks up message ID in table to avoid reprocessing. |
| If | n8n-nodes-base.if | Routes new messages forward and duplicates back to loop | Get row(s) | Insert row; Loop Over Items | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| If | n8n-nodes-base.if | Routes new messages forward and duplicates back to loop | Get row(s) | Insert row; Loop Over Items | ## Step 3 - Check Duplicates\nLooks up message ID in table to avoid reprocessing. |
| Insert row | n8n-nodes-base.dataTable | Stores newly seen Discord messages in the tracking table | If | Message a model | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| Insert row | n8n-nodes-base.dataTable | Stores newly seen Discord messages in the tracking table | If | Message a model | ## Step 4 - Store Messages\nSaves new messages to ensure idempotent workflow runs. |
| Message a model | @n8n/n8n-nodes-langchain.openAi | Classifies support messages into structured task metadata | Insert row | Format Task | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| Message a model | @n8n/n8n-nodes-langchain.openAi | Classifies support messages into structured task metadata | Insert row | Format Task | ## Step 5 - AI Classification\nGenerates assignee, priority, estimate, and title using AI. |
| Format Task | n8n-nodes-base.set | Parses AI output and prepares ClickUp fields | Message a model | Create ClickUp Task | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| Format Task | n8n-nodes-base.set | Parses AI output and prepares ClickUp fields | Message a model | Create ClickUp Task | ## Step 6 - Format Task\nTransforms AI output into ClickUp-compatible structure. |
| Create ClickUp Task | n8n-nodes-base.clickUp | Creates the final ClickUp task | Format Task | Loop Over Items | ## AI Discord → ClickUp Support Automation\n### How it works\nThis workflow automatically converts Discord support messages into structured ClickUp tasks using AI.\nIt runs on a schedule, fetches recent Discord messages, and processes them one by one. Each message is checked against a data table to prevent duplicates. New messages are stored and then sent to an AI model, which extracts structured task data like assignee, priority, estimate, and title.\nThe formatted output is then used to create a ClickUp task with all relevant context from the original message.\n### Setup steps\n1. Connect your Discord Bot credentials\n2. Select your Discord server and support channel\n3. Configure Data Table for message tracking\n4. Connect OpenAI credentials\n5. Set your ClickUp workspace, list, and team\n6. Test with a sample Discord message\n### Customization tips\n- Adjust AI prompt for team roles\n- Modify priority and SLA logic\n- Add filters for specific message types |
| Create ClickUp Task | n8n-nodes-base.clickUp | Creates the final ClickUp task | Format Task | Loop Over Items | ## Step 7 - Create Task\nCreates ClickUp task with message context and metadata. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation / setup notes |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for trigger and fetch block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for processing block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for duplicate-check block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for storage block |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation for AI classification block |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation for formatting block |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation for ClickUp creation block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Route Discord support messages into ClickUp tasks with OpenAI GPT-4.1-mini**.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Set the interval rule to run **every 1 minute**.

3. **Add a Discord node to fetch messages**
   - Node type: `Discord`
   - Connect it after `Schedule Trigger`.
   - Configure:
     - **Resource:** `Message`
     - **Operation:** `Get Many` / `Get All`
     - **Guild/Server:** select your Discord server
     - **Channel:** select your support channel
     - **Limit:** `10`
     - **Simplify:** `false`
   - Credentials:
     - Add Discord bot credentials.
     - Ensure the bot has access to read messages and message history in the selected channel.

4. **Add a Split In Batches node**
   - Node type: `Loop Over Items` / `Split In Batches`
   - Connect it after `Get many messages`.
   - Leave default options unless you want to tune loop behavior.
   - This node will process messages individually.

5. **Create a Data Table for deduplication**
   - In n8n Data Tables, create a table with at least these columns:
     - `message_id` — string
     - `channel_id` — string
     - `author` — string
     - `content` — string
   - This table is required for both lookup and insert operations.

6. **Add a Data Table lookup node**
   - Node type: `Data Table`
   - Name it `Get row(s)`.
   - Connect it from the looping output of `Loop Over Items`.
   - Configure:
     - **Operation:** `Get`
     - **Limit:** `1`
     - **Match type:** `All Conditions`
     - **Filter:** `message_id` equals `{{$json.id}}`
     - **Always Output Data:** enabled
   - Select the Data Table created in step 5.

7. **Add an If node**
   - Node type: `If`
   - Connect it after `Get row(s)`.
   - Configure a condition:
     - Check whether `{{$json.message_id}}` **is empty**
   - Interpretation:
     - **True** = message not found, so continue processing
     - **False** = duplicate, skip and continue the loop

8. **Connect the duplicate branch back to the loop**
   - Connect the **false** output of `If` back to `Loop Over Items`.
   - This causes already-processed messages to be skipped.

9. **Add a Data Table insert node**
   - Node type: `Data Table`
   - Name it `Insert row`.
   - Connect it to the **true** output of `If`.
   - Configure:
     - **Operation:** insert/add row
     - **Mapping mode:** define fields manually
     - Map:
       - `message_id` → `{{ $('Loop Over Items').item.json.id }}`
       - `channel_id` → `{{ $('Loop Over Items').item.json.channel_id }}`
       - `author` → `{{ $('Loop Over Items').item.json.author.global_name }}`
       - `content` → `{{ $('Loop Over Items').item.json.content }}`
   - Use the same Data Table as step 5.

10. **Add an OpenAI chat/model node**
    - Node type: `OpenAI` via LangChain, named `Message a model`
    - Connect it after `Insert row`.
    - Credentials:
      - Add OpenAI API credentials.
    - Configure model:
      - **Model:** `gpt-4.1-mini`
    - Add a **system message** with the task-routing instruction set:
      - Define team responsibilities and ClickUp assignee IDs
      - Require output as a strict JSON object with:
        - `assignee`
        - `priority`
        - `estimate_hours`
        - `task_title`
        - `start_date`
        - `due_date`
      - Require ISO 8601 dates
      - Require one-line minified JSON only
      - Forbid markdown and explanations
    - Add a **user message**:
      - `{{$json["content"]}}`

11. **Add a Set node to format the task**
    - Node type: `Set`
    - Name it `Format Task`
    - Connect it after `Message a model`.
    - Add these fields:

    **Number fields**
    - `Priority` → `{{ JSON.parse($json.output[0].content[0].text).priority }}`
    - `Estimate` → `{{ JSON.parse($json.output[0].content[0].text).estimate_hours }}`

    **String fields**
    - `taskName` → `Support - {{ JSON.parse($json.output[0].content[0].text).task_title }}`
    - `taskDescription` →  
      `User: {{ $('Loop Over Items').item.json.author.username }}`
      
      `Message:{{ $json.output[0].content[0].text }}`
      
      `Timestamp:`
      `{{ $('Get many messages').item.json.timestamp }}`
      
      `Discord Message ID:{{ $('Insert row').item.json.message_id }}`
    - `assignee` → `{{ [ JSON.parse($json.output[0].content[0].text).assignee ] }}`
    - `Start Date` → `{{ JSON.parse($json.output[0].content[0].text).start_date }}`
    - `Due Date` → `{{ JSON.parse($json.output[0].content[0].text).due_date }}`

12. **Add a ClickUp node**
    - Node type: `ClickUp`
    - Name it `Create ClickUp Task`
    - Connect it after `Format Task`.
    - Credentials:
      - Add ClickUp credentials.
    - Configure the destination workspace/list/team in the node UI.
    - Set:
      - **Task Name:** `{{$json["taskName"]}}`
      - **Folderless:** enabled
    - In additional fields, configure:
      - `content` → `{{ $('Loop Over Items').item.json.content }}`
      - `dueDate` → `{{ new Date(JSON.parse($json.output[0].content[0].text).due_date).getTime() }}`
      - `priority` → `{{ $json.Priority }}`
      - `assignees` → `{{ $json.assignee }}`
      - `startDate` → `{{ new Date(JSON.parse($json.output[0].content[0].text).start_date).getTime() }}`
      - `timeEstimate` → `{{ $json.Estimate }}`
      - `status` → leave blank unless you want a fixed initial status

13. **Loop back after task creation**
    - Connect `Create ClickUp Task` back to `Loop Over Items`.
    - This allows the next message to be processed.

14. **Optionally add sticky notes for canvas clarity**
    - Add notes corresponding to:
      - Trigger & Fetch
      - Process Messages
      - Check Duplicates
      - Store Messages
      - AI Classification
      - Format Task
      - Create Task

15. **Configure credentials carefully**
    - **Discord Bot**
      - Must have access to the target server/channel
      - Needs permission to read messages/history
    - **OpenAI**
      - API key with access to `gpt-4.1-mini`
    - **ClickUp**
      - Must be authorized for the target workspace/list
      - Assignee IDs returned by the model must correspond to valid users in ClickUp

16. **Test the workflow**
    - Post a new support-style Discord message in the configured channel.
    - Run the workflow manually.
    - Confirm:
      - The message is fetched
      - No existing Data Table row matches the message ID
      - The row is inserted
      - OpenAI returns valid JSON
      - A ClickUp task is created

17. **Validate idempotency**
    - Re-run the workflow without posting a new message.
    - Confirm the same Discord message is found in the Data Table and skipped.

18. **Recommended hardening improvements**
    - Replace repeated `JSON.parse(...)` calls with a single parse step in a Code or Set node.
    - Use original Discord content in `taskDescription` instead of the AI JSON string.
    - Add error handling for malformed model output.
    - Consider storing a processing status in the Data Table so failed ClickUp creation can be retried.
    - Confirm ClickUp `timeEstimate` unit expectations; convert hours if needed.
    - Replace `author.global_name` with a fallback if null, for example username.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflow and has only one entry point: `Schedule Trigger`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Discord → ClickUp Support Automation. Automatically converts Discord support messages into structured ClickUp tasks using AI. | Overall workflow purpose |
| Setup steps: Connect Discord Bot credentials, select Discord server/channel, configure Data Table, connect OpenAI credentials, set ClickUp workspace/list/team, test with a sample Discord message. | Operational setup guidance |
| Customization tips: Adjust AI prompt for team roles, modify priority and SLA logic, add filters for specific message types. | Suggested extensions |
| Step 1 - Trigger & Fetch: Runs on schedule and fetches recent Discord messages. | Trigger block |
| Step 2 - Process Messages: Loops through messages one-by-one for controlled execution. | Iteration block |
| Step 3 - Check Duplicates: Looks up message ID in table to avoid reprocessing. | Deduplication block |
| Step 4 - Store Messages: Saves new messages to ensure idempotent workflow runs. | Persistence block |
| Step 5 - AI Classification: Generates assignee, priority, estimate, and title using AI. | AI block |
| Step 6 - Format Task: Transforms AI output into ClickUp-compatible structure. | Formatting block |
| Step 7 - Create Task: Creates ClickUp task with message context and metadata. | ClickUp block |

## Additional implementation observations
- The workflow is functional but depends heavily on the exact response structure of the OpenAI node.
- `Format Task` and `Create ClickUp Task` both reparse the AI JSON; centralizing this parsing would make maintenance safer.
- The Data Table insert occurs before ClickUp task creation, which prevents duplicates but may also suppress retries if downstream nodes fail.
- The AI prompt includes hardcoded responsibility mapping and assignee IDs; this is effective but requires manual maintenance whenever team ownership changes.