Monitor & Manage Docker Containers with Telegram Bot & AI Log Analysis

https://n8nworkflows.xyz/workflows/monitor---manage-docker-containers-with-telegram-bot---ai-log-analysis-10476


# Monitor & Manage Docker Containers with Telegram Bot & AI Log Analysis

---
### 1. Workflow Overview

This workflow enables monitoring and management of Docker containers via a Telegram bot interface, enhanced with AI-powered log analysis. It targets system administrators or DevOps engineers who want to:

- Receive real-time Docker container status updates and alerts.
- Request logs and restart Docker containers through Telegram commands.
- Automatically analyze logs with AI to identify issues.
- Manage Docker container updates and deployments safely via chat.
- Integrate with external monitoring tools via incoming webhooks.

The workflow is logically divided into the following blocks:

- **1.1 Incoming Webhook Reception**: Receives heartbeat or status messages from external monitoring tools.
- **1.2 Telegram User Interaction**: Listens for user commands in Telegram and routes commands accordingly.
- **1.3 Docker Container Command Processing**: Parses and processes commands related to logs retrieval, container restart, status listing, and updates.
- **1.4 AI Log Analysis**: Uses OpenAI to analyze Docker logs and provide structured diagnostic feedback.
- **1.5 Docker Command Execution**: Executes SSH commands to manage Docker containers (get logs, restart containers, list status, perform updates).
- **1.6 Telegram Notification & Feedback**: Sends responses, status updates, and error messages back to the user on Telegram.
- **1.7 Custom Update Script Execution**: Runs a custom shell script to update all Docker containers and summarizes the results.

---

### 2. Block-by-Block Analysis

#### 2.1 Incoming Webhook Reception

- **Overview**: Receives POST requests via webhook from external monitoring tools (e.g., Uptime Kuma). Checks if the heartbeat message indicates running status or errors, then forwards the message for further handling.
- **Nodes Involved**: Webhook, Switch1, OK Message, ERROR Message
- **Node Details**:
  - **Webhook**
    - Type: HTTP Webhook
    - Config: POST method at path `c851e840-457c-4050-8803-4e6d258580e1`
    - Input: External HTTP POST
    - Output: JSON body with heartbeat info
    - Edge cases: Missing or malformed JSON body, unexpected heartbeat values
  - **Switch1**
    - Type: Conditional router
    - Config: Checks if `body.heartbeat.msg` equals "running"
    - Output 1: If running, sends OK Message
    - Output 2: If not running, sends ERROR Message
    - Edge cases: Missing heartbeat.msg field, unexpected values
  - **OK Message**
    - Type: Telegram Message node
    - Config: Sends plain text from `body.msg` to fixed chat ID
    - Edge cases: Telegram API downtime or invalid chat ID
  - **ERROR Message**
    - Type: Telegram Message node
    - Config: Sends error details with custom keyboard for "Logs" or "Restart" commands for the specific container
    - Edge cases: Message formatting issues, Telegram API errors

#### 2.2 Telegram User Interaction

- **Overview**: Listens for Telegram messages and triggers corresponding workflow branches based on commands like "logs", "restart", "status", or "update".
- **Nodes Involved**: Telegram Trigger, Switch, Extract the Service Name, Merge, Merge1
- **Node Details**:
  - **Telegram Trigger**
    - Type: Telegram Trigger Node
    - Config: Listens for new messages (updates "message")
    - Output: Incoming Telegram message JSON
    - Edge cases: Telegram API connection issues, rate limits
  - **Switch**
    - Type: Router based on message text content (case insensitive)
    - Routes incoming messages containing "logs", "restart", "status", or "update" to respective branches
    - Edge cases: Unrecognized commands, empty messages
  - **Extract the Service Name**
    - Type: Code Node (Python)
    - Config: Extracts first word from Telegram message text as the Docker service name
    - Input: Telegram message text
    - Output: JSON with `service_name`
    - Edge cases: Empty or malformed messages leading to empty service_name
  - **Merge / Merge1**
    - Type: Merge Node (combine by position)
    - Combines service_name extraction output with command branch for downstream processing
    - Edge cases: Missing inputs, synchronization issues between branches

#### 2.3 Docker Container Command Processing & Execution

- **Overview**: Based on user commands, runs SSH commands on the Docker host to retrieve logs, restart containers, show status, or update containers.
- **Nodes Involved**: get logs, restart container, docker ps, Update Docker, Code in Python (Beta)
- **Node Details**:
  - **get logs**
    - Type: SSH Command
    - Command: `docker logs --tail 100 <service_name>`
    - Input: service_name from merged data
    - Output: Logs text in stdout
    - Edge cases: SSH connectivity issues, container not found, permission errors
  - **restart container**
    - Type: SSH Command
    - Command: `docker restart <service_name>`
    - Edge cases: SSH failure, container restart failure
  - **docker ps**
    - Type: SSH Command
    - Command: `docker ps --format "{{.Names}}\t{{.Status}}"`
    - Purpose: List all running Docker containers with status
    - Edge cases: SSH failure, empty output if no containers running
  - **Update Docker**
    - Type: SSH Command
    - Command: `./update-all-docker-compose.sh` (custom script)
    - Always outputs data for further processing
    - Edge cases: Script execution failure, permission denied
  - **Code in Python (Beta)**
    - Type: Code Node (Python)
    - Parses update script output to generate a user-friendly update summary message
    - Edge cases: Unexpected output format, JSON parsing errors

#### 2.4 AI Log Analysis

- **Overview**: Sends the retrieved Docker logs to OpenAI GPT-4.1-mini model to analyze logs, diagnose issues, and generate structured feedback.
- **Nodes Involved**: Message a model, Status Update, Log Analysis
- **Node Details**:
  - **Message a model**
    - Type: OpenAI Node (LangChain integration)
    - Model: GPT-4.1-mini
    - System prompt instructs the AI to analyze logs and produce a structured diagnostic report including summary, root cause, impact, evidence, severity, recommended steps, and monitoring advice
    - Input: Docker logs from `get logs` node (stdout)
    - Edge cases: API authentication issues, model response timeout, malformed logs input
  - **Status Update**
    - Type: Telegram Message Node
    - Sends interim message "Analyzing Log File..." to user on Telegram while AI processes logs
    - Edge cases: Telegram API issues
  - **Log Analysis**
    - Type: Telegram Message Node
    - Sends AI-generated analysis text back to Telegram user
    - Edge cases: Empty or malformed AI response, Telegram API errors

#### 2.5 Telegram Notification & Feedback

- **Overview**: Sends various status, success, or failure messages to users on Telegram depending on command outcomes.
- **Nodes Involved**: OK Message, ERROR Message, Status Update, Log Analysis, Restart Message, Success restart, Restart Failed, Docker Status, Update Msg, Update Msg1
- **Node Details**:
  - Each node sends specific messages to a fixed Telegram chatId `8209283663`.
  - Messages include status updates, error alerts, restart attempts, restart success/failure, Docker container statuses, and update progress.
  - Edge cases: Telegram API rate limits, invalid chat IDs, message formatting errors

#### 2.6 Custom Update Script Execution

- **Overview**: Runs a custom bash script (`update-all-docker-compose.sh`) which iterates through all Docker Compose files (except the n8n instance), pulls updated images, restarts services, and generates a JSON summary of updates.
- **Nodes Involved**: Update Docker, Code in Python (Beta), Update Msg, Update Msg1
- **Node Details**:
  - **Update Docker**
    - Runs the shell script via SSH
  - **Code in Python (Beta)**
    - Parses the script output to extract update statuses per compose file and formats a summary message
  - **Update Msg**
    - Sends "Running Update..." message before running the update
  - **Update Msg1**
    - Sends formatted update summary message to Telegram user
  - Edge cases: Script errors, parsing errors, SSH failures

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                          | Input Node(s)          | Output Node(s)              | Sticky Note                                                                                                                     |
|---------------------|----------------------------------|----------------------------------------|-----------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Webhook             | HTTP Webhook                     | Receive external heartbeat POSTs       |                       | Switch1                     | Incoming Webhook: use a webhook from tools like Uptime Kuma                                                                     |
| Switch1             | Switch                          | Route based on heartbeat message status| Webhook               | OK Message, ERROR Message    |                                                                                                                                |
| OK Message          | Telegram                        | Send OK heartbeat message               | Switch1               |                             |                                                                                                                                |
| ERROR Message       | Telegram                        | Send error heartbeat message with buttons| Switch1             |                             |                                                                                                                                |
| Telegram Trigger    | Telegram Trigger                | Listen for Telegram user commands       |                       | Switch, Extract the Service Name | User Interaction: Allow users to triggers events directly from Telegram                                                      |
| Switch              | Switch                         | Route Telegram commands ("logs", "restart", "status", "update")| Telegram Trigger | Merge, Merge1, docker ps, Update Msg |                                                                                                                                |
| Extract the Service Name | Code (Python)               | Extract Docker service name from message| Telegram Trigger      | Merge, Merge1                |                                                                                                                                |
| Merge               | Merge                          | Combine service name with logs command  | Extract the Service Name, Switch | get logs                   |                                                                                                                                |
| Merge1              | Merge                          | Combine service name with restart command| Extract the Service Name, Switch | Restart Message             | Docker Restart Service: Automatically restart a given docker container                                                        |
| get logs            | SSH Command                    | Retrieve last 100 lines of container logs| Merge                  | Status Update                |                                                                                                                                |
| Restart Message     | Telegram                       | Inform user about restart attempt       | Merge1                 | restart container            |                                                                                                                                |
| restart container   | SSH Command                    | Restart specified Docker container      | Restart Message        | If                          |                                                                                                                                |
| If                  | If                            | Check if restart succeeded (stderr empty)| restart container      | Success restart, Restart Failed |                                                                                                                                |
| Success restart     | Telegram                       | Notify user of successful restart       | If                     |                             |                                                                                                                                |
| Restart Failed      | Telegram                       | Notify user of failed restart            | If                     |                             |                                                                                                                                |
| Status Update       | Telegram                       | Notify user that log analysis is ongoing| get logs                | Message a model              |                                                                                                                                |
| Message a model     | OpenAI (LangChain)              | Analyze logs with AI and provide feedback| Status Update           | Log Analysis                 | Issue Analyzer: Automatically analyse the log file for a docker container and provide feedback to the user                   |
| Log Analysis        | Telegram                       | Send AI log analysis result to user     | Message a model         |                             |                                                                                                                                |
| docker ps           | SSH Command                   | List all running Docker containers       | Switch                  | Docker Status                | Allow the user to get all current deployed docker container                                                                    |
| Docker Status       | Telegram                      | Send Docker container status to user    | docker ps               |                             |                                                                                                                                |
| Update Msg          | Telegram                      | Notify user update process started       | Switch                  | Update Docker                | Automatically update all docker images on the server                                                                           |
| Update Docker       | SSH Command                   | Run custom update script                  | Update Msg              | Code in Python (Beta)        | Custom Script for your Docker server (see note content)                                                                        |
| Code in Python (Beta) | Code (Python)                | Parse update script output and format summary| Update Docker           | Update Msg1                  |                                                                                                                                |
| Update Msg1         | Telegram                      | Send update summary message to user      | Code in Python (Beta)   |                             |                                                                                                                                |
| Sticky Note         | Sticky Note                   | Documentation note                        |                       |                             | Incoming Webhook: use a webhook from tools like Uptime Kuma                                                                    |
| Sticky Note1        | Sticky Note                   | Documentation note                        |                       |                             | User Interaction: Allow users to triggers events directly from Telegram                                                        |
| Sticky Note2        | Sticky Note                   | Documentation note                        |                       |                             | Issue Analyzer: Automatically analyse the log file for a docker container and provide feedback to the user                   |
| Sticky Note3        | Sticky Note                   | Documentation note                        |                       |                             | Docker Restart Service: Automatically restart a given docker container                                                        |
| Sticky Note4        | Sticky Note                   | Documentation note                        |                       |                             | Allow the user to get all current deployed docker container                                                                    |
| Sticky Note5        | Sticky Note                   | Documentation note                        |                       |                             | Automatically update all docker images on the server                                                                           |
| Sticky Note6        | Sticky Note                   | Documentation note                        |                       |                             | Custom Script for your Docker server (detailed bash script included in notes)                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: HTTP Webhook
   - Method: POST
   - Path: `c851e840-457c-4050-8803-4e6d258580e1`
   - Purpose: Receive heartbeat/status POSTs from external monitoring tools
   - Connect output to Switch1

2. **Create Switch1 Node**
   - Type: Switch (v3 or higher)
   - Condition: Check if `{{$json.body.heartbeat.msg}}` equals "running"
   - Output 1: If "running", connect to OK Message
   - Output 2: Otherwise, connect to ERROR Message

3. **Create OK Message Node**
   - Type: Telegram
   - Chat ID: `8209283663`
   - Text: `{{$json.body.msg}}`
   - Connect no further

4. **Create ERROR Message Node**
   - Type: Telegram
   - Chat ID: `8209283663`
   - Text: `{{$json.body.monitor.docker_container}} reported an issue : {{$json.body.heartbeat.msg}} at {{$json.body.heartbeat.time}}`
   - Reply Markup: Custom keyboard with buttons `<container_name> Logs` and `<container_name> Restart`
   - Connect no further

5. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Updates: "message"
   - Connect output to Switch and Extract the Service Name nodes in parallel

6. **Create Switch Node**
   - Type: Switch (v3 or higher)
   - Rules:
     - Contains "logs" → output 1 (to Merge)
     - Contains "restart" → output 2 (to Merge1)
     - Equals "status" → output 3 (to docker ps)
     - Equals "update" → output 4 (to Update Msg)
   - Connect outputs accordingly

7. **Create Extract the Service Name Node**
   - Type: Code (Python)
   - Code:
   ```python
   def extract_service_name(message):
       parts = message.split()
       return parts[0] if parts else ''

   incoming_message = _input.first().json.message.text
   service_name = extract_service_name(incoming_message)
   return {"service_name": service_name}
   ```
   - Output: `service_name`
   - Connect output to Merge and Merge1 nodes

8. **Create Merge Node**
   - Combine mode: combineByPosition
   - Inputs: Extract the Service Name + Switch output for "logs"
   - Connect output to get logs node

9. **Create Merge1 Node**
   - Combine mode: combineByPosition
   - Inputs: Extract the Service Name + Switch output for "restart"
   - Connect output to Restart Message node

10. **Create get logs Node**
    - Type: SSH Command
    - Command: `docker logs --tail 100 {{$json.service_name}}`
    - Connect output to Status Update

11. **Create Restart Message Node**
    - Type: Telegram Message
    - Text: "Attempting to restart..."
    - Chat ID: `8209283663`
    - Connect output to restart container node

12. **Create restart container Node**
    - SSH Command
    - Command: `docker restart {{$json.service_name}}`
    - Connect output to If node

13. **Create If Node**
    - Condition: Check if `stderr` is empty (restart success)
    - True output: Success restart node
    - False output: Restart Failed node

14. **Create Success restart Node**
    - Telegram Message
    - Text: "Successful restarting {{$json.stdout}}"
    - Chat ID: `8209283663`

15. **Create Restart Failed Node**
    - Telegram Message
    - Text: "Restart failed\n{{ $('restart container').item.json.stderr }}"
    - Chat ID: `8209283663`

16. **Create Status Update Node**
    - Telegram Message
    - Text: "Analyzing Log File..."
    - Chat ID: `8209283663`
    - Connect output to Message a model node

17. **Create Message a model Node**
    - Type: OpenAI (LangChain)
    - Model: GPT-4.1-mini
    - System Prompt: Provide structured log analysis as per preset instructions
    - Input: Docker logs from get logs node (stdout)
    - Connect output to Log Analysis node

18. **Create Log Analysis Node**
    - Telegram Message
    - Text: `{{$json.output[0].content[0].text}}`
    - Chat ID: `8209283663`

19. **Create docker ps Node**
    - SSH Command
    - Command: `docker ps --format "{{.Names}}\t{{.Status}}"`
    - Connect output to Docker Status node

20. **Create Docker Status Node**
    - Telegram Message
    - Text: `{{$json.stdout}}`
    - Chat ID: `8209283663`
    - Use HTML parse mode to format as needed

21. **Create Update Msg Node**
    - Telegram Message
    - Text: "Running Update..."
    - Chat ID: `8209283663`
    - Connect output to Update Docker node

22. **Create Update Docker Node**
    - SSH Command
    - Command: `./update-all-docker-compose.sh`
    - Configure to always output data
    - Connect output to Code in Python (Beta) node

23. **Create Code in Python (Beta) Node**
    - Python code to parse update summary JSON from script output and format message:
    ```python
    import re
    import json

    def extract_update_summary(stdout):
        match = re.search(r'Update Summary:\\n({.*?\\})', stdout, re.DOTALL)
        if not match:
            return {}
        update_json = match.group(1)
        update_json_clean = re.sub(r',\\s*([\\]\\}])', r'\\1', update_json)
        return json.loads(update_json_clean)

    def format_update_message(update_summary):
        status_parts = []
        for compose_file, updates in update_summary.items():
            service = compose_file.replace('-compose.yaml', '')
            if updates == ["none"]:
                status_parts.append(f"{service}: No updates")
            else:
                status_parts.append(f"{service}: Updated ({', '.join(updates)})")
        return "; ".join(status_parts)

    stdout = _input.first().json.stdout
    update_summary = extract_update_summary(stdout)
    message = format_update_message(update_summary)
    return [{"message": message}]
    ```
    - Connect output to Update Msg1 node

24. **Create Update Msg1 Node**
    - Telegram Message
    - Text: `{{$json.message}}`
    - Chat ID: `8209283663`

25. **Set up Credentials**
    - SSH credentials for Docker host must be configured and attached to all SSH nodes (`get logs`, `restart container`, `docker ps`, `Update Docker`)
    - Telegram Bot API credentials must be configured and attached to all Telegram nodes
    - OpenAI API credentials must be configured for the OpenAI node

26. **Add Sticky Notes**
    - For documentation and clarity, add sticky notes with the provided content near relevant nodes:
      - Incoming Webhook note near Webhook node
      - User Interaction note near Telegram Trigger node
      - Issue Analyzer note near Message a model node
      - Docker Restart Service note near Merge1/Restart Message nodes
      - Docker Container Listing note near docker ps/Docker Status nodes
      - Docker Update Automation note near Update Msg/Update Docker nodes
      - Custom Script note near Update Docker node with the shell script content

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                        | Context or Link                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| Incoming Webhook: use a webhook from tools like Uptime Kuma                                                                                                                                                                                                                         | Linked to Webhook node               |
| User Interaction: Allow users to triggers events directly from Telegram                                                                                                                                                                                                             | Telegram Trigger node context       |
| Issue Analyzer: Automatically analyse the log file for a docker container and provide feedback to the user                                                                                                                                                                         | Message a model node context         |
| Docker Restart Service: Automatically restart a given docker container                                                                                                                                                                                                              | Restart related nodes                |
| Allow the user to get all current deployed docker container                                                                                                                                                                                                                        | docker ps and Docker Status nodes    |
| Automatically update all docker images on the server                                                                                                                                                                                                                               | Update Docker related nodes          |
| Custom Script for your Docker server: The provided bash script iterates over compose files excluding the n8n instance, pulls updates, restarts services, and outputs a JSON summary of updated containers and images for parsing and reporting.                                     | Update Docker node and sticky note   |

---

This comprehensive reference allows understanding, reproduction, and modification of the entire workflow with attention to integration points, error handling, and functional logic.