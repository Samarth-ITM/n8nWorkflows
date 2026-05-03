Set Up Complete MERN Stack Development Environment on Linux Server

https://n8nworkflows.xyz/workflows/set-up-complete-mern-stack-development-environment-on-linux-server-6131


# Set Up Complete MERN Stack Development Environment on Linux Server

---
### 1. Workflow Overview

This workflow automates the complete setup of a MERN (MongoDB, Express, React, Node.js) stack development environment on a Linux server. It targets developers or system administrators who need to quickly provision a fully configured development server with essential tools, services, and user configurations for MERN development.

The workflow is logically divided into these blocks:

- **1.1 Input Parameter Initialization:** Setting up configurable variables such as server host, user credentials, software versions, and setup preferences.
- **1.2 System Preparation:** Updating and preparing the base Linux system with essential utilities.
- **1.3 Core Software Installation:** Sequential installation of Node.js & npm, MongoDB (with Compass and Shell), Git & GitHub CLI.
- **1.4 Development Tools Installation:** Installing popular development tools such as VS Code, Docker, Postman, Nginx, Redis, and PostgreSQL.
- **1.5 User Setup:** Creating a dedicated development user with configured SSH keys and Git settings.
- **1.6 Additional MERN Tools Installation:** Installing alternative package managers, deployment CLIs, and build tools.
- **1.7 Final Configuration:** Configuring firewall rules, environment variable templates, project directory structure, and ownership.
- **1.8 Completion Summary:** Outputting a final status message and summary details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Parameter Initialization

- **Overview:** Initializes all required parameters used throughout the workflow with defaults or from input JSON. This ensures flexibility for different server setups.
- **Nodes Involved:** `Set Parameters`
- **Node Details:**
  - Type: Set Node
  - Role: Define parameters such as `server_host`, `server_user`, `server_password`, `setup_type`, `node_version`, `mongodb_version`, `username`, `user_password`.
  - Configuration: Uses expressions to fall back on defaults if values are not provided in input JSON.
  - Input: Triggered by `Start` node.
  - Output: Passed to the next block for system preparation.
  - Edge Cases: Missing or incorrect input values could cause subsequent SSH commands to fail.

#### 1.2 System Preparation

- **Overview:** Updates the Linux system and installs essential base utilities and tools necessary for subsequent installations.
- **Nodes Involved:** `System Preparation`
- **Node Details:**
  - Type: SSH Node
  - Role: Runs shell commands remotely to update package lists, upgrade packages, and install core packages like curl, git, python3, snapd.
  - Configuration: Executes a Bash script with multiple install commands.
  - Input: Receives parameters from `Set Parameters`.
  - Output: Proceeds to Node.js installation.
  - Credentials: SSH private key authentication.
  - Edge Cases: Network issues, permission errors, or package repository problems may cause failure.
  - Sticky Note: "Prepares the system for installation"

#### 1.3 Core Software Installation

- **Overview:** Installs Node.js with npm, MongoDB (including Compass GUI and shell), and Git with GitHub CLI sequentially.
- **Nodes Involved:** `Install Node.js & npm`, `Install MongoDB`, `Install Git & GitHub CLI`
- **Node Details:**

  - **Install Node.js & npm**
    - Type: SSH Node
    - Role: Installs Node.js version specified (default v20) using NodeSource repo, verifies installation, installs global npm packages commonly used in MERN development.
    - Key npm packages: create-react-app, @angular/cli, express-generator, nodemon, pm2, typescript, eslint, prettier, etc.
    - Credentials: SSH private key.
    - Edge Cases: Repository availability, network, npm install errors.
    - Sticky Note: "Installs Node.js (v20 by default) with npm"

  - **Install MongoDB**
    - Type: SSH Node
    - Role: Adds MongoDB repository, installs MongoDB server, Compass GUI, and mongosh shell for version specified (default v7.0), starts and enables mongod service.
    - Credentials: SSH private key.
    - Edge Cases: Repository key import issues, service start failures.
    - Sticky Note: "Installs MongoDB (v7.0 by default) with Compass & Shell"

  - **Install Git & GitHub CLI**
    - Type: SSH Node
    - Role: Confirms Git installation, installs GitHub CLI for version control management.
    - Credentials: SSH private key.
    - Edge Cases: Repository access or permission errors.
    - Sticky Note: "Installs Git and GitHub CLI"

#### 1.4 Development Tools Installation

- **Overview:** Installs various essential developer tools and IDEs including VS Code, Postman, Docker (and Compose), Nginx, Redis, and PostgreSQL.
- **Nodes Involved:** `Install Development Tools`
- **Node Details:**
  - Type: SSH Node
  - Role: Adds MS package repository, installs VS Code, Postman via snap, Docker stack, web servers, caching, and alternative databases.
  - Credentials: SSH private key.
  - Edge Cases: Package conflicts, service start failures.
  - Sticky Note: "Installs VS Code, Docker, Docker Compose, Postman, Nginx, Redis, and PostgreSQL"

#### 1.5 User Setup

- **Overview:** Creates a dedicated development user with specified credentials, sets up SSH keys, configures Git global settings, and creates development directories.
- **Nodes Involved:** `Create Dev User`
- **Node Details:**
  - Type: SSH Node
  - Role: Adds user, sets password, adds sudo and docker group memberships, creates directories, generates SSH keys, configures Git.
  - Credentials: SSH private key.
  - Edge Cases: User already exists, permission errors, SSH key generation failures.
  - Sticky Note: "Creates a development user account"

#### 1.6 Additional MERN Tools Installation

- **Overview:** Installs alternative package managers (Yarn, pnpm), deployment CLIs (Heroku, Vercel, Netlify, Firebase, AWS CLI, Google Cloud SDK), and build utilities (webpack, gulp, lerna, nx).
- **Nodes Involved:** `Install Additional Tools`
- **Node Details:**
  - Type: SSH Node
  - Role: Installs and configures multiple global npm and apt packages for deployment and build automation.
  - Credentials: SSH private key.
  - Edge Cases: Network issues, package conflicts.
  - Sticky Note: "Installs package managers (npm, Yarn, pnpm), global npm packages, deployment tools, build tools, and security tools"

#### 1.7 Final Configuration

- **Overview:** Configures firewall rules to allow required ports, creates environment variable templates, sets up a sample MERN project structure with package.json, and sets ownership permissions.
- **Nodes Involved:** `Final Configuration`
- **Node Details:**
  - Type: SSH Node
  - Role: Enables UFW firewall, opens ports for SSH, HTTP, HTTPS, development servers, creates environment template and sample project directory, adjusts permissions.
  - Credentials: SSH private key.
  - Edge Cases: Firewall conflicts, file write permission issues.
  - Sticky Note: "Configures firewall, SSH keys, and environment variables template"

#### 1.8 Completion Summary

- **Overview:** Outputs a final structured message summarizing the setup status, server info, user credentials, installed software, project directory location, and next steps.
- **Nodes Involved:** `Setup Complete`
- **Node Details:**
  - Type: Set Node
  - Role: Sets summary strings using values from the initial `Set Parameters` node.
  - Input: From `Final Configuration`
  - Output: Workflow end.
  - Sticky Note: "Marks the completion of the setup process"

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                                           | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                       |
|-----------------------|---------------------|-----------------------------------------------------------|------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Start                 | Manual Trigger      | Workflow trigger to start the process                     | -                      | Set Parameters              | Start workflow                                                                                   |
| Set Parameters        | Set                 | Initialize configurable parameters with defaults          | Start                  | System Preparation          | Configures server host, user, password, setup type, Node.js version, MongoDB version, username, and user password |
| System Preparation    | SSH                 | Update system and install essential base utilities        | Set Parameters          | Install Node.js & npm       | Prepares the system for installation                                                            |
| Install Node.js & npm | SSH                 | Install Node.js, npm, and global npm packages             | System Preparation      | Install MongoDB             | Installs Node.js (v20 by default) with npm                                                      |
| Install MongoDB       | SSH                 | Install MongoDB server, Compass, and Shell                | Install Node.js & npm   | Install Git & GitHub CLI    | Installs MongoDB (v7.0 by default) with Compass & Shell                                        |
| Install Git & GitHub CLI | SSH               | Install GitHub CLI and confirm Git installation            | Install MongoDB         | Install Development Tools   | Installs Git and GitHub CLI                                                                     |
| Install Development Tools | SSH              | Install VS Code, Docker, Postman, Nginx, Redis, PostgreSQL | Install Git & GitHub CLI| Create Dev User             | Installs VS Code, Docker, Docker Compose, Postman, Nginx, Redis, and PostgreSQL                  |
| Create Dev User       | SSH                 | Create developer user, SSH key, Git config, directories    | Install Development Tools| Install Additional Tools    | Creates a development user account                                                             |
| Install Additional Tools | SSH               | Install Yarn, pnpm, deployment CLIs, build and security tools | Create Dev User        | Final Configuration         | Installs package managers (npm, Yarn, pnpm), global npm packages, deployment tools, build tools, and security tools |
| Final Configuration   | SSH                 | Configure firewall, environment variables, project setup   | Install Additional Tools| Setup Complete              | Configures firewall, SSH keys, and environment variables template                              |
| Setup Complete        | Set                 | Output final setup summary and next steps                  | Final Configuration     | -                           | Marks the completion of the setup process                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node named `Start`.**

2. **Create a Set node named `Set Parameters`:**
   - Configure string parameters with expressions and defaults:
     - `server_host`: `={{ $json.server_host || '192.168.1.100' }}`
     - `server_user`: `{{ $json.server_user || 'root' }}`
     - `server_password`: `{{ $json.server_password || 'your_password' }}`
     - `setup_type`: `={{ $json.setup_type || 'full' }}`
     - `node_version`: `{{ $json.node_version || '20' }}`
     - `mongodb_version`: `{{ $json.mongodb_version || '7.0' }}`
     - `username`: `{{ $json.username || 'developer' }}`
     - `user_password`: `{{ $json.user_password || 'dev123' }}`
   - Connect `Start` → `Set Parameters`.

3. **Create SSH node `System Preparation`:**
   - SSH credentials using private key with proper access to the Linux server.
   - Command:
     ```bash
     #!/bin/bash
     echo "🚀 Starting MERN Stack Development Environment Setup..."
     echo "======================================================"

     echo "📦 Updating system packages..."
     apt update -y && apt upgrade -y

     echo "🔧 Installing essential development tools..."
     apt install -y curl wget git vim nano build-essential software-properties-common apt-transport-https ca-certificates gnupg lsb-release

     apt install -y python3 python3-pip
     apt install -y snapd

     echo "✅ System preparation completed!"
     ```
   - Connect `Set Parameters` → `System Preparation`.

4. **Create SSH node `Install Node.js & npm`:**
   - SSH credentials same as above.
   - Command with embedded expression for node version:
     ```bash
     #!/bin/bash

     echo "📱 Installing Node.js and npm..."
     echo "================================"

     curl -fsSL https://deb.nodesource.com/setup_{{ $json.node_version }}.x | bash -
     apt install -y nodejs

     echo "Node.js version: $(node --version)"
     echo "npm version: $(npm --version)"

     npm install -g create-react-app @angular/cli express-generator nodemon pm2 serve typescript ts-node @nestjs/cli next vite eslint prettier json-server http-server concurrently cross-env dotenv-cli

     echo "✅ Node.js and npm packages installed successfully!"
     ```
   - Connect `System Preparation` → `Install Node.js & npm`.

5. **Create SSH node `Install MongoDB`:**
   - SSH credentials same as above.
   - Command with embedded expression for MongoDB version:
     ```bash
     #!/bin/bash

     echo "🍃 Installing MongoDB..."
     echo "========================"

     wget -qO - https://www.mongodb.org/static/pgp/server-{{ $json.mongodb_version }}.asc | apt-key add -
     echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/{{ $json.mongodb_version }} multiverse" | tee /etc/apt/sources.list.d/mongodb-org-{{ $json.mongodb_version }}.list

     apt update
     apt install -y mongodb-org

     systemctl start mongod
     systemctl enable mongod

     wget https://downloads.mongodb.com/compass/mongodb-compass_1.40.4_amd64.deb
     dpkg -i mongodb-compass_1.40.4_amd64.deb
     apt-get install -f -y

     wget -qO - https://www.mongodb.org/static/pgp/server-{{ $json.mongodb_version }}.asc | apt-key add -
     echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/{{ $json.mongodb_version }} multiverse" | tee /etc/apt/sources.list.d/mongodb-org-{{ $json.mongodb_version }}.list
     apt update
     apt install -y mongodb-mongosh

     echo "MongoDB version: $(mongod --version | head -n 1)"
     echo "MongoDB Shell version: $(mongosh --version)"

     echo "✅ MongoDB installed and started successfully!"
     ```
   - Connect `Install Node.js & npm` → `Install MongoDB`.

6. **Create SSH node `Install Git & GitHub CLI`:**
   - SSH credentials same as above.
   - Command:
     ```bash
     #!/bin/bash

     echo "🐙 Installing Git and Version Control Tools..."
     echo "============================================="

     echo "Git version: $(git --version)"

     curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
     echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null
     apt update
     apt install -y gh

     echo "GitHub CLI version: $(gh --version)"
     echo "✅ Git and version control tools installed successfully!"
     ```
   - Connect `Install MongoDB` → `Install Git & GitHub CLI`.

7. **Create SSH node `Install Development Tools`:**
   - SSH credentials same as above.
   - Command:
     ```bash
     #!/bin/bash

     echo "💻 Installing Development Tools and IDEs..."
     echo "=========================================="

     wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
     install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
     echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/trusted.gpg.d/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list
     apt update
     apt install -y code

     snap install postman

     apt install -y docker.io docker-compose
     systemctl start docker
     systemctl enable docker

     apt install -y nginx
     systemctl start nginx
     systemctl enable nginx

     apt install -y redis-server
     systemctl start redis-server
     systemctl enable redis-server

     apt install -y postgresql postgresql-contrib
     systemctl start postgresql
     systemctl enable postgresql

     echo "✅ Development tools installed successfully!"
     ```
   - Connect `Install Git & GitHub CLI` → `Install Development Tools`.

8. **Create SSH node `Create Dev User`:**
   - SSH credentials same as above.
   - Command with expressions for username and password:
     ```bash
     #!/bin/bash

     echo "👤 Creating Development User Account..."
     echo "======================================"

     useradd -m -s /bin/bash {{ $json.username }}
     echo "{{ $json.username }}:{{ $json.user_password }}" | chpasswd

     usermod -aG sudo {{ $json.username }}
     usermod -aG docker {{ $json.username }}

     su - {{ $json.username }} -c "mkdir -p ~/projects ~/websites ~/apis ~/mobile-apps"
     su - {{ $json.username }} -c "mkdir -p ~/tools ~/scripts ~/backup"

     su - {{ $json.username }} -c "ssh-keygen -t rsa -b 4096 -C '{{ $json.username }}@mern-dev' -N '' -f ~/.ssh/id_rsa"

     su - {{ $json.username }} -c "git config --global user.name '{{ $json.username }}'"
     su - {{ $json.username }} -c "git config --global user.email '{{ $json.username }}@example.com'"
     su - {{ $json.username }} -c "git config --global init.defaultBranch main"

     echo "✅ Development user created successfully!"
     echo "Username: {{ $json.username }}"
     echo "Password: {{ $json.user_password }}"
     echo "SSH Key: /home/{{ $json.username }}/.ssh/id_rsa.pub"
     ```
   - Connect `Install Development Tools` → `Create Dev User`.

9. **Create SSH node `Install Additional Tools`:**
   - SSH credentials same as above.
   - Command:
     ```bash
     #!/bin/bash

     echo "🚀 Installing Additional MERN Stack Tools..."
     echo "============================================"

     npm install -g yarn
     npm install -g pnpm

     curl https://cli-assets.heroku.com/install.sh | sh

     npm install -g vercel netlify-cli firebase-tools

     apt install -y awscli

     echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
     curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
     apt update
     apt install -y google-cloud-cli

     npm install -g @storybook/cli webpack webpack-cli parcel-bundler rollup gulp-cli grunt-cli lerna nx

     echo "✅ Additional MERN Stack tools installed successfully!"
     ```
   - Connect `Create Dev User` → `Install Additional Tools`.

10. **Create SSH node `Final Configuration`:**
    - SSH credentials same as above.
    - Command:
      ```bash
      #!/bin/bash

      echo "🔧 Final Configuration and Setup..."
      echo "==================================="

      ufw enable
      ufw allow 22/tcp
      ufw allow 80/tcp
      ufw allow 443/tcp
      ufw allow 3000/tcp
      ufw allow 3001/tcp
      ufw allow 27017/tcp
      ufw allow 5000/tcp
      ufw allow 8000/tcp

      cat > /home/{{ $json.username }}/.env.example << 'EOF'
      # MongoDB Configuration
      MONGO_URI=mongodb://localhost:27017/your-database
      MONGO_DB_NAME=your-database

      # JWT Configuration
      JWT_SECRET=your-super-secret-jwt-key
      JWT_EXPIRE=7d

      # API Configuration
      PORT=5000
      NODE_ENV=development

      # Frontend Configuration
      REACT_APP_API_URL=http://localhost:5000/api
      REACT_APP_SOCKET_URL=http://localhost:5000

      # Email Configuration (optional)
      EMAIL_HOST=smtp.gmail.com
      EMAIL_PORT=587
      EMAIL_USER=your-email@gmail.com
      EMAIL_PASS=your-app-password

      # Cloud Storage (optional)
      CLOUDINARY_CLOUD_NAME=your-cloud-name
      CLOUDINARY_API_KEY=your-api-key
      CLOUDINARY_API_SECRET=your-api-secret

      # Payment Gateway (optional)
      STRIPE_PUBLIC_KEY=your-stripe-public-key
      STRIPE_SECRET_KEY=your-stripe-secret-key
      EOF

      su - {{ $json.username }} -c "mkdir -p ~/projects/sample-mern-app/{client,server,database}"

      cat > /home/{{ $json.username }}/projects/sample-mern-app/package.json << 'EOF'
      {
        "name": "sample-mern-app",
        "version": "1.0.0",
        "description": "A sample MERN stack application",
        "main": "server/index.js",
        "scripts": {
          "dev": "concurrently \"npm run server\" \"npm run client\"",
          "server": "cd server && npm run dev",
          "client": "cd client && npm start",
          "build": "cd client && npm run build",
          "install-deps": "npm install && cd client && npm install && cd ../server && npm install"
        },
        "keywords": ["mern", "mongodb", "express", "react", "nodejs"],
        "author": "{{ $json.username }}",
        "license": "MIT",
        "devDependencies": {
          "concurrently": "^8.2.2"
        }
      }
      EOF

      chown -R {{ $json.username }}:{{ $json.username }} /home/{{ $json.username }}/projects/

      echo "✅ Final configuration completed!"
      echo "🎉 MERN Stack Development Environment Setup Complete!"
      echo "===================================================="
      echo "📊 Installation Summary:"
      echo "• Node.js version: $(node --version)"
      echo "• npm version: $(npm --version)"
      echo "• MongoDB: Installed and running"
      echo "• Git: $(git --version)"
      echo "• Docker: $(docker --version)"
      echo "• VS Code: Installed"
      echo "• Development User: {{ $json.username }}"
      echo "• Project Directory: /home/{{ $json.username }}/projects/"
      echo ""
      echo "🚀 Your MERN Stack development environment is ready!"
      echo "Login as: {{ $json.username }}"
      echo "Password: {{ $json.user_password }}"
      echo "Happy coding! 🎯"
      ```
    - Connect `Install Additional Tools` → `Final Configuration`.

11. **Create Set node `Setup Complete`:**
    - Configure string parameters:
      - `setup_status`: "✅ MERN Stack Development Environment Setup Complete!"
      - `server_info`: `Host: {{ $('Set Parameters').item.json.server_host }}`
      - `dev_user`: `Username: {{ $('Set Parameters').item.json.username }}`
      - `dev_password`: `Password: {{ $('Set Parameters').item.json.user_password }}`
      - `installed_tools`: "Node.js, MongoDB, Git, Docker, VS Code, Postman, Nginx, Redis, PostgreSQL"
      - `project_directory`: `/home/{{ $('Set Parameters').item.json.username }}/projects/`
      - `next_steps`: "SSH into server, switch to dev user, and start building MERN applications!"
    - Connect `Final Configuration` → `Setup Complete`.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow automates full MERN stack environment setup on Linux server in ~10 seconds after trigger.        | Workflow title and description                       |
| Uses SSH private key authentication for secure remote command execution.                                  | Workflow credential setup                            |
| Installs a broad suite of global npm packages commonly used in MERN development including build tools.    | Node.js installation node                           |
| Sets up a development user with SSH keys and Git global configs for easier secure access and version control. | User creation node                                  |
| Configures firewall with UFW to allow necessary ports for development servers and database access.        | Final Configuration node                            |
| Installs popular deployment CLI tools: Heroku, Vercel, Netlify, Firebase, AWS CLI, Google Cloud SDK.     | Additional Tools installation node                  |
| Provides a sample project structure and example package.json to bootstrap MERN projects.                   | Final Configuration node                            |
| Sticky notes provide quick context on each major node.                                                   | See nodes with sticky notes in workflow visual editor |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.