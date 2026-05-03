Manage files in S3

https://n8nworkflows.xyz/workflows/manage-files-in-s3-674


# Manage files in S3

### 1. Workflow Overview

This workflow automates the process of downloading a file from a public URL, uploading it to an Amazon S3 bucket, and then retrieving a complete list of all files stored in that bucket. It is designed for use cases involving automated file management and synchronization with S3 storage, such as backup, content distribution, or data aggregation workflows.

The logical structure is divided into three main blocks:

- **1.1 Manual Trigger:** Starts the workflow manually.
- **1.2 File Download and Upload:** Downloads a specified file via HTTP request and uploads it to the configured S3 bucket.
- **1.3 Bucket Content Retrieval:** Fetches a list of all files currently stored in the S3 bucket after the upload operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger

- **Overview:**  
  This initial block allows the workflow to be started manually by a user or system operator.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Configuration:** Default, no parameters required. It awaits manual initiation.  
  - **Expressions/Variables:** None.  
  - **Input/Output:** No input; output triggers the next node "HTTP Request".  
  - **Version Requirements:** Compatible with n8n versions supporting manual triggers (all modern versions).  
  - **Potential Failures:** None expected; manual trigger is stable.  
  - **Sub-Workflow:** Not applicable.

#### 2.2 File Download and Upload

- **Overview:**  
  Downloads a file from a predefined URL and uploads it to a specified S3 bucket for storage.

- **Nodes Involved:**  
  - HTTP Request  
  - S3 (Upload)

- **Node Details:**  

  - **Node Name:** HTTP Request  
    - **Type:** HTTP Request (n8n-nodes-base.httpRequest)  
    - **Role:** Downloads an external file as binary data.  
    - **Configuration:**  
      - URL: `https://n8n.io/n8n-logo.png` (a public URL for the n8n logo image).  
      - Response Format: File (binary response expected).  
      - Options: Default, no authentication or additional headers set.  
    - **Expressions/Variables:** None static; URL is hardcoded.  
    - **Input/Output:** Input from manual trigger; output is binary file data passed to S3 upload node.  
    - **Potential Failures:** Network errors, HTTP errors (404, 500), timeout, invalid URL, file size limits.  
    - **Sub-Workflow:** None.

  - **Node Name:** S3  
    - **Type:** S3 (n8n-nodes-base.s3)  
    - **Role:** Uploads the downloaded binary file to an S3 bucket.  
    - **Configuration:**  
      - Operation: Upload  
      - Bucket Name: `n8n` (target bucket)  
      - File Name: Dynamically assigned from the HTTP Request node’s binary file name (`={{$node["HTTP Request"].binary.data.fileName}}`)  
      - Additional Fields: None  
      - Credentials: AWS S3 credentials referenced as `s3-n8n`  
    - **Expressions/Variables:** Uses expression to set filename dynamically based on input binary data.  
    - **Input/Output:** Input is binary data from HTTP Request node; output triggers the next node "S".  
    - **Version Requirements:** Requires n8n with S3 node version supporting upload operation and expressions.  
    - **Potential Failures:** Authentication errors, permission denied on bucket, network timeouts, invalid bucket name, file size limits, expression evaluation errors.  
    - **Sub-Workflow:** None.

#### 2.3 Bucket Content Retrieval

- **Overview:**  
  Retrieves a list of all files currently stored in the specified S3 bucket after the upload is completed.

- **Nodes Involved:**  
  - S (S3)

- **Node Details:**  
  - **Node Name:** S  
  - **Type:** S3 (n8n-nodes-base.s3)  
  - **Role:** Lists all files in the specified S3 bucket.  
  - **Configuration:**  
    - Operation: Get All  
    - Bucket Name: `n8n`  
    - Return All: True (fetches all files instead of a limited subset)  
    - Options: Default (no filters or sorting)  
    - Credentials: Same AWS S3 credentials (`s3-n8n`)  
  - **Expressions/Variables:** None used; static bucket name.  
  - **Input/Output:** Input from S3 upload node; output is a JSON list of all bucket files (can be used for further processing or output).  
  - **Version Requirements:** Requires n8n supporting S3 list operation with “returnAll” parameter.  
  - **Potential Failures:** Authentication failures, bucket access errors, network issues, empty bucket returns empty list (not error).  
  - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role              | Input Node(s)         | Output Node(s) | Sticky Note                                                   |
|---------------------|-----------------------|-----------------------------|-----------------------|----------------|---------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger        | Manual start of workflow    | None                  | HTTP Request   |                                                               |
| HTTP Request        | HTTP Request          | Download file from URL       | On clicking 'execute' | S3             |                                                               |
| S3                  | S3                    | Upload downloaded file to S3 | HTTP Request          | S              | Uses S3 credentials named "s3-n8n".                          |
| S                   | S3                    | Retrieve list of all S3 files| S3                    | None           |                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Manual Trigger node:**
   - Name it "On clicking 'execute'".
   - Leave default settings.

3. **Add an HTTP Request node:**
   - Connect the Manual Trigger node’s output to the HTTP Request node’s input.
   - Set the node name to "HTTP Request".
   - Set "URL" parameter to `https://n8n.io/n8n-logo.png`.
   - Set "Response Format" to `File` to download the content as a binary file.
   - Leave other options default (no authentication or headers).

4. **Add an S3 node for upload:**
   - Connect the HTTP Request node’s output to this S3 node’s input.
   - Name this node "S3".
   - Set "Operation" to `Upload`.
   - Set "Bucket Name" to `n8n`.
   - For "File Name", use an expression to dynamically assign the filename from the HTTP Request node’s binary data:  
     `={{$node["HTTP Request"].binary.data.fileName}}`
   - Leave additional fields empty.
   - Under Credentials, select or create AWS S3 credentials named `s3-n8n`, configured with your AWS access key, secret, and region.

5. **Add another S3 node to list all files:**
   - Connect the output of the "S3" upload node to this new S3 node.
   - Rename it to "S".
   - Set "Operation" to `Get All`.
   - Set "Bucket Name" to `n8n`.
   - Set "Return All" to `true`.
   - Leave other options default.
   - Use the same AWS S3 credentials (`s3-n8n`).

6. **Activate and execute the workflow manually** via the "On clicking 'execute'" node to test the entire process.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                           |
|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow uses publicly available n8n logo image URL for demonstration. Replace with any valid URL as needed.      | https://n8n.io/n8n-logo.png                               |
| Credentials for AWS S3 must be configured in n8n with appropriate permissions to upload and list bucket contents.     | n8n Credentials Management                                |
| S3 node supports multiple operations; here specifically using "upload" and "getAll" for file management.              | Official n8n S3 node documentation                        |

---

This completes the comprehensive reference document for the "Upload a file and get a list of all the files in a bucket" workflow. It enables detailed understanding, stepwise reproduction, and anticipates typical integration issues.