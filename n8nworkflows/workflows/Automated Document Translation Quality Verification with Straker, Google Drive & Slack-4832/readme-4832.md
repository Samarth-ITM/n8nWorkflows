Automated Document Translation Quality Verification with Straker, Google Drive & Slack

https://n8nworkflows.xyz/workflows/automated-document-translation-quality-verification-with-straker--google-drive---slack-4832


# Automated Document Translation Quality Verification with Straker, Google Drive & Slack

### 1. Workflow Overview

This workflow automates the translation quality verification process by integrating Straker Verify's translation management API with Google Drive and Slack. It is designed for organizations that manage translation projects and want to streamline:

- Uploading source documents to Google Drive
- Submitting files for translation via Straker Verify
- Waiting for translation completion notifications
- Downloading translated files automatically
- Storing translated files back to Google Drive
- Sending Slack notifications on job status

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Preparation**: Watches a Google Drive folder for new files, downloads them, and initiates translation requests.
- **1.2 Translation Submission**: Sends the downloaded files to Straker Verify to start the translation process.
- **1.3 Translation Completion Handling**: Waits for a webhook callback signaling translation completion, retrieves job info, and downloads translated files.
- **1.4 Output Processing and Notification**: Saves translated files back to Google Drive and sends Slack notifications summarizing job status.
- **1.5 Utility and Data Transformation**: Flattens nested translation job data and aggregates results for notification.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Preparation

**Overview:**  
This block monitors a specific Google Drive folder for newly created files, downloads these files, and prepares them for submission to the translation service.

**Nodes Involved:**  
- Watch Input Folder  
- Download Files

**Node Details:**

- **Watch Input Folder**  
  - *Type:* Google Drive Trigger  
  - *Role:* Triggers workflow when a new file is created in a specified Google Drive folder.  
  - *Configuration:* Watches folder with ID `1jS3LMLVuHbsKFmtTRiRmUPUBO-CKqsfb`, polling every minute.  
  - *Input/Output:* No input; outputs file metadata including file ID.  
  - *Edge Cases:* Missing or revoked Google Drive credentials, folder permission issues, delayed polling.  
  - *Sticky Note:* Part of the overall input detection.

- **Download Files**  
  - *Type:* Google Drive Node  
  - *Role:* Downloads the actual file content based on file ID received from trigger.  
  - *Configuration:* Uses Google Drive OAuth2 credentials; downloads file by ID passed via expression `={{ $json.id }}`.  
  - *Input/Output:* Receives file metadata; outputs binary file data.  
  - *Edge Cases:* File may be deleted or moved before download; permission errors; network timeouts.

#### 2.2 Translation Submission

**Overview:**  
This block sends the downloaded files to Straker Verify to start translation projects.

**Nodes Involved:**  
- Start Translation

**Node Details:**

- **Start Translation**  
  - *Type:* Straker Verify API Node  
  - *Role:* Creates a new translation project/job with Straker Verify using file data.  
  - *Configuration:* Uses the "create" operation with a fixed workflow ID `78ed818e-3655-4bd2-b347-dcce14e58c1c` and a static title `"TEST 12"`. No target languages specified explicitly here (empty array).  
  - *Input/Output:* Takes file data from the previous node; outputs job creation response including job UUID.  
  - *Credentials:* Requires Straker Verify API key credential.  
  - *Edge Cases:* API authentication errors, invalid workflow ID, missing file data, network errors.

#### 2.3 Translation Completion Handling

**Overview:**  
This block waits for Straker Verify to signal that a translation job is completed, retrieves detailed job information, flattens the job data for easier processing, and downloads the translated files.

**Nodes Involved:**  
- Wait for Completion  
- Fetch Job Info  
- Flatten  
- Grab Translation

**Node Details:**

- **Wait for Completion**  
  - *Type:* Webhook (HTTP POST)  
  - *Role:* Listens for webhook callbacks from Straker Verify indicating job completion.  
  - *Configuration:* Webhook path `6540f764-161d-4627-b7c2-a78a992daf1d`, HTTP method POST.  
  - *Input/Output:* No input; outputs webhook payload containing job UUID.  
  - *Edge Cases:* Missed or delayed webhook calls, webhook URL exposure, security (no authentication layer visible).  

- **Fetch Job Info**  
  - *Type:* Straker Verify API Node  
  - *Role:* Retrieves detailed information about the completed translation job using the job UUID.  
  - *Configuration:* Uses the "get" operation with dynamic projectId expression `={{ $json.body.job_uuid }}` from webhook data.  
  - *Credentials:* Straker Verify API key required.  
  - *Edge Cases:* Invalid job UUID, API failures, permissions errors.

- **Flatten**  
  - *Type:* Code (JavaScript) Node  
  - *Role:* Processes nested job data to emit one item per translated target file with normalized metadata.  
  - *Configuration:* Custom JavaScript to iterate over `source_files` and their `target_files`, building filenames including language shortnames, preserving UUIDs and status.  
  - *Input/Output:* Takes job info JSON; outputs flattened list of objects each representing a single translated file.  
  - *Edge Cases:* Missing expected properties (e.g., no `source_files` array), malformed JSON, missing language codes.

- **Grab Translation**  
  - *Type:* Straker Verify API Node  
  - *Role:* Downloads a specific translated file from Straker Verify using its target file UUID.  
  - *Configuration:* Uses `fileId` expression `={{ $json.target_file_uuid }}` dynamically per item.  
  - *Credentials:* Straker Verify API key required.  
  - *Input/Output:* Receives flattened file metadata; outputs binary file data.  
  - *Edge Cases:* File not found, API errors, authorization issues.

#### 2.4 Output Processing and Notification

**Overview:**  
This block saves each translated file to a specified Google Drive output folder, aggregates all translation results, and sends a Slack notification summarizing the job status.

**Nodes Involved:**  
- Save to Output Folder  
- Aggregate  
- Notify

**Node Details:**

- **Save to Output Folder**  
  - *Type:* Google Drive Node  
  - *Role:* Uploads each translated file to a designated Google Drive folder.  
  - *Configuration:* Folder ID `16Ke3E8wSbTVfuOUXrex-ueNLuyIRhK95` in “My Drive” is target; filename is dynamically constructed from Flatten node’s output `={{ $('Flatten').item.json.filename }}`.  
  - *Credentials:* Google Drive OAuth2 required.  
  - *Input/Output:* Takes binary file data from Grab Translation; outputs file metadata.  
  - *Edge Cases:* Folder permission issues, file overwrite conflicts, quota limits.

- **Aggregate**  
  - *Type:* Aggregate Node  
  - *Role:* Aggregates all saved file items into a single JSON object for notification.  
  - *Configuration:* Default aggregation of all items.  
  - *Input/Output:* Takes multiple file metadata items; outputs aggregated JSON.  
  - *Edge Cases:* Large data sets causing memory issues.

- **Notify**  
  - *Type:* Slack Node  
  - *Role:* Sends a Slack message to a specific channel with summary text about the translation job.  
  - *Configuration:* Posts message to channel ID `C08NJCEJS8Y` using a Slack webhook; message text dynamically constructed using job title and status from `Fetch Job Info` node JSON.  
  - *Credentials:* Slack webhook credentials needed.  
  - *Edge Cases:* Slack API rate limits, invalid channel ID, message formatting errors.

#### 2.5 Utility and Information

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Provides detailed documentation about the Straker Verify node’s use and the overall workflow logic.  
  - *Content Summary:*  
    - Explains Straker Verify node credentials and API key setup.  
    - Describes two main workflow patterns: submitting for translation and retrieving translation.  
    - Gives step-by-step usage guidance within the workflow.  
  - *Edge Cases:* None (informational only).

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                       | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                                                                                                                                                          |
|---------------------|--------------------------------|-------------------------------------|---------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Watch Input Folder   | Google Drive Trigger            | Detect new files in input folder    | —                         | Download Files           |                                                                                                                                                                                                                                    |
| Download Files      | Google Drive                   | Download newly detected files       | Watch Input Folder         | Start Translation        |                                                                                                                                                                                                                                    |
| Start Translation    | Straker Verify API Node         | Submit files to start translation   | Download Files             | —                       |                                                                                                                                                                                                                                    |
| Wait for Completion  | Webhook                       | Wait for translation completion     | —                         | Fetch Job Info           |                                                                                                                                                                                                                                    |
| Fetch Job Info       | Straker Verify API Node         | Retrieve job details post-completion| Wait for Completion        | Flatten                  |                                                                                                                                                                                                                                    |
| Flatten              | Code (JavaScript)               | Flatten nested job data per file    | Fetch Job Info             | Grab Translation         |                                                                                                                                                                                                                                    |
| Grab Translation     | Straker Verify API Node         | Download translated file             | Flatten                    | Save to Output Folder    |                                                                                                                                                                                                                                    |
| Save to Output Folder| Google Drive                   | Save translated files to Drive      | Grab Translation           | Aggregate                |                                                                                                                                                                                                                                    |
| Aggregate            | Aggregate                     | Aggregate saved file data           | Save to Output Folder      | Notify                   |                                                                                                                                                                                                                                    |
| Notify               | Slack                         | Send Slack notification             | Aggregate                  | —                       |                                                                                                                                                                                                                                    |
| Sticky Note          | Sticky Note                   | Documentation of Straker Verify node and workflow logic | —                         | —                       | # Straker Verify n8n Node\n\nThis node allows you to interact with the Straker Verify API to manage translation projects, API keys, user information, and files directly within your n8n workflows.\n\nMore info: https://api-verify.straker.ai/docs |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node ("Watch Input Folder")**  
   - Type: Google Drive Trigger  
   - Event: fileCreated  
   - Poll Interval: every minute  
   - Folder to watch: set by folder ID `1jS3LMLVuHbsKFmtTRiRmUPUBO-CKqsfb`  
   - No credentials needed here, but Google Drive OAuth2 credentials must be configured in n8n.

2. **Create Google Drive Node ("Download Files")**  
   - Operation: download  
   - File ID: Expression `={{ $json.id }}` from trigger node  
   - Credentials: Google Drive OAuth2  
   - Connect "Watch Input Folder" → "Download Files"

3. **Create Straker Verify Node ("Start Translation")**  
   - Operation: create (project)  
   - Title: "TEST 12" (static or adjust as needed)  
   - Workflow ID: `78ed818e-3655-4bd2-b347-dcce14e58c1c` (fixed)  
   - Languages: Empty array (could be customized)  
   - Credentials: Straker Verify API key  
   - Connect "Download Files" → "Start Translation"

4. **Create Webhook Node ("Wait for Completion")**  
   - HTTP Method: POST  
   - Path: `6540f764-161d-4627-b7c2-a78a992daf1d` (unique webhook URL segment)  
   - No credentials needed  
   - This node waits for external POST callbacks notifying job completion.

5. **Create Straker Verify Node ("Fetch Job Info")**  
   - Operation: get (project)  
   - Project ID: Expression `={{ $json.body.job_uuid }}` from webhook payload  
   - Credentials: Straker Verify API key  
   - Connect "Wait for Completion" → "Fetch Job Info"

6. **Create Code Node ("Flatten")**  
   - JavaScript code as per the workflow:  
     - Checks for `source_files` array in job data  
     - Maps each source file and its target files to output items with fields: URL, filename (with language shortname), UUIDs, status  
   - Connect "Fetch Job Info" → "Flatten"

7. **Create Straker Verify Node ("Grab Translation")**  
   - Operation: get (file)  
   - File ID: Expression `={{ $json.target_file_uuid }}` from Flatten output  
   - Credentials: Straker Verify API key  
   - Connect "Flatten" → "Grab Translation"

8. **Create Google Drive Node ("Save to Output Folder")**  
   - Operation: upload (or save)  
   - Folder ID: `16Ke3E8wSbTVfuOUXrex-ueNLuyIRhK95`  
   - Filename: Expression `={{ $('Flatten').item.json.filename }}`  
   - Credentials: Google Drive OAuth2  
   - Connect "Grab Translation" → "Save to Output Folder"

9. **Create Aggregate Node ("Aggregate")**  
   - Aggregate all items into one output object  
   - Connect "Save to Output Folder" → "Aggregate"

10. **Create Slack Node ("Notify")**  
    - Operation: send message via webhook  
    - Channel ID: `C08NJCEJS8Y`  
    - Text: Expression `={{ $('Fetch Job Info').first().json.data.title }} {{ $('Fetch Job Info').first().json.data.status }}`  
    - Credentials: Slack webhook or Slack OAuth2 credentials configured  
    - Connect "Aggregate" → "Notify"

11. **Add Sticky Note Node ("Sticky Note")**  
    - Content: detailed explanation of Straker Verify node usage and workflow logic  
    - No connections required; purely documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                        | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Straker Verify API documentation provides detailed info on API usage, authentication, and endpoint capabilities.                                                                                                                                  | https://api-verify.straker.ai/docs                                                                     |
| Workflow assumes Google Drive OAuth2 and Slack credentials are properly configured in n8n for file operations and notifications.                                                                                                                  | n8n credential setup                                                                                   |
| Webhook node uses a fixed path; ensure external Straker Verify callbacks are configured to send completion notifications to this URL.                                                                                                            | Webhook security considerations                                                                       |
| The Flatten code node transforms complex nested job data into individual file items, crucial for per-file processing downstream.                                                                                                                 | Custom JavaScript in Code node                                                                         |
| Slack notifications include job title and status dynamically pulled from job info, providing timely updates to team channels.                                                                                                                    | Slack message formatting                                                                                |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.