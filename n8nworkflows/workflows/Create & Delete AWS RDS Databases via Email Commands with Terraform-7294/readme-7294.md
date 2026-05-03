Create & Delete AWS RDS Databases via Email Commands with Terraform

https://n8nworkflows.xyz/workflows/create---delete-aws-rds-databases-via-email-commands-with-terraform-7294


# Create & Delete AWS RDS Databases via Email Commands with Terraform

### 1. Workflow Overview

This workflow automates the management of AWS RDS databases by processing email commands received via Gmail. It listens for emails requesting the creation or deletion of RDS instances, extracts the relevant database parameters from these emails, executes Terraform scripts remotely over SSH to create or destroy the RDS instances, logs the operations in a Google Sheet, and sends confirmation emails back to the requester. The workflow includes robust error handling and status tracking.

The workflow is structured into the following logical blocks:

- **1.1 Trigger & Input Reception**  
  - Listens for incoming Gmail messages to initiate processing.

- **1.2 Email Parsing & Command Extraction**  
  - Extracts operation type (create/delete) and database parameters from the email content.

- **1.3 RDS Instance Management via Terraform**  
  - Uses SSH to connect to a server that runs Terraform commands to create or destroy AWS RDS instances based on extracted parameters.

- **1.4 Operation Logging**  
  - Appends operation details and status into a Google Sheet for audit and tracking.

- **1.5 Confirmation Email Notification**  
  - Sends a styled confirmation email to the requester indicating success or failure of the operation.

- **1.6 Configuration and Documentation Notes**  
  - Sticky notes provide Terraform configuration examples and sample terraform.tfvars variables.


---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

- **Overview:**  
  Watches a Gmail inbox and triggers the workflow when a new email arrives. This is the entry point of the workflow.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**  
  - **Gmail Trigger**  
    - *Type:* Gmail Trigger node (n8n-nodes-base.gmail)  
    - *Role:* Listens for new incoming emails using Gmail OAuth2 credentials.  
    - *Configuration:* Operation set to "trigger" (event-based trigger on new emails).  
    - *Input:* None (trigger node).  
    - *Output:* Email data passed to "Parse Email Content" node.  
    - *Version:* 2.1  
    - *Potential Failures:* OAuth token expiration or revocation, Gmail API quota limits, network errors.  
    - *Notes:* Requires properly configured Gmail OAuth2 credentials with read access.

#### 2.2 Email Parsing & Command Extraction

- **Overview:**  
  Parses incoming email content to extract command (create or delete) and the required database details (identifier, engine, instance class, storage, credentials, etc.).

- **Nodes Involved:**  
  - Parse Email Content (Code node)

- **Node Details:**  
  - **Parse Email Content**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Analyzes the email body to identify the requested operation and extract necessary parameters for Terraform.  
    - *Configuration:* Custom JavaScript code (not explicitly shown) that likely uses regex or string parsing to get parameters.  
    - *Input:* Email data from Gmail Trigger.  
    - *Output:* JSON object with structured fields such as `operation`, `db_identifier`, `db_engine`, `instance_class`, `allocated_storage`, `db_username`, `db_password`, `db_name`, `region`, `requester_email`, `server_user`, `server_ip`, `pwd` (password for SSH), etc.  
    - *Version:* 2  
    - *Potential Failures:* Parsing errors if email format is unexpected, missing required fields, malformed content.  
    - *Mitigation:* Validate input format and handle missing data gracefully.

#### 2.3 RDS Instance Management via Terraform

- **Overview:**  
  Connects via SSH to a remote server that hosts Terraform configurations. Based on the operation (create/delete), it runs Terraform commands to apply or destroy the RDS instance workspace corresponding to the database identifier.

- **Nodes Involved:**  
  - Manage RDS Instance (SSH node)

- **Node Details:**  
  - **Manage RDS Instance**  
    - *Type:* SSH node  
    - *Role:* Runs shell commands remotely to execute Terraform workflows for managing AWS RDS instances.  
    - *Configuration:*  
      - Command is conditionally constructed in JavaScript expression depending on `operation` field.  
      - For `create`: creates/selects Terraform workspace, runs `terraform init`, `terraform plan`, and `terraform apply`.  
      - For `delete`: selects workspace and runs `terraform destroy`.  
      - Uses SSH private key authentication.  
    - *Input:* Parsed JSON with SSH credentials and Terraform workspace info.  
    - *Output:* Resulting SSH command output, presumably Terraform apply/destroy status.  
    - *Credentials:* SSH Private Key configured.  
    - *Version:* 1  
    - *Potential Failures:* SSH connectivity issues, authentication failure, Terraform errors, workspace conflicts, network timeouts, insufficient AWS permissions.  
    - *Mitigation:* Ensure SSH key validity, error trapping in Terraform, and credentials correctness.

#### 2.4 Operation Logging

- **Overview:**  
  Records operation details and statuses into a Google Sheets document for audit trail and monitoring.

- **Nodes Involved:**  
  - Update Google Sheet

- **Node Details:**  
  - **Update Google Sheet**  
    - *Type:* Google Sheets node  
    - *Role:* Appends a new row to a specified Google Sheet with operation details such as timestamps, DB info, requester, and status.  
    - *Configuration:*  
      - Appends data to the sheet named "RDS_Operations" in a particular Google Sheets document (document ID masked as "YOUR_RDS_SHEET_ID").  
      - Maps JSON fields to columns: region, db_name, endpoint, db_engine, operation, timestamp, db_identifier, instance_class, requester_email, operation_status, allocated_storage.  
      - Uses Service Account authentication for API access.  
    - *Input:* JSON with all required fields post Terraform execution.  
    - *Output:* Confirmation of append operation.  
    - *Version:* 4.6  
    - *Potential Failures:* API quota limits, invalid document ID, permission errors, malformed data, network errors.  
    - *Mitigation:* Validate document ID and permissions; implement retries.

#### 2.5 Confirmation Email Notification

- **Overview:**  
  Sends a formatted confirmation email back to the requester, detailing the success or failure of the RDS operation, including a link to the AWS Console.

- **Nodes Involved:**  
  - Send Confirmation Email

- **Node Details:**  
  - **Send Confirmation Email**  
    - *Type:* Gmail node (send email)  
    - *Role:* Sends a confirmation email to the requester with HTML content summarizing the operation outcome.  
    - *Configuration:*  
      - Recipient: dynamically set to `requester_email` extracted from the original email.  
      - Subject and body vary based on operation (`create` or `delete`).  
      - Includes detailed database info, AWS Console URL, notes on credential security, and reminders.  
      - CCs an infrastructure email for monitoring.  
    - *Input:* JSON containing operation result and database info.  
    - *Output:* Email send status.  
    - *Credentials:* Gmail OAuth2 credential.  
    - *Version:* 2.1  
    - *Potential Failures:* OAuth token issues, invalid recipient email, email quota limits, malformed HTML content.  
    - *Mitigation:* Validate email addresses and maintain OAuth credentials.

#### 2.6 Configuration and Documentation Notes

- **Overview:**  
  Sticky notes provide important context and documentation for Terraform configuration files, variables, and terraform.tfvars example.

- **Nodes Involved:**  
  - Workflow Overview (Sticky Note)  
  - Terraform Vars (Sticky Note)  
  - Terraform Config (Sticky Note)

- **Node Details:**  
  - **Workflow Overview**  
    - Explains workflow purpose, features, and trigger.  
  - **Terraform Vars**  
    - Example terraform.tfvars file with typical AWS RDS parameters.  
  - **Terraform Config**  
    - Contains example Terraform configuration files (`main.tf` and `variables.tf`) with AWS provider setup, RDS resource definitions, and output endpoint.

---

### 3. Summary Table

| Node Name             | Node Type        | Functional Role                         | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                              |
|-----------------------|------------------|---------------------------------------|----------------------|------------------------|--------------------------------------------------------------------------------------------------------|
| Gmail Trigger         | Gmail Trigger    | Watches for incoming emails to start workflow | None                 | Parse Email Content     | Workflow Overview sticky note covers overall workflow explanation.                                    |
| Parse Email Content    | Code             | Extracts command and RDS details from email | Gmail Trigger        | Manage RDS Instance     |                                                                                                        |
| Manage RDS Instance    | SSH              | Executes Terraform commands remotely to create/delete RDS | Parse Email Content  | Update Google Sheet     |                                                                                                        |
| Update Google Sheet    | Google Sheets    | Logs operation details into Google Sheets | Manage RDS Instance   | Send Confirmation Email |                                                                                                        |
| Send Confirmation Email| Gmail (Send)     | Sends confirmation email about operation result | Update Google Sheet   | None                   |                                                                                                        |
| Workflow Overview      | Sticky Note      | Describes workflow features and trigger | None                 | None                   | Overview and feature summary for the workflow.                                                        |
| Terraform Vars         | Sticky Note      | Samples terraform.tfvars variable values | None                 | None                   | Shows example terraform.tfvars file with AWS RDS parameters.                                          |
| Terraform Config       | Sticky Note      | Provides Terraform main.tf and variables.tf code | None                 | None                   | Contains Terraform configuration code snippets, including AWS provider and RDS resource example.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Set operation to "trigger" to watch incoming emails.  
   - Configure Gmail OAuth2 credentials with proper scopes to read emails.

2. **Create Parse Email Content node**  
   - Type: Code node (JavaScript)  
   - Connect input from Gmail Trigger.  
   - Implement script to parse incoming email body to extract:  
     - `operation`: "create" or "delete"  
     - `db_identifier`, `db_engine`, `instance_class`, `allocated_storage`, `db_username`, `db_password`, `db_name`, `region`  
     - `requester_email` from sender address  
     - SSH connection info: `server_user`, `server_ip`, `pwd` (SSH password or private key passphrase)  
   - Output JSON with these fields.

3. **Create Manage RDS Instance node**  
   - Type: SSH node  
   - Connect input from Parse Email Content.  
   - Configure SSH private key credentials for remote server access.  
   - In command field, use expression to conditionally run:  
     - For "create":  
       ```
       # Variables
       SERVER_USER="{{ $json.server_user }}"
       SERVER_IP="{{ $json.server_ip }}"
       WORKSPACE_NAME="{{ $json.db_identifier }}"
       PWD="{{ $json.pwd }}"

       # SSH and run Terraform commands
       echo "$PWD" | ssh ${SERVER_USER}@${SERVER_IP} "
           cd /path/to/terraform/project &&
           terraform workspace new ${WORKSPACE_NAME} || terraform workspace select ${WORKSPACE_NAME} &&
           terraform init &&
           terraform plan -out=tfplan &&
           terraform apply -auto-approve tfplan
       "
       ```  
     - For "delete":  
       ```
       # Variables
       SERVER_USER="{{ $json.server_user }}"
       SERVER_IP="{{ $json.server_ip }}"
       WORKSPACE_NAME="{{ $json.db_identifier }}"
       PWD="{{ $json.pwd }}"

       # SSH and run Terraform destroy
       echo "$PWD" | ssh ${SERVER_USER}@${SERVER_IP} "
           cd /path/to/terraform/project &&
           terraform workspace select ${WORKSPACE_NAME} &&
           terraform destroy -auto-approve
       "
       ```  
   - Ensure SSH private key is configured.

4. **Create Update Google Sheet node**  
   - Type: Google Sheets (Append)  
   - Connect input from Manage RDS Instance.  
   - Configure Service Account credentials with access to the target Google Sheet.  
   - Set document ID to your Google Sheet ID.  
   - Set sheet name to "RDS_Operations".  
   - Define mapping of JSON fields to sheet columns:  
     - `timestamp` = current ISO timestamp  
     - `operation`, `db_identifier`, `db_name`, `db_engine`, `instance_class`, `allocated_storage`, `region`, `endpoint` (or "N/A"), `operation_status` (default "Completed"), `requester_email`

5. **Create Send Confirmation Email node**  
   - Type: Gmail (Send)  
   - Connect input from Update Google Sheet.  
   - Use Gmail OAuth2 credentials to send emails.  
   - Configure recipient dynamically from `requester_email` field.  
   - Use conditional expression to set subject and HTML body based on `operation` ("create" or "delete") with database details and AWS Console link.  
   - Add CC to infrastructure monitoring email (e.g., infrastructure@company.com).

6. **Add Sticky Notes for Documentation** (optional but recommended)  
   - Add sticky notes describing:  
     - Workflow overview and features.  
     - Example `terraform.tfvars` file with variables like `aws_region`, `db_identifier`, `db_engine`, `instance_class`, `allocated_storage`, `db_username`, `db_password`, `db_name`.  
     - Terraform configuration snippets for `main.tf` and `variables.tf` showing AWS provider and RDS instance resource.

7. **Connect the nodes as follows:**  
   - Gmail Trigger → Parse Email Content → Manage RDS Instance → Update Google Sheet → Send Confirmation Email

8. **Credential Setup Recap:**  
   - Gmail OAuth2 for reading emails and sending notifications.  
   - SSH private key for connecting to Terraform host server.  
   - Google Service Account for appending to Google Sheets.

9. **Terraform Setup on Remote Server:**  
   - Prepare Terraform project with configuration files (`main.tf`, `variables.tf`, `terraform.tfvars`) as per the sticky notes.  
   - Ensure the Terraform project directory path matches the path used in SSH commands.  
   - Enable AWS CLI access on the server with appropriate permissions.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow automates AWS RDS creation/deletion triggered by Gmail commands with feedback loop.        | Workflow Overview sticky note                                                                     |
| Example terraform.tfvars file includes sample parameters for region, DB identifier, engine, etc.     | Terraform Vars sticky note                                                                        |
| Terraform configuration includes AWS provider setup, RDS resource, and output for RDS endpoint.     | Terraform Config sticky note                                                                      |
| Confirmation emails include styled HTML with database details and AWS console link.                  | Send Confirmation Email node configuration                                                       |
| SSH commands handle Terraform workspace management to isolate each RDS instance per DB identifier.  | Manage RDS Instance node command template                                                         |
| Use of OAuth2 and service account credentials ensures secure API access for Gmail and Google Sheets. | Credential setups in Gmail Trigger, Send Confirmation Email, and Google Sheets nodes               |
| Reminder: Ensure the remote Terraform server has Terraform installed, AWS CLI configured, and access.| Workflow prerequisite outside n8n nodes                                                           |

---

This completes the detailed, structured reference for the "Create & Delete AWS RDS Databases via Email Commands with Terraform" workflow. It provides a comprehensive understanding of each component and instructions for rebuilding or modifying the setup.