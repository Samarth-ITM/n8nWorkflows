Automatically optimise images added to a Google drive folder

https://n8nworkflows.xyz/workflows/automatically-optimise-images-added-to-a-google-drive-folder-2185


# Automatically optimise images added to a Google drive folder

### 1. Workflow Overview

This workflow automates the optimization of images uploaded to a specific Google Drive folder by leveraging the TinyPNG API. It is designed to monitor a chosen Google Drive folder for new image files, send each detected image to the TinyPNG service for compression and optimization, and then save the optimized image back to a specified Google Drive folder. This workflow is ideal for users who want to reduce image file sizes automatically to save storage space or improve web performance without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception (Google Drive Trigger):** Watches a designated Google Drive folder for any new image files added.
- **1.2 Image Download:** Downloads the detected image file from Google Drive to prepare it for optimization.
- **1.3 Image Optimization (TinyPNG API):** Sends the downloaded image to TinyPNG for compression and retrieves the optimized version.
- **1.4 Upload Optimized Image:** Uploads the optimized image back to a chosen Google Drive folder.
- **1.5 Configuration and Guidance (Sticky Notes):** Various sticky notes provide setup instructions and usage tips.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception (Google Drive Trigger)

- **Overview:** This block monitors a specific Google Drive folder for newly created files, triggering the workflow each time a new image is added.
- **Nodes Involved:** 
  - Check GDrive for new images
- **Node Details:**

  - **Node Name:** Check GDrive for new images  
  - **Type:** Google Drive Trigger  
  - **Technical Role:** Watches a Google Drive folder for file creation events.  
  - **Configuration:**  
    - Event set to `fileCreated`.  
    - Polling every minute to check for new files.  
    - Trigger limited to a specific folder, which must be selected by the user.  
  - **Key Expressions:** Folder ID selected dynamically (`folderToWatch` is configured to list and select a folder).  
  - **Input/Output:** No input (trigger node). Outputs JSON metadata on new file creation, including file ID.  
  - **Version Requirements:** Uses Google Drive OAuth2 credentials.  
  - **Potential Failures:** Authentication errors (invalid/expired Google credentials), incorrect folder ID, API rate limits, or network timeouts.  
  - **Notes:** Requires Google Drive OAuth2 credentials preconfigured; the folder to watch must be explicitly selected.

#### 1.2 Image Download

- **Overview:** Downloads the newly added image from Google Drive so it can be sent to TinyPNG for optimization.
- **Nodes Involved:** 
  - Download image
- **Node Details:**

  - **Node Name:** Download image  
  - **Type:** Google Drive  
  - **Technical Role:** Downloads file content based on file ID.  
  - **Configuration:**  
    - Operation set to `download`.  
    - File ID taken dynamically from the trigger node’s output (`{{$json.id}}`).  
  - **Input/Output:** Input: file ID from trigger; Output: binary image data for optimization.  
  - **Version Requirements:** Uses Google Drive OAuth2 credentials consistent with the trigger node.  
  - **Potential Failures:** File not found, expired credentials, API limits, large file causing timeout, non-image files triggering the workflow.  
  - **Notes:** Must handle only image file types; workflow assumes the trigger folder only contains images.

#### 1.3 Image Optimization (TinyPNG API)

- **Overview:** Sends the downloaded image to TinyPNG’s API for compression and retrieves the optimized image URL.
- **Nodes Involved:** 
  - Optimise - Send image to TinyPNG  
  - Get optimised image from tinyPNG
- **Node Details:**

  - **Node Name:** Optimise - Send image to TinyPNG  
    - **Type:** HTTP Request  
    - **Technical Role:** Sends POST request with image binary data to TinyPNG’s `/shrink` API endpoint.  
    - **Configuration:**  
      - URL: `https://api.tinify.com/shrink`  
      - Method: POST  
      - Content-Type: `binaryData` (sends the image data)  
      - Headers include `Authorization` with API key in Base64 format (e.g., `"Basic YOUR_API_KEY_BASE64"`).  
      - Response configured to capture full HTTP response (important to get `location` header).  
    - **Input/Output:** Input: binary image data from Google Drive download; Output: JSON response containing `location` header URL for optimized image.  
    - **Potential Failures:** API key missing or invalid, authentication failed, rate limiting (500 free compressions/month), network timeout, invalid image data, TinyPNG service downtime.  

  - **Node Name:** Get optimised image from tinyPNG  
    - **Type:** HTTP Request  
    - **Technical Role:** Retrieves the optimized image from the URL provided in the previous node’s response header.  
    - **Configuration:**  
      - URL dynamically set from previous node’s output: `={{ $json.headers.location }}`.  
      - Method defaults to GET.  
    - **Input/Output:** Input: URL from previous node; Output: binary optimized image data.  
    - **Potential Failures:** URL missing or expired, network issues, service unavailability.  

#### 1.4 Upload Optimized Image

- **Overview:** Uploads the optimized image back to a selected Google Drive folder with a customizable file name.
- **Nodes Involved:** 
  - Google Drive
- **Node Details:**

  - **Node Name:** Google Drive  
  - **Type:** Google Drive  
  - **Technical Role:** Uploads the optimized image binary data to a specified Google Drive folder.  
  - **Configuration:**  
    - Operation is file upload.  
    - File name by default set to `name.png` (can be customized, e.g., original filename plus `-optimised`).  
    - Drive ID and folder ID are selectable from the user’s Google Drive account.  
  - **Input/Output:** Input: binary optimized image from TinyPNG retrieval; Output: metadata of uploaded file in Google Drive.  
  - **Version Requirements:** Uses the same Google Drive OAuth2 credentials.  
  - **Potential Failures:** Authentication errors, incorrect folder permissions, file name conflicts, quota exceeded, network issues.  

#### 1.5 Configuration and Guidance (Sticky Notes)

- **Overview:** Several sticky note nodes provide setup instructions, tips, and best practices for configuring the workflow.
- **Nodes Involved:**  
  - Sticky Note4  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3
- **Node Details:**

  - **Sticky Note4:** Describes the overall workflow purpose: automatic image optimization on Google Drive upload.  
  - **Sticky Note:** Explains setting up Google Drive credentials and applying them to all drive nodes (trigger, download, upload). Contains a link to n8n’s Google OAuth docs: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/  
  - **Sticky Note1:** Guides choosing the Google Drive folder to watch for new images.  
  - **Sticky Note2:** Instructs creating a TinyPNG API key and updating the HTTP Request node’s Authorization header accordingly. Contains link: https://tinypng.com/developers  
  - **Sticky Note3:** Advises selecting the Google Drive folder to save optimized images and optionally customizing the output filename.  

---

### 3. Summary Table

| Node Name                         | Node Type             | Functional Role                          | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                                      |
|----------------------------------|-----------------------|----------------------------------------|------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------|
| Check GDrive for new images       | Google Drive Trigger   | Watches folder for new image files     | None                         | Download image                   | See Sticky Note1: Instructions to select folder to watch                                                        |
| Download image                   | Google Drive          | Downloads new image from Google Drive  | Check GDrive for new images  | Optimise - Send image to TinyPNG | See Sticky Note: Setup Google Drive credentials and apply to all Drive nodes                                    |
| Optimise - Send image to TinyPNG | HTTP Request          | Sends image to TinyPNG API for shrink  | Download image               | Get optimised image from tinyPNG | See Sticky Note2: Setup TinyPNG API key and Authorization header                                               |
| Get optimised image from tinyPNG | HTTP Request          | Retrieves optimized image from TinyPNG | Optimise - Send image to TinyPNG | Google Drive                    |                                                                                                                  |
| Google Drive                    | Google Drive          | Uploads optimized image to Drive       | Get optimised image from tinyPNG | None                           | See Sticky Note3: Select output folder and optionally customize filename                                        |
| Sticky Note4                    | Sticky Note           | Workflow overview and purpose          | None                         | None                            |                                                                                                                  |
| Sticky Note                     | Sticky Note           | Google Drive credentials instructions  | None                         | None                            | Contains link to Google OAuth docs: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/ |
| Sticky Note1                    | Sticky Note           | Folder watch setup instructions        | None                         | None                            |                                                                                                                  |
| Sticky Note2                    | Sticky Note           | TinyPNG API key setup instructions     | None                         | None                            | Contains link to TinyPNG API: https://tinypng.com/developers                                                    |
| Sticky Note3                    | Sticky Note           | Output folder and filename instructions| None                         | None                            |                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive OAuth2 Credentials in n8n**
   - Follow n8n docs: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/
   - Ensure the credential has access to both read and write Google Drive scopes.

2. **Add a Google Drive Trigger Node**
   - Node Type: Google Drive Trigger  
   - Name: `Check GDrive for new images`  
   - Set event to `fileCreated`.  
   - Set polling mode to every 1 minute.  
   - Choose `Trigger on specific folder` and select the folder to watch for new images (must be an existing folder you manage).  
   - Assign the Google Drive OAuth2 credential created in Step 1.

3. **Add a Google Drive Node for Downloading Files**
   - Node Type: Google Drive  
   - Name: `Download image`  
   - Operation: `download`  
   - File ID: Set dynamically using an expression: `{{$json.id}}` from the trigger node output.  
   - Assign the same Google Drive OAuth2 credential.

4. **Add HTTP Request Node to Send Image to TinyPNG**
   - Node Type: HTTP Request  
   - Name: `Optimise - Send image to TinyPNG`  
   - Method: `POST`  
   - URL: `https://api.tinify.com/shrink`  
   - Content Type: `binaryData` (send image file content)  
   - Input Data Field Name: `data` (binary field from previous node)  
   - Headers: Add header `Authorization` with value `Basic YOUR_API_KEY_BASE64` (replace with your actual base64-encoded TinyPNG API key).  
   - Set to return the full response (to capture headers).  

5. **Add HTTP Request Node to Retrieve Optimized Image**
   - Node Type: HTTP Request  
   - Name: `Get optimised image from tinyPNG`  
   - Method: GET (default)  
   - URL: Use expression to dynamically get optimized image URL from previous node’s response header: `={{ $json.headers.location }}`  

6. **Add Google Drive Node to Upload Optimized Image**
   - Node Type: Google Drive  
   - Name: `Google Drive`  
   - Operation: Upload file  
   - File Name: e.g., use a static name or expression to append `-optimised` suffix to original filename (e.g., `={{ $json.name.split('.').slice(0, -1).join('.') + '-optimised.' + $json.name.split('.').pop() }}`)  
   - Folder ID: Select the Google Drive folder where optimized images will be saved.  
   - Assign the Google Drive OAuth2 credential.

7. **Connect the Nodes in Order**
   - `Check GDrive for new images` → `Download image` → `Optimise - Send image to TinyPNG` → `Get optimised image from tinyPNG` → `Google Drive`.

8. **Optional: Add Sticky Notes**
   - Add sticky notes to provide setup instructions for users or maintainers, as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Google Drive OAuth2 credential setup is essential and must be applied to all Google Drive nodes (trigger, download, upload). | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/                   |
| TinyPNG API key is required and must be base64-encoded for the Authorization header in HTTP requests.                        | https://tinypng.com/developers                                                                      |
| Free TinyPNG API tier allows 500 image optimizations per month; consider usage limits to avoid errors.                        | https://tinypng.com/developers                                                                      |
| Naming convention for optimized files can be customized; default is original filename with `-optimised` suffix.              | User customizable                                                                                   |
| Polling interval is set to 1 minute; adjust based on usage needs and API quota considerations.                                 |                                                                                                    |

---

This documentation fully describes the workflow's structure, node configurations, integration points, and setup instructions. It enables users and automation agents to understand, reproduce, and maintain the automated image optimization process efficiently.