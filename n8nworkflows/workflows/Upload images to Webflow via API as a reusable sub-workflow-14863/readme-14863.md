Upload images to Webflow via API as a reusable sub-workflow

https://n8nworkflows.xyz/workflows/upload-images-to-webflow-via-api-as-a-reusable-sub-workflow-14863


# Upload images to Webflow via API as a reusable sub-workflow

### 1. Workflow Overview

The **Upload images to Webflow via API** workflow is designed as a reusable sub-workflow. Its primary purpose is to automate the process of uploading binary image files to the Webflow Asset Manager using the Webflow API v2. 

Webflow requires a two-step upload process: first, announcing the file to create a placeholder and receive a signed S3 upload URL; second, uploading the actual binary data to that S3 bucket.

The workflow is divided into three functional blocks:
- **1.1 Trigger & Input:** Reception of binary data and site metadata.
- **1.2 AI/Data Processing:** Generation of the file hash and the two-stage upload sequence (Webflow Announcement $\rightarrow$ S3 Upload).
- **1.3 Output Generation:** Extraction and return of the final asset details.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input
**Overview:** This block serves as the entry point for the sub-workflow, defining the required parameters that must be passed from the parent workflow.

- **Nodes Involved:** `ARGS`
- **Node Details:**
  - **ARGS (Execute Workflow Trigger):** 
    - **Technical Role:** Sub-workflow entry point.
    - **Configuration:** Expects three input variables: `wfSiteId` (String), `imageData` (Object/Binary), and `fileName` (String).
    - **Input/Output:** Receives data from a parent "Execute Sub-Workflow" node; outputs to the Crypto node.

#### 2.2 Upload to Webflow
**Overview:** This is the core logic block that handles the security hashing of the file and the two-step API handshake required by Webflow.

- **Nodes Involved:** `Crypto`, `Step A – Announce asset in Webflow`, `Step B – Upload binary to S3`
- **Node Details:**
  - **Crypto:**
    - **Technical Role:** Data transformation.
    - **Configuration:** Uses **MD5** hashing on the binary data provided in `imageData`.
    - **Key Expression:** `{{ $('ARGS').item.json.imageData }}`.
    - **Output:** A hash string named `imageDataHash`.
  - **Step A – Announce asset in Webflow (HTTP Request):**
    - **Technical Role:** API Call to Webflow.
    - **Configuration:** POST request to `https://api.webflow.com/v2/sites/{{wfSiteId}}/assets`.
    - **Authentication:** Bearer Auth (via predefined credentials).
    - **Body:** JSON containing `fileName` and the MD5 `fileHash`.
    - **Potential Failures:** Invalid API token, incorrect Site ID, or Webflow API downtime.
  - **Step B – Upload binary to S3 (HTTP Request):**
    - **Technical Role:** Binary file transfer.
    - **Configuration:** POST request using `multipart-form-data`.
    - **URL:** Dynamically set to the `uploadUrl` returned by Step A.
    - **Key Expressions:** Maps multiple S3 parameters (ACL, Bucket, Policy, Signature, etc.) from the output of Step A.
    - **Binary Handling:** Sends the actual file binary from the `ARGS` node.
    - **Potential Failures:** S3 timeout, expired signed URL, or mismatched content-type.

#### 2.3 Return Values
**Overview:** This block cleans up the data and returns only the necessary identifiers to the parent workflow.

- **Nodes Involved:** `RETURN`
- **Node Details:**
  - **RETURN (Set):**
    - **Technical Role:** Data formatting.
    - **Configuration:** Creates a clean JSON object containing:
      - `originalFileName`: Passed from `ARGS`.
      - `fileId`: The ID of the asset created in Webflow.
      - `assetUrl`: The final public URL of the uploaded asset.
    - **Input/Output:** Receives confirmation from Step B; outputs the final result to the parent workflow.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ARGS | Execute Workflow Trigger | Input Reception | - | Crypto | ## 1. Trigger & Input |
| Crypto | Crypto | MD5 Hash Generation | ARGS | Step A | ## 2. Upload to Webflow |
| Step A – Announce asset in Webflow | HTTP Request | Register Asset with Webflow | Crypto | Step B | ## 2. Upload to Webflow |
| Step B – Upload binary to S3 | HTTP Request | Upload Binary to S3 | Step A | RETURN | ## 2. Upload to Webflow |
| RETURN | Set | Format Final Output | Step B | - | ## 3. Return values |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    - Create an **Execute Workflow Trigger** node named `ARGS`.
    - Add three input variables: `wfSiteId` (String), `imageData` (Object), and `fileName` (String).

2.  **Hash Generation:**
    - Create a **Crypto** node.
    - Set Action to **MD5**.
    - Set the input to the `imageData` binary property from the `ARGS` node.
    - Set the output property name to `imageDataHash`.

3.  **Webflow Registration:**
    - Create an **HTTP Request** node named `Step A – Announce asset in Webflow`.
    - **Method:** `POST`.
    - **URL:** `https://api.webflow.com/v2/sites/{{ $('ARGS').item.json.wfSiteId }}/assets`.
    - **Authentication:** Set to `Header Auth` or `Bearer Auth` with your Webflow API Token.
    - **Body Parameters (JSON):**
      - `fileName`: `{{ $('ARGS').item.json.fileName }}`
      - `fileHash`: `{{ $('Crypto').item.json.imageDataHash }}`

4.  **S3 Binary Upload:**
    - Create an **HTTP Request** node named `Step B – Upload binary to S3`.
    - **Method:** `POST`.
    - **URL:** `{{ $('Step A – Announce asset in Webflow').item.json.uploadUrl }}`.
    - **Content Type:** `multipart-form-data`.
    - **Body Parameters:** Add all the following fields using values from `Step A` (under `uploadDetails`):
      - `acl`, `bucket`, `X-Amz-Algorithm`, `X-Amz-Credential`, `X-Amz-Date`, `key`, `Policy`, `X-Amz-Signature`, `success_action_status`, `content-type`, `Cache-Control`.
    - **Binary Data:** Select "Form Binary Data" and map the file from the `ARGS` node binary data.

5.  **Final Output:**
    - Create a **Set** node named `RETURN`.
    - Add the following string assignments:
      - `originalFileName` $\rightarrow$ `{{ $('ARGS').item.json.fileName }}`
      - `fileId` $\rightarrow$ `{{ $('Step A – Announce asset in Webflow').item.json.id }}`
      - `assetUrl` $\rightarrow$ `{{ $('Step A – Announce asset in Webflow').item.json.assetUrl }}`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built by Oliver Engel | [ENGELdevelopment Homepage](https://www.engeldevelopment.de/) |
| Author Contact | [LinkedIn Profile](https://www.linkedin.com/in/oliver-engel-bb1984212) |
| Customization | To upload to a specific folder, add `folderId` to the JSON body of Step A. |
| Versatility | While designed for images, this logic supports any file type accepted by the Webflow Asset API (PDFs, Videos). |