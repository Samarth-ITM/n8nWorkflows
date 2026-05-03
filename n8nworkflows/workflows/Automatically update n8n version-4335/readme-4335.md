Automatically update n8n version

https://n8nworkflows.xyz/workflows/automatically-update-n8n-version-4335


# Automatically update n8n version

### 1. Workflow Overview

This workflow automates the process of checking the locally installed n8n version against the latest available version on npm and triggers an update command via SSH if a newer version exists. It is designed for environments where n8n is self-hosted and can be updated remotely using SSH commands. The workflow runs on a scheduled basis, compares versions, and conditionally executes update logic.

Logical blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow daily at a specified hour.
- **1.2 Latest Version Retrieval**: Fetches the latest n8n version from the npm registry.
- **1.3 Local Version Retrieval**: Obtains the locally installed n8n version via an HTTP API.
- **1.4 Version Comparison and Conditional Update**: Compares versions and triggers an SSH command to update if needed.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow automatically every day at 5 AM to start the update check process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Trigger set to 5 AM daily (`triggerAtHour: 5`)  
    - Inputs: None (entry point)  
    - Outputs: Connected to "Get the latest n8n version"  
    - Version-specific: Requires n8n that supports scheduleTrigger type version 1.2  
    - Potential Failures: None expected; may fail if n8n is offline at scheduled time.

#### 1.2 Latest Version Retrieval

- **Overview:**  
  This block fetches the latest stable release version number of n8n from the official npm registry.

- **Nodes Involved:**  
  - Get the latest n8n version

- **Node Details:**  
  - **Get the latest n8n version**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://registry.npmjs.org/n8n/latest`  
      - Options: Full response disabled (only JSON body fetched)  
      - No authentication  
    - Inputs: Triggered by Schedule Trigger  
    - Outputs: Connected to "Get Local n8n version"  
    - Version-specific: Uses HTTP Request node version 2  
    - Potential Failures: Network issues, npm registry downtime, or unexpected response structure.

#### 1.3 Local Version Retrieval

- **Overview:**  
  Retrieves the current n8n version running locally by querying the local n8n REST API, using Bearer token authentication.

- **Nodes Involved:**  
  - Get Local n8n version

- **Node Details:**  
  - **Get Local n8n version**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `http://192.168.100.18:5678/rest/settings` (local n8n API endpoint)  
      - Authentication: HTTP Bearer Token (configured via credential named "Bearer Auth account")  
      - No additional request options set  
    - Inputs: Receives data from "Get the latest n8n version"  
    - Outputs: Connected to "If" node for version comparison  
    - Version-specific: HTTP Request node version 4.2  
    - Potential Failures:  
      - Authentication failure due to invalid token  
      - Local n8n API not reachable or offline  
      - Unexpected response format

#### 1.4 Version Comparison and Conditional Update

- **Overview:**  
  Compares the local n8n version with the latest npm version. If they differ, it triggers an SSH command to create a file (`check_update.txt`) indicating update necessity.

- **Nodes Involved:**  
  - If  
  - SSH1

- **Node Details:**  
  - **If**  
    - Type: If (conditional node)  
    - Configuration:  
      - Condition: Checks if the latest npm version (`$('Get the latest n8n version').item.json.version`) is not equal to the local version (`$json.data.versionCli`)  
      - Case sensitive and strict type validation enabled  
      - Only executes true branch when versions differ  
    - Inputs: From "Get Local n8n version"  
    - Outputs: True branch and false branch both connected to "SSH1" node (likely a logical error or intentional redundancy)  
    - Version-specific: If node version 2.2  
    - Potential Failures: Expression evaluation failure if JSON paths are incorrect or data missing

  - **SSH1**  
    - Type: SSH  
    - Configuration:  
      - Remote command: `echo "{{ $('If').params.conditions ? 'true' : 'false' }}" > check_update.txt`  
      - Working directory: `n8n/`  
      - Credentials: SSH Password credential named "SSH Password account"  
    - Inputs: Both true and false branches of "If" connect here (which may cause the command to execute regardless of condition)  
    - Outputs: None (end node)  
    - Version-specific: SSH node version 1  
    - Potential Failures:  
      - SSH connection failure due to network, credentials, or host issues  
      - Command execution errors if directory does not exist or permissions are insufficient

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                     | Input Node(s)              | Output Node(s)             | Sticky Note                         |
|-------------------------|------------------|-----------------------------------|----------------------------|----------------------------|-----------------------------------|
| Schedule Trigger        | Schedule Trigger | Starts workflow daily at 5 AM     | None                       | Get the latest n8n version |                                   |
| Get the latest n8n version | HTTP Request    | Fetches latest n8n version from npm registry | Schedule Trigger           | Get Local n8n version       |                                   |
| Get Local n8n version   | HTTP Request     | Retrieves local n8n version via API | Get the latest n8n version | If                         |                                   |
| If                      | If               | Compares local and latest versions | Get Local n8n version       | SSH1 (true and false branches) |                                   |
| SSH1                    | SSH              | Executes remote command to indicate update status | If                        | None                       |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**

   - Type: Schedule Trigger  
   - Parameters: Set to trigger daily at 5:00 AM (`triggerAtHour: 5`)  
   - No credentials required  
   - Position it as the entry node.

2. **Create an HTTP Request node named "Get the latest n8n version":**

   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://registry.npmjs.org/n8n/latest`  
     - Options: Disable full response (only JSON body)  
   - Connect input from Schedule Trigger’s output.

3. **Create an HTTP Request node named "Get Local n8n version":**

   - Type: HTTP Request  
   - Parameters:  
     - URL: `http://192.168.100.18:5678/rest/settings` (replace IP with your local n8n API address)  
     - Authentication: HTTP Bearer Auth  
       - Set up credentials with the valid Bearer token for local n8n API  
   - Connect input from "Get the latest n8n version" output.

4. **Create an If node named "If":**

   - Type: If  
   - Parameters:  
     - Condition: Check if `{{$node["Get the latest n8n version"].json["version"]}}` not equals `{{$json["data"]["versionCli"]}}`  
     - Set case sensitivity and strict type validation to true  
   - Connect input from "Get Local n8n version" output.

5. **Create an SSH node named "SSH1":**

   - Type: SSH  
   - Parameters:  
     - Command: `echo "{{ $('If').params.conditions ? 'true' : 'false' }}" > check_update.txt`  
     - Working directory: `n8n/` (ensure this path exists on target server)  
   - Authentication: SSH Password (set up credentials with username and password for target host)  
   - Connect both true and false outputs of the If node to this SSH node.

6. **Activate the workflow** to run daily and perform version checks and conditional update signaling.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                   |
|------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow assumes local n8n API is accessible at `http://192.168.100.18:5678/rest/settings` and requires a valid Bearer token. Adjust accordingly. | Local API endpoint and authentication details    |
| SSH command writes a file named `check_update.txt` in the `n8n/` directory to signal update necessity. Ensure the SSH user has write permissions. | Remote update signaling mechanism                 |
| Both true and false branches of the If node connect to the SSH node, which may cause the command to run regardless of version comparison result. This might be intentional or require review. | Logical flow note                                  |
| For credentials setup, ensure the HTTP Bearer token and SSH password are securely stored in n8n credentials and have appropriate access rights. | Credential management best practices               |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.