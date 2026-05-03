Download Videos from Any Platform to Google Drive with RapidAPI Integration

https://n8nworkflows.xyz/workflows/download-videos-from-any-platform-to-google-drive-with-rapidapi-integration-7720


# Download Videos from Any Platform to Google Drive with RapidAPI Integration

### 1. Workflow Overview

This workflow automates downloading videos from any platform via a RapidAPI integration and saves them into Google Drive while managing errors through logging in Google Sheets. It is designed for users who want to submit a video URL through a form and have the video automatically downloaded, uploaded, shared publicly, and monitored for failures.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by a user submitting a video URL via a form.
- **1.2 Video Download Link Retrieval:** Calls the RapidAPI All-In-One Video Downloader API to get downloadable video links.
- **1.3 Error Check and Branching:** Checks if API returned any errors; branches accordingly.
- **1.4 Video Download:** Downloads the MP4 video file using the provided media URL.
- **1.5 Upload to Google Drive:** Uploads the downloaded video file to Google Drive.
- **1.6 Set Google Drive File Permissions:** Makes the uploaded video accessible publicly.
- **1.7 Error Handling and Logging:** Handles failures by waiting and appending error details to a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving a video URL from the user through a web form.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Start node, captures user input.  
    - Configuration: Displays a form titled *"All In one video downloader"* with a single required field labeled `URL`.  
    - Expressions: Passes the submitted `URL` value downstream as `$json.URL`.  
    - Input: None (trigger node).  
    - Output: JSON containing the user-submitted URL.  
    - Failure modes: User may submit invalid URLs (not validated here), but no technical errors expected at this stage.  
    - Notes: Serves as the sole entry point for the workflow.

---

#### 1.2 Video Download Link Retrieval

- **Overview:**  
  Sends the submitted URL to the RapidAPI All-In-One Video Downloader API to fetch downloadable media links.

- **Nodes Involved:**  
  - All in one video downloader

- **Node Details:**

  - **All in one video downloader**  
    - Type: HTTP Request  
    - Role: Calls external API to retrieve downloadable video URLs.  
    - Configuration:  
      - Method: POST  
      - URL: `https://best-all-in-one-video-downloader.p.rapidapi.com/index.php`  
      - Body: Multipart form data containing the `url` parameter set from the form input (`{{$json.URL}}`).  
      - Headers: Includes `x-rapidapi-host` and `x-rapidapi-key` (user must replace `"your key"` with their actual RapidAPI key).  
      - On error: `continueRegularOutput` to avoid workflow stopping on API errors.  
    - Input: Receives JSON with the user-submitted URL.  
    - Output: JSON response containing either media links under `.medias` or an `error` field.  
    - Failure modes:  
      - Invalid API key or quota exceeded causes API errors.  
      - Malformed URLs may return errors or empty results.  
      - Network timeouts or API downtime.  
    - Notes: Critical integration point with external service.

---

#### 1.3 Error Check and Branching

- **Overview:**  
  Checks if the API response has an error. If no error, proceeds to download; if error, initiates error handling path.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - Type: If Node (Conditional branching)  
    - Role: Evaluates if the `error` field exists or not in the API response.  
    - Configuration:  
      - Condition: Checks if `{{$json.error}}` is exactly `false` (i.e., no error present).  
      - Case sensitive and strict type validation enabled.  
    - Input: Receives JSON from API response.  
    - Output:  
      - True path (no error): Connects to video download.  
      - False path (error present): Connects to wait and log error.  
    - Failure modes: Expression failure if response structure changes or missing fields.  
    - Notes: Essential for error handling logic separation.

---

#### 1.4 Video Download

- **Overview:**  
  Downloads the MP4 video file from the first media URL provided by the API response.

- **Nodes Involved:**  
  - Download mp4

- **Node Details:**

  - **Download mp4**  
    - Type: HTTP Request  
    - Role: Downloads the actual video file binary.  
    - Configuration:  
      - URL: Set dynamically to `{{$json.medias[0].url}}`, the first media URL from API response.  
      - Method: GET (default).  
      - Options: Default HTTP options.  
    - Input: Receives JSON containing media URLs.  
    - Output: Binary data of downloaded MP4 video.  
    - Failure modes:  
      - Invalid or expired media URLs.  
      - Network errors or timeouts.  
      - Large file sizes causing timeout or memory issues.  
    - Notes: Downloads only the first media URL; assumes it's MP4.

---

#### 1.5 Upload to Google Drive

- **Overview:**  
  Uploads the downloaded video binary to a specified Google Drive folder.

- **Nodes Involved:**  
  - Upload To Google Drive

- **Node Details:**

  - **Upload To Google Drive**  
    - Type: Google Drive node  
    - Role: Uploads binary data as a file into Google Drive.  
    - Configuration:  
      - Drive ID: Set to "My Drive" (default user drive).  
      - Folder ID: Set to `"root"` (top-level folder).  
      - Uses OAuth2 Google Drive credentials named `"Google Drive account"`.  
    - Input: Receives binary MP4 data from download node.  
    - Output: JSON including the uploaded file's metadata and `id`.  
    - Failure modes:  
      - OAuth credentials invalid or expired.  
      - Insufficient storage quota.  
      - API rate limits.  
    - Notes: File ID used in subsequent permission setting.

---

#### 1.6 Set Google Drive File Permissions

- **Overview:**  
  Makes the uploaded file publicly accessible by setting permissions to “Anyone with the link can view.”

- **Nodes Involved:**  
  - Google Drive Set Permission

- **Node Details:**

  - **Google Drive Set Permission**  
    - Type: Google Drive node  
    - Role: Updates sharing permissions of the uploaded file.  
    - Configuration:  
      - File ID: Dynamically set from upload node output (`{{$json.id}}`).  
      - Operation: Share file.  
      - Permissions: Defaults to public viewing (no explicit role config shown but implied).  
      - Uses same OAuth2 credentials as upload node.  
    - Input: Receives uploaded file metadata.  
    - Output: Confirmation of permission update, including `webViewLink`.  
    - Failure modes:  
      - Permission API failures.  
      - File not found or ID mismatch.  
      - OAuth token expiration.  
    - Notes: Enables easy sharing of uploaded videos.

---

#### 1.7 Error Handling and Logging

- **Overview:**  
  Handles errors by waiting briefly before logging failed download attempts into a Google Sheet.

- **Nodes Involved:**  
  - Wait  
  - Google Sheets Append Row

- **Node Details:**

  - **Wait**  
    - Type: Wait node  
    - Role: Introduces delay to prevent rapid repeated logging.  
    - Configuration: Default wait time (not explicitly set, so default applies).  
    - Input: Receives error path from If node.  
    - Output: Triggers Google Sheets append row.  
    - Failure modes: None significant.  
    - Notes: Helps avoid flooding the Google Sheet with rapid error logs.

  - **Google Sheets Append Row**  
    - Type: Google Sheets node  
    - Role: Logs failed download attempts.  
    - Configuration:  
      - Operation: Append row.  
      - Sheet Name and Document ID: Empty in JSON, must be set by user to target sheet.  
      - Data appended includes:  
        - `URL` (original user input).  
        - `Drive_URL`: Hardcoded to `"N/A"` indicating failure.  
      - Authentication: Service Account credentials named `"Google Docs account"`.  
    - Input: Receives error path data.  
    - Output: Confirmation of row appended.  
    - Failure modes:  
      - Missing or invalid Google Sheets credentials.  
      - Incorrect sheet or document ID.  
      - API quota or permission errors.  
    - Notes: Useful for tracking and auditing failed downloads.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                                    | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                           |
|---------------------------|---------------------|---------------------------------------------------|----------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------|
| On form submission        | Form Trigger        | Trigger; receive video URL input                   | None                       | All in one video downloader    | 🟢 **1. On form submission:** Triggers workflow with URL input form.                                |
| All in one video downloader | HTTP Request       | Call RapidAPI to get downloadable video links      | On form submission         | If                             | 🌐 **2. All In One Downloader:** Calls API with URL to get media links.                             |
| If                        | If Node             | Check for API errors and branch                     | All in one video downloader | Download mp4 (true), Wait (false) | 🔍 **3. If:** Checks API error field to decide next step.                                          |
| Download mp4              | HTTP Request        | Download video binary from media URL                | If (true)                  | Upload To Google Drive          | ⬇️ **4. Download mp4:** Downloads MP4 using first media URL.                                       |
| Upload To Google Drive    | Google Drive        | Upload video file to Google Drive                    | Download mp4               | Google Drive Set Permission     | ☁️ **5. Upload To Google Drive:** Uploads video to Drive root folder.                               |
| Google Drive Set Permission | Google Drive       | Set file permission to public                         | Upload To Google Drive     | None                          | 🔑 **6. Google Drive Set Permission:** Sets file to public view.                                   |
| Wait                      | Wait                | Delay before error logging                            | If (false)                 | Google Sheets Append Row        | ⏱️ **8. Wait:** Delays workflow to avoid quick repeated logs on error.                             |
| Google Sheets Append Row  | Google Sheets       | Log failed download attempt                           | Wait                       | None                          | 📑 **9. Google Sheets Append Row:** Logs failed URL and Drive_URL=N/A in Sheet.                    |
| Sticky Note1              | Sticky Note         | Documentation for On form submission                 | None                       | None                          | See related node sticky note content.                                                              |
| Sticky Note2              | Sticky Note         | Documentation for API call                            | None                       | None                          | See related node sticky note content.                                                              |
| Sticky Note3              | Sticky Note         | Documentation for If node                             | None                       | None                          | See related node sticky note content.                                                              |
| Sticky Note4              | Sticky Note         | Documentation for Download mp4                        | None                       | None                          | See related node sticky note content.                                                              |
| Sticky Note5              | Sticky Note         | Documentation for Upload to Google Drive             | None                       | None                          | See related node sticky note content.                                                              |
| Sticky Note6              | Sticky Note         | Documentation for Google Drive Set Permission        | None                       | None                          | See related node sticky note content.                                                              |
| Sticky Note8              | Sticky Note         | Documentation for Wait node                           | None                       | None                          | See related node sticky note content.                                                              |
| Sticky Note9              | Sticky Note         | Documentation for Google Sheets Append Row           | None                       | None                          | See related node sticky note content.                                                              |
| Sticky Note               | Sticky Note         | Full workflow overview and description                | None                       | None                          | 🧩 Full workflow description, features, use cases, dependencies provided.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node:**
   - Name: `On form submission`
   - Type: Form Trigger
   - Configure form title: `"All In one video downloader"`
   - Add one required field labeled `"URL"`
   - No credentials needed
   - Position as desired, this is the workflow entry point

2. **Create HTTP Request Node for RapidAPI:**
   - Name: `All in one video downloader`
   - Type: HTTP Request
   - Method: POST
   - URL: `https://best-all-in-one-video-downloader.p.rapidapi.com/index.php`
   - Content-Type: `multipart/form-data`
   - Add Body Parameter: `url` with value expression `{{$json.URL}}`
   - Add Headers:
     - `x-rapidapi-host`: `linkedin-video-downloader3.p.rapidapi.com`
     - `x-rapidapi-key`: `<Your RapidAPI Key>` (replace `"your key"` with actual key)
   - Set "On Error" to `continueRegularOutput`
   - Connect `On form submission` output to this node's input

3. **Add If Node to Check for API Errors:**
   - Name: `If`
   - Condition: Check if `{{$json.error}}` is exactly `false` (boolean false)
   - Case sensitive and strict type validation enabled
   - Connect `All in one video downloader` output to this node's input

4. **Create HTTP Request Node to Download Video:**
   - Name: `Download mp4`
   - Type: HTTP Request
   - Method: GET (default)
   - URL: Expression `{{$json.medias[0].url}}`
   - Connect `If` node's **True** output to this node's input

5. **Create Google Drive Upload Node:**
   - Name: `Upload To Google Drive`
   - Type: Google Drive
   - Operation: Upload file (default)
   - Drive ID: Set to `"My Drive"`
   - Folder ID: Set to `"root"` (or specify a folder if desired)
   - Credentials: Set up OAuth2 Google Drive credentials (e.g., named `"Google Drive account"`)
   - Connect `Download mp4` node output (binary data) to this node's input

6. **Create Google Drive Set Permission Node:**
   - Name: `Google Drive Set Permission`
   - Type: Google Drive
   - Operation: Share file
   - Resource: File
   - File ID: Use expression `{{$json.id}}` from upload node output
   - Permissions: Default to "Anyone with the link can view"
   - Use same Google Drive OAuth2 credentials as upload node
   - Connect `Upload To Google Drive` output to this node's input

7. **Create Wait Node for Error Path:**
   - Name: `Wait`
   - Type: Wait
   - Keep default wait time or set a short delay (e.g., a few seconds)
   - Connect `If` node's **False** output (error path) to this node's input

8. **Create Google Sheets Append Row Node:**
   - Name: `Google Sheets Append Row`
   - Type: Google Sheets
   - Operation: Append Row
   - Document ID: Set to your Google Sheets document ID (must be set manually)
   - Sheet Name: Set to your target sheet name (e.g., "Sheet1")
   - Add data fields to append:
     - `URL`: map to input JSON field `URL` (submitted video URL)
     - `Drive_URL`: static value `"N/A"`
   - Credentials: Use Google API credentials with service account access (e.g., `"Google Docs account"`)
   - Connect `Wait` node output to this node's input

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow supports multiple video platforms such as LinkedIn, Facebook, Instagram, etc., via a universal video downloader API.                                                                                                  | Workflow description and use case                             |
| Ensure you have valid RapidAPI credentials and quota for the All-In-One Video Downloader API. Replace `"your key"` in the HTTP request header with your actual API key.                                                                | API key setup instruction                                     |
| Google Drive OAuth2 credentials must have permission to upload files and share links. Setup OAuth accordingly in n8n credentials.                                                                                                     | Google Drive OAuth setup                                      |
| Google Sheets OAuth or Service Account credentials must have write access to the target sheet for logging errors.                                                                                                                    | Google Sheets credential requirement                          |
| The workflow includes error handling with a wait node to prevent flooding Google Sheets with logs in case of multiple rapid failures.                                                                                                | Error handling design                                         |
| For further customization, consider adding URL validation or supporting multiple media formats beyond the first MP4 link.                                                                                                          | Potential workflow extension                                  |
| The workflow includes extensive sticky notes documenting each block and node, aiding maintenance and comprehension directly within n8n.                                                                                            | Embedded documentation                                        |

---

**Disclaimer:**  
The provided content originates solely from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.