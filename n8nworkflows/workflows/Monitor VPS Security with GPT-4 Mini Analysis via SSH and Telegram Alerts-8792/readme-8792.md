Monitor VPS Security with GPT-4 Mini Analysis via SSH and Telegram Alerts

https://n8nworkflows.xyz/workflows/monitor-vps-security-with-gpt-4-mini-analysis-via-ssh-and-telegram-alerts-8792


# Monitor VPS Security with GPT-4 Mini Analysis via SSH and Telegram Alerts

### 1. Workflow Overview

This workflow automates VPS security monitoring by periodically collecting system process and network data via SSH, analyzing it using an AI-powered security analysis (OpenAI GPT-4 Mini), and sending Telegram alerts if malicious or suspicious activities are detected. It targets system administrators or security teams managing Linux VPS instances who want proactive, AI-enhanced threat detection without manual log inspection.

The workflow’s logic is organized into these main blocks:

- **1.1 Scheduled Trigger & Configuration Setup**: Periodically triggers the workflow and sets user-specific parameters like Telegram chat ID and server name.

- **1.2 Data Collection via SSH**: Connects over SSH to the VPS, runs commands to gather process and network connection details, and stores this data.

- **1.3 AI Security Analysis**: Uses OpenAI GPT-4 Mini to analyze the collected data for suspicious or malicious activity, producing structured output.

- **1.4 Parsing AI Output**: Parses the structured AI response into explicit fields (malicious, suspicious, explanations, status).

- **1.5 Conditional Alerting**: Evaluates AI results to determine if alerts should be sent for malicious or suspicious activity.

- **1.6 Telegram Notifications**: Sends formatted Telegram messages to the administrator for detected threats or concerns.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration Setup

- **Overview:**  
  Initiates the workflow on a six-hour interval and establishes configuration variables such as Telegram chat ID and server name.

- **Nodes Involved:**  
  - Schedule Trigger - Every 6 Hours  
  - Configuration - User Settings

- **Node Details:**  

  - **Schedule Trigger - Every 6 Hours**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution every 6 hours.  
    - Configuration: Interval set to 6 hours, no specific start time.  
    - Connections: Outputs to Configuration - User Settings.  
    - Edge Cases: Trigger failures unlikely; possible system downtime or scheduler misconfiguration might delay runs.

  - **Configuration - User Settings**  
    - Type: Set  
    - Role: Defines key user parameters such as admin Telegram chat ID, server name, and alert level.  
    - Configuration:  
      - `admin_telegram_id`: Placeholder "YOUR_TELEGRAM_CHAT_ID" (must be replaced).  
      - `server_name`: Default "Production VPS".  
      - `alert_level`: Set to "high" (not used further in this workflow but available for customization).  
    - Connections: Feeds into SSH Data Collection node.  
    - Edge Cases: Missing or incorrect Telegram chat ID will cause alert delivery failures.

#### 2.2 Data Collection via SSH

- **Overview:**  
  Executes system commands via SSH to collect current processes sorted by CPU and memory usage plus active TCP/UDP listening sockets, storing results for AI analysis.

- **Nodes Involved:**  
  - SSH - Gather Process and Network Data

- **Node Details:**  

  - **SSH - Gather Process and Network Data**  
    - Type: SSH  
    - Role: Executes remote commands on the VPS.  
    - Configuration:  
      - Working Directory: `/root`  
      - Commands:  
        - `ps aux --sort=-%cpu,-%mem` (list processes sorted by CPU and memory)  
        - `ss -tulpn > /vps_process_report.txt` (dump network connections to a file)  
      - Output: The standard output from `ps aux` is used for AI input.  
    - Credentials: SSH password-based authentication (configured with credential ID "SSH_My_VPS").  
    - Connections: Outputs to AI Security Analysis node.  
    - Edge Cases: SSH connection failure (network issues, credential errors), command execution errors, permission issues writing to `/vps_process_report.txt`.

#### 2.3 AI Security Analysis

- **Overview:**  
  Sends the collected process and network data to an OpenAI GPT-4 Mini model, instructing it to identify malicious or suspicious activities with detailed reasoning, and expects structured JSON output.

- **Nodes Involved:**  
  - OpenAI GPT-4 Mini Model  
  - AI Security Analysis  
  - Parse Security Analysis Results

- **Node Details:**  

  - **OpenAI GPT-4 Mini Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Performs the AI language model inference.  
    - Configuration:  
      - Model: `gpt-4o-mini` (a smaller GPT-4 variant)  
      - Temperature: 0.1 (low randomness for deterministic output)  
    - Credentials: OpenAI API credentials configured (credential ID "OpenAi").  
    - Connections: Outputs to AI Security Analysis node.  
    - Edge Cases: API quota limits, network timeouts, invalid API key, model unavailability.

  - **AI Security Analysis**  
    - Type: LangChain Chain LLM  
    - Role: Defines the prompt and handles AI interaction logic.  
    - Configuration:  
      - Prompt: Instructs AI to:  
        1. Identify suspicious/malicious processes and connections.  
        2. Explain reasons (malware patterns, unusual connections, strange names, resource anomalies).  
        3. Focus on cryptocurrency miners, botnets, unauthorized services, suspicious shells, abnormal resource use.  
        4. Provide structured output separating malicious and suspicious findings.  
      - Uses expression `{{ $json.stdout }}` to inject SSH command output (process list) dynamically.  
    - Connections: Main output branches to two conditional nodes checking for malicious and suspicious activity; also connects to output parser.  
    - Edge Cases: Expression failure if SSH output missing, malformed prompt causing AI errors, slow response, or incomplete data.

  - **Parse Security Analysis Results**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses AI’s structured JSON result into explicit fields for downstream processing.  
    - Configuration:  
      - Schema defines properties:  
        - `malicious` (string)  
        - `malicious_explain` (string)  
        - `suspicious` (string)  
        - `suspicious_explain` (string)  
        - `status` (string; e.g., clean, suspicious, compromised)  
    - Connections: Output connected back into AI Security Analysis node as parsed output.  
    - Edge Cases: Parsing failures if AI output deviates from schema.

#### 2.4 Conditional Alerting

- **Overview:**  
  Determines if alerts should be sent based on presence of malicious or suspicious findings in the AI analysis output.

- **Nodes Involved:**  
  - Check for Malicious Activity  
  - Check for Suspicious Activity

- **Node Details:**  

  - **Check for Malicious Activity**  
    - Type: If  
    - Role: Checks if the `malicious` field from AI output is non-empty.  
    - Condition: `malicious` string is not empty.  
    - Connections: If true, triggers Send Malicious Activity Alert node.  
    - Edge Cases: False negatives if AI misses malicious activity; false positives if output formatting inconsistent.

  - **Check for Suspicious Activity**  
    - Type: If  
    - Role: Checks if the `suspicious` field from AI output is non-empty.  
    - Condition: `suspicious` string is not empty.  
    - Connections: If true, triggers Send Suspicious Activity Notice node.  
    - Edge Cases: Same as above; also potential overlap with malicious findings.

#### 2.5 Telegram Notifications

- **Overview:**  
  Sends formatted Telegram messages to the configured administrator based on detected threat level.

- **Nodes Involved:**  
  - Send Malicious Activity Alert  
  - Send Suspicious Activity Notice

- **Node Details:**  

  - **Send Malicious Activity Alert**  
    - Type: Telegram  
    - Role: Sends a high-priority alert for confirmed malicious activity.  
    - Configuration:  
      - Text message uses Markdown formatting with details: server name, timestamp, malicious processes, explanations, overall status, and action recommendation.  
      - Chat ID sourced from Configuration - User Settings node.  
      - Parse mode set to Markdown for message formatting.  
    - Credentials: Telegram bot API credentials (configured bot "IrEventsBot").  
    - Edge Cases: Telegram API failures, invalid chat ID, message formatting errors.

  - **Send Suspicious Activity Notice**  
    - Type: Telegram  
    - Role: Sends a lower-priority notification about suspicious but unconfirmed activity.  
    - Configuration:  
      - Similar to malicious alert but with different text emphasizing monitoring and investigation recommendation.  
      - Parses Markdown, targets same chat ID.  
    - Credentials: Same Telegram bot credentials.  
    - Edge Cases: Same as above.

---

### 3. Summary Table

| Node Name                            | Node Type                         | Functional Role                         | Input Node(s)                     | Output Node(s)                                    | Sticky Note                                                                                                            |
|------------------------------------|----------------------------------|---------------------------------------|----------------------------------|--------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger - Every 6 Hours    | Schedule Trigger                 | Periodically starts workflow          | -                                | Configuration - User Settings                     |                                                                                                                        |
| Configuration - User Settings       | Set                             | Defines user/server parameters        | Schedule Trigger - Every 6 Hours | SSH - Gather Process and Network Data             |                                                                                                                        |
| SSH - Gather Process and Network Data | SSH                             | Collects process and network info     | Configuration - User Settings    | AI Security Analysis                              | "### Step 1: Data Collection\n\nSSH into VPS and gather:\n- Running processes (sorted by CPU/memory)\n- Active network connections\n- System information" |
| OpenAI GPT-4 Mini Model             | LangChain LM Chat OpenAI         | Runs GPT-4 Mini AI model               | SSH - Gather Process and Network Data | AI Security Analysis                              | "### Step 2: AI Analysis\n\nOpenAI analyzes data for:\n- Known malware patterns\n- Suspicious network activity\n- Unusual resource usage\n- Botnet indicators" |
| AI Security Analysis                | LangChain Chain LLM              | AI prompt orchestration and output    | SSH - Gather Process and Network Data, OpenAI GPT-4 Mini Model | Check for Malicious Activity, Check for Suspicious Activity, Parse Security Analysis Results |                                                                                                                        |
| Parse Security Analysis Results     | LangChain Output Parser Structured | Parses structured AI output            | AI Security Analysis             | AI Security Analysis (parsed output)              |                                                                                                                        |
| Check for Malicious Activity        | If                              | Detects confirmed malicious findings  | AI Security Analysis             | Send Malicious Activity Alert                      |                                                                                                                        |
| Check for Suspicious Activity       | If                              | Detects suspicious findings           | AI Security Analysis             | Send Suspicious Activity Notice                    |                                                                                                                        |
| Send Malicious Activity Alert       | Telegram                        | Sends Telegram alert for malicious activity | Check for Malicious Activity    | -                                                | "### Step 3: Smart Alerting\n\nSeparate alerts for:\n- 🚨 Malicious: Confirmed threats\n- ⚠️ Suspicious: Needs investigation\n\nNo spam - only real threats!" |
| Send Suspicious Activity Notice     | Telegram                        | Sends Telegram notice for suspicious activity | Check for Suspicious Activity  | -                                                | "### Step 3: Smart Alerting\n\nSeparate alerts for:\n- 🚨 Malicious: Confirmed threats\n- ⚠️ Suspicious: Needs investigation\n\nNo spam - only real threats!" |
| Sticky Note - Main Explanation      | Sticky Note                    | Describes overall workflow purpose    | -                                | -                                                | "## 🔐 VPS Security Monitor with AI Analysis\n\nThis workflow automatically monitors your VPS for security threats using AI analysis and sends alerts via Telegram.\n\n### 📋 How it works:\n1. **Scheduled Monitoring**: Runs every 6 hours (customizable)\n2. **SSH Data Collection**: Gathers process and network information\n3. **AI Security Analysis**: Uses OpenAI GPT-4 Mini to identify threats\n4. **Smart Alerting**: Only sends notifications for actual threats\n\n### ⚙️ Configuration Required:\n- Update SSH credentials in \"SSH - Gather Process and Network Data\" node\n- Add OpenAI API key in \"OpenAI GPT-4 Mini Model\" node  \n- Set your Telegram chat ID in \"Configuration - User Settings\"\n- Add Telegram bot token in alert nodes\n\n### 🎯 Features:\n- Detects malware, cryptocurrency miners, botnet activity\n- Monitors unusual network connections and resource usage\n- Structured AI analysis with clear explanations\n- Separate alerts for malicious vs suspicious activity\n\n### 💡 Customization:\n- Adjust monitoring frequency in Schedule Trigger\n- Modify AI prompt for specific security concerns\n- Add multiple servers by duplicating SSH nodes\n- Extend with email/Slack notifications" |
| Sticky Note - Step 1                | Sticky Note                    | Explains Step 1: Data Collection      | -                                | -                                                | "### Step 1: Data Collection\n\nSSH into VPS and gather:\n- Running processes (sorted by CPU/memory)\n- Active network connections\n- System information"       |
| Sticky Note - Step 2                | Sticky Note                    | Explains Step 2: AI Analysis           | -                                | -                                                | "### Step 2: AI Analysis\n\nOpenAI analyzes data for:\n- Known malware patterns\n- Suspicious network activity\n- Unusual resource usage\n- Botnet indicators"  |
| Sticky Note - Step 3                | Sticky Note                    | Explains Step 3: Alerting              | -                                | -                                                | "### Step 3: Smart Alerting\n\nSeparate alerts for:\n- 🚨 Malicious: Confirmed threats\n- ⚠️ Suspicious: Needs investigation\n\nNo spam - only real threats!"     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger - Every 6 Hours" node**  
   - Type: Schedule Trigger  
   - Set interval to every 6 hours (field: hoursInterval=6).  
   - No additional credentials required.

2. **Create "Configuration - User Settings" node**  
   - Type: Set  
   - Add three string fields:  
     - `admin_telegram_id` with your Telegram chat ID (replace `"YOUR_TELEGRAM_CHAT_ID"`).  
     - `server_name` with your VPS name (default: `"Production VPS"`).  
     - `alert_level` set to `"high"` (optional, for future use).  
   - Connect from the Schedule Trigger node.

3. **Create "SSH - Gather Process and Network Data" node**  
   - Type: SSH  
   - Set working directory to `/root`.  
   - Command:   
     ```
     ps aux --sort=-%cpu,-%mem && ss -tulpn > /vps_process_report.txt
     ```  
   - Configure SSH credentials (password or key) for your VPS.  
   - Connect from Configuration - User Settings node.

4. **Create "OpenAI GPT-4 Mini Model" node**  
   - Type: LangChain LM Chat OpenAI  
   - Set model to `gpt-4o-mini`.  
   - Set temperature to 0.1 for deterministic results.  
   - Add OpenAI API credentials with your API key.  
   - Connect from SSH node.

5. **Create "AI Security Analysis" node**  
   - Type: LangChain Chain LLM  
   - Configure prompt with text (use expression to inject SSH output):  
     ```
     You are a security analyst AI. I will provide you with a list of running processes and open network ports from a Linux VPS. Your task:

     1. Identify any processes, commands, or connections that appear suspicious, malicious, or unusual.
     2. Explain why you think they are suspicious (e.g., known malware patterns, unusual network connections, strange process names, or abnormal resource usage).
     3. Focus on: cryptocurrency miners, botnet activity, unauthorized network services, suspicious shell processes, or processes with unusual resource consumption.
     4. Provide structured output with malicious and suspicious findings separately.

     Here is the process and network information:

     {{ $json.stdout }}
     ```  
   - Connect from SSH node as main input and from OpenAI GPT-4 Mini Model as the language model input.

6. **Create "Parse Security Analysis Results" node**  
   - Type: LangChain Output Parser Structured  
   - Define JSON schema with properties: `malicious`, `malicious_explain`, `suspicious`, `suspicious_explain`, and `status` as strings.  
   - Connect its output parser input to AI Security Analysis node and main output back to AI Security Analysis node.

7. **Create "Check for Malicious Activity" node**  
   - Type: If  
   - Condition: Check if `{{ $json.output.malicious }}` is not empty.  
   - Connect main output from AI Security Analysis node.

8. **Create "Check for Suspicious Activity" node**  
   - Type: If  
   - Condition: Check if `{{ $json.output.suspicious }}` is not empty.  
   - Connect main output from AI Security Analysis node.

9. **Create "Send Malicious Activity Alert" node**  
   - Type: Telegram  
   - Configure text with Markdown including:  
     - Server name: `{{ $('Configuration - User Settings').first().json.server_name }}`  
     - Time: `{{ new Date().toLocaleString() }}`  
     - Malicious processes and explanation from AI output.  
     - Overall status and action required message.  
   - Set chat ID to `{{ $('Configuration - User Settings').first().json.admin_telegram_id }}`.  
   - Use Telegram bot API credentials (your bot token).  
   - Connect from true output of Check for Malicious Activity node.

10. **Create "Send Suspicious Activity Notice" node**  
    - Type: Telegram  
    - Configure similar to malicious alert with text tailored for suspicious activity.  
    - Connect from true output of Check for Suspicious Activity node.

11. **Connect all nodes sequentially:**  
    Schedule Trigger → Configuration → SSH → OpenAI GPT-4 Mini → AI Security Analysis → Parse Results  
    AI Security Analysis → Check for Malicious Activity → Send Malicious Alert  
    AI Security Analysis → Check for Suspicious Activity → Send Suspicious Notice

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires you to replace placeholder values with your own credentials: SSH password/key, OpenAI API key, Telegram chat ID, and Telegram bot token.                                                                                                    | Configuration instructions within the workflow nodes                                               |
| The AI prompt is customizable to focus on specific threats or alter the level of detail in the analysis.                                                                                                                                                          | AI Security Analysis node prompt                                                                    |
| Telegram alerts utilize Markdown formatting for readability; ensure your Telegram bot has permission to message the specified chat ID.                                                                                                                           | Telegram nodes configuration                                                                        |
| To monitor multiple servers, duplicate the SSH node and adjust credentials and commands accordingly, then merge results for analysis.                                                                                                                             | Workflow customization note                                                                         |
| This workflow does not currently include error handling for SSH or API failures; consider adding error nodes or notifications for robustness.                                                                                                                     | General recommendation                                                                              |
| Workflow is based on n8n version supporting LangChain nodes v1.7+ and SSH node with password authentication.                                                                                                                                                        | Version-specific requirements                                                                       |
| More info on n8n LangChain integration: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                                                                                                                                                              | Official n8n documentation                                                                          |
| Telegram Bot API info: https://core.telegram.org/bots/api                                                                                                                                                                                                           | Telegram bot development and configuration                                                         |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, respecting all content policies. No illegal, offensive, or protected material is included. All data processed is legal and public.