Daily Google Tasks Briefing in Slack with Ollama-Powered Summaries

https://n8nworkflows.xyz/workflows/daily-google-tasks-briefing-in-slack-with-ollama-powered-summaries-8354


# Daily Google Tasks Briefing in Slack with Ollama-Powered Summaries

### 1. Workflow Overview

This workflow, titled **"Daily Google Tasks Briefing in Slack with Ollama-Powered Summaries"**, is designed to automate the delivery of a concise, actionable morning briefing of Google Tasks due on the current day, posted directly into a Slack channel. It targets users who want an efficient daily summary of their tasks without manual review, leveraging a local large language model (LLM) via Ollama to generate human-friendly text summaries.

**Use cases:**  
- Daily executive or personal task management briefing  
- Automated team updates on task progress  
- Reducing manual effort in summarizing task data

**Logical blocks:**

- **1.1 Scheduled Trigger**: Initiates the workflow every morning at 7:00 AM.
- **1.2 Google Tasks Retrieval**: Fetches all tasks from a selected Google Tasklist.
- **1.3 Filtering Tasks Due Today**: Filters tasks to keep only those due on the current day in a specified timezone.
- **1.4 Conditional Branching**: Determines if tasks exist for today and routes accordingly.
- **1.5 LLM Prompt Construction**: Builds a compact, Markdown-formatted prompt for the LLM summarization.
- **1.6 LLM Processing**: Sends the prompt to an Ollama-powered LangChain LLM to generate the briefing.
- **1.7 Output Cleanup**: Cleans LLM output by removing any internal `<think>` tags.
- **1.8 Slack Notification**: Posts the final briefing or a "no tasks" message to a configured Slack channel.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Scheduled Trigger

**Overview:**  
Starts the workflow daily at 7:00 AM to automate task briefing generation.

**Nodes Involved:**  
- Trigger at Morning  
- Sticky Note (Scheduled Trigger)

**Node Details:**

- **Trigger at Morning**  
  - Type: Cron Trigger  
  - Configuration: Fires daily at hour 7 (7:00 AM) local time.  
  - Input: None (trigger)  
  - Output: Starts the workflow chain.  
  - Failure types: Cron misconfiguration or system time errors may cause missed triggers.  
  - Version: n8n Cron node v1  
  - Sub-workflow: None

- **Sticky Note (Scheduled Trigger)**  
  - Type: Sticky Note  
  - Purpose: Documentation only, describing the scheduled trigger’s purpose.  
  - No inputs or outputs.

---

#### 1.2 Google Tasks Retrieval

**Overview:**  
Retrieves all tasks from a specific Google Tasklist using Google Tasks API credentials.

**Nodes Involved:**  
- Get many tasks  
- Sticky Note (Tasks Briefing)

**Node Details:**

- **Get many tasks**  
  - Type: Google Tasks node  
  - Operation: `getAll` retrieves all tasks from the specified Tasklist ID (`MTQzNzI1NzAyOTQxNTk3NzM5NDc6MDow`).  
  - Returns all tasks without pagination limit.  
  - Credentials: Requires valid Google Tasks OAuth2 API credentials with read or read/write scope.  
  - Input: Trigger from Cron node  
  - Output: Array of task objects (JSON)  
  - Failure types: OAuth token expiration, insufficient API permissions, quota limits, or invalid tasklist ID.  
  - Version: Google Tasks node v1  
  - Sub-workflow: None

- **Sticky Note (Tasks Briefing)**  
  - Contains comprehensive documentation about the workflow purpose and setup instructions.  
  - No inputs or outputs.

---

#### 1.3 Filtering Tasks Due Today

**Overview:**  
Filters the retrieved tasks to include only those due today, using timezone-aware date comparison for Asia/Dhaka timezone.

**Nodes Involved:**  
- Code (Filter Due Today)  
- If  
- Slack1 (No tasks notification)  
- Sticky Note1 (Filter Tasks)

**Node Details:**

- **Code (Filter Due Today)**  
  - Type: Code node (JavaScript)  
  - Function:  
    - Converts current date and task due dates into Asia/Dhaka timezone.  
    - Filters out tasks with statuses `completed`, `hidden`, or `deleted`.  
    - Emits individual items for tasks due today.  
    - If none found, emits a single item `{ __empty: "empty" }`.  
  - Inputs: Array of all tasks from Google Tasks node  
  - Outputs: Array of tasks due today or a single empty flag object  
  - Failure types: Date parsing errors, timezone misconfiguration, empty task arrays.  
  - Version: Code node v2

- **If**  
  - Type: If node  
  - Condition: Checks if the input JSON does not equal `{ __empty: "empty" }`.  
  - Routes:  
    - True: There are tasks due today → continue to LLM processing  
    - False: No tasks due today → send notification to Slack1 node  
  - Inputs: Output from Code (Filter Due Today)  
  - Outputs: Two branches (true/false)  
  - Failure types: Expression evaluation errors or unexpected input structure  
  - Version: If node v2

- **Slack1**  
  - Type: Slack node  
  - Purpose: Sends a message saying "🌤️ No Google Tasks due today. I’ll check again tomorrow" when no tasks are due.  
  - Configuration:  
    - Channel: Must be replaced with actual Slack channel name.  
    - Text: Static message.  
    - Credentials: Slack API with proper scopes (`chat:write`).  
  - Input: False branch from If node  
  - Output: None  
  - Failure types: Slack API errors, invalid channel, token expiration  
  - Version: Slack node v2

- **Sticky Note1 (Filter Tasks)**  
  - Documents filtering logic that routes tasks due today to LLM or notifies absence.  
  - No inputs or outputs.

---

#### 1.4 LLM Prompt Construction

**Overview:**  
Transforms the filtered tasks into a concise, Markdown-formatted prompt suitable for the LLM to generate a Slack-ready briefing.

**Nodes Involved:**  
- Code (Build LLM Prompt)  
- Sticky Note2 (Process Tasks Briefing)

**Node Details:**

- **Code (Build LLM Prompt)**  
  - Type: Code node (JavaScript)  
  - Function:  
    - Maps each task to a simplified object containing title, notes, due/updated dates, links, and parent info.  
    - Constructs instructions for the LLM to produce a Slack-ready summary with specific formatting rules.  
    - Embeds the JSON data as context for the LLM prompt.  
    - Output is a single JSON object with property `input` containing the full prompt string.  
  - Inputs: Tasks due today from If node (true branch)  
  - Outputs: Single prompt item for LLM node  
  - Failure types: JSON serialization issues, missing task fields, malformed input data  
  - Version: Code node v2

- **Sticky Note2 (Process Tasks Briefing)**  
  - Describes building the briefing and cleaning output from internal tags.  
  - No inputs or outputs.

---

#### 1.5 LLM Processing

**Overview:**  
Uses LangChain and the Ollama local LLM model to generate a concise text briefing based on the prompt.

**Nodes Involved:**  
- Ollama Model1  
- Basic LLM Chain1

**Node Details:**

- **Ollama Model1**  
  - Type: LangChain Ollama LLM node  
  - Configuration:  
    - Model: `qwen3:4b` (a 4-billion parameter Qwen3 model)  
    - Credentials: Ollama API credentials pointing to local Ollama instance  
  - Input: Receives prompt from Basic LLM Chain1 (ai_languageModel input)  
  - Output: LLM-generated text response  
  - Failure types: Ollama not running, model not available, connection refused, timeout  
  - Version: LangChain Ollama node v1

- **Basic LLM Chain1**  
  - Type: LangChain LLM Chain node  
  - Configuration:  
    - Uses the prompt text from Code (Build LLM Prompt) node as `text` input  
    - No batching configured  
    - Prompt type set to "define" to handle the custom prompt format  
  - Input: Receives prompt JSON from Code (Build LLM Prompt)  
  - Output: Sends prompt text to Ollama Model1, receives generated text  
  - Failure types: LangChain errors, prompt misconfiguration, input data errors  
  - Version: LangChain Chain LLM node v1.7

---

#### 1.6 Output Cleanup

**Overview:**  
Sanitizes the LLM output by removing any internal `<think>...</think>` tags, which may be used by the LLM to represent internal reasoning not meant for display.

**Nodes Involved:**  
- Code (Cleanup)  
- Sticky Note2 (also documents this step)

**Node Details:**

- **Code (Cleanup)**  
  - Type: Code node (JavaScript)  
  - Function:  
    - Extracts the `text` field from LLM output JSON.  
    - Uses a regex to remove all occurrences of `<think>...</think>` blocks (case-insensitive, multiline).  
    - Trims the cleaned text and returns it in `cleanText` property.  
  - Input: Output from Basic LLM Chain1 (LLM text)  
  - Output: Cleaned text ready for Slack message  
  - Failure types: Missing or malformed text field, regex errors  
  - Version: Code node v2

---

#### 1.7 Slack Notification

**Overview:**  
Posts the cleaned task briefing message into the configured Slack channel.

**Nodes Involved:**  
- Send a message1  
- Sticky Note3 (Notify in Slack)

**Node Details:**

- **Send a message1**  
  - Type: Slack node  
  - Configuration:  
    - Text: Uses the cleaned text from Code (Cleanup) node (`{{ $json.cleanText }}`).  
    - Channel: Must be replaced with the actual Slack channel name (e.g., `#general`).  
    - Credentials: Slack API with `chat:write` permission.  
  - Input: Cleaned LLM output from Code (Cleanup) node  
  - Output: None (sends message externally)  
  - Failure types: Slack API errors, invalid channel or token, network issues  
  - Version: Slack node v2.3

- **Sticky Note3 (Notify in Slack)**  
  - Documents the purpose of sending the briefing message to Slack.  
  - No inputs or outputs.

---

### 3. Summary Table

| Node Name              | Node Type                        | Functional Role                            | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                                             |
|------------------------|---------------------------------|-------------------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Trigger at Morning      | Cron Trigger                    | Scheduled daily trigger at 7:00 AM        | None                        | Get many tasks               | ## Scheduled Trigger: Every Morning at 7 AM Fetch all the tasks                                                                        |
| Get many tasks          | Google Tasks                    | Retrieve all tasks from specified tasklist| Trigger at Morning           | Code (Filter Due Today)       | ## Tasks Briefing (shared with other nodes)                                                                                            |
| Code (Filter Due Today) | Code (JavaScript)               | Filter tasks due today in Asia/Dhaka timezone| Get many tasks             | If                           | ## Filter Tasks: Send tasks due today to LLM or notify user that there are no tasks scheduled today                                      |
| If                     | If                             | Branch: tasks exist or not                 | Code (Filter Due Today)      | Code (Build LLM Prompt) / Slack1| ## Filter Tasks (see above)                                                                                                           |
| Slack1                 | Slack                          | Notify no tasks due today                  | If (false branch)            | None                         | ## Filter Tasks (see above)                                                                                                           |
| Code (Build LLM Prompt)| Code (JavaScript)               | Build concise Markdown prompt for LLM     | If (true branch)             | Basic LLM Chain1              | ## Process Tasks Briefing: Produce a solid briefing for the user with LLM and clean the output removing the Think tags                  |
| Basic LLM Chain1        | LangChain LLM Chain            | Send prompt to Ollama LLM and get response| Code (Build LLM Prompt)      | Code (Cleanup)                | ## Process Tasks Briefing (see above)                                                                                                  |
| Ollama Model1           | LangChain Ollama LLM           | Local LLM model for text generation        | Basic LLM Chain1 (ai_languageModel) | Basic LLM Chain1 (LLM output) | ## Process Tasks Briefing (see above)                                                                                                  |
| Code (Cleanup)          | Code (JavaScript)               | Remove <think> tags from LLM output       | Basic LLM Chain1             | Send a message1              | ## Process Tasks Briefing (see above)                                                                                                  |
| Send a message1         | Slack                          | Post final briefing message to Slack      | Code (Cleanup)               | None                         | ## Notify in Slack: Send the task briefing as a slack message                                                                           |
| Sticky Note             | Sticky Note                    | Documentation                             | None                        | None                         | ## Scheduled Trigger (see above)                                                                                                       |
| Sticky Note1            | Sticky Note                    | Documentation                             | None                        | None                         | ## Filter Tasks (see above)                                                                                                            |
| Sticky Note2            | Sticky Note                    | Documentation                             | None                        | None                         | ## Process Tasks Briefing (see above)                                                                                                  |
| Sticky Note3            | Sticky Note                    | Documentation                             | None                        | None                         | ## Notify in Slack (see above)                                                                                                         |
| Sticky Note7            | Sticky Note                    | Full workflow documentation & instructions| None                       | None                         | ## Tasks Briefing (full description, setup, credentials, and instructions)                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron node: "Trigger at Morning"**
   - Type: Cron Trigger  
   - Set trigger time: Daily at 7:00 AM (adjust hour as needed).  
   - No credentials needed.  

2. **Create a Google Tasks node: "Get many tasks"**
   - Type: Google Tasks  
   - Operation: `getAll`  
   - Tasklist ID: Set your Google Tasklist ID (example: `MTQzNzI1NzAyOTQxNTk3NzM5NDc6MDow`).  
   - Credentials: Set up Google Tasks OAuth2 API credentials with scope `https://www.googleapis.com/auth/tasks.readonly` (or read/write).  
   - Connect Trigger at Morning → Get many tasks

3. **Create a Code node: "Code (Filter Due Today)"**
   - Type: Code (JavaScript)  
   - Paste the provided script to filter tasks due today in timezone `Asia/Dhaka`.  
   - Adjust `TZ` constant if needed for your timezone.  
   - Connect Get many tasks → Code (Filter Due Today)

4. **Create an If node: "If"**
   - Type: If  
   - Condition: Check if `$json.__empty` is NOT equal to `"empty"`.  
   - Connect Code (Filter Due Today) → If

5. **Create a Slack node: "Slack1" (No tasks notification)**
   - Type: Slack  
   - Text: "🌤️ No Google Tasks due today. I’ll check again tomorrow"  
   - Channel: Replace with your Slack channel name (e.g., `#general`).  
   - Credentials: Slack API with `chat:write` scope.  
   - Connect If (false branch) → Slack1

6. **Create a Code node: "Code (Build LLM Prompt)"**
   - Type: Code (JavaScript)  
   - Paste the provided script to build a compact Markdown prompt for LLM from filtered tasks.  
   - Connect If (true branch) → Code (Build LLM Prompt)

7. **Create a LangChain LLM Chain node: "Basic LLM Chain1"**
   - Type: LangChain Chain LLM  
   - Prompt input: `{{ $json.input }}` from Code (Build LLM Prompt)  
   - Prompt type: `define`  
   - No batching  
   - Connect Code (Build LLM Prompt) → Basic LLM Chain1

8. **Create a LangChain Ollama node: "Ollama Model1"**
   - Type: LangChain Ollama LLM  
   - Model: `qwen3:4b` (or your preferred Ollama model)  
   - Credentials: Ollama API credentials pointing to your local Ollama instance (default `http://localhost:11434`)  
   - Connect Ollama Model1 (ai_languageModel output) → Basic LLM Chain1 (ai_languageModel input)  
   - Connect Basic LLM Chain1 → Ollama Model1 (loop for model call)

9. **Create a Code node: "Code (Cleanup)"**
   - Type: Code (JavaScript)  
   - Paste script to remove `<think>...</think>` blocks from LLM output text.  
   - Input: Output from Basic LLM Chain1 (LLM text)  
   - Connect Basic LLM Chain1 → Code (Cleanup)

10. **Create a Slack node: "Send a message1"**
    - Type: Slack  
    - Text: `{{ $json.cleanText }}` from Code (Cleanup)  
    - Channel: Replace with your Slack channel name  
    - Credentials: Slack API with `chat:write` scope  
    - Connect Code (Cleanup) → Send a message1

11. **Verify all connections and credentials:**
    - Google Tasks OAuth2 API credentials configured with correct scopes and valid token.  
    - Slack API credentials set with proper permissions.  
    - Ollama local server running with the specified model loaded.  
    - Replace placeholder Slack channel names with actual channel IDs or names.

12. **Test workflow manually:**
    - Run workflow manually to confirm retrieval, filtering, summarization, cleanup, and posting work as expected.  
    - Adjust timezone or schedule settings if necessary.

13. **Activate workflow:**
    - Once tested and confirmed, activate the workflow to run daily at scheduled time.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                               | Context or Link                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| This workflow posts a clean, Slack-ready morning summary of Google Tasks due today using a local LLM (Ollama) integrated via LangChain.                                                                                                                                                                    | Workflow purpose                                                    |
| Setup requires enabling Google Tasks API in Google Cloud Console and creating OAuth2 credentials with appropriate scopes.                                                                                                                                                                                  | Google Tasks API setup instructions                                |
| Slack API credentials require bot or user token with `chat:write` scope to send messages to channels.                                                                                                                                                                                                       | Slack API setup instructions                                       |
| Ollama must be installed and running locally (`http://localhost:11434` by default), and the desired model (e.g., `qwen3:4b`) must be pulled and available.                                                                                                                                                | Ollama setup and model pull instructions                           |
| The timezone constant in the filtering code (`Asia/Dhaka`) should be adjusted to your local timezone for accurate daily task filtering.                                                                                                                                                                    | Timezone adjustment note                                           |
| The workflow includes safety cleanup removing `<think>...</think>` tags from LLM output to avoid internal reasoning snippets leaking to Slack.                                                                                                                                                            | Output cleaning best practice                                      |
| Slack channel names must be replaced in the Slack nodes to target your actual communication channel.                                                                                                                                                                                                       | Slack channel configuration                                        |
| For further reading and detailed setup, refer to external blog: [https://n8n.io/blog/daily-google-tasks-briefing-with-ollama](https://n8n.io/blog/daily-google-tasks-briefing-with-ollama)                                                                                                                      | External blog reference                                            |

---

This completes the full structured reference document for the **"Daily Google Tasks Briefing in Slack with Ollama-Powered Summaries"** n8n workflow.