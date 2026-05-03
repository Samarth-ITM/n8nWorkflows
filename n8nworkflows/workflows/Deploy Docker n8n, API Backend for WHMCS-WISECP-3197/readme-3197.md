Deploy Docker n8n, API Backend for WHMCS/WISECP

https://n8nworkflows.xyz/workflows/deploy-docker-n8n--api-backend-for-whmcs-wisecp-3197


# Deploy Docker n8n, API Backend for WHMCS/WISECP

### 1. Workflow Overview

This n8n workflow serves as a **Docker deployment API backend** designed for integration with WHMCS/WISECP modules. It automates managing Docker containers for client services by exposing an authenticated API webhook that accepts commands, validates them, and executes corresponding Docker-related operations via SSH on a remote server.

The workflow logically decomposes into the following blocks:

- **1.1 API Reception and Validation**: Receives HTTP POST requests with Basic Auth, extracts parameters, and validates the target server domain.
- **1.2 Command Routing and Dispatching**: Depending on the command received, routes the request through nested decision structures to service-level or container-level actions.
- **1.3 Script Generation and Execution via SSH**: Constructs Bash scripts dynamically using template parameters, runs these scripts remotely via SSH to control Docker containers and file system mounts.
- **1.4 Result Parsing and Response Delivery**: Parses command output from SSH execution, formats JSON responses, and returns them via the API.
- **1.5 Configuration Parameter Management**: Holds configurable parameters (such as server domain, directories, Docker Compose and nginx configurations) allowing customization without code changes.
- **1.6 Ancillary Operations**: Includes commands for querying container stats, logs, ACLs, network usage, and managing user data and password changes within containers.

---

### 2. Block-by-Block Analysis

#### 2.1 API Reception and Validation

**Overview:**  
This block receives incoming API requests on a secured webhook, extracts JSON body parameters, sets global parameters, and validates the server domain before allowing further processing.

**Nodes Involved:**  
- API (Webhook)  
- Parametrs (Set)  
- If (Condition)  
- 422-Invalid server domain (Respond)

**Node Details:**

- **API (Webhook)**  
  - Type: Webhook  
  - Role: Entry point; receives POST requests at path `docker-n8n` with Basic Auth security  
  - Config: Basic Auth credential required; responds only after downstream processing  
  - Input: HTTP request body JSON  
  - Output: JSON body forwarded to Parametrs node  
  - Edge cases: Unauthorized access if credentials invalid; malformed JSON bodies

- **Parametrs (Set)**  
  - Type: Set  
  - Role: Defines static parameters such as `server_domain`, `clients_dir`, `mount_dir`, and template syntax (`screen_left`, `screen_right`) for use in scripts  
  - Config: Hardcoded parameter values (e.g., `server_domain` = `d01-test.uuq.pl`)  
  - Input: Data from API node  
  - Output: Combined data for validation

- **If (Condition)**  
  - Type: If  
  - Role: Checks if the `server_domain` from parameters matches the `server_domain` in API request body  
  - Config: String equality condition, case sensitive, strict type  
  - Input: Merged JSON from Parametrs and API node  
  - Output: Routes to next logical blocks on success, or error response on failure  
  - Edge cases: Domain mismatch triggers error response

- **422-Invalid server domain (Respond to Webhook)**  
  - Type: Respond to Webhook  
  - Role: Returns HTTP 422 with an error JSON if domain validation fails  
  - Input: Triggered by failed If condition  
  - Output: HTTP response with error message  
  - Edge cases: Early termination of workflow with error code

---

#### 2.2 Command Routing and Dispatching

**Overview:**  
Routes incoming commands into two main command groups: service-level actions (e.g., create, suspend) and container-level actions (e.g., start, stop, mount disk). Also handles special commands for version info, user management, and password changes.

**Nodes Involved:**  
- If1 (If)  
- Service Actions (Switch)  
- Container Stats (Switch)  
- Container Actions (Switch)  
- n8n (Switch)

**Node Details:**

- **If1 (If Condition)**  
  - Type: If  
  - Role: Checks if command is one of `create`, `change_package`, or `unsuspend` to route to "nginx" config block or service actions  
  - Input: API request command  
  - Output: Routes accordingly  
  - Edge cases: Commands outside this list proceed to service actions node

- **Service Actions (Switch)**  
  - Type: Switch  
  - Role: Routes service-level commands: `test_connection`, `create`, `suspend`, `unsuspend`, `terminate`, `change_package`  
  - Input: API command string  
  - Output: Passes to corresponding script generation nodes  
  - Edge cases: Unrecognized commands not matched here

- **Container Stats (Switch)**  
  - Type: Switch  
  - Role: Routes container-info commands: `container_information_inspect`, `container_information_stats`, `container_log`  
  - Input: API command string  
  - Output: Routes to corresponding info nodes  
  - Edge cases: Invalid or unknown commands ignored

- **Container Actions (Switch)**  
  - Type: Switch  
  - Role: Routes container control commands: `container_start`, `container_stop`, `container_mount_disk`, `container_unmount_disk`, `container_get_acl`, `container_set_acl`, `container_get_net`  
  - Edge cases: Commands not recognized here are ignored

- **n8n (Switch)**  
  - Type: Switch  
  - Role: Handles special commands related to the n8n application: `app_version`, `app_users`, `change_password`  
  - Edge cases: Commands outside these trigger no action here

---

#### 2.3 Script Generation and Execution via SSH

**Overview:**  
Generates custom Bash scripts for each command using template parameters and API data, then executes scripts remotely via SSH on the targeted Docker server. Scripts manage Docker containers, filesystem mounts, nginx configuration files, and user database operations.

**Nodes Involved:**  
- Multiple `Set` nodes (Deploy, Start, Stop, Suspend, Unsuspend, Terminated, ChangePackage, Test Connection1, Mount Disk, Unmount Disk, GET ACL, SET ACL, GET NET, Inspect, Stat, Log, Version, Users, Change Password)  
- SSH  
- Code1 (Code node for parsing SSH output)

**Node Details:**

- **Set nodes for each operation**  
  - Type: Set  
  - Role: Generate Bash script text dynamically using Mustache-style templating from parameters and API input  
  - Scripts include error handling functions, directory and file checks, Docker commands (`docker-compose up/down`, `docker inspect`, `docker stats`, `docker logs`), mount management (`fstab`, `mount`, `umount`), nginx config management, and SQLite database user operations  
  - Edge cases: Scripts include error detection and output JSON-formatted errors; potential failures include permission errors, Docker not running, missing files, mount failures

- **SSH**  
  - Type: SSH  
  - Role: Executes the generated Bash scripts on the remote Docker server  
  - Config: Uses configured SSH credential with password authentication; runs script in root `/` directory  
  - Input: Script text from Set nodes  
  - Output: Script stdout as JSON or string  
  - Edge cases: SSH connectivity issues, auth errors, script runtime errors; configured to continue on error output

- **Code1 (Code node)**  
  - Type: Code  
  - Role: Parses SSH output; if output is literal `"success"`, returns success JSON; otherwise attempts to parse JSON response from script; on parse failure returns error status with raw output  
  - Input: SSH output JSON string  
  - Output: Standardized JSON response for API answer

---

#### 2.4 Result Parsing and Response Delivery

**Overview:**  
Processes parsed command results and sends HTTP responses back to API clients for each request.

**Nodes Involved:**  
- API answer (Respond to Webhook)

**Node Details:**

- **API answer**  
  - Type: Respond to Webhook  
  - Role: Sends final HTTP 200 response with JSON data containing command execution results, status, messages, and data payloads  
  - Input: Parsed output from Code1 node  
  - Output: HTTP response to API caller  
  - Edge cases: Always outputs data, ensuring client receives a response even on errors

---

#### 2.5 Configuration Parameter Management

**Overview:**  
Defines static and dynamic configuration parameters used across the workflow for path, domain settings, Docker Compose, and nginx proxy configurations.

**Nodes Involved:**  
- Parametrs (Set)  
- Deploy-docker-compose (Set)  
- nginx (Set)

**Node Details:**

- **Parametrs**  
  - Holds server domain, directories for clients and mounts, and template syntax variables for script generation.

- **Deploy-docker-compose**  
  - Contains template for the Docker Compose YAML configuration as a string; dynamically inserts domain, resource limits, and volumes.

- **nginx**  
  - Contains main and main_location configuration snippets for the nginx proxy server, allowing adding custom directives and headers.

---

#### 2.6 Ancillary Operations

**Overview:**  
Handles additional utility commands like retrieving container information, logs, network usage, user lists, and changing user passwords within the container's SQLite database.

**Nodes Involved:**  
- Inspect (Set)  
- Stat (Set)  
- Log (Set)  
- GET ACL (Set)  
- SET ACL (Set)  
- GET NET (Set)  
- Users (Set)  
- Change Password (Set)

**Node Details:**

- These nodes create specific Bash scripts that query Docker, filesystem, or SQLite DB states, format the results as JSON, and return them for API consumption.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                               | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                                              |
|-------------------------|--------------------|-----------------------------------------------|------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| API                     | Webhook            | Receive API requests with Basic Auth          | -                      | Parametrs                   |                                                                                                                                          |
| Parametrs               | Set                | Set static parameters for domain and paths    | API                    | If                          |                                                                                                                                          |
| If                      | If                 | Validate server domain                         | Parametrs               | Container Stats, Container Actions, n8n, If1 / 422-Invalid server domain |                                                                                                                                          |
| 422-Invalid server domain| Respond to Webhook | Return error response on invalid domain       | If                     | -                           |                                                                                                                                          |
| Container Stats         | Switch             | Route container info commands                  | If                     | Inspect, Stat, Log           |                                                                                                                                          |
| Container Actions       | Switch             | Route container control commands               | If                     | Start, Stop, Mount Disk, Unmount Disk, GET ACL, SET ACL, GET NET |                                                                                                                                          |
| n8n                     | Switch             | Route n8n app-related commands                 | If                     | Version, Users, Change Password |                                                                                                                                          |
| If1                     | If                 | Route commands between nginx config or service actions | If                     | nginx, Service Actions       |                                                                                                                                          |
| nginx                   | Set                | Set nginx proxy configuration snippets        | If1                     | Deploy-docker-compose        |                                                                                                                                          |
| Deploy-docker-compose   | Set                | Set docker-compose YAML template               | nginx                   | Service Actions             |                                                                                                                                          |
| Service Actions         | Switch             | Route service-level commands                    | If1                     | Test Connection1, Deploy, Suspend, Unsuspend, Terminated, ChangePackage |                                                                                                                                          |
| Test Connection1        | Set                | Bash script to test Docker and nginx-proxy status | Service Actions         | SSH                         |                                                                                                                                          |
| Deploy                  | Set                | Bash script to create and start Docker service | Service Actions         | SSH                         |                                                                                                                                          |
| Suspend                 | Set                | Bash script to stop services and unmount disks | Service Actions         | SSH                         |                                                                                                                                          |
| Unsuspend               | Set                | Bash script to remount disks and start service | Service Actions         | SSH                         |                                                                                                                                          |
| Terminated              | Set                | Bash script to completely remove service files | Service Actions         | SSH                         |                                                                                                                                          |
| ChangePackage           | Set                | Bash script to update service package and resize disk | Service Actions         | SSH                         |                                                                                                                                          |
| Start                   | Set                | Bash script to start Docker container          | Container Actions       | SSH                         |                                                                                                                                          |
| Stop                    | Set                | Bash script to stop Docker container           | Container Actions       | SSH                         |                                                                                                                                          |
| Mount Disk              | Set                | Bash script to mount disk image                 | Container Actions       | SSH                         |                                                                                                                                          |
| Unmount Disk            | Set                | Bash script to unmount disk image               | Container Actions       | SSH                         |                                                                                                                                          |
| GET ACL                 | Set                | Bash script to read ACL IPs from nginx config  | Container Actions       | SSH                         |                                                                                                                                          |
| SET ACL                 | Set                | Bash script to write ACL IPs and reload nginx  | Container Actions       | SSH                         |                                                                                                                                          |
| GET NET                 | Set                | Bash script to get network usage stats          | Container Actions       | SSH                         |                                                                                                                                          |
| Inspect                 | Set                | Bash script to get container inspection info   | Container Stats         | SSH                         |                                                                                                                                          |
| Stat                    | Set                | Bash script to get container stats and disk info | Container Stats         | SSH                         |                                                                                                                                          |
| Log                     | Set                | Bash script to get container logs               | Container Stats         | SSH                         |                                                                                                                                          |
| Version                 | Set                | Bash script to get n8n container version        | n8n                     | SSH                         |                                                                                                                                          |
| Users                   | Set                | Bash script to get list of users from SQLite DB | n8n                     | SSH                         |                                                                                                                                          |
| Change Password         | Set                | Bash script to change user password in SQLite DB| n8n                     | SSH                         |                                                                                                                                          |
| SSH                     | SSH                | Executes Bash scripts remotely                    | Multiple Set nodes      | Code1                       |                                                                                                                                          |
| Code1                   | Code               | Parses SSH output and formats API response       | SSH                     | API answer                  |                                                                                                                                          |
| API answer              | Respond to Webhook | Sends final API response                          | Code1                   | -                           |                                                                                                                                          |
| Sticky Note             | Sticky Note        | Documentation and setup instructions             | -                      | -                           | ## 👋 Welcome to PUQ Docker n8n deploy! Template for n8n: API Backend for WHMCS/WISECP by PUQcloud v.1 Setup instructions and resources. [https://doc.puq.info/books/docker-n8n-whmcs-module/page/setting-up-n8n-workflow](https://doc.puq.info/books/docker-n8n-whmcs-module/page/setting-up-n8n-workflow) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node ("API")**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `docker-n8n`  
   - Authentication: Basic Auth (configure credential with username/password)  
   - Response Mode: Respond with a node (defer response until later)  

2. **Create a Set Node ("Parametrs")**  
   - Define variables:  
     - `server_domain` = your WHMCS/WISECP Docker server domain (e.g., `d01-test.uuq.pl`)  
     - `clients_dir` = path for client data storage (e.g., `/opt/docker/clients`)  
     - `mount_dir` = mount point for container disks (e.g., `/mnt`)  
     - `screen_left` = `{{`  
     - `screen_right` = `}}`  
   - Connect API -> Parametrs  

3. **Create an If Node ("If")**  
   - Condition: Check if `server_domain` param equals `server_domain` from API body  
   - If true: proceed to command routing  
   - If false: send error response  
   - Connect Parametrs -> If  

4. **Create a Respond to Webhook Node ("422-Invalid server domain")**  
   - Response Code: 422  
   - Response Body: JSON error message for invalid domain  
   - Connect If (false output) -> 422-Invalid server domain  

5. **Create a Switch Node ("Container Stats")**  
   - Rules for commands:  
     - `container_information_inspect` -> Inspect  
     - `container_information_stats` -> Stat  
     - `container_log` -> Log  
   - Connect If (true output) -> Container Stats  

6. **Create a Switch Node ("Container Actions")**  
   - Rules for commands:  
     - `container_start` -> Start  
     - `container_stop` -> Stop  
     - `container_mount_disk` -> Mount Disk  
     - `container_unmount_disk` -> Unmount Disk  
     - `container_get_acl` -> GET ACL  
     - `container_set_acl` -> SET ACL  
     - `container_get_net` -> GET NET  
   - Connect If (true output) -> Container Actions  

7. **Create a Switch Node ("n8n")**  
   - Rules for commands:  
     - `app_version` -> Version  
     - `app_users` -> Users  
     - `change_password` -> Change Password  
   - Connect If (true output) -> n8n  

8. **Create an If Node ("If1")**  
   - Condition: Check if command is one of `create`, `change_package`, `unsuspend`  
   - True output: nginx (Set)  
   - False output: Service Actions (Switch)  
   - Connect If (true output) -> If1  

9. **Create a Set Node ("nginx")**  
   - Variables:  
     - `main`: nginx server block custom params (e.g., proxy settings)  
     - `main_location`: root location custom headers (initially empty or with comment)  
   - Connect If1 (true) -> nginx  

10. **Create a Set Node ("Deploy-docker-compose")**  
    - Variable: `docker-compose` string template with dynamic placeholders for domain, volumes, resources  
    - Connect nginx -> Deploy-docker-compose  

11. **Create a Switch Node ("Service Actions")**  
    - Rules for commands:  
      - `test_connection` -> Test Connection1  
      - `create` -> Deploy  
      - `suspend` -> Suspend  
      - `unsuspend` -> Unsuspend  
      - `terminate` -> Terminated  
      - `change_package` -> ChangePackage  
    - Connect If1 (false) -> Service Actions  
    - Connect Deploy-docker-compose -> Service Actions  

12. **Create a Set Node for each command (e.g., "Deploy", "Start", "Stop", "Suspend", "Unsuspend", "Terminated", "ChangePackage", "Test Connection1", "Mount Disk", "Unmount Disk", "GET ACL", "SET ACL", "GET NET", "Inspect", "Stat", "Log", "Version", "Users", "Change Password")**  
    - Each Set node contains a Bash script in `sh` variable using Mustache template syntax to insert parameters and API inputs.  
    - Scripts include error handling, Docker commands, mounting logic, nginx config management, SQLite DB commands, etc.  
    - Connect the appropriate Switch outputs to these nodes.  

13. **Create a single SSH node ("SSH")**  
    - Credential: SSH with password or key for your Docker server  
    - Command: Execute the script from upstream Set nodes (`={{ $json.sh }}`)  
    - On error: Continue with error output  
    - Connect all Set nodes from step 12 to SSH node  

14. **Create a Code Node ("Code1")**  
    - Parse the SSH output:  
      - If output is "success", return JSON with status success  
      - Else try to parse JSON from output, if fail return error JSON with raw output  
    - Connect SSH -> Code1  

15. **Create Respond to Webhook Node ("API answer")**  
    - Response Code: 200  
    - Respond with all incoming items (the parsed JSON)  
    - Connect Code1 -> API answer  

16. **Add Sticky Note Node**  
    - Contains workflow instructions, setup notes, links to documentation and module site  
    - Position anywhere as documentation  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| **Setup prerequisites:** Requires an n8n instance with Basic Auth and SSH credentials configured. | See official n8n documentation for credential setup. |
| **Docker and nginx proxy assumptions:** The remote server must have Docker, Docker Compose, and an nginx-proxy setup with letsencrypt companion running. | See [PUQcloud Docker N8N WHMCS Module Docs](https://doc.puq.info/books/docker-n8n-whmcs-module/page/setting-up-n8n-workflow) |
| **Scripts use Bash with error handling:** Scripts are robust to common errors like missing files, permission issues, or container not running. | Can be customized per environment needs. |
| **API commands include service lifecycle and container management:** Supports create, suspend, unsuspend, terminate, start, stop, mount/unmount disk, ACL and net stats operations. | Useful for full container lifecycle automation. |
| **SQLite user management:** For containers running n8n, user data and passwords are handled via SQLite DB inside mounted volume, accessed remotely. | Passwords hashed using bcrypt via `htpasswd`. |
| **Links:**  
- [n8n Marketplace PUQcloud Templates](https://n8n.io/creators/puqcloud/)  
- [PUQcloud WHMCS Docker Module](https://puqcloud.com/whmcs-module-docker-n8n.php)  
- [Workflow Docs](https://doc.puq.info/books/docker-n8n-whmcs-module/page/setting-up-n8n-workflow) | These links provide additional setup help and module downloads. |

---

This reference fully describes the workflow structure, logic, and configuration, empowering users and automation agents to understand, reproduce, and safely modify the workflow in their n8n environments.