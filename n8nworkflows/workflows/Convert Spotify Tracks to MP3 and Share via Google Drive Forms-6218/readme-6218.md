Convert Spotify Tracks to MP3 and Share via Google Drive Forms

https://n8nworkflows.xyz/workflows/convert-spotify-tracks-to-mp3-and-share-via-google-drive-forms-6218


# Convert Spotify Tracks to MP3 and Share via Google Drive Forms

---

### 1. Workflow Overview

This workflow automates the conversion of Spotify track URLs into downloadable MP3 files, uploads these files to Google Drive, and sets their sharing permissions to be publicly accessible. It targets users who want to quickly convert and share Spotify tracks without manual downloading or file management.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Collects Spotify track URLs submitted by users via a web form.
- **1.2 Spotify MP3 Conversion:** Sends the URL to a RapidAPI service that generates a downloadable MP3 link.
- **1.3 Processing Delay:** Waits briefly to ensure the MP3 file is ready for download.
- **1.4 MP3 Download:** Downloads the MP3 file using the generated download link.
- **1.5 Google Drive Upload:** Uploads the downloaded MP3 to Google Drive using a service account.
- **1.6 Permission Update:** Updates the uploaded file’s permissions to make it publicly accessible.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by presenting a form to the user to submit a Spotify track URL.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - **Type:** `formTrigger` (Webhook Trigger)  
    - **Role:** Receives user input via a web form titled "Spotify To Mp3" with a single field named `url`.  
    - **Configuration:**  
      - Form Title: "Spotify To Mp3"  
      - Form Fields: Single field labeled `url` for user to input Spotify track URL.  
    - **Key Expressions:** None; directly captures form field input as `$json.url`.  
    - **Input Connections:** None (entry point).  
    - **Output Connections:** Connected to "Spotify Rapid Api" node.  
    - **Edge Cases / Failures:**  
      - Missing or malformed URL input may cause downstream errors.  
      - Webhook connectivity issues could prevent form submissions from triggering workflow.  
    - **Notes:** Simple and user-friendly front end for URL input.

#### 1.2 Spotify MP3 Conversion

- **Overview:**  
  Sends the collected Spotify URL to a third-party RapidAPI service that processes the URL and returns a downloadable MP3 link.

- **Nodes Involved:**  
  - Spotify Rapid Api

- **Node Details:**

  - **Spotify Rapid Api**  
    - **Type:** `httpRequest`  
    - **Role:** Posts the Spotify URL to RapidAPI endpoint to retrieve an MP3 download URL.  
    - **Configuration:**  
      - HTTP Method: POST  
      - URL: `https://spotify-downloader-mp3.p.rapidapi.com/spotify-downloader.php`  
      - Headers:  
        - `x-rapidapi-host`: `spotify-downloader-mp3.p.rapidapi.com`  
        - `x-rapidapi-key`: API key placeholder (`your key`) — must be replaced with a valid RapidAPI key.  
      - Body Type: `multipart/form-data` with `url` parameter set to `={{ $json.url }}` (value from form input).  
    - **Input Connections:** From "On form submission" node.  
    - **Output Connections:** To "Wait" node.  
    - **Edge Cases / Failures:**  
      - Invalid or unsupported Spotify URLs may cause API errors or no download URL returned.  
      - API key invalid or quota exceeded leads to authentication or rate limit errors.  
      - Network failures/timeouts.  
    - **Requirements:** Valid RapidAPI subscription and key.  
    - **Notes:** Critical external dependency; its reliability affects entire workflow.

#### 1.3 Processing Delay

- **Overview:**  
  Adds a delay to allow time for the MP3 file generation on the API side before attempting to download.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - **Type:** `wait`  
    - **Role:** Pauses the workflow briefly (default duration, unspecified) after receiving the MP3 download URL.  
    - **Configuration:** No specific wait time configured (default behavior).  
    - **Input Connections:** From "Spotify Rapid Api".  
    - **Output Connections:** To "Downloader".  
    - **Edge Cases / Failures:**  
      - If the wait duration is too short, download might fail due to incomplete file readiness.  
      - If too long, unnecessary delay increases workflow runtime.  
    - **Notes:** Configurable wait time recommended for optimization.

#### 1.4 MP3 Download

- **Overview:**  
  Downloads the MP3 file using the download URL received from RapidAPI.

- **Nodes Involved:**  
  - Downloader

- **Node Details:**

  - **Downloader**  
    - **Type:** `httpRequest`  
    - **Role:** Performs HTTP GET request to download the MP3 file.  
    - **Configuration:**  
      - HTTP Method: GET  
      - URL: Dynamically set as `={{ $json.download_url }}`, extracted from previous node output.  
    - **Input Connections:** From "Wait" node.  
    - **Output Connections:** To "Upload Mp3 To Google Drive".  
    - **Edge Cases / Failures:**  
      - Missing or invalid `download_url`.  
      - Network errors, timeouts, or file size limits.  
      - If the file is large, potential memory constraints in n8n.  
    - **Notes:** Ensures the workflow obtains the actual MP3 content for upload.

#### 1.5 Google Drive Upload

- **Overview:**  
  Uploads the downloaded MP3 file to Google Drive in a specified folder using service account authentication.

- **Nodes Involved:**  
  - Upload Mp3 To Google Drive

- **Node Details:**

  - **Upload Mp3 To Google Drive**  
    - **Type:** `googleDrive`  
    - **Role:** Uploads the MP3 binary data to Google Drive.  
    - **Configuration:**  
      - Drive: "My Drive" (default Google Drive root).  
      - Folder: `"root"` folder ID (default root directory).  
      - Authentication: Service Account credentials configured.  
    - **Input Connections:** From "Downloader" node (expects binary data).  
    - **Output Connections:** To "Update Permission".  
    - **Edge Cases / Failures:**  
      - Authentication errors if service account credentials invalid or insufficient permissions.  
      - Upload failure due to file size limits or quota exceeded.  
    - **Notes:** Service account must have sharing permissions on target Drive folder.

#### 1.6 Permission Update

- **Overview:**  
  Updates the uploaded MP3 file’s sharing permissions to be publicly accessible, enabling easy sharing.

- **Nodes Involved:**  
  - Update Permission

- **Node Details:**

  - **Update Permission**  
    - **Type:** `googleDrive`  
    - **Role:** Sets sharing permissions on uploaded file to allow public write access.  
    - **Configuration:**  
      - Operation: `share`  
      - Permission Role: `writer` (can edit) — configurable, commonly changed to `reader` for safer sharing.  
      - Permission Type: `anyone` (no sign-in required).  
      - File ID: Dynamically set from uploaded file metadata.  
      - Authentication: Service Account credentials.  
    - **Input Connections:** From "Upload Mp3 To Google Drive".  
    - **Output Connections:** None (end of workflow).  
    - **Edge Cases / Failures:**  
      - Invalid file ID or permission settings may cause errors.  
      - Overly permissive settings (writer) may pose security risks.  
    - **Notes:** Adjust permission role based on security needs.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                                | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                      |
|--------------------------|--------------------|-----------------------------------------------|------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| On form submission       | formTrigger        | Receives Spotify URL from user form           | None                   | Spotify Rapid Api            | Collects the Spotify URL from a user through a form with one field labeled `url`.              |
| Spotify Rapid Api        | httpRequest        | Sends URL to RapidAPI to get MP3 download link| On form submission     | Wait                        | Sends the Spotify URL to RapidAPI endpoint with POST and required headers.                     |
| Wait                    | wait               | Adds delay to ensure MP3 readiness             | Spotify Rapid Api       | Downloader                  | Adds a delay before downloading the MP3 to ensure file is ready.                              |
| Downloader              | httpRequest        | Downloads MP3 file from provided download URL  | Wait                   | Upload Mp3 To Google Drive  | Downloads the MP3 file using URL from previous node.                                          |
| Upload Mp3 To Google Drive| googleDrive       | Uploads MP3 to Google Drive folder              | Downloader             | Update Permission           | Uploads the MP3 file using service account to Google Drive root or specified folder.          |
| Update Permission       | googleDrive        | Sets public sharing permissions on uploaded file| Upload Mp3 To Google Drive | None                       | Makes file publicly accessible with `writer` role for anyone (can be changed to `reader`).    |
| Sticky Note              | stickyNote         | Documentation and overview                      | None                   | None                       | # 🎵 Spotify to MP3 → Google Drive ... (full workflow explanation and benefits)               |
| Sticky Note1             | stickyNote         | Notes on formTrigger node                       | None                   | None                       | ## 1. On form submission (`formTrigger`) - Purpose: Collects the Spotify URL from a user.      |
| Sticky Note2             | stickyNote         | Notes on Spotify Rapid API node                 | None                   | None                       | ## 2. Spotify Rapid Api (`httpRequest`) - Sends URL to RapidAPI to generate MP3.               |
| Sticky Note3             | stickyNote         | Notes on Wait node                              | None                   | None                       | ## 3. Wait (`wait`) - Adds delay before download.                                              |
| Sticky Note4             | stickyNote         | Notes on Downloader node                        | None                   | None                       | ## 4. Downloader (`httpRequest`) - Downloads MP3 file using dynamic URL.                       |
| Sticky Note5             | stickyNote         | Notes on Google Drive upload                    | None                   | None                       | ## 5. Upload Mp3 To Google Drive (`googleDrive`) - Uploads MP3 using service account.          |
| Sticky Note6             | stickyNote         | Notes on Permission update node                 | None                   | None                       | ## 6. Update Permission (`googleDrive`) - Makes uploaded file publicly accessible.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `formTrigger` node named `On form submission`:**  
   - Set `Form Title` to `"Spotify To Mp3"`.  
   - Add a single form field labeled `url`.  
   - This node will serve as the entry point for user-submitted Spotify URLs.

3. **Add an `httpRequest` node named `Spotify Rapid Api`:**  
   - Set Method to `POST`.  
   - Set URL to `https://spotify-downloader-mp3.p.rapidapi.com/spotify-downloader.php`.  
   - Under Headers, add:  
     - `x-rapidapi-host` = `spotify-downloader-mp3.p.rapidapi.com`  
     - `x-rapidapi-key` = your valid RapidAPI key (replace `"your key"` placeholder).  
   - Under Body Parameters, set content type to `multipart/form-data`.  
   - Add a parameter named `url` with value expression `{{$json["url"]}}` to send the Spotify URL.  
   - Connect the output of `On form submission` to this node.

4. **Add a `wait` node named `Wait`:**  
   - No configuration needed unless you want a specific delay (default is immediate).  
   - Connect the output of `Spotify Rapid Api` to this node.

5. **Add an `httpRequest` node named `Downloader`:**  
   - Set Method to `GET`.  
   - Set URL to the expression `{{$json["download_url"]}}` to dynamically use the download URL from the previous step.  
   - Connect the output of `Wait` node to this node.

6. **Add a `googleDrive` node named `Upload Mp3 To Google Drive`:**  
   - Set Authentication to `Service Account`.  
   - Select the Drive as `"My Drive"` or your desired Drive.  
   - Set Folder ID to `root` or specify a folder ID.  
   - Configure credentials with a Google service account that has permission to upload to the chosen Drive.  
   - Connect the output of `Downloader` to this node, ensuring the binary data from the MP3 download is passed.

7. **Add a `googleDrive` node named `Update Permission`:**  
   - Set Operation to `share`.  
   - Set Permission Role to `writer` (or `reader` for safer sharing).  
   - Set Permission Type to `anyone`.  
   - Set File ID to the uploaded file's ID dynamically (use expression referencing previous node output file ID).  
   - Use the same Google service account credentials.  
   - Connect the output of `Upload Mp3 To Google Drive` to this node.

8. **Optionally, add sticky notes or documentation nodes to annotate each step for clarity.**

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This workflow automates Spotify track URL conversion to MP3, uploads to Google Drive, and sets public access permissions.                                       | Workflow purpose summary.                                                                                                   |
| Replace the `x-rapidapi-key` with a valid RapidAPI key for the Spotify downloader API to function properly.                                                     | Critical API authentication.                                                                                                |
| The permission role `writer` in the sharing step can be changed to `reader` to limit file access to read-only for security.                                    | Sharing permissions best practices.                                                                                        |
| The `Wait` node delay can be adjusted or removed based on the performance of the MP3 generation API to optimize workflow duration.                              | Performance tuning note.                                                                                                    |
| Service Account credentials must be configured in n8n with appropriate Drive permissions to upload and share files.                                            | Google Drive integration requirements.                                                                                      |
| This workflow could be adapted for other media platforms or cloud storage services by modifying the API and upload nodes accordingly.                          | Extensibility note.                                                                                                         |
| For further troubleshooting, monitor API response payloads and error messages at each step to isolate issues (e.g., invalid URLs, API limits, auth failures). | Operational advice.                                                                                                         |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---