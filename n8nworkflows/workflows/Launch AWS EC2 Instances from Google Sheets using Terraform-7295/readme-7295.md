Launch AWS EC2 Instances from Google Sheets using Terraform

https://n8nworkflows.xyz/workflows/launch-aws-ec2-instances-from-google-sheets-using-terraform-7295


# Launch AWS EC2 Instances from Google Sheets using Terraform

### 1. Workflow Overview

This n8n workflow automates the launching of AWS EC2 instances based on requests recorded in a Google Sheets spreadsheet. It is designed for IT teams or DevOps engineers who want to manage EC2 instance deployment requests centrally via a spreadsheet and automate the provisioning process using Terraform with AWS. The workflow includes scheduled or manual triggers, reading requests from Google Sheets, executing Terraform commands via SSH to provision EC2 instances, updating the spreadsheet with instance details and status, and sending confirmation emails to requesters.

**Logical Blocks:**

- **1.1 Trigger Block:** Scheduled trigger to start the workflow daily at 10 AM or manual execution.
- **1.2 Input Data Extraction:** Reads EC2 launch requests from a Google Sheets document.
- **1.3 Terraform Execution via SSH:** Runs Terraform commands on a remote server over SSH to provision EC2 instances.
- **1.4 Google Sheets Update:** Appends launched instance details and statuses back into a separate Google Sheets tab.
- **1.5 Email Notification:** Sends rich HTML confirmation emails to requesters with EC2 instance details.
- **1.6 Documentation/Reference:** Sticky notes providing Terraform configuration and variable examples for context.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** Initiates the workflow daily at 10 AM (UTC) or allows for manual triggering.
- **Nodes Involved:** `Google Sheets Trigger`
- **Node Details:**
  - **Type:** Schedule Trigger
  - **Configuration:** Set to trigger once daily at 10:00 AM (hour-based interval).
  - **Input/Output:** No input; output triggers the next node `Extract Instance Details`.
  - **Potential Failures:** Scheduler misconfiguration or time zone misunderstandings.
  - **Version:** 1.2

#### 2.2 Input Data Extraction

- **Overview:** Retrieves EC2 launch request data from the Google Sheets tab named "Launch_Requests".
- **Nodes Involved:** `Extract Instance Details`
- **Node Details:**
  - **Type:** Google Sheets
  - **Configuration:** Reads rows from sheet "Launch_Requests" in a Google Sheets document identified by document ID placeholder `YOUR_EC2_SHEET_ID`.
  - **Authentication:** Uses a Google Service Account credential.
  - **Key Expressions:** None explicitly, but outputs JSON with instance parameters such as AMI ID, instance type, key pair, etc.
  - **Input:** Trigger from Schedule Trigger node.
  - **Output:** Sends extracted data to `Launch EC2 Instance`.
  - **Potential Failures:** Authentication errors, missing or malformed sheets, empty rows, API rate limits.
  - **Version:** 4.6

#### 2.3 Terraform Execution via SSH

- **Overview:** Connects to a remote server via SSH and executes Terraform commands to create the EC2 instance as per extracted parameters.
- **Nodes Involved:** `Launch EC2 Instance`
- **Node Details:**
  - **Type:** SSH
  - **Configuration:** Runs a shell script that sets environment variables from input JSON, then initializes Terraform, selects or creates a workspace, plans, and applies changes automatically.
  - **Key Expressions:** Uses Mustache expressions like `{{ $json.server_user }}`, `{{ $json.server_ip }}`, and Terraform workspace name from input.
  - **Input:** Receives instance parameter JSON from `Extract Instance Details`.
  - **Output:** Passes result to `Update Google Sheet`.
  - **Authentication:** SSH private key credential.
  - **Version:** 1
  - **Potential Failures:** SSH connection failure, incorrect credentials, Terraform errors (syntax, provider issues), workspace conflicts, timeouts.
  - **Notes:** Assumes Terraform project and scripts are preconfigured on the remote server at `/path/to/terraform/project`.

#### 2.4 Google Sheets Update

- **Overview:** Appends the launched EC2 instance details and status information into another Google Sheets tab "Launched_Instances".
- **Nodes Involved:** `Update Google Sheet`
- **Node Details:**
  - **Type:** Google Sheets
  - **Configuration:** Append operation, mapping multiple fields such as AMI ID, instance ID, public/private IP, status, timestamps, requester email, etc.
  - **Authentication:** Service Account credential identical to input extraction node.
  - **Input:** Receives data from `Launch EC2 Instance`.
  - **Output:** Triggers `Send Confirmation Email`.
  - **Potential Failures:** API rate limits, authentication errors, schema mismatches, network issues.
  - **Version:** 4.6

#### 2.5 Email Notification

- **Overview:** Sends an HTML-formatted email to the requester confirming the successful launch of the EC2 instance, including detailed instance info and useful links.
- **Nodes Involved:** `Send Confirmation Email`
- **Node Details:**
  - **Type:** Gmail
  - **Configuration:** Uses OAuth2 Gmail credential, sends to `requester_email` with subject referencing instance name.
  - **Message Content:** Rich HTML including instance details, network info, notes, and an AWS Console direct link.
  - **Input:** Receives data from `Update Google Sheet`.
  - **Potential Failures:** OAuth token expiration, quota limits, invalid email addresses, email formatting issues.
  - **Version:** 2.1

#### 2.6 Documentation/Reference

- **Overview:** Provides inline notes with Terraform `main.tf`, `variables.tf`, and `terraform.tfvars` example snippets for user reference.
- **Nodes Involved:** `Workflow Overview`, `Sticky Note`, `Sticky Note1`
- **Node Details:**
  - **Type:** Sticky Note
  - **Content:** Contains HCL code snippets explaining Terraform provider setup, EC2 resource configuration, variables definition, and example variable values.
  - **Position:** Placed visually around the workflow for documentation.
  - **No input/output or execution role.**

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                              | Input Node(s)        | Output Node(s)          | Sticky Note                                                                                                                |
|-----------------------|---------------------|----------------------------------------------|----------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview     | Sticky Note         | Documentation and workflow summary            |                      |                         | ## 🚀 AWS EC2 Auto Launcher Features and triggers description                                                             |
| Sticky Note           | Sticky Note         | Terraform main.tf and variables.tf reference  |                      |                         | Terraform main.tf and variables.tf example configuration code                                                              |
| Sticky Note1          | Sticky Note         | terraform.tfvars example values                |                      |                         | terraform.tfvars example values snippet                                                                                     |
| Google Sheets Trigger | Schedule Trigger    | Starts workflow daily at 10 AM or manually    |                      | Extract Instance Details |                                                                                                                            |
| Extract Instance Details | Google Sheets      | Reads EC2 launch request data from spreadsheet | Google Sheets Trigger | Launch EC2 Instance       |                                                                                                                            |
| Launch EC2 Instance   | SSH                 | Executes Terraform commands to launch EC2     | Extract Instance Details | Update Google Sheet      | Assumes preconfigured Terraform project on SSH host; uses SSH private key auth                                             |
| Update Google Sheet   | Google Sheets       | Appends launched instance details to spreadsheet | Launch EC2 Instance   | Send Confirmation Email  |                                                                                                                            |
| Send Confirmation Email | Gmail              | Sends confirmation email with instance details | Update Google Sheet   |                         |                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to daily, trigger at 10:00 AM (hour-based)  
   - Name: `Google Sheets Trigger`

2. **Create Google Sheets Node to Extract Requests**  
   - Type: Google Sheets  
   - Operation: Read rows from sheet "Launch_Requests"  
   - Document ID: Set your Google Sheets document ID (replace `YOUR_EC2_SHEET_ID`)  
   - Authentication: Use Service Account credential with access to the Google Sheet  
   - Name: `Extract Instance Details`  
   - Connect output of `Google Sheets Trigger` to input of this node

3. **Create SSH Node to Run Terraform**  
   - Type: SSH  
   - Command:  
     ```
     # Variables
     SERVER_USER="{{ $json.server_user }}"
     SERVER_IP="{{ $json.server_ip }}"
     WORKSPACE_NAME="{{ $json.workspace_name }}"
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
   - Authentication: Set up SSH private key credential with private key and username  
   - Name: `Launch EC2 Instance`  
   - Connect output of `Extract Instance Details` to input of this node

4. **Create Google Sheets Node to Update Launched Instances Sheet**  
   - Type: Google Sheets  
   - Operation: Append rows to sheet "Launched_Instances"  
   - Document ID: Same as above  
   - Authentication: Service Account credential  
   - Define columns mapping with fields like `ami_id`, `region`, `instance_id`, `public_ip`, `launch_status`, `requester_email`, etc.  
   - Name: `Update Google Sheet`  
   - Connect output of `Launch EC2 Instance` to input of this node

5. **Create Gmail Node to Send Confirmation Email**  
   - Type: Gmail  
   - Authentication: Gmail OAuth2 credentials  
   - Send To: Expression `{{ $json.requester_email }}`  
   - Subject: `✅ AWS EC2 Instance Launched Successfully - {{ $json.instance_name }}`  
   - Message: Use the provided rich HTML template, referencing instance details with expressions like `{{ $json.instance_name }}`, `{{ $json.instance_id }}`, etc.  
   - CC List: `infrastructure@company.com`  
   - Name: `Send Confirmation Email`  
   - Connect output of `Update Google Sheet` to input of this node

6. **Add Sticky Notes for Documentation**  
   - Add Sticky Notes nodes with Terraform code snippets (`main.tf`, `variables.tf`, `terraform.tfvars`) and a workflow overview note as per the original content for user reference.

7. **Credential Setup**  
   - Google Sheets nodes require a Google API Service Account credential with read/write access to the target spreadsheet.  
   - SSH node requires a private key credential for the remote server where Terraform is installed and preconfigured.  
   - Gmail node requires OAuth2 credentials authorized to send emails on behalf of the sender.

8. **Testing and Validation**  
   - Test the workflow manually to ensure data is fetched correctly from Google Sheets.  
   - Verify SSH connection and Terraform commands run successfully on the remote host.  
   - Confirm that launched instance details are appended back into Google Sheets.  
   - Validate that confirmation emails are received with correct information.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow reads launch requests from Google Sheets, executes Terraform commands remotely via SSH to automate EC2 provisioning, updates the spreadsheet with instance details, and sends confirmation emails.                                                                                                                                               | Workflow Overview Sticky Note                                                                   |
| Terraform configuration files (`main.tf`, `variables.tf`, `terraform.tfvars`) must be prepared on the remote server where SSH commands are executed. The `main.tf` includes AWS provider and EC2 resource definitions; variables are used for region, AMI, instance type, key pair, and instance name. Example values are provided in `terraform.tfvars`.                          | Sticky Notes containing HCL snippets                                                            |
| AWS credentials and Terraform CLI must be configured on the remote server properly with a profile matching the used AWS CLI profile variable.                                                                                                                                                                                                                | Implied by Terraform and SSH node setup                                                        |
| Gmail OAuth2 credentials must be set up with correct scopes to send emails. The email body uses HTML and must be carefully edited if customized to maintain formatting.                                                                                                                                                                                     | Gmail node details                                                                              |
| The workflow is designed for daily scheduled runs but supports manual triggering if immediate launch is needed.                                                                                                                                                                                                                                           | Workflow overview and schedule trigger settings                                                |
| Replace placeholder Google Sheets document IDs and Terraform project paths with actual values before deploying this workflow.                                                                                                                                                                                                                             | Throughout the workflow, especially Google Sheets nodes and SSH command                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.