Watermark and password-protect new Google Drive PDFs with Autype

https://n8nworkflows.xyz/workflows/watermark-and-password-protect-new-google-drive-pdfs-with-autype-13892


# Watermark and password-protect new Google Drive PDFs with Autype

# 1. Workflow Overview

This workflow monitors a specific Google Drive folder for newly created PDF files. Whenever a new PDF appears, it downloads the file, uploads it once to Autype, then processes it in two parallel branches:

- one branch adds a watermark and saves a new `-watermark.pdf` version to Google Drive
- the other branch password-protects the PDF and saves a new `-protected.pdf` version to Google Drive

The workflow is designed for document security and distribution scenarios such as:

- creating confidential review copies
- generating protected versions of sensitive PDFs
- automating post-upload compliance handling for shared folders

The logic can be grouped into the following blocks:

## 1.1 Trigger and Source File Retrieval

This block detects a newly created file in a specific Google Drive folder and downloads its binary content so it can be processed downstream.

## 1.2 Upload to Autype for Shared Processing

This block uploads the downloaded PDF to Autype once and reuses the returned file ID for both document-processing branches.

## 1.3 Watermark Branch

This branch applies a watermark to the uploaded PDF using Autype and stores the generated binary file back in Google Drive.

## 1.4 Password Protection Branch

This branch encrypts the uploaded PDF with user and owner passwords using Autype, then uploads the protected output back to Google Drive.

## 1.5 Documentation / In-Canvas Notes

Several sticky notes document the workflow purpose, requirements, and the two processing branches directly on the canvas.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Source File Retrieval

### Overview

This block watches a specific Google Drive folder for new files and downloads the created file as binary data. It is the workflow entry point and supplies the original PDF name and content used by all downstream steps.

### Nodes Involved

- New PDF Uploaded to Drive
- Download PDF from Drive

### Node Details

#### New PDF Uploaded to Drive

- **Type and technical role:**  
  `n8n-nodes-base.googleDriveTrigger`  
  Polling trigger node that watches Google Drive for file creation events.

- **Configuration choices:**  
  - Trigger event: `fileCreated`
  - Trigger scope: specific folder
  - Polling frequency: every minute
  - Folder watched: `YOUR_FOLDER_ID`

- **Key expressions or variables used:**  
  No custom expression is used in this node’s main parameters.

- **Input and output connections:**  
  - Input: none, this is a trigger node
  - Output: sends the new file metadata to **Download PDF from Drive**

- **Version-specific requirements:**  
  - Node type version: `1`
  - Uses Google Drive OAuth2 credentials

- **Edge cases or potential failure types:**  
  - Invalid or missing Google Drive OAuth2 credentials
  - Folder ID not found or inaccessible
  - Polling delays depending on API timing
  - Non-PDF files may still trigger if the folder receives them; despite the node name, there is no explicit MIME-type filter configured
  - Permission issues if the connected account cannot access the folder

- **Sub-workflow reference:**  
  None

---

#### Download PDF from Drive

- **Type and technical role:**  
  `n8n-nodes-base.googleDrive`  
  Downloads the newly created file from Google Drive as binary data.

- **Configuration choices:**  
  - Operation: `download`
  - File ID is taken dynamically from the trigger output using `{{$json.id}}`

- **Key expressions or variables used:**  
  - `={{ $json.id }}` to identify which file to download

- **Input and output connections:**  
  - Input: **New PDF Uploaded to Drive**
  - Output: **Upload PDF to Autype**

- **Version-specific requirements:**  
  - Node type version: `3`
  - Requires the same or equivalent Google Drive OAuth2 credential with read access

- **Edge cases or potential failure types:**  
  - File deleted or moved between trigger detection and download
  - Trigger item missing `id`
  - Download failure for unsupported Google-native files if a non-binary Drive file is created
  - Large file transfers may hit timeout or memory constraints depending on n8n environment

- **Sub-workflow reference:**  
  None

---

## 2.2 Upload to Autype for Shared Processing

### Overview

This block uploads the downloaded binary PDF into Autype’s file storage so that document processing operations can reference it by file ID. The returned ID is then reused by both parallel branches, avoiding duplicate uploads.

### Nodes Involved

- Upload PDF to Autype

### Node Details

#### Upload PDF to Autype

- **Type and technical role:**  
  `n8n-nodes-autype.autype`  
  Uploads the input PDF file to Autype so it can be used by downstream document-tools operations.

- **Configuration choices:**  
  - Resource: `file`
  - No extra operation is specified in the JSON, so the node behavior is the resource’s upload action

- **Key expressions or variables used:**  
  No visible custom expression in the node configuration. Downstream nodes rely on the returned `id`.

- **Input and output connections:**  
  - Input: **Download PDF from Drive**
  - Output: branches in parallel to:
    - **Add Watermark**
    - **Password-Protect PDF**

- **Version-specific requirements:**  
  - Node type version: `1`
  - Requires Autype API credentials
  - The sticky note states that the uploaded file ID is valid for 60 minutes

- **Edge cases or potential failure types:**  
  - Missing binary input if the previous node did not download correctly
  - Invalid Autype API key
  - Community node not installed
  - File upload failure due to API availability, file format, or size restrictions
  - If downstream processing is delayed beyond the file ID lifetime, Autype operations may fail

- **Sub-workflow reference:**  
  None

---

## 2.3 Watermark Branch

### Overview

This branch takes the uploaded Autype file ID, applies a watermark, and returns the generated PDF as binary output. The output is then saved into the same Google Drive folder using a derived filename.

### Nodes Involved

- Add Watermark
- Save Watermarked PDF to Drive

### Node Details

#### Add Watermark

- **Type and technical role:**  
  `n8n-nodes-autype.autype`  
  Uses Autype Document Tools to create a watermarked version of the uploaded PDF.

- **Configuration choices:**  
  - Resource: `documentTools`
  - Operation: `watermark`
  - Source file ID: `{{$json.id}}`
  - Download output: enabled, so the processed file is returned to n8n as binary
  - Watermark options:
    - opacity: `0.3`
    - font size: `50`
    - rotation: `-45`

- **Key expressions or variables used:**  
  - `={{ $json.id }}` for the uploaded Autype file ID

- **Input and output connections:**  
  - Input: **Upload PDF to Autype**
  - Output: **Save Watermarked PDF to Drive**

- **Version-specific requirements:**  
  - Node type version: `1`
  - Requires Autype API credentials and installed community node

- **Edge cases or potential failure types:**  
  - Invalid or expired Autype file ID
  - Watermark operation failure due to unsupported PDF structure
  - Binary output missing even if processing completes unexpectedly
  - The sticky note says the watermark text is `"CONFIDENTIAL"`, but no explicit text field appears in the provided JSON; behavior may depend on Autype defaults or omitted node settings. This should be verified before production use.

- **Sub-workflow reference:**  
  None

---

#### Save Watermarked PDF to Drive

- **Type and technical role:**  
  `n8n-nodes-base.googleDrive`  
  Uploads the watermarked binary PDF to Google Drive.

- **Configuration choices:**  
  - Destination drive: `My Drive`
  - Destination folder: `YOUR_FOLDER_ID`
  - File name is built from the original downloaded file name with `.pdf` removed, then `-watermark.pdf` appended

- **Key expressions or variables used:**  
  - `={{ $('Download PDF from Drive').item.json.name.replace(/\.pdf$/i, '') }}-watermark.pdf`

- **Input and output connections:**  
  - Input: **Add Watermark**
  - Output: none

- **Version-specific requirements:**  
  - Node type version: `3`
  - Requires Google Drive OAuth2 credentials with write access

- **Edge cases or potential failure types:**  
  - If the original file name is missing, the expression will fail
  - If the input file does not end with `.pdf`, the regex replacement does nothing and still appends `-watermark.pdf`
  - Possible overwrite or duplicate-name behavior depends on Google Drive node defaults and folder contents
  - Upload failure if binary property is not present in the incoming item
  - Folder access or quota issues

- **Sub-workflow reference:**  
  None

---

## 2.4 Password Protection Branch

### Overview

This branch uses the Autype file ID to generate an encrypted PDF protected by both a user password and an owner password. It then uploads the protected file back into Google Drive under a derived output name.

### Nodes Involved

- Password-Protect PDF
- Save Protected PDF to Drive

### Node Details

#### Password-Protect PDF

- **Type and technical role:**  
  `n8n-nodes-autype.autype`  
  Uses Autype Document Tools to apply PDF password protection.

- **Configuration choices:**  
  - Resource: `documentTools`
  - Operation: `protect`
  - Source file ID: `{{$json.id}}`
  - Download output: enabled
  - Protection options:
    - user password: `open-secret`
    - owner password: `owner-secret`

- **Key expressions or variables used:**  
  - `={{ $json.id }}` for the uploaded file reference

- **Input and output connections:**  
  - Input: **Upload PDF to Autype**
  - Output: **Save Protected PDF to Drive**

- **Version-specific requirements:**  
  - Node type version: `1`
  - Requires Autype credentials and installed community node

- **Edge cases or potential failure types:**  
  - Hard-coded passwords are insecure for production use
  - Expired or invalid Autype file ID
  - Unsupported or corrupted PDF input
  - API or credential failures
  - Missing binary output from Autype

- **Sub-workflow reference:**  
  None

---

#### Save Protected PDF to Drive

- **Type and technical role:**  
  `n8n-nodes-base.googleDrive`  
  Uploads the password-protected PDF binary to Google Drive.

- **Configuration choices:**  
  - Destination drive: `My Drive`
  - Destination folder: `YOUR_FOLDER_ID`
  - Output filename derived from original name plus `-protected.pdf`

- **Key expressions or variables used:**  
  - `={{ $('Download PDF from Drive').item.json.name.replace(/\.pdf$/i, '') }}-protected.pdf`

- **Input and output connections:**  
  - Input: **Password-Protect PDF**
  - Output: none  
  - Note: this node is present in the canvas and in the node list, but no outgoing connection is defined, which is normal for a terminal node

- **Version-specific requirements:**  
  - Node type version: `3`
  - Requires Google Drive OAuth2 credentials with write access

- **Edge cases or potential failure types:**  
  - Same naming-expression risks as the watermark upload node
  - Duplicate names or overwrite behavior may need explicit control
  - Upload can fail if the protected PDF binary is missing
  - Drive permissions, quota, or API-rate issues

- **Sub-workflow reference:**  
  None

---

## 2.5 Documentation / In-Canvas Notes

### Overview

These sticky notes provide embedded operational context: overall workflow purpose, requirements, and branch-level explanations. They are not executable but are important for maintainability and reproduction.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### Sticky Note

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Canvas documentation note describing the full workflow and prerequisites.

- **Configuration choices:**  
  Large note covering workflow description, required accounts, and installation requirements.

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  None

- **Version-specific requirements:**  
  - Node type version: `1`

- **Edge cases or potential failure types:**  
  None at runtime

- **Sub-workflow reference:**  
  None

---

#### Sticky Note1

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Documents the trigger/download/upload stage.

- **Configuration choices:**  
  Colored note associated with the first processing section.

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  None

- **Version-specific requirements:**  
  - Node type version: `1`

- **Edge cases or potential failure types:**  
  None at runtime

- **Sub-workflow reference:**  
  None

---

#### Sticky Note2

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Documents the watermark branch.

- **Configuration choices:**  
  Colored note associated with the watermark-and-save branch.

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  None

- **Version-specific requirements:**  
  - Node type version: `1`

- **Edge cases or potential failure types:**  
  None at runtime

- **Sub-workflow reference:**  
  None

---

#### Sticky Note3

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Documents the protection branch.

- **Configuration choices:**  
  Colored note associated with the protect-and-save branch.

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  None

- **Version-specific requirements:**  
  - Node type version: `1`

- **Edge cases or potential failure types:**  
  None at runtime

- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New PDF Uploaded to Drive | Google Drive Trigger | Watches a specific Drive folder for newly created files |  | Download PDF from Drive | ### 1. Trigger & Download<br>The Google Drive Trigger polls the watched folder every minute. When a new file appears, it downloads the PDF binary and uploads it once to Autype. The returned file ID is shared by both parallel branches. |
| Download PDF from Drive | Google Drive | Downloads the newly detected Drive file as binary | New PDF Uploaded to Drive | Upload PDF to Autype | ### 1. Trigger & Download<br>The Google Drive Trigger polls the watched folder every minute. When a new file appears, it downloads the PDF binary and uploads it once to Autype. The returned file ID is shared by both parallel branches. |
| Upload PDF to Autype | Autype | Uploads the PDF to Autype and returns a reusable file ID | Download PDF from Drive | Add Watermark; Password-Protect PDF | ### 1. Trigger & Download<br>The Google Drive Trigger polls the watched folder every minute. When a new file appears, it downloads the PDF binary and uploads it once to Autype. The returned file ID is shared by both parallel branches. |
| Add Watermark | Autype | Applies watermark processing to the uploaded PDF | Upload PDF to Autype | Save Watermarked PDF to Drive | ### 2.1 Watermark & Save<br>Stamps a diagonal "CONFIDENTIAL" watermark on the uploaded PDF and saves the result back to Google Drive as `*-watermark.pdf`. Runs in parallel with branch 2.2. |
| Save Watermarked PDF to Drive | Google Drive | Uploads the watermarked PDF back to Google Drive | Add Watermark |  | ### 2.1 Watermark & Save<br>Stamps a diagonal "CONFIDENTIAL" watermark on the uploaded PDF and saves the result back to Google Drive as `*-watermark.pdf`. Runs in parallel with branch 2.2. |
| Password-Protect PDF | Autype | Applies PDF password protection using user and owner passwords | Upload PDF to Autype | Save Protected PDF to Drive | ### 2.2 Protect & Save<br>Encrypts the uploaded PDF with a user password (to open) and owner password (to edit), then saves as `*-protected.pdf` in the same folder. Runs in parallel with branch 2.1 — no re-upload needed. |
| Save Protected PDF to Drive | Google Drive | Uploads the protected PDF back to Google Drive | Password-Protect PDF |  | ### 2.2 Protect & Save<br>Encrypts the uploaded PDF with a user password (to open) and owner password (to edit), then saves as `*-protected.pdf` in the same folder. Runs in parallel with branch 2.1 — no re-upload needed. |
| Sticky Note | Sticky Note | Canvas documentation for overall workflow purpose and requirements |  |  | ## Automatically Watermark and Password-Protect New PDFs in Google Drive<br>### When a new PDF is uploaded to a Google Drive folder, this workflow automatically creates two secure versions: one with a "CONFIDENTIAL" watermark and one with password protection.<br><br>Activate the workflow and it runs hands-free. Every new PDF in the watched folder gets processed within a minute.<br><br>### How it works<br>1. **New PDF Uploaded to Drive** — Google Drive Trigger watches a specific folder for new files (polls every minute).<br>2. **Download PDF from Drive** — Downloads the new file as binary data.<br>3. **Upload PDF to Autype** — Uploads to Autype Document Tools storage once (file ID valid for 60 min). Both branches use this same ID.<br>4. **Branch A — Watermark:** Add Watermark → Save `*-watermark.pdf` to Drive.<br>5. **Branch B — Protect:** Password-Protect PDF → Save `*-protected.pdf` to Drive.<br><br>Both branches run in parallel. No re-upload needed.<br><br>### Requirements<br>* **Autype account** — Sign up at [app.autype.com](https://app.autype.com) and go to **Settings → API Keys** to generate your API key.<br>* **Google Drive** — Connect your Google account via OAuth2 in n8n credentials.<br>* **n8n-nodes-autype** — Install the Autype community node via **Settings → Community Nodes** in your self-hosted n8n instance.<br><br>> ⚠️ Community node: requires self-hosted n8n. Not available on n8n Cloud. |
| Sticky Note1 | Sticky Note | Canvas documentation for trigger/download block |  |  | ### 1. Trigger & Download<br>The Google Drive Trigger polls the watched folder every minute. When a new file appears, it downloads the PDF binary and uploads it once to Autype. The returned file ID is shared by both parallel branches. |
| Sticky Note2 | Sticky Note | Canvas documentation for watermark branch |  |  | ### 2.1 Watermark & Save<br>Stamps a diagonal "CONFIDENTIAL" watermark on the uploaded PDF and saves the result back to Google Drive as `*-watermark.pdf`. Runs in parallel with branch 2.2. |
| Sticky Note3 | Sticky Note | Canvas documentation for protection branch |  |  | ### 2.2 Protect & Save<br>Encrypts the uploaded PDF with a user password (to open) and owner password (to edit), then saves as `*-protected.pdf` in the same folder. Runs in parallel with branch 2.1 — no re-upload needed. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Automatically Watermark and Password-Protect New PDFs in Google Drive with Autype`.
   - Ensure the workflow will run in active mode once tested.

2. **Install the Autype community node**
   - In self-hosted n8n, go to **Settings → Community Nodes**.
   - Install `n8n-nodes-autype`.
   - Note that this is required because the workflow uses `n8n-nodes-autype.autype`.
   - This setup is not available on n8n Cloud according to the embedded note.

3. **Create Google Drive credentials**
   - Add a **Google Drive OAuth2** credential in n8n.
   - Authorize access to the Google account that owns or can access the target folder.
   - Ensure the account has:
     - read access to detect and download files
     - write access to upload processed files back to the destination folder

4. **Create Autype credentials**
   - Create or log into an Autype account at [https://app.autype.com](https://app.autype.com).
   - Generate an API key under **Settings → API Keys**.
   - In n8n, create an **Autype API** credential using that key.

5. **Add the trigger node**
   - Create a **Google Drive Trigger** node.
   - Name it: `New PDF Uploaded to Drive`.
   - Configure:
     - **Event:** `File Created`
     - **Trigger On:** `Specific Folder`
     - **Folder to Watch:** select the target Google Drive folder
     - **Poll Time:** every minute
   - Attach your Google Drive OAuth2 credential.

6. **Add the download node**
   - Create a **Google Drive** node.
   - Name it: `Download PDF from Drive`.
   - Configure:
     - **Operation:** `Download`
     - **File ID:** `={{ $json.id }}`
   - Use the same Google Drive credential.
   - Connect `New PDF Uploaded to Drive → Download PDF from Drive`.

7. **Add the Autype upload node**
   - Create an **Autype** node.
   - Name it: `Upload PDF to Autype`.
   - Configure:
     - **Resource:** `File`
   - Attach your Autype credential.
   - Leave it set to upload the incoming binary file.
   - Connect `Download PDF from Drive → Upload PDF to Autype`.

8. **Add the watermark-processing node**
   - Create another **Autype** node.
   - Name it: `Add Watermark`.
   - Configure:
     - **Resource:** `Document Tools`
     - **Operation:** `Watermark`
     - **File ID:** `={{ $json.id }}`
     - **Download Output:** enabled
     - **Opacity:** `0.3`
     - **Font Size:** `50`
     - **Rotation:** `-45`
   - If the Autype node version exposes watermark text explicitly, set it to `CONFIDENTIAL` to match the note in the workflow.
   - Connect `Upload PDF to Autype → Add Watermark`.

9. **Add the Google Drive upload node for the watermarked file**
   - Create a **Google Drive** node.
   - Name it: `Save Watermarked PDF to Drive`.
   - Configure it to upload the incoming binary file to Google Drive.
   - Set:
     - **Drive:** `My Drive`
     - **Folder:** the same watched folder, or another target folder if preferred
     - **Name:**  
       `={{ $('Download PDF from Drive').item.json.name.replace(/\.pdf$/i, '') }}-watermark.pdf`
   - Use your Google Drive OAuth2 credential.
   - Connect `Add Watermark → Save Watermarked PDF to Drive`.

10. **Add the password-protection node**
    - Create another **Autype** node.
    - Name it: `Password-Protect PDF`.
    - Configure:
      - **Resource:** `Document Tools`
      - **Operation:** `Protect`
      - **File ID:** `={{ $json.id }}`
      - **Download Output:** enabled
      - **User Password:** `open-secret`
      - **Owner Password:** `owner-secret`
    - Attach your Autype credential.
    - Connect `Upload PDF to Autype → Password-Protect PDF`.

11. **Add the Google Drive upload node for the protected file**
    - Create a **Google Drive** node.
    - Name it: `Save Protected PDF to Drive`.
    - Configure it to upload the incoming binary file.
    - Set:
      - **Drive:** `My Drive`
      - **Folder:** the same watched folder, or another target folder
      - **Name:**  
        `={{ $('Download PDF from Drive').item.json.name.replace(/\.pdf$/i, '') }}-protected.pdf`
    - Use your Google Drive credential.
    - Connect `Password-Protect PDF → Save Protected PDF to Drive`.

12. **Verify parallel branching**
    - Confirm that `Upload PDF to Autype` has two outgoing connections:
      - one to `Add Watermark`
      - one to `Password-Protect PDF`
    - This ensures both document transformations run independently from the same uploaded Autype file ID.

13. **Optionally add sticky notes for maintainability**
    - Add a general note describing the workflow purpose and requirements.
    - Add one note near the trigger/download/upload block.
    - Add one note near the watermark branch.
    - Add one note near the protect branch.

14. **Test with a sample PDF**
    - Upload a PDF into the watched Google Drive folder.
    - Wait for the trigger polling interval.
    - Confirm that:
      - the original file is detected
      - the PDF is downloaded successfully
      - the file uploads to Autype
      - two output files are created:
        - `originalname-watermark.pdf`
        - `originalname-protected.pdf`

15. **Activate the workflow**
    - Once validation succeeds, activate the workflow.
    - The workflow will then continue polling the folder every minute.

## Credential and setup expectations

- **Google Drive OAuth2**
  - Must support file listing, file download, and file upload in the chosen folder
  - Ensure the selected Drive/folder exists and is accessible

- **Autype API**
  - Must be valid and active
  - Used by all Autype nodes in the workflow

## Important implementation constraints

- The trigger does not explicitly filter MIME type in the provided configuration. If only PDFs should be processed, consider adding:
  - an IF node checking MIME type
  - or a file-extension validation step before download or before Autype upload

- The file naming logic assumes the original item has a `name` field and typically ends with `.pdf`.

- The workflow uses hard-coded passwords:
  - `open-secret`
  - `owner-secret`  
  For real deployments, replace these with secure values from:
  - environment variables
  - encrypted credentials
  - a secrets manager
  - data derived dynamically per file or customer

- The Autype upload note indicates the returned file ID is valid for 60 minutes, so downstream processing should happen promptly.

- If you save output files into the same folder that the trigger watches, verify whether these newly created files could retrigger the workflow. Since the trigger is based on file creation and there is no explicit guard in the workflow, this may create recursive processing depending on Google Drive trigger behavior and file filtering. A safer production design is often:
  - watch an input folder
  - save results into a separate output folder

- No sub-workflow nodes are used in this workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Autype account required; generate an API key under Settings → API Keys | [https://app.autype.com](https://app.autype.com) |
| Google Drive must be connected through OAuth2 credentials in n8n | n8n credentials setup |
| The Autype integration depends on the `n8n-nodes-autype` community node | Install from **Settings → Community Nodes** in self-hosted n8n |
| Community nodes are indicated as unavailable on n8n Cloud for this setup | Operational constraint |
| The embedded canvas note states the workflow creates one watermarked version and one password-protected version for each new PDF | Workflow purpose |
| The embedded canvas note states the watermark branch uses a diagonal “CONFIDENTIAL” watermark | Verify actual watermark text in the Autype node configuration, because the provided JSON does not explicitly show a text parameter |