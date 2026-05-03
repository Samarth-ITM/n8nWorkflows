Deploy Docker NextCloud, API Backend for WHMCS/WISECP

https://n8nworkflows.xyz/workflows/deploy-docker-nextcloud--api-backend-for-whmcs-wisecp-4015


# Deploy Docker NextCloud, API Backend for WHMCS/WISECP

---

### 1. Workflow Overview

This **PUQ Docker NextCloud deploy** workflow automates the deployment, configuration, and management of NextCloud and NextCloud Office Docker containers, specifically designed as an API backend for WHMCS/WISECP integrations. It securely processes API commands via webhook, performs Docker container orchestration over SSH, manages NGINX proxy configurations, handles disk mounting operations, and supports DNS record management.

The workflow is logically divided into the following functional blocks:

- **1.1 API Input Reception and Validation**  
  Receives incoming HTTP POST API requests with Basic Auth, validates server domains, and sets workflow parameters.

- **1.2 Command Routing and Execution**  
  Interprets commands from the API, routing them through switches to appropriate action blocks such as service lifecycle management, container stats retrieval, NextCloud-specific commands, DNS management, etc.

- **1.3 Docker Compose Deployment and NGINX Configuration**  
  Generates Docker Compose files, NGINX configuration snippets, and manages container lifecycle commands (create, start, stop, suspend, unsuspend, terminate, change package).

- **1.4 Disk Mounting and File System Management**  
  Manages mounting and unmounting of disk images for container storage, including fstab updates and permissions.

- **1.5 NextCloud Application Management**  
  Executes NextCloud-specific commands inside containers such as retrieving version, listing users, changing passwords, and installing/configuring NextCloud Office (Collabora).

- **1.6 DNS Record Management**  
  Handles DNS CNAME record updates using external HTTP API calls to a PowerDNS or compatible DNS server, including adding and deleting records.

- **1.7 SSH Execution and Server Routing**  
  Routes commands to the correct Docker host server over SSH, based on server domain, and executes dynamically generated Bash scripts.

- **1.8 API Response Handling**  
  Parses the raw SSH command outputs, formats them into structured JSON responses, and returns to the API caller.


---

### 2. Block-by-Block Analysis

#### 2.1 API Input Reception and Validation

- **Overview:**  
  Receives POST requests on webhook `docker-nextcloud` secured by Basic Auth. Validates the provided `server_domain` against allowed domains and initializes workflow parameters.

- **Nodes Involved:**  
  - `API` (Webhook)  
  - `Parametrs` (Set)  
  - `If` (If condition)  
  - `422-Invalid server domain` (Respond to Webhook)

- **Node Details:**  
  - **API**  
    - Type: Webhook node (HTTP POST)  
    - Configuration: Path `docker-nextcloud`, Basic Auth credential required  
    - Input: External HTTP POST request JSON body  
    - Output: Passes to parameter setting node  
    - Failure: Authentication failure or missing body leads to workflow halt

  - **Parametrs**  
    - Type: Set node  
    - Configuration: Defines static parameters such as `clients_dir`, `mount_dir`, and technical strings `screen_left` and `screen_right` used in script templates.  
    - Input: From `API` node  
    - Output: Passes to domain validation

  - **If**  
    - Type: If node  
    - Configuration: Checks if `server_domain` in API body equals either `d01-test.uuq.pl` or `d02-test.uuq.pl`  
    - Input: From `Parametrs` node  
    - Output:  
      - True: Continue workflow  
      - False: Trigger `422-Invalid server domain`

  - **422-Invalid server domain**  
    - Type: Respond to webhook  
    - Configuration: HTTP 422 response with JSON error message `"Invalid server domain"`  
    - Input: From `If` node (False branch)  
    - Output: Ends workflow with error response

- **Edge Cases:**  
  - Invalid or missing `server_domain` will cause a 422 error response.  
  - Authentication failure on webhook prevents further execution.

---

#### 2.2 Command Routing and Execution

- **Overview:**  
  Routes the API command (`command` field in JSON body) to appropriate action blocks using multiple Switch nodes. Commands include container management, service lifecycle, NextCloud app management, and DNS updates.

- **Nodes Involved:**  
  - `If1` (If node for create/change_package/unsuspend)  
  - `Service Actions` (Switch node for main service commands)  
  - `Container Stats` (Switch node for container info commands)  
  - `Container Actions` (Switch node for container lifecycle commands)  
  - `NextCloud` (Switch node for NextCloud app commands)  
  - `DNS Service Actions` (Switch node for DNS commands)  
  - `Servers Switch` (Switch node for routing to SSH server)  

- **Node Details:**  
  - **If1**  
    - Checks if command is one of `create`, `change_package`, or `unsuspend`.  
    - Routes to set `nginx` parameters or to `Service Actions`.

  - **Service Actions**  
    - Switch node routing commands:  
      - `test_connection`  
      - `create`  
      - `suspend`  
      - `unsuspend`  
      - `terminate`  
      - `change_package`  
    - Outputs mapped to respective Bash script set nodes.

  - **Container Stats**  
    - Switch node routing commands:  
      - `container_information_inspect`  
      - `container_information_stats`  
      - `container_log`  
      - `dependent_containers_information_stats`  
      - `container_update_dns_record`  
    - Each output leads to set nodes generating corresponding bash scripts or domain splitting.

  - **Container Actions**  
    - Switch node routing commands:  
      - `container_start`  
      - `container_stop`  
      - `container_mount_disk`  
      - `container_unmount_disk`  
      - `container_get_acl`  
      - `container_set_acl`  
      - `container_get_net`  
    - Each output leads to a set node with a specific script.

  - **NextCloud**  
    - Switch node routing commands:  
      - `app_version`  
      - `app_users`  
      - `change_password`  
    - Each output leads to a bash script set node executing NextCloud CLI commands inside containers.

  - **DNS Service Actions**  
    - Switch routing commands for DNS operations:  
      - `container_update_dns_record`  
      - `create`  
      - `suspend`  
      - `unsuspend`  
      - `terminate`  
      - `change_package`  
    - Calls HTTP request nodes to add or delete DNS CNAME records.

  - **Servers Switch**  
    - Routes commands to appropriate SSH nodes based on `server_domain` (`d01-test.uuq.pl` or `d02-test.uuq.pl`).

- **Edge Cases:**  
  - Unrecognized commands may not be routed properly.  
  - Failure to match `server_domain` to SSH nodes causes no execution.  
  - Complex commands with invalid parameters may cause script errors.

---

#### 2.3 Docker Compose Deployment and NGINX Configuration

- **Overview:**  
  Generates dynamic Docker Compose YAML and NGINX config snippets according to API parameters. Supports lifecycle operations like deploy, suspend, unsuspend, terminate, and change package.

- **Nodes Involved:**  
  - `Deploy-docker-compose` (Set node)  
  - `nginx` (Set node)  
  - `Deploy` (Set node - Bash script)  
  - `Suspend` (Set node - Bash script)  
  - `Unsuspend` (Set node - Bash script)  
  - `Terminated` (Set node - Bash script)  
  - `ChangePackage` (Set node - Bash script)

- **Node Details:**  
  - **Deploy-docker-compose**  
    - Creates a Docker Compose YAML file with services: NextCloud, MariaDB, Redis, Collabora/NextCloud Office.  
    - Uses API parameters like domain, admin user/password, MySQL credentials, RAM, CPU, etc.  
    - Networks attached to `nginx-proxy_web` external network.

  - **nginx**  
    - Defines NGINX configuration fragments including proxy headers, timeouts, WebSocket support.  
    - Contains main server block and office location block for Collabora integration.

  - **Deploy, Suspend, Unsuspend, Terminated, ChangePackage**  
    - Each contains extensive bash scripts to:  
      - Create/remove directories, config files, docker-compose.yml, mount/unmount disk image  
      - Manage NGINX config files and permissions  
      - Start/stop containers with `docker-compose` or `docker compose`  
      - Install NextCloud Office app asynchronously  
      - Handle cron jobs for NextCloud  
      - Update `/etc/fstab` and perform filesystem operations on disk images  
      - Log errors and manage status file on server  

- **Edge Cases:**  
  - Failures in Docker Compose commands, file system permissions, or disk image handling may occur.  
  - Background installation of NextCloud Office may fail or timeout.  
  - Permissions issues on NGINX configuration files or mount points.  
  - Docker network or container conflicts.

---

#### 2.4 Disk Mounting and File System Management

- **Overview:**  
  Handles mounting and unmounting of the disk image used as persistent storage for NextCloud data, ensuring proper system mount entries and permissions.

- **Nodes Involved:**  
  - `Mount Disk` (Set node - Bash script)  
  - `Unmount Disk` (Set node - Bash script)

- **Node Details:**  
  - **Mount Disk**  
    - Creates mount directory with permissions  
    - Adds image file mount to `/etc/fstab` if not present  
    - Executes `mount -a` to mount disk  
    - Checks if already mounted and prevents duplicate mounts  
    - Reports success or failure

  - **Unmount Disk**  
    - Checks if mount exists, unmounts disk  
    - Removes mount entry from `/etc/fstab`  
    - Deletes mount directory  
    - Handles errors in unmount or removal steps

- **Edge Cases:**  
  - Mount point already in use or disk image missing  
  - Permission denied on mount or unmount operations  
  - Corrupted or locked filesystem preventing operations

---

#### 2.5 NextCloud Application Management

- **Overview:**  
  Executes inside-container NextCloud CLI commands: retrieve version, list users, change user password, install/enable NextCloud Office app.

- **Nodes Involved:**  
  - `NextCloud` (Switch node)  
  - `Version` (Set node - Bash script)  
  - `Users` (Set node - Bash script)  
  - `Change Password` (Set node - Bash script)

- **Node Details:**  
  - **Version**  
    - Runs `php occ status` inside NextCloud container as UID 33 (www-data)  
    - Parses JSON output to extract version string  
    - Returns JSON with version info or error

  - **Users**  
    - Runs `php occ user:list` inside container  
    - Reformats user list to JSON array with username and display name  
    - Returns JSON list or error

  - **Change Password**  
    - Runs `php occ user:resetpassword` with password from environment variable  
    - Validates input presence of user and password  
    - Returns success or error JSON

- **Edge Cases:**  
  - Container not found or not running  
  - Invalid username or missing password  
  - PHP/occ command failures inside container

---

#### 2.6 DNS Record Management

- **Overview:**  
  Manages DNS CNAME records via external HTTP API (expected to be PowerDNS or compatible server). Supports adding and deleting DNS records corresponding to deployed domains.

- **Nodes Involved:**  
  - `Split domain` (Code node)  
  - `DNS Parametrs` (Set node)  
  - `DNS Service Actions` (Switch node)  
  - `Add record` (HTTP Request node)  
  - `Delete record` (HTTP Request node)  
  - `API answer1` (Respond to Webhook node)

- **Node Details:**  
  - **Split domain**  
    - JavaScript splits full domain into subdomain and main domain parts  
    - Used to determine DNS zone for API calls

  - **DNS Parametrs**  
    - Holds API URL and API key for DNS management  
    - User must configure these parameters

  - **DNS Service Actions**  
    - Routes commands for DNS management to add or delete CNAME records via HTTP PATCH requests

  - **Add record**  
    - PATCH to `/zones/{mainDomain}` with JSON body to add/replace CNAME record for full domain pointing to `server_domain`

  - **Delete record**  
    - PATCH to `/zones/{mainDomain}` with empty record list to remove the CNAME record

  - **API answer1**  
    - Sends success 200 JSON response after DNS operation

- **Edge Cases:**  
  - Invalid API URL or key results in HTTP errors  
  - Incorrect domain parsing can cause wrong zone updates  
  - Network or API unavailability

---

#### 2.7 SSH Execution and Server Routing

- **Overview:**  
  Routes commands to specific Docker host servers (`d01-test.uuq.pl` or `d02-test.uuq.pl`) over SSH. Executes generated bash scripts remotely with appropriate credentials.

- **Nodes Involved:**  
  - `Servers Switch` (Switch node)  
  - `d01-test.uuq.pl` (SSH node)  
  - `d02-test.uuq.pl` (SSH node)  
  - `Code` (Code node to parse SSH output)  
  - `API answer2` (Respond to webhook)

- **Node Details:**  
  - **Servers Switch**  
    - Routes based on `server_domain` parameter to one of the two SSH nodes

  - **SSH Nodes**  
    - Execute bash script commands (passed via JSON `sh` field) on remote Docker server  
    - Use SSH credentials configured in n8n (password or key-based auth)  
    - Execute once per workflow execution

  - **Code**  
    - Parses stdout from SSH node  
    - Converts bash output (JSON or plain text) into structured JSON response with status and message fields

  - **API answer2**  
    - Returns final structured response to API caller with HTTP 200 status and all processed data

- **Edge Cases:**  
  - SSH connection failures or authentication errors  
  - Command execution errors or timeouts on remote server  
  - Parsing errors if output is malformed

---

#### 2.8 API Response Handling

- **Overview:**  
  Handles formatting and sending API responses back to the HTTP caller based on execution results.

- **Nodes Involved:**  
  - `422-Invalid server domain`  
  - `API answer1`  
  - `API answer2`  
  - `Code` (output formatter)

- **Node Details:**  
  - **422-Invalid server domain**  
    - Returns HTTP 422 with error JSON if domain invalid

  - **API answer1**  
    - Returns HTTP 200 success response after DNS record updates

  - **API answer2**  
    - Returns HTTP 200 with full JSON output after SSH execution and parsing

  - **Code**  
    - Ensures consistent JSON response format for success or error messages

- **Edge Cases:**  
  - Errors during response formulation or missing data

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                                    | Input Node(s)        | Output Node(s)                  | Sticky Note                                                                                                          |
|-------------------------|-----------------------------|---------------------------------------------------|----------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| API                     | Webhook                     | Receives API POST requests with Basic Auth       | —                    | Parametrs                     |                                                                                                                      |
| Parametrs               | Set                         | Sets static parameters for scripts and paths     | API                  | If                           |                                                                                                                      |
| If                      | If                          | Validates server domain parameter                  | Parametrs             | Container Stats, 422-Invalid server domain |                                                                                                                      |
| 422-Invalid server domain| Respond to Webhook          | Returns error if server domain invalid             | If                   | —                             |                                                                                                                      |
| Container Stats         | Switch                      | Routes container info commands                      | If                   | Inspect, Stat, Log, Dependent containers Stat, Split domain |                                                                                                                      |
| Inspect                 | Set                         | Bash script for `docker inspect`                   | Container Stats       | Servers Switch                |                                                                                                                      |
| Stat                    | Set                         | Bash script for container stats + disk info        | Container Stats       | Servers Switch                |                                                                                                                      |
| Log                     | Set                         | Bash script to fetch docker logs                    | Container Stats       | Servers Switch                |                                                                                                                      |
| Dependent containers Stat| Set                        | Bash script for dependent containers info          | Container Stats       | Servers Switch                |                                                                                                                      |
| Split domain            | Code                        | Splits domain into subdomain and main domain       | Container Stats, Service Actions, Terminated, ChangePackage | DNS Parametrs, DNS Service Actions |                                                                                                                      |
| DNS Parametrs           | Set                         | Holds DNS API URL and key                           | Split domain          | If2                          |                                                                                                                      |
| DNS Service Actions     | Switch                      | Routes DNS-related commands                         | If2                   | Add record, Delete record     |                                                                                                                      |
| Add record              | HTTP Request                | Adds or updates DNS CNAME record                    | DNS Service Actions   | API answer1                  |                                                                                                                      |
| Delete record           | HTTP Request                | Deletes DNS CNAME record                            | DNS Service Actions   | API answer1                  |                                                                                                                      |
| API answer1             | Respond to Webhook          | Returns DNS operation success response             | Add record, Delete record | —                          |                                                                                                                      |
| If1                     | If                          | Checks if command is create, change_package, unsuspend | Container Actions    | nginx, Service Actions        |                                                                                                                      |
| nginx                   | Set                         | Creates NGINX config snippets                       | If1                   | Deploy-docker-compose         |                                                                                                                      |
| Deploy-docker-compose   | Set                         | Generates Docker Compose YAML                        | nginx                  | Service Actions               |                                                                                                                      |
| Service Actions         | Switch                      | Routes main service commands                        | If1                    | Test Connection, Deploy, Suspend, Unsuspend, Terminated, ChangePackage |                                                                                                                      |
| Test Connection         | Set                         | Bash script to verify Docker and proxy containers  | Service Actions       | Servers Switch                |                                                                                                                      |
| Deploy                  | Set                         | Bash script to deploy and start Docker containers  | Service Actions       | Servers Switch                |                                                                                                                      |
| Suspend                 | Set                         | Bash script to suspend services                      | Service Actions       | Servers Switch                |                                                                                                                      |
| Unsuspend               | Set                         | Bash script to unsuspend services                    | Service Actions       | Servers Switch                |                                                                                                                      |
| Terminated              | Set                         | Bash script to terminate and clean up services      | Service Actions       | Servers Switch                |                                                                                                                      |
| ChangePackage           | Set                         | Bash script to change service package                | Service Actions       | Servers Switch                |                                                                                                                      |
| Container Actions       | Switch                      | Routes container lifecycle commands                 | Container Stats       | Start, Stop, Mount Disk, Unmount Disk, GET ACL, SET ACL, GET NET |                                                                                                                      |
| Start                   | Set                         | Bash script to start container                       | Container Actions     | Servers Switch                |                                                                                                                      |
| Stop                    | Set                         | Bash script to stop container                        | Container Actions     | Servers Switch                |                                                                                                                      |
| Mount Disk              | Set                         | Bash script to mount disk image                      | Container Actions     | Servers Switch                |                                                                                                                      |
| Unmount Disk            | Set                         | Bash script to unmount disk image                    | Container Actions     | Servers Switch                |                                                                                                                      |
| GET ACL                 | Set                         | Bash script to get NGINX ACL IPs                     | Container Actions     | Servers Switch                |                                                                                                                      |
| SET ACL                 | Set                         | Bash script to set NGINX ACL IPs                     | Container Actions     | Servers Switch                |                                                                                                                      |
| GET NET                 | Set                         | Bash script to get container network stats           | Container Actions     | Servers Switch                |                                                                                                                      |
| NextCloud               | Switch                      | Routes NextCloud app management commands             | Container Stats       | Version, Users, Change Password |                                                                                                                      |
| Version                 | Set                         | Bash script to get NextCloud version                  | NextCloud             | Servers Switch                |                                                                                                                      |
| Users                   | Set                         | Bash script to list NextCloud users                    | NextCloud             | Servers Switch                |                                                                                                                      |
| Change Password         | Set                         | Bash script to change NextCloud user password         | NextCloud             | Servers Switch                |                                                                                                                      |
| Servers Switch          | Switch                      | Routes execution to correct SSH server node           | Various               | d01-test.uuq.pl, d02-test.uuq.pl |                                                                                                                      |
| d01-test.uuq.pl         | SSH                         | Executes bash scripts on Docker server d01-test       | Servers Switch        | Code                         |                                                                                                                      |
| d02-test.uuq.pl         | SSH                         | Executes bash scripts on Docker server d02-test       | Servers Switch        | Code                         |                                                                                                                      |
| Code                    | Code                        | Parses SSH output and formats JSON response           | d01-test.uuq.pl, d02-test.uuq.pl | API answer2             |                                                                                                                      |
| API answer2             | Respond to Webhook          | Returns final API response with execution results     | Code                  | —                            |                                                                                                                      |
| If2                     | If                          | Validates main domain for DNS operations              | DNS Parametrs         | DNS Service Actions          |                                                                                                                      |
| Sticky Note             | Sticky Note                 | Provides setup instructions, links, and guidance      | —                     | —                            | ## 👋 Welcome to PUQ Docker NextCloud deploy! Setup instructions and documentation links included.                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Webhook Node**  
- Create a new Webhook node named `API`  
- Set HTTP Method to POST  
- Set Path to `docker-nextcloud`  
- Enable Basic Auth and configure Basic Auth credentials (e.g., username/password for secure access)  
- Set Response Mode to `responseNode` to enable dynamic responses

**Step 2: Set Parameters Node**  
- Add a Set node named `Parametrs` connected to `API`  
- Define parameters:  
  - `clients_dir`: `/opt/docker/clients`  
  - `mount_dir`: `/mnt`  
  - `screen_left`: `{{` (used in bash templates)  
  - `screen_right`: `}}`

**Step 3: Add an If Node for Server Domain Validation**  
- Add an If node named `If` connected to `Parametrs`  
- Configure conditions to check if `server_domain` from API JSON body equals `d01-test.uuq.pl` OR `d02-test.uuq.pl`

**Step 4: Add Respond to Webhook Node for Invalid Domain**  
- Add a Respond to Webhook node named `422-Invalid server domain` connected to the False output of `If`  
- Set HTTP status code to 422  
- Set JSON response body to `{ "status": "error", "error": "Invalid server domain" }`

**Step 5: Create Switch Nodes for Command Routing**  
- Create the following Switch nodes connected from the True output of `If` (via `Parametrs`):  
  - `Container Stats` (routes container info commands)  
  - `Container Actions` (routes container lifecycle commands)  
  - `NextCloud` (routes NextCloud app commands)  
  - `Service Actions` (routes service lifecycle commands)  
  - `DNS Service Actions` (for DNS update commands)  
- Use the `command` field from API JSON body for routing

**Step 6: Create Set Nodes for Bash Script Generation**  
- For each command (e.g., `Deploy`, `Suspend`, `Unsuspend`, `Terminated`, `ChangePackage`, `Start`, `Stop`, `Mount Disk`, `Unmount Disk`, `GET ACL`, `SET ACL`, `GET NET`, `Version`, `Users`, `Change Password`, `Test Connection`, `Dependent containers Stat`, `Inspect`, `Stat`, `Log`), create a Set node  
- In each Set node, write the corresponding bash script in the `sh` field, using expressions to inject API parameters and static parameters from `Parametrs`  
- Ensure scripts handle error logging, directory creation, docker-compose operations, mounting, and NextCloud CLI commands

**Step 7: Create Set Nodes for Docker Compose and NGINX Configurations**  
- Create `Deploy-docker-compose` node to generate docker-compose YAML dynamically from API parameters  
- Create `nginx` node to generate NGINX configuration blocks for main and office locations

**Step 8: Create Switch Node for Server Routing**  
- Create `Servers Switch` node to route based on `server_domain` to the appropriate SSH node (`d01-test.uuq.pl` or `d02-test.uuq.pl`)

**Step 9: Create SSH Nodes for Servers**  
- Create SSH nodes named `d01-test.uuq.pl` and `d02-test.uuq.pl`  
- Configure SSH credentials (password or key-based) for each target server  
- Set command to execute from the incoming JSON `sh` field  
- Enable Execute Once

**Step 10: Create Code Node for SSH Output Parsing**  
- Create a Code node named `Code` connected after each SSH node  
- Write JavaScript to parse the `stdout` or `error` fields, transform bash output to structured JSON with `status`, `message`, and `data`

**Step 11: Create Respond to Webhook Node for Final API Response**  
- Create `API answer2` node to send back the final JSON response to the API caller with HTTP 200

**Step 12: Create DNS Management Nodes**  
- Create `Split domain` Code node to extract subdomain and main domain from the full domain string  
- Create `DNS Parametrs` Set node to hold API URL and API key for DNS management  
- Create `Add record` and `Delete record` HTTP Request nodes to PATCH DNS records on the external DNS API  
- Create `API answer1` Respond to Webhook node for DNS operations

**Step 13: Connect All Nodes According to Logic**  
- Connect nodes to mirror the command routing and execution flow as described in the overview  
- Ensure error handling flows properly  
- Validate all expressions and credentials

**Step 14: Configure Credentials**  
- Create Basic Auth credential in n8n for the webhook API  
- Create SSH credentials for each Docker server  
- Store DNS API key securely in credentials or environment variables

**Step 15: Test Workflow**  
- Test each command via API POST calls with correct Basic Auth  
- Monitor logs and responses  
- Adjust bash scripts or parameters as necessary

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is a template by PUQcloud for integrating NextCloud Docker deployments with WHMCS/WISECP. Setup requires dedicated n8n server or official n8n cloud.    | https://n8n.io/creators/puqcloud/                                                                        |
| Comprehensive documentation and setup guides are available at PUQcloud’s documentation site.                                                                         | https://doc.puq.info/books/docker-nextcloud-whmcs-module                                                  |
| WHMCS module page with additional information and purchase options.                                                                                                  | https://puqcloud.com/whmcs-module-docker-nextcloud.php                                                   |
| Required Debian packages on Docker host: `sqlite3` and `apache2-utils`.                                                                                              | Install with: `apt-get install sqlite3 apache2-utils -y`                                                  |
| The workflow dynamically generates Bash scripts incorporating API parameters to execute container and system operations securely over SSH.                          |                                                                                                          |
| Persistent storage is handled via disk image files mounted as ext4 loopback devices.                                                                                 |                                                                                                          |
| NGINX proxy configurations include custom headers, WebSocket support, and timeout settings optimized for NextCloud and Collabora integration.                       |                                                                                                          |
| NextCloud Office (Collabora) installation is performed asynchronously in the background after container deployment.                                                |                                                                                                          |
| Cron jobs for NextCloud maintenance are automatically added and removed during suspend/unsuspend lifecycle events.                                                  |                                                                                                          |
| DNS management uses PowerDNS-compatible API to update CNAME records dynamically.                                                                                     |                                                                                                          |

---

This completes the detailed analysis and reproduction instructions for the **PUQ Docker NextCloud deploy** workflow in n8n.