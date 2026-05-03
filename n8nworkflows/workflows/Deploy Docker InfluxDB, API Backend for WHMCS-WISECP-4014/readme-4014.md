Deploy Docker InfluxDB, API Backend for WHMCS/WISECP

https://n8nworkflows.xyz/workflows/deploy-docker-influxdb--api-backend-for-whmcs-wisecp-4014


# Deploy Docker InfluxDB, API Backend for WHMCS/WISECP

---

# Reference Document for n8n Workflow: Deploy Docker InfluxDB, API Backend for WHMCS/WISECP

---

### 1. Workflow Overview

This workflow automates the deployment and management of Docker-based InfluxDB instances tailored for integration with WHMCS/WISECP billing platforms. It exposes a secure API webhook enabling external commands to control Docker container lifecycle and configuration via SSH on a target server that runs Docker.

The workflow is logically divided into these main blocks:

- **1.1 API Reception & Validation**  
  Receives external API POST requests with Basic Auth, validates the server domain, and initializes parameters.

- **1.2 Command Routing**  
  Routes incoming commands into service lifecycle commands (create, suspend, unsuspend, terminate, change_package, test_connection) and container-specific commands (start, stop, mount_disk, unmount_disk, ACL and networking info).

- **1.3 Bash Script Generation & Execution via SSH**  
  Generates customized Bash scripts for each command scenario and executes them remotely via SSH to manage Docker containers, volumes, mounts, NGINX proxy configurations, and disk images.

- **1.4 Response Handling**  
  Parses command execution output, handles errors, and sends structured JSON responses back to the API caller.

- **1.5 Parameter & Configuration Management**  
  Defines and manages parameters for server domain, directories, Docker Compose configuration, and NGINX proxy settings, allowing customization of deployment templates.

---

### 2. Block-by-Block Analysis

---

#### 2.1 API Reception & Validation

- **Overview:**  
  This block exposes an authenticated HTTP POST webhook endpoint which receives API requests. It validates the server domain to ensure requests target the correct deployment server. It also sets static and configurable parameters used downstream.

- **Nodes Involved:**  
  - API (Webhook)  
  - Parametrs (Set)  
  - If (Conditional)  
  - 422-Invalid server domain (Respond to Webhook)

- **Node Details:**  
  - **API**  
    - Type: Webhook  
    - Role: Entry point for API calls, secured by Basic Auth credential named "InfluxDB"  
    - Configuration: Path `/docker-influxdb`, POST method, responds using a response node  
    - Inputs: External HTTP POST requests  
    - Outputs: Forwards JSON body to Parametrs node

  - **Parametrs**  
    - Type: Set  
    - Role: Defines static parameters like `server_domain`, `clients_dir`, `mount_dir`, and technical delimiters `screen_left` and `screen_right` for script templating  
    - Configuration: Hardcoded strings (e.g., `server_domain = d01-test.uuq.pl`)  
    - Inputs: From API node  
    - Outputs: To If node

  - **If**  
    - Type: If Condition (v2.2)  
    - Role: Validates if incoming request’s `server_domain` matches the configured domain  
    - Condition: Checks if `$('API').item.json.body.server_domain` equals `Parametrs.server_domain`  
    - Inputs: From Parametrs node  
    - Outputs: Passes to Container Stat block if true, else to 422 response

  - **422-Invalid server domain**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 422 JSON error if domain validation fails  
    - Outputs: Ends workflow execution for invalid domain requests

- **Edge Cases & Potential Failures:**  
  - Authentication failure on webhook (Basic Auth misconfiguration)  
  - Missing or malformed JSON body in API request  
  - Mismatched server domain causing 422 error response  

---

#### 2.2 Command Routing

- **Overview:**  
  Routes incoming API commands into two major switches: one for service lifecycle commands, the other for container-related actions. Also includes commands related to InfluxDB app version and password changes.

- **Nodes Involved:**  
  - Container Stat (Switch)  
  - Container Actions (Switch)  
  - Service Actions (Switch)  
  - InfluxDB (Switch)  
  - If1 (If condition)

- **Node Details:**  
  - **Container Stat**  
    - Type: Switch  
    - Role: Routes container info commands (`container_information_inspect`, `container_information_stats`, `container_log`)  
    - Inputs: From If node (valid requests)  
    - Outputs: To respective nodes for inspect/stat/log

  - **Container Actions**  
    - Type: Switch  
    - Role: Routes container commands like start, stop, mount/unmount disk, ACL get/set, get network stats  
    - Inputs: From Container Stat node  
    - Outputs: To respective Bash script nodes

  - **Service Actions**  
    - Type: Switch  
    - Role: Routes service lifecycle commands like create, suspend, unsuspend, terminate, change_package, test_connection  
    - Inputs: From If1 node (conditional on command being create/change_package/unsuspend) or If node in alternate path  
    - Outputs: To Bash script nodes for each action

  - **InfluxDB**  
    - Type: Switch  
    - Role: Routes InfluxDB-specific commands for app version and password change  
    - Inputs: From Container Stat node  
    - Outputs: To Version or Change Password nodes

  - **If1**  
    - Type: If  
    - Role: Determines if command is `create`, `change_package`, or `unsuspend` to route to nginx configuration assignment or service actions  
    - Inputs: From Container Stat node  
    - Outputs: To nginx node or Service Actions node

- **Edge Cases & Potential Failures:**  
  - Unrecognized or unsupported command in API request  
  - Missing command attribute in JSON body  
  - Command routing errors if expressions fail

---

#### 2.3 Bash Script Generation & SSH Execution

- **Overview:**  
  For each routed command, this block generates a custom Bash script embedding parameters from the API request and the workflow’s configuration. Scripts handle Docker container lifecycle, volume and disk mount management, NGINX proxy configuration, and InfluxDB internal commands. Scripts execute remotely via SSH on the target Docker server.

- **Nodes Involved:**  
  - SSH (Single node reused for all commands)  
  - Multiple Set nodes generating scripts for each command:  
    - Deploy  
    - Suspend  
    - Unsuspend  
    - Terminated  
    - ChangePackage  
    - Start  
    - Stop  
    - Mount Disk  
    - Unmount Disk  
    - GET ACL  
    - SET ACL  
    - GET NET  
    - Inspect  
    - Stat  
    - Log  
    - Test Connection1  
    - Version  
    - Change Password  
  - Code1 (JavaScript)  
  - API answer (Respond to Webhook)

- **Node Details:**  
  - **Set nodes for scripts**  
    - Type: Set  
    - Role: Prepare Bash script strings with embedded parameters from API and Parametrs nodes using mustache-like expressions `{{ }}`  
    - Scripts include error handling functions, docker-compose commands, disk image management, NGINX configuration file updates, mount/unmount operations, and container checks  
    - Outputs: Script text as `sh` parameter

  - **SSH**  
    - Type: SSH node  
    - Role: Executes the prepared Bash script on the remote server  
    - Configuration: Uses SSH credentials named for the target server (e.g., `d01-test.uuq.pl-puq`)  
    - Input: Bash script string from Set nodes  
    - Outputs: Command stdout, stderr, or error  
    - On error: configured to continue with error output passed downstream

  - **Code1**  
    - Type: JavaScript Code  
    - Role: Parses the SSH command output, interprets JSON or plain success strings, and formats the final JSON response with fields: status, message, data  
    - Handles exceptions gracefully to return error messages

  - **API answer**  
    - Type: Respond to Webhook  
    - Role: Sends the final JSON response with HTTP 200 status back to the API caller  
    - Inputs: From Code1 node

- **Version Requirements:**  
  - SSH node version 1+ with error output continuation  
  - Code node version 2+ for JavaScript scripting  
  - Use of Docker Compose v2 syntax in Bash scripts assumed

- **Potential Failures & Edge Cases:**  
  - SSH authentication or connectivity failure  
  - Bash script execution failures (permissions, command errors)  
  - Docker service or container state issues (missing containers, stopped services)  
  - File system permissions and mount errors  
  - JSON parse errors in Code1 node if script output malformed  
  - Concurrent execution conflicts if multiple requests target same domain/container  

---

#### 2.4 Parameter & Configuration Management

- **Overview:**  
  This block manages configurable parameters that define deployment specifics such as server domain, client directories, disk mount paths, and template snippets for Docker Compose and NGINX proxy configuration. It allows administrators to customize deployment without modifying core scripts.

- **Nodes Involved:**  
  - Parametrs (Set)  
  - Deploy-docker-compose (Set)  
  - nginx (Set)

- **Node Details:**  
  - **Parametrs**  
    - Holds base parameters like `server_domain`, `clients_dir`, `mount_dir`, and technical delimiters `screen_left`/`screen_right` used for escaping in scripts

  - **Deploy-docker-compose**  
    - Defines the Docker Compose YAML for the InfluxDB service as a template string with variables replaced by API inputs like domain, username, RAM, CPU  
    - Uses base64 encoding to safely pass multiline YAML to Bash scripts

  - **nginx**  
    - Holds NGINX proxy server configuration snippets:  
      - `main`: parameters for the server block (empty default)  
      - `main_location`: directives added to the location `/` block (headers etc.)

- **Edge Cases:**  
  - Improper parameter changes may break script execution or Docker deployment  
  - Incorrect domain names or directory paths cause failures downstream  
  - Base64 encoding/decoding errors if template contains unsupported characters  

---

#### 2.5 Documentation & Notes

- **Overview:**  
  A sticky note node provides human-readable setup instructions and links to external documentation and module pages for ease of use by administrators.

- **Node Involved:**  
  - Sticky Note

- **Details:**  
  Contains setup instructions, credential creation guides, parameter modification advice, and links:  
  - Documentation: https://doc.puq.info/books/docker-influxdb-whmcs-module  
  - WHMCS module: https://puqcloud.com/whmcs-module-docker-influxdb.php  
  - n8n Marketplace profile: https://n8n.io/creators/puqcloud/

---

### 3. Summary Table

| Node Name              | Node Type                | Functional Role                                     | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                       |
|------------------------|--------------------------|----------------------------------------------------|------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| API                    | Webhook                  | API entry point with Basic Auth                     | External HTTP request  | Parametrs                   |                                                                                                |
| Parametrs              | Set                      | Defines server and directory parameters             | API                    | If                          |                                                                                                |
| If                     | If Condition             | Validates server domain                              | Parametrs               | Container Stat, 422 Invalid  |                                                                                                |
| 422-Invalid server domain | Respond to Webhook       | Responds with error if domain invalid                | If                     | None                        |                                                                                                |
| Container Stat         | Switch                   | Routes container info commands                       | If                      | Container Actions, InfluxDB, If1 |                                                                                                |
| Container Actions      | Switch                   | Routes container management commands                 | Container Stat           | Multiple Bash script sets    |                                                                                                |
| Service Actions        | Switch                   | Routes service lifecycle commands                    | If1                      | Multiple Bash script sets    |                                                                                                |
| InfluxDB               | Switch                   | Routes InfluxDB-specific commands                    | Container Stat           | Version, Change Password    |                                                                                                |
| If1                    | If Condition             | Routes create/change_package/unsuspend commands      | Container Stat           | nginx, Service Actions      |                                                                                                |
| Deploy-docker-compose  | Set                      | Docker Compose template                              | nginx                    | Service Actions             |                                                                                                |
| nginx                  | Set                      | NGINX proxy config templates                         | If1                      | Deploy-docker-compose       |                                                                                                |
| Deploy                 | Set                      | Bash script to create and deploy containers          | Service Actions          | SSH                         |                                                                                                |
| Suspend                | Set                      | Bash script to suspend (stop & clean) container      | Service Actions          | SSH                         |                                                                                                |
| Unsuspend              | Set                      | Bash script to unsuspend (restart & remount) container| Service Actions         | SSH                         |                                                                                                |
| Terminated             | Set                      | Bash script to terminate and clean all resources     | Service Actions          | SSH                         |                                                                                                |
| ChangePackage          | Set                      | Bash script to change package and resize disk        | Service Actions          | SSH                         |                                                                                                |
| Start                  | Set                      | Bash script to start container                        | Container Actions        | SSH                         |                                                                                                |
| Stop                   | Set                      | Bash script to stop container                         | Container Actions        | SSH                         |                                                                                                |
| Mount Disk             | Set                      | Bash script to mount disk image                       | Container Actions        | SSH                         |                                                                                                |
| Unmount Disk           | Set                      | Bash script to unmount disk image                     | Container Actions        | SSH                         |                                                                                                |
| GET ACL                | Set                      | Bash script to get NGINX ACL                          | Container Actions        | SSH                         |                                                                                                |
| SET ACL                | Set                      | Bash script to set NGINX ACL and reload NGINX        | Container Actions        | SSH                         |                                                                                                |
| GET NET                | Set                      | Bash script to get container network stats           | Container Actions        | SSH                         |                                                                                                |
| Inspect                | Set                      | Bash script to inspect container details              | Container Stat           | SSH                         |                                                                                                |
| Stat                   | Set                      | Bash script to get container stats and disk info     | Container Stat           | SSH                         |                                                                                                |
| Log                    | Set                      | Bash script to get container logs                      | Container Stat           | SSH                         |                                                                                                |
| Test Connection1       | Set                      | Bash script to test Docker and proxy container status| Service Actions          | SSH                         |                                                                                                |
| Version                | Set                      | Bash script to get InfluxDB container version         | InfluxDB                 | SSH                         |                                                                                                |
| Change Password        | Set                      | Bash script to change InfluxDB user password          | InfluxDB                 | SSH                         |                                                                                                |
| SSH                    | SSH                      | Executes Bash scripts on remote server                 | Multiple Set nodes       | Code1                       |                                                                                                |
| Code1                  | Code                     | Parses SSH output and formats API response            | SSH                      | API answer                  |                                                                                                |
| API answer             | Respond to Webhook       | Sends final JSON response to API caller                | Code1                    | None                        |                                                                                                |
| Sticky Note            | Sticky Note              | Provides setup instructions and links                 | None                     | None                        | ## 👋 Welcome to PUQ Docker InfluxDB deploy! Setup guide and external documentation links.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("API")**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `docker-influxdb`  
   - Authentication: Basic Auth with credential "InfluxDB"  
   - Response Mode: Use Response Node  
   - Save and connect to next node.

2. **Create Set Node ("Parametrs")**  
   - Define parameters:  
     - `server_domain` = `"d01-test.uuq.pl"` (replace with your domain)  
     - `clients_dir` = `"/opt/docker/clients"`  
     - `mount_dir` = `"/mnt"`  
     - `screen_left` = `"{{"`  
     - `screen_right` = `"}}"`  
   - Connect from API node.

3. **Create If Node ("If")**  
   - Condition: Check if `{{$json.server_domain}} == {{$json.body.server_domain}}` where left is Parametrs `server_domain` and right is `API.body.server_domain`  
   - True path leads to Container Stat block, False path to Respond 422 node.

4. **Create Respond to Webhook Node ("422-Invalid server domain")**  
   - Response code: 422  
   - Response body: JSON error message `{ "status": "error", "error": "Invalid server domain" }`  
   - Connect from If node False output.

5. **Create Switch Node ("Container Stat")**  
   - Rules matching `API.body.command`:  
     - `container_information_inspect` -> output "inspect"  
     - `container_information_stats` -> output "stats"  
     - `container_log` -> output "log"  
   - Connect from If node True output.

6. **Create Switch Node ("Container Actions")**  
   - Rules matching `API.body.command`: `container_start`, `container_stop`, `container_mount_disk`, `container_unmount_disk`, `container_get_acl`, `container_set_acl`, `container_get_net`  
   - Connect from Container Stat.

7. **Create Switch Node ("Service Actions")**  
   - Rules matching `API.body.command`: `test_connection`, `create`, `suspend`, `unsuspend`, `terminate`, `change_package`  
   - Connect from If1 node (created next).

8. **Create If Node ("If1")**  
   - Condition: `API.body.command` in `[create, change_package, unsuspend]`  
   - True path to nginx Set node, else to Service Actions  
   - Connect from Container Stat.

9. **Create Set Node ("nginx")**  
   - Assign:  
     - `main` = empty string  
     - `main_location` =  
       ```
       proxy_pass_header Server;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Scheme $scheme;
       proxy_set_header Host $http_host;
       ```
   - Connect from If1 True output.

10. **Create Set Nodes for Bash Scripts** (Deploy, Suspend, Unsuspend, Terminated, ChangePackage, Start, Stop, Mount Disk, Unmount Disk, GET ACL, SET ACL, GET NET, Inspect, Stat, Log, Test Connection1, Version, Change Password)  
    - Each node defines a Bash script string in `sh` parameter embedding variables from API and Parametrs nodes.  
    - Copy script contents as specified in workflow, ensuring error handling and base64 encoding where needed.

11. **Create SSH Node ("SSH")**  
    - Credential: SSH credential configured for target Docker server  
    - Command: `={{ $json.sh }}` (executes embedded script)  
    - On error: continue with error output  
    - Connect all Bash script Set nodes to this SSH node.

12. **Create Code Node ("Code1")**  
    - JavaScript to parse SSH output, check for success strings or parse JSON, and create final JSON with status, message, and data fields.  
    - Connect from SSH node.

13. **Create Respond to Webhook Node ("API answer")**  
    - Responds with HTTP 200 and all incoming items as JSON response.  
    - Connect from Code1 node.

14. **Create Sticky Note**  
    - Add instructions, links, and usage notes as per the existing sticky note content.

15. **Connect all nodes as per defined connections in the workflow** ensuring proper paths for each command flow.

16. **Configure Credentials**  
    - Create Basic Auth credential for API webhook with username and password.  
    - Create SSH credential with key or password access to the Docker server.

17. **Set Timezone and Execution Settings**  
    - Set workflow timezone to America/Winnipeg (or as needed).  
    - Caller policy: workflowsFromSameOwner  
    - Execution order: sequential (`v1`)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                          | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Full documentation and setup instructions for the Docker InfluxDB WHMCS module are available online.                                                                                                                                                                                                | https://doc.puq.info/books/docker-influxdb-whmcs-module                                         |
| The WHMCS module for Docker InfluxDB by PUQcloud with additional information and purchase details.                                                                                                                                                                                                   | https://puqcloud.com/whmcs-module-docker-influxdb.php                                           |
| Official n8n Marketplace profile of PUQcloud providing latest workflow templates.                                                                                                                                                                                                                     | https://n8n.io/creators/puqcloud/                                                               |
| Workflow requires an accessible SSH server with Docker and Docker Compose installed and configured correctly.                                                                                                                                                                                        | N/A                                                                                             |
| The workflow uses advanced Bash scripting with Docker CLI and NGINX proxy management, requiring users to have adequate permissions and understanding of Linux server administration to safely modify.                                                                                                | N/A                                                                                             |

---

This document provides a comprehensive and precise view of the workflow architecture, nodes, configurations, failure modes, and reproduction instructions to facilitate deployment, modification, or integration by advanced users or automation tools.