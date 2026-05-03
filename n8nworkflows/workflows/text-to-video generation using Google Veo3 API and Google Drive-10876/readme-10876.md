text-to-video generation using Google Veo3 API and Google Drive

https://n8nworkflows.xyz/workflows/text-to-video-generation-using-google-veo3-api-and-google-drive-10876


# text-to-video generation using Google Veo3 API and Google Drive

### 1. Workflow Overview

This workflow automates the generation of videos from text prompts using the Google Veo3 API, followed by managing the resulting video files on Google Drive and notifying users via email. It is designed for CGI ad creation or any use case requiring automated text-to-video generation with cloud storage and alerting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input via a form submission containing a text prompt.
- **1.2 Video Generation Request:** Sends the prompt to the Google Veo3 API to initiate video generation.
- **1.3 Task Status Polling:** Periodically checks the status of the video generation task until completion or failure.
- **1.4 Error Handling:** Sends email notifications if the task ID is missing or if the video generation fails.
- **1.5 Video Download and Upload:** Downloads the generated video and uploads it to Google Drive.
- **1.6 Google Drive Permissions:** Sets sharing permissions on the uploaded video file.
- **1.7 Final Notification:** Sends an email to the user with a link to the completed video.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user submits a form containing a text prompt for video generation.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Configuration: Listens for submissions on a form titled "Txt to video With Google Veo3" with one required field labeled "Prompt".  
    - Key Expressions: The prompt is captured from the form field `Prompt`.  
    - Input: External user form submission  
    - Output: JSON containing the prompt text  
    - Edge Cases: Missing prompt field would prevent continuation as it is required.  
    - Sticky Note: Describes the trigger and prompt field.

---

#### 2.2 Video Generation Request

- **Overview:**  
  Sends the user’s prompt to the Google Veo3 API to start the video generation task.

- **Nodes Involved:**  
  - Veo 3 API Processor

- **Node Details:**

  - **Veo 3 API Processor**  
    - Type: HTTP Request  
    - Configuration: POST request to `https://google-veo-3.p.rapidapi.com/predictions` with a JSON body containing the prompt and an enhancement flag. Uses RapidAPI headers including `x-rapidapi-host` and `x-rapidapi-key` (API key placeholder).  
    - Key Expressions: The prompt is injected dynamically using `{{ $json.Prompt }}`.  
    - Input: JSON from form submission node  
    - Output: JSON response expected to contain a task ID for tracking  
    - Edge Cases: API errors, invalid API key, or malformed prompt could cause failure. The node is configured to continue on error to allow workflow error handling.  
    - Sticky Note: Describes the API call to Veo3.

---

#### 2.3 Task Status Polling

- **Overview:**  
  This block checks if the task ID exists, waits for the API to process the request, and polls the task status until it is either succeeded, processing, or failed.

- **Nodes Involved:**  
  - Condition: Check Task Id  
  - Wait for API Response  
  - API Request: Check Task Status  
  - Condition: Task Output Status  
  - Wait for Task to Complete

- **Node Details:**

  - **Condition: Check Task Id**  
    - Type: If  
    - Configuration: Checks if the `id` field exists in the API response JSON.  
    - Input: Output from Veo 3 API Processor  
    - Output: Continues if ID exists; otherwise triggers error email.  
    - Edge Cases: Missing task ID triggers error handling.  
    - Sticky Note: Notes the check for task ID presence.

  - **Wait for API Response**  
    - Type: Wait  
    - Configuration: Waits 35 seconds before checking task status.  
    - Input: From Condition node if task ID exists  
    - Output: Triggers status check request  
    - Edge Cases: Fixed wait time may be insufficient or excessive depending on API response time.  
    - Sticky Note: Explains the wait period.

  - **API Request: Check Task Status**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://google-veo-3.p.rapidapi.com/predictions/{{ $json.id }}` with RapidAPI headers.  
    - Input: Task ID from previous node  
    - Output: JSON with current task status (`succeeded`, `processing`, or `failed`)  
    - Edge Cases: Network errors, invalid task ID, or API rate limits.  
    - Sticky Note: Describes the status check request.

  - **Condition: Task Output Status**  
    - Type: Switch  
    - Configuration: Routes workflow based on the `status` field in the task status response:  
      - `succeeded` → proceed to download video  
      - `processing` → wait and recheck  
      - `failed` → trigger failure email  
    - Input: API Request node output  
    - Output: Branches accordingly  
    - Edge Cases: Unexpected status values or missing status field.  
    - Sticky Note: Explains status-based routing.

  - **Wait for Task to Complete**  
    - Type: Wait  
    - Configuration: Waits 30 seconds before rechecking task status if still processing.  
    - Input: From Condition node if status is `processing`  
    - Output: Loops back to API Request node  
    - Edge Cases: Potential infinite loop if task never completes; no max retry limit configured.  
    - Sticky Note: Describes wait before rechecking.

---

#### 2.4 Error Handling

- **Overview:**  
  Sends email notifications if the task ID is missing or if the video generation task fails.

- **Nodes Involved:**  
  - Send Email: API Error - Task ID Missing  
  - Send Email: API Error - Task Failed

- **Node Details:**

  - **Send Email: API Error - Task ID Missing**  
    - Type: Email Send  
    - Configuration: Sends an email to `dev@test.com` from `it-admin@test.com` with subject "Failed To Get Task ID" and a message indicating the API did not return a task ID.  
    - Input: Triggered if task ID check fails  
    - Output: Ends workflow or error branch  
    - Credentials: SMTP account configured  
    - Edge Cases: SMTP failures or invalid email addresses.  
    - Sticky Note: Notes purpose of this error email.

  - **Send Email: API Error - Task Failed**  
    - Type: Email Send  
    - Configuration: Sends an email to `dev@test.com` from `it-admin@test.com` with subject "Failed To Generate Image" including the failed task ID in the message.  
    - Input: Triggered if task status is `failed`  
    - Output: Ends workflow or error branch  
    - Credentials: SMTP account configured  
    - Edge Cases: SMTP failures or invalid email addresses.  
    - Sticky Note: Notes purpose of this error email.

---

#### 2.5 Video Download and Upload

- **Overview:**  
  Downloads the generated video from the URL provided by the API and uploads it to Google Drive.

- **Nodes Involved:**  
  - Download Video  
  - Upload File to Google Drive

- **Node Details:**

  - **Download Video**  
    - Type: HTTP Request  
    - Configuration: Downloads the video file from the URL found in the `output` field of the task status response.  
    - Input: From Condition node on `succeeded` status  
    - Output: Binary video data for upload  
    - Edge Cases: Download failures, invalid URLs, or network issues.  
    - Sticky Note: Describes video download step.

  - **Upload File to Google Drive**  
    - Type: Google Drive  
    - Configuration: Uploads the downloaded video file to the root folder of the configured Google Drive account.  
    - Input: Binary video data from Download Video node  
    - Output: JSON with uploaded file metadata including file ID and web view link  
    - Credentials: Google Drive OAuth2 account configured  
    - Edge Cases: Upload failures, permission issues, or quota limits.  
    - Sticky Note: Notes upload to Google Drive.

---

#### 2.6 Google Drive Permissions

- **Overview:**  
  Sets sharing permissions on the uploaded video file to allow access.

- **Nodes Involved:**  
  - Set Google Drive Permissions

- **Node Details:**

  - **Set Google Drive Permissions**  
    - Type: Google Drive  
    - Configuration: Shares the uploaded file by setting permissions using the file ID from the upload node. Uses OAuth2 authentication.  
    - Input: File ID from Upload File to Google Drive node  
    - Output: Confirmation of permission settings  
    - Credentials: Google Drive OAuth2 account configured  
    - Edge Cases: Permission setting failures or insufficient privileges.  
    - Sticky Note: Describes permission setting.

---

#### 2.7 Final Notification

- **Overview:**  
  Sends an email to the user with a link to the generated video on Google Drive.

- **Nodes Involved:**  
  - Send an email : Video Link

- **Node Details:**

  - **Send an email : Video Link**  
    - Type: Email Send  
    - Configuration: Sends an email to `user@test.com` from `admin@test.com` with a message containing the Google Drive web view link to the video.  
    - Input: File metadata from Set Google Drive Permissions node  
    - Output: Ends workflow successfully  
    - Credentials: SMTP account configured  
    - Edge Cases: SMTP failures or invalid email addresses.  
    - Sticky Note: Notes final notification email.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                                  | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                      |
|----------------------------------|---------------------|-------------------------------------------------|--------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| On form submission               | Form Trigger        | Receives user prompt input                       | -                              | Veo 3 API Processor               | Triggers the workflow when a form is submitted with a prompt field.                            |
| Veo 3 API Processor             | HTTP Request        | Sends prompt to Google Veo3 API                  | On form submission             | Condition: Check Task Id          | Processes the form's prompt by making an API call to the "Veo 3" API.                          |
| Condition: Check Task Id        | If                  | Checks if task ID exists in API response         | Veo 3 API Processor            | Wait for API Response, Send Email: API Error - Task ID Missing | Checks if the task ID is present before continuing; sends an error email if missing.           |
| Wait for API Response           | Wait                | Waits 35 seconds before checking task status     | Condition: Check Task Id       | API Request: Check Task Status    | Waits for 35 seconds to allow the API response to be processed.                               |
| API Request: Check Task Status  | HTTP Request        | Checks current status of video generation task   | Wait for API Response          | Condition: Task Output Status     | Sends an HTTP request to check the status of a task using its ID.                             |
| Condition: Task Output Status   | Switch              | Routes workflow based on task status             | API Request: Check Task Status | Download Video, Wait for Task to Complete, Send Email: API Error - Task Failed | Checks the task's output status for success, processing, or failure.                           |
| Wait for Task to Complete       | Wait                | Waits 30 seconds before rechecking task status   | Condition: Task Output Status  | API Request: Check Task Status    | Waits for 30 seconds before rechecking the task’s completion status.                          |
| Download Video                 | HTTP Request        | Downloads the generated video file                | Condition: Task Output Status  | Upload File to Google Drive       | Downloads the processed video from the output URL provided by the API response.               |
| Upload File to Google Drive     | Google Drive        | Uploads video file to Google Drive                | Download Video                 | Set Google Drive Permissions      | Uploads the processed video to Google Drive.                                                  |
| Set Google Drive Permissions    | Google Drive        | Sets sharing permissions on uploaded video       | Upload File to Google Drive    | Send an email : Video Link        | Sets the necessary permissions for the uploaded Google Drive file.                            |
| Send an email : Video Link      | Email Send          | Sends user email with video link                   | Set Google Drive Permissions   | -                                | Sends an email if the video generation is completed.                                         |
| Send Email: API Error - Task ID Missing | Email Send          | Sends error email if task ID is missing           | Condition: Check Task Id       | -                                | Sends an error email if the task ID is missing in the response.                              |
| Send Email: API Error - Task Failed | Email Send          | Sends error email if video generation fails       | Condition: Task Output Status  | -                                | Sends an email if the video generation task fails.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Type: Form Trigger  
   - Configure form title as "Txt to video With Google Veo3"  
   - Add one required field labeled "Prompt"  
   - This node will start the workflow upon form submission.

2. **Add an HTTP Request node named "Veo 3 API Processor"**  
   - Method: POST  
   - URL: `https://google-veo-3.p.rapidapi.com/predictions`  
   - Headers:  
     - `x-rapidapi-host`: `google-veo-3.p.rapidapi.com`  
     - `x-rapidapi-key`: *Your RapidAPI key here*  
   - Body (JSON):  
     ```json
     {
       "version": "590348ebd4cb656f3fc5b9270c4c19fb2abc5d1ae6101f7874413a3ec545260d",
       "input": {
         "prompt": "{{ $json.Prompt }}",
         "enhance_prompt": true
       }
     }
     ```  
   - Set Content-Type to JSON  
   - Connect from Form Trigger node  
   - Enable "Continue on Fail" to allow error handling downstream.

3. **Add an If node named "Condition: Check Task Id"**  
   - Condition: Check if `id` field exists in the JSON response (`{{ $json.id }}` exists)  
   - Connect from Veo 3 API Processor node  
   - True branch continues workflow; False branch leads to error email.

4. **Add a Wait node named "Wait for API Response"**  
   - Wait time: 35 seconds  
   - Connect from True output of Condition: Check Task Id node.

5. **Add an HTTP Request node named "API Request: Check Task Status"**  
   - Method: GET  
   - URL: `https://google-veo-3.p.rapidapi.com/predictions/{{ $json.id }}`  
   - Headers same as Veo 3 API Processor node  
   - Connect from Wait for API Response node.

6. **Add a Switch node named "Condition: Task Output Status"**  
   - Evaluate `{{ $json.status }}`  
   - Cases:  
     - Equals "succeeded" → proceed to download video  
     - Equals "processing" → wait and recheck  
     - Equals "failed" → send failure email  
   - Connect from API Request: Check Task Status node.

7. **Add a Wait node named "Wait for Task to Complete"**  
   - Wait time: 30 seconds  
   - Connect from "processing" output of Condition: Task Output Status node  
   - Connect output back to API Request: Check Task Status node to poll again.

8. **Add an HTTP Request node named "Download Video"**  
   - Method: GET  
   - URL: `{{ $json.output }}` (URL from task status response)  
   - Connect from "succeeded" output of Condition: Task Output Status node.

9. **Add a Google Drive node named "Upload File to Google Drive"**  
   - Operation: Upload file  
   - Folder: Root (`/ (Root folder)`) or specify desired folder  
   - Credentials: Configure Google Drive OAuth2 credentials  
   - Connect from Download Video node.

10. **Add a Google Drive node named "Set Google Drive Permissions"**  
    - Operation: Share file  
    - File ID: Use `{{ $json.id }}` from Upload File to Google Drive node output  
    - Permissions: Default sharing (e.g., anyone with link can view)  
    - Credentials: Use same Google Drive OAuth2 credentials  
    - Connect from Upload File to Google Drive node.

11. **Add an Email Send node named "Send an email : Video Link"**  
    - To: `user@test.com` (replace with actual recipient)  
    - From: `admin@test.com`  
    - Subject: "Your video is ready"  
    - Body (HTML):  
      ```
      Hey,  
      Your video is ready.  
      Link: {{ $('Upload File to Google Drive').item.json.webViewLink }}
      ```  
    - Credentials: Configure SMTP credentials  
    - Connect from Set Google Drive Permissions node.

12. **Add an Email Send node named "Send Email: API Error - Task ID Missing"**  
    - To: `dev@test.com`  
    - From: `it-admin@test.com`  
    - Subject: "Failed To Get Task ID"  
    - Body (HTML): "Hey This Just Inform To You API Doesn't Return task id"  
    - Credentials: Configure SMTP credentials  
    - Connect from False output of Condition: Check Task Id node.

13. **Add an Email Send node named "Send Email: API Error - Task Failed"**  
    - To: `dev@test.com`  
    - From: `it-admin@test.com`  
    - Subject: "Failed To Generate Image"  
    - Body (HTML):  
      ```
      Task id : {{ $json.id }}  
      Failed to generate video.
      ```  
    - Credentials: Configure SMTP credentials  
    - Connect from "failed" output of Condition: Task Output Status node.

---

This stepwise reconstruction ensures the workflow can be recreated fully in n8n, including all API calls, waits, condition checks, error handling, file management, and notifications. Credentials for Google Drive OAuth2 and SMTP must be properly set up and linked to the respective nodes. The RapidAPI key for Google Veo3 API must be obtained and inserted in the HTTP Request nodes.