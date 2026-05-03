Complete LAMP Stack (Linux, Apache, MySQL, PHP) Automated Server Setup

https://n8nworkflows.xyz/workflows/complete-lamp-stack--linux--apache--mysql--php--automated-server-setup-6136


# Complete LAMP Stack (Linux, Apache, MySQL, PHP) Automated Server Setup

### 1. Workflow Overview

This workflow automates the complete setup of a LAMP stack (Linux, Apache, MySQL, PHP) on a Linux server within approximately 10 seconds. It is designed for developers or system administrators who want to quickly provision a fully functional web server environment with essential development tools and a configured user account.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Parameter Setup:** Receives or sets default parameters required for the server setup, such as server IP, user credentials, PHP and MySQL versions, and domain name.

- **1.2 System Preparation:** Updates the Linux system and installs essential utilities required for the subsequent installations.

- **1.3 Apache Installation and Configuration:** Installs Apache web server, enables necessary modules, configures the virtual host for the specified domain.

- **1.4 MySQL and phpMyAdmin Installation:** Installs MySQL server and client, secures the MySQL installation, creates a development database, installs and configures phpMyAdmin.

- **1.5 PHP Installation and Configuration:** Adds PHP repository, installs PHP with extensions, installs Composer, and configures PHP settings.

- **1.6 Development Tools Installation:** Installs Node.js, npm, development editors (vim, nano, VS Code Server), Git, and PHP development tools (Laravel installer, Symfony CLI, WP-CLI).

- **1.7 Development User Setup:** Creates a non-root development user with sudo privileges, sets up project directories, Git configuration, SSH keys, and permissions.

- **1.8 Final Configuration:** Applies firewall rules, creates sample PHP files for testing, sets permissions, and creates an example environment configuration file.

- **1.9 Setup Completion Summary:** Outputs a summary of the installation and key URLs for access.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Parameter Setup

- **Overview:** Initializes the workflow manually and sets or overrides default parameters for the server setup, including host IP, user names, passwords, PHP and MySQL versions, and domain details.

- **Nodes Involved:**  
  - Start  
  - Set Parameters

- **Node Details:**  

  - **Start**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually  
    - Configuration: No parameters; triggers the workflow on demand  
    - Inputs: None  
    - Outputs: Connects to "Set Parameters"  

  - **Set Parameters**  
    - Type: Set  
    - Role: Defines server setup parameters with default values and allows overrides via incoming JSON  
    - Configuration: Defines strings such as `server_host`, `server_user`, `server_password`, `mysql_root_password`, `php_version`, `mysql_version`, `username`, `user_password`, `domain_name` with defaults and expressions to override if provided  
    - Key expressions: `={{ $json.<param> || '<default>' }}`  
    - Inputs: From "Start"  
    - Outputs: To "System Preparation"  
    - Edge cases: Missing parameters will fall back to defaults; malformed input JSON could cause expression failures  

---

#### 1.2 System Preparation

- **Overview:** Updates the Linux system packages and installs essential base utilities needed for further installations.

- **Nodes Involved:**  
  - System Preparation

- **Node Details:**  

  - **System Preparation**  
    - Type: SSH  
    - Role: Executes shell commands remotely on the Linux server  
    - Configuration: Runs bash script to update packages (`apt update -y && apt upgrade -y`), install tools like curl, wget, git, vim, nano, software-properties-common, apt-transport-https, ca-certificates, gnupg, lsb-release, unzip  
    - Authentication: Uses private key SSH credential  
    - Inputs: From "Set Parameters"  
    - Outputs: To "Install Apache"  
    - Edge cases: SSH connection failure, package install errors, permission issues, network connectivity problems  

---

#### 1.3 Apache Installation and Configuration

- **Overview:** Installs Apache web server, enables common modules, sets up virtual host configuration for the specified domain, and restarts Apache service.

- **Nodes Involved:**  
  - Install Apache

- **Node Details:**  

  - **Install Apache**  
    - Type: SSH  
    - Role: Installs Apache2, enables modules (rewrite, ssl, headers, deflate), configures Apache with a virtual host for the domain, enables the site, reloads Apache  
    - Configuration: Bash script with domain name injected via expression `{{ $json.domain_name }}`; creates `/etc/apache2/sites-available/{{ domain_name }}.conf` with standard directives  
    - Authentication: SSH private key  
    - Inputs: From "System Preparation"  
    - Outputs: To "Install MySQL"  
    - Edge cases: Apache install failure, virtual host config syntax errors, missing permissions, service restart failure  

---

#### 1.4 MySQL and phpMyAdmin Installation

- **Overview:** Installs MySQL server and client, preconfigures root password, secures MySQL installation by removing anonymous users and test DB, creates development database, installs phpMyAdmin, and configures Apache to include phpMyAdmin.

- **Nodes Involved:**  
  - Install MySQL

- **Node Details:**  

  - **Install MySQL**  
    - Type: SSH  
    - Role: Installs MySQL server/client, sets root password non-interactively, secures installation, creates `lamp_dev` DB, installs phpMyAdmin with preconfigured passwords, reloads Apache to include phpMyAdmin configuration  
    - Configuration: Uses expressions for `{{ $json.mysql_root_password }}` in commands and debconf-set-selections  
    - Authentication: SSH private key  
    - Inputs: From "Install Apache"  
    - Outputs: To "Install PHP"  
    - Edge cases: MySQL installation failure, password misconfiguration, service start failure, phpMyAdmin install or config failure, permission issues  

---

#### 1.5 PHP Installation and Configuration

- **Overview:** Adds a PPA for PHP versions, installs specified PHP version with common extensions, installs Composer globally, adjusts PHP configuration for file upload limits, memory, execution time, enables PHP module in Apache, restarts Apache.

- **Nodes Involved:**  
  - Install PHP

- **Node Details:**  

  - **Install PHP**  
    - Type: SSH  
    - Role: Installs PHP `{{ $json.php_version }}` and extensions, installs Composer, modifies php.ini to increase limits, enables PHP module in Apache, restarts Apache  
    - Configuration: Uses expressions for PHP version, runs bash commands with `sed` to update php.ini, moves composer.phar to `/usr/local/bin/composer`  
    - Authentication: SSH private key  
    - Inputs: From "Install MySQL"  
    - Outputs: To "Install Dev Tools"  
    - Edge cases: Repository add failure, PHP install failure, Composer install failure, PHP.ini modification errors, Apache restart failure  

---

#### 1.6 Development Tools Installation

- **Overview:** Installs Node.js and npm for frontend development, installs development tools (git, vim, nano, htop, tree), installs VS Code Server, installs PHP tools (Laravel installer, Symfony CLI), and WordPress CLI (WP-CLI).

- **Nodes Involved:**  
  - Install Dev Tools

- **Node Details:**  

  - **Install Dev Tools**  
    - Type: SSH  
    - Role: Installs Node.js 20.x, development utilities, VS Code Server, and global PHP CLI tools using Composer and curl  
    - Configuration: Installs Node.js via NodeSource setup script, Composer global packages for Laravel and Symfony, downloads WP-CLI and makes it executable  
    - Authentication: SSH private key  
    - Inputs: From "Install PHP"  
    - Outputs: To "Create Dev User"  
    - Edge cases: Network failures during downloads, installation failures, permission issues  

---

#### 1.7 Development User Setup

- **Overview:** Creates a new development user with sudo and www-data group membership, sets password, creates project directories, configures Git globally for the user, generates an SSH key pair, and sets ownership and permissions for `/var/www/html`.

- **Nodes Involved:**  
  - Create Dev User

- **Node Details:**  

  - **Create Dev User**  
    - Type: SSH  
    - Role: Adds user `{{ $json.username }}`, sets password, adds to groups, creates project and backup directories, configures git user name and email, generates SSH RSA key without passphrase, sets permissions on web root  
    - Configuration: Uses expressions for username and password; runs commands as that user using `su - username -c "<command>"`  
    - Authentication: SSH private key  
    - Inputs: From "Install Dev Tools"  
    - Outputs: To "Final Configuration"  
    - Edge cases: User already exists, permission denied, SSH keygen failure  

---

#### 1.8 Final Configuration

- **Overview:** Enables UFW firewall with rules for SSH, HTTP, HTTPS, and development server port 3000; creates sample PHP info and index pages with dynamic content using domain name and MySQL root password; sets permissions; creates example environment file for database and server configuration; outputs summary messages.

- **Nodes Involved:**  
  - Final Configuration

- **Node Details:**  

  - **Final Configuration**  
    - Type: SSH  
    - Role: Configures firewall, creates sample PHP files (`info.php`, `index.php`) with server info and quick links, sets ownership and permissions, creates `.env.example` environment file, outputs installation summary and access URLs  
    - Configuration: Bash script with embedded PHP for dynamic runtime information, uses expressions for domain name, usernames, and passwords  
    - Authentication: SSH private key  
    - Inputs: From "Create Dev User"  
    - Outputs: To "Setup Complete"  
    - Edge cases: Firewall command failures, file write permission issues, PHP code execution issues on server side  

---

#### 1.9 Setup Completion Summary

- **Overview:** Sets final output parameters indicating successful setup, server URLs, user information, and stack details for downstream use or display.

- **Nodes Involved:**  
  - Setup Complete

- **Node Details:**  

  - **Setup Complete**  
    - Type: Set  
    - Role: Constructs final output JSON with keys such as `setup_status`, `server_url`, `phpmyadmin_url`, `dev_user`, `mysql_root_password`, `web_root`, and `installed_stack` including PHP version  
    - Configuration: Uses expressions referencing previous "Set Parameters" node to populate values  
    - Inputs: From "Final Configuration"  
    - Outputs: Terminal node, no further outputs  
    - Edge cases: Expression evaluation failures if prior nodes missing or misconfigured  

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                      | Input Node(s)         | Output Node(s)         | Sticky Note                              |
|---------------------|--------------------|------------------------------------|-----------------------|-----------------------|-----------------------------------------|
| Start               | Manual Trigger     | Entry point to start workflow       | —                     | Set Parameters         | Start LAMP Stack Setup                   |
| Set Parameters      | Set                | Defines initial server parameters   | Start                 | System Preparation     | Configure server parameters              |
| System Preparation  | SSH                | Updates system & installs essentials| Set Parameters         | Install Apache         | Update system & install essentials       |
| Install Apache      | SSH                | Installs and configures Apache      | System Preparation     | Install MySQL          | Install Apache web server                 |
| Install MySQL       | SSH                | Installs MySQL & phpMyAdmin         | Install Apache         | Install PHP            | Install MySQL & phpMyAdmin                |
| Install PHP         | SSH                | Installs PHP, extensions & Composer | Install MySQL          | Install Dev Tools      | Install PHP & extensions                  |
| Install Dev Tools   | SSH                | Installs Node.js, editors, dev tools| Install PHP            | Create Dev User        | Install development tools                 |
| Create Dev User     | SSH                | Creates dev user, dirs, keys, perms | Install Dev Tools      | Final Configuration    | Create development user                   |
| Final Configuration | SSH                | Firewall, sample pages, env setup   | Create Dev User        | Setup Complete         | Final setup & configuration               |
| Setup Complete      | Set                | Outputs final setup summary          | Final Configuration    | —                     | Setup completion summary                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: Start  
   - Purpose: Manual start of the workflow  

2. **Create a Set Node**  
   - Name: Set Parameters  
   - Connect input from Start  
   - Add string parameters with names and default values using expressions to allow override:  
     - `server_host`: `={{ $json.server_host || '192.168.1.100' }}`  
     - `server_user`: `={{ $json.server_user || 'root' }}`  
     - `server_password`: `={{ $json.server_password || 'your_password' }}`  
     - `mysql_root_password`: `={{ $json.mysql_root_password || 'mysql_root_123' }}`  
     - `php_version`: `={{ $json.php_version || '8.2' }}`  
     - `mysql_version`: `={{ $json.mysql_version || '8.0' }}`  
     - `username`: `={{ $json.username || 'webdev' }}`  
     - `user_password`: `={{ $json.user_password || 'web123' }}`  
     - `domain_name`: `={{ $json.domain_name || 'localhost' }}`  

3. **Create SSH Node for System Preparation**  
   - Name: System Preparation  
   - Connect input from Set Parameters  
   - Configure SSH with private key credentials for target server  
   - Command: Bash script to update and upgrade system packages and install essential tools (`curl wget git vim nano software-properties-common apt-transport-https ca-certificates gnupg lsb-release unzip`)  

4. **Create SSH Node for Apache Installation**  
   - Name: Install Apache  
   - Connect input from System Preparation  
   - Use SSH with private key credential  
   - Command: Bash script to install Apache2, enable modules (rewrite, ssl, headers, deflate), create virtual host config for `{{ $json.domain_name }}`, enable site, reload Apache  

5. **Create SSH Node for MySQL Installation**  
   - Name: Install MySQL  
   - Connect input from Install Apache  
   - Use SSH with private key  
   - Command: Bash script to preconfigure MySQL root password, install MySQL server/client, secure installation, create `lamp_dev` database, install phpMyAdmin with pre-seeded debconf answers, include phpMyAdmin Apache config, reload Apache  

6. **Create SSH Node for PHP Installation**  
   - Name: Install PHP  
   - Connect input from Install MySQL  
   - Use SSH with private key  
   - Command: Bash script to add PHP PPA, install PHP `{{ $json.php_version }}` and extensions, install Composer globally, modify php.ini settings, enable PHP module in Apache, restart Apache  

7. **Create SSH Node for Development Tools Installation**  
   - Name: Install Dev Tools  
   - Connect input from Install PHP  
   - Use SSH with private key  
   - Command: Bash script to install Node.js 20.x, git, vim, nano, htop, tree, VS Code Server, Laravel installer, Symfony CLI, WP-CLI  

8. **Create SSH Node for Development User Creation**  
   - Name: Create Dev User  
   - Connect input from Install Dev Tools  
   - Use SSH with private key  
   - Command: Bash script to create user `{{ $json.username }}`, set password, add to sudo and www-data groups, create project directories, configure git user, generate SSH key, set ownership of `/var/www/html`  

9. **Create SSH Node for Final Configuration**  
   - Name: Final Configuration  
   - Connect input from Create Dev User  
   - Use SSH with private key  
   - Command: Bash script to enable UFW firewall with ports 22, 80, 443, 3000, create sample `info.php` and `index.php` pages with embedded PHP dynamic content, set permissions, create `.env.example` file with database and server config, echo summary and access URLs  

10. **Create Set Node for Setup Completion Summary**  
    - Name: Setup Complete  
    - Connect input from Final Configuration  
    - Set string parameters:  
      - `setup_status`: "✅ LAMP Stack Setup Complete!"  
      - `server_url`: "http://{{ $('Set Parameters').item.json.domain_name }}"  
      - `phpmyadmin_url`: "http://{{ $('Set Parameters').item.json.domain_name }}/phpmyadmin"  
      - `dev_user`: "{{ $('Set Parameters').item.json.username }}"  
      - `mysql_root_password`: "{{ $('Set Parameters').item.json.mysql_root_password }}"  
      - `web_root`: "/var/www/html"  
      - `installed_stack`: "Linux + Apache + MySQL + PHP {{ $('Set Parameters').item.json.php_version }}"  

11. **Connect all nodes in sequence:**  
    Start → Set Parameters → System Preparation → Install Apache → Install MySQL → Install PHP → Install Dev Tools → Create Dev User → Final Configuration → Setup Complete  

12. **Credentials Setup:**  
    - Configure SSH private key credential with appropriate access to the target Linux server for all SSH nodes  

13. **Optional:** Add Sticky Notes with relevant content above nodes to clarify purpose  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                          |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Workflow automates LAMP stack setup on Debian-based Linux servers, using apt package manager.  | Context: Linux server automation                                        |
| Uses SSH private key authentication; ensure SSH keys are configured for passwordless access.  | Context: SSH credential setup                                           |
| PHP sample pages include dynamic runtime checks for PHP, Apache versions, and MySQL connectivity.| Useful for quick verification post-installation                        |
| Installs VS Code Server (https://code-server.dev/) for remote development environment.          | Useful for remote coding                                                |
| Includes installation of Laravel installer and Symfony CLI globally for PHP development ease.  | Enhances PHP development capabilities                                  |
| WP-CLI installation allows WordPress management via CLI.                                      | Useful for WordPress developers                                        |
| Firewall configured via UFW to allow essential services; adjust as needed for custom ports.    | Important for server security                                          |
| Workflow designed to run sequentially; long-running SSH commands might need timeout adjustments.| Potential requirement depending on server speed                        |

---

**Disclaimer:**  
This document is generated based on an automated n8n workflow for legal and public server automation tasks. It does not contain illicit or offensive content. Use and modify responsibly.