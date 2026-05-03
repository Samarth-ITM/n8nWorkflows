Generate Videos Automatically with Veo3 Fast API, Google Drive & Email Notifications

https://n8nworkflows.xyz/workflows/generate-videos-automatically-with-veo3-fast-api--google-drive---email-notifications-10860


# Generate Videos Automatically with Veo3 Fast API, Google Drive & Email Notifications

### 1. Workflow Overview

This workflow automates the generation of CGI advertisement videos using the Veo 3 Fast API, integrates with Google Drive for file storage, and notifies users and administrators via email. It is designed for users who submit a video prompt through a form, triggering the creation of a video which is then uploaded to Google Drive and shared via email.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures the user prompt from a form submission.
- **1.2 Video Generation Request:** Sends the prompt to the Veo 3 Fast API to initiate video generation.
- **1.3 Task ID Verification:** Validates that the API returned a task ID to track the video generation.
- **1.4 Task Status Polling:** Periodically checks the status of the video generation task until completion or failure.
- **1.5 Error Handling:** Sends email notifications if the task ID is missing or if the video generation fails.
- **1.6 Video Download and Storage:** Downloads the generated video and uploads it to Google Drive.
- **1.7 File Sharing and User Notification:** Sets sharing permissions on the Google Drive file and emails the user a link to the video.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for form submissions where users input their video prompt. This is the workflow’s entry point.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Triggers workflow on user submission of a form titled "CGI Ads With Google Veo3" with a required "Prompt" field.  
    - Configuration: Form has one required field labeled "Prompt."  
    - Inputs: External form submission via webhook  
    - Outputs: Passes form data (including prompt) downstream  
    - Edge Cases: Missing prompt field submission is prevented by required setting; webhook connectivity issues may cause trigger failure.

---

#### 1.2 Video Generation Request

- **Overview:**  
  Sends the user’s prompt to the Veo 3 Fast API to start video generation.

- **Nodes Involved:**  
  - Veo 3 Fast API Processor

- **Node Details:**

  - **Veo 3 Fast API Processor**  
    - Type: HTTP Request  
    - Role: Posts JSON body containing prompt and API version to Veo 3 Fast API endpoint to initiate a video generation task.  
    - Configuration:  
      - POST to `https://veo-3-fast.p.rapidapi.com/predictions`  
      - Headers include RapidAPI host and API key (must be replaced with a valid key).  
      - JSON body includes "version" (fixed API model version) and "input" with "prompt" from form data and "enhance_prompt": true.  
    - Expressions: Uses `{{ $json.Prompt }}` to insert prompt dynamically.  
    - Input: Form submission data  
    - Output: API response expected to contain a task ID.  
    - OnError: Continues workflow even if error occurs (to handle with conditional branching).  
    - Edge Cases: Invalid or missing API key, network errors, API rate limits, malformed prompt, partial or error responses from API.

---

#### 1.3 Task ID Verification

- **Overview:**  
  Checks if the API response includes a task ID, which is required to monitor task status.

- **Nodes Involved:**  
  - Condition: Check Task Id  
  - Send Email: API Error - Task ID Missing

- **Node Details:**

  - **Condition: Check Task Id**  
    - Type: If Node  
    - Role: Checks if `$json.id` exists in the API response.  
    - Configuration: Condition tests existence of `id` property strictly.  
    - Input: Response from Veo 3 Fast API Processor  
    - Output:  
      - If true: proceeds to wait for API response.  
      - If false: triggers error email node.  
    - Edge Cases: Missing or malformed API response; false negatives if API changes response structure.

  - **Send Email: API Error - Task ID Missing**  
    - Type: Email Send  
    - Role: Notifies developers/admins that the API did not return a task ID.  
    - Configuration:  
      - Sends email with subject "Failed To Get Task ID"  
      - Fixed recipient and sender addresses (`dev@test.com` and `it-admin@test.com`)  
      - SMTP credentials configured for sending  
    - Input: Triggered when task ID is missing.  
    - Edge Cases: Email server downtime, invalid SMTP credentials.

---

#### 1.4 Task Status Polling

- **Overview:**  
  Polls the Veo 3 Fast API to check the status of the video generation task until it is succeeded, processing, or failed.

- **Nodes Involved:**  
  - Wait for API Response  
  - API Request: Check Task Status  
  - Condition: Task Output Status  
  - Wait for Task to Complete  
  - Send Email: API Error - Task Failed

- **Node Details:**

  - **Wait for API Response**  
    - Type: Wait  
    - Role: Pauses workflow for 35 seconds before the first status check to allow API processing.  
    - Configuration: Wait 35 seconds.  
    - Edge Cases: Excessive wait time may delay workflow; insufficient wait may cause premature status checking.

  - **API Request: Check Task Status**  
    - Type: HTTP Request  
    - Role: Queries Veo 3 Fast API for task status using task ID.  
    - Configuration:  
      - GET to `https://veo-3-fast.p.rapidapi.com/predictions/{{ $json.id }}`  
      - Headers include RapidAPI host and API key.  
    - Input: Task ID from previous nodes.  
    - Output: Task status JSON.  
    - Edge Cases: Network errors, invalid task ID, API rate limit, auth errors.

  - **Condition: Task Output Status**  
    - Type: Switch  
    - Role: Routes workflow based on task status field (`succeeded`, `processing`, `failed`).  
    - Output:  
      - `succeeded`: proceeds to download video.  
      - `processing`: waits and rechecks status.  
      - `failed`: triggers error email.  
    - Edge Cases: Unexpected status values, case sensitivity issues.

  - **Wait for Task to Complete**  
    - Type: Wait  
    - Role: Pauses for 30 seconds before rechecking task status if still processing.  
    - Configuration: Wait 30 seconds.  
    - Edge Cases: Similar to previous wait node.

  - **Send Email: API Error - Task Failed**  
    - Type: Email Send  
    - Role: Notifies developers/admins that video generation failed for the given task ID.  
    - Configuration:  
      - Email subject: "Failed To Generate Image"  
      - Includes task ID in email body.  
      - SMTP credentials configured.  
    - Edge Cases: Email delivery issues as before.

---

#### 1.5 Video Download and Storage

- **Overview:**  
  Downloads the generated video from the API output URL and uploads it to Google Drive.

- **Nodes Involved:**  
  - Download Video  
  - Upload File to Google Drive

- **Node Details:**

  - **Download Video**  
    - Type: HTTP Request  
    - Role: Downloads the video binary using the URL in the API response’s output field.  
    - Configuration: URL dynamically set as `{{ $json.output }}` (expected to be direct video URL).  
    - Output: Binary video data for upload.  
    - Edge Cases: URL invalid or expired, network errors, large file size handling.

  - **Upload File to Google Drive**  
    - Type: Google Drive  
    - Role: Uploads the downloaded video file to Google Drive user’s root folder.  
    - Configuration:  
      - Drive: "My Drive"  
      - Folder: root  
      - Uses Google OAuth2 credentials.  
    - Input: Binary data from download node.  
    - Output: Google Drive file metadata including file ID and webViewLink.  
    - Edge Cases: Invalid OAuth tokens, upload failures, API quota limits.

---

#### 1.6 File Sharing and User Notification

- **Overview:**  
  Sets sharing permissions on the uploaded Google Drive file and sends the user an email with the video link.

- **Nodes Involved:**  
  - Set Google Drive Permissions  
  - Send an email : Video Link

- **Node Details:**

  - **Set Google Drive Permissions**  
    - Type: Google Drive  
    - Role: Shares the uploaded video file by setting required permissions (default is public or specified sharing).  
    - Configuration:  
      - Operation: Share file by file ID from upload node.  
      - Uses OAuth2 credentials.  
    - Edge Cases: Permission setting failures, insufficient rights.

  - **Send an email : Video Link**  
    - Type: Email Send  
    - Role: Sends an email to the user notifying that the video is ready, including a link to the Google Drive file.  
    - Configuration:  
      - Recipient: `user@test.com` (fixed in workflow, may require dynamic input for real use)  
      - Sender: `admin@test.com`  
      - Email body includes dynamic link `{{ $('Upload File to Google Drive').item.json.webViewLink }}`  
      - SMTP credentials configured.  
    - Edge Cases: Email server errors, incorrect recipient address.

---

### 3. Summary Table

| Node Name                       | Node Type            | Functional Role                                    | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                           |
|--------------------------------|----------------------|--------------------------------------------------|--------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger         | Entry point; receives prompt from user form       | -                              | Veo 3 Fast API Processor               | Triggers the workflow when a form is submitted with a prompt field.                                  |
| Veo 3 Fast API Processor       | HTTP Request         | Sends prompt to Veo 3 Fast API for video generation | On form submission             | Condition: Check Task Id               | Processes the form's prompt by making an API call to the "Veo 3 Fast" API.                           |
| Condition: Check Task Id       | If                   | Validates presence of task ID in API response      | Veo 3 Fast API Processor       | Wait for API Response / Send Email: API Error - Task ID Missing | Checks if the task ID is present before continuing; sends an error email if missing.                 |
| Send Email: API Error - Task ID Missing | Email Send         | Sends alert if API response lacks task ID          | Condition: Check Task Id (false branch) | -                                     | Sends an error email if the task ID is missing in the response.                                      |
| Wait for API Response          | Wait                 | Waits 35 seconds before checking task status       | Condition: Check Task Id (true branch) | API Request: Check Task Status        | Waits for 35 seconds to allow the API response to be processed.                                      |
| API Request: Check Task Status | HTTP Request         | Queries API for current task status                 | Wait for API Response          | Condition: Task Output Status          | Sends an HTTP request to check the status of a task using its ID.                                   |
| Condition: Task Output Status  | Switch               | Routes based on task status: succeeded, processing, failed | API Request: Check Task Status | Download Video / Wait for Task to Complete / Send Email: API Error - Task Failed | Checks the task's output status for success, processing, or failure.                                |
| Wait for Task to Complete      | Wait                 | Waits 30 seconds before rechecking task status      | Condition: Task Output Status (processing branch) | API Request: Check Task Status        | Waits for 30 seconds before rechecking the task’s completion status.                                |
| Send Email: API Error - Task Failed | Email Send         | Alerts on video generation failure                   | Condition: Task Output Status (failed branch) | -                                     | Sends an email if the video generation task fails.                                                  |
| Download Video                 | HTTP Request         | Downloads generated video from API output URL       | Condition: Task Output Status (succeeded branch) | Upload File to Google Drive            | Downloads the processed video from the output URL provided by the API response.                     |
| Upload File to Google Drive    | Google Drive         | Uploads video file to Google Drive                   | Download Video                | Set Google Drive Permissions           | Uploads the processed video to Google Drive.                                                        |
| Set Google Drive Permissions   | Google Drive         | Sets sharing permissions for uploaded video file    | Upload File to Google Drive    | Send an email : Video Link             | Sets the necessary permissions for the uploaded Google Drive file.                                  |
| Send an email : Video Link     | Email Send           | Sends video link email to the user                   | Set Google Drive Permissions   | -                                     | Sends an email if the video generation is completed.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "On form submission"**  
   - Type: Form Trigger  
   - Configure form title: "CGI Ads With Google Veo3"  
   - Add one required field labeled "Prompt"  
   - Save and note the webhook URL for testing.

2. **Add HTTP Request Node: "Veo 3 Fast API Processor"**  
   - Connect from "On form submission"  
   - Method: POST  
   - URL: `https://veo-3-fast.p.rapidapi.com/predictions`  
   - Headers:  
     - `x-rapidapi-host`: `veo-3-fast.p.rapidapi.com`  
     - `x-rapidapi-key`: *Your API Key* (replace with valid key)  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "d7aca9396ea28c4ef46700a43cb59546c9948396eb571ca083df8344391335b3",
       "input": {
         "prompt": "{{ $json.Prompt }}",
         "enhance_prompt": true
       }
     }
     ```  
   - On Error: Continue  
   - Save.

3. **Add If Node: "Condition: Check Task Id"**  
   - Connect from "Veo 3 Fast API Processor"  
   - Condition: Check if `$json.id` exists (strict)  
   - True branch: Continue workflow  
   - False branch: Send error email.

4. **Add Email Send Node: "Send Email: API Error - Task ID Missing"**  
   - Connect from False branch of previous node  
   - Configure SMTP credentials (e.g., "SMTP account 2")  
   - To: `dev@test.com`  
   - From: `it-admin@test.com`  
   - Subject: "Failed To Get Task ID"  
   - Body (HTML): "Hey This Just Inform To You API Doesn't Return task id"  
   - Save.

5. **Add Wait Node: "Wait for API Response"**  
   - Connect from True branch of task ID check  
   - Duration: 35 seconds  
   - Save.

6. **Add HTTP Request Node: "API Request: Check Task Status"**  
   - Connect from "Wait for API Response"  
   - Method: GET  
   - URL: `https://veo-3-fast.p.rapidapi.com/predictions/{{ $json.id }}`  
   - Headers:  
     - `x-rapidapi-host`: `veo-3-fast.p.rapidapi.com`  
     - `x-rapidapi-key`: *Your API Key* (same as above)  
   - Save.

7. **Add Switch Node: "Condition: Task Output Status"**  
   - Connect from "API Request: Check Task Status"  
   - Conditions: Check `$json.status` equals one of:  
     - "succeeded"  
     - "processing"  
     - "failed"  
   - Route accordingly:  
     - succeeded → Download Video  
     - processing → Wait for Task to Complete  
     - failed → Send Email: API Error - Task Failed

8. **Add Wait Node: "Wait for Task to Complete"**  
   - Connect from processing branch of Switch  
   - Duration: 30 seconds  
   - Connect back to "API Request: Check Task Status" for polling loop.

9. **Add Email Send Node: "Send Email: API Error - Task Failed"**  
   - Connect from failed branch of Switch  
   - Configure SMTP credentials  
   - To: `dev@test.com`  
   - From: `it-admin@test.com`  
   - Subject: "Failed To Generate Image"  
   - Body (HTML): `Task id : {{ $json.id }} Failed to generate video.`  
   - Save.

10. **Add HTTP Request Node: "Download Video"**  
    - Connect from succeeded branch of Switch  
    - Method: GET  
    - URL: `{{ $json.output }}` (dynamic from API response)  
    - Save.

11. **Add Google Drive Node: "Upload File to Google Drive"**  
    - Connect from "Download Video"  
    - Operation: Upload file  
    - Drive: "My Drive"  
    - Folder: root (or choose as needed)  
    - Credentials: Google Drive OAuth2 configured  
    - Upload binary from previous node  
    - Save.

12. **Add Google Drive Node: "Set Google Drive Permissions"**  
    - Connect from "Upload File to Google Drive"  
    - Operation: Share file  
    - File ID: `{{ $json.id }}` from upload node output  
    - Credentials: Google Drive OAuth2  
    - Save.

13. **Add Email Send Node: "Send an email : Video Link"**  
    - Connect from "Set Google Drive Permissions"  
    - Configure SMTP credentials  
    - To: `user@test.com` (adjust for actual user email as needed)  
    - From: `admin@test.com`  
    - Subject: "Your video is ready" (optional)  
    - Body (HTML):  
      ```
      Hey,  
      Your video is ready.  
      Link: {{ $('Upload File to Google Drive').item.json.webViewLink }}
      ```  
    - Save.

14. **Test the entire workflow:**  
    - Submit the form with a prompt.  
    - Monitor logs and emails.  
    - Ensure API keys and OAuth credentials are valid and permissions correct.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates video generation using Veo 3 Fast API, Google Drive upload, and email notifications.                                                                     | Workflow purpose summary from Sticky Note13                                                    |
| Be sure to replace all placeholder API keys (`Your key`) with valid RapidAPI keys for Veo 3 Fast API.                                                                             | Critical for API authentication                                                                 |
| Google Drive OAuth2 credentials must be configured with appropriate scopes for uploading and sharing files.                                                                       | Required for Google Drive nodes                                                                 |
| SMTP credentials must be configured correctly for email notifications to be sent.                                                                                                 | Required for all Email Send nodes                                                              |
| The fixed email addresses (`dev@test.com`, `user@test.com`) should be replaced or parameterized for production use.                                                              | For real-world use, dynamic user emails are recommended                                        |
| Wait durations (35s and 30s) are designed to balance API processing time and workflow responsiveness; adjust as needed.                                                           | Timing considerations                                                                           |
| Video download URL is expected to be directly accessible; if API changes, update the URL extraction logic.                                                                         | Potential API response changes                                                                 |
| The workflow includes error handling to notify on missing task ID and task failure. These emails help monitor the automated process.                                              | Reliability and monitoring                                                                      |
| More information on n8n’s Google Drive and HTTP Request nodes can be found in the official documentation: https://docs.n8n.io/nodes/n8n-nodes-base.googledrive/ and https://docs.n8n.io/nodes/n8n-nodes-base.httprequest/ | Official n8n documentation                                                                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.