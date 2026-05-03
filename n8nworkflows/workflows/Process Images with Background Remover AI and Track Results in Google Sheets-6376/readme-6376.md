Process Images with Background Remover AI and Track Results in Google Sheets

https://n8nworkflows.xyz/workflows/process-images-with-background-remover-ai-and-track-results-in-google-sheets-6376


# Process Images with Background Remover AI and Track Results in Google Sheets

### 1. Workflow Overview

This workflow automates the process of removing backgrounds from images submitted via a web form using an AI-powered background remover service. It manages the entire lifecycle from image upload, processing via external APIs, temporary storage of the processed image, to logging the results in a Google Sheets document for tracking. It also handles success and failure cases distinctly to maintain accurate status records.

**Target Use Cases:**  
- Automating background removal for images submitted by users through a form.  
- Tracking processed images and their statuses in a centralized Google Sheets file.  
- Handling file conversion and temporary uploads seamlessly between services.

**Logical Blocks:**

- **1.1 Input Reception:** Captures image uploads triggered by form submission.  
- **1.2 Background Removal Processing:** Sends the image to an AI service for background removal and converts the response into a binary file.  
- **1.3 Temporary File Upload:** Uploads the processed image to a temporary file hosting service.  
- **1.4 Conditional Branching and Logging:** Checks processing success and logs the result (success or failure) into Google Sheets.  
- **1.5 Workflow Synchronization:** Waits briefly to ensure the processing steps complete before updating Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Triggers the workflow upon a user submitting a form containing an image file for background removal.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Configuration: Triggered by a form titled "Background Removal AI" that requires a single image file upload. Upon submission, the user is redirected to a Google Sheets URL for tracking.  
    - Key Variables: Captures the uploaded image file under the field "Image".  
    - Input: External user form submission.  
    - Output: Passes the uploaded image file as binary data to the next node.  
    - Failure Modes: Form submission failures or missing required file field.  
    - Notes: Redirects users after submission to a Google Sheets URL for transparency.

#### 1.2 Background Removal Processing

- **Overview:**  
  Sends the uploaded image to an external background removal AI service via an HTTP POST request, then converts the returned base64 image data into a binary file.

- **Nodes Involved:**  
  - HTTP Request  
  - Convert to File

- **Node Details:**  
  - **HTTP Request**  
    - Type: HTTP Request  
    - Configuration: POST request to the background removal AI API endpoint (`https://background-remover-ai3.p.rapidapi.com/index.php`), sending the uploaded image as multipart form data. Requires RapidAPI host and key headers.  
    - Key Expressions: Uses the binary input from "On form submission" node as the file parameter.  
    - Input: Binary image from form submission.  
    - Output: JSON response including base64 encoded image and status field.  
    - Failure Modes: API authentication errors, network timeouts, invalid image formats, or API service failures.  
    - Credential Requirements: RapidAPI key for background remover API.  

  - **Convert to File**  
    - Type: Convert to File  
    - Configuration: Converts the base64-encoded image in the JSON response field `base64` into a binary file for further processing.  
    - Input: JSON containing base64 image string from the HTTP Request node.  
    - Output: Binary file data representing the processed image.  
    - Failure Modes: Base64 decoding errors, empty or malformed response data.

#### 1.3 Temporary File Upload

- **Overview:**  
  Uploads the processed binary image to a temporary file hosting service via an HTTP POST request.

- **Nodes Involved:**  
  - File Upload Api

- **Node Details:**  
  - **File Upload Api**  
    - Type: HTTP Request  
    - Configuration: POST request to `https://temp-file-upload.p.rapidapi.com/temp-file.php` with the processed image binary as multipart form data. Requires RapidAPI host and key headers.  
    - Input: Binary file from "Convert to File" node.  
    - Output: JSON response including the URL of the temporarily hosted file and status.  
    - Failure Modes: API key errors, network timeouts, upload failures.  
    - Credential Requirements: RapidAPI key for temporary file upload API.

#### 1.4 Conditional Branching and Logging

- **Overview:**  
  Evaluates the success of the background removal process and branches accordingly to update Google Sheets either with success details or failure logs.

- **Nodes Involved:**  
  - If  
  - Google Sheets  
  - Google Sheets1

- **Node Details:**  
  - **If**  
    - Type: Conditional Node  
    - Configuration: Checks if the `status` field in the JSON response equals "success".  
    - Input: JSON from "File Upload Api".  
    - Output:  
      - True branch: Continues to "Wait" node, then updates Google Sheets with success.  
      - False branch: Updates Google Sheets with failure log.  
    - Failure Modes: Missing or malformed status field causing incorrect branching.

  - **Google Sheets** (Success Logging)  
    - Type: Google Sheets Node  
    - Configuration: Appends or updates a row in Google Sheets with columns: Link (URL of processed image), status, expiration timestamp (current time + 60 minutes), and original image filename. Uses service account authentication.  
    - Input: JSON from "Wait" node (which follows "If" true branch).  
    - Output: None (logging only).  
    - Failure Modes: Authentication failures, API rate limits, incorrect spreadsheet ID or permissions.

  - **Google Sheets1** (Failure Logging)  
    - Type: Google Sheets Node  
    - Configuration: Appends or updates a row in Google Sheets with failure details: Link as "Not found", status as "Failed", current timestamp, and image filename. Uses service account authentication.  
    - Input: JSON from "If" false branch.  
    - Output: None (logging only).  
    - Failure Modes: Same as above.

#### 1.5 Workflow Synchronization

- **Overview:**  
  Introduces a brief wait to ensure all processing and uploads finalize before updating the Google Sheets document.

- **Nodes Involved:**  
  - Wait

- **Node Details:**  
  - **Wait**  
    - Type: Wait Node  
    - Configuration: Default wait with no explicit time configured (likely default short pause).  
    - Input: True branch from "If".  
    - Output: Connects to Google Sheets node for success logging.  
    - Failure Modes: Minimal, but could introduce latency or synchronization issues if duration is insufficient.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                                   | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                   |
|---------------------|-----------------------|-------------------------------------------------|-------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger          | Captures image upload via form submission       | External (form submit)  | HTTP Request             | **On form submission**: Triggers the workflow on form submission collecting image file.                                       |
| HTTP Request        | HTTP Request          | Sends image to background removal AI API        | On form submission      | Convert to File          | **HTTP Request**: Sends image to AI service and receives processed image or failure status.                                   |
| Convert to File     | Convert to File       | Converts base64 image from API to binary file   | HTTP Request            | File Upload Api          | **Convert to File**: Converts AI service base64 response to binary file.                                                      |
| File Upload Api     | HTTP Request          | Uploads processed image to temporary storage     | Convert to File         | If                       | **File Upload API**: Uploads processed image to temporary file storage service.                                               |
| If                  | Conditional           | Checks processing status, branches workflow      | File Upload Api         | Wait (true), Google Sheets1 (false) | **If**: Evaluates status; updates Google Sheets on success or logs failure.                                                   |
| Wait                | Wait                  | Pauses workflow to ensure processing completion | If (true branch)        | Google Sheets            | **Wait**: Pauses briefly to ensure image processing completion before continuing.                                            |
| Google Sheets       | Google Sheets         | Logs success details in Google Sheets            | Wait                    | None                     | **Google Sheets**: Appends or updates processed image data (link, status, expiration).                                        |
| Google Sheets1      | Google Sheets         | Logs failure details in Google Sheets            | If (false branch)       | None                     | **Google Sheets1**: Logs failure (not found link, failed status, timestamp) in Google Sheets.                                 |
| Sticky Note         | Sticky Note           | Documentation and explanation                     | None                    | None                     | See sticky notes content for detailed node explanations and workflow overview.                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Type: Form Trigger  
   - Set form title to "Background Removal AI".  
   - Add one required file field labeled "Image" allowing a single file upload.  
   - Configure response mode to "lastNode" with redirect URL set to the Google Sheets tracking URL (e.g., `https://docs.google.com/spreadsheets/d/...`).  

2. **Add an HTTP Request node (Background Removal API):**  
   - Connect from Form Trigger node.  
   - Method: POST  
   - URL: `https://background-remover-ai3.p.rapidapi.com/index.php`  
   - Content-Type: multipart/form-data  
   - Body Parameters: Add a parameter named "file" of type formBinaryData, mapped to the "Image" binary data from form trigger.  
   - Headers:  
     - `x-rapidapi-host`: `background-remover-ai3.p.rapidapi.com`  
     - `x-rapidapi-key`: your RapidAPI key for this service.  
   - Ensure credentials for RapidAPI key are set.

3. **Add Convert to File node:**  
   - Connect from HTTP Request node.  
   - Operation: toBinary  
   - Source Property: `base64` (the property in the HTTP response containing base64 image).  

4. **Add HTTP Request node (Temporary File Upload):**  
   - Connect from Convert to File node.  
   - Method: POST  
   - URL: `https://temp-file-upload.p.rapidapi.com/temp-file.php`  
   - Content-Type: multipart/form-data  
   - Body Parameters: Add a parameter named "file" of type formBinaryData, mapped to the binary data from Convert to File node.  
   - Headers:  
     - `x-rapidapi-host`: `temp-file-upload.p.rapidapi.com`  
     - `x-rapidapi-key`: your RapidAPI key for this service.  
   - Ensure credentials for RapidAPI key are set.

5. **Add If node (Conditional check):**  
   - Connect from Temporary File Upload node.  
   - Condition: Check if `{{$json.status}}` equals `"success"`.  
   - True output: Connect to Wait node.  
   - False output: Connect to Google Sheets failure logging node.

6. **Add Wait node:**  
   - Connect from If node's true output.  
   - Use default settings or configure a brief wait if needed.

7. **Add Google Sheets node (Success logging):**  
   - Connect from Wait node.  
   - Operation: Append or Update  
   - Authentication: Service Account (configure Google Sheets credentials).  
   - Document ID: Set to your Google Sheets file ID.  
   - Sheet name: Use the sheet with `gid=0` or name "Sheet1".  
   - Columns mapping:  
     - Link: `={{ $json.data.url }}` (URL from temporary file upload response)  
     - status: `={{ $json.status }}`  
     - Expire at: `={{ new Date(new Date($now).setMinutes(new Date($now).getMinutes() + 60)).toISOString() }}` (one hour from now)  
     - Image name: `={{ $('On form submission').item.json.Image.filename }}` (original file name)

8. **Add Google Sheets node (Failure logging):**  
   - Connect from If node's false output.  
   - Same Google Sheets document and sheet as above.  
   - Columns mapping:  
     - Link: `"Not found"`  
     - status: `"Failed"`  
     - Expire at: `={{ $now }}` (current timestamp)  
     - Image name: `={{ $('On form submission').item.json.Image.filename }}`  

9. **Set all credentials:**  
   - Configure RapidAPI credentials for both APIs (background removal and temp file upload).  
   - Configure Google Sheets credentials with access to the target spreadsheet (preferably via service account).

10. **Test workflow:**  
    - Submit an image through the form.  
    - Verify the processed image URL appears in Google Sheets on success or failure is logged appropriately.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow demonstrates a practical integration of AI-powered image processing with Google Sheets for tracking processed results.                                 | Workflow overview                                                                                   |
| Uses RapidAPI-hosted AI background removal and temporary file upload services requiring valid API keys from RapidAPI.                                               | API integration                                                                                     |
| The Google Sheets nodes use Service Account authentication, which requires setting up a Google Cloud project and sharing the spreadsheet with the service account.  | Google Sheets API setup                                                                            |
| Redirect URL after form submission points to the Google Sheets document for user transparency on image processing status.                                           | Form Trigger node configuration                                                                    |
| Background removal status is strictly checked against the string "success" to determine workflow branching.                                                        | If node logic                                                                                       |
| Expiration time for processed images is set to one hour from current processing time to manage temporary file availability.                                         | Google Sheets success logging                                                                       |
| Handling failure by logging "Not found" and "Failed" status ensures visibility of unsuccessful attempts.                                                           | Google Sheets failure logging                                                                       |

---

*Disclaimer:* The provided text is exclusively derived from an automated workflow created with n8n, adhering strictly to all content policies and handling only legal and publicly available data.