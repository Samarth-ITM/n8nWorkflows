Create, Delete, and Organize AWS S3 Buckets & Files Directly from Your Email

https://n8nworkflows.xyz/workflows/create--delete--and-organize-aws-s3-buckets---files-directly-from-your-email-7244


# Create, Delete, and Organize AWS S3 Buckets & Files Directly from Your Email

### 1. Workflow Overview

This workflow automates AWS S3 bucket and file management by interpreting email commands and performing the requested S3 operations. It supports creating and deleting buckets, uploading, downloading, copying, deleting files, and listing files within buckets. After executing the requested operation, it sends a confirmation or failure notification email back to the requester.

Logical blocks of the workflow:

- **1.1 Input Reception**: Reads incoming emails with specific criteria to trigger S3 commands.
- **1.2 Email Content Parsing**: Extracts the type of S3 operation and relevant parameters from the email body.
- **1.3 Task Routing**: Determines which S3 operation to perform based on parsed command.
- **1.4 AWS S3 Operations**: Performs the requested operation such as bucket creation, deletion, file upload, download, copy, delete, or listing files.
- **1.5 Outcome Evaluation**: Checks if the operation succeeded or failed.
- **1.6 Notification Dispatch**: Sends email notifications for success or failure of the operation.
- **1.7 Documentation and Guidance**: Sticky notes explaining workflow purpose and operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block reads unread emails with "s3" in their subject line to trigger the workflow.

- **Nodes Involved:**  
  - Start Workflow (GET Request)

- **Node Details:**  

  - **Start Workflow (GET Request)**  
    - Type: Email Read (IMAP)  
    - Role: Monitors an IMAP email inbox for unread messages with subjects containing "s3".  
    - Configuration: Searches for unseen emails with subject containing "s3".  
    - Credentials: Uses configured IMAP credentials for authentication.  
    - Inputs: None (trigger node).  
    - Outputs: Email items containing the full email data.  
    - Edge Cases: Possible authentication errors if credentials expire; email fetching delays; malformed emails.

#### 2.2 Email Content Parsing

- **Overview:**  
  Parses the plain text body of the incoming email to extract the S3 bucket name, file names, source/destination paths, and determines the intended task type.

- **Nodes Involved:**  
  - Extract Data from Email

- **Node Details:**  

  - **Extract Data from Email**  
    - Type: Code (JavaScript)  
    - Role: Uses regex patterns on the email’s plain text to identify bucket names, file names, source and destination files, and the S3 operation to perform.  
    - Expressions: Regex patterns detect keywords and extract parameters; task type is set based on keywords like "create bucket", "delete bucket", "copy file", etc.  
    - Inputs: Email JSON from IMAP node.  
    - Outputs: JSON containing extracted parameters: bucket, fileName, sourceFile, destFile, task_type.  
    - Edge Cases: Emails without expected keywords or malformed text could yield null parameters or no task_type; could cause downstream routing issues.

#### 2.3 Task Routing

- **Overview:**  
  Routes the workflow execution to the specific AWS S3 operation node based on the extracted task_type.

- **Nodes Involved:**  
  - Check Task Type (Switch)

- **Node Details:**  

  - **Check Task Type**  
    - Type: Switch  
    - Role: Matches task_type string to one of the supported operations: create_bucket, delete_bucket, copy_file, delete_file, download_file, upload_file, get_files.  
    - Inputs: Parsed JSON from Extract Data from Email.  
    - Outputs: Routes to one of seven AWS S3 operation nodes.  
    - Edge Cases: Unknown or empty task_type results in no route and no operation; could be extended with default handling.

#### 2.4 AWS S3 Operations

- **Overview:**  
  Executes the corresponding AWS S3 operation as requested by the email command.

- **Nodes Involved:**  
  - Create a bucket  
  - Delete a bucket  
  - Copy a file  
  - Delete a file  
  - Download a file  
  - Upload a file  
  - Get many files

- **Node Details:**  

  - **Create a bucket**  
    - Type: AWS S3  
    - Role: Creates a new bucket using the name extracted from the email.  
    - Configuration: Resource = bucket; name parameter is currently statically set to "new-project" but should be dynamic from input.  
    - Inputs: Routed from Check Task Type.  
    - Outputs: Operation result JSON.  
    - Edge Cases: Bucket name conflicts, invalid bucket names, permission errors.

  - **Delete a bucket**  
    - Type: AWS S3  
    - Role: Deletes an existing bucket by name.  
    - Configuration: Resource = bucket; operation = delete; bucket name statically set to "new-project" (should be dynamic).  
    - Inputs: Routed from Check Task Type.  
    - Outputs: Operation result JSON.  
    - Edge Cases: Bucket not empty, bucket not found, permission errors.

  - **Copy a file**  
    - Type: AWS S3  
    - Role: Copies a file from sourcePath to destinationPath within S3.  
    - Configuration: Operation = copy; sourcePath and destinationPath statically set but should be dynamic.  
    - Inputs: Routed from Check Task Type.  
    - Outputs: Operation result JSON.  
    - Edge Cases: Source file does not exist, permission denied, invalid paths.

  - **Delete a file**  
    - Type: AWS S3  
    - Role: Deletes a specified file from a bucket.  
    - Configuration: Operation = delete; fileKey and bucketName currently static but should be dynamic.  
    - Inputs: Routed from Check Task Type.  
    - Outputs: Operation result JSON.  
    - Edge Cases: File not found, permission denied.

  - **Download a file**  
    - Type: AWS S3  
    - Role: Downloads a file from a bucket.  
    - Configuration: fileKey and bucketName currently static; should be dynamic inputs.  
    - Inputs: Routed from Check Task Type.  
    - Outputs: File content or metadata.  
    - Edge Cases: File missing, access denied.

  - **Upload a file**  
    - Type: AWS S3  
    - Role: Uploads a file to a bucket.  
    - Configuration: operation = upload; bucketName, fileName currently static; should handle dynamic input and file content.  
    - Inputs: Routed from Check Task Type.  
    - Outputs: Operation result JSON.  
    - Edge Cases: File content missing, invalid bucket, permission issues.

  - **Get many files**  
    - Type: AWS S3  
    - Role: Lists all files in the specified bucket.  
    - Configuration: operation = getAll; bucketName dynamically taken from input.  
    - Inputs: Routed from Check Task Type.  
    - Outputs: List of files in bucket.  
    - Edge Cases: Bucket not found, permission denied.

  - All nodes use the same AWS credentials with appropriate permissions.

#### 2.5 Outcome Evaluation

- **Overview:**  
  Checks if the AWS operation returned a successful result by verifying the presence of a key in the response.

- **Nodes Involved:**  
  - Check - Success or Fail

- **Node Details:**  

  - **Check - Success or Fail**  
    - Type: If  
    - Role: Uses a condition to check if the JSON response contains a non-empty "Key" field indicating success.  
    - Inputs: Output from any AWS S3 operation node.  
    - Outputs: Routes to success or failure email notification nodes.  
    - Edge Cases: Operations that don’t return a "Key" field might cause false negatives; can be improved with more robust success criteria.

#### 2.6 Notification Dispatch

- **Overview:**  
  Sends emails to notify the requester about the success or failure of their requested AWS S3 operation.

- **Nodes Involved:**  
  - Send Success Email  
  - Send Failed Email

- **Node Details:**  

  - **Send Success Email**  
    - Type: Email Send (SMTP)  
    - Role: Sends a text email confirming successful operation.  
    - Configuration: Subject and body text indicate success; reply-to and to-address dynamically set from original email sender; from address is static.  
    - Inputs: From Check - Success or Fail (success branch).  
    - Credentials: SMTP credentials configured.  
    - Edge Cases: SMTP failures, invalid recipient emails.

  - **Send Failed Email**  
    - Type: Email Send (SMTP)  
    - Role: Sends a text email notifying failure of the operation.  
    - Configuration: Subject and body text indicate failure; reply-to and to-address dynamically set; from address is static.  
    - Inputs: From Check - Success or Fail (failure branch).  
    - Credentials: Same SMTP credentials.  
    - Edge Cases: SMTP errors, invalid email addresses.

#### 2.7 Documentation and Guidance

- **Overview:**  
  Provides in-workflow documentation for clarity to users and maintainers.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content: Describes the overall purpose of the workflow and supported operations.  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: Explains workflow steps and node roles in a summary format.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                      |
|-------------------------|---------------------|----------------------------------------|--------------------------------|-------------------------------|-----------------------------------------------------------------|
| Start Workflow (GET Request) | Email Read (IMAP)   | Reads triggering emails                 | None                           | Extract Data from Email        | See sticky notes for workflow purpose and operation summary.    |
| Extract Data from Email  | Code                | Parses email text for commands          | Start Workflow (GET Request)   | Check Task Type                | See sticky notes for workflow purpose and operation summary.    |
| Check Task Type          | Switch              | Routes flow by detected task type       | Extract Data from Email        | Create a bucket, Delete a bucket, Copy a file, Delete a file, Download a file, Upload a file, Get many files | See sticky notes for workflow purpose and operation summary.    |
| Create a bucket         | AWS S3               | Creates a new S3 bucket                  | Check Task Type                | Check - Success or Fail        | See sticky notes for workflow purpose and operation summary.    |
| Delete a bucket         | AWS S3               | Deletes an S3 bucket                     | Check Task Type                | Check - Success or Fail        | See sticky notes for workflow purpose and operation summary.    |
| Copy a file             | AWS S3               | Copies a file within S3                  | Check Task Type                | Check - Success or Fail        | See sticky notes for workflow purpose and operation summary.    |
| Delete a file           | AWS S3               | Deletes a file from S3                   | Check Task Type                | Check - Success or Fail        | See sticky notes for workflow purpose and operation summary.    |
| Download a file         | AWS S3               | Downloads a file from S3                 | Check Task Type                | Check - Success or Fail        | See sticky notes for workflow purpose and operation summary.    |
| Upload a file           | AWS S3               | Uploads a file to S3                     | Check Task Type                | Check - Success or Fail        | See sticky notes for workflow purpose and operation summary.    |
| Get many files          | AWS S3               | Lists files in a bucket                  | Check Task Type                | Check - Success or Fail        | See sticky notes for workflow purpose and operation summary.    |
| Check - Success or Fail | If                   | Determines success or failure            | Create a bucket, Delete a bucket, Copy a file, Delete a file, Download a file, Upload a file, Get many files | Send Success Email, Send Failed Email | See sticky notes for workflow purpose and operation summary.    |
| Send Success Email      | Email Send (SMTP)    | Sends success notification email        | Check - Success or Fail        | None                          | See sticky notes for workflow purpose and operation summary.    |
| Send Failed Email       | Email Send (SMTP)    | Sends failure notification email        | Check - Success or Fail        | None                          | See sticky notes for workflow purpose and operation summary.    |
| Sticky Note             | Sticky Note          | Workflow purpose description             | None                          | None                          | "## What is this workflow for?... Enables management of AWS S3 buckets and files by email commands." |
| Sticky Note1            | Sticky Note          | Workflow step summary                    | None                          | None                          | "## How It Works... Lists nodes and their functions in the workflow." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create IMAP Email Read Node**  
   - Name: "Start Workflow (GET Request)"  
   - Type: Email Read (IMAP)  
   - Configure to read unread emails with subject containing "s3" using IMAP credentials.  
   - Set options: `["UNSEEN", ["SUBJECT", "s3"]]`.

2. **Add Code Node to Parse Email Data**  
   - Name: "Extract Data from Email"  
   - Type: Code (JavaScript)  
   - Paste JavaScript code that extracts bucket, file names, sourceFile, destFile, and task_type from email body using regex.  
   - Connect input from the IMAP node.

3. **Add Switch Node to Check Task Type**  
   - Name: "Check Task Type"  
   - Type: Switch (Version 3.2)  
   - Add rules for each task_type value: create_bucket, delete_bucket, copy_file, delete_file, download_file, upload_file, get_files.  
   - Connect input from "Extract Data from Email".

4. **Add AWS S3 Nodes for Each Operation**

   - **Create a bucket**  
     - Type: AWS S3  
     - Resource: bucket  
     - Operation: default (create)  
     - Name: Set dynamically from incoming JSON `={{ $json.bucket }}` (update from static "new-project").  
     - Connect from "Check Task Type" route for "create_bucket".

   - **Delete a bucket**  
     - Type: AWS S3  
     - Resource: bucket  
     - Operation: delete  
     - Name: `={{ $json.bucket }}` (dynamic)  
     - Connect from "Check Task Type" route for "delete_bucket".

   - **Copy a file**  
     - Type: AWS S3  
     - Operation: copy  
     - Source Path: `={{ $json.sourceFile }}`  
     - Destination Path: `={{ $json.destFile }}`  
     - Connect from "Check Task Type" route for "copy_file".

   - **Delete a file**  
     - Type: AWS S3  
     - Operation: delete  
     - File Key: `={{ $json.fileName }}`  
     - Bucket Name: `={{ $json.bucket }}`  
     - Connect from "Check Task Type" route for "delete_file".

   - **Download a file**  
     - Type: AWS S3  
     - Operation: get (download)  
     - File Key: `={{ $json.fileName }}`  
     - Bucket Name: `={{ $json.bucket }}`  
     - Connect from "Check Task Type" route for "download_file".

   - **Upload a file**  
     - Type: AWS S3  
     - Operation: upload  
     - Bucket Name: `={{ $json.bucket }}`  
     - File Name: `={{ $json.fileName }}`  
     - Provide file content dynamically as input (may require additional setup not shown here).  
     - Connect from "Check Task Type" route for "upload_file".

   - **Get many files**  
     - Type: AWS S3  
     - Operation: getAll  
     - Bucket Name: `={{ $json.bucket }}`  
     - Connect from "Check Task Type" route for "get_files".

5. **Add If Node to Check Operation Success**  
   - Name: "Check - Success or Fail"  
   - Type: If  
   - Condition: Check if response contains a non-empty key, typically `={{ $json.Key !== "" }}`.  
   - Connect inputs from all AWS S3 operation nodes.  
   - Set two outputs: True for success, False for failure.

6. **Add Email Send Nodes for Notifications**

   - **Send Success Email**  
     - Type: Email Send (SMTP)  
     - Configure with SMTP credentials.  
     - Set "To" and "Reply-To" dynamically to original email sender: `={{ $('Start Workflow (GET Request)').item.json.from }}`  
     - Subject: "AWS S3 Operation Successful"  
     - Text: "AWS S3 Operation Successful"  
     - Connect to "Check - Success or Fail" True output.

   - **Send Failed Email**  
     - Type: Email Send (SMTP)  
     - Configure with same SMTP credentials.  
     - Set "To" and "Reply-To" same as above.  
     - Subject: "AWS S3 Operation Failed"  
     - Text: "AWS S3 Operation Failed"  
     - Connect to "Check - Success or Fail" False output.

7. **Add Sticky Notes (Optional but Recommended)**  
   - Add two sticky notes summarizing the workflow purpose and step explanation for maintainers.

8. **Credential Setup**  
   - Configure IMAP credentials for email reading.  
   - Configure AWS credentials with full S3 permissions for all bucket and file operations.  
   - Configure SMTP credentials for sending notification emails.

9. **Testing and Validation**  
   - Test by sending emails with commands in the subject or body such as "create bucket", "upload file", specifying bucket and file names.  
   - Verify operations are executed and responses emailed back.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                       |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| This workflow enables the management of AWS S3 buckets and files by processing email commands. It supports a comprehensive set of S3 operations with email notification. | Sticky Note content in workflow                       |
| Workflow logic is designed with scalability to add more operations or handle additional edge cases by extending the switch and code node. | Workflow design principle                             |
| AWS S3 bucket and file names should be passed dynamically for full flexibility; static values seen in example nodes require adjustment. | Best practice for production use                      |
| SMTP and IMAP credentials must be valid and have required permissions to send and receive emails reliably.                        | Credential management                                 |
| Regex parsing in code node depends on consistent email formatting; consider updating regex patterns for robustness.               | Email content parsing note                            |
| Ensure AWS IAM roles used have adequate permissions for all S3 operations to avoid authorization errors.                          | AWS permissions best practice                         |
| For advanced file uploads, additional nodes or manual file content handling may be necessary beyond simple file name passing.    | Workflow extension guidance                           |
| n8n documentation for AWS S3 node configuration: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.awss3/         | Reference link                                        |
| n8n documentation for email nodes IMAP and SMTP: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.emailReadImap/ and https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.emailSend/ | Reference link                                        |

---

_Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or offensive elements. All data processed are legal and public._