Safely Update n8n with Version Checks and Telegram Notifications

https://n8nworkflows.xyz/workflows/safely-update-n8n-with-version-checks-and-telegram-notifications-9792


# Safely Update n8n with Version Checks and Telegram Notifications

### 1. Workflow Overview

This workflow automates the safe updating of an n8n instance by checking the current installed version against the latest available release on GitHub. It runs daily and at n8n startup, notifying users via Telegram about update availability and progress. The update procedure only proceeds if no workflows are currently running, ensuring stability and preventing disruptions.

**Logical Blocks:**

- **1.1 Triggers & Version Retrieval:** Initiates the workflow on schedule or n8n startup, fetching current and latest n8n versions.
- **1.2 Version Comparison:** Compares versions to detect if an update is needed.
- **1.3 Update Decision & Running Workflows Check:** Determines whether an update can proceed based on running executions.
- **1.4 Notifications & Update Execution:** Sends Telegram alerts regarding update status and performs the actual update command safely.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers & Version Retrieval

- **Overview:** This block initiates the workflow at scheduled intervals (daily at 9 AM) and at n8n instance startup. It retrieves the current installed n8n version and queries the latest release version from GitHub.

- **Nodes Involved:**  
  - Daily Trigger  
  - n8n Startup Trigger  
  - Get Current Version  
  - Get Latest Version  
  - Merge Versions  
  - Sticky Note - Trigger

- **Node Details:**

  - **Daily Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Runs workflow daily at 9:00 AM.  
    - *Configuration:* Interval trigger set to hour 9 every day.  
    - *Input:* None (trigger).  
    - *Output:* Initiates "Get Current Version" and "Get Latest Version".  
    - *Edge Cases:* Trigger time misconfiguration, time zone differences.  
    - *Sticky Note Context:* Notes block purpose as daily and startup trigger for version checks.

  - **n8n Startup Trigger**  
    - *Type:* n8n Trigger (System Events)  
    - *Role:* Runs workflow on n8n initialization/restart.  
    - *Configuration:* Event set to "init".  
    - *Input:* None (trigger).  
    - *Output:* Initiates "Get Latest Version" and "Get Current Version".  
    - *Edge Cases:* Startup event may not fire if n8n misconfigured; multiple startups in short time.

  - **Get Current Version**  
    - *Type:* Execute Command  
    - *Role:* Executes shell command `n8n -v` to retrieve current installed n8n version.  
    - *Configuration:* Command: `n8n -v`  
    - *Input:* Trigger output from Daily or Startup Trigger.  
    - *Output:* Raw stdout string with current version.  
    - *Edge Cases:* Command failing (e.g., n8n not in PATH), permission errors.  
    - *Sticky Note Context:* Part of version check block.

  - **Get Latest Version**  
    - *Type:* HTTP Request  
    - *Role:* Fetches latest n8n release info from GitHub API.  
    - *Configuration:* URL: `https://api.github.com/repos/n8n-io/n8n/releases/latest`  
    - *Input:* Trigger output from Daily or Startup Trigger.  
    - *Output:* JSON containing latest version tag under `tag_name`.  
    - *Edge Cases:* GitHub API rate limiting, network failures, JSON parsing errors.

  - **Merge Versions**  
    - *Type:* Merge  
    - *Role:* Combines outputs from "Get Current Version" and "Get Latest Version" for comparison.  
    - *Configuration:* Default merge settings (merge by index).  
    - *Input:* Output from "Get Current Version" (input 1) and "Get Latest Version" (input 2).  
    - *Output:* Merged data array containing both versions.  
    - *Edge Cases:* Missing input data, asynchronous timing issues.

  - **Sticky Note - Trigger**  
    - *Type:* Sticky Note  
    - *Role:* Describes this block as running daily and on startup to check for new n8n releases.

---

#### 1.2 Version Comparison

- **Overview:** Compares the current installed version against the latest available version, setting a boolean flag `updateAvailable`.

- **Nodes Involved:**  
  - Compare Versions  
  - If Update Available?  
  - Sticky Note - Comparison

- **Node Details:**

  - **Compare Versions**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts version strings, normalizes them, and compares equality.  
    - *Configuration:*  
      - Trims and removes prefixes from versions (e.g., `n8n@`, `v`).  
      - Sets `updateAvailable` to `true` if versions differ.  
    - *Key Expressions:*  
      ```js
      const current = $items()[0].json.stdout.trim();
      const latest = $items()[1].json.tag_name.trim();
      const c = current.replace(/^n8n@/, '').replace(/^v/, '');
      const l = latest.replace(/^n8n@/, '').replace(/^v/, '');
      return [{ json: { current: c, latest: l, updateAvailable: c !== l } }];
      ```  
    - *Input:* Merged version data.  
    - *Output:* JSON with current, latest versions and `updateAvailable` boolean.  
    - *Edge Cases:* Version format deviations, null or missing version strings.

  - **If Update Available?**  
    - *Type:* If (Boolean condition)  
    - *Role:* Routes flow based on whether an update is available (`updateAvailable === true`).  
    - *Configuration:* Condition: `updateAvailable` equals `true`.  
    - *Input:* Output from "Compare Versions".  
    - *Outputs:*  
      - True branch: begins update notification and execution process.  
      - False branch: sends notification that current version is latest.  
    - *Edge Cases:* Expression evaluation failure, missing `updateAvailable` field.

  - **Sticky Note - Comparison**  
    - *Type:* Sticky Note  
    - *Role:* Notes this block compares current vs latest versions and sets `updateAvailable`.

---

#### 1.3 Update Decision & Running Workflows Check

- **Overview:** Checks current running executions to determine if it is safe to update. Prevents update if workflows are running.

- **Nodes Involved:**  
  - Notify Update Available  
  - Get Executions  
  - Check Running Workflows  
  - If Can Update?  
  - Notify Latest Version  
  - Wait  
  - Sticky Note - Notifications

- **Node Details:**

  - **Notify Update Available**  
    - *Type:* Telegram  
    - *Role:* Sends a Telegram message notifying that an update is available.  
    - *Configuration:* Message text: `"Tienes una actualización disponible de n8n"`  
    - *Input:* True output branch from "If Update Available?"  
    - *Output:* Triggers "Get Executions".  
    - *Edge Cases:* Telegram API rate limit, invalid bot credentials or chat ID.

  - **Get Executions**  
    - *Type:* n8n API Node (Execution resource)  
    - *Role:* Retrieves all current executions to check their status.  
    - *Configuration:* No filters (fetches all executions).  
    - *Input:* After "Notify Update Available".  
    - *Output:* List of execution objects with status.  
    - *Edge Cases:* API permission issues, large execution history size.

  - **Check Running Workflows**  
    - *Type:* Function  
    - *Role:* Counts how many executions have status `"running"`.  
    - *Code:*  
      ```js
      const running = $items().filter(i => i.json.status === 'running');
      return [{ json: { canUpdate: running.length === 0, runningCount: running.length } }];
      ```  
    - *Input:* Output from "Get Executions".  
    - *Output:* JSON with `canUpdate` boolean and count of running executions.  
    - *Edge Cases:* Unexpected execution statuses.

  - **If Can Update?**  
    - *Type:* If (Boolean condition)  
    - *Role:* Routes flow depending on whether it is safe to update (`canUpdate === true`).  
    - *Input:* Output from "Check Running Workflows".  
    - *Outputs:*  
      - True: triggers notification and execution of update.  
      - False: triggers wait node to retry later.  
    - *Edge Cases:* Evaluation errors, missing `canUpdate`.

  - **Notify Latest Version**  
    - *Type:* Telegram  
    - *Role:* Sends message confirming n8n is up to date, used on false branch of "If Update Available?".  
    - *Configuration:* Text: `"Tu n8n está en la última versión v{{ $json.latest }}"`  
    - *Input:* False output from "If Update Available?".  
    - *Edge Cases:* Telegram failures as above.

  - **Wait**  
    - *Type:* Wait  
    - *Role:* Pauses workflow for 1 minute before rechecking for update possibility.  
    - *Configuration:* Wait for 1 minute.  
    - *Input:* False output from "If Can Update?".  
    - *Output:* Loops back to "Get Executions".  
    - *Edge Cases:* Long wait times delaying updates.

  - **Sticky Note - Notifications**  
    - *Type:* Sticky Note  
    - *Role:* Describes notification logic and safe update procedure ensuring no running executions during update.

---

#### 1.4 Notifications & Update Execution

- **Overview:** Sends Telegram alert that update is starting, then executes the system command to update n8n and restart the service.

- **Nodes Involved:**  
  - Notify Update Start  
  - Execute Update  
  - Sticky Note - Command

- **Node Details:**

  - **Notify Update Start**  
    - *Type:* Telegram  
    - *Role:* Sends Telegram message informing update is about to start.  
    - *Configuration:* Text: `"Se va a proceder a actualizar n8n"`  
    - *Input:* True output from "If Can Update?".  
    - *Output:* Triggers "Execute Update".  
    - *Edge Cases:* Telegram API issues.

  - **Execute Update**  
    - *Type:* Execute Command  
    - *Role:* Runs shell command to update n8n via npm and restarts systemd service.  
    - *Configuration:* Command:  
      ```bash
      sudo npm install -g n8n && sudo systemctl restart n8n
      ```  
    - *Input:* After "Notify Update Start".  
    - *Edge Cases:* Permission errors, failed npm install, service restart failures, sudo password prompts.

  - **Sticky Note - Command**  
    - *Type:* Sticky Note  
    - *Role:* Provides manual update commands for Linux and Docker environments for reference.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                              | Input Node(s)                          | Output Node(s)                          | Sticky Note                              |
|-------------------------|-------------------------|----------------------------------------------|--------------------------------------|---------------------------------------|----------------------------------------|
| Daily Trigger           | Schedule Trigger        | Initiate workflow daily at 9 AM               | None                                 | Get Current Version, Get Latest Version | Trigger & Version Check                 |
| n8n Startup Trigger      | n8n Trigger             | Initiate workflow on n8n startup              | None                                 | Get Latest Version, Get Current Version | Trigger & Version Check                 |
| Get Current Version      | Execute Command         | Retrieve installed n8n version via CLI        | Daily Trigger, n8n Startup Trigger   | Merge Versions                       | Trigger & Version Check                 |
| Get Latest Version       | HTTP Request            | Fetch latest n8n release from GitHub API      | Daily Trigger, n8n Startup Trigger   | Merge Versions                       | Trigger & Version Check                 |
| Merge Versions           | Merge                   | Combine current and latest version data       | Get Current Version, Get Latest Version | Compare Versions                    | Trigger & Version Check                 |
| Compare Versions         | Code                    | Compare versions and flag if update needed    | Merge Versions                      | If Update Available?                 | Version Comparison                     |
| If Update Available?     | If                      | Branch logic based on update availability     | Compare Versions                    | Notify Update Available / Notify Latest Version | Version Comparison                     |
| Notify Update Available  | Telegram                | Alert that update is available                 | If Update Available? (true)          | Get Executions                      | Notification Logic & Safe Update       |
| Get Executions           | n8n API (Execution)     | Retrieve current executions to check running  | Notify Update Available              | Check Running Workflows             | Notification Logic & Safe Update       |
| Check Running Workflows  | Function                | Count running executions and allow update     | Get Executions                      | If Can Update?                     | Notification Logic & Safe Update       |
| If Can Update?           | If                      | Decide if update can proceed safely            | Check Running Workflows              | Notify Update Start / Execute Update / Wait | Notification Logic & Safe Update       |
| Notify Update Start      | Telegram                | Notify update process is starting              | If Can Update? (true)                | Execute Update                     | Notification Logic & Safe Update       |
| Execute Update           | Execute Command         | Perform update command and restart n8n service | Notify Update Start                 | None                              | Update Command Info                    |
| Notify Latest Version    | Telegram                | Notify that n8n is already up to date          | If Update Available? (false)         | None                              | Notification Logic & Safe Update       |
| Wait                    | Wait                    | Delay before retrying update eligibility check | If Can Update? (false)               | Get Executions                    | Notification Logic & Safe Update       |
| Sticky Note - Trigger    | Sticky Note             | Describes trigger and version retrieval block | None                                 | None                              | Trigger & Version Check                 |
| Sticky Note - Comparison | Sticky Note             | Describes version comparison block             | None                                 | None                              | Version Comparison                     |
| Sticky Note - Notifications | Sticky Note          | Describes notification and safe update logic  | None                                 | None                              | Notification Logic & Safe Update       |
| Sticky Note - Command    | Sticky Note             | Provides manual update commands                 | None                                 | None                              | Update Command Info                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Trigger" node**  
   - Type: Schedule Trigger  
   - Set to run daily at 9:00 AM

2. **Create "n8n Startup Trigger" node**  
   - Type: n8n Trigger (System Events)  
   - Configure event: `init`

3. **Create "Get Current Version" node**  
   - Type: Execute Command  
   - Command: `n8n -v`  
   - Connect outputs of "Daily Trigger" and "n8n Startup Trigger" to this node

4. **Create "Get Latest Version" node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.github.com/repos/n8n-io/n8n/releases/latest`  
   - Connect outputs of "Daily Trigger" and "n8n Startup Trigger" to this node

5. **Create "Merge Versions" node**  
   - Type: Merge  
   - Mode: Merge by index (default)  
   - Connect output of "Get Current Version" (input 1) and "Get Latest Version" (input 2) here

6. **Create "Compare Versions" node**  
   - Type: Code (JavaScript)  
   - Paste the following code:  
     ```js
     const current = $items()[0].json.stdout.trim();
     const latest = $items()[1].json.tag_name.trim();
     const c = current.replace(/^n8n@/, '').replace(/^v/, '');
     const l = latest.replace(/^n8n@/, '').replace(/^v/, '');
     return [{ json: { current: c, latest: l, updateAvailable: c !== l } }];
     ```  
   - Connect "Merge Versions" output here

7. **Create "If Update Available?" node**  
   - Type: If  
   - Condition: Boolean, check if `updateAvailable === true`  
   - Connect output of "Compare Versions" to this node

8. **Create "Notify Update Available" node**  
   - Type: Telegram  
   - Configure credentials with your Telegram bot token  
   - Message text: `"Tienes una actualización disponible de n8n"`  
   - Connect true output of "If Update Available?" to this node

9. **Create "Get Executions" node**  
   - Type: n8n API node, resource: Execution, operation: List  
   - Connect output of "Notify Update Available" here

10. **Create "Check Running Workflows" node**  
    - Type: Function  
    - Paste code:  
      ```js
      const running = $items().filter(i => i.json.status === 'running');
      return [{ json: { canUpdate: running.length === 0, runningCount: running.length } }];
      ```  
    - Connect output of "Get Executions" here

11. **Create "If Can Update?" node**  
    - Type: If  
    - Condition: Boolean, check if `canUpdate === true`  
    - Connect output of "Check Running Workflows" here

12. **Create "Notify Update Start" node**  
    - Type: Telegram  
    - Message text: `"Se va a proceder a actualizar n8n"`  
    - Connect true output of "If Can Update?" here

13. **Create "Execute Update" node**  
    - Type: Execute Command  
    - Command: `sudo npm install -g n8n && sudo systemctl restart n8n`  
    - Connect output of "Notify Update Start" here

14. **Create "Wait" node**  
    - Type: Wait  
    - Duration: 1 minute  
    - Connect false output of "If Can Update?" here

15. **Connect "Wait" output back to "Get Executions" to retry**

16. **Create "Notify Latest Version" node**  
    - Type: Telegram  
    - Message text: `"Tu n8n está en la última versión v{{ $json.latest }}"`  
    - Connect false output of "If Update Available?" here

17. **Connect all nodes to ensure proper flow as per connections described**

18. **Add all sticky notes as documentation inside the workflow for clarity**

19. **Configure Telegram credentials with a valid bot token and chat ID in all Telegram nodes**

20. **Ensure n8n instance has permissions to run shell commands with sudo without password or configure accordingly**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                  |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Manual update commands for Linux and Docker environments:                                      | See Sticky Note - Command content                                |
| ```                                                                                           |                                                                 |
| sudo npm install -g n8n && sudo systemctl restart n8n                                         |                                                                 |
| docker pull n8nio/n8n:latest                                                                 |                                                                 |
| ```                                                                                           |                                                                 |
| Workflow runs safely by checking for running executions before updating, preventing failures. | Best practice for production n8n updates                        |
| Telegram bot setup required to send notifications.                                           | Telegram Bot API documentation: https://core.telegram.org/bots |

---

Disclaimer: The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.