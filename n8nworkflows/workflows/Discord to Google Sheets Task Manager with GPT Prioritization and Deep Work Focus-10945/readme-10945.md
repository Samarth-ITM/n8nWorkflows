Discord to Google Sheets Task Manager with GPT Prioritization and Deep Work Focus

https://n8nworkflows.xyz/workflows/discord-to-google-sheets-task-manager-with-gpt-prioritization-and-deep-work-focus-10945


# Discord to Google Sheets Task Manager with GPT Prioritization and Deep Work Focus

### 1. Workflow Overview

This workflow automates task management by syncing Discord messages from a designated task channel into a structured Google Sheets task tracker, leveraging AI (OpenAI GPT models) to prioritize and categorize tasks according to a detailed mission-aligned framework. It also handles task completion tracking and posts daily prioritized task digests back to Discord for focused execution.

The workflow is logically divided into these functional blocks:

- **1.1 Input Reception and Task Fetching**  
  Periodic trigger to fetch new Discord messages from the input task channel and prepare data for processing.

- **1.2 AI Task Processing and Categorization**  
  Uses GPT-4 mini with a detailed prompt to analyze each task message, assign priority scores, categories, and other metadata, then appends the processed task to Google Sheets.

- **1.3 Task Completion Sync and Cleanup**  
  Detects tasks marked complete on Discord (via reaction emojis), updates their status in Google Sheets, archives completed tasks to a separate sheet, and deletes them from the active task list.

- **1.4 Daily Priority List Generation and Posting**  
  Retrieves all active tasks, uses GPT-5 mini to select the top 6 tasks (3 high-energy, 3 low-energy) based on various AI-calculated scores, formats a Discord-friendly daily agenda, and posts it to a designated Discord channel.

- **1.5 Auxiliary Controls and Utilities**  
  Includes batching, rate limiting, wait nodes, and message splitting for API limits, plus nodes for setting IDs and reacting to Discord messages for confirmation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Task Fetching

**Overview:**  
Triggered hourly, this block fetches all messages from the configured Discord `tasks-to-do` channel, then iterates over each message to prepare them for AI processing.

**Nodes Involved:**  
- Schedule Trigger  
- Set discord IDs here  
- get data - tasks Channel  
- Loop Over Items1  
- if message is recorded already  
- clean data  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Runs every hour to initiate data fetch.  
  - Output: starts the chain by triggering the Discord fetch.

- **Set discord IDs here**  
  - Type: Set  
  - Stores server_id, input_channel_id, output_channel_id as objects for access in downstream nodes.  
  - Inputs: from Schedule Trigger  
  - Outputs: to "get data - tasks Channel"  
  - Edge cases: IDs must be correctly configured or Discord API calls will fail.

- **get data - tasks Channel**  
  - Type: Discord node (getAll messages)  
  - Retrieves all messages from the input channel using oAuth2 credentials.  
  - Input: server_id and input_channel_id from previous node.  
  - Output: list of Discord messages for processing.  
  - Failure modes: API rate limits, auth errors, channel ID misconfiguration.

- **Loop Over Items1**  
  - Type: SplitInBatches  
  - Processes messages one by one (or in batches) for sequential AI analysis.  
  - Input: messages array from Discord.  
  - Output: single message objects for downstream processing.

- **if message is recorded already**  
  - Type: If  
  - Checks if the message already has a reaction emoji "✍️" indicating prior processing.  
  - Logic: If reaction count >=1 and emoji is "✍️", skip AI processing; else continue.  
  - Input: single message from Loop Over Items1.

- **clean data**  
  - Type: Set  
  - Extracts and renames relevant fields from Discord message JSON for AI processing: content, message_id, timestamp, emoji, forwarded content if any.  
  - Input: message JSON from "if message is recorded already" false branch.  
  - Output: structured data for AI analysis.

---

#### 2.2 AI Task Processing and Categorization

**Overview:**  
Processes each task message content with a detailed GPT prompt (based on mission-aligned productivity frameworks) to analyze, prioritize, categorize, and append the structured task data to Google Sheets.

**Nodes Involved:**  
- ai task organizer  
- Append row in task sheet  
- react to confirm  
- Wait1  

**Node Details:**

- **ai task organizer**  
  - Type: Langchain Agent (OpenAI GPT-4 mini)  
  - Uses a complex system prompt embedding user mission, prioritization logic, and task metadata extraction instructions.  
  - Input: cleaned task content, message_id, timestamp, emoji from "clean data".  
  - Output: structured task data with fields for Google Sheets.  
  - Edge cases: prompt failures, API limits, malformed input.

- **Append row in task sheet**  
  - Type: Google Sheets Tool (append operation)  
  - Appends AI-processed task data as a new row to the "Tasks" sheet in the configured spreadsheet.  
  - Fields mapped exactly as per AI output.  
  - Input: AI output from "ai task organizer".  
  - Output: confirmation of append success.

- **react to confirm**  
  - Type: Discord react (emoji reaction)  
  - Adds "✍️" emoji reaction to the original Discord message signaling successful processing.  
  - Input: message_id from cleaned data.  
  - Output: triggers Wait1 node.

- **Wait1**  
  - Type: Wait  
  - Waits 1 second before continuing to next iteration, helping rate-limit API calls.  
  - Input: from react to confirm.  
  - Output: loops back to Loop Over Items1 for next task.

---

#### 2.3 Task Completion Sync and Cleanup

**Overview:**  
Synchronizes task completion by detecting messages in Discord reacted with ✅ emoji, updates the corresponding task status in Google Sheets, moves completed tasks to an archive sheet, and deletes them from the active tasks list.

**Nodes Involved:**  
- get checked ones  
- Filter for checked ones  
- Update row in sheet  
- get completed rows  
- move completed rows to completed sheet  
- get completed rows1  
- get only one row  
- if completed does not exist  
- delete completed rows  
- Limit  
- Limit1  
- Limit3  
- Sticky Note1  
- Sticky Note4  
- Sticky Note5  

**Node Details:**

- **get checked ones**  
  - Type: Discord getAll messages from input channel  
  - Fetches all messages from Discord input channel again to find those with ✅ reactions.  
  - Input: server_id and input_channel_id.  
  - Output: messages with reactions.

- **Filter for checked ones**  
  - Type: Filter  
  - Filters messages with a ✅ emoji reaction on either first or second reaction index.  
  - Input: messages from get checked ones.  
  - Output: only checked messages.

- **Update row in sheet**  
  - Type: Google Sheets update operation  
  - Updates matching rows in "Tasks" sheet setting Status to "Completed" and Priority score to 0.  
  - Matching on message_id.  
  - Input: filtered checked messages.  
  - Output: triggers Limit1.

- **get completed rows**  
  - Type: Google Sheets read operation  
  - Reads all rows with Status = "Completed" in "Tasks" sheet.  
  - Output: list of completed tasks.

- **move completed rows to completed sheet**  
  - Type: Google Sheets appendOrUpdate  
  - Moves completed tasks from "Tasks" sheet to "completed tasks" archive sheet.  
  - Matches by message_id; updates or appends accordingly.  
  - Input: completed tasks from get completed rows.  
  - Output: triggers Limit3.

- **get completed rows1**  
  - Type: Google Sheets read  
  - Again reads completed tasks for cleanup loop.  
  - Output: list for deletion.

- **get only one row**  
  - Type: Limit  
  - Limits to one completed row per iteration to delete.  
  - Input: completed rows list.

- **if completed does not exist**  
  - Type: If  
  - Checks if there are no completed rows left (Status field not existing).  
  - If none remain: wait then end cleanup.  
  - Else: continue deleting.

- **delete completed rows**  
  - Type: Google Sheets delete operation  
  - Deletes row in "Tasks" sheet corresponding to completed task (based on row_number).  
  - Input: from get only one row.  
  - Output: loops back to get completed rows1 to continue cleanup.

- **Limit, Limit1, Limit3**  
  - Type: Limit  
  - Control max concurrency and batch sizes to avoid API overload.

- **Sticky Notes (1,4,5)**  
  - Provide high-level descriptions for these cleanup steps.

---

#### 2.4 Daily Priority List Generation and Posting

**Overview:**  
Fetches all in-progress tasks, uses GPT-5 mini with a custom prompt to select and format the top 6 tasks (3 high-energy, 3 low-energy) for the day, then posts this agenda to the Discord output channel.

**Nodes Involved:**  
- Get tasks to do  
- Edit Fields  
- Aggregate  
- AI Agent (GPT-5 mini)  
- split if more than 2,000 chars (code)  
- Loop Over Items  
- Send a message  
- Sticky Note2  

**Node Details:**

- **Get tasks to do**  
  - Type: Google Sheets read  
  - Retrieves all tasks with Status = "In Progress".  
  - Input: Google Sheets credentials and document ID.  

- **Edit Fields**  
  - Type: Set  
  - Normalizes and prepares key fields (Task, Priority score, Priority Ranking, Status, Notes, Energy Level, Impact Score, message id) for AI input.  

- **Aggregate**  
  - Type: Aggregate  
  - Aggregates all items into a single data structure for AI input.  

- **AI Agent (GPT-5 mini)**  
  - Type: Langchain Agent with GPT-5 mini model  
  - Uses a detailed prompt to analyze tasks and return a Discord-formatted daily priority agenda with 3 high-energy and 3 low-energy tasks.  
  - Output: text message with task plan.  

- **split if more than 2,000 chars**  
  - Type: Code  
  - Splits the AI message if it exceeds Discord's character limit to avoid rejection.  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Sends each chunked message part sequentially.  

- **Send a message**  
  - Type: Discord send message  
  - Posts the daily priority agenda message(s) to the configured output Discord channel.  

- **Sticky Note2**  
  - Describes this block's logic for daily priority list generation and posting.

---

#### 2.5 Auxiliary Controls and Utilities

**Overview:**  
Includes wait nodes, limits, and set nodes used throughout the workflow to manage timing, rate limits, and configuration.

**Nodes Involved:**  
- Wait  
- Wait1  
- Limit  
- Limit1  
- Limit3  
- limit iteration  
- Sticky Note (general notes on input and saving to sheets)  

**Node Details:**

- **Wait / Wait1**  
  - Wait nodes add small delays between API calls to prevent rate limiting.  
- **Limit / Limit1 / Limit3 / limit iteration**  
  - Control batch sizes and concurrency for API stability and orderly processing.  
- **Sticky Notes (general)**  
  - Provide documentation and setup instructions embedded in workflow.

---

### 3. Summary Table

| Node Name                    | Node Type                      | Functional Role                                  | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                      |
|------------------------------|-------------------------------|-------------------------------------------------|--------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger               | Hourly trigger to start workflow                 | -                              | Set discord IDs here            |                                                                                                |
| Set discord IDs here         | Set                           | Store Discord server and channel IDs             | Schedule Trigger               | get data - tasks Channel        | ## 📥 Input: Fetch New Tasks and set discord IDs. Pulls unprocessed messages from Discord every hour. Set your IDs here. |
| get data - tasks Channel     | Discord                       | Fetch all messages from tasks-to-do channel      | Set discord IDs here           | Loop Over Items1                |                                                                                                |
| Loop Over Items1             | SplitInBatches                | Iterate over messages one by one                  | get data - tasks Channel       | if message is recorded already, Limit |                                                                                                |
| if message is recorded already | If                            | Check if message has already been processed       | Loop Over Items1               | clean data / Loop Over Items1   |                                                                                                |
| clean data                  | Set                           | Extract relevant fields for AI processing         | if message is recorded already | ai task organizer              | ## 🤖 AI Task Processor: Analyzes task text and assigns priority, impact, energy level, and category using AI prompts |
| ai task organizer           | Langchain Agent (GPT-4 mini)  | Process task with AI and generate structured data | clean data                    | react to confirm               |                                                                                                |
| Append row in task sheet    | Google Sheets Tool (Append)    | Append processed task to Google Sheets            | ai task organizer             | -                             | ## 💾 Save to Sheets: Appends processed task to Google Sheets and reacts to Discord message to confirm |
| react to confirm            | Discord React                 | React to original Discord message with "✍️"      | ai task organizer             | Wait1                         |                                                                                                |
| Wait1                       | Wait                         | Wait 1 second before next iteration                | react to confirm              | Loop Over Items1               |                                                                                                |
| limit iteration             | Limit                        | Limit concurrency for task fetching loop           | Wait                         | Get tasks to do                |                                                                                                |
| Get tasks to do             | Google Sheets                | Fetch all tasks with Status = "In Progress"        | limit iteration               | Edit Fields                   |                                                                                                |
| Edit Fields                 | Set                          | Normalize fields for priority AI input             | Get tasks to do               | Aggregate                     |                                                                                                |
| Aggregate                   | Aggregate                    | Aggregate all tasks into one JSON object            | Edit Fields                  | AI Agent                     |                                                                                                |
| AI Agent                   | Langchain Agent (GPT-5 mini)  | Generate prioritized daily task list                | Aggregate                    | split if more than 2,000 chars | ## 🔥 Daily Priority List: Gets all in-progress tasks, uses AI to pick top 6, posts to Discord   |
| split if more than 2,000 chars | Code                         | Split long messages to fit Discord limits          | AI Agent                     | Loop Over Items               |                                                                                                |
| Loop Over Items            | SplitInBatches                | Send each split message sequentially                | split if more than 2,000 chars | Send a message                |                                                                                                |
| Send a message             | Discord Send Message           | Post daily priority list to output Discord channel | Loop Over Items              | -                            |                                                                                                |
| get checked ones           | Discord GetAll Messages        | Fetch all messages from tasks-to-do channel         | Limit                       | Filter for checked ones       |                                                                                                |
| Filter for checked ones    | Filter                       | Filter messages with ✅ reaction                      | get checked ones             | Update row in sheet           |                                                                                                |
| Update row in sheet        | Google Sheets Update           | Mark tasks as completed in sheet                     | Filter for checked ones      | Limit1                       | ## ✅ Sync Completed Tasks: Finds Discord messages with ✅ reactions and marks them completed in sheets |
| Limit1                     | Limit                        | Limit concurrency for completed tasks update         | Update row in sheet          | get completed rows            |                                                                                                |
| get completed rows         | Google Sheets GetRows          | Retrieve all completed tasks                           | Limit1                      | move completed rows to completed sheet |                                                                                                |
| move completed rows to completed sheet | Google Sheets AppendOrUpdate | Archive completed tasks to separate sheet           | get completed rows           | Limit3                       | ## 📦 Archive Completed: Moves completed tasks to archive and deletes from active list          |
| Limit3                     | Limit                        | Limit concurrency for archive operation                | move completed rows to completed sheet | get completed rows1          |                                                                                                |
| get completed rows1        | Google Sheets GetRows          | Retrieve completed tasks for deletion loop            | Limit3                      | get only one row             |                                                                                                |
| get only one row           | Limit                        | Limit to one row for deletion at a time                | get completed rows1          | if completed does not exist   |                                                                                                |
| if completed does not exist | If                           | Check if any completed tasks remain                      | get only one row            | Wait / delete completed rows |                                                                                                |
| delete completed rows      | Google Sheets Delete           | Delete completed task rows from active sheet           | if completed does not exist  | get completed rows1          |                                                                                                |
| Limit                      | Limit                        | Controls concurrency in initial loops                    | Loop Over Items1             | get checked ones             |                                                                                                |
| Sticky Notes (multiple)    | Sticky Note                   | Documentation and instructions                          | -                           | -                           | See individual notes in sticky note content above                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Set to trigger every hour (interval: 1 hour).  
   - This triggers the entire workflow.

2. **Add a Set Node ("Set discord IDs here")**  
   - Create variables: `server_id`, `input_channel_id`, `output_channel_id` (object type).  
   - Assign your Discord server and channel IDs accordingly.

3. **Add a Discord Node ("get data - tasks Channel")**  
   - Operation: `getAll` messages from channel.  
   - Configure with `guildId` = `server_id`, `channelId` = `input_channel_id`.  
   - Use OAuth2 credentials for Discord.  
   - Connect output from "Set discord IDs here".

4. **Add a SplitInBatches Node ("Loop Over Items1")**  
   - Batch size: default (process messages one-by-one).  
   - Connect from "get data - tasks Channel".

5. **Add an If Node ("if message is recorded already")**  
   - Condition: check if message reactions contain "✍️" emoji with count >=1.  
   - True branch: skip processing, loop back or discard.  
   - False branch: continue.

6. **Add a Set Node ("clean data")**  
   - Extract fields: `content`, `message_id`, `timestamp`, `emoji` (from reactions), `content_if_forwarded`.  
   - Connect from False branch of If node.

7. **Add a Langchain Agent Node ("ai task organizer")**  
   - Model: GPT-4 mini.  
   - System prompt: detailed mission-aligned task processing instructions (see overview).  
   - Input: cleaned data fields.  
   - Output: structured task data matching Google Sheets columns.

8. **Add a Google Sheets Tool Node ("Append row in task sheet")**  
   - Operation: Append row.  
   - Document ID: your Google Sheets document.  
   - Sheet name: "Tasks" sheet.  
   - Map fields exactly as AI output (Task, Priority Ranking, Status, Due Date, Notes, Energy Level, etc.).  
   - Connect input from "ai task organizer" node.

9. **Add a Discord React Node ("react to confirm")**  
   - Operation: React to message.  
   - Emoji: "✍️" (pen).  
   - Use original message_id from cleaned data.  
   - OAuth2 credentials for Discord.  
   - Connect from "ai task organizer".

10. **Add a Wait Node ("Wait1")**  
    - Amount: 1 second.  
    - Connect from "react to confirm".  
    - Output back into "Loop Over Items1" to process next message.

---

11. **Add a Limit Node ("limit iteration")**  
    - Controls concurrency for downstream nodes.  
    - Connect from "Wait".

12. **Add a Google Sheets Node ("Get tasks to do")**  
    - Operation: Read rows.  
    - Filter: Status column equals "In Progress".  
    - Document: same as above.  
    - Sheet: "Tasks".  
    - Connect from "limit iteration".

13. **Add a Set Node ("Edit Fields")**  
    - Normalize fields for AI input: Task, Priority score, Priority Ranking, Status, Notes, Energy Level, Impact Score, message id.  
    - Connect from "Get tasks to do".

14. **Add an Aggregate Node ("Aggregate")**  
    - Aggregate all items into one JSON object with `data` array.  
    - Connect from "Edit Fields".

15. **Add a Langchain Agent Node ("AI Agent")**  
    - Model: GPT-5 mini.  
    - System prompt: detailed daily priority list generation (select 3 high-energy + 3 low-energy tasks).  
    - Input: aggregated task data.  
    - Output: formatted Discord message text.

16. **Add a Code Node ("split if more than 2,000 chars")**  
    - JavaScript to split long messages into < 2000 char chunks for Discord.  
    - Connect from "AI Agent".

17. **Add another SplitInBatches Node ("Loop Over Items")**  
    - Batch size: 1 (send messages one by one).  
    - Connect from "split if more than 2,000 chars".

18. **Add Discord Node ("Send a message")**  
    - Operation: send message.  
    - Channel: `output_channel_id`.  
    - OAuth2 credentials.  
    - Connect from "Loop Over Items".

---

19. **Add a Discord Node ("get checked ones")**  
    - Operation: getAll messages from `input_channel_id`.  
    - Connect from "Limit" node (part of cleanup cycle).

20. **Add a Filter Node ("Filter for checked ones")**  
    - Filter messages with "✅" emoji in reactions.  
    - Connect from "get checked ones".

21. **Add Google Sheets Update Node ("Update row in sheet")**  
    - Update rows where message_id matches, set Status = "Completed", Priority score = 0.  
    - Connect from filter node.

22. **Add Limit Node ("Limit1")**  
    - Control concurrency for updating rows.  
    - Connect from "Update row in sheet".

23. **Add Google Sheets Node ("get completed rows")**  
    - Read rows with Status = "Completed" from "Tasks" sheet.  
    - Connect from "Limit1".

24. **Add Google Sheets AppendOrUpdate Node ("move completed rows to completed sheet")**  
    - Append or update rows in "completed tasks" sheet.  
    - Connect from "get completed rows".

25. **Add Limit Node ("Limit3")**  
    - Control concurrency for archiving tasks.  
    - Connect from "move completed rows to completed sheet".

26. **Add Google Sheets Node ("get completed rows1")**  
    - Read completed tasks again for deletion loop.  
    - Connect from "Limit3".

27. **Add Limit Node ("get only one row")**  
    - Limit to one row per deletion cycle.  
    - Connect from "get completed rows1".

28. **Add If Node ("if completed does not exist")**  
    - Check if completed rows still exist.  
    - True branch: Wait node (1 second), then end loop.  
    - False branch: Delete row node.

29. **Add Google Sheets Delete Node ("delete completed rows")**  
    - Delete rows from "Tasks" sheet by row_number.  
    - Connect from False branch of If node.  
    - Loops back to "get completed rows1" for next row.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The workflow is based on a detailed AI prompt embedding mission-aligned productivity principles from Stephen Covey, Cal Newport, and James Clear. | Embedded in "ai task organizer" node prompt. |
| Google Sheets template recommended: https://docs.google.com/spreadsheets/d/151-Fcn6veIxqrlG_4jpaibkCtEDuQiT2vxbCj56CWgo/edit?usp=sharing | Setup instructions in Sticky Note "📋 AI-Powered Discord Task Manager" |
| Discord message reactions "✍️" mark processed tasks; "✅" mark completed tasks for syncing. | Workflow logic for task tracking. |
| Priority scoring uses a complex weighted formula combining urgency, priority ranking, impact score, energy level, and deadline type. | Detailed in AI prompts and field processing instructions. |
| Daily digest message limits Discord message size; splitting is handled by a code node to avoid posting errors. | Node: "split if more than 2,000 chars". |
| OAuth2 credentials required for Discord and Google Sheets integrations. | Credentials nodes referenced throughout. |
| The workflow respects API rate limits by using Wait and Limit nodes to pace requests. | Ensures stability and avoids throttling. |

---

This document fully describes the Discord to Google Sheets Task Manager workflow with AI prioritization and daily digest posting, enabling advanced users and automation agents to understand, recreate, and maintain it reliably.