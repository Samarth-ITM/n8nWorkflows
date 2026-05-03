Automated Credentials Backup to Google Drive via SSH and Docker

https://n8nworkflows.xyz/workflows/automated-credentials-backup-to-google-drive-via-ssh-and-docker-11842


# Automated Credentials Backup to Google Drive via SSH and Docker

### 1. Workflow Overview

This workflow automates the secure backup of n8n credentials from a Docker container to a designated Google Drive folder. It is designed primarily for users running n8n within Docker containers who want to regularly export and store their decrypted credential files off-host for safekeeping or disaster recovery.

The workflow includes three main logical blocks:

- **1.1 Initialization & Triggering:** Sets up essential variables and triggers the workflow either manually or on a schedule.
- **1.2 Credentials Export via SSH & Docker:** Connects via SSH to the Docker host, runs Docker commands to export the decrypted n8n credentials into a JSON file inside the container.
- **1.3 Upload to Google Drive:** Reads the exported credentials file and uploads it to a specified folder in Google Drive with a timestamped filename.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization & Triggering

- **Overview:**  
  This block initializes necessary variables such as Docker container name, backup folder, and credentials file path. It also includes triggers to start the workflow either manually or on a recurring schedule.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Schedule Trigger  
  - Variables

- **Node Details:**

  - **On clicking 'execute'**  
    - *Type & Role:* Manual trigger node; allows the workflow to be started on demand by a user.  
    - *Configuration:* No parameters required.  
    - *Connections:* Outputs to Variables node.  
    - *Edge Cases:* None significant; manual triggers depend on user action.

  - **Schedule Trigger**  
    - *Type & Role:* Time-based trigger node; automatically initiates the workflow at regular intervals.  
    - *Configuration:* Interval set to default (every minute/hour/day depending on user setup).  
    - *Connections:* Outputs to Variables node.  
    - *Edge Cases:* Misconfiguration of interval can cause too frequent or too sparse backups.

  - **Variables**  
    - *Type & Role:* Set node; stores workflow configuration variables.  
    - *Configuration:*  
      - Backup Folder = "n8n backups" (Google Drive folder name)  
      - Docker Container name = "YOUR n8n DOCKER CONTAINER NAME" (placeholder, must be replaced)  
      - Docker - n8n credentials files = "/home/node/.n8n-files/credentials.json" (file path inside Docker container)  
    - *Connections:* Receives input from both triggers, outputs to Execute a command node.  
    - *Edge Cases:* Variables must be correctly configured before running; incorrect Docker container name or paths will cause failures downstream.

---

#### Block 1.2: Credentials Export via SSH & Docker

- **Overview:**  
  This block executes commands on the Docker host via SSH to export all decrypted n8n credentials into a JSON file inside the Docker container.

- **Nodes Involved:**  
  - Execute a command

- **Node Details:**

  - **Execute a command**  
    - *Type & Role:* SSH node; runs shell commands on the remote Docker host.  
    - *Configuration:*  
      - Command composed dynamically using variables:  
        `docker exec -u node {{ Docker Container name }} mkdir -p /home/node/.n8n-files && docker exec -u node {{ Docker Container name }} n8n export:credentials --all --decrypted --output={{ Docker - n8n credentials files }}`  
      - This command first ensures the target directory exists, then exports all decrypted credentials to the specified JSON file.  
    - *Key Expressions:* Uses expressions to insert variable values for container name and file path.  
    - *Connections:* Input from Variables node, output to Read File node.  
    - *Edge Cases:*  
      - SSH connection failure (wrong host, credentials, or network issues)  
      - Docker container name incorrect or container not running  
      - Permission issues inside container  
      - Command syntax errors or n8n CLI incompatibility  
      - Timeout if command takes too long

---

#### Block 1.3: Upload to Google Drive

- **Overview:**  
  Reads the exported credentials JSON file from the filesystem and uploads it to a specified Google Drive folder with a timestamped filename.

- **Nodes Involved:**  
  - Read File  
  - Google Drive Upload File

- **Node Details:**

  - **Read File**  
    - *Type & Role:* Read/Write File node; reads the exported credentials JSON file from the local filesystem.  
    - *Configuration:*  
      - File path dynamically set to the variable `Docker - n8n credentials files` (e.g., `/home/node/.n8n-files/credentials.json`).  
      - Data read into the property `data`.  
    - *Connections:* Input from Execute a command node, output to Google Drive Upload File node.  
    - *Edge Cases:*  
      - File not found if the export command failed or file path incorrect  
      - Permission denied reading file  
      - File empty or corrupted

  - **Google Drive Upload File**  
    - *Type & Role:* Google Drive node; uploads the read JSON file to Google Drive.  
    - *Configuration:*  
      - Filename: `n8n_credentials <weekday> <time> <date>.json` (e.g., "n8n_credentials Monday 3:00 PM 01-01-2024.json") using current timestamp formatting.  
      - Drive: "My Drive" (default Google Drive root)  
      - Folder: "n8n backups" (must match configured folder)  
      - Input data field: `data` (content read from file)  
    - *Connections:* Input from Read File node.  
    - *Edge Cases:*  
      - Google Drive authentication failures (credential misconfiguration)  
      - Folder ID invalid or inaccessible  
      - Upload size limits or API rate limiting  
      - Filename conflicts (usually overwritten or versioned by Drive)

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                                         |
|----------------------|---------------------|----------------------------------------|-----------------------|------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger      | Manual workflow start                   | —                     | Variables              |                                                                                                                                     |
| Schedule Trigger      | Schedule Trigger    | Scheduled workflow start                | —                     | Variables              |                                                                                                                                     |
| Variables            | Set                 | Define configuration variables          | On clicking 'execute', Schedule Trigger | Execute a command      |                                                                                                                                     |
| Execute a command    | SSH                  | Export n8n credentials inside Docker    | Variables              | Read File              | ## Export All Credentials From n8n Docker Container<br>* Setup your variables first<br>* Connect to the host using SSH or your details<br>* Executing Docker command to backup credentials<br>* Read the file from the host |
| Read File            | Read/Write File      | Read exported credentials JSON file     | Execute a command      | Google Drive Upload File |                                                                                                                                     |
| Google Drive Upload File | Google Drive      | Upload credentials file to Google Drive | Read File              | —                      | ## Google Drive Folder<br>* Setup your Google Drive credentials<br>* Upload credentials from n8n to Google Drive selected folder |
| Sticky Note1          | Sticky Note          | Notes on backup approach and config     | —                     | —                      | ## Backup Credentials to Drive for **n8n 2.x.x**<br>This solution will work with the n8n version 2.x.x and 1.x.x.<br>It is not used CLI commands what are not working in NEW n8n version.<br>**NOTE** This solution was configured to work with Docker<br>Configure required variables from "Variables" tool in the following way by default:<br>... (see full content above) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "On clicking 'execute'"  
   - No parameters needed.

2. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Schedule Trigger"  
   - Configure interval as desired (e.g., daily at specific time).

3. **Create Variables Node**  
   - Type: Set  
   - Name: "Variables"  
   - Add three string variables with these exact names and default values:  
     - Backup Folder — default: "n8n backups"  
     - Docker Container name — default: "YOUR n8n DOCKER CONTAINER NAME" (replace with actual container name or ID)  
     - Docker - n8n credentials files — default: "/home/node/.n8n-files/credentials.json"  
   - Connect outputs of both the Manual Trigger and Schedule Trigger nodes to this Variables node.

4. **Create SSH Node**  
   - Type: SSH  
   - Name: "Execute a command"  
   - Configure SSH credentials for the Docker host machine (username, host, port, private key, or password).  
   - Set command to:  
     ```bash
     docker exec -u node {{ $json["Docker Container name"] }} mkdir -p /home/node/.n8n-files && docker exec -u node {{ $json["Docker Container name"] }} n8n export:credentials --all --decrypted --output={{ $json["Docker - n8n credentials files"] }}
     ```  
   - Connect output of Variables node to this node.

5. **Create Read/Write File Node**  
   - Type: Read/Write File  
   - Name: "Read File"  
   - Configure to read file path from variable: `{{ $('Variables').item.json['Docker - n8n credentials files'] }}`  
   - Set data property name to `data`.  
   - Connect output of SSH node to this node.

6. **Create Google Drive Node**  
   - Type: Google Drive  
   - Name: "Google Drive Upload File"  
   - Configure Google Drive OAuth2 credentials (set up in n8n credentials manager).  
   - Set upload filename to:  
     `n8n_credentials {{ $now.format('cccc t dd-MM-yyyy') }}.json`  
   - Set Drive ID to "My Drive".  
   - Set Folder ID to the variable or select folder named "n8n backups" (ensure folder exists in Drive).  
   - Use input data field name: `data` (content read from file).  
   - Connect output of Read File node to this node.

7. **Verify Connections**  
   - Both Manual Trigger and Schedule Trigger → Variables → Execute a command → Read File → Google Drive Upload File

8. **Testing and Validation**  
   - Replace placeholder variables with actual Docker container name.  
   - Confirm SSH credentials and access to Docker host.  
   - Ensure n8n container is running and supports CLI export command.  
   - Confirm Google Drive credentials have permission to upload to the target folder.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                               | Context or Link                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| This workflow supports n8n versions 1.x.x and 2.x.x, using a Docker-based approach without relying on deprecated CLI commands incompatible with newer n8n versions.                                                                                                        | Sticky Note1 content                     |
| For help or customization support, contact alex@elitiv.com.                                                                                                                                                                                                               | Sticky Note1 contact info                |
| The workflow requires setting up SSH access to the Docker host machine where n8n runs, and Google Drive OAuth2 credentials configured in n8n for file uploads.                                                                                                            | Workflow prerequisites                   |
| Google Drive folder "n8n backups" must exist prior to running the workflow or be created manually.                                                                                                                                                                         | Google Drive Upload File node settings  |
| The export command runs inside the Docker container with user 'node' to ensure proper permissions and environment for the n8n CLI export operation.                                                                                                                    | Execute a command node description      |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.