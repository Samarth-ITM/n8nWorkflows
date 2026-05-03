Automated Server Health Check with SSH, Azure GPT-4 Analysis & Email Alerts

https://n8nworkflows.xyz/workflows/automated-server-health-check-with-ssh--azure-gpt-4-analysis---email-alerts-9852


# Automated Server Health Check with SSH, Azure GPT-4 Analysis & Email Alerts

### 1. Workflow Overview

This workflow automates server health monitoring by remotely executing system diagnostic commands over SSH, analyzing the raw output using an AI agent powered by Azure GPT-4, and then sending an email alert with a detailed, structured health report. It is designed for DevOps teams or system administrators who want a lightweight, automated server health check without installing dedicated monitoring software.

The workflow logically divides into these blocks:

- **1.1 Schedule Trigger**: Initiates the health check at regular intervals.
- **1.2 SSH Command Execution**: Runs a comprehensive set of shell commands on the target Linux server to collect system metrics.
- **1.3 AI Analysis and Structuring**: Uses a LangChain AI agent with Azure OpenAI GPT-4 to interpret the raw command output, generate a structured JSON report containing summaries, recommendations, and status, and parse the output to ensure strict formatting.
- **1.4 Email Notification**: Sends the generated report via email, formatted in HTML and including key health indicators and recommendations.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview**:  
  Triggers the entire workflow at a predefined interval (default every 5 minutes), enabling automated, periodic health checks.

- **Nodes Involved**:  
  - Schedule Trigger

- **Node Details**:  
  - **Name**: Schedule Trigger  
  - **Type**: `n8n-nodes-base.scheduleTrigger`  
  - **Configuration**: Uses an interval-based trigger with a default setting to execute periodically (default 5 minutes, configurable).  
  - **Inputs**: None (starting point)  
  - **Outputs**: Connected to "Execute a command"  
  - **Version Requirements**: n8n version supporting scheduleTrigger v1.2 or higher.  
  - **Edge Cases / Failures**:  
    - Misconfiguration of interval can cause unwanted frequency.  
    - Workflow may queue executions if prior runs take longer than interval.  
  - **Sticky Note**:  
    > **Schedule Trigger:**  
    > *“Triggers the workflow at your chosen interval to run server health checks automatically.”*

#### 2.2 SSH Command Execution

- **Overview**:  
  Connects to the Linux server using SSH to execute a series of commands that gather critical system health information: hostname, uptime, CPU, memory, disk usage, running services, network stats, and top CPU-consuming processes.

- **Nodes Involved**:  
  - Execute a command

- **Node Details**:  
  - **Name**: Execute a command  
  - **Type**: `n8n-nodes-base.ssh`  
  - **Configuration**:  
    - Runs a multiline shell script that outputs labeled sections for:  
      - System info (`hostname`, `uptime`)  
      - CPU usage (`top -bn1 | grep "Cpu(s)"`)  
      - Memory usage (`free -m`)  
      - Disk usage (`df -h`)  
      - Running services (`systemctl list-units --type=service --state=running`)  
      - Network interface stats (`ip -s link`)  
      - Top CPU-consuming processes (`ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 10`)  
    - Authentication uses private key SSH credentials for secure connection.  
  - **Inputs**: Triggered by Schedule Trigger  
  - **Outputs**: Raw command output passed to AI Agent  
  - **Version Requirements**: Requires SSH node v1 or higher with private key support configured.  
  - **Edge Cases / Failures**:  
    - SSH connection failure due to network, credential issues, or host unavailability.  
    - Command execution errors or unexpected output format.  
    - Permission issues on the server.  
  - **Sticky Note**:  
    > **Execute Command:**  
    > *“Collects detailed server metrics: hostname, uptime, CPU, memory, disk, services, network stats, and top processes.”*

#### 2.3 AI Analysis and Structuring

- **Overview**:  
  Processes the raw SSH output with an AI agent using Azure GPT-4 via LangChain integration. The AI generates a structured JSON report including an executive summary, concise technical summary, detailed HTML report, overall system status, and prioritized recommendations. The output is then validated and parsed to ensure strict adherence to the expected JSON schema.

- **Nodes Involved**:  
  - AI Agent  
  - Azure OpenAI Chat Model  
  - Structured Output Parser

- **Node Details**:  

  - **Azure OpenAI Chat Model**  
    - **Type**: `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`  
    - **Role**: Provides GPT-4.1 model for AI processing  
    - **Configuration**:  
      - Model chosen: gpt-4.1  
      - Uses OAuth2 authentication via Azure Entra Cognitive Services OAuth2 API credentials stored in n8n  
    - **Inputs**: Feeds prompt and raw SSH output to AI Agent  
    - **Outputs**: Supplies language model interface to AI Agent and Structured Output Parser  
    - **Edge Cases / Failures**:  
      - API authentication failures  
      - Rate limiting or quota exceeded errors  
      - Model response latency or timeouts  

  - **AI Agent**  
    - **Type**: `@n8n/n8n-nodes-langchain.agent`  
    - **Role**: Orchestrates AI prompt execution and output retrieval  
    - **Configuration**:  
      - Receives raw SSH output as `{{ $json.stdout }}`  
      - Prompt explicitly instructs to produce JSON strictly matching the schema with fields: executiveSummary, summary, details (HTML), status (OK/WARNING/CRITICAL), and recommendations (with issue, priority, action).  
      - Requires the AI to analyze CPU, memory, disk, network usage, and top processes against defined thresholds for health status.  
    - **Inputs**: Receives raw SSH output from "Execute a command"  
    - **Outputs**: JSON report to "Send email" node  
    - **Connections**: Uses Azure OpenAI Chat Model as language model and Structured Output Parser as output parser  
    - **Edge Cases / Failures**:  
      - AI output not matching schema (mitigated by Structured Output Parser with autoFix)  
      - Unexpected raw input format impacting AI understanding  

  - **Structured Output Parser**  
    - **Type**: `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - **Role**: Validates and auto-fixes AI agent output to strictly conform to the JSON schema, preventing malformed results.  
    - **Configuration**:  
      - Manual JSON schema defined with required properties and types  
      - Auto-fix enabled to correct minor AI output deviations  
    - **Inputs**: Receives AI agent raw output  
    - **Outputs**: Passes structured, validated JSON back to AI Agent node for downstream use  
    - **Edge Cases / Failures**:  
      - AI output too malformed to auto-fix (would cause workflow error)  

- **Sticky Notes**:  
  - AI Agent:  
    > **AI Agent:**  
    > *“Analyzes the collected server data for anomalies and prepares structured results.”*  
  - Structured Output Parser:  
    > **Structured Output Parser:**  
    > *“Ensures AI output is formatted correctly for the email alert.”*

#### 2.4 Email Notification

- **Overview**:  
  Sends the structured server health report as an HTML email to predefined recipients. The email subject reflects the current health status (OK/WARNING/CRITICAL), and the body contains the detailed HTML report generated by the AI.

- **Nodes Involved**:  
  - Send email

- **Node Details**:  
  - **Name**: Send email  
  - **Type**: `n8n-nodes-base.emailSend`  
  - **Configuration**:  
    - Email subject dynamically set to: `Health Report - Server | {{ $json.output.status }}`  
    - HTML body uses `{{ $json.output.details }}` from AI output for rich content with tables, highlights, and graphs  
    - SMTP credentials configured securely in n8n under "SMTP account 2"  
  - **Inputs**: Receives structured AI report from AI Agent  
  - **Outputs**: None (end node)  
  - **Version Requirements**: Supports v2.1 or higher for emailSend node  
  - **Edge Cases / Failures**:  
    - SMTP authentication or connectivity failures  
    - Email formatting issues if AI output malformed  
  - **Sticky Note**:  
    > **Send Email:**  
    > *“Sends an alert email summarizing the server health or any detected issues.”*

---

### 3. Summary Table

| Node Name              | Node Type                                | Functional Role                        | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                     |
|------------------------|-----------------------------------------|-------------------------------------|----------------------|----------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger       | n8n-nodes-base.scheduleTrigger           | Initiates workflow at set intervals | None                 | Execute a command     | **Schedule Trigger:** *“Triggers the workflow at your chosen interval to run server health checks automatically.”* |
| Execute a command      | n8n-nodes-base.ssh                       | Runs server diagnostic commands via SSH | Schedule Trigger     | AI Agent             | **Execute Command:** *“Collects detailed server metrics: hostname, uptime, CPU, memory, disk, services, network stats, and top processes.”* |
| Azure OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Provides GPT-4 model for AI processing | AI Agent (ai_languageModel) | AI Agent, Structured Output Parser |                                                                                                |
| AI Agent              | @n8n/n8n-nodes-langchain.agent            | Analyzes SSH output, generates structured report | Execute a command, Azure OpenAI Chat Model, Structured Output Parser | Send email           | **AI Agent:** *“Analyzes the collected server data for anomalies and prepares structured results.”* |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Validates and formats AI output JSON | AI Agent (ai_outputParser) | AI Agent             | **Structured Output Parser:** *“Ensures AI output is formatted correctly for the email alert.”* |
| Send email             | n8n-nodes-base.emailSend                  | Sends structured health report email | AI Agent             | None                 | **Send Email:** *“Sends an alert email summarizing the server health or any detected issues.”* |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node**  
   - Node Type: `Schedule Trigger`  
   - Set interval to desired frequency (default every 5 minutes).  
   - No inputs.  
   - Connect output to the SSH command node.

2. **Create the SSH Command Node**  
   - Node Type: `SSH`  
   - Set authentication to use a private key credential configured in n8n for your server.  
   - In the command field, paste the multiline shell script:  
     ```
     echo "---SYSTEM INFO---"; hostname; uptime;
     echo "---CPU---"; top -bn1 | grep "Cpu(s)";
     echo "---MEMORY---"; free -m;
     echo "---DISK---"; df -h;
     echo "---SERVICES---"; systemctl list-units --type=service --state=running;
     echo "---NETWORK---"; ip -s link;
     echo "---TOP PROCESSES---"; ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 10
     ```
   - Connect output to the AI Agent node.

3. **Create the Azure OpenAI Chat Model Node**  
   - Node Type: `LangChain Azure OpenAI Chat Model`  
   - Select model: `gpt-4.1`  
   - Configure OAuth2 credentials for Azure Entra Cognitive Services in n8n.  
   - Connect the node as a language model resource to the AI Agent.

4. **Create the AI Agent Node**  
   - Node Type: `LangChain Agent`  
   - Copy the prompt configuration to instruct the AI to:  
     - Receive raw SSH output (`{{ $json.stdout }}`)  
     - Generate structured JSON with fields: executiveSummary, summary, details (HTML), status, recommendations  
     - Follow strict schema with defined thresholds for system health.  
   - Set the language model to the previously created Azure OpenAI Chat Model node.  
   - Set the output parser to the Structured Output Parser node to validate AI output.  
   - Connect input from SSH Command node.  
   - Connect output to Send email node.

5. **Create the Structured Output Parser Node**  
   - Node Type: `LangChain Structured Output Parser`  
   - Define manual JSON schema as:  
     ```json
     {
       "type": "object",
       "properties": {
         "executiveSummary": { "type": "string" },
         "summary": { "type": "string" },
         "details": { "type": "string" },
         "status": { "type": "string", "enum": ["OK", "WARNING", "CRITICAL"] },
         "recommendations": {
           "type": "array",
           "items": {
             "type": "object",
             "properties": {
               "issue": { "type": "string" },
               "priority": { "type": "string", "enum": ["HIGH", "MEDIUM", "LOW"] },
               "action": { "type": "string" }
             },
             "required": ["issue", "priority", "action"]
           }
         }
       },
       "required": ["executiveSummary", "summary", "details", "status", "recommendations"]
     }
     ```  
   - Enable auto-fix option for minor corrections.  
   - Connect input from AI Agent node and output back to AI Agent for downstream usage.

6. **Create the Send Email Node**  
   - Node Type: `Email Send`  
   - Configure SMTP credentials securely in n8n.  
   - Set email subject to:  
     ```
     Health Report - Server | {{ $json.output.status }}
     ```  
   - Set email body HTML content to:  
     ```
     {{ $json.output.details }}
     ```  
   - Connect input from AI Agent node.

7. **Verify connections**:  
   - Schedule Trigger → Execute a command → AI Agent (uses Azure OpenAI Chat Model and Structured Output Parser) → Send email.

8. **Credentials Setup**:  
   - SSH private key configured for the target Linux server.  
   - Azure OpenAI API credentials configured with OAuth2 for Azure Entra Cognitive Services.  
   - SMTP credentials for sending email notifications.

9. **Optional**: Add Sticky Note nodes to document each part of the workflow visually within n8n for easier maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is ideal for automated, agent-driven server health checks without installing heavy monitoring software. It supports self-hosted n8n instances and can be extended to monitor multiple servers by duplicating or adding command nodes.                                                                                                                     | See sticky note "Automated Server Health Check with AI & Email Alerts" in the workflow UI.       |
| Ensure all credentials (SSH keys, Azure OpenAI, SMTP) are securely stored in n8n credentials and never hardcoded in workflows.                                                                                                                                                                                                                                       | Security best practice.                                                                           |
| The workflow uses Azure GPT-4 via LangChain integration, requiring a valid Azure subscription and API configuration with cognitive services.                                                                                                                                                                                                                         | Azure OpenAI documentation: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/  |
| The generated HTML report includes inline styles, colored highlights, simple graphs, and tables to improve readability for email recipients.                                                                                                                                                                                                                        | Customizable in AI prompt if different styling or metrics are desired.                           |
| For scalability, consider adding error handling nodes or retry mechanisms around SSH or email sending steps to handle transient network or service errors gracefully.                                                                                                                                                                                                 | Recommended for production-grade workflows.                                                     |

---

**Disclaimer:** The source text originates exclusively from an automated workflow created with n8n, respecting all applicable content policies and containing no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.