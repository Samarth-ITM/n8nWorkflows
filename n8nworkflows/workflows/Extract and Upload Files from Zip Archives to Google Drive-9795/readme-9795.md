Extract and Upload Files from Zip Archives to Google Drive

https://n8nworkflows.xyz/workflows/extract-and-upload-files-from-zip-archives-to-google-drive-9795


# Extract and Upload Files from Zip Archives to Google Drive

### 1. Workflow Overview

This n8n workflow automates the extraction and upload of files contained within a ZIP archive to Google Drive. It is designed for scenarios where users submit ZIP files via a form, and the workflow processes these archives by saving locally, decompressing the contents, splitting the extracted files, and finally uploading each file to a specified Google Drive folder.

The workflow is organized into four logical blocks:

- **1.1 Form Input Reception:** Captures the ZIP file upload via a user-submitted form.
- **1.2 Local File Handling:** Saves the uploaded ZIP file to the local filesystem and reads it back for processing.
- **1.3 Decompression and Splitting:** Unzips the archive and splits the extracted files to process each individually.
- **1.4 Upload to Google Drive:** Uploads each extracted file to a designated folder in Google Drive.

---

### 2. Block-by-Block Analysis

#### 1.1 Form Input Reception

**Overview:**  
This block initiates the workflow by receiving a ZIP file uploaded through a web form. It simplifies user input by providing a single file upload field in the form.

**Nodes Involved:**  
- On form submission  
- Sticky Note (explaining the step)

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Starts the workflow when a form is submitted by a user.  
  - *Configuration:*  
    - Form titled "my example form" with one field labeled "file_upload" configured to accept a single file upload.  
    - No additional options; uses webhook-based triggering.  
  - *Expressions:* None; directly outputs the uploaded file data.  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Passes the uploaded file data downstream.  
  - *Potential Failures:* Form submission errors, missing file upload, webhook failure, or malformed file upload data.

- **Sticky Note**  
  - *Content:* "## #1 Optional Step \nUse a form to collect the zip file. THis form has one field to simplify the concept"  
  - *Role:* Provides contextual explanation for users or maintainers.

---

#### 1.2 Local File Handling

**Overview:**  
This block handles writing the uploaded ZIP file to a local disk path and then reading it back for decompression. This local file access ensures compatibility with the compression node's requirements.

**Nodes Involved:**  
- Read/Write Files from Disk  
- Read/Write Files from Disk1  
- Sticky Note (explaining the block)

**Node Details:**

- **Read/Write Files from Disk**  
  - *Type:* Read/Write File  
  - *Role:* Writes the uploaded ZIP file to the local disk (C:\temp_n8n.zip).  
  - *Configuration:*  
    - Operation: Write  
    - File path: `c:/temp_n8n.zip` (absolute path)  
    - Data source: The file binary data from the form submission (`file_upload` field) via expression: `=file_upload`  
  - *Input:* Receives form submission data containing the uploaded file.  
  - *Output:* Passes metadata of the written file downstream.  
  - *Potential Failures:* File write permission errors, invalid file data, path availability issues.

- **Read/Write Files from Disk1**  
  - *Type:* Read/Write File  
  - *Role:* Reads the previously saved ZIP file from disk for further processing.  
  - *Configuration:*  
    - Operation: Read (default)  
    - File selector path: Dynamically set from previous node’s output expression: `={{ $('Read/Write Files from Disk').item.json.fileName }}`  
  - *Input:* Receives output from the write operation.  
  - *Output:* Passes the binary file data downstream to decompression.  
  - *Potential Failures:* File not found, read permission errors.

- **Sticky Note1**  
  - *Content:* "## #2 Local Processing\nDownload the file to the local n8n server so it can be accessed to properly decompress in the next step"  
  - *Role:* Explains the necessity of local disk operations for decompression compatibility.

---

#### 1.3 Decompression and Splitting

**Overview:**  
This block decompresses the ZIP archive into its constituent files and splits these files into individual items for separate downstream processing.

**Nodes Involved:**  
- Compression  
- Split Out  
- Sticky Note (explaining the block)

**Node Details:**

- **Compression**  
  - *Type:* Compression Node  
  - *Role:* Unzips the archive file passed from the local read operation.  
  - *Configuration:*  
    - Operation: Decompress (default)  
    - No additional options set, implying default ZIP handling.  
  - *Input:* Receives binary ZIP file from “Read/Write Files from Disk1.”  
  - *Output:* Binary content of extracted files as a batch.  
  - *Potential Failures:* Corrupt ZIP files, unsupported compression formats, large file size timeouts.

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the decompressed binary files array into individual workflow items for batch processing.  
  - *Configuration:*  
    - Field to split out: `$binary` (the binary data array of extracted files).  
  - *Input:* Receives decompressed files as an array from Compression node.  
  - *Output:* Outputs one item per extracted file downstream.  
  - *Potential Failures:* Empty archive content, unexpected binary structure.

- **Sticky Note2**  
  - *Content:* "## #3 Decompress the file\n Allows access of one or more files inside, then split them out to batch process down stream\n"  
  - *Role:* Explains decompression and splitting functionality.

---

#### 1.4 Upload to Google Drive

**Overview:**  
This block uploads each extracted file individually to a specified folder within Google Drive, using OAuth2 credentials for authentication.

**Nodes Involved:**  
- Upload to Google Drive  
- Sticky Note (explaining the block)

**Node Details:**

- **Upload to Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Uploads individual extracted files to Google Drive.  
  - *Configuration:*  
    - File name is dynamically set based on the current binary file’s name: `={{ $binary[Object.keys($binary)[$itemIndex]].fileName }}`  
    - Drive selection: "My Drive" (default Google Drive root)  
    - Folder ID: `1leEuOROy_V6bis5oLEsLB3GBK5Kjq9d1` (target folder named "n8n_test")  
    - Input data field name: dynamically constructed as `file_` plus the current item index (`file_0`, `file_1`, etc.) to map the correct binary data.  
  - *Credentials:* Uses a configured Google Drive OAuth2 credential named "Google Drive account" with ID `a9N0beCVBPULZsRd`.  
  - *Input:* Receives individual extracted file items from Split Out node.  
  - *Output:* None (end node).  
  - *Potential Failures:* OAuth token expiration, insufficient Google Drive permissions, quota limits, invalid folder ID, network timeouts.

- **Sticky Note3**  
  - *Content:* "## #4 Upload to Drive\nYOu can upload the file to do anything you want to the files from this point on"  
  - *Role:* Provides context and flexibility note for file handling after upload.

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                           | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                     |
|------------------------|-----------------------|-----------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger          | Receives ZIP file upload via form       | -                           | Read/Write Files from Disk   | ## #1 Optional Step \nUse a form to collect the zip file. THis form has one field to simplify the concept |
| Read/Write Files from Disk | Read/Write File       | Writes uploaded ZIP file to local disk  | On form submission          | Read/Write Files from Disk1  | ## #2 Local Processing\nDownload the file to the local n8n server so it can be accessed to properly decompress in the next step |
| Read/Write Files from Disk1 | Read/Write File       | Reads ZIP file from local disk           | Read/Write Files from Disk  | Compression                 | ## #2 Local Processing\nDownload the file to the local n8n server so it can be accessed to properly decompress in the next step |
| Compression            | Compression           | Decompresses ZIP archive                 | Read/Write Files from Disk1 | Split Out                   | ## #3 Decompress the file\n Allows access of one or more files inside, then split them out to batch process down stream\n |
| Split Out              | Split Out             | Splits decompressed files for individual processing | Compression                 | Upload to Google Drive       | ## #3 Decompress the file\n Allows access of one or more files inside, then split them out to batch process down stream\n |
| Upload to Google Drive | Google Drive          | Uploads each extracted file to Google Drive folder | Split Out                   | -                           | ## #4 Upload to Drive\nYOu can upload the file to do anything you want to the files from this point on |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `On form submission`  
   - Type: Form Trigger  
   - Configuration:  
     - Form Title: "my example form"  
     - Form Description: "testing"  
     - Add one field:  
       - Field Type: File  
       - Field Label: "file_upload"  
       - Allow multiple files: No  
   - This node will start the workflow upon form submission with a ZIP file.

2. **Add a Read/Write File node**  
   - Name: `Read/Write Files from Disk`  
   - Type: Read/Write File  
   - Operation: Write  
   - File Name: `c:/temp_n8n.zip` (Windows absolute path)  
   - Data Property Name: `=file_upload` (binds to the uploaded file binary from the form)  
   - Connect `On form submission` output to this node.

3. **Add another Read/Write File node**  
   - Name: `Read/Write Files from Disk1`  
   - Type: Read/Write File  
   - Operation: Read (default)  
   - File Selector: Set dynamically using expression: `={{ $('Read/Write Files from Disk').item.json.fileName }}` (reads the written file path)  
   - Connect `Read/Write Files from Disk` output to this node.

4. **Add a Compression node**  
   - Name: `Compression`  
   - Type: Compression  
   - Operation: Decompress (default)  
   - Connect `Read/Write Files from Disk1` output to this node.

5. **Add a Split Out node**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Field to split out: `$binary` (this targets the array of extracted files)  
   - Connect `Compression` output to this node.

6. **Add a Google Drive node**  
   - Name: `Upload to Google Drive`  
   - Type: Google Drive  
   - Operation: Upload File (default)  
   - File Name: Use expression: `={{ $binary[Object.keys($binary)[$itemIndex]].fileName }}` to dynamically assign each file’s name.  
   - Drive ID: Select "My Drive"  
   - Folder ID: Enter or select the desired folder ID, for example `"1leEuOROy_V6bis5oLEsLB3GBK5Kjq9d1"` (folder must exist in your Google Drive)  
   - Input Data Field Name: Use expression: `={{ 'file_'+$itemIndex }}` to map the current binary item.  
   - Credentials: Setup Google Drive OAuth2 credentials before use and select them here.  
   - Connect `Split Out` output to this node.

7. **Optional:** Add sticky notes to explain each block for clarity.

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                           |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow requires local file system access on the n8n host machine to write and read ZIP files. | Important for deployment environments (e.g., Docker mounts) |
| Google Drive OAuth2 credentials must be pre-configured with appropriate scopes for file upload.    | Google Drive API documentation for OAuth2 scopes          |
| The form trigger requires a publicly accessible n8n webhook URL to accept form submissions.         | n8n form trigger documentation                             |
| ZIP file corruption or unsupported compression formats will cause failures in decompression step.  | Consider validation or error handling improvements         |

---

**Disclaimer:**  
The provided content is extracted solely from an automated n8n workflow. It adheres strictly to current content policies and contains no illegal or protected material. All processed data is legal and publicly accessible.