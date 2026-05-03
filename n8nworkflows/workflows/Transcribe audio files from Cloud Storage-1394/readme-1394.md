Transcribe audio files from Cloud Storage

https://n8nworkflows.xyz/workflows/transcribe-audio-files-from-cloud-storage-1394


# Transcribe audio files from Cloud Storage

### 1. Workflow Overview

This workflow automates the transcription of audio files uploaded to Google Drive by uploading them to AWS S3, initiating an AWS Transcribe job, waiting for its completion, and finally storing the transcription results in Google Sheets. It is designed for use cases where audio data needs to be automatically transcribed and logged without manual intervention.

**Logical Blocks:**

- **1.1 Input Reception:** Detect new audio file uploads in a specific Google Drive folder.
- **1.2 Cloud Storage Upload:** Upload the received file to an AWS S3 bucket and retrieve it for transcription.
- **1.3 Transcription Job Management:** Create and monitor the AWS Transcribe job for the uploaded audio file.
- **1.4 Data Formatting and Storage:** Format the transcription output and store it in a Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow upon the creation of a new file in a designated Google Drive folder.

**Nodes Involved:**  
- Google Drive Trigger1

**Node Details:**

- **Google Drive Trigger1**  
  - **Type & Role:** Trigger node. Listens for new file creation events in Google Drive.  
  - **Configuration:**  
    - Event: `fileCreated`  
    - Trigger On: `specificFolder`  
    - Folder to Watch: URL of the Google Drive folder (replace `[your_id]` with actual folder ID).  
  - **Expressions / Variables:** None beyond the folder URL.  
  - **Input/Output:** No input; outputs file metadata JSON.  
  - **Version Requirements:** n8n version supporting Google Drive Trigger node (default).  
  - **Edge Cases:**  
    - Permission errors if OAuth2 credentials lack scope for the folder.  
    - Delay in event triggering due to Google Drive API latency.  
    - Non-audio files triggering workflow if not filtered downstream.  
  - **Sub-workflow:** None.

#### 1.2 Cloud Storage Upload

**Overview:**  
This block uploads the newly detected file from Google Drive to an AWS S3 bucket, then fetches the file metadata from S3.

**Nodes Involved:**  
- AWS S3 1  
- AWS S3 2

**Node Details:**

- **AWS S3 1**  
  - **Type & Role:** AWS S3 node for uploading files.  
  - **Configuration:**  
    - Operation: `upload`  
    - Bucket Name: `mybucket`  
    - File Name: Uses the incoming file's `name` property from Google Drive trigger.  
    - File Content: Static value `"street"` (note: this seems incorrect; normally should use binary or downloaded content).  
    - Tags: Sets tag `source: gdrive`.  
  - **Expressions / Variables:**  
    - `{{$json["name"]}}` for filename.  
  - **Input/Output:** Input comes from Google Drive Trigger; output is upload confirmation JSON.  
  - **Edge Cases:**  
    - Upload of incorrect content due to static `"street"` value instead of actual file binary.  
    - AWS credential errors or bucket permission issues.  
    - File size limits or timeout.  
  - **Sub-workflow:** None.

- **AWS S3 2**  
  - **Type & Role:** AWS S3 node for retrieving file metadata or files.  
  - **Configuration:**  
    - Operation: `getAll`  
    - Bucket Name: `mybucket`  
  - **Input/Output:** Input from AWS S3 1 output; outputs list of files in the bucket.  
  - **Edge Cases:**  
    - Large bucket leading to performance issues.  
    - AWS credential or permission errors.  
  - **Sub-workflow:** None.

#### 1.3 Transcription Job Management

**Overview:**  
This block starts a transcription job on AWS Transcribe for the uploaded file, waits for job completion, and retrieves transcription results.

**Nodes Involved:**  
- AWS Transcribe 1  
- Wait  
- AWS Transcribe 2

**Node Details:**

- **AWS Transcribe 1**  
  - **Type & Role:** AWS Transcribe node to create a transcription job.  
  - **Configuration:**  
    - Operation: `create` (default for this node)  
    - Media File URI: Constructed as `s3://[bucketName]/[Key]` from previous node outputs, specifically:  
      - Bucket Name from `AWS S3 2` node parameter `bucketName`  
      - Key from current JSON input `Key` property  
    - Transcription Job Name: Set to the audio file's `Key` value (filename in S3).  
  - **Expressions / Variables:**  
    - `=s3://{{$node["AWS S3 2"].parameter["bucketName"]}}/{{$json["Key"]}}`  
    - `={{$json["Key"]}}`  
  - **Input/Output:**  
    - Input: File metadata from AWS S3 2  
    - Output: Job creation confirmation JSON  
  - **Edge Cases:**  
    - Job name conflicts if same file uploaded multiple times.  
    - AWS Transcribe service limitations (max job concurrency, supported formats).  
    - Incorrect media URI formatting.  
  - **Sub-workflow:** None.

- **Wait**  
  - **Type & Role:** Wait node to delay workflow until transcription job completes.  
  - **Configuration:**  
    - Resume via webhook (asynchronously waits for an external trigger to resume).  
    - Response mode: `lastNode` (returns data from last node)  
    - Response property name: `transcript`  
  - **Input/Output:**  
    - Input: From AWS Transcribe 1 output  
    - Output: Passes the transcription job status to AWS Transcribe 2  
  - **Edge Cases:**  
    - If transcription job takes too long or fails, workflow may stall.  
    - Webhook resume must be triggered externally or via manual intervention.  
  - **Sub-workflow:** None.

- **AWS Transcribe 2**  
  - **Type & Role:** AWS Transcribe node to get transcription job details and results.  
  - **Configuration:**  
    - Operation: `get`  
    - Transcription Job Name: Uses `Key` property from input JSON, same as job name.  
  - **Expressions / Variables:**  
    - `={{$json["Key"]}}`  
  - **Input/Output:**  
    - Input: From Wait node output  
    - Output: Transcription job details including transcript text  
  - **Edge Cases:**  
    - Job not found or incomplete leading to errors.  
    - Transcription result missing or delayed.  
  - **Sub-workflow:** None.

#### 1.4 Data Formatting and Storage

**Overview:**  
This final block formats the transcription job output into a structured set of values and appends the data to a Google Sheets spreadsheet.

**Nodes Involved:**  
- Set  
- Google Sheets

**Node Details:**

- **Set**  
  - **Type & Role:** Prepares and formats data for storage.  
  - **Configuration:**  
    - Sets three string fields:  
      - `recording_name`: transcription job name (from AWS Transcribe 1 JSON)  
      - `recording_link`: Google Drive file webContentLink (from Google Drive Trigger)  
      - `transcription`: transcript text (from current JSON, expected from AWS Transcribe 2)  
    - Sets one number field:  
      - `transcription_date`: creation time of transcription job (from AWS Transcribe 1 JSON)  
  - **Expressions / Variables:**  
    - `={{$node["AWS Transcribe 1"].json["TranscriptionJobName"]}}`  
    - `={{$node["Google Drive Trigger"].json["webContentLink"]}}`  
    - `={{$json["transcript"]}}`  
    - `={{$node["AWS Transcribe 1"].json["CreationTime"]}}`  
  - **Input/Output:**  
    - Input: Transcription job details from AWS Transcribe 2  
    - Output: Structured JSON for Google Sheets insertion  
  - **Edge Cases:**  
    - Missing or malformed transcription data.  
    - Null or undefined webContentLink if file is inaccessible.  
  - **Sub-workflow:** None.

- **Google Sheets**  
  - **Type & Role:** Appends data to Google Sheets.  
  - **Configuration:**  
    - Operation: `append`  
    - Range: Columns A to D (`A:D`)  
    - Sheet ID: `qwertz` (placeholder ID)  
    - Authentication: OAuth2 with configured Google Sheets credentials  
  - **Input/Output:**  
    - Input: Structured data from Set node  
    - Output: Confirmation of append operation  
  - **Edge Cases:**  
    - Permission errors if OAuth2 token lacks write access.  
    - Sheet ID incorrect or sheet missing.  
    - Rate limits from Google Sheets API.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                     | Input Node(s)        | Output Node(s)       | Sticky Note                                      |
|---------------------|-------------------------|-----------------------------------|----------------------|----------------------|-------------------------------------------------|
| Google Drive Trigger1| googleDriveTrigger      | Trigger on new file upload        | None                 | AWS S3 1              |                                                 |
| AWS S3 1             | awsS3                   | Upload file to AWS S3 bucket      | Google Drive Trigger1 | AWS S3 2              |                                                 |
| AWS S3 2             | awsS3                   | Retrieve file metadata from S3    | AWS S3 1              | AWS Transcribe 1      |                                                 |
| AWS Transcribe 1     | awsTranscribe           | Start transcription job           | AWS S3 2              | Wait                  |                                                 |
| Wait                 | wait                    | Wait for transcription completion | AWS Transcribe 1      | AWS Transcribe 2      |                                                 |
| AWS Transcribe 2     | awsTranscribe           | Get transcription job results     | Wait                  | Set                   |                                                 |
| Set                  | set                     | Format transcription data         | AWS Transcribe 2      | Google Sheets         |                                                 |
| Google Sheets        | googleSheets            | Append transcription data to sheet| Set                   | None                  |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**  
   - Node Type: `Google Drive Trigger`  
   - Set event to `fileCreated`.  
   - Trigger on: `specificFolder`.  
   - Folder to Watch: Provide Google Drive folder URL (replace `[your_id]` with actual folder ID).  
   - Configure OAuth2 credentials with Google Drive API access.  
   - Position node as the workflow entry point.

2. **Create AWS S3 1 node:**  
   - Node Type: `AWS S3`  
   - Operation: `upload`.  
   - Bucket Name: `mybucket` (replace with your bucket).  
   - File Name: Use expression to capture filename from trigger: `{{$json["name"]}}`.  
   - File Content: **Note:** The current workflow uses static `"street"` which is likely an error. Instead, configure to use binary data from Google Drive (download the file content first or link via binary data).  
   - Add tag: key `source`, value `gdrive`.  
   - Connect input from Google Drive Trigger node.  
   - Configure AWS credentials with appropriate access rights.

3. **Create AWS S3 2 node:**  
   - Node Type: `AWS S3`  
   - Operation: `getAll`.  
   - Bucket Name: same as above (`mybucket`).  
   - Connect input from AWS S3 1 node.  
   - Use same AWS credentials.

4. **Create AWS Transcribe 1 node:**  
   - Node Type: `AWS Transcribe`  
   - Operation: `create` (default).  
   - Transcription Job Name: use expression `{{$json["Key"]}}` from AWS S3 2 node.  
   - Media File URI: construct expression: `s3://{{$node["AWS S3 2"].parameter["bucketName"]}}/{{$json["Key"]}}`.  
   - Connect input from AWS S3 2 node.  
   - Configure AWS credentials with permissions for Transcribe.

5. **Create Wait node:**  
   - Node Type: `Wait`  
   - Mode: `Webhook`  
   - Resume on webhook call (set `resume` to `webhook`).  
   - Response mode: `lastNode`.  
   - Response property name: `transcript`.  
   - Connect input from AWS Transcribe 1 node.  
   - This node waits for external signal that transcription job is complete.

6. **Create AWS Transcribe 2 node:**  
   - Node Type: `AWS Transcribe`  
   - Operation: `get`.  
   - Transcription Job Name: use expression `{{$json["Key"]}}`.  
   - Connect input from Wait node.  
   - Use same AWS credentials.

7. **Create Set node:**  
   - Node Type: `Set`  
   - Add fields:  
     - String `recording_name`: `{{$node["AWS Transcribe 1"].json["TranscriptionJobName"]}}`  
     - String `recording_link`: `{{$node["Google Drive Trigger"].json["webContentLink"]}}`  
     - String `transcription`: `{{$json["transcript"]}}`  
     - Number `transcription_date`: `{{$node["AWS Transcribe 1"].json["CreationTime"]}}`  
   - Connect input from AWS Transcribe 2 node.

8. **Create Google Sheets node:**  
   - Node Type: `Google Sheets`  
   - Operation: `append`.  
   - Range: `A:D`.  
   - Sheet ID: Use your target sheet ID (replace `qwertz`).  
   - Authentication: OAuth2 with Google Sheets API credentials.  
   - Connect input from Set node.

9. **Connect all nodes as per above sequence:**  
   Google Drive Trigger1 → AWS S3 1 → AWS S3 2 → AWS Transcribe 1 → Wait → AWS Transcribe 2 → Set → Google Sheets

10. **Credentials:**  
    - Set up Google Drive OAuth2 credentials with read access to the specific folder.  
    - Set up AWS credentials with S3 upload/download and Transcribe permissions.  
    - Set up Google Sheets OAuth2 credentials with write access to the target spreadsheet.

11. **Additional Considerations:**  
    - Confirm or fix the file content upload step to correctly handle audio file content instead of static string.  
    - Set proper permissions and API quotas to avoid throttling.  
    - Implement error handling or notifications for failed transcription jobs or upload errors.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                               |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow screenshot referenced as `![workflow-screenshot](fileId:592)` is not included here.| Original workflow visual aid (not embedded in this document). |
| The static value `"street"` for file content in AWS S3 upload node is likely a misconfiguration.| Audio file binary content should be used instead.             |
| AWS Transcribe limits: supported audio formats and max job concurrency should be checked.        | AWS Transcribe official docs: https://docs.aws.amazon.com/transcribe/latest/dg/what-is-transcribe.html |
| Google Sheets API rate limits may affect high-frequency appending.                              | https://developers.google.com/sheets/api/limits                |
| Use of Wait node with webhook requires external trigger or manual resume step to proceed.       | See n8n Wait node documentation for webhook mode usage.       |

---

This completes the structured reference documentation for the workflow titled "Transcribe audio files from Cloud Storage."