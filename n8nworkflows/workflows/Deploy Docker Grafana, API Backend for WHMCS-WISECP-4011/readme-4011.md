Deploy Docker Grafana, API Backend for WHMCS/WISECP

https://n8nworkflows.xyz/workflows/deploy-docker-grafana--api-backend-for-whmcs-wisecp-4011


# Deploy Docker Grafana, API Backend for WHMCS/WISECP

---

# Deploy Docker Grafana, API Backend for WHMCS/WISECP - Workflow Reference Document

---

### 1. Workflow Overview

This n8n workflow automates the deployment and management of Grafana Docker containers tailored for integration with WHMCS/WISECP billing platforms. It exposes a secured API webhook to receive commands for managing Grafana Docker instances per client domain, supporting lifecycle actions such as create, start, stop, suspend, unsuspend, terminate, change package, and retrieve container information or logs.

The workflow is logically structured into these main blocks:

- **1.1 Input Reception & Validation:** Receives API requests over a secured webhook, validates the server domain, and initializes parameters.
- **1.2 Command Routing:** Routes commands to the appropriate processing switch nodes for container-level or service-level actions.
- **1.3 Container Management:** Handles Docker container lifecycle commands including start, stop, mount/unmount disks, ACL management, and network statistics.
- **1.4 Service Management:** Manages service-level commands like create, suspend, unsuspend, terminate, change package, and test connection.
- **1.5 Command Execution via SSH:** Executes bash scripts over SSH on the target server, performing Docker and filesystem operations.
- **1.6 Response Handling:** Processes command execution results and sends standardized API responses.
- **1.7 Workflow Configuration & Documentation:** Contains parameter setup, Docker-compose template generation, NGINX configuration setup, and embedded instructions.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Validation

**Overview:**  
This block receives incoming API requests via a secured webhook, sets essential parameters, and validates if the request’s server domain matches the expected domain.

**Nodes Involved:**  
- API (Webhook)  
- Parametrs (Set)  
- If (Domain Validation)  
- 422-Invalid server domain (Respond to Webhook)

**Node Details:**

- **API**  
  - Type: Webhook (HTTP POST)  
  - Role: Entry point; receives REST API requests at `/docker-grafana` path with Basic Auth.  
  - Config: Accepts POST method, Basic Auth credential named "Grafana".  
  - Input: External HTTP requests  
  - Output: JSON payload with command and domain info.  
  - Edge cases: Auth failure, incorrect HTTP method.

- **Parametrs**  
  - Type: Set  
  - Role: Defines static workflow parameters such as expected server domain, client directories, mount points, and fixed expression delimiters.  
  - Key Parameters:  
    - `server_domain`: `d01-test.uuq.pl` (must match API calls)  
    - `clients_dir`: Directory for client data `/opt/docker/clients`  
    - `mount_dir`: Default mount point `/mnt`  
    - `screen_left`, `screen_right`: Expression delimiters for inner script use (`{{` and `}}`)  
  - Input: API node output  
  - Output: Parameter JSON for use downstream.

- **If**  
  - Type: Conditional  
  - Role: Checks if the `server_domain` in API payload matches the configured `server_domain`.  
  - Condition: `$json.server_domain == $('API').item.json.body.server_domain`  
  - Inputs: Parametrs node output  
  - Outputs:  
    - True: Continues workflow  
    - False: Routes to error response node.

- **422-Invalid server domain**  
  - Type: Respond to Webhook  
  - Role: Returns HTTP 422 JSON error response when domain validation fails.  
  - Output: JSON error message `{"status":"error","error":"Invalid server domain"}`  
  - Input: If node false output  
  - Edge cases: None, standard error response.

---

#### 1.2 Command Routing

**Overview:**  
Routes validated API commands into two main categories: container-specific actions and service-level actions, based on the command string.

**Nodes Involved:**  
- Container Stat (Switch)  
- Container Actions (Switch)  
- Grafana (Switch)  
- If1 (Switch)  
- Service Actions (Switch)

**Node Details:**

- **Container Stat**  
  - Type: Switch  
  - Role: Routes container information commands like inspect, stats, and logs.  
  - Condition: Based on `command` field equal to `container_information_inspect`, `container_information_stats`, or `container_log`.

- **Container Actions**  
  - Type: Switch  
  - Role: Routes container lifecycle commands: start, stop, mount/unmount disk, get/set ACL, get net stats.  
  - Commands Supported: `container_start`, `container_stop`, `container_mount_disk`, `container_unmount_disk`, `container_get_acl`, `container_set_acl`, `container_get_net`.

- **Grafana**  
  - Type: Switch  
  - Role: Routes Grafana-specific commands: `app_version` and `change_password`.

- **If1**  
  - Type: Conditional  
  - Role: Checks if the command is one of `create`, `change_package`, or `unsuspend` to route to NGINX config node or service actions.  
  - True: Routes to `nginx` node  
  - False: Routes to `Service Actions` node.

- **Service Actions**  
  - Type: Switch  
  - Role: Handles service lifecycle commands: `test_connection`, `create`, `suspend`, `unsuspend`, `terminate`, `change_package`.

---

#### 1.3 Container Management

**Overview:**  
Executes container-level commands by generating corresponding bash scripts and running them over SSH. Operations include starting/stopping containers, mounting/unmounting disks, inspecting container stats, managing ACLs, and network statistics.

**Nodes Involved:**  
- Start (Set)  
- Stop (Set)  
- Mount Disk (Set)  
- Unmount Disk (Set)  
- GET ACL (Set)  
- SET ACL (Set)  
- GET NET (Set)  
- Inspect (Set)  
- Stat (Set)  
- Log (Set)  
- SSH (SSH Execution)  
- Code1 (Code)  
- API answer (Respond to Webhook)

**Node Details:**

- **Start / Stop / Mount Disk / Unmount Disk / GET ACL / SET ACL / GET NET / Inspect / Stat / Log**  
  - Type: Set  
  - Role: Prepare bash scripts dynamically using input parameters and templates for each container action.  
  - Scripts manage Docker compose commands, mount points, filesystem operations, NGINX config files, and data/image files.  
  - Use expressions to insert API request data and parameter values.  
  - Error handling via bash functions that write errors and exit.

- **SSH**  
  - Type: SSH  
  - Role: Executes the generated bash scripts on the remote server using SSH credentials (`d01-test.uuq.pl-puq`).  
  - Config: Runs script in root directory via password-based SSH authentication.  
  - Handles errors by continuing execution but capturing output.  
  - Input: Bash script from Set node  
  - Output: Command stdout for parsing.

- **Code1**  
  - Type: Code (JavaScript)  
  - Role: Parses SSH command output JSON or success strings and standardizes status, message, and data for API response.  
  - Handles JSON parsing errors gracefully.

- **API answer**  
  - Type: Respond to Webhook  
  - Role: Returns the final structured JSON response back to the API caller with HTTP 200.

---

#### 1.4 Service Management

**Overview:**  
Handles high-level service commands by preparing deployment scripts and configurations, including creating new containers, suspending, unsuspending, terminating, changing packages, and testing Docker environment status.

**Nodes Involved:**  
- Test Connection1 (Set)  
- Deploy (Set)  
- Suspend (Set)  
- Unsuspend (Set)  
- Terminated (Set)  
- ChangePackage (Set)  
- SSH (SSH Execution) [shared with container management]  
- Code1 (Code)  
- API answer (Respond to Webhook) [shared]

**Node Details:**

- Each Set node generates a bash script with detailed logic for the respective service action. For example:  
  - **Test Connection1:** Verifies Docker installation, service running, and critical containers (nginx-proxy and letsencrypt companion).  
  - **Deploy:** Creates directories, writes docker-compose.yml, sets permissions, creates disk image, mounts it, sets up NGINX configs, and launches containers.  
  - **Suspend:** Stops containers, unmounts disks, removes mount entries, deletes mount directories, and cleans NGINX configs, then marks status as suspended.  
  - **Unsuspend:** Recreates directories, mounts disk, restores docker-compose and NGINX configs, updates ACLs, and starts containers.  
  - **Terminated:** Cleans up all associated files, containers, mounts, and NGINX configurations.  
  - **ChangePackage:** Resizes disk image, updates docker-compose and NGINX, restarts containers.

- SSH node executes these scripts similarly to container management.

---

#### 1.5 Workflow Configuration & Template Setup

**Overview:**  
Sets template configurations and generates Docker-compose and NGINX configuration snippets dynamically based on API input.

**Nodes Involved:**  
- Parametrs (Set)  
- Deploy-docker-compose (Set)  
- nginx (Set)  
- Sticky Note (Documentation)

**Node Details:**

- **Deploy-docker-compose**  
  - Generates a docker-compose YAML string for the Grafana container with parameters: container name, image, volumes, environment variables, resource limits (RAM, CPU), and network.  
  - Uses base64 encoding for later decoding in scripts.

- **nginx**  
  - Sets NGINX proxy configuration snippets for reverse proxy headers and proxy pass settings.

- **Sticky Note**  
  - Contains detailed setup instructions, links to documentation, and credits for the workflow template.

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                                      | Input Node(s)                | Output Node(s)                     | Sticky Note                                              |
|-----------------------|----------------------------|-----------------------------------------------------|-----------------------------|----------------------------------|----------------------------------------------------------|
| API                   | Webhook                    | Entry point for API requests                         | External HTTP               | Parametrs                        |                                                          |
| Parametrs             | Set                        | Sets static parameters for domain and paths         | API                         | If                              |                                                          |
| If                    | If                         | Validates server domain consistency                  | Parametrs                   | Container Stat, Grafana, If1, 422-Invalid server domain |                                                          |
| 422-Invalid server domain | Respond to Webhook       | Returns error on invalid domain                       | If (false)                  | None                            |                                                          |
| Container Stat        | Switch                     | Routes container info commands                       | If (true)                   | Container Actions, Grafana, If1 |                                                          |
| Container Actions     | Switch                     | Routes container lifecycle commands                  | Container Stat              | Start, Stop, Mount Disk, Unmount Disk, GET ACL, SET ACL, GET NET |                                                          |
| Grafana               | Switch                     | Routes Grafana app commands                          | Container Stat              | Version, Change Password         |                                                          |
| If1                   | If                         | Routes create/change_package/unsuspend to NGINX or service actions | Container Stat           | nginx, Service Actions           |                                                          |
| Service Actions       | Switch                     | Routes service-level commands                        | If1                         | Test Connection1, Deploy, Suspend, Unsuspend, Terminated, ChangePackage |                                                          |
| Start                 | Set                        | Creates bash script to start Docker container        | Container Actions (start)   | SSH                            |                                                          |
| Stop                  | Set                        | Creates bash script to stop Docker container         | Container Actions (stop)    | SSH                            |                                                          |
| Mount Disk            | Set                        | Creates bash script to mount disk image              | Container Actions (mount_disk) | SSH                          |                                                          |
| Unmount Disk          | Set                        | Creates bash script to unmount disk image            | Container Actions (unmount_disk) | SSH                          |                                                          |
| GET ACL               | Set                        | Creates bash script to get ACL configuration          | Container Actions (container_get_acl) | SSH                       |                                                          |
| SET ACL               | Set                        | Creates bash script to set ACL configuration          | Container Actions (container_set_acl) | SSH                       |                                                          |
| GET NET               | Set                        | Creates bash script to get network usage stats        | Container Actions (container_get_net) | SSH                       |                                                          |
| Inspect               | Set                        | Creates bash script to docker inspect container       | Container Stat (inspect)    | SSH                            |                                                          |
| Stat                  | Set                        | Creates bash script to get Docker stats and disk info | Container Stat (stats)      | SSH                            |                                                          |
| Log                   | Set                        | Creates bash script to retrieve container logs        | Container Stat (log)        | SSH                            |                                                          |
| SSH                   | SSH                        | Executes generated bash scripts on remote server      | All Set nodes               | Code1                          |                                                          |
| Code1                 | Code                       | Parses SSH output and standardizes API response       | SSH                        | API answer                     |                                                          |
| API answer            | Respond to Webhook         | Sends final JSON response to API caller               | Code1                      | None                           |                                                          |
| Test Connection1      | Set                        | Checks Docker and proxy container status              | Service Actions (test_connection) | SSH                       |                                                          |
| Deploy                | Set                        | Creates deployment script for new container           | Service Actions (create)    | SSH                            |                                                          |
| Suspend               | Set                        | Creates suspend script to stop and unmount container  | Service Actions (suspend)   | SSH                            |                                                          |
| Unsuspend             | Set                        | Creates unsuspend script to remount and restart       | Service Actions (unsuspend) | SSH                            |                                                          |
| Terminated            | Set                        | Creates termination script to remove all traces       | Service Actions (terminate) | SSH                            |                                                          |
| ChangePackage         | Set                        | Creates script to resize disk and update container    | Service Actions (change_package) | SSH                         |                                                          |
| Deploy-docker-compose | Set                        | Generates docker-compose YAML configuration            | nginx                      | Service Actions                |                                                          |
| nginx                 | Set                        | Defines NGINX proxy config snippets                    | If1                        | Deploy-docker-compose          |                                                          |
| Version               | Set                        | Retrieves Grafana version from container               | Grafana (version)           | SSH                            |                                                          |
| Change Password       | Set                        | Creates script to reset Grafana admin password         | Grafana (change_password)   | SSH                            |                                                          |
| Sticky Note           | Sticky Note                | Documentation and setup instructions                   | None                       | None                           | See Setup Instructions and Official Documentation Links  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (API):**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `docker-grafana`  
   - Authentication: Basic Auth (create credential with username/password)  
   - Response Mode: Response Node  
   - Allow multiple methods: true

2. **Create Set Node (Parametrs):**  
   - Set these static parameters:  
     - `server_domain`: e.g. `d01-test.uuq.pl`  
     - `clients_dir`: `/opt/docker/clients`  
     - `mount_dir`: `/mnt`  
     - `screen_left`: `{{`  
     - `screen_right`: `}}`  
   - Connect API node output to this node.

3. **Create If Node (Domain Validation):**  
   - Condition: Check if input `server_domain` equals `API.body.server_domain` (case-sensitive, strict)  
   - Connect Parametrs node output to this If node.

4. **Create Respond to Webhook Node (422-Invalid server domain):**  
   - Response Code: 422  
   - Response Body JSON: `[{ "status": "error", "error": "Invalid server domain" }]`  
   - Connect `If` node false output to this node.

5. **Create Switch Nodes for Command Routing:**  
   - **Container Stat:** Route commands `container_information_inspect`, `container_information_stats`, `container_log`  
   - **Container Actions:** Route commands: `container_start`, `container_stop`, `container_mount_disk`, `container_unmount_disk`, `container_get_acl`, `container_set_acl`, `container_get_net`  
   - **Grafana:** Route commands `app_version`, `change_password`  
   - **If1:** Condition on commands `create`, `change_package`, `unsuspend` to route to NGINX or Service Actions  
   - **Service Actions:** Route commands `test_connection`, `create`, `suspend`, `unsuspend`, `terminate`, `change_package`

6. **Create Set Nodes for Each Container Action:**  
   - For each container command, create a Set node generating the appropriate bash script. Use templating expressions to insert parameters from API and Parametrs nodes.  
   - Scripts contain error handling functions and Docker/Filesystem commands as per node descriptions.

7. **Create Set Nodes for Each Service Action:**  
   - Similarly, create Set nodes generating bash scripts for `test_connection`, `deploy`, `suspend`, `unsuspend`, `terminate`, and `change_package`.  
   - Include the logic for directory creation, disk image management, NGINX config management, docker-compose manipulation, and container lifecycle.

8. **Create SSH Node for Command Execution:**  
   - Type: SSH  
   - Credentials: Create SSH credential with password or key for target server.  
   - Command: Use incoming `sh` string from Set nodes.  
   - Configure to continue on error to capture output.  
   - Connect all Set nodes from container and service actions to this SSH node.

9. **Create Code Node (Code1) to Parse SSH Output:**  
   - JavaScript to parse stdout JSON or success string and format standardized response with `status`, `message`, and `data`.  
   - Connect SSH node output to Code node.

10. **Create Respond to Webhook Node (API answer):**  
    - HTTP 200 response code  
    - Respond with all incoming items (standardized response)  
    - Connect Code node output here.

11. **Create Supporting Set Nodes:**  
    - **Deploy-docker-compose:** Generates base64-encoded docker-compose YAML based on API inputs.  
    - **nginx:** Generates NGINX proxy configuration snippets.  
    - Connect these to service actions as needed (e.g., deploy, change package, unsuspend).

12. **Create Sticky Note Node:**  
    - Add setup instructions, credential setup, parameter explanations, and links to documentation for user reference.

13. **Connect all nodes as per connections in the workflow:**  
    - Parametrs > If  
    - If (true) > Container Stat  
    - Container Stat > Container Actions, Grafana, If1  
    - If1 (true) > nginx > Deploy-docker-compose > Service Actions  
    - Service Actions > respective Set nodes  
    - Container Actions > respective Set nodes  
    - All Set nodes > SSH > Code1 > API answer  
    - If (false) > 422-Invalid server domain

14. **Configure Credentials:**  
    - Basic Auth credential for API webhook (username/password)  
    - SSH credential for server access (password or private key)  
    - Ensure SSH user has necessary sudo privileges for Docker and file operations.

15. **Deploy and Test:**  
    - Import or create the workflow  
    - Set credentials as above  
    - Test API calls with valid and invalid domains and commands  
    - Monitor responses and logs for error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Welcome note and setup instructions for PUQ Docker Grafana deploy template.                              | See Sticky Note node content in the workflow.                                                         |
| Documentation: Full docs on setup and usage.                                                             | https://doc.puq.info/books/docker-grafana-whmcs-module                                                 |
| WHMCS Module info and purchase link.                                                                     | https://puqcloud.com/whmcs-module-docker-grafana.php                                                  |
| Parameter `server_domain` must match exactly the domain configured in WHMCS/WISECP to authenticate API calls. | Critical for security and proper routing.                                                             |
| SSH user must have sudo privileges without password prompt or the workflow must handle sudo password entry.| To enable Docker and filesystem operations remotely.                                                  |
| Docker must be installed and running on the target server with nginx-proxy and letsencrypt-nginx-proxy-companion containers active. | Required for proper reverse proxy and SSL termination.                                                |
| Scripts use base64 encoding to safely transport multiline YAML and configuration strings in node parameters. | Ensures correct file creation on remote server.                                                       |
| Error handling in bash scripts is consistent: errors write JSON status to status files and output error messages. | Enables clear API feedback and troubleshooting.                                                      |
| This workflow is designed for integration with WHMCS/WISECP billing platforms to automate container deployment for clients. | Enterprise-grade automation for SaaS or hosting providers.                                           |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---