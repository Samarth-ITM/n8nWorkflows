Generate 🤖🧠 AI-powered video 🎥 from image and upload it on Google Drive

https://n8nworkflows.xyz/workflows/generate------ai-powered-video----from-image-and-upload-it-on-google-drive-3088


# Generate 🤖🧠 AI-powered video 🎥 from image and upload it on Google Drive

### 1. Workflow Overview

This workflow automates the generation of AI-powered videos from user-submitted images and uploads the resulting videos to Google Drive. It is designed for users who want to create videos based on textual descriptions and images without manual video editing, leveraging an external AI video generation API.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input via a form trigger, including video description, duration, aspect ratio, and image URL.
- **1.2 Data Preparation:** Structures the form data for API consumption.
- **1.3 Video Generation Request:** Sends a request to an AI video generation API to start video creation.
- **1.4 Video Generation Monitoring:** Waits and polls the API to check if the video generation is complete.
- **1.5 Video Retrieval and Upload:** Once ready, retrieves the video URL, downloads the video file, and uploads it to a specified Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by capturing user input through a form submission. It collects all necessary parameters for video generation.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point that listens for form submissions containing video parameters.  
    - *Configuration:*  
      - Pre-configured form fields:  
        - Description (text)  
        - Duration (seconds, numeric)  
        - Aspect Ratio (select: 16:9, 9:16, 1:1)  
        - Image URL (text)  
      - Webhook ID set for external form integration.  
    - *Input:* External HTTP POST from form submission  
    - *Output:* Form data passed downstream  
    - *Edge Cases:*  
      - Missing or malformed form fields may cause downstream errors.  
      - Invalid image URLs or unsupported aspect ratios require validation.  
    - *Version Requirements:* n8n version supporting Form Trigger node v2.2 or higher.

---

#### 2.2 Data Preparation

- **Overview:**  
  Organizes and formats the raw form input into a structured format suitable for the API request.

- **Nodes Involved:**  
  - Set data

- **Node Details:**

  - **Set data**  
    - *Type:* Set  
    - *Role:* Maps and formats form input fields into variables for the video creation API.  
    - *Configuration:*  
      - Extracts fields from form submission: description, duration, aspect ratio, image URL.  
      - May rename or reformat fields as required by the API.  
    - *Input:* Data from "On form submission" node  
    - *Output:* Structured JSON object for API consumption  
    - *Edge Cases:*  
      - Missing fields or unexpected data types may cause API request failures.  
      - Ensure duration is numeric and aspect ratio matches allowed values.  
    - *Version Requirements:* n8n Set node v3.4 or higher.

---

#### 2.3 Video Generation Request

- **Overview:**  
  Sends a POST request to the AI video generation API to initiate video creation using the prepared data.

- **Nodes Involved:**  
  - Create Video

- **Node Details:**

  - **Create Video**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to the video generation API endpoint.  
    - *Configuration:*  
      - HTTP Method: POST  
      - URL: API endpoint for video creation (configured in node)  
      - Headers:  
        - Authorization: `Key YOURAPIKEY` (replace with actual API key)  
      - Body: JSON including description, image URL, duration, aspect ratio.  
      - Authentication: HTTP Header Authentication with API key.  
    - *Input:* Structured data from "Set data" node  
    - *Output:* API response containing video generation job ID or status  
    - *Edge Cases:*  
      - API key invalid or expired → authentication errors.  
      - Network timeouts or API rate limits.  
      - Invalid parameters causing API rejection.  
    - *Version Requirements:* HTTP Request node v4.2 or higher.

---

#### 2.4 Video Generation Monitoring

- **Overview:**  
  Waits for a fixed interval, then polls the API to check if the video generation is complete. Loops until completion.

- **Nodes Involved:**  
  - Wait 60 sec.  
  - Get status  
  - Completed?

- **Node Details:**

  - **Wait 60 sec.**  
    - *Type:* Wait  
    - *Role:* Pauses workflow execution for 60 seconds to allow video generation.  
    - *Configuration:* Fixed wait time of 60 seconds.  
    - *Input:* Triggered after video creation request.  
    - *Output:* Triggers status check after wait.  
    - *Edge Cases:*  
      - Excessive waiting if video generation is slow.  
      - Workflow timeout if API takes too long.  
    - *Version Requirements:* Wait node v1.1 or higher.

  - **Get status**  
    - *Type:* HTTP Request  
    - *Role:* Sends GET request to API to retrieve current status of video generation job.  
    - *Configuration:*  
      - HTTP Method: GET  
      - URL: API endpoint with job ID parameter from previous response.  
      - Headers: Authorization with API key.  
    - *Input:* Triggered after wait node.  
    - *Output:* API response with current job status.  
    - *Edge Cases:*  
      - API errors or invalid job ID.  
      - Network issues causing failed status retrieval.  
    - *Version Requirements:* HTTP Request node v4.2 or higher.

  - **Completed?**  
    - *Type:* If  
    - *Role:* Checks if the video generation status indicates completion.  
    - *Configuration:*  
      - Condition: Checks if status field equals "completed" (or equivalent).  
      - True branch: Proceed to video retrieval.  
      - False branch: Loop back to "Wait 60 sec." node.  
    - *Input:* Status response from "Get status" node.  
    - *Output:* Conditional routing based on completion status.  
    - *Edge Cases:*  
      - Status field missing or unexpected values.  
      - Infinite loop if status never reaches completion.  
    - *Version Requirements:* If node v2.2 or higher.

---

#### 2.5 Video Retrieval and Upload

- **Overview:**  
  After confirming video generation completion, retrieves the video URL, downloads the video file, and uploads it to Google Drive.

- **Nodes Involved:**  
  - Get Url Video  
  - Get File Video  
  - Upload Video

- **Node Details:**

  - **Get Url Video**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves the direct URL of the generated video from the API.  
    - *Configuration:*  
      - HTTP Method: GET  
      - URL: API endpoint to fetch video URL using job ID.  
      - Headers: Authorization with API key.  
    - *Input:* Triggered from "Completed?" node true branch.  
    - *Output:* JSON containing video URL.  
    - *Edge Cases:*  
      - API errors or missing URL in response.  
      - Network issues.  
    - *Version Requirements:* HTTP Request node v4.2 or higher.

  - **Get File Video**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the video file from the retrieved URL.  
    - *Configuration:*  
      - HTTP Method: GET  
      - URL: Video URL obtained from previous node.  
      - Response Format: File (binary data).  
    - *Input:* Video URL from "Get Url Video" node.  
    - *Output:* Binary video file data.  
    - *Edge Cases:*  
      - Download failures due to invalid URL or network issues.  
      - Large file size causing timeout or memory issues.  
    - *Version Requirements:* HTTP Request node v4.2 or higher.

  - **Upload Video**  
    - *Type:* Google Drive  
    - *Role:* Uploads the downloaded video file to a specified Google Drive folder.  
    - *Configuration:*  
      - Authentication: Google Drive OAuth2 credentials configured in n8n.  
      - Operation: Upload file  
      - Folder ID: Target Google Drive folder specified.  
      - File Data: Binary data from "Get File Video" node.  
      - File Name: Can be dynamically set (e.g., based on description or timestamp).  
    - *Input:* Binary video file from "Get File Video" node.  
    - *Output:* Confirmation of upload with file metadata.  
    - *Edge Cases:*  
      - Authentication errors if credentials expire or are invalid.  
      - Insufficient permissions on target folder.  
      - File size limits or quota exceeded on Google Drive.  
    - *Version Requirements:* Google Drive node v3 or higher.

---

### 3. Summary Table

| Node Name         | Node Type         | Functional Role                     | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                  |
|-------------------|-------------------|-----------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------|
| On form submission | Form Trigger      | Entry point for user input form   | —                     | Set data             |                                                                                              |
| Set data          | Set               | Formats form data for API request | On form submission    | Create Video          |                                                                                              |
| Create Video      | HTTP Request      | Sends video generation request    | Set data              | Wait 60 sec.          | Requires API key in Authorization header                                                     |
| Wait 60 sec.      | Wait              | Pauses workflow for 60 seconds    | Create Video, Completed? (false branch) | Get status           |                                                                                              |
| Get status        | HTTP Request      | Polls API for video generation status | Wait 60 sec.          | Completed?            |                                                                                              |
| Completed?        | If                | Checks if video generation is done | Get status            | Get Url Video (true), Wait 60 sec. (false) |                                                                                              |
| Get Url Video     | HTTP Request      | Retrieves video URL from API      | Completed? (true)      | Get File Video        |                                                                                              |
| Get File Video    | HTTP Request      | Downloads video file              | Get Url Video          | Upload Video          |                                                                                              |
| Upload Video      | Google Drive      | Uploads video to Google Drive     | Get File Video         | —                    | Requires Google Drive OAuth2 credentials and target folder ID                                |
| Sticky Note3      | Sticky Note       | (Empty content)                   | —                     | —                    |                                                                                              |
| Sticky Note5      | Sticky Note       | (Empty content)                   | —                     | —                    |                                                                                              |
| Sticky Note6      | Sticky Note       | (Empty content)                   | —                     | —                    |                                                                                              |
| Sticky Note7      | Sticky Note       | (Empty content)                   | —                     | —                    |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a **Form Trigger** node named "On form submission".  
   - Configure form fields:  
     - Description (text)  
     - Duration (number, seconds)  
     - Aspect Ratio (dropdown with options: 16:9, 9:16, 1:1)  
     - Image URL (text)  
   - Save and note the webhook URL for form integration.

2. **Add a Set Node**  
   - Add a **Set** node named "Set data".  
   - Map incoming form fields to variables expected by the API:  
     - description → description  
     - duration → duration  
     - aspectRatio → aspect_ratio (or as required)  
     - imageUrl → image_url  
   - Connect "On form submission" output to "Set data" input.

3. **Add HTTP Request Node for Video Creation**  
   - Add an **HTTP Request** node named "Create Video".  
   - Set method to POST.  
   - Enter the API endpoint URL for video creation.  
   - Under Headers, add:  
     - Key: Authorization  
     - Value: `Key YOURAPIKEY` (replace with your actual API key)  
   - Set body to JSON with fields from "Set data" node.  
   - Connect "Set data" output to "Create Video" input.

4. **Add Wait Node**  
   - Add a **Wait** node named "Wait 60 sec."  
   - Configure to wait for 60 seconds.  
   - Connect "Create Video" output to "Wait 60 sec." input.

5. **Add HTTP Request Node for Status Check**  
   - Add an **HTTP Request** node named "Get status".  
   - Set method to GET.  
   - Configure URL to query video generation status using job ID from "Create Video" response.  
   - Add Authorization header with API key.  
   - Connect "Wait 60 sec." output to "Get status" input.

6. **Add If Node to Check Completion**  
   - Add an **If** node named "Completed?".  
   - Configure condition to check if status field equals "completed".  
   - Connect "Get status" output to "Completed?" input.  
   - Connect "Completed?" true output to next block, false output back to "Wait 60 sec." node to loop.

7. **Add HTTP Request Node to Get Video URL**  
   - Add an **HTTP Request** node named "Get Url Video".  
   - Set method to GET.  
   - Configure URL to retrieve video URL using job ID.  
   - Add Authorization header with API key.  
   - Connect "Completed?" true output to "Get Url Video" input.

8. **Add HTTP Request Node to Download Video File**  
   - Add an **HTTP Request** node named "Get File Video".  
   - Set method to GET.  
   - Use the video URL from "Get Url Video" node as the request URL.  
   - Set response format to binary to download the file.  
   - Connect "Get Url Video" output to "Get File Video" input.

9. **Add Google Drive Node to Upload Video**  
   - Add a **Google Drive** node named "Upload Video".  
   - Configure credentials with Google Drive OAuth2 in n8n.  
   - Set operation to "Upload File".  
   - Specify the target folder ID in Google Drive.  
   - Use binary data from "Get File Video" node as file content.  
   - Optionally set file name dynamically (e.g., based on description or timestamp).  
   - Connect "Get File Video" output to "Upload Video" input.

10. **Test the Workflow**  
    - Activate the workflow.  
    - Submit the form with valid inputs.  
    - Monitor execution to ensure video is generated and uploaded correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow requires an API key for the AI video generation service; ensure it is kept secure.     | API key setup instructions in "Create Video" node configuration.                                  |
| Google Drive credentials must be configured in n8n with appropriate permissions for file upload.    | Google Drive OAuth2 credential setup in n8n.                                                      |
| The workflow includes a 60-second wait loop to poll video generation status; adjust timing as needed.| Consider API rate limits and video generation time variability.                                   |
| Form Trigger node fields can be customized to add more aspect ratios or other parameters.            | Modify "On form submission" node form fields accordingly.                                         |
| Example video output is available here: [Watch the resulting video](https://n3wstorage.b-cdn.net/n3witalia/imagetovideo_output.mp4) | Demonstrates the final video generated by this workflow.                                          |
| For advanced customization, consider adding notification nodes (e.g., email or Slack) after upload. | Enhances user feedback on video generation completion.                                            |
| The workflow uses standard n8n nodes compatible with versions supporting HTTP Request v4.2 and Google Drive v3. | Ensure n8n instance is updated accordingly for best compatibility.                                |

---

This document provides a detailed, structured reference to understand, reproduce, and modify the AI-powered video generation workflow in n8n, including potential failure points and integration considerations.