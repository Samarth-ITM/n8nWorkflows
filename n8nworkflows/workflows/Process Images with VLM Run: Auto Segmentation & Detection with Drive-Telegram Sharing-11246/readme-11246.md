Process Images with VLM Run: Auto Segmentation & Detection with Drive-Telegram Sharing

https://n8nworkflows.xyz/workflows/process-images-with-vlm-run--auto-segmentation---detection-with-drive-telegram-sharing-11246


# Process Images with VLM Run: Auto Segmentation & Detection with Drive-Telegram Sharing

### 1. Workflow Overview

This workflow automates the processing of user-uploaded image files by leveraging two AI agents from VLM Run for segmentation and detection tasks. The processed images are then securely downloaded and shared across multiple platforms including Google Drive and Telegram. It is designed for use cases requiring automatic image segmentation and object detection with seamless multi-channel distribution, suitable for medical imaging, research, or automated reporting workflows.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** User uploads image or data files via a web form.
- **1.2 AI Processing with VLM Run Agents:** The uploaded file is sent in parallel to two separate VLM Run agents for segmentation and detection.
- **1.3 Webhook Reception & URL Extraction:** Each agent asynchronously returns results via webhooks; a code node extracts the full signed URL to the processed images.
- **1.4 Image Downloading:** The processed images are downloaded securely using the extracted signed URLs.
- **1.5 Multi-Channel Distribution:** Downloaded images are uploaded to Google Drive and sent to a Telegram chat for end-user access.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block allows users to upload files through an interactive form interface, initiating the workflow.

**Nodes Involved:**  
- Upload Image

**Node Details:**

- **Upload Image**  
  - Type: Form Trigger  
  - Role: Captures user file uploads via a web form.  
  - Configuration: Accepts files with `.pdf` or `.csv` extensions as required input. Form titled "Upload your data to test RAG".  
  - Inputs: External user submission via webhook.  
  - Outputs: Passes uploaded file data downstream.  
  - Edge Cases: Invalid file types, missing files, or upload failures may cause immediate termination or error.  
  - Version: 2.2  

#### 2.2 AI Processing with VLM Run Agents

**Overview:**  
This block forwards the uploaded file to two distinct VLM Run agents—one for segmenting objects in the image, the other for object detection—each operating asynchronously.

**Nodes Involved:**  
- VLM Run (Segmentation)  
- VLM Run (Detection)

**Node Details:**

- **VLM Run (Segmentation)**  
  - Type: VLM Run Agent Node  
  - Role: Sends the uploaded file to the segmentation AI agent.  
  - Configuration:  
    - Operation: executeAgent  
    - Agent Prompt: Request to segment all objects and return a completed URL of the segmented image.  
    - Callback URL: Webhook endpoint `/image_segmentation` to receive results asynchronously.  
  - Inputs: Triggered from "Upload Image" node.  
  - Outputs: None directly; results arrive later via webhook.  
  - Credentials: VLM Run API credentials required.  
  - Edge Cases: Agent timeout, malformed prompts, callback failures.  

- **VLM Run (Detection)**  
  - Type: VLM Run Agent Node  
  - Role: Sends the uploaded file to the detection AI agent.  
  - Configuration:  
    - Operation: executeAgent  
    - Agent Prompt: Request to detect all objects with bounding boxes and return a completed image URL.  
    - Callback URL: Webhook endpoint `/image_detection`.  
  - Inputs: Triggered concurrently with segmentation node.  
  - Outputs: None directly; results via webhook.  
  - Credentials: Same as segmentation node.  
  - Edge Cases: Same as above.

#### 2.3 Webhook Reception & URL Extraction

**Overview:**  
This block receives asynchronous webhook callbacks from the VLM Run agents, extracts the full signed URLs from the payloads, and prepares for secure downloading.

**Nodes Involved:**  
- Webhook for Segmented Image  
- Webhook for Detected Image  
- Code

**Node Details:**

- **Webhook for Segmented Image**  
  - Type: Webhook  
  - Role: Receives POST requests from VLM Run segmentation agent with the processed image URL.  
  - Configuration: Path set to `/image_segmentation`.  
  - Inputs: External HTTP POST from VLM Run.  
  - Outputs: Feeds data to the "Code" node.  
  - Edge Cases: Missing or malformed payload, unauthorized calls.

- **Webhook for Detected Image**  
  - Type: Webhook  
  - Role: Receives POST requests from VLM Run detection agent.  
  - Configuration: Path set to `/image_detection`.  
  - Inputs: External HTTP POST.  
  - Outputs: Feeds data to the "Code" node.  
  - Edge Cases: Same as above.

- **Code**  
  - Type: Code Node (JavaScript)  
  - Role: Parses webhook JSON payloads to extract the full signed Google Cloud Storage URL of the processed image.  
  - Configuration: Uses regex to match and capture the entire signed URL including query parameters to ensure valid authentication.  
  - Inputs: From both webhook nodes.  
  - Outputs: JSON objects containing only `{ url: fullSignedUrl }`.  
  - Edge Cases: Regex failures if URL format changes, missing URL in payload.

#### 2.4 Image Downloading

**Overview:**  
This block downloads the processed images from the signed URLs extracted by the code node.

**Nodes Involved:**  
- Download Image

**Node Details:**

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads image data securely from the full signed URL.  
  - Configuration: URL parameter dynamically set from incoming JSON `url`. No additional options specified.  
  - Inputs: From "Code" node.  
  - Outputs: Binary image data forwarded downstream.  
  - Edge Cases: Network timeouts, expired signed URLs, HTTP errors.

#### 2.5 Multi-Channel Distribution

**Overview:**  
The downloaded images are simultaneously uploaded to Google Drive and sent to a Telegram chat to facilitate accessible sharing.

**Nodes Involved:**  
- Upload File  
- Send Image

**Node Details:**

- **Upload File**  
  - Type: Google Drive  
  - Role: Uploads the downloaded image to a designated folder in Google Drive.  
  - Configuration:  
    - Target folder ID: `1S6baavqJn98MjUlbB6KtmARCWuWEekIZ` (a specific folder in "My Drive").  
    - File named "Patient Info".  
  - Inputs: Receives binary data from the "Download Image" node.  
  - Credentials: Google Drive OAuth2 credentials with upload permissions.  
  - Edge Cases: Auth token expiration, folder permissions, file size limits.

- **Send Image**  
  - Type: Telegram  
  - Role: Sends the image as a document to a configured Telegram chat ID.  
  - Configuration:  
    - Chat ID: `1872183963`  
    - Operation: sendDocument with binary data enabled.  
  - Inputs: Receives binary image data from "Download Image".  
  - Credentials: Telegram Bot API token.  
  - Edge Cases: Invalid chat ID, bot blocked, Telegram API rate limits.

---

### 3. Summary Table

| Node Name                 | Node Type                   | Functional Role                       | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                              |
|---------------------------|-----------------------------|------------------------------------|----------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| Upload Image              | Form Trigger                | Receives user file upload          | —                                | VLM Run (Segmentation), VLM Run (Detection) |                                                                                                        |
| VLM Run (Segmentation)    | VLM Run Agent               | Executes segmentation AI agent     | Upload Image                     | — (asynchronous webhook)       |                                                                                                        |
| VLM Run (Detection)       | VLM Run Agent               | Executes detection AI agent        | Upload Image                     | — (asynchronous webhook)       |                                                                                                        |
| Webhook for Segmented Image| Webhook                    | Receives segmented image results   | External (VLM Run Segment agent) | Code                          | Sticky Note1 (VLM Run Agents & Webhooks)                                                               |
| Webhook for Detected Image| Webhook                     | Receives detected image results    | External (VLM Run Detection agent)| Code                         | Sticky Note1 (VLM Run Agents & Webhooks)                                                               |
| Code                      | Code Node (JavaScript)      | Extracts full signed URL           | Webhook for Segmented Image, Webhook for Detected Image | Download Image              | Sticky Note2 (Code Node & Image Download)                                                              |
| Download Image            | HTTP Request                | Downloads processed image          | Code                            | Upload File, Send Image        | Sticky Note2 (Code Node & Image Download)                                                              |
| Upload File               | Google Drive                | Uploads image to Drive             | Download Image                  | —                             | Sticky Note3 (Upload to Drive & Send via Telegram)                                                     |
| Send Image                | Telegram                    | Sends image to Telegram chat       | Download Image                  | —                             | Sticky Note3 (Upload to Drive & Send via Telegram)                                                     |
| Sticky Note               | Sticky Note                 | Documentation and instructions     | —                                | —                             | Describes entire workflow purpose and overview                                                        |
| Sticky Note1              | Sticky Note                 | Describes VLM Run agents & webhooks| —                                | —                             | Covers VLM Run agents and webhook handling                                                            |
| Sticky Note2              | Sticky Note                 | Describes Code node and image download| —                              | —                             | Explains URL extraction and download security                                                         |
| Sticky Note3              | Sticky Note                 | Describes upload and Telegram send | —                                | —                             | Covers Drive upload and Telegram sharing                                                              |
| Sticky Note4              | Sticky Note                 | Empty (no content)                 | —                                | —                             |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Upload Image"):**  
   - Type: Form Trigger (version 2.2)  
   - Configure form title to "Upload your data to test RAG".  
   - Add a single required file field labeled "data" accepting `.pdf` and `.csv` files.  
   - Save the node.

2. **Create VLM Run Agent Nodes:**  
   - Add two VLM Run nodes (version 1) with credentials configured for VLM Run API.  
   - First node ("VLM Run (Segmentation)"):  
     - Set operation to "executeAgent".  
     - Agent prompt: "Segment all objects from the given input and generate segmented image. Gve only completed url of the generated segmented image for download."  
     - Agent callback URL: webhook path `/image_segmentation`.  
   - Second node ("VLM Run (Detection)"):  
     - Operation: "executeAgent".  
     - Agent prompt: "Detect all objects from the given input and generate tight bounding box image. Gve only completed url of the generated image for download."  
     - Callback URL: webhook path `/image_detection`.  
   - Connect "Upload Image" node output to both VLM Run nodes in parallel.

3. **Create Webhook Nodes for Callbacks:**  
   - Add webhook node "Webhook for Segmented Image" (version 2.1) with path `/image_segmentation` and HTTP method POST.  
   - Add webhook node "Webhook for Detected Image" (version 2.1) with path `/image_detection` and POST method.

4. **Create Code Node ("Code"):**  
   - Add a Code node (JavaScript, version 2).  
   - Insert the following code to extract the full signed URL from the webhook JSON payload:  
     ```js
     return $input.all().map(item => {
       const text = JSON.stringify(item.json);
       const regex = /(https?:\/\/[^\s"]+)/;
       const match = text.match(regex);
       const fullSignedUrl = match ? match[0] : null;
       return { json: { url: fullSignedUrl } };
     });
     ```  
   - Connect both webhook nodes' outputs to this Code node.

5. **Create HTTP Request Node ("Download Image"):**  
   - Type: HTTP Request (version 4.2).  
   - Set URL field to reference extracted URL: `={{ $json.url }}`.  
   - No extra options needed.  
   - Connect output of "Code" node to this node.

6. **Create Google Drive Upload Node ("Upload File"):**  
   - Type: Google Drive (version 3).  
   - Configure credentials with Google Drive OAuth2 having upload permissions.  
   - Set target folder ID to `1S6baavqJn98MjUlbB6KtmARCWuWEekIZ` (replace with your folder ID).  
   - Name file as "Patient Info" (or dynamic filename if desired).  
   - Connect output of "Download Image" node to this node.

7. **Create Telegram Send Node ("Send Image"):**  
   - Type: Telegram (version 1.2).  
   - Configure Telegram Bot credentials with proper token and access.  
   - Set Chat ID to `1872183963` (replace with your chat ID).  
   - Set operation to "sendDocument" and enable binary data sending.  
   - Connect output of "Download Image" node to this node.

8. **Ensure all nodes are linked and credentials are properly set.**  
   - The flow:  
     Upload Image → VLM Run Agents (Segmentation & Detection) → respective Webhooks → Code → Download Image → (Upload File and Send Image in parallel).

9. **Test the workflow:**  
   - Deploy and trigger the form upload, verify segmentation and detection results are returned, downloaded, saved to Drive, and sent to Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The workflow uses VLM Run AI agents for advanced image segmentation and detection via custom prompts.         | VLM Run documentation and API: https://vlm.run                                                             |
| The Google Drive folder ID and Telegram chat ID must be replaced with your own valid IDs for production use. | Google Drive API docs: https://developers.google.com/drive/api/v3/about-sdk                                  |
| Telegram Bot token requires proper setup and chat permissions.                                                | Telegram Bot API: https://core.telegram.org/bots/api                                                       |
| The Code node’s regex may require updates if the webhook payload format changes.                              | Regex designed to capture full signed URLs including query parameters for secure downloads.                 |
| The multi-agent asynchronous design allows parallel processing and faster results without blocking the workflow. | This design pattern is recommended for workflows handling multiple AI services concurrently.                |

---

**Disclaimer:**  
The provided workflow is generated from an automated n8n process and fully complies with content policies. It handles only legal and public data and does not include any sensitive or protected content.