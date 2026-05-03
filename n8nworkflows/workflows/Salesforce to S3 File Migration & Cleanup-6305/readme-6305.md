Salesforce to S3 File Migration & Cleanup

https://n8nworkflows.xyz/workflows/salesforce-to-s3-file-migration---cleanup-6305


# Salesforce to S3 File Migration & Cleanup

### 1. Workflow Overview

This workflow automates the migration and cleanup of Salesforce files by transferring old ContentDocuments (files) from Salesforce to an AWS S3 bucket, then deleting the original files in Salesforce to free up space and maintain data hygiene. It is designed for Salesforce administrators or integration engineers who want to archive Salesforce file data efficiently while maintaining traceability through custom Salesforce records.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & ContentDocument Retrieval:** Periodically triggers the workflow to fetch all Salesforce ContentDocuments older than 365 days.
- **1.2 Batch Processing Loop:** Processes each ContentDocument record in batches to manage load.
- **1.3 File Download & S3 Upload:** Downloads the file content from Salesforce and uploads it to a configured AWS S3 bucket.
- **1.4 Metadata Linking & Filtering:** Retrieves the linked entity of each file, filters out user attachments to avoid migrating irrelevant files, and creates a custom Salesforce record to track the S3 file.
- **1.5 Cleanup & Notification:** Prepares file identifiers for deletion, deletes the original Salesforce file, and sends Slack notifications about the operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & ContentDocument Retrieval

- **Overview:**  
  Initiates the workflow on a schedule and retrieves all Salesforce files older than one year (365 days) for migration.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get All ContentDocuments older than 365 days ago

- **Node Details:**

  - **Schedule Trigger**  
    - Type: scheduleTrigger  
    - Role: Starts the workflow on a defined schedule (default schedule not explicitly configured)  
    - Key Config: Default scheduling parameters (could be cron or interval)  
    - Input: None (trigger node)  
    - Output: Triggers the Salesforce query node  
    - Edge Cases: Workflow will not start if scheduling is misconfigured or disabled.

  - **Get All ContentDocuments older than 365 days ago**  
    - Type: salesforce  
    - Role: Queries Salesforce for ContentDocument records where the creation date is over 365 days ago  
    - Key Config: SOQL query or filter on ContentDocument with date condition (exact query not shown)  
    - Input: Trigger from Schedule Trigger  
    - Output: Sends results to batch processor  
    - Edge Cases: Query may fail due to Salesforce API limits or auth errors; empty result sets should be handled gracefully.

#### 2.2 Batch Processing Loop

- **Overview:**  
  Processes the retrieved ContentDocument records in batches to avoid overloading downstream nodes and API limits.

- **Nodes Involved:**  
  - Loop Over Each File Record

- **Node Details:**

  - **Loop Over Each File Record**  
    - Type: splitInBatches  
    - Role: Splits the incoming list of files into manageable batches for sequential processing  
    - Key Config: Batch size not specified (default or configured in node)  
    - Input: Output of ContentDocument query node  
    - Output: Two connections - one to Slack Notification, one to Download File Content  
    - Error Handling: Continues regular output on errors (does not stop workflow)  
    - Edge Cases: Batch size too large may cause timeouts; small batch sizes slow throughput.

#### 2.3 File Download & S3 Upload

- **Overview:**  
  Downloads each file's binary content from Salesforce and uploads it to AWS S3.

- **Nodes Involved:**  
  - Download File Content  
  - Upload to S3

- **Node Details:**

  - **Download File Content**  
    - Type: httpRequest  
    - Role: Fetches the actual file content from Salesforce via HTTP API  
    - Key Config: URL and headers configured to access ContentVersion or ContentDocument binary data  
    - Input: Batch processor node  
    - Output: Sends binary data to AWS S3 node  
    - Edge Cases: Auth token expiry, file not found, network errors, large file timeout.

  - **Upload to S3**  
    - Type: awsS3  
    - Role: Uploads the downloaded file content into a predefined AWS S3 bucket  
    - Key Config: Bucket name, key (file path), credentials set in node  
    - Input: Binary data from Download File Content  
    - Output: Passes metadata to next node for Salesforce linking  
    - Edge Cases: AWS permission errors, network failures, file size limits.

#### 2.4 Metadata Linking & Filtering

- **Overview:**  
  Retrieves the linked Salesforce entity for each file, filters out user attachments (which should not be migrated), and creates a custom Salesforce record to track the migrated file in S3.

- **Nodes Involved:**  
  - Get LinkedEntityId from ContentDocumentLink  
  - Filter Out User Attachments  
  - Create S3_File__c Record

- **Node Details:**

  - **Get LinkedEntityId from ContentDocumentLink**  
    - Type: salesforce  
    - Role: Queries ContentDocumentLink object to find the Salesforce record linked to the document  
    - Input: Output of AWS S3 upload  
    - Output: Feeds filtered code node  
    - Edge Cases: Missing links, query errors, API rate limits.

  - **Filter Out User Attachments**  
    - Type: code  
    - Role: Custom JavaScript to exclude files linked to User records (user attachments) from migration  
    - Input: LinkedEntityId query  
    - Output: Passes only relevant files to Salesforce record creation  
    - Key Expressions: Condition checking LinkedEntityId type or ID patterns  
    - Edge Cases: Logic errors causing false positives/negatives, empty inputs.

  - **Create S3_File__c Record**  
    - Type: salesforce  
    - Role: Creates a custom SObject record (S3_File__c) to log the migrated file's metadata and S3 location  
    - Input: Filtered list of files  
    - Output: Passes info for deletion preparation  
    - Edge Cases: Write permissions, validation errors, API throttling.

#### 2.5 Cleanup & Notification

- **Overview:**  
  Prepares the ContentDocumentId for deletion, deletes the original file from Salesforce, and sends a Slack notification about the migration status.

- **Nodes Involved:**  
  - Prepare ContentDocumentId for Deletion  
  - Delete Original File in Salesforce  
  - Send Slack Notification

- **Node Details:**

  - **Prepare ContentDocumentId for Deletion**  
    - Type: code  
    - Role: Formats or extracts ContentDocumentId for the deletion HTTP request  
    - Input: Salesforce custom record creation node  
    - Output: Provides input to HTTP request node for deletion  
    - Edge Cases: Incorrect ID formatting, missing fields.

  - **Delete Original File in Salesforce**  
    - Type: httpRequest  
    - Role: Sends HTTP DELETE request to Salesforce API to remove the original ContentDocument  
    - Key Config: HTTP method DELETE, URL constructed with ContentDocumentId, auth credentials  
    - Input: Prepared deletion ID  
    - Output: Loops back to process next batch item  
    - Edge Cases: API auth errors, file in use or locked, network timeouts.

  - **Send Slack Notification**  
    - Type: slack  
    - Role: Posts a message to Slack channel/webhook about the migration start or completion per batch  
    - Key Config: Webhook ID configured, message content possibly templated  
    - Input: Batch processor initial output  
    - Edge Cases: Slack webhook disabled, message formatting errors.

---

### 3. Summary Table

| Node Name                                | Node Type         | Functional Role                                | Input Node(s)                             | Output Node(s)                        | Sticky Note |
|-----------------------------------------|-------------------|-----------------------------------------------|------------------------------------------|-------------------------------------|-------------|
| Schedule Trigger                        | scheduleTrigger   | Initiates workflow on schedule                 | None                                     | Get All ContentDocuments older than 365 days ago |             |
| Get All ContentDocuments older than 365 days ago | salesforce        | Retrieves files older than 365 days from Salesforce | Schedule Trigger                         | Loop Over Each File Record           |             |
| Loop Over Each File Record              | splitInBatches    | Processes files in batches                      | Get All ContentDocuments older than 365 days ago | Send Slack Notification, Download File Content |             |
| Send Slack Notification                 | slack             | Sends Slack messages for batch processing status | Loop Over Each File Record               | None                                |             |
| Download File Content                   | httpRequest       | Downloads file binary from Salesforce          | Loop Over Each File Record                | Upload to S3                        |             |
| Upload to S3                           | awsS3             | Uploads file to AWS S3 bucket                   | Download File Content                     | Get LinkedEntityId from ContentDocumentLink |             |
| Get LinkedEntityId from ContentDocumentLink | salesforce        | Retrieves linked Salesforce entity for file    | Upload to S3                             | Filter Out User Attachments          |             |
| Filter Out User Attachments             | code              | Filters out user attachments from processing   | Get LinkedEntityId from ContentDocumentLink | Create S3_File__c Record             |             |
| Create S3_File__c Record                | salesforce        | Creates custom record for migrated file metadata | Filter Out User Attachments              | Prepare ContentDocumentId for Deletion |             |
| Prepare ContentDocumentId for Deletion  | code              | Prepares file ID for deletion                   | Create S3_File__c Record                  | Delete Original File in Salesforce   |             |
| Delete Original File in Salesforce      | httpRequest       | Deletes original file from Salesforce           | Prepare ContentDocumentId for Deletion   | Loop Over Each File Record            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: scheduleTrigger  
   - Configure: Set desired schedule (e.g., daily at midnight) to start workflow.

2. **Add Salesforce Node - Get All ContentDocuments older than 365 days ago**  
   - Type: salesforce  
   - Credentials: Set Salesforce OAuth2 credentials.  
   - Configuration: Use SOQL query to select ContentDocuments with CreatedDate older than 365 days (e.g., `SELECT Id, Title FROM ContentDocument WHERE CreatedDate <= LAST_N_DAYS:365`).  
   - Connect scheduleTrigger output to this node.

3. **Add Split In Batches Node - Loop Over Each File Record**  
   - Type: splitInBatches  
   - Configuration: Set batch size (e.g., 5-10) to manage API limits.  
   - Connect output of Salesforce query node to this node.

4. **Add Slack Node - Send Slack Notification**  
   - Type: slack  
   - Credentials: Configure Slack webhook credentials.  
   - Parameters: Set message template for batch start or completion.  
   - Connect first output of splitInBatches node to this node.

5. **Add HTTP Request Node - Download File Content**  
   - Type: httpRequest  
   - Credentials: Use Salesforce OAuth2 or session token.  
   - Configuration: URL should point to ContentVersion file download endpoint (e.g., use ContentDocumentId to get latest ContentVersion and download).  
   - Connect second output of splitInBatches node to this node.

6. **Add AWS S3 Node - Upload to S3**  
   - Type: awsS3  
   - Credentials: Configure AWS credentials with write permission to the target bucket.  
   - Parameters: Bucket name, file key (path), and use the binary data from HTTP Request node.  
   - Connect Download File Content output to this node.

7. **Add Salesforce Node - Get LinkedEntityId from ContentDocumentLink**  
   - Type: salesforce  
   - Credentials: Use same Salesforce credentials.  
   - Configuration: Query ContentDocumentLink where ContentDocumentId equals current file's Id to get LinkedEntityId.  
   - Connect Upload to S3 output to this node.

8. **Add Code Node - Filter Out User Attachments**  
   - Type: code  
   - Code: JavaScript to filter out records where LinkedEntityId corresponds to User object (e.g., check if LinkedEntityId starts with user prefix '005').  
   - Connect output of LinkedEntityId node to this node.

9. **Add Salesforce Node - Create S3_File__c Record**  
   - Type: salesforce  
   - Credentials: Use Salesforce OAuth2.  
   - Configuration: Create record in custom SObject S3_File__c with fields mapping e.g., original ContentDocumentId, S3 file location, linked entity info.  
   - Connect output of code filter node to this node.

10. **Add Code Node - Prepare ContentDocumentId for Deletion**  
    - Type: code  
    - Code: Extract or format ContentDocumentId for deletion API call.  
    - Connect output of create record node to this node.

11. **Add HTTP Request Node - Delete Original File in Salesforce**  
    - Type: httpRequest  
    - Credentials: Salesforce OAuth2.  
    - Configuration: HTTP Method DELETE, URL targeting ContentDocument with ContentDocumentId from previous node.  
    - Connect output of prepare deletion node to this node.

12. **Loop to Batch Processor**  
    - Connect output of Delete Original File node back to Loop Over Each File Record node to continue processing next batch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow efficiently handles large file sets by processing in batches and includes error handling to continue on batch errors.                      | Best practices for batch processing in n8n      |
| Ensure Salesforce API permissions include access to ContentDocument, ContentDocumentLink, and custom object S3_File__c.                                  | Salesforce permission setup                       |
| AWS S3 credentials must have PutObject permission for the target bucket.                                                                                   | AWS IAM policy reference                          |
| Slack webhook integration requires prior configuration of incoming webhook URL in Slack workspace.                                                       | Slack Incoming Webhooks documentation             |
| The workflow assumes files older than 365 days are safe to archive and delete; adjust date filtering according to organizational policies.                 | Data retention policy considerations              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.