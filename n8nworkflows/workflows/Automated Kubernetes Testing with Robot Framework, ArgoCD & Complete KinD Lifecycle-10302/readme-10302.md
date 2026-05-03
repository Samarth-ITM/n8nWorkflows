Automated Kubernetes Testing with Robot Framework, ArgoCD & Complete KinD Lifecycle

https://n8nworkflows.xyz/workflows/automated-kubernetes-testing-with-robot-framework--argocd---complete-kind-lifecycle-10302


# Automated Kubernetes Testing with Robot Framework, ArgoCD & Complete KinD Lifecycle

### 1. Workflow Overview

This workflow automates the full lifecycle of Kubernetes testing using KinD (Kubernetes in Docker), Robot Framework for automated tests, and ArgoCD for GitOps deployment. It targets scenarios where users want to:

- Set up a Kubernetes test environment on a remote host
- Deploy applications declaratively with ArgoCD
- Run automated functional tests via Robot Framework with browser automation
- Tear down and clean all installed components after testing

The workflow is structured into three main logical blocks reflecting the lifecycle phases:

- **1.1 INIT Phase**: Prepares the remote environment including Docker installation, KinD cluster creation, Helm and ingress setup, HAProxy proxy configuration, and ArgoCD installation with ApplicationSet deployment.

- **1.2 TEST Phase**: Downloads Robot Framework test scripts, installs required runtime dependencies (Python, Robot Framework libraries, Chromium browser), adds DNS entries, runs the tests, packages logs and reports, and sends them via Telegram.

- **1.3 DESTROY Phase**: Cleans up the testing environment by removing HAProxy, deleting the KinD cluster, uninstalling KinD and Docker, removing host entries, and sending a completion notification.

Supporting these phases, the workflow includes:

- **Triggering Mechanisms**: Manual trigger, webhook (e.g., GitLab commit), or scheduled trigger (default daily at 1 AM).

- **Parameter Setup and Phase Routing**: A Switch node controls the phase flow based on a `progress` parameter (`INIT`, `TEST`, `DESTROY`) and `progress_only` flag to run full pipeline or single phases.

- **Robust SSH and Command Execution**: Uses `sshpass` for automated SSH commands to the remote host, with checks and conditional installs for necessary tools.

- **File Retrieval from GitLab**: Gets configuration files, test scripts, and ArgoCD ApplicationSet manifests from a GitLab repository using OAuth2 credentials.

---

### 2. Block-by-Block Analysis

#### 2.1 INIT Phase

**Overview:**  
Sets up the entire Kubernetes test infrastructure on a remote host, including installing Docker and KinD, creating the cluster, installing Helm and ingress controllers, configuring HAProxy for port forwarding, and deploying ArgoCD with the ApplicationSet.

**Nodes Involved:**  
- Check Sshpass Exist Local  
- Is Installed Local?  
- Install Sshpass Local  
- Check Docker on Target  
- Is Docker Installed on Target?  
- Install Docker on Target  
- Check KinD on Target  
- Is KinD Installed on Target?  
- Install KinD on Target  
- Get a file - KinD Config  
- Write Dowloaded KinD Config  
- Open KinD Config  
- Create KinD Cluster on Target  
- Install Helm and Nginx-Ingress in KinD Cluster  
- Install HAProxy on Target and Config Port-80 to KinD  
- Install ArgoCD to KinD Cluster  
- Get a file - ArgoCD ApplicationSet  
- Write Dowloaded ArgoCD ApplicationSet  
- Open ArgoCD ApplicationSet  
- Apply ArgoCD ApplicationSet  
- Set Initialized  
- Is INIT Only?  
- No Operation, do nothing  

**Node Details:**

- **Check Sshpass Exist Local**  
  - Type: Execute Command  
  - Role: Verify if `sshpass` is installed locally for SSH automation  
  - Command: `which sshpass`  
  - On failure, continues to attempt installation  
  - Potential issues: missing package, unsupported OS package manager  

- **Is Installed Local?**  
  - Type: If  
  - Role: Checks exit code and error output from previous node to decide if `sshpass` needs installing  
  - Conditions: exit code ≤ 0 and error does not contain "sshpass"  

- **Install Sshpass Local**  
  - Type: Execute Command  
  - Role: Installs `sshpass` and `rsync` on local machine via available package manager (apt-get, yum, dnf, apk)  
  - Edge cases: unsupported package manager, installation failure  

- **Check Docker on Target**  
  - Type: Execute Command  
  - Role: Checks if Docker is installed on remote target via SSH  
  - Command: `sshpass ... ssh ... "which docker"`  
  - Continues on failure (to trigger install if missing)  

- **Is Docker Installed on Target?**  
  - Type: If  
  - Role: Checks Docker presence on target host based on exit code and error output  

- **Install Docker on Target**  
  - Type: Execute Command  
  - Role: Installs Docker on remote host via SSH using `apt-get`, `yum`, `dnf`, or `apk` depending on OS  
  - Handles removal of old Docker versions and installs required dependencies  
  - Starts and enables Docker service  
  - Edge cases: password authentication errors, unsupported OS, command failures  

- **Check KinD on Target**  
  - Type: Execute Command  
  - Role: Checks if KinD binary is installed on remote host  

- **Is KinD Installed on Target?**  
  - Type: If  
  - Role: Decides to install KinD or proceed based on check  

- **Install KinD on Target**  
  - Type: Execute Command  
  - Role: Downloads KinD binary v0.24.0 on remote host, makes executable and moves to `/usr/local/bin`  

- **Get a file - KinD Config**  
  - Type: GitLab  
  - Role: Downloads KinD cluster config YAML from GitLab repo via OAuth2  

- **Write Dowloaded KinD Config**  
  - Type: Read/Write File  
  - Role: Writes downloaded KinD config to `/tmp/config.yaml` locally  

- **Open KinD Config**  
  - Type: Execute Command  
  - Role: Reads content of KinD config file (`cat`) for use in cluster creation  

- **Create KinD Cluster on Target**  
  - Type: Execute Command  
  - Role: SSH to target, uploads KinD config, and creates KinD cluster named `automate-tst` using config file  

- **Install Helm and Nginx-Ingress in KinD Cluster**  
  - Type: Execute Command  
  - Role: Installs Helm in KinD control plane container, installs `ingress-nginx` controller via Helm  
  - Waits for ingress controller pods to be ready  

- **Install HAProxy on Target and Config Port-80 to KinD**  
  - Type: Execute Command  
  - Role: Installs HAProxy on target host, configures proxy to forward port 80 to KinD ingress on localhost:60080  
  - Adds host entry `127.0.0.1 autotest.innersite` if missing in `/etc/hosts`  

- **Install ArgoCD to KinD Cluster**  
  - Type: Execute Command  
  - Role: Installs ArgoCD via Helm in KinD cluster, configures ingress for ArgoCD server on HTTP  
  - Patches ArgoCD secret and RBAC to create `autotest` admin user with password `autotest`  

- **Get a file - ArgoCD ApplicationSet**  
  - Type: GitLab  
  - Role: Downloads ArgoCD ApplicationSet YAML manifest from GitLab repo  

- **Write Dowloaded ArgoCD ApplicationSet**  
  - Type: Read/Write File  
  - Role: Writes ApplicationSet manifest to `/tmp/demo-applicationSet.yaml`  

- **Open ArgoCD ApplicationSet**  
  - Type: Execute Command  
  - Role: Reads content of ApplicationSet manifest file (`cat`)  

- **Apply ArgoCD ApplicationSet**  
  - Type: Execute Command  
  - Role: Applies ApplicationSet manifest in ArgoCD namespace inside KinD cluster via kubectl  
  - Lists ApplicationSets after applying  

- **Set Initialized**  
  - Type: Set  
  - Role: Updates `progress` parameter to `TEST` to advance to next phase  

- **Is INIT Only?**  
  - Type: If  
  - Role: Checks `progress_only` parameter to decide whether to continue to TEST phase or stop  

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Terminates workflow if configured to run only INIT phase  

---

#### 2.2 TEST Phase

**Overview:**  
Runs automated tests using Robot Framework with browser automation on the target host, packages test results including logs, screenshots, and HTML reports, and sends results via Telegram.

**Nodes Involved:**  
- Get a file - ROBOT Script  
- Write Dowloaded ROBOT Script  
- Add Robot Framework, Browser Library, Chromium Driver  
- Add Target to Hosts  
- Read ROBOT Script to Execute  
- Open ROBOT Script to Framework  
- Robot Framework  
- Pack ROBOT Script Exports  
- Read ROBOT Script Exports to Send  
- Send ROBOT Script Export Pack  
- Set Tested  
- Is TEST Only?  
- No Operation, do nothing  

**Node Details:**

- **Get a file - ROBOT Script**  
  - Type: GitLab  
  - Role: Downloads Robot Framework test script from GitLab repo  

- **Write Dowloaded ROBOT Script**  
  - Type: Read/Write File  
  - Role: Writes test script to `/tmp/process_it.robot`  

- **Add Robot Framework, Browser Library, Chromium Driver**  
  - Type: Execute Command  
  - Role: Installs dependencies on local or remote environment: python3, pip, robotframework, robotframework-browser, chromium browser and drivers, and initializes Robot Framework browser library  

- **Add Target to Hosts**  
  - Type: Execute Command  
  - Role: Adds an entry to `/etc/hosts` mapping target IP to `autotest.innersite` for local DNS resolution  

- **Read ROBOT Script to Execute**  
  - Type: Read/Write File  
  - Role: Reads the Robot Framework test script file content (`cat`)  

- **Open ROBOT Script to Framework**  
  - Type: Execute Command  
  - Role: Prepares the test script content for Robot Framework execution  

- **Robot Framework**  
  - Type: Robot Framework Node (Community node)  
  - Role: Runs the Robot Framework test script with browser automation, includes HTML logs and screenshots  
  - Key parameter: `robotScript` is set from previous node's stdout  

- **Pack ROBOT Script Exports**  
  - Type: Execute Command  
  - Role: Archives robot logs and reports from `/root/n8n_robot_logs` into a tar.gz file and cleans up logs  

- **Read ROBOT Script Exports to Send**  
  - Type: Read/Write File  
  - Role: Reads the compressed archive for sending in binary form  

- **Send ROBOT Script Export Pack**  
  - Type: Telegram  
  - Role: Sends the compressed test result archive to a Telegram chat ID  
  - Requires Telegram API credentials and chat ID configuration  

- **Set Tested**  
  - Type: Set  
  - Role: Updates `progress` parameter to `DESTROY` to proceed to cleanup phase  

- **Is TEST Only?**  
  - Type: If  
  - Role: Checks `progress_only` flag to decide whether to continue to DESTROY phase or stop  

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Ends workflow execution if configured to run only TEST phase  

---

#### 2.3 DESTROY Phase

**Overview:**  
Removes all installed components and cleans the environment on the remote host, including HAProxy removal, KinD cluster deletion, KinD and Docker uninstallation, host file cleanup, and sends final notifications.

**Nodes Involved:**  
- Check Sshpass Exist Local (D)  
- Is Installed Local? (D)  
- Install Sshpass Local (D)  
- Delete HAProxy on Target  
- Delete KinD Cluster on Target  
- Check KinD on Target (D)  
- Is KinD Installed on Target?1  
- UnInstall KinD on Target  
- Check Docker on Target (D)  
- Is Docker Installed on Target?1  
- UnInstall Docker on Target  
- Remove Target from Hosts  
- Set Destroyed  
- Process Finish Report --- Telegam & SMS1  

**Node Details:**

- **Check Sshpass Exist Local (D)**  
  - Same as in INIT phase, checking local `sshpass` before cleanup commands  

- **Is Installed Local? (D)**  
  - If node checking if `sshpass` is installed for DESTROY phase  

- **Install Sshpass Local (D)**  
  - Installs `sshpass` locally if missing  

- **Delete HAProxy on Target**  
  - Executes SSH commands to stop and uninstall HAProxy on remote host  
  - Removes HAProxy config files and cleans `/etc/hosts` entries related to `autotest.innersite`  
  - Edge cases: service stop failures, missing files, permission issues  

- **Delete KinD Cluster on Target**  
  - Deletes KinD cluster named `automate-tst` via SSH  

- **Check KinD on Target (D)**  
  - Checks KinD binary presence on target after deletion  

- **Is KinD Installed on Target?1**  
  - Conditional node deciding to uninstall KinD binary or proceed  

- **UnInstall KinD on Target**  
  - Removes KinD binary `/usr/local/bin/kind` on remote host  

- **Check Docker on Target (D)**  
  - Checks Docker installation on target after KinD cleanup  

- **Is Docker Installed on Target?1**  
  - Conditional node to decide Docker uninstallation  

- **UnInstall Docker on Target**  
  - Uninstalls Docker and dependencies from the remote host  
  - Handles stopping services, removing packages, and cleaning binaries  
  - Supports multiple package managers  

- **Remove Target from Hosts**  
  - Removes host entry for `autotest.innersite` pointing to target IP from local `/etc/hosts`  

- **Set Destroyed**  
  - Updates `progress` parameter to `CLEAR` indicating workflow completion  

- **Process Finish Report --- Telegam & SMS1**  
  - Sends Telegram notification about workflow completion  
  - Optionally supports SMS notifications via TextBelt (commented out)  
  - Requires Telegram bot token and chat ID configured in command environment variables  

---

#### 2.4 Trigger and Parameter Setup

**Overview:**  
Manages workflow start via different triggers and initializes parameters.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Webhook (HTTP POST for GitLab commit)  
- Schedule Trigger (default daily at 1 AM)  
- Set Parameters  
- Switch (routes phase based on `progress`)  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Manual trigger node for on-demand execution  

- **Webhook**  
  - HTTP POST webhook endpoint `/webhook/gitlab-commit` for GitLab integration  

- **Schedule Trigger**  
  - Cron-like schedule, triggers daily at 1 AM by default  

- **Set Parameters**  
  - Sets all required parameters such as SSH target details, file paths in GitLab, initial progress state, and flags  

- **Switch**  
  - Routes execution to INIT, TEST, or DESTROY phase blocks based on `progress` parameter value  

---

### 3. Summary Table

| Node Name                             | Node Type                 | Functional Role                             | Input Node(s)                           | Output Node(s)                          | Sticky Note                                                                                                                                                                                                                                    |
|-------------------------------------|---------------------------|---------------------------------------------|---------------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’    | Manual Trigger            | Manual execution start                       |                                       | Set Parameters                        | Trigger Configuration: Supports Manual Trigger, Webhook, Schedule.                                                                                                                                                                            |
| Webhook                             | Webhook                   | Trigger via GitLab commit                    |                                       | Set Parameters                        | Trigger Configuration: Supports Webhook trigger.                                                                                                                                                                                              |
| Schedule Trigger                    | Schedule Trigger          | Scheduled execution (daily 1 AM)             |                                       | Set Parameters                        | Trigger Configuration: Supports Schedule trigger.                                                                                                                                                                                             |
| Set Parameters                     | Set                      | Initialize parameters and variables          | When clicking ‘Execute workflow’, Webhook, Schedule Trigger | Switch                             | Configuration Guide: Update SSH and file path parameters before running.                                                                                                                                                                       |
| Switch                            | Switch                   | Route execution based on `progress` value  | Set Parameters, Set Tested, Set Initialized | INIT, TEST, DESTROY branches         | Phase Control Logic: Routes INIT, TEST, DESTROY phases based on progress; uses `progress_only` for partial runs.                                                                                                                                 |
| Check Sshpass Exist Local           | Execute Command           | Check `sshpass` locally                      | Switch (INIT)                        | Is Installed Local                   | INIT Phase Summary: Checks and installs prerequisites.                                                                                                                                                                                         |
| Is Installed Local?                 | If                        | Decide whether to install `sshpass`         | Check Sshpass Exist Local            | Install Sshpass Local or Check Docker on Target | INIT Phase Summary                                                                                                                                                                                                                              |
| Install Sshpass Local               | Execute Command           | Install `sshpass` locally                    | Is Installed Local?                  | Check Docker on Target              | INIT Phase Summary                                                                                                                                                                                                                              |
| Check Docker on Target              | Execute Command           | Check Docker presence on remote host        | Is Installed Local? or Install Sshpass Local | Is Docker Installed on Target?     | INIT Phase Summary                                                                                                                                                                                                                              |
| Is Docker Installed on Target?      | If                        | Decide on Docker install necessity           | Check Docker on Target               | Install Docker on Target or Check KinD on Target | INIT Phase Summary                                                                                                                                                                                                                              |
| Install Docker on Target            | Execute Command           | Install Docker remotely                      | Is Docker Installed on Target?       | Check KinD on Target               | INIT Phase Summary                                                                                                                                                                                                                              |
| Check KinD on Target               | Execute Command           | Check KinD binary presence on remote host   | Install Docker on Target or Is Docker Installed on Target? | Is KinD Installed on Target?        | INIT Phase Summary                                                                                                                                                                                                                              |
| Is KinD Installed on Target?        | If                        | Decide on KinD install necessity             | Check KinD on Target                | Get a file - KinD Config or Install KinD on Target | INIT Phase Summary                                                                                                                                                                                                                              |
| Install KinD on Target             | Execute Command           | Install KinD binary remotely                 | Is KinD Installed on Target?        | Get a file - KinD Config           | INIT Phase Summary                                                                                                                                                                                                                              |
| Get a file - KinD Config            | GitLab                    | Download KinD cluster config YAML            | Install KinD on Target              | Write Dowloaded KinD Config        | GitLab Integration: Downloads KinD config file from GitLab repo.                                                                                                                                                                               |
| Write Dowloaded KinD Config         | Read/Write File           | Write KinD config locally                     | Get a file - KinD Config            | Open KinD Config                  | INIT Phase Summary                                                                                                                                                                                                                              |
| Open KinD Config                   | Execute Command           | Read KinD config content                      | Write Dowloaded KinD Config         | Create KinD Cluster on Target      | INIT Phase Summary                                                                                                                                                                                                                              |
| Create KinD Cluster on Target       | Execute Command           | Create KinD cluster on remote host            | Open KinD Config                   | Install Helm and Nginx-Ingress in KinD Cluster | INIT Phase Summary                                                                                                                                                                                                                              |
| Install Helm and Nginx-Ingress in KinD Cluster | Execute Command           | Install Helm and ingress-nginx controller    | Create KinD Cluster on Target       | Install HAProxy on Target and Config Port-80 to KinD | INIT Phase Summary                                                                                                                                                                                                                              |
| Install HAProxy on Target and Config Port-80 to KinD | Execute Command           | Install and configure HAProxy on remote host | Install Helm and Nginx-Ingress in KinD Cluster | Install ArgoCD to KinD Cluster     | INIT Phase Summary                                                                                                                                                                                                                              |
| Install ArgoCD to KinD Cluster      | Execute Command           | Install ArgoCD and configure admin user      | Install HAProxy on Target and Config Port-80 to KinD | Get a file - ArgoCD ApplicationSet | INIT Phase Summary                                                                                                                                                                                                                              |
| Get a file - ArgoCD ApplicationSet  | GitLab                    | Download ArgoCD ApplicationSet manifest       | Install ArgoCD to KinD Cluster      | Write Dowloaded ArgoCD ApplicationSet | GitLab Integration: Downloads ApplicationSet YAML from GitLab repo.                                                                                                                                                                            |
| Write Dowloaded ArgoCD ApplicationSet | Read/Write File           | Write ApplicationSet manifest locally         | Get a file - ArgoCD ApplicationSet  | Open ArgoCD ApplicationSet         | INIT Phase Summary                                                                                                                                                                                                                              |
| Open ArgoCD ApplicationSet          | Execute Command           | Read ApplicationSet manifest content          | Write Dowloaded ArgoCD ApplicationSet | Apply ArgoCD ApplicationSet        | INIT Phase Summary                                                                                                                                                                                                                              |
| Apply ArgoCD ApplicationSet         | Execute Command           | Apply ApplicationSet to ArgoCD in KinD cluster | Open ArgoCD ApplicationSet          | Is INIT Only?                      | INIT Phase Summary                                                                                                                                                                                                                              |
| Set Initialized                   | Set                      | Update progress to TEST phase                  | Is INIT Only?                      | Switch                           | Phase Control Logic                                                                                                                                                                                                                              |
| Is INIT Only?                     | If                        | Check if only INIT phase is requested          | Apply ArgoCD ApplicationSet         | Set Initialized or No Operation     | Phase Control Logic                                                                                                                                                                                                                              |
| No Operation, do nothing           | NoOp                     | Stop workflow if partial run is configured      | Is INIT Only?                      |                                   | Phase Control Logic                                                                                                                                                                                                                              |
| Get a file - ROBOT Script            | GitLab                    | Download Robot Framework test script           | Switch (TEST)                      | Write Dowloaded ROBOT Script       | GitLab Integration: Downloads Robot Framework test script.                                                                                                                                                                                     |
| Write Dowloaded ROBOT Script         | Read/Write File           | Write Robot Framework script locally            | Get a file - ROBOT Script            | Add Robot Framework, Browser Library, Chromium Driver | TEST Phase Summary                                                                                                                                                                                                                               |
| Add Robot Framework, Browser Library, Chromium Driver | Execute Command           | Install dependencies for Robot Framework tests | Write Dowloaded ROBOT Script         | Add Target to Hosts                | TEST Phase Summary                                                                                                                                                                                                                               |
| Add Target to Hosts                | Execute Command           | Add target host entry to local /etc/hosts        | Add Robot Framework, Browser Library, Chromium Driver | Read ROBOT Script to Execute     | TEST Phase Summary                                                                                                                                                                                                                               |
| Read ROBOT Script to Execute         | Read/Write File           | Read test script content for execution           | Add Target to Hosts                 | Open ROBOT Script to Framework     | TEST Phase Summary                                                                                                                                                                                                                               |
| Open ROBOT Script to Framework       | Execute Command           | Prepare test script content for Robot Framework   | Read ROBOT Script to Execute         | Robot Framework                   | TEST Phase Summary                                                                                                                                                                                                                               |
| Robot Framework                   | RobotFramework Node       | Execute Robot Framework tests with browser automation | Open ROBOT Script to Framework       | Pack ROBOT Script Exports         | Sticky Note4: Requires installation of n8n-nodes-robotframework community node.                                                                                                                                                                  |
| Pack ROBOT Script Exports          | Execute Command           | Archive Robot Framework logs and reports           | Robot Framework                   | Read ROBOT Script Exports to Send  | TEST Phase Summary                                                                                                                                                                                                                               |
| Read ROBOT Script Exports to Send    | Read/Write File           | Read archived test results for sending              | Pack ROBOT Script Exports           | Send ROBOT Script Export Pack      | TEST Phase Summary                                                                                                                                                                                                                               |
| Send ROBOT Script Export Pack       | Telegram                  | Send test result archive via Telegram               | Read ROBOT Script Exports to Send    | Is TEST Only?                    | Telegram Notification: Configure Telegram API and chat ID.                                                                                                                                                                                     |
| Set Tested                       | Set                      | Update progress to DESTROY phase                      | Send ROBOT Script Export Pack        | Switch                           | Phase Control Logic                                                                                                                                                                                                                              |
| Is TEST Only?                     | If                        | Check if only TEST phase is requested                | Set Tested                       | Set Tested or No Operation          | Phase Control Logic                                                                                                                                                                                                                              |
| No Operation, do nothing           | NoOp                     | Stop workflow if partial run is configured            | Is TEST Only?                     |                                   | Phase Control Logic                                                                                                                                                                                                                              |
| Check Sshpass Exist Local (D)        | Execute Command           | Check `sshpass` locally for DESTROY phase           | Switch (DESTROY)                  | Is Installed Local (D)             | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Is Installed Local? (D)            | If                        | Decide whether to install sshpass locally for DESTROY | Check Sshpass Exist Local (D)        | Install Sshpass Local (D) or Delete HAProxy on Target | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Install Sshpass Local (D)          | Execute Command           | Install `sshpass` locally for DESTROY phase          | Is Installed Local (D)              | Delete HAProxy on Target           | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Delete HAProxy on Target            | Execute Command           | Remove HAProxy service and config on remote host      | Install Sshpass Local (D)           | Delete KinD Cluster on Target       | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Delete KinD Cluster on Target       | Execute Command           | Delete KinD cluster on remote host                    | Delete HAProxy on Target            | Check KinD on Target (D)            | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Check KinD on Target (D)           | Execute Command           | Verify KinD binary presence post deletion              | Delete KinD Cluster on Target       | Is KinD Installed on Target?1       | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Is KinD Installed on Target?1       | If                        | Decide whether to uninstall KinD binary                | Check KinD on Target (D)            | UnInstall KinD on Target or Check Docker on Target (D) | DESTROY Phase Summary                                                                                                                                                                                                                            |
| UnInstall KinD on Target           | Execute Command           | Remove KinD binary from remote host                     | Is KinD Installed on Target?1       | Check Docker on Target (D)          | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Check Docker on Target (D)         | Execute Command           | Check Docker presence on remote host post KinD uninstall | UnInstall KinD on Target           | Is Docker Installed on Target?1    | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Is Docker Installed on Target?1     | If                        | Decide whether to uninstall Docker                      | Check Docker on Target (D)          | UnInstall Docker on Target or Remove Target from Hosts | DESTROY Phase Summary                                                                                                                                                                                                                            |
| UnInstall Docker on Target         | Execute Command           | Remove Docker and dependencies from remote host         | Is Docker Installed on Target?1     | Remove Target from Hosts           | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Remove Target from Hosts           | Execute Command           | Remove `/etc/hosts` entry for target IP                  | UnInstall Docker on Target          | Set Destroyed                    | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Set Destroyed                    | Set                      | Set progress parameter to CLEAR indicating completion    | Remove Target from Hosts            | Process Finish Report --- Telegam & SMS1 | DESTROY Phase Summary                                                                                                                                                                                                                            |
| Process Finish Report --- Telegam & SMS1 | Execute Command           | Send Telegram notification about workflow completion     | Set Destroyed                    |                                   | Final Notification: Configure Telegram credentials and chat ID. Optional SMS via TextBelt (commented out).                                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add **Manual Trigger** node named `When clicking ‘Execute workflow’`.
   - Add **Webhook** node named `Webhook`:
     - HTTP Method: POST
     - Path: `gitlab-commit`
   - Add **Schedule Trigger** node named `Schedule Trigger`:
     - Interval: daily at 1 AM

2. **Add Parameter Initialization:**

   - Add **Set** node named `Set Parameters`.
   - Define parameters:
     - `target_host` (string): IP of remote host
     - `target_port` (string): SSH port (e.g., 22)
     - `target_user` (string): SSH username
     - `target_password` (string): SSH password (use credentials for security)
     - `progress` (string): initial phase (`INIT`)
     - `progress_only` (string): `false` to run full pipeline, `true` to run single phase
     - `KIND_CONFIG`, `ROBOT_SCRIPT`, `ARGOCD_APPSET`: paths of respective files in GitLab repo

3. **Connect Triggers to Set Parameters:**

   - Connect outputs of Manual Trigger, Webhook, and Schedule Trigger to `Set Parameters`.

4. **Add Phase Routing:**

   - Add **Switch** node named `Switch`.
   - Condition on `progress` parameter with outputs:
     - `init` for `INIT`
     - `test` for `TEST`
     - `destroy` for `DESTROY`
   - Connect `Set Parameters` output to `Switch`.

5. **Build INIT Phase:**

   - Add **Execute Command** node `Check Sshpass Exist Local`: command `which sshpass`.
   - Add **If** node `Is Installed Local?` to check if `sshpass` is missing.
   - Add **Execute Command** node `Install Sshpass Local` to install `sshpass` (supports apt-get, yum, dnf, apk).
   - Connect flow accordingly: `Check Sshpass Exist Local` → `Is Installed Local?` → (`Install Sshpass Local` or continue).
   - Add **Execute Command** node `Check Docker on Target` to SSH and check `docker`.
   - Add **If** node `Is Docker Installed on Target?` to decide Docker install.
   - Add **Execute Command** node `Install Docker on Target` with multi-OS install script, start and enable docker service.
   - Add **Execute Command** node `Check KinD on Target` to check KinD binary on remote.
   - Add **If** node `Is KinD Installed on Target?` to decide KinD install.
   - Add **Execute Command** node `Install KinD on Target` to download KinD binary v0.24.0 and place it in `/usr/local/bin`.
   - Add **GitLab** node `Get a file - KinD Config` to download KinD config YAML from repo.
     - Use OAuth2 credentials configured in n8n.
   - Add **Read/Write File** node `Write Dowloaded KinD Config` to write config to `/tmp/config.yaml`.
   - Add **Execute Command** node `Open KinD Config` to read config file.
   - Add **Execute Command** node `Create KinD Cluster on Target` to upload config via SSH and create cluster.
   - Add **Execute Command** node `Install Helm and Nginx-Ingress in KinD Cluster` to install Helm, ingress-nginx via Helm in KinD container.
   - Add **Execute Command** node `Install HAProxy on Target and Config Port-80 to KinD` to install and configure HAProxy on remote host for port 80 proxying.
   - Add **Execute Command** node `Install ArgoCD to KinD Cluster` to deploy ArgoCD via Helm, configure admin user and ingress.
   - Add **GitLab** node `Get a file - ArgoCD ApplicationSet` to download ApplicationSet manifest.
   - Add **Read/Write File** node `Write Dowloaded ArgoCD ApplicationSet` to write manifest locally.
   - Add **Execute Command** node `Open ArgoCD ApplicationSet` to read manifest content.
   - Add **Execute Command** node `Apply ArgoCD ApplicationSet` to apply manifest inside KinD cluster.
   - Add **Set** node `Set Initialized` to update `progress` to `TEST`.
   - Add **If** node `Is INIT Only?` to check `progress_only` flag:
     - If `false`, continue to next phase.
     - If `true`, connect to **No Operation, do nothing** node to stop.

6. **Build TEST Phase:**

   - Add **GitLab** node `Get a file - ROBOT Script` to download Robot Framework test script.
   - Add **Read/Write File** node `Write Dowloaded ROBOT Script` to write script locally.
   - Add **Execute Command** node `Add Robot Framework, Browser Library, Chromium Driver` to install Python, Robot Framework, browser libs, Chromium.
   - Add **Execute Command** node `Add Target to Hosts` to add target IP -> `autotest.innersite` in `/etc/hosts`.
   - Add **Read/Write File** node `Read ROBOT Script to Execute` to read test script.
   - Add **Execute Command** node `Open ROBOT Script to Framework` to prepare script content.
   - Add **Robot Framework** node (community node) to run tests with browser automation.
     - Set `robotScript` parameter to previous node’s stdout.
     - Enable HTML logs.
   - Add **Execute Command** node `Pack ROBOT Script Exports` to archive logs and reports.
   - Add **Read/Write File** node `Read ROBOT Script Exports to Send` to read archive.
   - Add **Telegram** node `Send ROBOT Script Export Pack` to send results to chat ID.
     - Configure Telegram credentials and update chat ID.
   - Add **Set** node `Set Tested` to update `progress` to `DESTROY`.
   - Add **If** node `Is TEST Only?` to check `progress_only` flag:
     - If `false`, continue to next phase.
     - If `true`, connect to **No Operation, do nothing** node to stop.

7. **Build DESTROY Phase:**

   - Add **Execute Command** node `Check Sshpass Exist Local (D)` and **If** node `Is Installed Local? (D)` to check/install `sshpass` locally.
   - Add **Execute Command** node `Delete HAProxy on Target` to uninstall HAProxy and clean config on remote host.
   - Add **Execute Command** node `Delete KinD Cluster on Target` to delete KinD cluster remotely.
   - Add **Execute Command** node `Check KinD on Target (D)` and **If** node `Is KinD Installed on Target?1` to decide KinD uninstallation.
   - Add **Execute Command** node `UnInstall KinD on Target` to remove KinD binary.
   - Add **Execute Command** node `Check Docker on Target (D)` and **If** node `Is Docker Installed on Target?1` to decide Docker uninstall.
   - Add **Execute Command** node `UnInstall Docker on Target` to remove Docker packages and binaries.
   - Add **Execute Command** node `Remove Target from Hosts` to remove `/etc/hosts` entry.
   - Add **Set** node `Set Destroyed` to update `progress` to `CLEAR`.
   - Add **Execute Command** node `Process Finish Report --- Telegam & SMS1` to send completion notification via Telegram.
     - Configure Telegram bot token and chat ID in command environment variables.

8. **Connect phase outputs back to `Switch` node** for continuous flow control as per `progress_only` flag.

9. **Add Sticky Notes** in the editor to document each phase and node groups with the provided content for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow Overview: This workflow automates the complete lifecycle of Kubernetes testing including INIT, TEST, DESTROY phases with KinD, Robot Framework, and ArgoCD.                                                                                                                                                                                                                                                                         | Workflow high-level description.                                                                   |
| Trigger Configuration: Supports Manual, Webhook (GitLab commit), and Scheduled triggers (daily 1 AM).                                                                                                                                                                                                                                                                                                                                     | Trigger setup instructions.                                                                        |
| Configuration Guide: Requires setting SSH parameters, file paths for KinD config, Robot script, and ArgoCD ApplicationSet.                                                                                                                                                                                                                                                                                                                | Parameter setup instructions.                                                                      |
| GitLab Integration: Downloads configuration and test files from GitLab using OAuth2 credentials.                                                                                                                                                                                                                                                                                                                                           | GitLab nodes require OAuth2 credentials configuration.                                            |
| Phase Control Logic: Switch node routes between INIT, TEST, DESTROY phases based on `progress` parameter; `progress_only` flag controls partial run.                                                                                                                                                                                                                                                                                      | Logic flow control explanation.                                                                    |
| Telegram Notification: Sends test results and completion messages; requires Telegram bot token and chat ID configuration; SMS option via TextBelt is commented out.                                                                                                                                                                                                                                                                          | Telegram setup instructions, find chat ID by messaging bot or group.                              |
| KinD Config Sample: Example KinD YAML config with control-plane and worker nodes, port mappings, and mounts.                                                                                                                                                                                                                                                                                                                               | Included in Sticky Note5 for reference.                                                           |
| Robot Framework Script Sample: Includes Browser library usage to open ArgoCD, login, take screenshots, and teardown. Note must add `executablePath=/usr/bin/chromium-browser` in `New Browser` keyword for browser tests.                                                                                                                                                                                                                   | Included in Sticky Note6 for reference.                                                           |
| ArgoCD ApplicationSet Sample: YAML manifest for ArgoCD to deploy applicationsets from GitHub repo.                                                                                                                                                                                                                                                                                                                                         | Included in Sticky Note7 for reference.                                                           |
| GitLab CI Sample: Example `.gitlab-ci.yml` snippet to trigger this workflow via webhook on commit.                                                                                                                                                                                                                                                                                                                                         | Included in Sticky Note8 for reference.                                                           |
| Important: Must install `n8n-nodes-robotframework` community node to execute Robot Framework tests properly.                                                                                                                                                                                                                                                                                                                               | Highlighted in Sticky Note4.                                                                       |

---

**Disclaimer:**  
The provided text is generated exclusively from an n8n automated workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available.