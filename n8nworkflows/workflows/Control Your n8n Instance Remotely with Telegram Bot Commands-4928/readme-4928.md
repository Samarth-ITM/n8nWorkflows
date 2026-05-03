Control Your n8n Instance Remotely with Telegram Bot Commands

https://n8nworkflows.xyz/workflows/control-your-n8n-instance-remotely-with-telegram-bot-commands-4928


# Control Your n8n Instance Remotely with Telegram Bot Commands

### 1. Workflow Overview

This workflow enables remote management and control of an n8n instance through Telegram bot commands. It is designed for administrators who want to operate their n8n workflows remotely using simple text commands sent via Telegram.

**Target Use Cases:**
- Remote triggering and execution of workflows.
- Activating or deactivating workflows remotely.
- Listing workflows and their executions.
- Performing backups and cleanup operations.
- Receiving notifications on workflow errors and instance startup.

**Logical Blocks:**

- **1.1 Input Reception:** Receiving and parsing Telegram messages.
- **1.2 Command Routing:** Routing parsed commands to corresponding action blocks.
- **1.3 Workflow Listing & Execution:** Listing workflows, finding specific workflows, and executing them.
- **1.4 Workflow Activation/Deactivation:** Activating or deactivating workflows based on user commands.
- **1.5 Execution History Retrieval:** Listing recent executions of workflows.
- **1.6 Cleanup of Archived Workflows:** Deleting archived workflows permanently.
- **1.7 Backup Creation and Delivery:** Creating backups of workflows and credentials, compressing and sending them via Telegram.
- **1.8 Notifications and Error Handling:** Sending notifications for workflow errors and n8n instance startup.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block listens for incoming Telegram messages from a specific chat and user, then parses the command and its argument.
- **Nodes Involved:** Telegram Trigger, Cmd Parse

- **Node Details:**

  - **Telegram Trigger**
    - Type: `telegramTrigger` (Telegram integration)
    - Role: Listens for message updates from Telegram bot restricted to a specific chat ID and user ID.
    - Configuration:
      - Listens to "message" updates only.
      - Restricted to chat ID "123456789" and user ID "123456789" for security.
      - Uses credentials for Telegram Bot API.
    - Outputs: Raw Telegram message JSON.
    - Edge Cases: Telegram API downtime, invalid credentials, incorrect chat/user ID restrictions.

  - **Cmd Parse**
    - Type: `set`
    - Role: Parses Telegram message text to extract the command and first argument.
    - Configuration:
      - Extracts command by removing leading "/" and splitting by first space.
      - Command is lowercased.
      - `command` stores the first word.
      - `arg1` stores the rest of the message after the command.
    - Inputs: Message JSON from Telegram Trigger.
    - Outputs: JSON with `command` and `arg1` properties.
    - Edge Cases: Empty or malformed messages, messages without arguments.

#### 1.2 Command Routing

- **Overview:** Routes the parsed commands to the corresponding functional blocks using a Switch node.
- **Nodes Involved:** Cmd Switch

- **Node Details:**

  - **Cmd Switch**
    - Type: `switch`
    - Role: Matches the parsed command string to one of the predefined commands: start, help, backup, cleanup, workflows, execute, activate, deactivate, executions.
    - Configuration:
      - Case-insensitive exact match on `command` field from Cmd Parse.
      - Fallback output named "Error" for unrecognized commands.
    - Inputs: Parsed command JSON.
    - Outputs: Separate output for each command route.
    - Edge Cases: Unrecognized commands routed to error handler.

#### 1.3 Workflow Listing & Execution

- **Overview:** Handles listing workflows and executing a specified workflow by name.
- **Nodes Involved:** Execute Arg, List Workflows 1, Find Workflow 1, Workflow Found 1, Execute Workflow, Executed, Execution Error, Arg Error, Workflow Name Error

- **Node Details:**

  - **Execute Arg**
    - Type: `if`
    - Role: Checks if the argument (`arg1`) exists for commands that require workflow name.
    - Configuration: Condition tests if `arg1` is not empty.
    - Outputs: True (has argument), False (no argument).
    - Edge Cases: Missing argument triggers Arg Error.

  - **List Workflows 1**
    - Type: `n8n`
    - Role: Fetches all workflows from the n8n instance.
    - Credentials: n8n API credential.
    - Edge Cases: API errors, permission issues.

  - **Find Workflow 1**
    - Type: `filter`
    - Role: Filters workflows by name matching lowercased `arg1` and excludes archived workflows.
    - Edge Cases: Workflow not found triggers Workflow Name Error.

  - **Workflow Found 1**
    - Type: `if`
    - Role: Checks if any workflow matched the filter.
    - Outputs:
      - True: Workflow found, proceed to execute.
      - False: Workflow not found, send error.
  
  - **Execute Workflow**
    - Type: `executeWorkflow`
    - Role: Executes the identified workflow.
    - Inputs: Workflow ID from filter.
    - On error: Continues to Execution Error node.
    - Edge Cases: Execution failure, missing "When Executed by Another Workflow" trigger.

  - **Executed**
    - Type: `telegram`
    - Role: Sends success message upon workflow execution.
  
  - **Execution Error**
    - Type: `telegram`
    - Role: Sends failure notification if execution fails.

  - **Arg Error**
    - Type: `telegram`
    - Role: Sends error if argument missing when required.

  - **Workflow Name Error**
    - Type: `telegram`
    - Role: Sends error if workflow by name is not found.

#### 1.4 Workflow Activation/Deactivation

- **Overview:** Activates or deactivates workflows based on commands with workflow name arguments.
- **Nodes Involved:** Activate Arg, List Workflows 2, Find Workflow 2, Workflow Found 2, If Inactive, Activate Workflow, Activated, Activation Error, Workflow Active Error, Deactivate Arg, List Workflows 3, Find Workflow 3, Workflow Found 3, If Active, Deactivate Workflow, Deactivated, Deactivation Error, Workflow Inactive Error, Arg Error, Workflow Name Error

- **Node Details:**

  - **Activate Arg, Deactivate Arg**
    - Type: `if`
    - Role: Checks for presence of `arg1`.
    - Edge Cases: Missing argument triggers Arg Error.

  - **List Workflows 2, 3**
    - Type: `n8n`
    - Role: Fetch all workflows.

  - **Find Workflow 2, 3**
    - Type: `filter`
    - Role: Find workflow by name (lowercase) and exclude archived workflows.

  - **Workflow Found 2, 3**
    - Type: `if`
    - Role: Check if workflow exists.

  - **If Inactive / If Active**
    - Type: `if`
    - Role: Checks current active status of workflow to avoid unnecessary activation/deactivation.
    - Edge Cases:
      - Trying to activate an already active workflow triggers Workflow Active Error.
      - Trying to deactivate an already inactive workflow triggers Workflow Inactive Error.

  - **Activate Workflow / Deactivate Workflow**
    - Type: `n8n`
    - Role: Activates or deactivates workflow by ID.
    - On error: Continues to Activation Error or Deactivation Error nodes.

  - **Activated / Deactivated**
    - Type: `telegram`
    - Role: Sends success messages for activation/deactivation.

  - **Activation Error / Deactivation Error**
    - Type: `telegram`
    - Role: Sends error messages on failure.

  - **Arg Error / Workflow Name Error**
    - Role: Sends errors for missing arguments or unknown workflows.

#### 1.5 Execution History Retrieval

- **Overview:** Lists recent executions of a specified workflow.
- **Nodes Involved:** Executions Arg, List Workflows 4, Find Workflow 4, Workflow Found 4, List Workflow Executions, Executions Fields, Executions Message, Executions, Arg Error, Workflow Name Error

- **Node Details:**

  - **Executions Arg**
    - Type: `if`
    - Role: Checks if argument exists for workflow name.

  - **List Workflows 4**
    - Type: `n8n`
    - Role: Retrieves all workflows.

  - **Find Workflow 4**
    - Type: `filter`
    - Role: Filters workflows by name and excludes archived.

  - **Workflow Found 4**
    - Type: `if`
    - Role: Checks if workflow exists.

  - **List Workflow Executions**
    - Type: `n8n`
    - Role: Lists last 50 executions filtered by workflow ID.

  - **Executions Fields**
    - Type: `set`
    - Role: Formats execution fields like id, mode, start time, finished status, and workflow name.

  - **Executions Message**
    - Type: `code`
    - Role: Builds a formatted string listing executions with status icons and timestamps.

  - **Executions**
    - Type: `telegram`
    - Role: Sends the formatted executions message.

  - **Arg Error / Workflow Name Error**
    - Role: Sends error messages for missing argument or unknown workflow.

#### 1.6 Cleanup of Archived Workflows

- **Overview:** Finds archived workflows and deletes them permanently.
- **Nodes Involved:** List Archived, Only Archived, Archived List, Delete Archived, Archived Message, Cleanup

- **Node Details:**

  - **List Archived**
    - Type: `n8n`
    - Role: Retrieves workflows with `activeWorkflows` filter set to false.

  - **Only Archived**
    - Type: `filter`
    - Role: Filters workflows where `isArchived` is true.

  - **Archived List**
    - Type: `set`
    - Role: Prepares list of archived workflows with name and id.

  - **Delete Archived**
    - Type: `n8n`
    - Role: Deletes workflows by ID.

  - **Archived Message**
    - Type: `code`
    - Role: Constructs a Telegram message listing deleted archived workflows or none found.

  - **Cleanup**
    - Type: `telegram`
    - Role: Sends the cleanup result message.

#### 1.7 Backup Creation and Delivery

- **Overview:** Creates backup of all workflows and credentials, compresses them, sends the backup file via Telegram, then cleans up local files.
- **Nodes Involved:** Backup Workflows, Backup Credentials, Backup Tarball, Read File, Backup, Cleanup Files, Backup Error

- **Node Details:**

  - **Backup Workflows**
    - Type: `executeCommand`
    - Role: Runs shell command to export workflows with backup flag to local directory.

  - **Backup Credentials**
    - Type: `executeCommand`
    - Role: Exports decrypted credentials for backup.

  - **Backup Tarball**
    - Type: `executeCommand`
    - Role: Compresses backup folders into a tar.gz archive.

  - **Read File**
    - Type: `readWriteFile`
    - Role: Reads the backup tarball for sending.

  - **Backup**
    - Type: `telegram`
    - Role: Sends the backup file as a document to Telegram.

  - **Cleanup Files**
    - Type: `executeCommand`
    - Role: Deletes backup folders and archive after sending.

  - **Backup Error**
    - Type: `telegram`
    - Role: Sends error message if backup process fails.

  - Edge Cases:
    - Backup commands depend on self-hosted environment with access to n8n CLI.
    - Permission issues or disk space may cause failures.

#### 1.8 Notifications and Error Handling

- **Overview:** Sends notifications when workflows fail or when n8n instance starts.
- **Nodes Involved:** Error Trigger, If not manual exec, Workflow Error Msg, n8n Started Trigger, n8n Started Msg

- **Node Details:**

  - **Error Trigger**
    - Type: `errorTrigger`
    - Role: Triggers on workflow errors.

  - **If not manual exec**
    - Type: `if`
    - Role: Filters out manual executions to avoid duplicate notifications.

  - **Workflow Error Msg**
    - Type: `telegram`
    - Role: Sends detailed error message including workflow name, execution id, error message, and timestamp.

  - **n8n Started Trigger**
    - Type: `n8nTrigger`
    - Role: Triggers when the n8n instance starts (init event).

  - **n8n Started Msg**
    - Type: `telegram`
    - Role: Sends notification message that instance has started including timestamp.

  - Edge Cases:
    - Telegram API connectivity issues.
    - Large error messages or frequent errors may cause message flood.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                                 | Input Node(s)        | Output Node(s)                             | Sticky Note                                                                                          |
|-----------------------|--------------------|------------------------------------------------|----------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger      | telegramTrigger    | Receive Telegram messages                        |                      | Cmd Parse                                  | Setup instructions and notes on credentials                                                        |
| Cmd Parse             | set                | Parse command and argument from message         | Telegram Trigger     | Cmd Switch                                 |                                                                                                    |
| Cmd Switch            | switch             | Route commands to appropriate blocks             | Cmd Parse            | Help, Backup Workflows, List Archived, ... |                                                                                                    |
| Execute Arg           | if                 | Check if execute command has argument            | Cmd Switch (execute) | List Workflows 1, Arg Error                 |                                                                                                    |
| List Workflows 1      | n8n                 | Get all workflows                                 | Execute Arg          | Find Workflow 1                            |                                                                                                    |
| Find Workflow 1       | filter              | Find workflow by name                             | List Workflows 1     | Workflow Found 1                           |                                                                                                    |
| Workflow Found 1      | if                  | Check if workflow found                           | Find Workflow 1      | Execute Workflow, Workflow Name Error      |                                                                                                    |
| Execute Workflow      | executeWorkflow     | Execute the specified workflow                    | Workflow Found 1     | Executed, Execution Error                   |                                                                                                    |
| Executed              | telegram            | Notify successful execution                       | Execute Workflow     |                                            |                                                                                                    |
| Execution Error       | telegram            | Notify execution failure                          | Execute Workflow     |                                            |                                                                                                    |
| Arg Error             | telegram            | Notify missing argument error                     | Execute Arg, Activate Arg, Deactivate Arg, Executions Arg |                                            |                                                                                                    |
| Workflow Name Error   | telegram            | Notify workflow not found                         | Workflow Found 1, 2, 3, 4 |                                            |                                                                                                    |
| Activate Arg          | if                  | Check argument for activate command               | Cmd Switch (activate) | List Workflows 2, Arg Error                 |                                                                                                    |
| List Workflows 2      | n8n                 | Get all workflows                                 | Activate Arg         | Find Workflow 2                            |                                                                                                    |
| Find Workflow 2       | filter              | Find workflow by name                             | List Workflows 2     | Workflow Found 2                           |                                                                                                    |
| Workflow Found 2      | if                  | Check if workflow found                           | Find Workflow 2      | If Inactive, Workflow Name Error            |                                                                                                    |
| If Inactive           | if                  | Check if workflow inactive before activating     | Workflow Found 2     | Activate Workflow, Workflow Active Error    |                                                                                                    |
| Activate Workflow     | n8n                 | Activate the workflow                             | If Inactive          | Activated, Activation Error                  |                                                                                                    |
| Activated             | telegram            | Notify successful activation                      | Activate Workflow    |                                            |                                                                                                    |
| Activation Error      | telegram            | Notify activation failure                         | Activate Workflow    |                                            |                                                                                                    |
| Deactivate Arg        | if                  | Check argument for deactivate command             | Cmd Switch (deactivate) | List Workflows 3, Arg Error                 |                                                                                                    |
| List Workflows 3      | n8n                 | Get all workflows                                 | Deactivate Arg       | Find Workflow 3                            |                                                                                                    |
| Find Workflow 3       | filter              | Find workflow by name                             | List Workflows 3     | Workflow Found 3                           |                                                                                                    |
| Workflow Found 3      | if                  | Check if workflow found                           | Find Workflow 3      | If Active, Workflow Name Error               |                                                                                                    |
| If Active             | if                  | Check if workflow active before deactivating     | Workflow Found 3     | Deactivate Workflow, Workflow Inactive Error |                                                                                                    |
| Deactivate Workflow   | n8n                 | Deactivate the workflow                           | If Active            | Deactivated, Deactivation Error              |                                                                                                    |
| Deactivated           | telegram            | Notify successful deactivation                    | Deactivate Workflow  |                                            |                                                                                                    |
| Deactivation Error    | telegram            | Notify deactivation failure                       | Deactivate Workflow  |                                            |                                                                                                    |
| Executions Arg        | if                  | Check argument for executions command             | Cmd Switch (executions) | List Workflows 4, Arg Error                 |                                                                                                    |
| List Workflows 4      | n8n                 | Get all workflows                                 | Executions Arg       | Find Workflow 4                            |                                                                                                    |
| Find Workflow 4       | filter              | Find workflow by name                             | List Workflows 4     | Workflow Found 4                           |                                                                                                    |
| Workflow Found 4      | if                  | Check if workflow found                           | Find Workflow 4      | List Workflow Executions, Workflow Name Error |                                                                                                    |
| List Workflow Executions | n8n               | List executions of specified workflow            | Workflow Found 4     | Executions Fields                          |                                                                                                    |
| Executions Fields     | set                | Format execution data                             | List Workflow Executions | Executions Message                         |                                                                                                    |
| Executions Message    | code               | Format executions into a message string          | Executions Fields    | Executions                                 |                                                                                                    |
| Executions            | telegram           | Send executions list to Telegram                  | Executions Message   |                                            |                                                                                                    |
| List Archived          | n8n                 | Retrieve inactive workflows                        | Cmd Switch (cleanup) | Only Archived                              |                                                                                                    |
| Only Archived          | filter              | Filter workflows marked as archived               | List Archived        | Archived List                              |                                                                                                    |
| Archived List          | set                 | Prepare list of archived workflows                 | Only Archived        | Delete Archived                            |                                                                                                    |
| Delete Archived        | n8n                 | Delete archived workflows                         | Archived List        | Archived Message                           |                                                                                                    |
| Archived Message       | code                | Format message listing deleted workflows          | Delete Archived      | Cleanup                                    |                                                                                                    |
| Cleanup                | telegram            | Send cleanup result message                       | Archived Message     |                                            |                                                                                                    |
| Backup Workflows       | executeCommand      | Export workflows for backup                       | Cmd Switch (backup)  | Backup Credentials                         |                                                                                                    |
| Backup Credentials     | executeCommand      | Export decrypted credentials                      | Backup Workflows     | Backup Tarball                             |                                                                                                    |
| Backup Tarball         | executeCommand      | Compress backup files                             | Backup Credentials   | Read File                                  |                                                                                                    |
| Read File              | readWriteFile       | Read compressed backup tarball                    | Backup Tarball       | Backup, Backup Error                       |                                                                                                    |
| Backup                 | telegram            | Send backup tarball to Telegram                   | Read File            | Cleanup Files                              |                                                                                                    |
| Cleanup Files          | executeCommand      | Remove backup files after sending                  | Backup               |                                            |                                                                                                    |
| Backup Error           | telegram            | Notify backup failure                             | Read File (error)    |                                            |                                                                                                    |
| Error Trigger          | errorTrigger        | Trigger on workflow errors                        |                      | If not manual exec                         |                                                                                                    |
| If not manual exec     | if                  | Filter out manual executions                       | Error Trigger        | Workflow Error Msg                         |                                                                                                    |
| Workflow Error Msg     | telegram            | Notify detailed workflow error                    | If not manual exec    |                                            |                                                                                                    |
| n8n Started Trigger    | n8nTrigger          | Trigger on n8n instance start                      |                      | n8n Started Msg                           |                                                                                                    |
| n8n Started Msg        | telegram            | Notify n8n instance startup                        | n8n Started Trigger  |                                            |                                                                                                    |
| Cmd Error              | telegram            | Notify invalid command                             | Cmd Switch (fallback)|                                            |                                                                                                    |
| Arg Error              | telegram            | Notify missing argument                            | Execute Arg, Activate Arg, Deactivate Arg, Executions Arg |                                            |                                                                                                    |
| Workflow Name Error    | telegram            | Notify workflow not found                          | Workflow Found nodes |                                            |                                                                                                    |
| Workflow Active Error  | telegram            | Notify workflow already active                     | If Inactive          |                                            |                                                                                                    |
| Workflow Inactive Error| telegram            | Notify workflow already inactive                   | If Active            |                                            |                                                                                                    |
| Sticky Note            | stickyNote          | Documentation on workflow purpose and commands    |                      |                                            | Contains detailed explanation and command list                                                     |
| Sticky Note1           | stickyNote          | Setup instructions and credentials guidance       |                      |                                            | Provides detailed setup instructions with links                                                   |
| Telegram Nodes         | telegram            | Placeholder/test Telegram node                      |                      |                                            |                                                                                                    |
| n8n Nodes              | n8n                 | Placeholder/test n8n API node                       |                      |                                            |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and n8n API Credentials:**
   - Generate Telegram bot token via @BotFather.
   - Obtain your Telegram user ChatID/UserID for access control.
   - Create n8n API key in n8n settings and generate n8n API credentials in n8n.
   - Create Telegram API credentials in n8n using the bot token.

2. **Create Telegram Trigger Node:**
   - Node Type: Telegram Trigger
   - Parameters: Listen for "message" updates.
   - Restrict to your Telegram chat ID and user ID.
   - Set credentials to Telegram API credential.

3. **Create Cmd Parse Node:**
   - Node Type: Set
   - Parameters:
     - Extract `command`: Lowercase the message text, remove leading "/", split by first space, take first word.
     - Extract `arg1`: Lowercase the message text, remove leading "/", split by first space, take second part or empty string.
   - Connect Telegram Trigger → Cmd Parse.

4. **Create Cmd Switch Node:**
   - Node Type: Switch
   - Parameters:
     - Case-insensitive equals match on `command`.
     - Output keys: start, help, backup, cleanup, workflows, execute, activate, deactivate, executions.
     - Fallback output: Error.
   - Connect Cmd Parse → Cmd Switch.

5. **Create Help Command Telegram Node:**
   - Node Type: Telegram
   - Text: Detailed list of commands and usage.
   - Restrict chat ID.
   - Connect Cmd Switch (start and help outputs) → Help.

6. **Implement Execute Command Path:**
   - Create Execute Arg (if node) to check if `arg1` is not empty, connect Cmd Switch (execute) → Execute Arg.
   - On true path:
     - List Workflows 1 (n8n node) to get workflows.
     - Find Workflow 1 (filter) by comparing lowercased workflow name to `arg1`, exclude archived.
     - Workflow Found 1 (if) to check if workflow found.
     - True path → Execute Workflow (executeWorkflow node) with workflow ID.
       - On success → Executed (telegram) with "Workflow executed" message.
       - On error → Execution Error (telegram) with failure message.
     - False path → Workflow Name Error (telegram) with "Workflow name not found!"
   - On false path → Arg Error (telegram) with "This command requires an argument!"

7. **Implement Activate Command Path:**
   - Repeat similar structure as execute path:
     - Activate Arg (if) checks argument presence.
     - List Workflows 2 → Find Workflow 2 → Workflow Found 2
     - If Inactive (if) checks workflow active status.
       - True → Activate Workflow (n8n node) to activate workflow by ID.
         - On success → Activated (telegram)
         - On error → Activation Error (telegram)
       - False → Workflow Active Error (telegram)
     - False from Workflow Found 2 → Workflow Name Error
     - False from Activate Arg → Arg Error

8. **Implement Deactivate Command Path:**
   - Similar structure as activate path but inverse:
     - Deactivate Arg (if) → List Workflows 3 → Find Workflow 3 → Workflow Found 3
     - If Active (if)
       - True → Deactivate Workflow (n8n node)
         - On success → Deactivated (telegram)
         - On error → Deactivation Error (telegram)
       - False → Workflow Inactive Error (telegram)
     - Else error handlers as above.

9. **Implement Executions Command Path:**
   - Executions Arg (if) → List Workflows 4 → Find Workflow 4 → Workflow Found 4
   - List Workflow Executions (n8n) fetches last 50 executions for workflow ID.
   - Executions Fields (set) formats fields.
   - Executions Message (code) builds a text listing executions with status.
   - Executions (telegram) sends the message.
   - Error paths for missing argument and unknown workflow.

10. **Implement Cleanup Command Path:**
    - List Archived (n8n) fetches inactive workflows.
    - Only Archived (filter) filters for archived workflows.
    - Archived List (set) prepares workflow name and ID.
    - Delete Archived (n8n) deletes each workflow by ID.
    - Archived Message (code) composes confirmation message.
    - Cleanup (telegram) sends confirmation.

11. **Implement Backup Command Path:**
    - Backup Workflows (executeCommand) runs `n8n export:workflow --backup --output=/home/node/backup/workflows`.
    - Backup Credentials (executeCommand) runs `n8n export:credentials --backup --decrypted --output=/home/node/backup/credentials`.
    - Backup Tarball (executeCommand) creates tar.gz archive of backups.
    - Read File (readWriteFile) reads the tarball.
    - Backup (telegram) sends the tarball as document.
    - Cleanup Files (executeCommand) deletes backup files.
    - Backup Error (telegram) on failure.

12. **Implement Error Notifications:**
    - Error Trigger (errorTrigger node) triggers on workflow errors.
    - If not manual exec (if) filters out manual executions.
    - Workflow Error Msg (telegram) sends detailed error message.

13. **Implement Instance Startup Notification:**
    - n8n Started Trigger (n8nTrigger node) triggers on instance init.
    - n8n Started Msg (telegram) sends instance started notification.

14. **Add Cmd Error (telegram) node:**
    - Connected to Cmd Switch fallback output to notify of invalid commands.

15. **Set Credentials for all Telegram Nodes:**
    - Use Telegram API credential with bot token.
    - Set chat ID to your Telegram user ID.

16. **Set Credentials for all n8n Nodes:**
    - Use n8n API credential with API key and server URL.

17. **Activate the workflow and test commands in Telegram chat.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow connects your n8n instance with a Telegram bot to enable remote control using chat commands. Commands include listing workflows, executing workflows, activating/deactivating workflows, listing executions, cleanup of archived workflows, and backup creation.                                                                                                                                                                                                                                                             | Sticky Note at workflow start                                                                                                              |
| Setup requires creation of Telegram bot token, acquiring your Telegram user ID, and creating n8n API key credentials. Configure these credentials in all Telegram and n8n nodes accordingly.                                                                                                                                                                                                                                                                                                                                                   | Sticky Note with detailed setup instructions and links: https://docs.n8n.io/credentials/ and https://bigone.zendesk.com/hc/en-us/articles/360008014894-How-to-get-the-Telegram-user-ID#:~:text=1.,The%20Number%20ID |
| Backup commands only work on self-hosted n8n instances with access to the n8n CLI and filesystem. Backup files contain decrypted credentials; handle with care and secure accordingly.                                                                                                                                                                                                                                                                                                                                                       | Sticky Note warnings                                                                                                                        |
| To execute a workflow remotely, it must have a trigger node of type "When Executed by Another Workflow". To activate a workflow remotely, it must have a trigger node that supports activation.                                                                                                                                                                                                                                                                                                                                             | Sticky Note command usage guidelines                                                                                                       |
| Telegram API may have rate limits; frequent commands or error notifications may result in delays or dropped messages.                                                                                                                                                                                                                                                                                                                                                                                                                      | General caution                                                                                                                             |
| For security, restrict Telegram bot responses to your user ID and chat ID only.                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Security best practice                                                                                                                      |

---

**Disclaimer:** The text provided is exclusively derived from an n8n workflow automation. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.