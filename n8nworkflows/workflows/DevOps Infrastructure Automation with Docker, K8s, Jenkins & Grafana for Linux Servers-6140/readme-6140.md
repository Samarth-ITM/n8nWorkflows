DevOps Infrastructure Automation with Docker, K8s, Jenkins & Grafana for Linux Servers

https://n8nworkflows.xyz/workflows/devops-infrastructure-automation-with-docker--k8s--jenkins---grafana-for-linux-servers-6140


# DevOps Infrastructure Automation with Docker, K8s, Jenkins & Grafana for Linux Servers

### 1. Workflow Overview

This workflow automates the full setup of a DevOps infrastructure stack on a Linux server, targeting users who need to deploy a comprehensive CI/CD and monitoring environment quickly and reliably. The automation covers system preparation, installation of container orchestration and CI/CD tools, security configuration, and final project scaffolding.

**Use cases include:**  
- Rapid provisioning of a Linux server ready for containerized application development and deployment  
- Complete DevOps environment setup including Docker, Kubernetes (K3s), Jenkins, Prometheus, and Grafana  
- Creating a dedicated DevOps user with configured permissions and project directories  
- Enforcing security best practices with firewall rules and installation of common cloud CLIs

**Logical blocks grouped by functional role and node dependencies:**

- **1.1 Input Reception & Parameter Setup**  
  - Manual trigger for start  
  - Configuration of input parameters such as server IP, user credentials, and versions  

- **1.2 System Preparation**  
  - System updates and base package installations including Python automation tools  

- **1.3 Core Tool Installation**  
  - Docker and Docker Compose installation  
  - Kubernetes (K3s) cluster setup with kubectl, Helm, and k9s  
  - Jenkins CI/CD server installation and Docker integration  

- **1.4 Monitoring Stack Installation**  
  - Prometheus and Grafana deployment via Helm charts  

- **1.5 User and Security Configuration**  
  - Creation of a dedicated DevOps user with sudo and Docker group privileges  
  - Firewall rules setup and installation of popular DevOps tools and cloud CLIs  

- **1.6 Final Project Setup and Summary**  
  - Creation of sample Docker Compose, Kubernetes manifests, and Jenkins pipeline files  
  - Output summary of installed tools and access URLs  

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Parameter Setup

- **Overview:** Receives manual start trigger and sets all necessary parameters with defaults and overrides from incoming JSON.
- **Nodes Involved:**  
  - Start DevOps Setup  
  - Configure Parameters  
  - Wait  

- **Node Details:**  
  - **Start DevOps Setup**  
    - Type: Manual Trigger  
    - Role: Entry point to initiate the workflow  
    - Configuration: No parameters; simply triggers workflow manually  
    - Input: None  
    - Output: Connects to Configure Parameters  
    - Failures: None expected  
  - **Configure Parameters**  
    - Type: Set  
    - Role: Define all variables used downstream such as server_host, user names, passwords, versions  
    - Configuration: Sets variables with expressions to fallback to default values if not supplied in input JSON  
    - Expressions example: `={{ $json.server_host || '192.168.1.100' }}`  
    - Input: From Start DevOps Setup  
    - Output: Connects to Wait node  
    - Failures: Expression evaluation failure if input JSON is malformed (unlikely here)  
  - **Wait**  
    - Type: Wait  
    - Role: Pause to allow any asynchronous readiness if needed before system preparation  
    - Configuration: Minimal, with a webhook ID for external trigger if applicable  
    - Input: From Configure Parameters  
    - Output: Connects to System Preparation  
    - Failures: Timeout not configured; minimal risk  

---

#### Block 1.2: System Preparation

- **Overview:** Updates the Linux system and installs essential packages and Python automation libraries.
- **Nodes Involved:**  
  - System Preparation  

- **Node Details:**  
  - **System Preparation**  
    - Type: SSH  
    - Role: Runs a bash script on target Linux server to update OS packages and install required base tools and Python packages  
    - Configuration:  
      - Commands include `apt update -y && apt upgrade -y`  
      - Installs utilities: curl, wget, git, vim, nano, build-essential, software-properties-common, Python3, pip3, openssh-server, ufw, etc.  
      - Installs Python packages: ansible, boto3, kubernetes, docker-compose  
    - Input: From Wait node  
    - Output: Connects to Install Docker  
    - Credentials: SSH Private Key authentication (credential ID ilPh8oO4GfSlc0Qy)  
    - Failures: SSH connection failure, command timeout, apt errors if repository unavailable or network issues  

---

#### Block 1.3: Core Tool Installation

- **Overview:** Installs main DevOps tooling: Docker, Kubernetes (K3s), Jenkins with Docker integration.
- **Nodes Involved:**  
  - Install Docker  
  - Install Kubernetes  
  - Install Jenkins  

- **Node Details:**  
  - **Install Docker**  
    - Type: SSH  
    - Role: Installs Docker Engine and Docker Compose on the server  
    - Configuration: Adds Docker repository and GPG key, installs packages, downloads Docker Compose binary, starts and enables Docker service  
    - Input: From System Preparation  
    - Output: Connects to Install Kubernetes  
    - Failures: Network issues, permission denied errors, Docker service failing to start  
  - **Install Kubernetes**  
    - Type: SSH  
    - Role: Installs lightweight K3s Kubernetes, kubectl CLI, Helm, and k9s for cluster management  
    - Configuration: Uses curl to get install scripts, sets K3s version dynamically, waits 30 seconds for cluster to stabilize  
    - Input: From Install Docker  
    - Output: Connects to Install Jenkins  
    - Failures: Script download failure, install permission errors, K3s startup errors  
  - **Install Jenkins**  
    - Type: SSH  
    - Role: Adds Jenkins repository, installs Java and Jenkins, starts Jenkins service, adds Jenkins user to Docker group  
    - Configuration: Also downloads Jenkins CLI jar (commented as optional), outputs initial admin password and access URL  
    - Input: From Install Kubernetes  
    - Output: Connects to Install Monitoring  
    - Failures: Repository key issues, Java install failures, Jenkins service startup issues  

---

#### Block 1.4: Monitoring Stack Installation

- **Overview:** Deploys Prometheus and Grafana monitoring tools inside the Kubernetes cluster using Helm charts.
- **Nodes Involved:**  
  - Install Monitoring  

- **Node Details:**  
  - **Install Monitoring**  
    - Type: SSH  
    - Role: Creates Kubernetes namespace, adds Helm repos for Prometheus and Grafana, installs them with specific configurations, patches services to NodePort  
    - Configuration: Persistent storage enabled for Grafana, retention configured for Prometheus, admin password preset  
    - Input: From Install Jenkins  
    - Output: Connects to Create DevOps User  
    - Failures: Helm repo access errors, Kubernetes API issues, resource quota problems, Helm installation conflicts  

---

#### Block 1.5: User and Security Configuration

- **Overview:** Creates a dedicated user for DevOps tasks, configures permissions, SSH keys, and sets up firewall and installs additional tools.
- **Nodes Involved:**  
  - Create DevOps User  
  - Security Configuration  

- **Node Details:**  
  - **Create DevOps User**  
    - Type: SSH  
    - Role: Creates Linux user with specified username and password, adds to sudo and docker groups, configures file structure and Git, copies Kubernetes config, generates SSH keys  
    - Configuration: Uses variables for user and password, sets ownership correctly  
    - Input: From Install Monitoring  
    - Output: Connects to Security Configuration  
    - Failures: User creation conflicts, permission denied, SSH key generation failure  
  - **Security Configuration**  
    - Type: SSH  
    - Role: Enables UFW firewall and opens ports for SSH, HTTP, HTTPS, Jenkins, Prometheus, Grafana, Kubernetes API, installs VS Code, Helm, Terraform via snap, installs cloud CLIs for AWS, Azure, Google Cloud  
    - Configuration: Uses command scripts and curl installers for cloud CLIs, enables firewall with force  
    - Input: From Create DevOps User  
    - Output: Connects to Final Configuration  
    - Failures: Firewall conflicts, snap install errors, network connectivity issues  

---

#### Block 1.6: Final Project Setup and Summary

- **Overview:** Sets up example project files for Docker Compose, Kubernetes deployments, Jenkins pipeline, sets ownership and outputs a summary.
- **Nodes Involved:**  
  - Final Configuration  
  - Setup Complete  

- **Node Details:**  
  - **Final Configuration**  
    - Type: SSH  
    - Role: Creates sample Docker Compose YAML, Kubernetes deployment/service manifests, Jenkinsfile pipeline script in the DevOps user’s project directories, displays summary of versions and URLs  
    - Configuration: Uses heredocs to write configuration files, ownership set to devops user  
    - Input: From Security Configuration  
    - Output: Connects to Setup Complete  
    - Failures: File write permission errors, script errors, missing directories  
  - **Setup Complete**  
    - Type: Set  
    - Role: Outputs final JSON summary with status, server info, user credentials, installed tools, and access URLs for Jenkins, Grafana, Prometheus  
    - Configuration: Uses expressions referencing Configure Parameters node for dynamic values  
    - Input: From Final Configuration  
    - Output: None (end of workflow)  
    - Failures: Expression evaluation errors if upstream nodes fail or data missing  

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                       | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                   |
|---------------------|-----------------------|------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Start DevOps Setup   | Manual Trigger        | Entry point to start the workflow   | None                   | Configure Parameters      |                                                                                                              |
| Configure Parameters | Set                   | Define server, versions, credentials| Start DevOps Setup      | Wait                     | ## Main Components - **Configure Parameters** - Defines server details, tool versions, and credentials       |
| Wait                | Wait                  | Pause before running system prep    | Configure Parameters    | System Preparation        |                                                                                                              |
| System Preparation   | SSH                   | Update system & install base packages| Wait                   | Install Docker            | ## Main Components - **System Preparation** - Updates the system and installs base packages                   |
| Install Docker       | SSH                   | Install Docker Engine & Compose     | System Preparation      | Install Kubernetes        | ## Main Components - **Install Docker** - Deploys Docker Engine and Docker Compose                            |
| Install Kubernetes   | SSH                   | Install K3s, kubectl, Helm, k9s    | Install Docker          | Install Jenkins           | ## Main Components - **Install Kubernetes** - Sets up K3s cluster with kubectl, Helm, and k9s                 |
| Install Jenkins      | SSH                   | Install Jenkins CI/CD & Docker group| Install Kubernetes      | Install Monitoring        | ## Main Components - **Install Jenkins** - Configures Jenkins CI/CD server with Docker integration           |
| Install Monitoring   | SSH                   | Deploy Prometheus & Grafana         | Install Jenkins         | Create DevOps User        | ## Main Components - **Install Monitoring** - Deploys Prometheus and Grafana using Helm charts                |
| Create DevOps User   | SSH                   | Create user, configure SSH & Git    | Install Monitoring      | Security Configuration    | ## Main Components - **Create DevOps User** - Establishes a dedicated user with appropriate permissions       |
| Security Configuration | SSH                 | Configure firewall & install tools  | Create DevOps User      | Final Configuration       | ## Main Components - **Security Configuration** - Implements firewall, VS Code, and Terraform                |
| Final Configuration  | SSH                   | Create project files, output summary| Security Configuration  | Setup Complete            | ## Main Components - **Final Configuration** - Sets up sample projects and configuration files               |
| Setup Complete       | Set                   | Output final setup summary           | Final Configuration     | None                     | ## Main Components - **Setup Complete** - Provides a summary and access details                              |
| Sticky Note1         | Sticky Note           | Documentation note                   | None                   | None                     | ## Main Components - Lists main nodes and their roles in the workflow                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `Start DevOps Setup` to manually start the workflow.

2. **Create a Set node** named `Configure Parameters`:
   - Define string parameters with expressions defaulting if not provided in input JSON:  
     - `server_host`: `={{ $json.server_host || '192.168.1.100' }}`  
     - `server_user`: `={{ $json.server_user || 'root' }}`  
     - `server_password`: `={{ $json.server_password || 'your_password' }}`  
     - `docker_version`: `={{ $json.docker_version || 'latest' }}`  
     - `k8s_version`: `={{ $json.k8s_version || '1.28' }}`  
     - `devops_user`: `={{ $json.devops_user || 'devops' }}`  
     - `user_password`: `={{ $json.user_password || 'devops123' }}`  
     - `cluster_name`: `={{ $json.cluster_name || 'devops-cluster' }}`  
   - Connect `Start DevOps Setup` output to this node.

3. **Create a Wait node** named `Wait` with default settings, connect output of `Configure Parameters` to this node.

4. **Create an SSH node** named `System Preparation`:
   - Credentials: configure SSH with private key (e.g., credential ID for target server).  
   - Command: Bash script to update the system, install essential packages, and Python libraries (ansible, boto3, kubernetes, docker-compose).  
   - Connect `Wait` output to this node.

5. **Create an SSH node** named `Install Docker`:
   - Same SSH credentials as above.  
   - Command: Bash script to add Docker repo, install Docker Engine, Docker Compose, start Docker service.  
   - Connect `System Preparation` output to this node.

6. **Create an SSH node** named `Install Kubernetes`:
   - Same SSH credentials.  
   - Command: Bash script to install K3s with version from `k8s_version` parameter, install kubectl, Helm, k9s, wait for readiness.  
   - Connect `Install Docker` output.

7. **Create an SSH node** named `Install Jenkins`:
   - Same SSH credentials.  
   - Command: Bash script to add Jenkins repo, install Java, Jenkins, start Jenkins service, configure Docker group.  
   - Connect `Install Kubernetes` output.

8. **Create an SSH node** named `Install Monitoring`:
   - Same SSH credentials.  
   - Command: Bash script to create Kubernetes monitoring namespace, add Helm repos, install Prometheus and Grafana with configurations, patch services to NodePort.  
   - Connect `Install Jenkins` output.

9. **Create an SSH node** named `Create DevOps User`:
   - Same SSH credentials.  
   - Command: Create user with username and password from parameters, add to sudo and docker groups, create project directories, copy kubeconfig, generate SSH key, configure Git.  
   - Connect `Install Monitoring` output.

10. **Create an SSH node** named `Security Configuration`:
    - Same SSH credentials.  
    - Command: Enable UFW firewall, allow ports (22, 80, 443, 8080, 9090, 3000, 6443), install VS Code, Helm, Terraform, AWS CLI, Azure CLI, Google Cloud SDK.  
    - Connect `Create DevOps User` output.

11. **Create an SSH node** named `Final Configuration`:
    - Same SSH credentials.  
    - Command: Create sample Docker Compose file, Kubernetes deployment and service YAML, Jenkins pipeline file under the DevOps user’s project directory, set ownership, echo summary including versions and URLs.  
    - Connect `Security Configuration` output.

12. **Create a Set node** named `Setup Complete`:
    - Define string outputs summarizing the setup with expressions referencing `Configure Parameters` node for dynamic values such as:  
      - `setup_status`: "✅ DevOps Stack Setup Complete!"  
      - `server_info`: "Host: {{ $('Configure Parameters').item.json.server_host }}"  
      - `devops_user`: "Username: {{ $('Configure Parameters').item.json.devops_user }}"  
      - `user_password`: "Password: {{ $('Configure Parameters').item.json.user_password }}"  
      - `tools_installed`: "Docker, Kubernetes, Jenkins, Prometheus, Grafana, Helm, Terraform"  
      - `jenkins_url`, `grafana_url`, `prometheus_url`: Construct URLs using the server host and standard ports  
    - Connect `Final Configuration` output to this node.

13. **Connections:**  
    - `Start DevOps Setup` → `Configure Parameters` → `Wait` → `System Preparation` → `Install Docker` → `Install Kubernetes` → `Install Jenkins` → `Install Monitoring` → `Create DevOps User` → `Security Configuration` → `Final Configuration` → `Setup Complete`.

14. **Credentials Setup:**  
    - SSH credentials must use private key authentication with access to the Linux server where setup is performed.  
    - Ensure firewall and server allow SSH connections.  

15. **Defaults & Constraints:**  
    - The workflow uses default IP `192.168.1.100` and user `root` unless overridden in JSON input.  
    - Kubernetes version default is 1.28.  
    - Passwords and usernames should be secured when running in production.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow automates a complex multi-tool DevOps environment setup in around 10 seconds, suitable for Linux servers with SSH access.                                                                                                                | Workflow title: "Automate Full DevOps Infrastructure Setup on Linux Server in Just 10 Seconds"      |
| Main components include Docker, K3s Kubernetes, Jenkins, Prometheus, Grafana, Helm, Terraform, and cloud CLIs for AWS, Azure, and GCP.                                                                                                                | Sticky Note content in workflow                                                                          |
| The workflow requires a Linux server with SSH key authentication access and internet connectivity to download and install packages.                                                                                                                   |                                                                                                    |
| Firewall ports opened include SSH (22), HTTP (80), HTTPS (443), Jenkins (8080), Prometheus (9090), Grafana (3000), Kubernetes API (6443).                                                                                                              | Security Configuration node script details                                                          |
| For monitoring access URLs, the workflow extracts the server IP dynamically and outputs them at the end for easy access.                                                                                                                                 | Final Configuration and Setup Complete nodes                                                        |
| SSH node scripts use bash heredocs for multi-line file creation, which can be adapted as needed for different project structures or environments.                                                                                                      | Final Configuration node                                                                              |
| This workflow can be used as a baseline for automated infrastructure provisioning in CI/CD pipelines or cloud-init scripts with minor adjustments.                                                                                                       | General application                                                                                  |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, a workflow automation tool. All content complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.