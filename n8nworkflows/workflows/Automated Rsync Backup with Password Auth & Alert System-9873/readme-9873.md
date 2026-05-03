Automated Rsync Backup with Password Auth & Alert System

https://n8nworkflows.xyz/workflows/automated-rsync-backup-with-password-auth---alert-system-9873


# Automated Rsync Backup with Password Auth & Alert System

### 1. Workflow Overview

This workflow automates secure file backups between two remote servers using `rsync` over SSH with password authentication. It manages dependency installation (`sshpass` and `rsync`) on both the local machine running n8n and the source server before executing the backup. After the backup, it generates a detailed success or failure report and sends notifications through Telegram and SMS. The workflow supports manual and scheduled triggers and is designed for Linux environments including Ubuntu, Debian, RHEL, CentOS, and Alpine.

**Logical Blocks:**

- **1.1 Input Reception:** Accepts manual or scheduled triggers and sets server connection parameters.
- **1.2 Dependency Verification and Installation:** Ensures `sshpass` and `rsync` are installed locally and on the source server.
- **1.3 Backup Execution:** Runs the `rsync` command remotely via SSH, performing the backup from source to target server.
- **1.4 Status Handling:** Checks the backup command exit status and prepares a success or failure report.
- **1.5 Notification Dispatch:** Sends the backup result via Telegram and SMS alerts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block handles workflow initiation, either manually or via a schedule, and sets all necessary server and backup parameters.

- **Nodes Involved:**  
  - Manual Trigger  
  - Schedule Trigger  
  - Server Parameters  

- **Node Details:**  

  - **Manual Trigger**  
    - *Type:* Manual trigger node  
    - *Role:* Allows user to start the workflow on demand  
    - *Parameters:* None (default)  
    - *Connections:* Outputs to Server Parameters  
    - *Potential Failures:* None  

  - **Schedule Trigger**  
    - *Type:* Schedule trigger node  
    - *Role:* Automatically triggers the workflow at defined intervals (default: every minute)  
    - *Parameters:* Default interval with empty object, implying 1-minute interval if unchanged  
    - *Connections:* Outputs to Server Parameters  
    - *Potential Failures:* None  

  - **Server Parameters**  
    - *Type:* Set node  
    - *Role:* Defines all source and target server connection details and rsync options as string variables  
    - *Configuration:*  
      - `source_host`, `source_port`, `source_user`, `source_password`  
      - `source_folder`  
      - `target_host`, `target_port`, `target_user`, `target_password`  
      - `target_folder`  
      - `rsync_options` (default: `-avz --delete`)  
    - *Expressions:* Passwords are set as expressions expecting external injection or environment variables  
    - *Connections:* Outputs to Check Sshpass Local  
    - *Potential Failures:* Missing or incorrect parameters could cause downstream failures  

#### 1.2 Dependency Verification and Installation

- **Overview:**  
  This block verifies `sshpass` is installed locally and on the source server and installs it with `rsync` if missing. Supports multiple Linux package managers.

- **Nodes Involved:**  
  - Check Sshpass Local  
  - Is Installed Local?  
  - Install Sshpass Local  
  - Check Sshpass on Source  
  - Is Installed on Source?  
  - Install Sshpass on Source  

- **Node Details:**  

  - **Check Sshpass Local**  
    - *Type:* Execute Command  
    - *Role:* Runs `which sshpass` locally to check installation  
    - *Parameters:* Command: `which sshpass`  
    - *Connections:* Outputs to Is Installed Local?  
    - *Continue On Fail:* True (to handle not installed scenario)  
    - *Potential Failures:* Command may fail if environment differs; handled gracefully  

  - **Is Installed Local?**  
    - *Type:* If node  
    - *Role:* Checks if previous command exit code is 0 (sshpass installed)  
    - *Parameters:* Condition: exitCode == 0  
    - *Connections:*  
      - True → Check Sshpass on Source  
      - False → Install Sshpass Local  

  - **Install Sshpass Local**  
    - *Type:* Execute Command  
    - *Role:* Installs `sshpass` and `rsync` locally using appropriate package manager  
    - *Parameters:* Multi-branch shell script detecting package manager among apt-get, yum, dnf, apk  
    - *Connections:* Outputs to Check Sshpass on Source  
    - *Potential Failures:* Unsupported OS/package manager; installation errors  

  - **Check Sshpass on Source**  
    - *Type:* Execute Command  
    - *Role:* SSH into source server using password and check `sshpass` installation remotely  
    - *Parameters:* Command runs `which sshpass` on source server (password passed via sshpass)  
    - *Connections:* Outputs to Is Installed on Source?  
    - *Continue On Fail:* True  
    - *Potential Failures:* SSH connectivity issues, auth failures, network timeouts  

  - **Is Installed on Source?**  
    - *Type:* If node  
    - *Role:* Checks if sshpass is installed on source server (exitCode == 0)  
    - *Connections:*  
      - True → Execute Rsync Backup  
      - False → Install Sshpass on Source  

  - **Install Sshpass on Source**  
    - *Type:* Execute Command  
    - *Role:* Installs `sshpass` and `rsync` on source server via SSH with sudo, handling multiple package managers  
    - *Parameters:* Complex shell script with sudo and password injection to install dependencies  
    - *Connections:* Outputs to Execute Rsync Backup  
    - *Potential Failures:* Sudo permission issues, wrong source password, unsupported OS  

#### 1.3 Backup Execution

- **Overview:**  
  Executes the `rsync` backup command via SSH. The command connects first to the source server, which then initiates an `rsync` to the target server. Both connections use password authentication with `sshpass`.

- **Nodes Involved:**  
  - Execute Rsync Backup  

- **Node Details:**  

  - **Execute Rsync Backup**  
    - *Type:* Execute Command  
    - *Role:* Runs remote `rsync` command via `sshpass` for password-based authentication  
    - *Parameters:* Complex nested sshpass command chaining:  
      - SSH to source server with password  
      - Run sshpass rsync from source to target server over SSH with specified options  
      - `StrictHostKeyChecking=no` to avoid host key prompts  
    - *Connections:* Outputs to Success?  
    - *Potential Failures:* SSH connectivity, password errors, rsync errors, command syntax errors  

#### 1.4 Status Handling

- **Overview:**  
  Evaluates the exit code from the backup command, branches to success or failure handling, and prepares detailed status messages.

- **Nodes Involved:**  
  - Success?  
  - Backup Successful  
  - Backup Failed  

- **Node Details:**  

  - **Success?**  
    - *Type:* If node  
    - *Role:* Checks if `Execute Rsync Backup` exitCode equals 0 (success)  
    - *Connections:*  
      - True → Backup Successful  
      - False → Backup Failed  

  - **Backup Successful**  
    - *Type:* Set node  
    - *Role:* Constructs a JSON object reporting success status, timestamp, source-target info, and stdout of rsync  
    - *Parameters:*  
      - `status`: `"SUCCESS"`  
      - `timestamp`: current datetime (format: YYYY-MM-DD HH:mm:ss)  
      - `source`: combination of source host and folder  
      - `target`: combination of target host and folder  
      - `output`: standard output from rsync command  
    - *Connections:* Outputs to Process Finish Report --- Telegram & SMS  

  - **Backup Failed**  
    - *Type:* Set node  
    - *Role:* Constructs a JSON object reporting error status, timestamp, source-target info, and error details (exit code and stderr)  
    - *Parameters:*  
      - `status`: `"ERROR"`  
      - `timestamp`: current datetime  
      - `source`: source host  
      - `target`: target host  
      - `output`: concatenation of exitCode and stderr from rsync command  
    - *Connections:* Outputs to Process Finish Report --- Telegram & SMS  

#### 1.5 Notification Dispatch

- **Overview:**  
  Sends backup status notifications to configured Telegram channel and via SMS using Textbelt API.

- **Nodes Involved:**  
  - Process Finish Report --- Telegam & SMS  

- **Node Details:**  

  - **Process Finish Report --- Telegam & SMS**  
    - *Type:* Execute Command  
    - *Role:* Runs shell commands to:  
      - Install curl (apk add curl)  
      - Send Telegram message with backup report via Bot API  
      - Send SMS message to predefined phone numbers via Textbelt API  
    - *Parameters:*  
      - Environment variables placeholders: `TOKEN`, `CHAT_ID`, `NUMBERS`, `YOUR-TEXTBELT-API-KEY` to be replaced by user  
      - Constructs message string from previous node JSON fields  
    - *Connections:* None (end node)  
    - *On Error:* Continues regular output (does not fail workflow on notification error)  
    - *Potential Failures:* Missing or invalid API keys, network issues, Telegram bot restrictions  

---

### 3. Summary Table

| Node Name                       | Node Type           | Functional Role                           | Input Node(s)            | Output Node(s)                       | Sticky Note                                                                                                               |
|--------------------------------|---------------------|-----------------------------------------|--------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger                 | Manual Trigger      | Start workflow manually                  |                          | Server Parameters                  |                                                                                                                          |
| Schedule Trigger               | Schedule Trigger    | Start workflow on schedule               |                          | Server Parameters                  | Scheduling: Currently manual trigger and Schedule Trigger.                                                                |
| Server Parameters             | Set                 | Define server and backup parameters      | Manual Trigger, Schedule Trigger | Check Sshpass Local               | Server configuration required: define all source/target connection and backup parameters.                                 |
| Check Sshpass Local           | Execute Command      | Check sshpass installed locally          | Server Parameters        | Is Installed Local?                | Dependency Management: Checks and installs sshpass and rsync locally and remotely.                                         |
| Is Installed Local?           | If                  | Branch based on local sshpass presence   | Check Sshpass Local      | Check Sshpass on Source, Install Sshpass Local |                                                                                                                          |
| Install Sshpass Local         | Execute Command      | Install sshpass and rsync locally        | Is Installed Local? (False) | Check Sshpass on Source          |                                                                                                                          |
| Check Sshpass on Source       | Execute Command      | Check sshpass installed on source server | Is Installed Local?, Install Sshpass Local | Is Installed on Source?          |                                                                                                                          |
| Is Installed on Source?       | If                  | Branch based on sshpass presence on source | Check Sshpass on Source | Execute Rsync Backup, Install Sshpass on Source |                                                                                                                          |
| Install Sshpass on Source     | Execute Command      | Install sshpass and rsync on source server | Is Installed on Source? (False) | Execute Rsync Backup             |                                                                                                                          |
| Execute Rsync Backup          | Execute Command      | Perform the rsync backup                  | Is Installed on Source? (True), Install Sshpass on Source | Success?                      | Backup Execution: Runs rsync over SSH with password authentication, disables StrictHostKeyChecking.                        |
| Success?                     | If                  | Check if rsync backup succeeded          | Execute Rsync Backup     | Backup Successful, Backup Failed  | Status Handling: Evaluates rsync exit code and branches accordingly.                                                      |
| Backup Successful            | Set                 | Prepare success report                    | Success?                 | Process Finish Report --- Telegam & SMS |                                                                                                                          |
| Backup Failed                | Set                 | Prepare failure report                    | Success?                 | Process Finish Report --- Telegam & SMS |                                                                                                                          |
| Process Finish Report --- Telegam & SMS | Execute Command      | Send Telegram and SMS notifications       | Backup Successful, Backup Failed |                                | Notification Configuration: Replace placeholders for tokens, chat IDs, phone numbers, and API keys before use.          |
| Sticky Note                  | Sticky Note          | General workflow description              |                          |                                    | Automated Rsync Backup: password-based SSH backup, auto dependency install, notification via Telegram and SMS.            |
| Sticky Note1                 | Sticky Note          | Server configuration instructions         |                          |                                    | Server configuration required: parameters for source, target, and rsync options.                                          |
| Sticky Note2                 | Sticky Note          | Dependency management explanation          |                          |                                    | Dependency Management: automatic checks and installs of sshpass and rsync locally and on source server.                   |
| Sticky Note3                 | Sticky Note          | Backup execution explanation                |                          |                                    | Backup Execution: SSH to source server, rsync to target server, password auth with StrictHostKeyChecking disabled.        |
| Sticky Note4                 | Sticky Note          | Status handling explanation                 |                          |                                    | Status Handling: exit code check, detailed status report generation.                                                      |
| Sticky Note5                 | Sticky Note          | Notification setup explanation               |                          |                                    | Notification Configuration: placeholders for Telegram and SMS API keys and recipient info.                                |
| Sticky Note6                 | Sticky Note          | Scheduling explanation                       |                          |                                    | Scheduling: manual and schedule trigger nodes included.                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node (default settings).  
   - Add a **Schedule Trigger** node configured with desired interval (default is every minute or customize as needed).  

2. **Create and Configure Server Parameters Node:**  
   - Add a **Set** node named `Server Parameters`.  
   - Add string fields:  
     - `source_host` (e.g., IP or hostname of source server)  
     - `source_port` (SSH port, usually 22)  
     - `source_user` (username on source)  
     - `source_password` (password for source user, set as expression for security)  
     - `source_folder` (path to backup on source server)  
     - `target_host` (IP or hostname of target server)  
     - `target_port` (SSH port for target)  
     - `target_user` (username on target)  
     - `target_password` (password for target user, set as expression)  
     - `target_folder` (destination folder on target)  
     - `rsync_options` (default: `-avz --delete`)  

3. **Connect Triggers to Server Parameters:**  
   - Connect outputs of both **Manual Trigger** and **Schedule Trigger** to the `Server Parameters` node.  

4. **Check if sshpass is Installed Locally:**  
   - Add **Execute Command** node named `Check Sshpass Local` with command: `which sshpass`.  
   - Set **Continue On Fail** to true.  
   - Connect `Server Parameters` output to this node.  

5. **Add If Node `Is Installed Local?`:**  
   - Condition: numerical check if `exitCode` equals 0.  
   - Connect `Check Sshpass Local` output to this node.  

6. **Add Node `Install Sshpass Local`:**  
   - **Execute Command** node to install `sshpass` and `rsync` locally.  
   - Use multi-package manager shell script detecting `apt-get`, `yum`, `dnf`, or `apk` and installing packages accordingly.  
   - Connect `Is Installed Local?` **false** branch to this node.  

7. **Add Node `Check Sshpass on Source`:**  
   - **Execute Command** node that runs:  
     ```
     sshpass -p '{{ $json.source_password }}' ssh -o StrictHostKeyChecking=no -p {{ $json.source_port }} {{ $json.source_user }}@{{ $json.source_host }} "which sshpass"
     ```  
   - Use expressions to inject parameters from `Server Parameters`.  
   - Set **Continue On Fail** to true.  
   - Connect both `Is Installed Local?` **true** branch and `Install Sshpass Local` output to this node.  

8. **Add If Node `Is Installed on Source?`:**  
   - Checks if exitCode is 0 from previous node.  
   - Connect `Check Sshpass on Source` output to this node.  

9. **Add Node `Install Sshpass on Source`:**  
   - **Execute Command** node that SSH-es into source server with password, runs package manager checks and installs `sshpass` and `rsync` with sudo, passing the password.  
   - Use expressions for all parameters.  
   - Connect `Is Installed on Source?` **false** branch to this node.  

10. **Add Node `Execute Rsync Backup`:**  
    - **Execute Command** node that executes nested sshpass command:  
      ```
      sshpass -p '{{ $json.source_password }}' ssh -o StrictHostKeyChecking=no -p {{ $json.source_port }} {{ $json.source_user }}@{{ $json.source_host }} "sshpass -p '{{ $json.target_password }}' rsync {{ $json.rsync_options }} -e 'ssh -o StrictHostKeyChecking=no -p {{ $json.target_port }}' {{ $json.source_folder }}/ {{ $json.target_user }}@{{ $json.target_host }}:{{ $json.target_folder }}/"
      ```  
    - Connect both `Is Installed on Source?` **true** branch and `Install Sshpass on Source` output to this node.  

11. **Add If Node `Success?`:**  
    - Check if exitCode from `Execute Rsync Backup` equals 0.  
    - Connect `Execute Rsync Backup` output to this node.  

12. **Add Node `Backup Successful`:**  
    - **Set** node with fields:  
      - `status`: `"SUCCESS"`  
      - `timestamp`: current timestamp formatted as `YYYY-MM-DD HH:mm:ss`  
      - `source`: `source_host:source_folder` (concatenate)  
      - `target`: `target_host:target_folder`  
      - `output`: stdout from `Execute Rsync Backup`  
    - Connect `Success?` **true** branch to this node.  

13. **Add Node `Backup Failed`:**  
    - **Set** node with fields:  
      - `status`: `"ERROR"`  
      - `timestamp`: current timestamp  
      - `source`: `source_host` (only host)  
      - `target`: `target_host`  
      - `output`: concatenation of exitCode and stderr from `Execute Rsync Backup`  
    - Connect `Success?` **false** branch to this node.  

14. **Add Node `Process Finish Report --- Telegam & SMS`:**  
    - **Execute Command** node that:  
      - Installs `curl` via `apk add curl` (Alpine package manager)  
      - Sends Telegram message via Bot API using environment variables for `TOKEN`, `CHAT_ID`, and message built from previous node JSON  
      - Sends SMS via Textbelt API for configured phone numbers and API key  
    - Use placeholders for sensitive info: `YOUR-TELEGRAM-BOT-TOKEN`, `YOUR-TELEGRAM-CHANNEL-ID`, `YOUR-TEXTBELT-API-KEY`, and phone numbers.  
    - Connect both `Backup Successful` and `Backup Failed` outputs to this node.  
    - Set **On Error** to continue without failing workflow.  

15. **Optional:** Add Sticky Notes as documentation within the workflow for clarity.  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Automated Rsync Backup: password-based SSH backup between servers using rsync. Automatically installs dependencies, performs backup, sends Telegram and SMS notifications. Supports Ubuntu, Debian, RHEL, CentOS, Alpine. | Workflow general description (Sticky Note)                                                                |
| SERVER CONFIGURATION REQUIRED: Define all source and target connection parameters and rsync options. | Server Parameters node explanation (Sticky Note1)                                                         |
| DEPENDENCY MANAGEMENT: Automatically checks and installs sshpass and rsync locally and on source server. Supports apt, yum, dnf, apk. | Dependency block description (Sticky Note2)                                                                |
| BACKUP EXECUTION: SSH to source server, runs rsync to target server with password authentication, disables StrictHostKeyChecking to avoid prompts. | Rsync execution explanation (Sticky Note3)                                                                 |
| STATUS HANDLING: Checks rsync exit code for success/failure, generates detailed status report with timestamps. | Success/error handling info (Sticky Note4)                                                                 |
| NOTIFICATION CONFIGURATION: Replace placeholders for Telegram bot token, chat ID, phone numbers, and Textbelt API key. Sends backup status reports. | Notification node instructions (Sticky Note5)                                                              |
| SCHEDULING: Workflow supports manual trigger and schedule trigger (currently default interval). | Scheduling explanation (Sticky Note6)                                                                       |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.