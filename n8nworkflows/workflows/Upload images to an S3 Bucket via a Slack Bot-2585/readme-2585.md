Upload images to an S3 Bucket via a Slack Bot

https://n8nworkflows.xyz/workflows/upload-images-to-an-s3-bucket-via-a-slack-bot-2585


# Upload images to an S3 Bucket via a Slack Bot

### 1. Workflow Overview

This n8n workflow facilitates seamless uploading of public-facing images to a Cloudflare S3 bucket directly from Slack using modal interactions. It targets teams that rely on Slack for communication and need to manage image assets efficiently, such as DevOps, marketing, or content teams.

The workflow is logically divided into these blocks:

- **1.1 Slack Event Reception and Payload Parsing:** Receives Slack events via webhook and extracts useful payload data.
- **1.2 Slack Interaction Routing:** Routes incoming Slack interaction payloads (modal submissions, block actions) to appropriate processing branches.
- **1.3 Slack Modal Management:** Opens Slack modals for selecting upload options (new folder or existing folder) and collecting upload data.
- **1.4 Folder Decision Handling:** Based on user input, either presents a modal to create a new folder or select an existing folder.
- **1.5 File Processing Loop:** Splits submitted files and loops over them for individual processing.
- **1.6 File Download and Upload:** Downloads each file from Slack, uploads it to Cloudflare S3, and checks upload success.
- **1.7 Response Aggregation and Slack Notification:** Aggregates success/failure messages and posts a confirmation message back to Slack with file URLs.
- **1.8 Slack Webhook Response:** Sends appropriate HTTP responses back to Slack to acknowledge event receipt and close modals.

---

### 2. Block-by-Block Analysis

#### 1.1 Slack Event Reception and Payload Parsing

- **Overview:**  
Receives incoming Slack events via webhook and extracts the relevant `payload` object to a structured JSON format for further processing.

- **Nodes Involved:**  
  - Webhook  
  - Parse Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point, listens for POST requests on `/slack-image-upload-bot` path.  
    - Config: No special authentication; configured as Slack event subscription webhook.  
    - Inputs: External HTTP POST from Slack Events API.  
    - Outputs: Raw payload including `body.payload` as URL-encoded JSON string.

  - **Parse Webhook**  
    - Type: Set  
    - Role: Extracts and parses the Slack `payload` JSON string from the webhook body.  
    - Config: Assigns `$json.body.payload` to `response` as a parsed object.  
    - Inputs: Webhook output.  
    - Outputs: Parsed payload under `response` field for downstream nodes.  
    - Edge Cases: Malformed or missing payload may cause parsing errors.

- **Sticky Notes:**  
  Explains Slack Events API setup and the parsing step.

---

#### 1.2 Slack Interaction Routing

- **Overview:**  
Routes Slack interaction payloads based on their `callback_id` and interaction type (`block_actions`, `view_submission`, etc.) to appropriate workflow branches.

- **Nodes Involved:**  
  - Route Message (Switch)  
  - Route Action (Switch)  
  - Route Message1 (Switch)

- **Node Details:**  
  - **Route Message**  
    - Type: Switch  
    - Role: Directs incoming Slack interactions based on `response.callback_id` and `response.type`.  
    - Config:  
      - Output "Idea Selector" if `callback_id == "idea_selector"`.  
      - Output "Block Action" if `response.type == "block_actions"`.  
      - Output "Submit Data" if `response.type == "view_submission"`.  
    - Inputs: Parsed webhook response.  
    - Outputs: Routes to success response, no action, or modal close nodes.

  - **Route Action**  
    - Type: Switch  
    - Role: Routes block_actions type messages based on a condition `t == f` (likely a placeholder or specific flag).  
    - Inputs: From modal close response node.  
    - Outputs: To split file processing node or default.

  - **Route Message1**  
    - Type: Switch  
    - Role: Routes modal submission data based on user's folder selection in the modal (`createfolder` vs `selectfolder`).  
    - Inputs: From "Respond to Slack Webhook - No Action".  
    - Outputs: To "Create Folder" modal or "Select Folder" modal nodes.

- **Edge Cases:**  
  Incorrect or missing fields in payload may cause messages to be routed incorrectly or dropped.

- **Sticky Notes:**  
  Detailed explanation on Slack interaction routing, modal management, and ensuring proper responses.

---

#### 1.3 Slack Modal Management

- **Overview:**  
Handles opening Slack modals for file upload initiation and collects user input for folder selection or creation.

- **Nodes Involved:**  
  - Idea Selector Modal (HTTP Request)  
  - Create Folder (HTTP Request)  
  - Select Folder (HTTP Request)  
  - Respond to Slack Webhook - Success (Respond to Webhook)  
  - Respond to Slack Webhook - No Action (Respond to Webhook)  
  - Close Modal Popup (Respond to Webhook)

- **Node Details:**  
  - **Idea Selector Modal**  
    - Type: HTTP Request  
    - Role: Opens initial Slack modal for user to choose upload options (new or existing folder).  
    - Config: POST to `https://slack.com/api/views.open` with JSON modal view containing radio buttons and instructions.  
    - Inputs: Trigger ID from parsed webhook.  
    - Credentials: Slack API token with `views:write` permission.  
    - Edge Cases: Invalid trigger ID or Slack API errors.

  - **Create Folder**  
    - Type: HTTP Request  
    - Role: Opens modal prompting user to enter new folder name and upload files.  
    - Config: POST to `https://slack.com/api/views.push` with modal view including folder name input and file input (max 10 files, jpg/png/pdf).  
    - Inputs: Trigger ID from parsed webhook.  
    - Credentials: Slack API token.

  - **Select Folder**  
    - Type: HTTP Request  
    - Role: Opens modal with folder selector (external select) and file upload input for choosing existing folder.  
    - Config: Similar to Create Folder but with external select for folder names.  
    - Inputs: Trigger ID from parsed webhook.  
    - Credentials: Slack API token.

  - **Respond to Slack Webhook - Success / No Action / Close Modal Popup**  
    - Type: Respond to Webhook  
    - Role: Send HTTP response back to Slack to acknowledge events and close modals as needed.  
    - Config: JSON response with `response_action` to clear modals or no data for acknowledgments.

- **Sticky Notes:**  
  Screenshots and links to Slack modal documentation; reminders about Slack credential usage.

- **Edge Cases:**  
  Slack API rate limits; errors in modal payloads; missing or expired trigger IDs.

---

#### 1.4 Folder Decision Handling

- **Overview:**  
Determines user choice for folder creation or selection and pushes appropriate Slack modal for further input.

- **Nodes Involved:**  
  - Route Message1 (Switch)  
  - Create Folder (HTTP Request)  
  - Select Folder (HTTP Request)

- **Node Details:**  
  This block follows from routing logic and branches into either the "Create Folder" modal or the "Select Folder" modal based on the user's radio button choice.

- **Edge Cases:**  
  User selecting an invalid option or modal failing to push.

---

#### 1.5 File Processing Loop

- **Overview:**  
Splits multiple uploaded files into individual items and loops through them to process each file independently.

- **Nodes Involved:**  
  - Split Out Files (SplitOut)  
  - Loop Over Items (SplitInBatches)  
  - No Operation, do nothing (NoOp)

- **Node Details:**  
  - **Split Out Files**  
    - Type: SplitOut  
    - Role: Extracts array of file objects from modal submission (`response.view.state.values.input_block_file.file_input_action.files`) and emits each as a separate item.  
    - Inputs: Output from "Route Action".  
    - Outputs: Single file objects.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes files one by one or in batches for controlled sequential handling.  
    - Inputs: From "Split Out Files".  
    - Outputs: To file download or skip nodes.

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Placeholder node for skipping processing if needed.  
    - Inputs: From "Loop Over Items" on failure path.  
    - Outputs: To aggregation.

- **Sticky Notes:**  
  Explains file splitting and loop processing.

- **Edge Cases:**  
  Empty file list; processing large batches causing timeouts.

---

#### 1.6 File Download and Upload

- **Overview:**  
Downloads each file binary from Slack file URL and uploads it to Cloudflare S3 bucket under the specified folder path.

- **Nodes Involved:**  
  - Download File Binary (HTTP Request)  
  - Upload to S3 Bucket (S3)  
  - Check if uploaded successfully (If)  
  - Success Response (Set)  
  - Failure Response (Set)  
  - move on to next (NoOp)

- **Node Details:**  
  - **Download File Binary**  
    - Type: HTTP Request  
    - Role: Downloads file binary using private Slack file download URL (`url_private_download`).  
    - Config: Uses Slack API credentials with access token for authorization.  
    - Outputs: Binary file data.  
    - Edge Cases: Auth errors, file deleted or inaccessible, network timeouts.

  - **Upload to S3 Bucket**  
    - Type: S3  
    - Role: Uploads downloaded file binary to Cloudflare S3 bucket `n8n-uploads`.  
    - Config: Filename constructed dynamically from folder name (new or existing) and sanitized file name.  
    - Credentials: Cloudflare S3 API credentials with write access.  
    - Edge Cases: Upload failures, permission issues, bucket not found.

  - **Check if uploaded successfully**  
    - Type: If  
    - Role: Checks boolean `success` flag from upload result to determine success or failure path.  
    - Outputs: To success or failure response nodes.

  - **Success Response**  
    - Type: Set  
    - Role: Creates Slack message block with uploaded file URL (public URL on `uploads.n8n.io`).  
    - Output: Slack-formatted message string for aggregation.

  - **Failure Response**  
    - Type: Set  
    - Role: Creates Slack message block warning about failed upload with URL placeholder.  
    - Output: Slack-formatted failure message for aggregation.

  - **move on to next**  
    - Type: NoOp  
    - Role: Moves the loop forward after each file's upload result.

- **Sticky Notes:**  
  Explains per-file upload processing and success/failure reporting.

---

#### 1.7 Response Aggregation and Slack Notification

- **Overview:**  
Aggregates individual file upload responses into a single Slack message and posts it to a specified Slack channel.

- **Nodes Involved:**  
  - Aggregate (Aggregate)  
  - Post Image to Channel (Slack)

- **Node Details:**  
  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects all `slackresponse` messages from processed files into a combined list for final notification.  
    - Config: Merges all slackresponse strings without merging lists.

  - **Post Image to Channel**  
    - Type: Slack  
    - Role: Posts a message to a configured Slack channel with folder name and all uploaded file URLs.  
    - Config: Uses Slack API credentials; posts a block message with folder name and aggregated file links.  
    - Channel ID: Hardcoded `"C081EHWKKH6"` (should be customized).  
    - Edge Cases: Slack API errors, invalid channel ID.

- **Sticky Notes:**  
  Explains aggregation and final Slack notification.

---

#### 1.8 Slack Webhook Response

- **Overview:**  
Sends HTTP responses to Slack to acknowledge the receipt of events and close modals as needed, ensuring Slack does not timeout or retry.

- **Nodes Involved:**  
  - Respond to Slack Webhook - Success (Respond to Webhook)  
  - Respond to Slack Webhook - No Action (Respond to Webhook)  
  - Close Modal Popup (Respond to Webhook)

- **Node Details:**  
  - Respond with appropriate JSON or empty response, depending on the Slack interaction type being handled.  
  - Close Modal Popup sends JSON `{"response_action": "clear"}` to close Slack modal on submission.

- **Edge Cases:**  
  Missing or late responses cause Slack to retry or show errors.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                     | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                 |
|-------------------------------|---------------------|----------------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook             | Receives Slack event POST requests                  |                             | Parse Webhook               | Explains Slack Events API trigger and payload extraction                                                   |
| Parse Webhook                 | Set                 | Parses Slack JSON payload from webhook body         | Webhook                     | Route Message               | Explains payload extraction                                                                                |
| Route Message                 | Switch              | Routes Slack interactions by callback_id/type       | Parse Webhook               | Respond to Slack Webhook - Success,<br>Respond to Slack Webhook - No Action,<br>Close Modal Popup | Explains Slack interaction routing and modal management                                                    |
| Respond to Slack Webhook - Success | Respond to Webhook | Sends success HTTP response to Slack                 | Route Message               | Idea Selector Modal         |                                                                                                             |
| Respond to Slack Webhook - No Action | Respond to Webhook | Sends no-data HTTP response to Slack                 | Route Message               | Route Message1              |                                                                                                             |
| Close Modal Popup             | Respond to Webhook   | Closes Slack modal popup                              | Route Message               | Route Action                |                                                                                                             |
| Idea Selector Modal           | HTTP Request        | Opens initial Slack modal for folder type selection | Respond to Slack Webhook - Success |                             | Reminder to configure Slack credentials                                                                    |
| Route Action                 | Switch              | Routes based on block action condition               | Close Modal Popup           | Split Out Files             |                                                                                                             |
| Split Out Files               | SplitOut            | Splits uploaded files array into individual items   | Route Action                | Loop Over Items             | Explains file splitting for processing                                                                     |
| Loop Over Items               | SplitInBatches      | Processes each file individually                      | Split Out Files             | No Operation, do nothing,<br>Download File Binary     | Explains looping through files to upload                                                                     |
| No Operation, do nothing      | NoOp                | Placeholder for skipping processing                   | Loop Over Items             | Aggregate                   |                                                                                                             |
| Download File Binary          | HTTP Request        | Downloads file binary from Slack                      | Loop Over Items             | Upload to S3 Bucket         |                                                                                                             |
| Upload to S3 Bucket           | S3                  | Uploads file to Cloudflare S3 bucket                  | Download File Binary        | Check if uploaded successfully |                                                                                                             |
| Check if uploaded successfully | If                  | Branches on upload success/failure                    | Upload to S3 Bucket         | Success Response,<br>Failure Response |                                                                                                             |
| Success Response              | Set                 | Prepares Slack success message block for file URL    | Check if uploaded successfully | move on to next             |                                                                                                             |
| Failure Response              | Set                 | Prepares Slack failure message block for file URL    | Check if uploaded successfully | move on to next             |                                                                                                             |
| move on to next              | NoOp                | Continues loop processing                              | Success Response,<br>Failure Response | Loop Over Items             |                                                                                                             |
| Aggregate                    | Aggregate           | Aggregates all Slack message blocks for final post   | No Operation, do nothing    | Post Image to Channel       | Combines success and failure messages for final Slack notification                                         |
| Post Image to Channel        | Slack               | Posts confirmation with uploaded file URLs to Slack | Aggregate                  |                             |                                                                                                             |
| Route Message1               | Switch              | Routes folder type selection modal submission         | Respond to Slack Webhook - No Action | Create Folder,<br>Select Folder |                                                                                                             |
| Create Folder               | HTTP Request        | Opens modal for creating new upload folder            | Route Message1              |                             |                                                                                                             |
| Select Folder               | HTTP Request        | Opens modal for selecting existing upload folder      | Route Message1              |                             |                                                                                                             |
| Sticky Note                 | Sticky Note          | Explains Slack Events webhook trigger and parsing    |                             |                             | See detailed notes in workflow overview                                                                   |
| Sticky Note15               | Sticky Note          | Explains Slack interaction routing and modal handling |                             |                             | See detailed notes in workflow overview                                                                   |
| Sticky Note11               | Sticky Note          | Explains Slack modal popup usage                       |                             |                             | See detailed notes in workflow overview                                                                   |
| Sticky Note2                | Sticky Note          | Reminder about Slack credentials configuration         |                             |                             |                                                                                                             |
| Sticky Note1                | Sticky Note          | Explains split file processing                         |                             |                             |                                                                                                             |
| Sticky Note3                | Sticky Note          | Explains per file upload looping                       |                             |                             |                                                                                                             |
| Sticky Note4                | Sticky Note          | Explains aggregation of success/failure messages      |                             |                             |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Set HTTP Method: POST  
   - Path: `slack-image-upload-bot`  
   - Purpose: Receive Slack events and interactions via Events API.  

2. **Add a Set Node ("Parse Webhook")**  
   - Extract `body.payload` from webhook JSON body.  
   - Assign it to a new field `response` as parsed JSON.  

3. **Add a Switch Node ("Route Message")**  
   - Define rules:  
     - If `response.callback_id == "idea_selector"` → Output "Idea Selector".  
     - If `response.type == "block_actions"` → Output "Block Action".  
     - If `response.type == "view_submission"` → Output "Submit Data".  
   - Connect "Parse Webhook" to this node.

4. **Create Respond to Webhook Nodes for Slack Acknowledgment**  
   - "Respond to Slack Webhook - Success": No data response.  
   - "Respond to Slack Webhook - No Action": No data response.  
   - "Close Modal Popup": Respond with JSON `{ "response_action": "clear" }` to close modals.  

5. **From "Route Message" node:**  
   - Connect "Idea Selector" output to "Respond to Slack Webhook - Success".  
   - Connect "Submit Data" output to "Close Modal Popup".  
   - Connect "Block Action" output to "Respond to Slack Webhook - No Action".

6. **Add HTTP Request Node ("Idea Selector Modal")**  
   - POST to `https://slack.com/api/views.open`.  
   - Body: Slack modal JSON with radio buttons for folder choice and instructions.  
   - Use Slack credentials with permissions: `commands`, `files:read`, `files:write`, `chat:write`, `views:write`.  
   - Input: Use `trigger_id` from `Parse Webhook.response.trigger_id`.  

7. **Add Switch Node ("Route Message1")**  
   - Routes folder choice from modal submission:  
     - If `folder-type` value equals `createfolder` → Output "Create Folder".  
     - If `folder-type` value equals `selectfolder` → Output "Select Folder".  

8. **Add HTTP Request Nodes for Folder Modals:**  
   - "Create Folder": POST to `https://slack.com/api/views.push` with modal for folder name input and file uploads.  
   - "Select Folder": POST to `https://slack.com/api/views.push` with external folder select and file uploads.  
   - Use Slack API credentials.  
   - Input: Use `trigger_id` from parsed webhook.  

9. **Add Switch Node ("Route Action")**  
   - Route interactions after modal close based on some condition (e.g., specific flag).  
   - Output to file processing.  

10. **Add SplitOut Node ("Split Out Files")**  
    - Split the array of uploaded files from modal submission (`response.view.state.values.input_block_file.file_input_action.files`).  

11. **Add SplitInBatches Node ("Loop Over Items")**  
    - Process files in batches or individually.  
    - Output to next nodes.  

12. **Add NoOp Node ("No Operation, do nothing")**  
    - Placeholder for skipping files if needed.  
    - Connect as failure output from "Loop Over Items" or upload checks.  

13. **Add HTTP Request Node ("Download File Binary")**  
    - Download file binary from Slack `url_private_download`.  
    - Use Slack API token with access permissions.  

14. **Add S3 Node ("Upload to S3 Bucket")**  
    - Upload file binary to configured Cloudflare S3 bucket.  
    - Filename constructed as `<folder>/<filename>` with spaces replaced by underscores.  
    - Use Cloudflare S3 credentials with write access.  

15. **Add If Node ("Check if uploaded successfully")**  
    - Check upload success flag from S3 node output.  
    - Route to success/failure response nodes.  

16. **Add Set Nodes ("Success Response" and "Failure Response")**  
    - Prepare Slack message blocks with file URLs or failure warning.  

17. **Add NoOp Node ("move on to next")**  
    - Continue loop processing after each file.  

18. **Add Aggregate Node ("Aggregate")**  
    - Collect all Slack message blocks from success/failure nodes into one combined message.  

19. **Add Slack Node ("Post Image to Channel")**  
    - Post final aggregated message to Slack channel (configure channel ID).  
    - Use Slack API credentials.  

20. **Connect Workflow:**  
    - Chain nodes according to logical flow described above.  
    - Ensure all nodes have correct inputs and outputs.  

21. **Credentials Setup:**  
    - Setup Slack API credentials with required scopes: `commands`, `files:write`, `files:read`, `chat:write`, `views:write`.  
    - Setup Cloudflare S3 credentials with API token for bucket access.  

22. **Testing:**  
    - Activate workflow.  
    - Configure Slack app with Events API pointing to webhook URL.  
    - Test modal triggering and file uploads from Slack.  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                            |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Slack Events API Setup: https://api.slack.com/apis/connections/events-api                                             | Slack official documentation                               |
| Slack Modals Documentation: https://api.slack.com/surfaces/modals                                                    | Slack modals details                                       |
| Cloudflare S3 Setup Guide: https://developers.cloudflare.com/r2/                                                     | Cloudflare R2 S3-compatible storage                        |
| n8n Documentation: https://docs.n8n.io/                                                                               | n8n official docs                                          |
| Slack Bot Permissions Needed: commands, files:write, files:read, chat:write, views:write                               | Slack app configuration                                   |
| Workflow designed for teams needing efficient image uploads to public CDN via Slack                                   | Use case summary                                           |
| Sticky notes embedded in workflow explain node configuration and provide UI screenshots                               | Helpful for workflow customization                         |
| The Slack channel ID for posting results is hardcoded and should be customized                                        | Modify in "Post Image to Channel" node                     |

---

This structured documentation enables advanced users and automation agents to fully understand, reproduce, and modify the workflow, while anticipating key error cases and integration points.