Create Stunning CGI Videos with Sora 2 API and Google Drive Integration

https://n8nworkflows.xyz/workflows/create-stunning-cgi-videos-with-sora-2-api-and-google-drive-integration-10861


# Create Stunning CGI Videos with Sora 2 API and Google Drive Integration

### 1. Workflow Overview

This workflow automates the creation of stunning CGI videos using the Sora 2 API, integrated with Google Drive for file storage and email notifications for status updates. It is designed to handle user prompt input, video generation request, task status polling, video retrieval, uploading to Google Drive, permission setting, and user notification. The main logical blocks include:

- **1.1 Input Reception:** Triggered by a form submission containing a video generation prompt.
- **1.2 Sora 2 API Video Generation:** Sends the prompt to the Sora 2 API and obtains a prediction/task ID.
- **1.3 Task Status Polling:** Waits and repeatedly checks the status of the video generation task.
- **1.4 Task Result Handling:** Handles success, ongoing generation, or failure states.
- **1.5 Video Download & Upload:** Downloads the generated video and uploads it to Google Drive.
- **1.6 Google Drive Sharing & Notification:** Sets sharing permissions on the uploaded video and sends an email notification with the video link.
- **1.7 Error Handling:** Sends error emails when prediction ID is missing or task fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception  
- **Overview:** Captures user input from a web form submission with a prompt field for video generation.  
- **Nodes Involved:**  
  - On form submission  
  - Sticky Note1  
- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Configuration: Listens for submissions of form titled "CGI Ads With Sora 2" with a required "Prompt" field.  
    - Input: HTTP webhook (form submit)  
    - Output: JSON containing the prompt entered  
    - Edge Cases: Missing prompt will reject submission; form server errors or webhook issues possible.  
  - **Sticky Note1**  
    - Functional: Describes the form trigger purpose.

#### 1.2 Sora 2 API Video Generation  
- **Overview:** Sends the prompt to Sora 2 API to start video generation and retrieve a prediction ID.  
- **Nodes Involved:**  
  - Sora API Processor  
  - Condition: Check Prediction Id  
  - Send Email: API Error - Task ID Missing  
  - Sticky Note2  
  - Sticky Note3  
- **Node Details:**  
  - **Sora API Processor**  
    - Type: HTTP Request (POST)  
    - Configuration: Sends JSON body with prompt, quality="hd", aspect_ratio="landscape". Uses RapidAPI headers with host and key.  
    - Inputs: Prompt JSON from form submission  
    - Outputs: API response including predictionId  
    - On error: Continue regular output (handles API errors gracefully)  
    - Edge Cases: API key invalid, network errors, malformed response, prompt injection attacks  
  - **Condition: Check Prediction Id**  
    - Type: If Node  
    - Configuration: Checks if predictionId exists and is non-empty  
    - Input: Output from API Processor  
    - Outputs: Two branches - predictionId present or missing  
  - **Send Email: API Error - Task ID Missing**  
    - Type: Email Send  
    - Configuration: Sends email to dev@test.com notifying missing predictionId error  
    - Input: Condition failure branch  
    - Edge Cases: SMTP authentication failure, email delivery issues  
  - **Sticky Notes 2 & 3**  
    - Explain the API request and prediction ID validation steps.

#### 1.3 Task Status Polling  
- **Overview:** Waits 60 seconds and polls the Sora API for task status using the predictionId until completion or failure.  
- **Nodes Involved:**  
  - Wait for API Response  
  - API Request: Check Task Status  
  - Condition: Task Output Status  
  - Wait for Task to Complete  
  - Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note7  
- **Node Details:**  
  - **Wait for API Response**  
    - Type: Wait  
    - Configuration: Waits 60 seconds before initial status check  
  - **API Request: Check Task Status**  
    - Type: HTTP Request (GET)  
    - Configuration: Calls https://sora-2-openai.p.rapidapi.com/result.php with predictionId query parameter and RapidAPI headers  
    - Edge Cases: API timeouts, invalid ID, network errors  
  - **Condition: Task Output Status**  
    - Type: Switch Node  
    - Configuration: Checks status field in response: "success", "generating", or "failed"  
    - Outputs:  
      - Success: proceeds to clean output  
      - Generating: waits again 60 seconds (loops)  
      - Failed: triggers error email  
  - **Wait for Task to Complete**  
    - Type: Wait  
    - Configuration: Waits 60 seconds before re-polling status  
  - **Sticky Notes 4-7**  
    - Provide explanations for waiting and polling logic, HTTP status check, and task result evaluation.

#### 1.4 Task Result Handling  
- **Overview:** On success, processes the API response to extract video URLs, downloads the video file, and handles failure by notifying via email.  
- **Nodes Involved:**  
  - Clean Output  
  - Download Video  
  - Send Email: API Error - Task Failed  
  - Sticky Note12, Sticky Note11, Sticky Note13  
- **Node Details:**  
  - **Clean Output**  
    - Type: Code (JavaScript)  
    - Configuration: Parses JSON response, extracts `resultUrls` array for video URLs  
    - Input: API success response  
    - Output: JSON with `resultUrls` field  
  - **Download Video**  
    - Type: HTTP Request (GET)  
    - Configuration: Downloads video from first URL in `resultUrls`  
    - Edge Cases: URL invalid, download failure, slow response  
  - **Send Email: API Error - Task Failed**  
    - Type: Email Send  
    - Configuration: Sends email to dev@test.com with task id and failure notice  
    - Input: Failure branch from status condition  
  - **Sticky Notes 11-13**  
    - Describe failure notification, video download, and output cleaning.

#### 1.5 Video Upload to Google Drive  
- **Overview:** Uploads the downloaded video file to Google Drive root folder and sets sharing permissions.  
- **Nodes Involved:**  
  - Upload File to Google Drive  
  - Set Google Drive Permissions  
  - Sticky Note9, Sticky Note10  
- **Node Details:**  
  - **Upload File to Google Drive**  
    - Type: Google Drive Node (Upload File)  
    - Configuration: Uploads file to My Drive root folder  
    - Credentials: Google Drive OAuth2  
    - Input: Binary video data from download  
    - Output: Google Drive file metadata including file ID and webViewLink  
  - **Set Google Drive Permissions**  
    - Type: Google Drive Node (Share File)  
    - Configuration: Shares the uploaded file for public or specific access (default permissions)  
    - Input: File ID from upload node  
    - Credentials: Google Drive OAuth2  
  - **Sticky Notes 9 & 10**  
    - Explain upload and permission setting steps.

#### 1.6 Notification Email to User  
- **Overview:** Sends an email to the user notifying that the video is ready, including a link to the Google Drive hosted video.  
- **Nodes Involved:**  
  - Send an email : Video Link  
  - Sticky Note (unnumbered at end)  
- **Node Details:**  
  - **Send an email : Video Link**  
    - Type: Email Send  
    - Configuration: Sends an email to user@test.com with link to video using Google Drive `webViewLink` from upload node  
    - Credentials: SMTP  
  - **Sticky Note**  
    - Notes that this email is sent upon successful video generation.

#### 1.7 Error Handling  
- **Overview:** Sends emails to developers if prediction ID is missing or video generation task fails.  
- **Nodes Involved:**  
  - Send Email: API Error - Task ID Missing  
  - Send Email: API Error - Task Failed  
  - Sticky Note8, Sticky Note11  
- **Node Details:**  
  - Sends descriptive emails to dev@test.com with relevant error information for troubleshooting.  
  - Edge Cases: SMTP errors, email delivery failures.

---

### 3. Summary Table

| Node Name                        | Node Type            | Functional Role                                  | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                             |
|---------------------------------|----------------------|------------------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger         | Receives prompt input from user form           | —                            | Sora API Processor            | Triggers the workflow when a form is submitted with a prompt field.                                   |
| Sora API Processor              | HTTP Request         | Sends prompt to Sora 2 API to generate video   | On form submission           | Condition: Check Prediction Id | Processes the form's prompt by making an API call to the Sora API.                                    |
| Condition: Check Prediction Id  | If                   | Verifies presence of prediction ID in response | Sora API Processor           | Wait for API Response, Send Email: API Error - Task ID Missing | Checks if the prediction id is present before continuing; sends an error email if missing.            |
| Send Email: API Error - Task ID Missing | Email Send          | Notifies missing prediction ID error           | Condition: Check Prediction Id | —                             | Sends an error email if the prediction id is missing in the response.                                |
| Wait for API Response           | Wait                 | Waits 60 seconds before first status check     | Condition: Check Prediction Id | API Request: Check Task Status | Waits for 60 seconds to allow the API response to be processed.                                       |
| API Request: Check Task Status  | HTTP Request         | Polls Sora API for task status                  | Wait for API Response, Wait for Task to Complete | Condition: Task Output Status     | Sends an HTTP request to check the status of a task using its ID.                                    |
| Condition: Task Output Status   | Switch               | Routes based on task status (success, generating, failed) | API Request: Check Task Status | Clean Output, Wait for Task to Complete, Send Email: API Error - Task Failed | Checks the task's output status for success, processing, or failure.                                 |
| Wait for Task to Complete       | Wait                 | Waits 60 seconds before rechecking task status | Condition: Task Output Status | API Request: Check Task Status | Waits for 60 seconds before rechecking the task’s completion status.                                 |
| Send Email: API Error - Task Failed | Email Send          | Notifies task failure                           | Condition: Task Output Status | —                             | Sends an email if the video generation task fails.                                                    |
| Clean Output                   | Code (JavaScript)    | Extracts result URLs from the API response      | Condition: Task Output Status | Download Video                | Clean output & process result Url.                                                                    |
| Download Video                 | HTTP Request         | Downloads video file from provided URL          | Clean Output                 | Upload File to Google Drive   | Downloads the processed video from the output URL provided by the API response.                      |
| Upload File to Google Drive    | Google Drive         | Uploads video file to Google Drive root folder  | Download Video               | Set Google Drive Permissions  | Uploads the processed video to Google Drive.                                                         |
| Set Google Drive Permissions   | Google Drive         | Shares the uploaded file with appropriate permissions | Upload File to Google Drive   | Send an email : Video Link     | Sets the necessary permissions for the uploaded Google Drive file.                                  |
| Send an email : Video Link     | Email Send           | Sends notification email with video link        | Set Google Drive Permissions | —                             | Sends an email if the video generation is completed.                                                 |
| Sticky Note1                  | Sticky Note          | Describes form trigger                           | —                            | —                             | Triggers the workflow when a form is submitted with a prompt field.                                   |
| Sticky Note2                  | Sticky Note          | Describes API request to Sora 2                  | —                            | —                             | Processes the form's prompt by making an API call to the Sora API.                                    |
| Sticky Note3                  | Sticky Note          | Explains predictionId check                      | —                            | —                             | Checks if the prediction id is present before continuing; sends an error email if missing.            |
| Sticky Note4                  | Sticky Note          | Explains initial wait                             | —                            | —                             | Waits for 60 seconds to allow the API response to be processed.                                       |
| Sticky Note5                  | Sticky Note          | Explains wait before rechecking task             | —                            | —                             | Waits for 60 seconds before rechecking the task’s completion status.                                 |
| Sticky Note6                  | Sticky Note          | Explains API request for task status              | —                            | —                             | Sends an HTTP request to check the status of a task using its ID.                                    |
| Sticky Note7                  | Sticky Note          | Explains task status check                         | —                            | —                             | Checks the task's output status for success, processing, or failure.                                 |
| Sticky Note8                  | Sticky Note          | Explains missing prediction ID error notification | —                            | —                             | Sends an error email if the prediction id is missing in the response.                                |
| Sticky Note9                  | Sticky Note          | Explains Google Drive upload                       | —                            | —                             | Uploads the processed video to Google Drive.                                                         |
| Sticky Note10                 | Sticky Note          | Explains Google Drive permission setting          | —                            | —                             | Sets the necessary permissions for the uploaded Google Drive file.                                  |
| Sticky Note11                 | Sticky Note          | Explains task failure email notification           | —                            | —                             | Sends an email if the video generation task fails.                                                    |
| Sticky Note12                 | Sticky Note          | Explains video download step                        | —                            | —                             | Downloads the processed video from the output URL provided by the API response.                      |
| Sticky Note13                 | Sticky Note          | Explains output cleaning                            | —                            | —                             | Clean output & process result Url.                                                                    |
| Sticky Note14                 | Sticky Note          | Describes overall workflow overview                 | —                            | —                             | **CGI Ads with Sora 2: Seamless Integration and Video Generation** - detailed workflow overview.      |
| Sticky Note                   | Sticky Note          | Explains final email notification                   | —                            | —                             | Sends an email if the video generation is completed.                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure form title "CGI Ads With Sora 2"  
   - Add required field "Prompt" (text input)  
   - Connect this as the workflow start node.

2. **Create "Sora API Processor" node**  
   - Type: HTTP Request (POST)  
   - URL: `https://sora-2-openai.p.rapidapi.com/txttovideo.php`  
   - Headers:  
     - `x-rapidapi-host`: `sora-2-openai.p.rapidapi.com`  
     - `x-rapidapi-key`: `[Your RapidAPI Key]`  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "prompt": "{{ $json.Prompt }}",
       "quality": "hd",
       "aspect_ratio": "landscape"
     }
     ```  
   - Set "On Error" to continue regular output  
   - Connect output from "On form submission".  

3. **Create "Condition: Check Prediction Id" node**  
   - Type: If  
   - Condition: Check if `predictionId` exists and is non-empty in the response JSON  
   - Connect output from "Sora API Processor".  
   - True branch continues, False branch connects to error email.

4. **Create "Send Email: API Error - Task ID Missing" node**  
   - Type: Email Send  
   - SMTP credentials configured (e.g., "SMTP account 2")  
   - To: `dev@test.com`  
   - From: `it-admin@test.com`  
   - Subject: "Failed To Get Prediction ID"  
   - Body: "Hey This Just Inform To You API Doesn't Return Prediction id"  
   - Connect from False branch of Prediction Id condition.

5. **Create "Wait for API Response" node**  
   - Type: Wait  
   - Duration: 60 seconds  
   - Connect from True branch of Prediction Id condition.

6. **Create "API Request: Check Task Status" node**  
   - Type: HTTP Request (GET)  
   - URL: `https://sora-2-openai.p.rapidapi.com/result.php`  
   - Query Parameter: `predictionId={{ $json.predictionId }}`  
   - Headers same as Sora API Processor  
   - Connect from "Wait for API Response" node; also from later wait node for polling.

7. **Create "Condition: Task Output Status" node**  
   - Type: Switch  
   - Check `status` field with three cases:  
     - Equals "success"  
     - Equals "generating"  
     - Equals "failed"  
   - Connect output of API Request.

8. **Create "Clean Output" node**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const data = JSON.parse($input.first().json.result);
     return [{ json: { resultUrls: data.resultUrls } }];
     ```  
   - Connect from "success" output of Task Output Status.

9. **Create "Download Video" node**  
   - Type: HTTP Request (GET)  
   - URL: `{{ $json.resultUrls[0] }}`  
   - Connect from "Clean Output".

10. **Create "Upload File to Google Drive" node**  
    - Type: Google Drive (Upload File)  
    - Drive: My Drive (root folder)  
    - Credentials: Google Drive OAuth2 configured  
    - Connect from "Download Video".

11. **Create "Set Google Drive Permissions" node**  
    - Type: Google Drive (Share File)  
    - File ID: `{{ $json.id }}` (output from upload node)  
    - Permissions: default sharing (e.g., anyone with link can view)  
    - Credentials: Google Drive OAuth2  
    - Connect from "Upload File to Google Drive".

12. **Create "Send an email : Video Link" node**  
    - Type: Email Send  
    - SMTP credentials configured  
    - To: `user@test.com`  
    - From: `admin@test.com`  
    - Subject: (optional) e.g., "Your CGI Video is Ready"  
    - Body (HTML):  
      ```
      Hey,  
      Your video is ready.  
      Link: {{ $('Upload File to Google Drive').item.json.webViewLink }}
      ```  
    - Connect from "Set Google Drive Permissions".

13. **Create "Wait for Task to Complete" node**  
    - Type: Wait  
    - Duration: 60 seconds  
    - Connect from "generating" output of Task Output Status.

14. **Connect "Wait for Task to Complete" back to "API Request: Check Task Status"**  
    - Enables polling loop until task completes or fails.

15. **Create "Send Email: API Error - Task Failed" node**  
    - Type: Email Send  
    - SMTP credentials configured  
    - To: `dev@test.com`  
    - From: `it-admin@test.com`  
    - Subject: "Failed To Generate Image"  
    - Body:  
      ```
      Task id : {{ $json.id }}  
      Failed to generate video.
      ```  
    - Connect from "failed" output of Task Output Status.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| **CGI Ads with Sora 2: Seamless Integration and Video Generation** - This overview explains the entire workflow from form submission to video generation and sharing.                                                                     | Sticky Note14 in workflow                                                                                                             |
| API key for Sora 2 API must be obtained via RapidAPI and kept confidential; replace `"your key"` placeholders accordingly.                                                                                                               | API documentation at RapidAPI (https://rapidapi.com/)                                                                                 |
| SMTP credentials must be properly configured for sending emails; test email nodes separately to ensure delivery.                                                                                                                         | SMTP provider documentation                                                                                                          |
| Google Drive OAuth2 credentials require enabling Drive API and OAuth consent; ensure scopes permit file upload and sharing.                                                                                                              | Google Cloud Console and Drive API docs                                                                                              |
| The workflow uses polling with fixed 60-second waits; consider adjusting wait times and maximum retries based on API SLA and rate limits.                                                                                                | Workflow optimization recommendation                                                                                                |
| Error emails notify developers of failure points for prompt troubleshooting and maintenance.                                                                                                                                            | Set up appropriate monitoring and alerting based on these notifications.                                                             |

---

**Disclaimer:**  
The provided text and workflow are created exclusively via an n8n automated workflow. The content complies strictly with current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and publicly accessible.