Upload files via n8n form and save them to Digital Ocean Spaces

https://n8nworkflows.xyz/workflows/upload-files-via-n8n-form-and-save-them-to-digital-ocean-spaces-2660


# Upload files via n8n form and save them to Digital Ocean Spaces

### 1. Workflow Overview

This workflow enables users to upload files through a web form and save them to Digital Ocean Spaces, a scalable S3-compatible object storage service. The uploaded files are set to be publicly accessible immediately after upload. The workflow’s primary use case is for quickly hosting files (especially images for SEO tags) on Digital Ocean Spaces via a simple form interface.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures user file uploads through an n8n web form trigger.
- **1.2 File Upload Processing:** Uploads the received file to Digital Ocean Spaces using the S3 node.
- **1.3 User Confirmation:** Provides the user with a confirmation message including the public URL of the uploaded file.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow by listening for form submissions containing file uploads. It presents a simple form with one required file field and captures the submitted file data.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **Node:** On form submission  
  - **Type and Role:** Form Trigger node; starts the workflow upon user submitting the form.  
  - **Configuration:**  
    - Form titled "Upload File" with one required field labeled "File to Upload" of type file.  
    - Form description prompts for uploading a file to public storage.  
    - Webhook ID assigned to receive submissions.  
  - **Key Expressions/Variables:** None on this node; data output includes JSON metadata and binary data for the uploaded file.  
  - **Input/Output Connections:** No input nodes; output connected to S3 upload node.  
  - **Version Requirements:** Uses formTrigger node version 2.2.  
  - **Edge Cases / Potential Failures:**  
    - Missing file input (form enforces required field).  
    - Network or webhook registration failures could prevent reception.  
    - Large file uploads may hit size/time limits.  
  - **Sub-workflow:** None.

---

#### 1.2 File Upload Processing

**Overview:**  
Uploads the file received from the form submission to Digital Ocean Spaces using the S3-compatible API, setting the file as publicly readable.

**Nodes Involved:**  
- S3

**Node Details:**

- **Node:** S3  
  - **Type and Role:** S3 node; performs file upload operation to Digital Ocean Spaces.  
  - **Configuration:**  
    - Operation: upload  
    - Bucket name: "dailyai"  
    - File name: dynamically set to the original uploaded filename from the form input (`{{$json['File to Upload'][0].filename}}`).  
    - Binary property name: "File_to_Upload" (matches the binary data from form submission).  
    - Additional fields: sets ACL to "publicRead" to make the uploaded file publicly accessible.  
  - **Key Expressions/Variables:** Uses expression for filename from incoming JSON.  
  - **Input/Output Connections:** Input from "On form submission"; output to "Form" node.  
  - **Credential Configuration:** Uses Digital Ocean Spaces credentials configured with S3-compatible access (includes URL, bucket, region, and keys).  
  - **Version Requirements:** S3 node version 1.  
  - **Edge Cases / Potential Failures:**  
    - Incorrect bucket name or region will cause upload failure.  
    - Misconfigured or expired credentials will cause authentication errors.  
    - ACL setting errors may prevent public access.  
    - Large file uploads may timeout.  
  - **Sub-workflow:** None.

---

#### 1.3 User Confirmation

**Overview:**  
Delivers a completion message back to the user after successful upload, providing a direct public URL to the uploaded file in Digital Ocean Spaces.

**Nodes Involved:**  
- Form

**Node Details:**

- **Node:** Form  
  - **Type and Role:** Form node; sends a completion confirmation message to the user post-upload.  
  - **Configuration:**  
    - Operation: completion  
    - Completion title: "Your file path is below!"  
    - Completion message: dynamically generates the public URL to the uploaded file using the filename from the original form submission:  
      `https://dailyai.nyc3.cdn.digitaloceanspaces.com/{{ $('On form submission').first().json['File to Upload'][0].filename }}`  
  - **Key Expressions/Variables:** Expression referencing the filename from the "On form submission" node's JSON data.  
  - **Input/Output Connections:** Input from S3 upload node; no further outputs.  
  - **Version Requirements:** Form node version 1.  
  - **Edge Cases / Potential Failures:**  
    - If upload failed or filename missing, the URL will be incorrect or empty.  
    - Expression evaluation failure if referenced data is unavailable.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role           | Input Node(s)       | Output Node(s) | Sticky Note                                                                                      |
|--------------------|---------------------|--------------------------|---------------------|----------------|------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger        | Captures file upload via form submission | None                | S3             | Credentials are tricky, check screenshot for URL, bucket, etc. See setup instructions and ACL.  |
| S3                 | S3                  | Uploads received file to Digital Ocean Spaces | On form submission  | Form           | Use S3 node with ACL set to publicRead. Check bucket name, region, permissions if upload fails. |
| Form               | Form                | Provides upload confirmation with file URL | S3                  | None           | Completion message shows public URL formed dynamically from original filename.                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a **Form Trigger** node named `On form submission`.  
   - Configure the form title as `"Upload File"`.  
   - Add one form field:  
     - Type: `file`  
     - Label: `"File to Upload"`  
     - Mark as required.  
   - Add a form description: `"Upload the file to the public storage area"`.  
   - Save to generate a webhook URL (ensure webhook is active).

2. **Create the S3 Upload Node**  
   - Add an **S3** node named `S3`.  
   - Set operation to `upload`.  
   - Set bucket name to `"dailyai"` (or your Digital Ocean Space bucket).  
   - Configure the file name with expression: `{{$json["File to Upload"][0].filename}}` to use the uploaded file’s original name.  
   - Set binary property name to `"File_to_Upload"`.  
   - Under Additional Fields, set ACL to `"publicRead"` to make files publicly accessible.  
   - Configure credentials:  
     - Create Digital Ocean Spaces credentials using the S3 credential type with:  
       - Endpoint URL (e.g., `https://nyc3.digitaloceanspaces.com`)  
       - Access key and secret key from your Digital Ocean Spaces dashboard.  
       - Correct region and bucket settings as per your Space.  
   - Connect the output of `On form submission` to this node’s input.

3. **Create the Confirmation Form Node**  
   - Add a **Form** node named `Form`.  
   - Set operation to `completion`.  
   - Set a completion title: `"Your file path is below!"`.  
   - Set the completion message with expression:  
     `=https://dailyai.nyc3.cdn.digitaloceanspaces.com/{{ $('On form submission').first().json['File to Upload'][0].filename }}`  
   - Connect the output of `S3` node to this node’s input.

4. **Activate the Workflow**  
   - Ensure all nodes are active and credentials are properly configured.  
   - Test by submitting a file via the form URL.  
   - Confirm the file is uploaded to Digital Ocean Spaces and the confirmation message displays the correct public URL.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                          |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| Credentials setup can be tricky; refer to the provided screenshots for URL, bucket, and region setup | https://dailyai.nyc3.cdn.digitaloceanspaces.com/do_template_example.png |
| Set ACL to "publicRead" in S3 node to ensure files are publicly accessible                           | https://dailyai.nyc3.cdn.digitaloceanspaces.com/do_settings.png       |
| Verify bucket name, region, and credentials if uploads fail                                         | Troubleshooting section in the description               |
| Video demonstration available (goes live 24h after creation)                                        | https://youtu.be/pYOpy3Ntt1o                              |

---

This structured documentation enables complete understanding, modification, and reimplementation of the workflow, covering all nodes, configurations, potential error points, and references for credential setup and troubleshooting.