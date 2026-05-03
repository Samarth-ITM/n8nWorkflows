Form-Triggered Instagram Video Downloads to Google Drive with Sheets Logging

https://n8nworkflows.xyz/workflows/form-triggered-instagram-video-downloads-to-google-drive-with-sheets-logging-8364


# Form-Triggered Instagram Video Downloads to Google Drive with Sheets Logging

---
### 1. Workflow Overview

This workflow automates the process of converting Instagram video URLs submitted via a web form into downloadable MP4 files stored on Google Drive. It logs both successful and failed download attempts in Google Sheets for tracking purposes.

Logical blocks:

- **1.1 Input Reception:** Captures Instagram video URLs through a form submission.
- **1.2 Instagram Video Download Request:** Sends the URL to a third-party Instagram Downloader API to obtain downloadable video links.
- **1.3 Validation (If Node):** Checks if the API response indicates success or failure.
- **1.4 MP4 Downloading:** Downloads the video file from the provided media URL.
- **1.5 Google Drive Upload & Permission:** Uploads the downloaded MP4 to Google Drive and sets public access permissions.
- **1.6 Success Logging:** Appends successful download records (original URL and Drive link) to Google Sheets.
- **1.7 Failure Handling:** Delays workflow, then logs failed download attempts with `N/A` in place of the Drive URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow via a web form where users submit Instagram video URLs for conversion.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  - **Node:** On form submission  
    - **Type & Role:** Form Trigger node; starts the workflow on a user-submitted form.  
    - **Configuration:**  
      - Form titled "Instagram to MP4" with a single required field "URL" (placeholder: https://instagram.com/).  
      - No extra options configured.  
    - **Expressions/Variables:** Outputs user-entered `URL` as `{{ $json.URL }}`.  
    - **Connections:** Output connected to "Instagram Downloader API Request" node.  
    - **Edge Cases:**  
      - User submits invalid or malformed URLs (validation not handled here).  
      - Form submission failures due to webhook issues.  
    - **Sub-workflow:** None.

#### 2.2 Instagram Video Download Request

- **Overview:**  
  Sends a POST request to a third-party Instagram Downloader API to retrieve downloadable MP4 links for the submitted video URL.

- **Nodes Involved:**  
  - Instagram Downloader API Request

- **Node Details:**  

  - **Node:** Instagram Downloader API Request  
    - **Type & Role:** HTTP Request node; performs API call to retrieve downloadable video links.  
    - **Configuration:**  
      - URL: `https://instagram-downloader51.p.rapidapi.com/download.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Header parameters include RapidAPI host and API key (`x-rapidapi-key` placeholder `"yourkey"`).  
      - Body parameter `url` set to the submitted Instagram URL (`{{ $json.URL }}`).  
      - On error: continue workflow to allow error handling downstream.  
    - **Expressions/Variables:** Uses form submission output `{{ $json.URL }}`.  
    - **Connections:** Output connected to "If" node for response validation.  
    - **Edge Cases:**  
      - API key invalid or rate-limited (authorization errors).  
      - API returning errors or malformed responses.  
      - Network timeouts or unreachable API.  
    - **Sub-workflow:** None.

#### 2.3 Validation (If Node)

- **Overview:**  
  Checks the API response status to determine if the video link retrieval was successful or not.

- **Nodes Involved:**  
  - If

- **Node Details:**  

  - **Node:** If  
    - **Type & Role:** Conditional node; evaluates if API response status equals `"success"`.  
    - **Configuration:**  
      - Condition: string equals comparison of `$json.status` to `"success"`.  
      - True path: proceed to MP4 Downloader node.  
      - False path: proceed to Wait node for failure handling.  
    - **Expressions/Variables:** `$json.status` from API response.  
    - **Connections:** True output to "MP4 Downloader", false output to "Wait".  
    - **Edge Cases:**  
      - API returns unexpected status codes or missing `status` field.  
      - Expression evaluation errors if response structure varies.  
    - **Sub-workflow:** None.

#### 2.4 MP4 Downloading

- **Overview:**  
  Downloads the MP4 video file from the media URL provided by the Instagram Downloader API.

- **Nodes Involved:**  
  - MP4 Downloader

- **Node Details:**  

  - **Node:** MP4 Downloader  
    - **Type & Role:** HTTP Request node; downloads raw MP4 binary data.  
    - **Configuration:**  
      - URL dynamically set to `{{ $json.data[0].downloadUrl }}`, the first downloadable media link in the API response.  
      - Default HTTP GET method.  
    - **Expressions/Variables:** `$json.data[0].downloadUrl` extracted from API response JSON.  
    - **Connections:** Output connected to "Upload To Google Drive" node.  
    - **Edge Cases:**  
      - `data` array empty or missing, causing URL evaluation failure.  
      - Download failures or HTTP errors.  
      - Large file sizes causing timeouts.  
    - **Sub-workflow:** None.

#### 2.5 Google Drive Upload & Permission

- **Overview:**  
  Uploads the downloaded MP4 file to Google Drive and sets its sharing permissions to allow public viewing.

- **Nodes Involved:**  
  - Upload To Google Drive  
  - Google Drive Set Permission

- **Node Details:**  

  - **Node:** Upload To Google Drive  
    - **Type & Role:** Google Drive node; uploads binary file to a specified folder.  
    - **Configuration:**  
      - Drive ID: "My Drive"  
      - Folder ID: Root folder (`root`)  
      - Uses OAuth2 credentials.  
    - **Expressions/Variables:** Receives binary data from "MP4 Downloader".  
    - **Connections:** Output connected to "Google Drive Set Permission".  
    - **Edge Cases:**  
      - Authentication failures (OAuth token expiration).  
      - Insufficient permissions on Google Drive folder.  
      - File size limits or quota exceeded errors.  
    - **Sub-workflow:** None.

  - **Node:** Google Drive Set Permission  
    - **Type & Role:** Google Drive node; updates file sharing permissions.  
    - **Configuration:**  
      - File ID sourced dynamically from upload response `{{ $json.id }}`.  
      - Operation: Share file with permission "Anyone with the link can view".  
      - Uses OAuth2 credentials.  
    - **Expressions/Variables:** File ID from previous node.  
    - **Connections:** Output connected to "Google Sheets" logging node.  
    - **Edge Cases:**  
      - Permission API errors or quota limits.  
      - Invalid file ID.  
    - **Sub-workflow:** None.

#### 2.6 Success Logging

- **Overview:**  
  Logs successful Instagram video download attempts into Google Sheets for record-keeping.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  

  - **Node:** Google Sheets  
    - **Type & Role:** Google Sheets node; appends a row to a sheet.  
    - **Configuration:**  
      - Operation: Append  
      - Target sheet and document identified via URL (values masked in JSON).  
      - Authentication via Service Account credentials.  
      - Row values include original URL and publicly accessible Drive link.  
    - **Expressions/Variables:** Uses data from "Google Drive Set Permission" (e.g., sharable link) and original URL.  
    - **Connections:** Final node in success path.  
    - **Edge Cases:**  
      - Service account permission issues on the sheet.  
      - Invalid sheet or document IDs.  
      - API rate limits or network issues.  
    - **Sub-workflow:** None.

#### 2.7 Failure Handling

- **Overview:**  
  Handles API failure responses by delaying workflow execution before logging the failure in Google Sheets.

- **Nodes Involved:**  
  - Wait  
  - Google Sheets Append Row

- **Node Details:**  

  - **Node:** Wait  
    - **Type & Role:** Wait node; delays workflow to prevent rapid logging.  
    - **Configuration:** No specific parameters; default delay used.  
    - **Connections:** Output connected to "Google Sheets Append Row" node.  
    - **Edge Cases:**  
      - Workflow may be held if multiple failures occur in short succession.  
    - **Sub-workflow:** None.

  - **Node:** Google Sheets Append Row  
    - **Type & Role:** Google Sheets node; appends failure record with URL and `Drive_URL` as `N/A`.  
    - **Configuration:**  
      - Operation: Append  
      - Target sheet and document identified via URL (values masked).  
      - Authentication via Service Account credentials.  
      - Row includes original URL and `Drive_URL: N/A`.  
    - **Connections:** Terminal node in failure path.  
    - **Edge Cases:**  
      - Failure to write due to permissions or API limits.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                        | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                          |
|--------------------------------|---------------------|-------------------------------------|------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger        | Captures Instagram URL input         | -                            | Instagram Downloader API Request | 🟢 **1. On form submission**: Trigger with form to accept URL input.                               |
| Instagram Downloader API Request | HTTP Request       | Sends URL to Instagram downloader API | On form submission            | If                              | 🌐 **2. Instagram Downloader API Request**: POST to API to get downloadable video links.           |
| If                            | If                  | Checks API response success status   | Instagram Downloader API Request | MP4 Downloader (true), Wait (false) | 🔍 **3. If**: Routes workflow based on API success or error.                                      |
| MP4 Downloader                | HTTP Request        | Downloads MP4 from media URL          | If (true)                    | Upload To Google Drive           | ⬇️ **4. MP4 Downloader**: Downloads video file from media URL.                                    |
| Upload To Google Drive         | Google Drive        | Uploads MP4 to Drive folder           | MP4 Downloader               | Google Drive Set Permission      | ☁️ **5. Upload To Google Drive**: Upload video to Drive root folder.                              |
| Google Drive Set Permission    | Google Drive        | Sets public sharing permission        | Upload To Google Drive       | Google Sheets                   | 🔑 **6. Google Drive Set Permission**: Make uploaded file publicly viewable.                      |
| Google Sheets                 | Google Sheets       | Logs successful downloads             | Google Drive Set Permission  | -                               | 📄 **7. Google Sheets**: Logs URL and Drive link of successful downloads.                         |
| Wait                         | Wait                | Delays workflow on failure             | If (false)                   | Google Sheets Append Row         | ⏱️ **8. Wait**: Delay before logging failures to avoid rapid writes.                             |
| Google Sheets Append Row       | Google Sheets       | Logs failed download attempts          | Wait                        | -                               | 📑 **9. Google Sheets Append Row**: Logs failures with `Drive_URL` as `N/A`.                      |
| Sticky Note1                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 1.                                                              |
| Sticky Note2                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 2.                                                              |
| Sticky Note3                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 3.                                                              |
| Sticky Note4                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 4.                                                              |
| Sticky Note5                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 5.                                                              |
| Sticky Note6                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 6.                                                              |
| Sticky Note7                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 7.                                                              |
| Sticky Note8                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 8.                                                              |
| Sticky Note9                  | Sticky Note         | Documentation note                     | -                            | -                               | See Sticky Note content for block 9.                                                              |
| Sticky Note                   | Sticky Note         | Overall workflow documentation        | -                            | -                               | Full workflow description and summary.                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node:**
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Configure form title as "Instagram to MP4".  
   - Add one required text field labeled "URL" with placeholder "https://instagram.com/".  
   - Save and activate webhook.

2. **Add HTTP Request Node for Instagram Downloader API:**
   - Type: HTTP Request  
   - Name: "Instagram Downloader API Request"  
   - Set method to POST.  
   - URL: `https://instagram-downloader51.p.rapidapi.com/download.php`  
   - Content-Type: multipart/form-data.  
   - Add header parameters:  
     - `x-rapidapi-host`: `instagram-downloader51.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key (replace `"yourkey"`).  
   - Add body parameter named `url` with value `{{ $json.URL }}` from form trigger.  
   - Set error handling to "Continue on Fail" to allow failure branch.  
   - Connect output from form trigger node to this node.

3. **Add If Node to Check API Response:**
   - Type: If  
   - Name: "If"  
   - Condition: Check if `$json.status` equals `"success"`.  
   - Connect output of API Request node to this node.  
   - True branch connects to MP4 Downloader.  
   - False branch connects to Wait node.

4. **Add HTTP Request Node for MP4 Downloading:**
   - Type: HTTP Request  
   - Name: "MP4 Downloader"  
   - Method: GET (default)  
   - URL: `{{ $json.data[0].downloadUrl }}` (first downloadable media link)  
   - Connect True output from If node to this node.

5. **Add Google Drive Upload Node:**
   - Type: Google Drive  
   - Name: "Upload To Google Drive"  
   - Drive ID: Select "My Drive".  
   - Folder ID: Select root folder (`root`).  
   - Upload binary data from MP4 Downloader node.  
   - Authenticate using OAuth2 credentials linked to your Google Drive account.  
   - Connect output of MP4 Downloader node to this node.

6. **Add Google Drive Set Permission Node:**
   - Type: Google Drive  
   - Name: "Google Drive Set Permission"  
   - Operation: Share file  
   - Resource: File  
   - File ID: Use expression `{{ $json.id }}` from upload response.  
   - Set permission to "Anyone with the link can view".  
   - Use same OAuth2 credentials as upload.  
   - Connect output of Upload To Google Drive node to this node.

7. **Add Google Sheets Node for Success Logging:**
   - Type: Google Sheets  
   - Name: "Google Sheets"  
   - Operation: Append  
   - Document ID and Sheet Name: Provide your Google Sheet URL or ID and target sheet.  
   - Authentication: Service Account credentials configured with access to the sheet.  
   - Map fields to append original URL and sharable Google Drive link from previous node outputs.  
   - Connect output of Google Drive Set Permission node to this node.

8. **Add Wait Node for Failure Delay:**
   - Type: Wait  
   - Name: "Wait"  
   - No configuration needed unless custom delay desired.  
   - Connect False output of If node to this node.

9. **Add Google Sheets Append Node for Failure Logging:**
   - Type: Google Sheets  
   - Name: "Google Sheets Append Row"  
   - Operation: Append  
   - Document ID and Sheet Name: Provide your Google Sheet URL or ID and target sheet for failures.  
   - Authentication: Service Account credentials.  
   - Append the original URL and set `Drive_URL` field to `"N/A"`.  
   - Connect output of Wait node to this node.

10. **Test the Workflow:**
    - Submit the form with a valid Instagram video URL.  
    - Verify the video is downloaded, uploaded to Drive, permissions set, and success logged.  
    - Test failure path with invalid URL or API failure to verify failure logging after delay.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                 | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow uses a third-party Instagram Downloader API accessible via RapidAPI. You must replace `"yourkey"` with a valid RapidAPI key for the Instagram downloader API. | RapidAPI service for Instagram video downloads: https://rapidapi.com/                                            |
| Google Drive OAuth2 credentials must have appropriate scopes for file upload and permission setting, typically `drive.file` and `drive.permission`.                            | Google Drive API documentation: https://developers.google.com/drive/api/v3/about-auth                             |
| Google Sheets Service Account credentials require sharing the target spreadsheet with the service account email to grant write access.                                       | Google Sheets API guide: https://developers.google.com/sheets/api/guides/authorizing                               |
| The Wait node helps prevent rapid consecutive writes to Google Sheets, which can trigger API rate limits or quota issues.                                                   | n8n Wait node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.wait/                                       |
| The workflow includes detailed sticky notes explaining each block's purpose and key configurations for maintainability and future modification.                              | Sticky notes are embedded in the workflow for user reference.                                                    |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.