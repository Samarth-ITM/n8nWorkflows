Deploy Docker MinIO, API Backend for WHMCS/WISECP

https://n8nworkflows.xyz/workflows/deploy-docker-minio--api-backend-for-whmcs-wisecp-3198


# Deploy Docker MinIO, API Backend for WHMCS/WISECP

### 1. Workflow Overview

This n8n workflow automates the deployment and management of Docker MinIO containers for WHMCS/WISECP modules via an API backend. It listens for incoming API webhook commands, validates requests, and uses SSH to execute predefined bash scripts on a remote Docker host. The workflow is designed for server administrators who want to programmatically control container lifecycle, disk mounts, ACLs, and other service operations.

Logical blocks include:  
- **1.1 Input Reception & Validation:** Receives API requests via webhook, authenticates, and validates server domain.  
- **1.2 Parameter Initialization:** Loads configurable parameters such as server domain, client directories, and mount points.  
- **1.3 Command Routing:** Switch nodes that route API commands into service-related or container-related actions.  
- **1.4 SSH Execution & Bash Scripts:** Sends bash scripts over SSH to the Docker server to perform actions like create, start, stop, suspend, mount disks, manage ACLs, etc.  
- **1.5 Response Handling:** Parses SSH command outputs, formats JSON responses, and returns them to API clients.  
- **1.6 Configuration Templates:** Contains editable templates for Docker Compose files and Nginx proxy configurations.  
- **1.7 Documentation & Setup Notes:** Provides user instructions and workflow context in sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:** This block exposes the API webhook endpoint, authenticates requests using Basic Auth, and validates the `server_domain` parameter from incoming JSON bodies. If invalid, it returns a 422 error response.  
- **Nodes Involved:** API (Webhook), Parametrs (Set), If (Condition), 422-Invalid server domain (RespondToWebhook)  
- **Node Details:**  
  - **API (Webhook):**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Path: `/docker-minio`  
    - Auth: Basic Auth Credential named "MinIO"  
    - Output: Raw JSON body with `command` and domain-related data  
    - Potential Failures: Auth failure, invalid/missing JSON body, incorrect webhook path  
  - **Parametrs (Set):**  
    - Assigns static parameters: `server_domain` (e.g., d01-test.uuq.pl), `clients_dir`, `mount_dir`, `screen_left`, `screen_right`  
    - These parameters are used throughout bash scripts for path and config generation  
  - **If (Condition):**  
    - Compares incoming `server_domain` from API body to configured `server_domain`  
    - Routes valid requests downstream, invalid requests to error response  
  - **422-Invalid server domain (RespondToWebhook):**  
    - Returns JSON error with HTTP status 422 if domain check fails  
    - No output downstream  

#### 2.2 Parameter Initialization

- **Overview:** Centralizes configurable parameters used in bash scripts and configuration templates.  
- **Nodes Involved:** Parametrs (Set)  
- **Node Details:**  
  - Assigns server domain, client directory, mount directory, and template braces `screen_left` and `screen_right` for embedding JSON in bash scripts.  
  - These values are critical for generating paths and configuration files on the remote server.

#### 2.3 Command Routing

- **Overview:** Routes the API `command` field into different functional branches based on service lifecycle or container management commands.  
- **Nodes Involved:** Container Actions (Switch), Service Actions (Switch), Container Stat (Switch), MinIO (Switch), If1 (If)  
- **Node Details:**  
  - **Container Actions (Switch):** Routes container-related commands such as `container_start`, `container_stop`, `container_mount_disk`, `container_unmount_disk`, `container_get_acl`, `container_set_acl`, `container_get_net`.  
  - **Service Actions (Switch):** Routes service lifecycle commands such as `test_connection`, `create`, `suspend`, `unsuspend`, `terminate`, `change_package`.  
  - **Container Stat (Switch):** Routes container information commands like `container_information_inspect`, `container_information_stats`, `container_log`.  
  - **MinIO (Switch):** Routes MinIO specific commands like `app_version`, `app_users`.  
  - **If1 (If):** Checks if command is one of `create`, `change_package`, or `unsuspend` to route to nginx config setup or service actions.

#### 2.4 SSH Execution & Bash Scripts

- **Overview:** Executes dynamically generated bash scripts on the remote Docker host over SSH to perform all operational tasks.  
- **Nodes Involved:** SSH (SSH node), multiple Set nodes generating bash scripts (e.g., Deploy, Start, Stop, Suspend, Unsuspend, Terminated, ChangePackage, Mount Disk, Unmount Disk, GET ACL, SET ACL, GET NET, Inspect, Stat, Log, Test Connection1, Users, Version)  
- **Node Details:**  
  - **SSH Node:**  
    - Executes bash script from the incoming JSON field `sh` on the remote host under root or sudo privileges.  
    - Credential: SSH password credential linked to the server domain.  
    - On error: continues with error output for further handling.  
  - **Set Nodes (Script Generators):**  
    - Each node constructs a bash script tailored to a specific command.  
    - Scripts manage Docker containers (start, stop, inspect), filesystem mounts, Nginx ACL configurations, Docker Compose deployment, version retrieval, MinIO user management, etc.  
    - Scripts include error handling to return JSON-formatted errors or success messages.  
    - Scripts use parameters from the Parametrs node and dynamic API body input (e.g., domain, username, password, disk size).  
    - Potential failures: SSH connection issues, permission errors, Docker daemon errors, filesystem errors (mount/unmount), command timeouts, malformed or missing input parameters.

#### 2.5 Response Handling

- **Overview:** Parses the output of the SSH execution to standardize API responses to the client.  
- **Nodes Involved:** Code1 (Code), API answer (RespondToWebhook)  
- **Node Details:**  
  - **Code1 (Code):**  
    - Parses `stdout` from SSH execution.  
    - If output is `"success"`, returns a success JSON object with empty message and data.  
    - If output is JSON, parses it and returns status, message, and data accordingly.  
    - On parse error or unexpected output, returns an error JSON with message from output or error.  
  - **API answer (RespondToWebhook):**  
    - Sends back HTTP 200 with the final parsed JSON response to the API client.

#### 2.6 Configuration Templates

- **Overview:** Holds editable templates for Docker Compose YAML and Nginx proxy configurations used in deployment and proxy setup.  
- **Nodes Involved:** Deploy-docker-compose (Set), nginx (Set)  
- **Node Details:**  
  - **Deploy-docker-compose (Set):**  
    - Defines a Docker Compose v3 file with the MinIO service configuration.  
    - Uses API inputs for domain, username, password, RAM, CPU limits.  
    - Sets volumes for persistent data and connects to external nginx-proxy network.  
  - **nginx (Set):**  
    - Provides configuration snippets for Nginx proxy server (main and location sections).  
    - Allows customization of headers, proxy timeouts, buffering, and connection upgrades.  
    - Used in bash scripts to generate Nginx config files on the server.

#### 2.7 Documentation & Setup Notes

- **Overview:** Contains user-facing documentation and setup instructions embedded in the workflow for convenience.  
- **Nodes Involved:** Sticky Note  
- **Node Details:**  
  - Provides welcome message, setup instructions, credential configuration, parameter modification guidance, and links to full documentation and module pages.  
  - Includes links to official n8n marketplace templates and PUQcloud resources.  
  - Useful for operators deploying or maintaining the workflow.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                              | Input Node(s)                         | Output Node(s)                              | Sticky Note                                                                                                      |
|-------------------------|----------------------|----------------------------------------------|-------------------------------------|---------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| API                     | Webhook              | Receives API POST requests, Basic Auth      | —                                   | Parametrs                                   |                                                                                                                  |
| Parametrs               | Set                  | Sets global parameters for scripts           | API                                 | If                                          |                                                                                                                  |
| If                      | If                   | Validates `server_domain`                     | Parametrs                           | Container Stat, Container Actions, MinIO, If1; or 422-Invalid server domain |                                                                                                                  |
| 422-Invalid server domain | RespondToWebhook    | Returns 422 error for invalid domain          | If                                 | —                                           |                                                                                                                  |
| Container Stat          | Switch               | Routes container info commands                | If                                 | Inspect, Stat, Log                           |                                                                                                                  |
| Inspect                 | Set                  | Generates bash script for `docker inspect`   | Container Stat                     | SSH                                         |                                                                                                                  |
| Stat                    | Set                  | Generates bash script for container stats     | Container Stat                     | SSH                                         |                                                                                                                  |
| Log                     | Set                  | Generates bash script to fetch container logs | Container Stat                     | SSH                                         |                                                                                                                  |
| SSH                     | SSH                  | Executes bash scripts remotely via SSH        | Start, Stop, Stat, Inspect, Log, etc. | Code1                                       |                                                                                                                  |
| Code1                   | Code                 | Parses SSH output, formats API response       | SSH                               | API answer                                   |                                                                                                                  |
| API answer              | RespondToWebhook     | Sends HTTP response back to API client        | Code1                             | —                                           |                                                                                                                  |
| Container Actions       | Switch               | Routes container lifecycle commands           | If                                | Start, Stop, Mount Disk, Unmount Disk, GET ACL, SET ACL, GET NET |                                                                                                                  |
| Start                   | Set                  | Bash script to start Docker container         | Container Actions                 | SSH                                         |                                                                                                                  |
| Stop                    | Set                  | Bash script to stop Docker container          | Container Actions                 | SSH                                         |                                                                                                                  |
| Mount Disk              | Set                  | Bash script to mount disk image                | Container Actions                 | SSH                                         |                                                                                                                  |
| Unmount Disk            | Set                  | Bash script to unmount disk image              | Container Actions                 | SSH                                         |                                                                                                                  |
| GET ACL                 | Set                  | Bash script to get Nginx ACL IP lists          | Container Actions                 | SSH                                         |                                                                                                                  |
| SET ACL                 | Set                  | Bash script to set Nginx ACL IP lists          | Container Actions                 | SSH                                         |                                                                                                                  |
| GET NET                 | Set                  | Bash script to get container network stats     | Container Actions                 | SSH                                         |                                                                                                                  |
| Service Actions         | Switch               | Routes service commands                         | If1                              | Test Connection1, Deploy, Suspend, Unsuspend, Terminated, ChangePackage |                                                                                                                  |
| Test Connection1        | Set                  | Bash script to test server Docker and proxy    | Service Actions                  | SSH                                         |                                                                                                                  |
| Deploy                  | Set                  | Bash script to deploy container and configs   | Service Actions                  | SSH                                         |                                                                                                                  |
| Suspend                 | Set                  | Bash script to suspend container and cleanup  | Service Actions                  | SSH                                         |                                                                                                                  |
| Unsuspend               | Set                  | Bash script to unsuspend container and remount | Service Actions                  | SSH                                         |                                                                                                                  |
| Terminated              | Set                  | Bash script to terminate and cleanup container | Service Actions                  | SSH                                         |                                                                                                                  |
| ChangePackage           | Set                  | Bash script to change container package & resize disk | Service Actions                  | SSH                                         |                                                                                                                  |
| MinIO                   | Switch               | Routes MinIO specific commands                 | If                               | Version, Users                               |                                                                                                                  |
| Version                 | Set                  | Bash script to get MinIO version               | MinIO                            | SSH                                         |                                                                                                                  |
| Users                   | Set                  | Bash script to list MinIO users                 | MinIO                            | SSH                                         |                                                                                                                  |
| If1                     | If                   | Checks if command requires nginx config setup | If                              | nginx, Service Actions                        |                                                                                                                  |
| nginx                   | Set                  | Sets Nginx proxy config templates               | If1                             | Deploy-docker-compose                         |                                                                                                                  |
| Deploy-docker-compose   | Set                  | Sets Docker Compose YAML template                | nginx                           | Service Actions                               |                                                                                                                  |
| Sticky Note             | StickyNote           | Documentation, setup instructions                | —                               | —                                            | ## 👋 Welcome to PUQ Docker MinIO deploy! Template for MinIO: API Backend for WHMCS/WISECP by PUQcloud v.1 ... \nFull doc: https://doc.puq.info/books/docker-minio-whmcs-module \nModule: https://puqcloud.com/whmcs-module-docker-minio.php |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("API")**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `docker-minio`  
   - Authentication: Basic Auth (create and assign Basic Auth credentials named "MinIO")  
   - Response Mode: Response Node  
   - Allow multiple HTTP methods: true  

2. **Create Set Node ("Parametrs")**  
   - Assign static parameters:  
     - `server_domain` (e.g. `d01-test.uuq.pl`)  
     - `clients_dir` (e.g. `/opt/docker/clients`)  
     - `mount_dir` (e.g. `/mnt`)  
     - `screen_left` = `{{`  
     - `screen_right` = `}}`  
   - Connect `API` → `Parametrs`  

3. **Create If Node ("If")**  
   - Condition: Check if `{{$json["server_domain"]}}` equals `{{$('API').item.json.body.server_domain}}`  
   - Connect `Parametrs` → `If`  

4. **Create RespondToWebhook Node ("422-Invalid server domain")**  
   - Response Code: 422  
   - Respond With: JSON  
   - Response Body: `[{ "status": "error", "error": "Invalid server domain" }]`  
   - Connect `If` (false output) → `422-Invalid server domain`  

5. **Create Switch Node ("Container Stat")**  
   - Rules for commands:  
     - `container_information_inspect` → output "inspect"  
     - `container_information_stats` → output "stats"  
     - `container_log` → output "log"  
   - Connect `If` (true output) → `Container Stat`  

6. **Create Set Nodes for container info scripts:**  
   - **"Inspect"** (bash script for docker inspect)  
   - **"Stat"** (bash script for docker stats and disk info)  
   - **"Log"** (bash script for container logs)  
   - Connect `Container Stat` outputs accordingly → `Inspect`, `Stat`, `Log`  

7. **Create Switch Node ("Container Actions")**  
   - Rules for commands: `container_start`, `container_stop`, `container_mount_disk`, `container_unmount_disk`, `container_get_acl`, `container_set_acl`, `container_get_net`  
   - Connect `If` (true output) → `Container Actions`  

8. **Create Set Nodes for container action scripts:**  
   - "Start", "Stop", "Mount Disk", "Unmount Disk", "GET ACL", "SET ACL", "GET NET"  
   - Connect `Container Actions` outputs → corresponding Set nodes  

9. **Create Switch Node ("Service Actions")**  
   - Rules for commands: `test_connection`, `create`, `suspend`, `unsuspend`, `terminate`, `change_package`  
   - Connect `If1` (below) → `Service Actions`  

10. **Create If Node ("If1")**  
    - Condition: command equals one of `create`, `change_package`, `unsuspend`  
    - Connect `If` (true output) → `If1`  
    - `If1` true output → `nginx` node  
    - `If1` false output → `Service Actions`  

11. **Create Set Node ("nginx")**  
    - Assign nginx configuration templates (main, main_location, console, console_location)  
    - Connect `If1` → `nginx` → `Deploy-docker-compose`  

12. **Create Set Node ("Deploy-docker-compose")**  
    - Assign Docker Compose YAML template using API input variables  
    - Connect `nginx` → `Deploy-docker-compose` → `Service Actions`  

13. **Create Set Nodes for service action scripts:**  
    - "Test Connection1", "Deploy", "Suspend", "Unsuspend", "Terminated", "ChangePackage"  
    - Connect `Service Actions` outputs → corresponding Set nodes  

14. **Create Switch Node ("MinIO")**  
    - Rules for commands: `app_version`, `app_users`  
    - Connect `If` → `MinIO` → outputs to "Version" and "Users" nodes  

15. **Create Set Nodes ("Version", "Users")**  
    - Bash scripts for MinIO version and user listing  
    - Connect `MinIO` outputs → `Version`, `Users`  

16. **Create SSH Node ("SSH")**  
    - Credential: SSH credential for Docker host (password or key-based)  
    - Command: `={{ $json.sh }}` to accept bash script from input  
    - On Error: Continue with error output  
    - Connect all Set nodes generating scripts → SSH node  
    - Connect SSH node output → "Code1"  

17. **Create Code Node ("Code1")**  
    - JavaScript to parse SSH output:  
      - If output is "success", return success JSON  
      - If output JSON with status, return accordingly  
      - On error, return error JSON  
    - Connect SSH → Code1 → API answer  

18. **Create RespondToWebhook Node ("API answer")**  
    - HTTP Status 200  
    - Respond with all incoming items  
    - Connect Code1 → API answer  

19. **Create Sticky Note**  
    - Add user instructions, setup steps, and links to documentation  

20. **Credential Setup:**  
    - Create Basic Auth credentials in n8n with username/password for API webhook  
    - Create SSH credentials for remote Docker host with proper user and authentication method  

21. **Final Connections:**  
    - Ensure all nodes are connected as described above following the command routing logic.  
    - Validate that error paths (invalid domain, SSH errors) are handled gracefully.  

22. **Workflow Settings:**  
    - Timezone set to America/Winnipeg  
    - Caller policy: workflowsFromSameOwner  
    - Execution order: v1  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Full documentation for Docker MinIO WHMCS module including API and deployment details.                                                                                                                                           | https://doc.puq.info/books/docker-minio-whmcs-module                                                           |
| WHMCS module official product page by PUQcloud with version and feature info.                                                                                                                                                     | https://puqcloud.com/whmcs-module-docker-minio.php                                                              |
| Official n8n marketplace profile to find latest workflow templates by PUQcloud.                                                                                                                                                  | https://n8n.io/creators/puqcloud/                                                                               |
| Official n8n cloud installations recommended for hosting n8n workflows.                                                                                                                                                           | https://n8n.partnerlinks.io/o692v7cg297k                                                                        |
| Workflow includes comprehensive error handling in bash scripts to return JSON errors and status messages for robust API integration.                                                                                           | Internal workflow design                                                                                         |
| SSH commands and file management require the remote server to have appropriate sudo permissions for the n8n SSH user, Docker and docker-compose installed, and network access to Nginx proxy containers.                          | Operational prerequisite                                                                                        |
| Docker Compose template is dynamically generated per API input for flexibility in container resource allocation and domain configuration.                                                                                      | Node: Deploy-docker-compose                                                                                      |
| Nginx proxy configuration supports custom headers and ACL control via editable files managed by the workflow.                                                                                                                   | Node: nginx, GET ACL, SET ACL                                                                                    |

---

This completes the structured reference document for the "Deploy Docker MinIO, API Backend for WHMCS/WISECP" n8n workflow. It enables users and AI agents to understand, reproduce, and modify the workflow with clarity and confidence.