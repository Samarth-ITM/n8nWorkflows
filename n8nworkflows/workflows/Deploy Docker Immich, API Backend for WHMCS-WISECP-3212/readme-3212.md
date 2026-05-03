Deploy Docker Immich, API Backend for WHMCS/WISECP

https://n8nworkflows.xyz/workflows/deploy-docker-immich--api-backend-for-whmcs-wisecp-3212


# Deploy Docker Immich, API Backend for WHMCS/WISECP

### 1. Workflow Overview

This workflow automates the deployment and management of the Docker Immich server via an API backend designed for integration with WHMCS/WISECP billing systems. It exposes a secured webhook API to receive commands and executes corresponding Docker container management operations on a remote server via SSH. The workflow is structured into logical blocks that handle API reception, parameter validation, command routing, Docker and NGINX configuration management, and command execution with error handling.

Logical blocks:

- **1.1 Input Reception and Validation**: Receive API requests via webhook and validate the target server domain.
- **1.2 Parameter Initialization**: Define and manage operational parameters like directories and mount points.
- **1.3 Command Routing**: Direct received API commands to appropriate logical paths for container or service actions.
- **1.4 Docker and NGINX Configuration Generation**: Dynamically generate Docker Compose and NGINX configuration files based on API inputs.
- **1.5 SSH Command Execution**: Execute bash scripts remotely via SSH to perform Docker container lifecycle management and system operations.
- **1.6 Response Handling**: Process execution results and respond to the API caller with structured success or error messages.
- **1.7 Error Handling and Logging**: Manage error conditions gracefully for authentication, execution failures, or invalid inputs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
  This block handles incoming HTTP POST requests to the `/docker-immich` webhook, authenticates with Basic Auth, and validates the `server_domain` in the request body against an internal parameter. It ensures requests are intended for the correct server domain.

- **Nodes Involved:**  
  - `API` (Webhook)  
  - `Parametrs` (Set node with server parameters)  
  - `If` (Domain validation)  
  - `422-Invalid server domain` (Webhook response node)

- **Node Details:**

  - **API (Webhook)**
    - Type: Webhook  
    - Role: Receives API calls with Basic Auth authentication.  
    - Configuration: POST method, path `/docker-immich`, Basic Auth credential named "Immich".  
    - Inputs: External HTTP requests  
    - Outputs: Request data JSON passed to `Parametrs`.  
    - Failure modes: Auth failures, malformed requests.

  - **Parametrs (Set)**
    - Type: Set  
    - Role: Defines static parameters (`server_domain`, `clients_dir`, `mount_dir`, `screen_left`, `screen_right`) used throughout workflow.  
    - Configuration: Hardcoded strings with default values, e.g. `server_domain = d01-test.uuq.pl`.  
    - Inputs: From `API`  
    - Outputs: Parameter JSON merged with API data forwarded to `If` node.  
    - Failure modes: None critical; misconfiguration leads to validation failure.

  - **If (Conditional)**
    - Type: If  
    - Role: Compares `server_domain` from parameters with the one in the API request.  
    - Configuration: Checks if `{{$json.server_domain}} === {{$('API').item.json.body.server_domain}}`.  
    - Inputs: From `Parametrs`  
    - Outputs: Passes valid requests forward; invalid requests routed to error response node.  
    - Failure modes: False negatives if domain mismatches; case sensitivity enforced.

  - **422-Invalid server domain (Respond to Webhook)**
    - Type: Respond to Webhook  
    - Role: Sends HTTP 422 error with JSON body indicating invalid domain error.  
    - Configuration: Response code 422, JSON error message.  
    - Inputs: From `If` node (false condition)  
    - Outputs: Ends workflow for invalid domain requests.

---

#### 2.2 Parameter Initialization

- **Overview:**  
  Establishes workflow-level parameters for directory paths and template syntax delimiters used in bash scripts and configuration generation.

- **Nodes Involved:**  
  - `Parametrs`

- **Node Details:**  
  Already described in 2.1.

---

#### 2.3 Command Routing

- **Overview:**  
  Routes incoming API commands to specific subflows based on the command type, differentiating container actions, service actions, and Immich app-specific commands.

- **Nodes Involved:**  
  - `Container Stat` (Switch for container info commands)  
  - `Container Actions` (Switch for container lifecycle commands)  
  - `Service Actions` (Switch for service lifecycle commands)  
  - `Immich` (Switch for Immich app-related commands)  
  - `If1` (Further branching for select commands)

- **Node Details:**

  - **Container Stat (Switch)**
    - Type: Switch  
    - Role: Routes commands related to container inspection, stats, logs, and dependent container stats.  
    - Conditions example:  
      - `container_information_inspect` → `Inspect` node  
      - `container_information_stats` → `Stat` node  
      - `container_log` → `Log` node  
      - `dependent_containers_information_stats` → `Dependent containers Stat` node  
    - Inputs: From `If` node (valid domain)  
    - Outputs: One per command type.

  - **Container Actions (Switch)**
    - Type: Switch  
    - Role: Routes commands related to container start, stop, mount/unmount disk, ACL get/set, and network info.  
    - Conditions example:  
      - `container_start` → `Start`  
      - `container_stop` → `Stop`  
      - `container_mount_disk` → `Mount Disk`  
      - `container_unmount_disk` → `Unmount Disk`  
      - `container_get_acl` → `GET ACL`  
      - `container_set_acl` → `SET ACL`  
      - `container_get_net` → `GET NET`  
    - Inputs: From `Container Stat` node  
    - Outputs: One per action.

  - **Service Actions (Switch)**
    - Type: Switch  
    - Role: Routes service lifecycle commands like test connection, create, suspend, unsuspend, terminate, change package.  
    - Conditions example:  
      - `test_connection` → `Test Connection1`  
      - `create` → `Deploy`  
      - `suspend` → `Suspend`  
      - `unsuspend` → `Unsuspend`  
      - `terminate` → `Terminated`  
      - `change_package` → `ChangePackage`  
    - Inputs: From `If1` node  
    - Outputs: One per action.

  - **Immich (Switch)**
    - Type: Switch  
    - Role: Routes Immich-specific commands like app version, users, and password change.  
    - Conditions example:  
      - `app_version` → `Version`  
      - `app_users` → `Users`  
      - `change_password` → `Change Password`  
    - Inputs: From `Container Stat` node  
    - Outputs: One per command.

  - **If1 (If)**
    - Type: If  
    - Role: Checks if command is `create`, `change_package`, or `unsuspend` to route to `nginx` node or else to `Service Actions`.  
    - Inputs: From `If` node  
    - Outputs: True → `nginx` node, False → `Service Actions`.

---

#### 2.4 Docker and NGINX Configuration Generation

- **Overview:**  
  Generates Docker Compose and NGINX configuration files dynamically using API parameters, encoding the content as base64 strings to be used in remote bash scripts.

- **Nodes Involved:**  
  - `Deploy-docker-compose` (Set node with Docker Compose yaml)  
  - `nginx` (Set node with NGINX proxy configuration snippets)

- **Node Details:**

  - **Deploy-docker-compose (Set)**
    - Type: Set  
    - Role: Creates a Docker Compose YAML configuration string populated with API parameters like domain, CPU, RAM, DB credentials, and mounts.  
    - Configuration: YAML template with embedded `{{$json.body.*}}` expressions, base64-encoded for transmission.  
    - Inputs: From `nginx` node  
    - Outputs: To `Service Actions` node for deployment scripts.

  - **nginx (Set)**
    - Type: Set  
    - Role: Defines NGINX proxy server configuration snippets for the main server block and location block, including headers and timeouts.  
    - Configuration: Text templates for `main` and `main_location` keys, editable per deployment needs.  
    - Inputs: From `If1` node  
    - Outputs: To `Deploy-docker-compose`.

---

#### 2.5 SSH Command Execution

- **Overview:**  
  Executes all Docker and system management operations on the target server via SSH, running bash scripts crafted to perform container lifecycle, disk mounting, NGINX config updates, and status retrieval.

- **Nodes Involved:**  
  - `SSH` (SSH command execution)  
  - Multiple `Set` nodes that prepare bash scripts (e.g., `Deploy`, `Start`, `Stop`, `Suspend`, `Unsuspend`, `Terminated`, `Mount Disk`, `Unmount Disk`, `GET ACL`, `SET ACL`, `GET NET`, `Stat`, `Inspect`, `Log`, `Dependent containers Stat`, `Version`, `Users`, `Change Password`)

- **Node Details:**

  - **SSH (SSH)**
    - Type: SSH  
    - Role: Runs bash scripts on remote Docker server using SSH credentials.  
    - Configuration: Command set dynamically from `$json.sh` property, shell script execution.  
    - Credentials: SSH password credential (e.g., for `d01-test.uuq.pl-puq`)  
    - Inputs: From various script-preparing Set nodes.  
    - Outputs: Raw command stdout/stderr passed to `Code1` for parsing.  
    - Failure modes: SSH connection failure, authentication error, command timeout, script error.

  - **Set Nodes with Bash Scripts**
    - Role: Each node prepares a specific bash script for a particular Docker or system operation, embedding necessary parameters from API and internal settings.  
    - Example scripts include:
      - **Deploy**: Creates directories, writes Docker Compose and NGINX configs, creates disk image, mounts, and starts Docker containers.
      - **Start/Stop**: Start or stop Docker containers.
      - **Suspend/Unsuspend**: Stop containers, unmount disks, remove or recreate NGINX configs.
      - **Terminated**: Complete cleanup of all resources.
      - **Mount Disk / Unmount Disk**: Manage disk image mounting via fstab.
      - **GET ACL / SET ACL**: Retrieve or update IP access control lists for NGINX.
      - **GET NET**: Retrieve network statistics from container.
      - **Stat / Inspect / Log / Dependent containers Stat**: Retrieve container info, stats, logs.
      - **Version / Users / Change Password**: Immich app-specific commands.
    - Inputs: From command routing nodes.
    - Outputs: Sets script in `$json.sh` for `SSH` node.
    - Edge cases: Script failures, permission issues, missing files, Docker errors.

---

#### 2.6 Response Handling

- **Overview:**  
  Parses SSH command output to standardized JSON responses and sends final HTTP responses to API clients.

- **Nodes Involved:**  
  - `Code1` (Code node for parsing and formatting responses)  
  - `API answer` (Webhook response node)

- **Node Details:**

  - **Code1 (Code)**
    - Type: Code (JavaScript)  
    - Role: Parses SSH stdout. If output is `"success"`, returns success JSON. Otherwise attempts to parse JSON from stdout. On failure, returns error JSON with message.  
    - Inputs: From `SSH` node  
    - Outputs: Structured JSON for API response.

  - **API answer (Respond to Webhook)**
    - Type: Respond to Webhook  
    - Role: Sends HTTP 200 response with the JSON data returned by `Code1`.  
    - Inputs: From `Code1`  
    - Outputs: HTTP response to API caller.

---

#### 2.7 Error Handling and Logging

- **Overview:**  
  The workflow incorporates error handling at multiple levels: SSH errors continue with error output, bash scripts include internal error handling functions that exit with error messages, and workflow nodes handle fallback paths for invalid inputs or failures.

- **Nodes Involved:**  
  - `422-Invalid server domain` (Responds to invalid domain)  
  - `SSH` node with `continueErrorOutput` on error  
  - `Code1` node parses error messages from SSH output  
  - Bash scripts include `handle_error()` functions for graceful exit and error reporting.

- **Details:**  
  - Invalid API domain triggers immediate HTTP 422 response.  
  - SSH node continues workflow even on command error, enabling error messages to propagate.  
  - Bash scripts return structured JSON error messages or error string prefixes.  
  - `Code1` converts error outputs into consistent JSON API error responses.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                                 | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                          |
|---------------------------|-------------------------|------------------------------------------------|------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| API                       | Webhook                 | Receives API requests, authenticates via Basic Auth | -                      | Parametrs                |                                                                                                    |
| Parametrs                 | Set                     | Defines static parameters including server domain and directories | API                    | If                       |                                                                                                    |
| If                        | If                      | Validates `server_domain` against API request  | Parametrs               | Container Stat, Container Actions, Immich, If1, 422-Invalid server domain |                                                                                                    |
| 422-Invalid server domain | Respond to Webhook      | Responds with 422 error for invalid domain      | If (false output)       | -                        |                                                                                                    |
| Container Stat            | Switch                  | Routes container information commands           | If (true output)        | Inspect, Stat, Log, Dependent containers Stat |                                                                                                    |
| Inspect                   | Set                     | Prepares bash script for `docker inspect`       | Container Stat          | SSH                      |                                                                                                    |
| Stat                      | Set                     | Prepares bash script for docker stats and disk usage | Container Stat          | SSH                      |                                                                                                    |
| Log                       | Set                     | Prepares bash script to retrieve container logs | Container Stat          | SSH                      |                                                                                                    |
| Dependent containers Stat | Set                     | Prepares bash script for dependent container stats | Container Stat          | SSH                      |                                                                                                    |
| Container Actions         | Switch                  | Routes container lifecycle actions               | Container Stat          | Start, Stop, Mount Disk, Unmount Disk, GET ACL, SET ACL, GET NET |                                                                                                    |
| Start                     | Set                     | Bash script to start Docker containers           | Container Actions       | SSH                      |                                                                                                    |
| Stop                      | Set                     | Bash script to stop Docker containers            | Container Actions       | SSH                      |                                                                                                    |
| Mount Disk                | Set                     | Bash script to mount disk image                   | Container Actions       | SSH                      |                                                                                                    |
| Unmount Disk              | Set                     | Bash script to unmount disk image                 | Container Actions       | SSH                      |                                                                                                    |
| GET ACL                   | Set                     | Bash script to get NGINX ACL configuration        | Container Actions       | SSH                      |                                                                                                    |
| SET ACL                   | Set                     | Bash script to set NGINX ACL configuration        | Container Actions       | SSH                      |                                                                                                    |
| GET NET                   | Set                     | Bash script to get container network stats        | Container Actions       | SSH                      |                                                                                                    |
| Immich                   | Switch                  | Routes Immich app specific commands               | Container Stat          | Version, Users, Change Password |                                                                                                    |
| Version                   | Set                     | Bash script to get Immich app version             | Immich                  | SSH                      |                                                                                                    |
| Users                     | Set                     | Bash script to query Immich app users              | Immich                  | SSH                      |                                                                                                    |
| Change Password           | Set                     | Bash script to change Immich admin password       | Immich                  | SSH                      |                                                                                                    |
| If1                      | If                      | Routes commands to nginx config or service actions | If                      | nginx, Service Actions    |                                                                                                    |
| nginx                    | Set                     | Defines NGINX proxy configuration snippets        | If1                     | Deploy-docker-compose     |                                                                                                    |
| Deploy-docker-compose     | Set                     | Generates Docker Compose configuration yaml       | nginx                    | Service Actions           |                                                                                                    |
| Service Actions          | Switch                  | Routes service lifecycle commands                  | If1                      | Test Connection1, Deploy, Suspend, Unsuspend, Terminated, ChangePackage |                                                                                                    |
| Test Connection1         | Set                     | Bash script to test Docker and proxy container health | Service Actions          | SSH                      |                                                                                                    |
| Deploy                   | Set                     | Bash script to deploy Docker containers and mount disks | Service Actions          | SSH                      |                                                                                                    |
| Suspend                  | Set                     | Bash script to suspend service and unmount disks  | Service Actions          | SSH                      |                                                                                                    |
| Unsuspend                | Set                     | Bash script to unsuspend service and remount disks | Service Actions          | SSH                      |                                                                                                    |
| Terminated               | Set                     | Bash script to fully remove service and resources | Service Actions          | SSH                      |                                                                                                    |
| ChangePackage            | Set                     | Bash script to change package and resize disk image | Service Actions          | SSH                      |                                                                                                    |
| SSH                      | SSH                     | Executes bash scripts on remote Docker server     | Multiple Set nodes       | Code1                     |                                                                                                    |
| Code1                    | Code                    | Parses SSH output and formats API JSON response    | SSH                      | API answer                |                                                                                                    |
| API answer               | Respond to Webhook      | Sends final HTTP response with JSON result        | Code1                    | -                         |                                                                                                    |
| Sticky Note              | Sticky Note             | Documentation and setup instructions               | -                        | -                         | ## 👋 Welcome to PUQ Docker Immich deploy! Template for Immich API Backend for WHMCS/WISECP by PUQcloud. Documentation links provided. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - Create a **Basic Auth Credential** named "Immich" for the webhook node.
   - Create an **SSH Credential** with username and password/key for the target Docker server.

2. **Create the Webhook Node (`API`):**
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `docker-immich`  
   - Authentication: Basic Auth (use the "Immich" credential)  
   - Response Mode: `responseNode` (will respond from a later node)

3. **Create the Parameter Node (`Parametrs`):**
   - Type: Set  
   - Assign static variables:  
     - `server_domain`: string, e.g. `d01-test.uuq.pl`  
     - `clients_dir`: string, e.g. `/opt/docker/clients`  
     - `mount_dir`: string, e.g. `/mnt`  
     - `screen_left`: string, value `{{` (must not be changed)  
     - `screen_right`: string, value `}}` (must not be changed)  
   - Connect `API` output to this node.

4. **Create the Domain Validation Node (`If`):**
   - Type: If (v2)  
   - Condition: Check if `{{$json.server_domain}}` equals `{{$('API').item.json.body.server_domain}}`  
   - Case sensitive, strict type validation.  
   - Connect `Parametrs` output to this node.

5. **Create Error Response Node (`422-Invalid server domain`):**
   - Type: Respond to Webhook  
   - Response Code: 422  
   - Response Body (JSON):  
     ```json
     [{
       "status": "error",
       "error": "Invalid server domain"
     }]
     ```  
   - Connect `If` false output to this node.

6. **Create Command Routing Switches:**

   - **Container Stat (Switch):**  
     - Route commands: `"container_information_inspect"`, `"container_information_stats"`, `"container_log"`, `"dependent_containers_information_stats"`  
     - Connect `If` true output to this node.

   - **Container Actions (Switch):**  
     - Route commands: `"container_start"`, `"container_stop"`, `"container_mount_disk"`, `"container_unmount_disk"`, `"container_get_acl"`, `"container_set_acl"`, `"container_get_net"`  
     - Connect `Container Stat` outputs to these specific action nodes.

   - **Immich (Switch):**  
     - Route commands: `"app_version"`, `"app_users"`, `"change_password"`  
     - Connect `Container Stat` outputs to these nodes.

   - **If1 (If):**  
     - Condition: Command equals `"create"`, `"change_package"`, or `"unsuspend"`  
     - Connect `If` true output to this node.

   - **nginx (Set):**  
     - Create two string parameters: `main` (NGINX server block config), `main_location` (NGINX location `/` config)  
     - Connect `If1` true output here.

   - **Deploy-docker-compose (Set):**  
     - Create a string parameter `docker-compose` with Docker Compose YAML template filled with API parameters for domain, CPU, RAM, DB credentials, mounts.  
     - Connect `nginx` output here.

   - **Service Actions (Switch):**  
     - Route commands: `"test_connection"`, `"create"`, `"suspend"`, `"unsuspend"`, `"terminate"`, `"change_package"`  
     - Connect `If1` false output and `Deploy-docker-compose` output here.

7. **Create Bash Script Preparation Nodes (Set nodes) for Each Action:**

   - Script nodes prepare bash scripts with embedded parameters using expressions like `{{$('API').item.json.body.domain}}`, `{{$('Parametrs').item.json.clients_dir}}`, etc.

   - Examples include `Deploy`, `Start`, `Stop`, `Suspend`, `Unsuspend`, `Terminated`, `Mount Disk`, `Unmount Disk`, `GET ACL`, `SET ACL`, `GET NET`, `Stat`, `Inspect`, `Log`, `Dependent containers Stat`, `Version`, `Users`, `Change Password`, `Test Connection1`, `ChangePackage`.

   - Connect these nodes to the respective switches.

8. **Create SSH Node:**

   - Type: SSH  
   - Credentials: Use the SSH Credential created earlier.  
   - Command: Set to `{{$json.sh}}` (runs the bash script from input JSON)  
   - OnError: Continue with error output (to parse errors downstream)  
   - Connect all script nodes outputs to this node.

9. **Create Code Node (`Code1`):**

   - Type: Code (JavaScript)  
   - Purpose: Parse SSH output to structured JSON for API response.  
   - Logic:  
     - If output is `"success"`, return success JSON.  
     - Else try to parse output as JSON; if error or missing, return error JSON with message.  
   - Connect SSH node output here.

10. **Create API Response Node (`API answer`):**

    - Type: Respond to Webhook  
    - Response Code: 200  
    - Respond With: All incoming items (the JSON from `Code1`)  
    - Connect `Code1` output here.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is a template for the PUQ Docker Immich deploy module, creating an API backend for WHMCS/WISECP integration. | Workflow purpose                                                                                 |
| Setup requires n8n server with SSH access to Docker host and Basic Auth for API webhook security.   | Setup prerequisites                                                                             |
| Do not modify technical parameters `screen_left` and `screen_right` used in bash scripts.           | Parameter constraints                                                                           |
| Documentation: [https://doc.puq.info/books/docker-immich-whmcs-module](https://doc.puq.info/books/docker-immich-whmcs-module) | Official documentation link                                                                     |
| WHMCS module info: [https://puqcloud.com/whmcs-module-docker-immich.php](https://puqcloud.com/whmcs-module-docker-immich.php) | Product page                                                                                     |
| Recommended to use official n8n cloud or self-hosted n8n server for this workflow.                   | Hosting recommendation                                                                          |
| The workflow dynamically generates Docker Compose and NGINX config files and manages them via SSH.  | Key design note                                                                                |
| All bash scripts include error handlers and produce JSON or clear text output for robust response.  | Error handling approach                                                                         |

---

This comprehensive documentation provides all necessary details for users or AI agents to understand, reproduce, and maintain the "Deploy Docker Immich, API Backend for WHMCS/WISECP" workflow.