🤖 Advanced Slackbot with n8n

https://n8nworkflows.xyz/workflows/---advanced-slackbot-with-n8n-2157


# 🤖 Advanced Slackbot with n8n

### 1. Workflow Overview

This workflow implements an advanced Slackbot using n8n, designed for internal automation tasks such as running tests or managing users via Slack commands. It focuses on modularity by delegating specific commands to separate subworkflows, making it easier to extend, debug, and maintain.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation**: Receives Slack commands via webhook, validates the request authenticity using Slack’s signing secret and a pre-shared Slack token.
- **1.2 Command Parsing**: Parses the command text into command name, parameters, flags, and environment variables, and determines if a corresponding subworkflow exists.
- **1.3 Command Routing and Execution**: Depending on whether the command has an associated workflow, it either runs the subworkflow, handles special commands like help, or responds with an unknown command message.
- **1.4 Thread and User Interaction**: Manages Slack threads for commands that require it, sends confirmations and debug info back to the Slack user or thread.
- **1.5 Example Command Subworkflow (Skeleton)**: Two example subworkflows show how to implement commands with or without Slack threads, including user info retrieval and user deletion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

- **Overview:**  
  Receives incoming Slack command requests via webhook, validates the Slack request signature and token to ensure authenticity and freshness.

- **Nodes Involved:**  
  - Webhook to call for Slack command  
  - Set vars  
  - Set config  
  - Validate webhook signature  
  - Validate Slack token

- **Node Details:**

  - **Webhook to call for Slack command**  
    - Type: Webhook  
    - Role: Entry point receiving POST requests from Slack slash commands  
    - Configuration: Path set to a unique webhook ID, rawBody enabled to access raw payload for signature validation, HTTP method POST  
    - Inputs: HTTP requests from Slack  
    - Outputs: JSON and binary payload to downstream nodes  
    - Edge cases: Missing or malformed requests, slow or dropped requests  

  - **Set vars**  
    - Type: Set  
    - Role: Extracts and stores key parameters from the Slack command payload (command text, user, response URL, token, command name)  
    - Key expressions:  
      - `command_text` = `$json.body.text`  
      - `user` = `$json.body.user_name`  
      - `response_url` = `$json.body.response_url`  
      - `request_token` = `$json.body.token`  
      - `command_name` = `$json.body.command`  
    - Inputs: Webhook output  
    - Outputs: JSON with extracted variables  

  - **Set config**  
    - Type: Set  
    - Role: Contains configurable parameters such as:  
      - `commands`: Object mapping command names to workflow IDs and thread behavior  
      - `alerts_channel`: Slack channel to start alert threads  
      - `instance_url`: n8n instance URL for debug links  
      - `slack_token`: Token to validate incoming Slack command tokens  
      - `slack_secret_signature`: Slack signing secret for webhook validation  
      - `help_docs_url`: URL for the help documentation  
    - Inputs: Set vars output  
    - Outputs: JSON enriched with config  
    - Notes: User must fill in sensitive tokens and URLs before activation  

  - **Validate webhook signature**  
    - Type: Code  
    - Role: Validates the Slack request signature using the Slack secret and raw request body to prevent replay and forgery attacks  
    - Key logic:  
      - Checks timestamp freshness (within 5 minutes)  
      - Computes HMAC SHA256 signature and compares to Slack header  
    - Inputs: Set config output (has secret), Webhook binary payload  
    - Outputs: Passed inputs if valid; otherwise throws error stopping workflow  
    - Edge cases: Missing headers, expired requests, signature mismatch  

  - **Validate Slack token**  
    - Type: If  
    - Role: Compares the configured Slack token against the token received in the command payload to verify the request source  
    - Conditions: `$json.slack_token === $json.request_token`  
    - Inputs: Validate webhook signature output  
    - Outputs: Passes if equal, otherwise stops or branches  
    - Edge cases: Token mismatch results in workflow termination  

---

#### 1.2 Command Parsing

- **Overview:**  
  Parses the command text into structured components (command name, flags, parameters, environment variables) and determines if a corresponding subworkflow exists.

- **Nodes Involved:**  
  - parse command  
  - if has workflow

- **Node Details:**

  - **parse command**  
    - Type: Code  
    - Role:  
      - Splits the command text by spaces  
      - Extracts the first word as the command name  
      - Extracts flags starting with `--`  
      - Extracts parameters (words excluding flags and environment variables)  
      - Extracts environment variables specified with `-e` flag in `key=value` format  
      - Looks up the command in the `commands` config to find workflow metadata  
    - Key expressions: JavaScript code parsing `$json.command_text` and `$json.commands`  
    - Inputs: Validate Slack token output  
    - Outputs: JSON extended with `command`, `flags`, `params`, `env`, and `workflow` (workflow info if exists)  
    - Edge cases: Commands with malformed flags or env vars; missing commands result in empty workflow  

  - **if has workflow**  
    - Type: If  
    - Role: Checks if a workflow ID exists for the parsed command  
    - Condition: Workflow object is not empty  
    - Inputs: parse command output  
    - Outputs: True branch for existing workflow, false branch for unknown or special commands  

---

#### 1.3 Command Routing and Execution

- **Overview:**  
  Routes commands to corresponding subworkflows or handles special commands such as help or unknown commands. Also determines if a Slack thread should be created.

- **Nodes Involved:**  
  - if create thread  
  - Start thread  
  - Alert user that thread was created  
  - Send debug url  
  - Execute target workflow  
  - Handle other commands  
  - send help  
  - Unknown command

- **Node Details:**

  - **if create thread**  
    - Type: If  
    - Role: Checks if the command’s workflow config requires starting a Slack thread  
    - Condition: `startThread` is true or not explicitly set (default to true)  
    - Inputs: if has workflow output (true branch)  
    - Outputs: True branch to create thread, false branch to run workflow without thread  

  - **Start thread**  
    - Type: Slack node  
    - Role: Posts a message starting a new thread in the configured alerts channel, notifying of the incoming command request  
    - Configuration:  
      - Channel: `alerts_channel` from config  
      - Text: "🧵 Got request to `command` from @user"  
      - Uses Slack bot credentials  
    - Inputs: if create thread true branch  
    - Outputs: Message details including channel and thread timestamp  

  - **Alert user that thread was created**  
    - Type: HTTP Request  
    - Role: Sends a Slack message back to the user response_url confirming thread creation  
    - Body: Attachment with text "🧵 Thread created on alerts_channel"  
    - Inputs: Start thread output  
    - Outputs: Confirmation HTTP response  

  - **Send debug url**  
    - Type: HTTP Request  
    - Role: Sends a debug link referencing the n8n workflow execution to the Slack response_url for troubleshooting  
    - Body: Attachment with a markdown link to the execution URL  
    - Inputs: if create thread false branch  
    - Outputs: Confirmation HTTP response  

  - **Execute target workflow**  
    - Type: Execute Workflow  
    - Role: Runs the subworkflow corresponding to the command based on workflow ID from config  
    - Inputs: Add thread info or direct branch  
    - Outputs: Execution results passed downstream  

  - **Handle other commands**  
    - Type: Switch  
    - Role: Routes commands that do not have workflows mapped  
    - Cases:  
      - Help command: routes to send help  
      - Default: routes to unknown command response  

  - **send help**  
    - Type: HTTP Request  
    - Role: Responds to the Slack response_url with a help message linking to documentation  
    - Body: Attachment with text linking to help_docs_url  

  - **Unknown command**  
    - Type: HTTP Request  
    - Role: Responds with a message indicating the command is unknown  
    - Body: Attachment with apology and unknown command name  

---

#### 1.4 Thread and User Interaction

- **Overview:**  
  Manages Slack message replies either within threads or directly to users, sending debug info and confirmations.

- **Nodes Involved:**  
  - Reply to user that command was received  
  - Set thread info  
  - Add thread info  
  - Add debug info  
  - Replying to thread  
  - Reply to user directly

- **Node Details:**

  - **Reply to user that command was received**  
    - Type: HTTP Request  
    - Role: Immediately acknowledges the Slack command by posting a message confirming receipt of the command and parameters  
    - Body: Attachment with "ℹ️ Got command `command_name command_text`"  
    - Inputs: Validate Slack token output  
    - Outputs: HTTP response  

  - **Set thread info**  
    - Type: Set  
    - Role: Extracts and sets Slack channel ID and thread timestamp from the message event to pass along for thread replies  
    - Inputs: Start thread output  
    - Outputs: JSON with `channel_id` and `thread_ts`  

  - **Add thread info**  
    - Type: Merge  
    - Role: Combines thread info with command JSON before executing the target workflow, multiplexing inputs  
    - Inputs: Set thread info and Start thread outputs  
    - Outputs: Merged JSON for subworkflow execution  

  - **Add debug info**  
    - Type: Slack node  
    - Role: Posts a debug link back to the Slack thread or channel where the command was issued  
    - Configuration: Posts message with link to current execution on n8n instance URL  
    - Inputs: Start thread output  

  - **Replying to thread**  
    - Type: Slack node  
    - Role: Posts debug or status messages inside the existing Slack thread for commands running in threads  
    - Inputs: Subworkflow output  

  - **Reply to user directly**  
    - Type: HTTP Request  
    - Role: Posts debug or confirmation messages directly to the user’s response_url for direct replies (non-threaded)  
    - Inputs: Command workflow trigger output  

---

#### 1.5 Example Command Subworkflow (Skeleton)

- **Overview:**  
  Two example subworkflows illustrate how to implement commands with or without Slack threads, including data querying and user deletion workflows.

- **Nodes Involved:**  
  - Command workflow trigger  
  - if has flag  
  - If matches env variable  
  - Get user here for example (Postgres select)  
  - Format data into nice structure (Code)  
  - Found user (Slack message)  
  - REPLACE ME WITH TRIGGER (Set, placeholder)  
  - Delete user here for example (Postgres delete)  
  - Confirm user was deleted (Slack message)  

- **Node Details:**

  - **Command workflow trigger**  
    - Type: Execute Workflow Trigger (disabled in main)  
    - Role: Entry point for subworkflow execution by main workflow  
    - Inputs: Parameters passed from main workflow execution  

  - **if has flag**  
    - Type: If  
    - Role: Checks for presence of specific flags (e.g., `--full-info`) in command flags array  

  - **If matches env variable**  
    - Type: If  
    - Role: Condition based on environment variable, e.g., env=prod  

  - **Get user here for example**  
    - Type: Postgres node (select)  
    - Role: Retrieves user information based on username parameter from database  

  - **Format data into nice structure**  
    - Type: Code  
    - Role: Formats retrieved user data into Slack Block Kit message format for rich display  

  - **Found user**  
    - Type: Slack node  
    - Role: Posts formatted user information back to Slack  

  - **REPLACE ME WITH TRIGGER**  
    - Type: Set (placeholder)  
    - Role: Placeholder node to be replaced with actual trigger logic in user deletion example  

  - **Delete user here for example**  
    - Type: Postgres node (delete) (disabled)  
    - Role: Deletes user record from the database based on parameters  

  - **Confirm user was deleted**  
    - Type: Slack node  
    - Role: Posts confirmation that user deletion succeeded within Slack thread  

---

### 3. Summary Table

| Node Name                      | Node Type                   | Functional Role                                | Input Node(s)                    | Output Node(s)                              | Sticky Note                                                                                                   |
|-------------------------------|-----------------------------|------------------------------------------------|---------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook to call for Slack command | Webhook                    | Entry point receiving Slack command requests   | -                               | Set vars                                    | Setup instructions for Slack command webhook and config                                                       |
| Set vars                      | Set                         | Extracts variables from Slack payload           | Webhook to call for Slack command | Set config                                   |                                                                                                               |
| Set config                   | Set                         | Holds config like commands, tokens, URLs        | Set vars                        | Validate webhook signature                   | ### 👆🏽 Set all custom config here                                                                             |
| Validate webhook signature    | Code                        | Validates Slack webhook signature for security | Set config                     | Validate Slack token                         |                                                                                                               |
| Validate Slack token          | If                          | Validates Slack command token                    | Validate webhook signature      | Reply to user that command was received, parse command |                                                                                                               |
| Reply to user that command was received | HTTP Request               | Acknowledges command receipt                      | Validate Slack token            | parse command                               |                                                                                                               |
| parse command                | Code                        | Parses command text into parts                    | Reply to user that command was received | if has workflow                              |                                                                                                               |
| if has workflow              | If                          | Checks if command has an associated workflow     | parse command                  | if create thread (true), Handle other commands (false) |                                                                                                               |
| if create thread             | If                          | Determines if command workflow requires thread   | if has workflow                | Start thread (true), Send debug url + Execute target workflow (false) |                                                                                                               |
| Start thread                | Slack                       | Starts Slack thread to notify alerts channel     | if create thread               | Add debug info, Set thread info              |                                                                                                               |
| Alert user that thread was created | HTTP Request               | Notifies user that thread was created             | Start thread                   | -                                           |                                                                                                               |
| Add debug info              | Slack                       | Posts debug link into thread/channel              | Start thread                   | -                                           |                                                                                                               |
| Set thread info             | Set                         | Sets Slack thread info (channel id, ts)          | Start thread                   | Add thread info                             |                                                                                                               |
| Add thread info             | Merge                       | Merges thread info with command JSON              | Set thread info, Start thread   | Execute target workflow                      |                                                                                                               |
| Execute target workflow     | Execute Workflow            | Runs the subworkflow associated with command     | Add thread info / if create thread false branch | -                                         |                                                                                                               |
| Send debug url              | HTTP Request               | Sends debug URL to Slack response_url             | if create thread false branch  | Execute target workflow                      |                                                                                                               |
| Handle other commands        | Switch                      | Handles help and unknown commands                 | if has workflow false branch   | send help (help), Unknown command (extra)  |                                                                                                               |
| send help                   | HTTP Request               | Sends help documentation link to Slack           | Handle other commands (help)   | -                                           |                                                                                                               |
| Unknown command             | HTTP Request               | Notifies unknown command to Slack user            | Handle other commands (extra)  | -                                           |                                                                                                               |
| Reply to user directly      | HTTP Request               | Replies directly to user with debug info          | Command workflow trigger       | if has flag                                 |                                                                                                               |
| Command workflow trigger    | Execute Workflow Trigger (disabled) | Subworkflow entry for command execution           | -                               | Reply to user directly, if has flag         | Example subworkflow for command WITHOUT Slack thread                                                          |
| if has flag                 | If                          | Checks for specific flags in command              | Command workflow trigger       | If matches env variable                      |                                                                                                               |
| If matches env variable     | If                          | Checks if env variable matches criteria           | if has flag                   | Get user here for example                    |                                                                                                               |
| Get user here for example   | Postgres (select)           | Retrieves user info from DB                        | If matches env variable        | Format data into nice structure              |                                                                                                               |
| Format data into nice structure | Code                      | Formats user info into Slack message blocks       | Get user here for example      | Found user                                  |                                                                                                               |
| Found user                  | Slack                       | Sends formatted user info to Slack                | Format data into nice structure | -                                           |                                                                                                               |
| REPLACE ME WITH TRIGGER     | Set (placeholder)           | Placeholder for subworkflow trigger                | -                               | Replying to thread, Delete user here for example | Example subworkflow for command WITH Slack thread                                                             |
| Replying to thread          | Slack                       | Posts debug info in Slack thread                    | REPLACE ME WITH TRIGGER        | -                                           |                                                                                                               |
| Delete user here for example | Postgres (delete) (disabled) | Deletes a user in DB                                | REPLACE ME WITH TRIGGER        | Confirm user was deleted                      |                                                                                                               |
| Confirm user was deleted    | Slack                       | Confirms user deletion in Slack thread             | Delete user here for example   | -                                           |                                                                                                               |
| Webhook to call for Slack command | Webhook                    | Entry-point webhook for Slack commands             | -                               | Set vars                                    | See setup instructions sticky note                                                                              |
| Sticky Note                 | Sticky Note                 | Example subworkflow for command WITHOUT Slack thread | -                               | -                                           | Example subworkflow for command WITHOUT Slack thread                                                          |
| Sticky Note2                | Sticky Note                 | Example subworkflow for command WITH Slack thread  | -                               | -                                           | Example subworkflow for command WITH Slack thread                                                             |
| Sticky Note3                | Sticky Note                 | Setup instructions for Slackbot and config         | -                               | -                                           | Setup instructions including Slack command, config, and activation steps                                       |
| Sticky Note1                | Sticky Note                 | Reminder to set all custom config parameters        | -                               | -                                           | ### 👆🏽 Set all custom config here                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: Unique identifier (e.g., `a14585bb-b757-410e-a5b2-5f05a087b388`)  
   - HTTP Method: POST  
   - Raw Body: Enabled  
   - Response Data: Wait for it...  
   - Binary Property Name: `data`

2. **Create Set Node ("Set vars")**  
   - Extract from webhook body:  
     - `command_text` = `{{$json.body.text}}`  
     - `user` = `{{$json.body.user_name}}`  
     - `response_url` = `{{$json.body.response_url}}`  
     - `request_token` = `{{$json.body.token}}`  
     - `command_name` = `{{$json.body.command}}`  
   - Connect webhook output to this node.

3. **Create Set Node ("Set config")**  
   - Assign these variables:  
     - `commands` = JSON object mapping commands to workflow IDs, e.g.:  
       ```json
       {
         "info": { "workflowId": 142, "startThread": false },
         "delete-user": { "workflowId": "pTh9HMZVYcQNXypJ" }
       }
       ```  
     - `alerts_channel` = Slack channel name for threads (e.g., `#adore_bot_test`)  
     - `instance_url` = Your n8n instance URL (e.g., `https://x.app.n8n.cloud/`)  
     - `slack_token` = Your Slack bot token (must be filled)  
     - `slack_secret_signature` = Your Slack signing secret (must be filled)  
     - `help_docs_url` = URL to your help documentation  
   - Connect "Set vars" output to this node.

4. **Create Code Node ("Validate webhook signature")**  
   - Paste JavaScript code to validate Slack signature and freshness using `slack_secret_signature` and raw webhook body (see node details above)  
   - Connect "Set config" output to this node.

5. **Create If Node ("Validate Slack token")**  
   - Condition: `$json.slack_token === $json.request_token`  
   - Connect "Validate webhook signature" output to this node.

6. **Create HTTP Request Node ("Reply to user that command was received")**  
   - Method: POST  
   - URL: `{{$json.response_url}}`  
   - Body (JSON): Attachment with text `ℹ️ Got command \`{{$json.command_name}} {{$json.command_text}}\``  
   - Connect "Validate Slack token" true branch output to this node.

7. **Create Code Node ("parse command")**  
   - JavaScript code to parse command text into `command`, `flags`, `params`, `env`, and find associated workflow from `commands` config  
   - Connect "Reply to user that command was received" output to this node.

8. **Create If Node ("if has workflow")**  
   - Condition: Workflow object is not empty (not empty check)  
   - Connect "parse command" output to this node.

9. **Create If Node ("if create thread")**  
   - Condition: `$json.workflow.startThread === true` OR not set (default true)  
   - Connect "if has workflow" true branch to this node.

10. **Create Slack Node ("Start thread")**  
    - Post to Slack channel: `{{$json.alerts_channel}}`  
    - Text: `🧵 Got request to \`{{$json.command}}\` from @{{$json.user}}`  
    - Use Slack bot credentials  
    - Connect "if create thread" true branch to this node.

11. **Create HTTP Request Node ("Alert user that thread was created")**  
    - POST to `{{$json.response_url}}`  
    - Body: Attachment with text `🧵 Thread created on {{$json.alerts_channel}}`  
    - Connect "Start thread" output to this node.

12. **Create Set Node ("Set thread info")**  
    - Set `channel_id` = `{{$json.channel}}`  
    - Set `thread_ts` = `{{$json.message.ts}}`  
    - Connect "Start thread" output to this node.

13. **Create Merge Node ("Add thread info")**  
    - Mode: Combine (multiplex)  
    - Connect "Set thread info" and "Start thread" outputs to this node.

14. **Create Execute Workflow Node ("Execute target workflow")**  
    - Workflow ID: `={{$json.commands[$json.command].workflowId}}` or `{{$json.workflow.workflowId}}`  
    - Connect "Add thread info" output to this node.

15. **Create HTTP Request Node ("Send debug url")**  
    - POST to `{{$json.response_url}}`  
    - Body: Attachment with markdown link `<{{$json.instance_url}}/workflow/{{$workflow.id}}/executions/{{$execution.id}}|To debug entry point execution>`  
    - Connect "if create thread" false branch to this node.

16. **Connect "Send debug url" output to "Execute target workflow"**  
    - Ensures debug info is sent before workflow execution for non-threaded commands.

17. **Create Switch Node ("Handle other commands")**  
    - Case "help": if `$json.command === "help"`  
    - Fallback: unknown command  
    - Connect "if has workflow" false branch to this node.

18. **Create HTTP Request Node ("send help")**  
    - POST to `{{$json.response_url}}`  
    - Body: Attachment with text `ℹ️ <{{$json.help_docs_url}}|You can find help page here>`  
    - Connect "Handle other commands" help case to this node.

19. **Create HTTP Request Node ("Unknown command")**  
    - POST to `{{$json.response_url}}`  
    - Body: Attachment with text `🤷🏽‍♂️ Sorry, unknown command \`{{$json.command}}\``  
    - Connect "Handle other commands" fallback to this node.

20. **Create Slack Node ("Add debug info")**  
    - Posts debug link to Slack thread/channel: `<{{$vars.instance_url}}/workflow/{{$workflow.id}}/executions/{{$execution.id}}|To debug entry point execution>`  
    - Channel: `{{$json.channel}}`  
    - Thread timestamp: `{{$json.message.ts}}`  
    - Connect "Start thread" output to this node.

21. **Subworkflow Setup (Example without Slack thread)**  
    - Create separate workflow triggered via Execute Workflow node  
    - Implement your command logic  
    - Connect outputs to HTTP Request nodes to reply to user or thread  

22. **Subworkflow Setup (Example with Slack thread)**  
    - Use Execute Workflow Trigger node as entry point  
    - Include nodes for flag checks, env variable checks  
    - Use database nodes (Postgres) for user info or deletion  
    - Format response with Code nodes and Slack nodes for rich formatting  
    - Reply inside Slack thread using Slack nodes  

23. **Credential Setup**  
    - Create Slack API credential with bot token for Slack nodes  
    - Configure OAuth or API keys for database nodes (Postgres) if used  

24. **Activate Workflow**  
    - Ensure all tokens and secrets are filled  
    - Test Slack slash command pointing to webhook URL  
    - Debug using Slack messages and debug URL links  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup instructions: Add Slack command pointing to webhook, configure tokens and URLs in Set config node, build additional command workflows and activate.     | Sticky Note3 content inside workflow                                                                |
| This Slackbot supports flags and environment variables in commands, e.g. `/cloudbot-test info mutasem --full-info -e env=prod`.                               | See parse command node code and example commands                                                    |
| Example subworkflow templates provided for commands with and without Slack threads to guide custom command implementation.                                     | Sticky Note and Sticky Note2 nodes                                                                  |
| Debug URLs posted back to Slack enable quick access to workflow executions for troubleshooting and development.                                               | Send debug url and Add debug info Slack nodes                                                      |
| Slack signature validation code protects against replay and forgery attacks by verifying request freshness and signature correctness.                         | Validate webhook signature node                                                                     |
| This workflow uses modular subworkflows triggered via Execute Workflow nodes to isolate command logic and maintain code clarity.                              | Command workflow trigger and Execute target workflow nodes                                          |
| For detailed Slack slash command setup, refer to official Slack documentation: https://api.slack.com/interactivity/slash-commands                          | Helpful external link for Slack slash command configuration                                        |
| n8n Slack node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/                                                       | Reference for Slack node configuration and usage                                                    |
| n8n Execute Workflow node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.executeWorkflow/                                                           | Guidance on subworkflow execution                                                                    |

---

This documentation fully describes the "🤖 Advanced Slackbot with n8n" workflow in a structured manner, enabling advanced users and AI agents to understand, reproduce, and extend it effectively.