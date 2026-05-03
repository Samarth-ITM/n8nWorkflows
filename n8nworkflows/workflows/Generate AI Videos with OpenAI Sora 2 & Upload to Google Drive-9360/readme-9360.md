Generate AI Videos with OpenAI Sora 2 & Upload to Google Drive

https://n8nworkflows.xyz/workflows/generate-ai-videos-with-openai-sora-2---upload-to-google-drive-9360


# Generate AI Videos with OpenAI Sora 2 & Upload to Google Drive

### 1. Workflow Overview

This workflow automates the generation of AI videos using the OpenAI Sora 2 API and uploads the resulting video and thumbnail to Google Drive. It targets users who want to create custom AI-generated videos based on a text prompt, optional image reference, and configurable parameters like model, video size, and duration. The workflow handles the full lifecycle from form submission, video creation, polling for completion, downloading assets, uploading to Google Drive, and presenting results or errors via forms.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Settings Preparation:** Collects user inputs via a form trigger and sets up parameters for the video creation request.
- **1.2 Video Creation Request:** Sends the video generation request to OpenAI’s Sora 2 API, conditionally including an image reference.
- **1.3 Video Status Polling:** Periodically checks the video rendering status until completion, queuing, or failure.
- **1.4 Download & Upload Assets:** Downloads the completed video and thumbnail, uploads them to Google Drive, and merges their metadata.
- **1.5 Completion or Failure Handling:** Presents completion information with download links or error messages back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Settings Preparation

**Overview:**  
This block initiates the workflow by receiving user inputs from an n8n form and prepares the necessary parameters for the video generation API calls.

**Nodes Involved:**  
- On form submission  
- Settings  
- Reference  
- Sticky Note1  
- Sticky Note4

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing user's access token, prompt, model, size, duration, and optional image reference.  
  - Configuration: Fields include YOUR_ACCESS_TOKEN (required), Prompt (textarea, required), Model (dropdown: sora-2, sora-2-pro), Size (dropdown with resolutions), Duration (dropdown: 4,8,12), and Image Reference (file upload, .jpg/.png/.webp).  
  - Input: HTTP form submission  
  - Output: User input JSON object  
  - Failure cases: Missing required fields, malformed inputs, or file upload errors.

- **Settings**  
  - Type: Set  
  - Role: Extracts and assigns relevant parameters from form input to standardized keys (prompt, model, size, seconds).  
  - Configuration: Assigns values from form JSON fields to variables for next steps.  
  - Input: Output of form submission  
  - Output: JSON with cleaned parameters  
  - Failure cases: Expression errors if form data is missing or malformed.

- **Reference**  
  - Type: Switch  
  - Role: Determines if an image reference file was provided, branching the flow accordingly.  
  - Configuration: Checks if 'Image Reference' size exists in the form data; outputs "True" if present, else "False".  
  - Input: Output of Settings node (which includes form data)  
  - Output: Two branches: True (image reference exists), False (no image reference)  
  - Failure cases: File metadata missing or corrupted leading to incorrect branching.

- **Sticky Note1 & Sticky Note4**  
  - Type: Sticky Note  
  - Role: Document the settings and workflow overview for user reference inside the editor.  
  - No computational role.

---

#### 2.2 Video Creation Request

**Overview:**  
Depending on presence of an image reference, this block sends a POST request to OpenAI’s video creation endpoint with appropriate parameters and authentication.

**Nodes Involved:**  
- Create Video A (without image reference)  
- Create Video B (with image reference)  
- Sticky Note2

**Node Details:**

- **Create Video A**  
  - Type: HTTP Request  
  - Role: Sends video generation request without image reference.  
  - Configuration: POST to `https://api.openai.com/v1/videos` with JSON body including prompt, model, size, seconds. Authentication header uses Bearer token from form input.  
  - Input: Reference node "False" branch  
  - Output: JSON containing video creation response including video id and initial status  
  - Failure cases: HTTP errors, authorization failure, invalid parameters.

- **Create Video B**  
  - Type: HTTP Request  
  - Role: Sends video generation request including image reference as multipart form-data.  
  - Configuration: POST to same endpoint but sends binary file data for "input_reference" along with other parameters in multipart form. Auth header as above.  
  - Input: Reference node "True" branch  
  - Output: Same as Create Video A  
  - Failure cases: Same as Create Video A plus potential file upload issues.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Marks this block as "Get Generated Video" in the editor.

---

#### 2.3 Video Status Polling

**Overview:**  
Polls the OpenAI API repeatedly to check the status of the video generation until it is completed, still in progress/queued, or failed. Controls the flow based on status.

**Nodes Involved:**  
- Check Status  
- Status (Switch)  
- Wait  
- Sticky Note3

**Node Details:**

- **Check Status**  
  - Type: HTTP Request  
  - Role: Queries the status of the video generation by GET request to `https://api.openai.com/v1/videos/{id}` using the video id from previous step.  
  - Configuration: Auth header with Bearer token from form input.  
  - Input: From Create Video A or B (first run) or Wait (subsequent polls)  
  - Output: JSON with current status of video ("completed", "in_progress", "queued", "failed")  
  - Failure cases: Network errors, invalid video id, auth errors.

- **Status**  
  - Type: Switch  
  - Role: Branches execution based on the returned status string.  
  - Configuration: Checks the `$json.status` field and routes to:  
    - Completed: proceed to download phase  
    - In Progress: wait and poll again  
    - Queued: wait and poll again  
    - Failed: route to error handling form  
  - Input: Check Status output  
  - Output: Four branches as above  
  - Failure cases: Unexpected status values, expression errors.

- **Wait**  
  - Type: Wait  
  - Role: Delays execution for 30 seconds before polling again.  
  - Configuration: 30 seconds wait time  
  - Input: Status node branches "In Progress" or "Queued"  
  - Output: Check Status node for repeated polling  
  - Failure cases: Delay interruptions or workflow timeouts.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Marks this block for "Upload Video & Thumbnail to Google Drive".

---

#### 2.4 Download & Upload Assets

**Overview:**  
Downloads the completed video content and thumbnail from OpenAI, uploads both files to a specified Google Drive folder, and merges metadata for presentation.

**Nodes Involved:**  
- Download  
- Thumbnail1  
- Upload Video  
- Upload Thumbnail  
- Merge  
- Aggregate  
- Sticky Note

**Node Details:**

- **Download**  
  - Type: HTTP Request  
  - Role: Downloads the generated video file content via GET request to `https://api.openai.com/v1/videos/{id}/content`.  
  - Configuration: Auth header, no additional params.  
  - Input: Status node "Completed" branch  
  - Output: Binary video file data  
  - Failure cases: Network errors, file not found, auth errors.

- **Thumbnail1**  
  - Type: HTTP Request  
  - Role: Downloads the video thumbnail image (variant=thumbnail) from OpenAI.  
  - Configuration: GET request to `https://api.openai.com/v1/videos/{id}/content?variant=thumbnail`, with auth header.  
  - Input: Status node "Completed" branch  
  - Output: Binary image file data named "thumbnail.webp"  
  - Failure cases: Same as Download node.

- **Upload Video**  
  - Type: Google Drive  
  - Role: Uploads the video binary data to a specific folder on Google Drive.  
  - Configuration: Target folder ID "1hUOUwcbOPpkfT4zCOGCju61qXVC3uenw" (named "Sora"), file named `{id}.mp4`. Uses configured Google Drive OAuth2 credentials.  
  - Input: Download node output (binary video)  
  - Output: Google Drive file metadata (including webViewLink)  
  - Failure cases: Auth errors, quota limits, folder access errors.

- **Upload Thumbnail**  
  - Type: Google Drive  
  - Role: Uploads the thumbnail image to the same Google Drive folder.  
  - Configuration: Same folder and credentials, file named `{id}.webp`, uses binary input from Thumbnail1 node.  
  - Input: Thumbnail1 output  
  - Output: Google Drive file metadata (thumbnailLink)  
  - Failure cases: Same as Upload Video.

- **Merge**  
  - Type: Merge  
  - Role: Combines the outputs of Upload Video and Upload Thumbnail into a single data stream.  
  - Configuration: Default merge mode (combine inputs into one)  
  - Input: Upload Video (main output 0), Upload Thumbnail (main output 1)  
  - Output: Combined metadata for video and thumbnail  
  - Failure cases: Mismatched data causing merge failure.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates all merged items into a single JSON array under the field "content".  
  - Configuration: Aggregate all item data destination field name = "content"  
  - Input: Merge output  
  - Output: Single JSON with "content" array containing video and thumbnail metadata  
  - Failure cases: Empty inputs or aggregation errors.

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Marks this block as "Handle Error, Thumbnail and Download Link".

---

#### 2.5 Completion or Failure Handling

**Overview:**  
Finalizes the workflow by presenting the user with either a success form containing download links and embedded thumbnail or an error form with the failure message.

**Nodes Involved:**  
- Completed  
- Failed

**Node Details:**

- **Completed**  
  - Type: Form  
  - Role: Presents a completion message with clickable thumbnail image linking to the video on Google Drive, plus a download link.  
  - Configuration: Uses the aggregated "content" JSON to extract `webViewLink` for the video and `thumbnailLink` for the thumbnail image.  
  - Input: Aggregate node output  
  - Output: HTTP response form completion  
  - Failure cases: Missing metadata, broken links, form rendering errors.

- **Failed**  
  - Type: Form  
  - Role: Presents an error message to the user with error code and message from the failed API response.  
  - Configuration: Extracts `$json.error.code` and `$json.error.message` for display.  
  - Input: Status node "Failed" branch  
  - Output: HTTP error form response  
  - Failure cases: Missing error fields or malformed error response.

---

### 3. Summary Table

| Node Name         | Node Type                 | Functional Role                          | Input Node(s)                      | Output Node(s)                       | Sticky Note                                             |
|-------------------|---------------------------|----------------------------------------|----------------------------------|------------------------------------|---------------------------------------------------------|
| On form submission | Form Trigger              | Entry point, collects user inputs      | -                                | Settings                           |                                                         |
| Settings          | Set                       | Prepares and normalizes parameters     | On form submission               | Reference                         |                                                         |
| Reference         | Switch                    | Branches based on presence of image ref| Settings                        | Create Video A, Create Video B    |                                                         |
| Create Video A    | HTTP Request              | Sends video creation request without image | Reference (False branch)       | Check Status                     |                                                         |
| Create Video B    | HTTP Request              | Sends video creation request with image   | Reference (True branch)        | Check Status                     |                                                         |
| Check Status      | HTTP Request              | Polls video generation status           | Create Video A/B, Wait          | Status                          |                                                         |
| Status            | Switch                    | Routes flow based on video status       | Check Status                   | Download, Thumbnail1, Wait, Failed |                                                         |
| Wait              | Wait                      | Delays for 30 seconds before polling again | Status (In Progress/Queued)   | Check Status                   |                                                         |
| Download          | HTTP Request              | Downloads completed video content       | Status (Completed)              | Upload Video                    |                                                         |
| Thumbnail1        | HTTP Request              | Downloads video thumbnail image          | Status (Completed)              | Upload Thumbnail               |                                                         |
| Upload Video      | Google Drive              | Uploads video file to Google Drive       | Download                       | Merge                          |                                                         |
| Upload Thumbnail  | Google Drive              | Uploads thumbnail image to Google Drive  | Thumbnail1                    | Merge                          |                                                         |
| Merge             | Merge                     | Combines video and thumbnail metadata   | Upload Video, Upload Thumbnail  | Aggregate                      |                                                         |
| Aggregate         | Aggregate                 | Aggregates merged metadata into array   | Merge                         | Completed                     |                                                         |
| Completed         | Form                      | Shows success message with download links| Aggregate                    | -                              |                                                         |
| Failed            | Form                      | Shows error message on failure           | Status (Failed)                | -                              |                                                         |
| Sticky Note1      | Sticky Note               | Workflow overview and settings info     | -                            | -                              | ## Sora 1 & 4 (Settings info and workflow overview)      |
| Sticky Note2      | Sticky Note               | Marks video generation block             | -                            | -                              | ### Get Generated Video                                  |
| Sticky Note3      | Sticky Note               | Marks upload block                       | -                            | -                              | ### Upload Video & Thumbnail to Google Drive            |
| Sticky Note4      | Sticky Note               | Shows detailed settings explanation      | -                            | -                              | ### Settings: Model, Size, Duration, Reference details  |
| Sticky Note       | Sticky Note               | Marks error handling and download link block | -                          | -                              | ### Handle Error, Thumbnail and Download Link           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**
   - Node Type: Form Trigger  
   - Name: On form submission  
   - Configure webhook with title "OpenAI Sora"  
   - Add form fields:  
     - YOUR_ACCESS_TOKEN (text, required)  
     - Prompt (textarea, required)  
     - Model (dropdown: sora-2, sora-2-pro, required)  
     - Size (dropdown: 1280x720, 720x1280, 1024x1792, 1792x1024, required)  
     - Duration (dropdown: 4, 8, 12, required)  
     - Image Reference (file upload, accept .jpg, .png, .webp, optional)  

2. **Add a Set Node:**
   - Name: Settings  
   - Assign the following variables from form input:  
     - prompt = {{$json.Prompt}}  
     - model = {{$json.Model}}  
     - size = {{$json.Size}}  
     - seconds = {{$json.Duration}}  
   - Include other fields as needed.

3. **Add a Switch Node:**
   - Name: Reference  
   - Condition: Check if 'Image Reference' file size exists ({{$json['Image Reference'].size}})  
   - Output branches: True (file present), False (file absent)  

4. **Add HTTP Request Node for Video Creation without Image:**
   - Name: Create Video A  
   - Method: POST  
   - URL: https://api.openai.com/v1/videos  
   - Headers: Authorization: Bearer {{$json.YOUR_ACCESS_TOKEN}}  
   - Body (application/json): prompt, model, size, seconds (from Settings)  

5. **Add HTTP Request Node for Video Creation with Image:**
   - Name: Create Video B  
   - Method: POST  
   - URL: https://api.openai.com/v1/videos  
   - Headers: Authorization: Bearer {{$json.YOUR_ACCESS_TOKEN}}  
   - Body type: multipart-form-data  
   - Parameters: prompt, model, size, seconds from Settings,  
   - Binary file: input_reference from uploaded image file  

6. **Connect Reference Node Outputs:**
   - False → Create Video A  
   - True → Create Video B  

7. **Add HTTP Request Node to Check Video Status:**
   - Name: Check Status  
   - Method: GET  
   - URL: https://api.openai.com/v1/videos/{{$json.id}}  
   - Headers: Authorization: Bearer {{$json.YOUR_ACCESS_TOKEN}}  

8. **Add Switch Node to Handle Video Status:**
   - Name: Status  
   - Conditions on $json.status:  
     - Completed → Download + Thumbnail1  
     - In Progress → Wait  
     - Queued → Wait  
     - Failed → Failed form response  

9. **Add Wait Node:**
   - Name: Wait  
   - Duration: 30 seconds  
   - Connect Wait output back to Check Status for polling loop  

10. **Add HTTP Request Nodes to Download Video and Thumbnail:**
    - Download: GET https://api.openai.com/v1/videos/{{$json.id}}/content  
      - Headers: Authorization: Bearer {{$json.YOUR_ACCESS_TOKEN}}  
      - Output: Video binary data  
    - Thumbnail1: GET https://api.openai.com/v1/videos/{{$json.id}}/content?variant=thumbnail  
      - Headers: Authorization: Bearer {{$json.YOUR_ACCESS_TOKEN}}  
      - Response format: file (thumbnail.webp)  

11. **Add Google Drive Upload Nodes:**
    - Upload Video  
      - File name: {{$json.id}}.mp4  
      - Folder ID: "1hUOUwcbOPpkfT4zCOGCju61qXVC3uenw" (or your own)  
      - Credentials: Google Drive OAuth2  
      - Input: Video binary data from Download node  
    - Upload Thumbnail  
      - File name: {{$json.id}}.webp  
      - Folder ID: same as above  
      - Credentials: Google Drive OAuth2  
      - Input: Binary file from Thumbnail1 node  

12. **Add Merge Node:**
    - Name: Merge  
    - Combine outputs from Upload Video (main output 0) and Upload Thumbnail (main output 1)  

13. **Add Aggregate Node:**
    - Name: Aggregate  
    - Aggregate all items into a single JSON field named "content"  

14. **Add Form Nodes for Final Output:**
    - Completed  
      - Completion Title: "Generation Completed"  
      - Completion Message: HTML embedding thumbnail and download link using `content[0].webViewLink` and `content[1].thumbnailLink`  
    - Failed  
      - Completion Title: "Error"  
      - Completion Message: Display error code and message from API error response  

15. **Connect all nodes by logical flow:**
    - On form submission → Settings → Reference  
    - Reference False → Create Video A → Check Status  
    - Reference True → Create Video B → Check Status  
    - Check Status → Status → according to status branch:  
      - Completed → Download + Thumbnail1 → Upload Video + Upload Thumbnail → Merge → Aggregate → Completed  
      - In Progress / Queued → Wait → Check Status (loop)  
      - Failed → Failed  

16. **Set Credentials:**
    - Google Drive OAuth2 account with drive access to target folder  
    - OpenAI API token provided by user in form input (passed as Bearer token)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sora 2 video generation pricing details and video size options: Model sora-2 ($0.1/sec), sora-2-pro ($0.3/sec or $0.5/sec) with different resolutions and durations. Reference images must match target resolution and be in jpg/png/webp format.                                                                                                                               | Inside Sticky Note4 in the workflow; important for cost and quality considerations.                                                                              |
| Workflow shows an elegant polling mechanism using Wait and Status nodes to avoid API quota issues and handle asynchronous video rendering.                                                                                                                                                                                                                                | Best practice for handling async video generation APIs.                                                                                                        |
| Google Drive folder ID "1hUOUwcbOPpkfT4zCOGCju61qXVC3uenw" is a placeholder; replace with your own folder ID for uploads.                                                                                                                                                                                                                                                 | Credential and folder setup requirement.                                                                                                                       |
| The workflow uses n8n Form Trigger and Form nodes to interact with users via UI for input and output, enabling standalone operation without external frontend.                                                                                                                                                                                                              | Useful for headless or low-code automation.                                                                                                                    |
| OpenAI Sora 2 API requires valid Bearer token for authorization — user supplies this at form submission time ensuring security and flexibility.                                                                                                                                                                                                                          | Security note: token must be kept confidential.                                                                                                                |
| For troubleshooting, monitor HTTP Request nodes for response errors including 401 Unauthorized, 429 Rate Limits, and 500+ Server errors.                                                                                                                                                                                                                                 | Common API failure modes.                                                                                                                                        |

---

**Disclaimer:** The provided text is exclusively generated from an n8n automated workflow integration. It complies with content policies and contains no illegal or offensive material. All data handled is legal and publicly accessible.