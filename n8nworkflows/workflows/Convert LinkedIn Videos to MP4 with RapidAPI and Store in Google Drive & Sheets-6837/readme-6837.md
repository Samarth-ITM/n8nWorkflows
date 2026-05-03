Convert LinkedIn Videos to MP4 with RapidAPI and Store in Google Drive & Sheets

https://n8nworkflows.xyz/workflows/convert-linkedin-videos-to-mp4-with-rapidapi-and-store-in-google-drive---sheets-6837


# Convert LinkedIn Videos to MP4 with RapidAPI and Store in Google Drive & Sheets

---

### 1. Workflow Overview

This workflow automates the process of converting LinkedIn video URLs into downloadable MP4 files using a RapidAPI service, and then storing the resulting videos in Google Drive while logging successes and failures into Google Sheets. It is designed primarily for users needing to archive or manage LinkedIn videos efficiently.

**Logical Blocks:**

- **1.1 Input Reception:** Captures the LinkedIn video URL via a web form.
- **1.2 LinkedIn Video Retrieval:** Calls the RapidAPI LinkedIn Video Downloader to fetch downloadable media links.
- **1.3 Response Validation:** Checks for errors in the API response.
- **1.4 Video Downloading:** Downloads the MP4 video binary from the provided media URL.
- **1.5 Google Drive Upload:** Uploads the downloaded MP4 file to Google Drive.
- **1.6 Permission Setting:** Makes the uploaded file publicly accessible.
- **1.7 Failure Handling & Logging:** Waits before logging failed attempts to Google Sheets to prevent rapid consecutive writes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user submits a form containing a LinkedIn video URL.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Initiates workflow via user form input.  
    - Configuration:  
      - Form title: "Linkedin to MP4"  
      - Single required field labeled "URL" with placeholder "https://linkedin.com/abcdefg"  
      - Form description: "Linkedin to MP4 Converter"  
    - Inputs: External form submission (HTTP request webhook)  
    - Outputs: JSON containing `{ URL: <user input> }`  
    - Edge Cases:  
      - Missing or malformed URL input will prevent continuation as field is required.  
      - Webhook must be publicly accessible.  
    - Version: 2.2  

#### 2.2 LinkedIn Video Retrieval

- **Overview:**  
  Sends a POST request to the RapidAPI LinkedIn Video Downloader API to retrieve media download links for the given URL.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Fetch media download links from RapidAPI.  
    - Configuration:  
      - Method: POST  
      - URL: https://linkedin-video-downloader3.p.rapidapi.com/index.php  
      - Body Type: multipart/form-data  
      - Body Parameter: `url` set dynamically from `{{$json.URL}}`  
      - Headers include required RapidAPI host and API key (user must provide key)  
      - On error: Continue workflow (does not fail workflow on HTTP error)  
    - Inputs: JSON from form submission node  
    - Outputs: JSON response with media links or error info  
    - Edge Cases:  
      - Invalid API key or quota exceeded leads to error in response JSON  
      - API downtime or network issues may cause timeouts or empty responses  
    - Version: 4.2  

#### 2.3 Response Validation

- **Overview:**  
  Checks if the API response contains an error, routing the workflow accordingly.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - Type: If  
    - Role: Branch workflow based on presence of error in API response  
    - Configuration:  
      - Condition: Checks if `$json.error` is boolean false  
      - True path: No error → proceed to Download MP4  
      - False path: Error present → proceed to Wait (failure handling)  
    - Inputs: HTTP Request output  
    - Outputs: Two branches (true and false)  
    - Edge Cases:  
      - Missing or unexpected response structure could lead to incorrect branching  
      - Expression failures if `$json.error` is undefined or non-boolean  
    - Version: 2.2  

#### 2.4 Video Downloading

- **Overview:**  
  Downloads the MP4 video binary from the first media URL provided by the API.

- **Nodes Involved:**  
  - Download mp4

- **Node Details:**

  - **Download mp4**  
    - Type: HTTP Request  
    - Role: Download video file binary  
    - Configuration:  
      - URL dynamically set to `{{$json.medias[0].url}}` from API response  
      - Default GET method, no additional headers or parameters  
    - Inputs: If node's true branch output JSON with media URLs  
    - Outputs: Binary data of MP4 file  
    - Edge Cases:  
      - If `medias` array is empty or missing, URL will be invalid  
      - Network timeouts or 404 errors if media URL is expired or invalid  
    - Version: 4.2  

#### 2.5 Google Drive Upload

- **Overview:**  
  Uploads the downloaded MP4 video file to Google Drive under a specified folder.

- **Nodes Involved:**  
  - Upload To Google Drive

- **Node Details:**

  - **Upload To Google Drive**  
    - Type: Google Drive  
    - Role: Upload binary video data to Google Drive  
    - Configuration:  
      - Drive: "My Drive" (default)  
      - Folder: root or specified folder (default "root")  
      - Authentication: OAuth2 using Google Drive account credentials  
    - Inputs: Binary output from Download mp4 node  
    - Outputs: JSON with uploaded file metadata including file ID  
    - Edge Cases:  
      - Insufficient permissions or quota on Google Drive  
      - Network issues causing upload failure  
      - Large files may require chunked upload (not configured here)  
    - Version: 3  

#### 2.6 Permission Setting

- **Overview:**  
  Sets the uploaded Google Drive file permission to "Anyone with the link can view," making it publicly accessible.

- **Nodes Involved:**  
  - Google Drive Set Permission

- **Node Details:**

  - **Google Drive Set Permission**  
    - Type: Google Drive  
    - Role: Adjust file sharing permissions  
    - Configuration:  
      - File ID dynamically taken from upload response (`{{$json.id}}`)  
      - Operation: share (set permission)  
      - Permission: public view (anyone with link)  
      - Authentication: OAuth2 Google Drive account  
    - Inputs: Output metadata from Upload To Google Drive  
    - Outputs: Confirmation of permission change and sharable link  
    - Edge Cases:  
      - Permission API limits or failures  
      - Trying to set permission on invalid file ID  
    - Version: 3  

#### 2.7 Failure Handling & Logging

- **Overview:**  
  Introduces a wait before logging failures to Google Sheets, preventing rapid-fire writes, then appends failed URL attempts.

- **Nodes Involved:**  
  - Wait  
  - Google Sheets Append Row

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Delay execution to avoid rapid consecutive Google Sheets writes  
    - Configuration: Default wait time (not explicitly set)  
    - Inputs: False branch of If node (indicating failure)  
    - Outputs: Passes to Sheets append node  
    - Edge Cases: Long delays may cause workflow timeouts if excessive  
    - Version: 1.1  

  - **Google Sheets Append Row**  
    - Type: Google Sheets  
    - Role: Log failed conversion attempts  
    - Configuration:  
      - Operation: Append row  
      - Columns:  
        - URL: original LinkedIn URL from form submission  
        - Drive_URL: "N/A" indicating failure to download/upload  
      - Document and Sheet IDs configured via URL mode (user must set)  
      - Authentication: Service Account credentials for Google Sheets  
    - Inputs: Output of Wait node  
    - Outputs: Confirmation of row append  
    - Edge Cases:  
      - Incorrect or missing Sheet/Document IDs  
      - Insufficient permissions for Sheets API  
      - Rate limits on Sheets API  
    - Version: 4.6  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                      | Input Node(s)          | Output Node(s)                  | Sticky Note                                                                                                                                |
|-------------------------|-------------------------|------------------------------------|------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger            | Receive LinkedIn URL input          | -                      | HTTP Request                   | 🟢 **1. On form submission**: Trigger form for URL input                                                                                   |
| HTTP Request            | HTTP Request            | Call RapidAPI to get media links   | On form submission     | If                             | 🌐 **2. HTTP Request**: POST to RapidAPI LinkedIn Video Downloader                                                                         |
| If                      | If                      | Check for API errors               | HTTP Request           | Download mp4 (true), Wait (false) | 🔍 **3. If**: Branch based on error presence                                                                                                |
| Download mp4            | HTTP Request            | Download video binary              | If (true)              | Upload To Google Drive          | ⬇️ **4. Download mp4**: Download MP4 from media URL                                                                                        |
| Upload To Google Drive  | Google Drive            | Upload video to Drive              | Download mp4           | Google Drive Set Permission     | ☁️ **5. Upload To Google Drive**: Upload MP4 file                                                                                          |
| Google Drive Set Permission | Google Drive         | Make uploaded file public          | Upload To Google Drive | -                              | 🔑 **6. Google Drive Set Permission**: Set file accessible publicly                                                                        |
| Wait                    | Wait                    | Delay before failure logging       | If (false)             | Google Sheets Append Row        | ⏱️ **8. Wait**: Pause before logging failure                                                                                              |
| Google Sheets Append Row | Google Sheets           | Log failure in Sheets              | Wait                   | -                              | 📑 **9. Google Sheets Append Row**: Log failed URLs                                                                                        |
| Sticky Note1            | Sticky Note             | Documentation                     | -                      | -                              | See respective notes per node                                                                                                              |
| Sticky Note2            | Sticky Note             | Documentation                     | -                      | -                              | See respective notes per node                                                                                                              |
| Sticky Note3            | Sticky Note             | Documentation                     | -                      | -                              | See respective notes per node                                                                                                              |
| Sticky Note4            | Sticky Note             | Documentation                     | -                      | -                              | See respective notes per node                                                                                                              |
| Sticky Note5            | Sticky Note             | Documentation                     | -                      | -                              | See respective notes per node                                                                                                              |
| Sticky Note6            | Sticky Note             | Documentation                     | -                      | -                              | See respective notes per node                                                                                                              |
| Sticky Note8            | Sticky Note             | Documentation                     | -                      | -                              | See respective notes per node                                                                                                              |
| Sticky Note9            | Sticky Note             | Documentation                     | -                      | -                              | See respective notes per node                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Set form title: "Linkedin to MP4"  
   - Add one required field: Label "URL" (placeholder: "https://linkedin.com/abcdefg")  
   - Set description: "Linkedin to MP4 Converter"  
   - Save and activate webhook.

2. **Add HTTP Request Node**  
   - Name: "HTTP Request"  
   - Method: POST  
   - URL: `https://linkedin-video-downloader3.p.rapidapi.com/index.php`  
   - Content Type: multipart/form-data  
   - Body Parameters: Add parameter "url" with value expression: `{{$json.URL}}` (from form submission)  
   - Headers:  
     - `x-rapidapi-host`: "linkedin-video-downloader3.p.rapidapi.com"  
     - `x-rapidapi-key`: Enter your RapidAPI key here (credential required)  
   - Set error handling: On error continue  
   - Connect output of "On form submission" to this node.

3. **Add If Node**  
   - Name: "If"  
   - Condition: Check if `$json.error` is boolean false  
   - True branch: proceed if no error  
   - False branch: proceed if error present  
   - Connect output of HTTP Request to this node.

4. **Add Download mp4 Node (HTTP Request)**  
   - Name: "Download mp4"  
   - Method: GET (default)  
   - URL: Expression: `{{$json.medias[0].url}}` (first media URL from API response)  
   - Connect the True output of "If" node to this node.

5. **Add Upload To Google Drive Node**  
   - Name: "Upload To Google Drive"  
   - Operation: Upload file  
   - Drive: "My Drive" (default)  
   - Folder: root or select specific folder  
   - Authentication: Google Drive OAuth2 credentials (set up prior)  
   - Input: Binary data from "Download mp4" node  
   - Connect output of "Download mp4" to this node.

6. **Add Google Drive Set Permission Node**  
   - Name: "Google Drive Set Permission"  
   - Operation: Share file  
   - File ID: Expression: `{{$json.id}}` (from upload node output)  
   - Set permission type: Anyone with link can view  
   - Authentication: Same Google Drive OAuth2 credentials  
   - Connect output of "Upload To Google Drive" to this node.

7. **Add Wait Node**  
   - Name: "Wait"  
   - Use default wait time (or configure if desired)  
   - Connect False output of "If" node to this node.

8. **Add Google Sheets Append Row Node**  
   - Name: "Google Sheets Append Row"  
   - Operation: Append  
   - Document ID and Sheet Name: Configure with target Google Sheets URL or ID  
   - Columns to append:  
     - URL: Expression: `{{$node["On form submission"].json.URL}}`  
     - Drive_URL: Set static value "N/A"  
   - Authentication: Google Sheets Service Account credentials (set up prior)  
   - Connect output of "Wait" node to this node.

9. **Activate the workflow** and test by submitting LinkedIn URLs via the form.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| RapidAPI LinkedIn Video Downloader API requires a valid API key and has usage quotas.               | https://rapidapi.com/                                                                           |
| Google Drive OAuth2 credentials must have permission to upload and change file sharing settings.   | https://developers.google.com/drive/api/v3/about-auth                                            |
| Google Sheets Service Account requires Editor access to the target spreadsheet for row appends.    | https://developers.google.com/sheets/api/guides/authorizing                                      |
| The workflow includes a wait step to mitigate rapid API calls or Google Sheets rate limits.        | Prevents write spamming and potential throttling                                                |
| The form trigger node requires a publicly accessible webhook URL for users to submit URLs.         | Deploy n8n with public access or use tunneling (e.g., n8n cloud, ngrok)                          |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---