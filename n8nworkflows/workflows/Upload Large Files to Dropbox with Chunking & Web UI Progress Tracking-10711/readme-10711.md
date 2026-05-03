Upload Large Files to Dropbox with Chunking & Web UI Progress Tracking

https://n8nworkflows.xyz/workflows/upload-large-files-to-dropbox-with-chunking---web-ui-progress-tracking-10711


# Upload Large Files to Dropbox with Chunking & Web UI Progress Tracking

### 1. Workflow Overview

This n8n workflow facilitates uploading large files to Dropbox using chunked uploading, combined with a web-based user interface (UI) to track upload progress in real time. It targets use cases where users need to upload files exceeding typical single-request size limits, ensuring reliability and user feedback through progressive chunk transmission.

The workflow is organized into the following logical blocks:

- **1.1 Web UI Serving Block:** Serves the upload page UI to users initiating the file upload.
- **1.2 Start Upload Session Block:** Handles requests to start a new Dropbox upload session and responds with a session ID.
- **1.3 Append Chunk Block:** Receives file chunks from the UI, appends them to the Dropbox upload session, and confirms success.
- **1.4 Finish Upload Session Block:** Finalizes the upload session by committing the uploaded chunks and signals completion.
- **1.5 Sticky Notes Documentation Block:** Contains multiple sticky notes providing inline documentation and UI design notes for clarity.

---

### 2. Block-by-Block Analysis

#### 2.1 Web UI Serving Block

- **Overview:**  
  This block serves the initial HTML page to the user that facilitates file selection and upload initiation.

- **Nodes Involved:**  
  - Serve Upload Page (Webhook)  
  - Respond with HTML (RespondToWebhook)

- **Node Details:**

  - **Serve Upload Page**  
    - Type: Webhook (HTTP Request Listener)  
    - Configuration: Listens at webhook ID `upload-ui` for incoming GET requests.  
    - Inputs: External HTTP requests from client browsers.  
    - Outputs: Triggers the next node to send the HTML response.  
    - Edge Cases: HTTP method other than GET could be unsupported; ensure CORS and security handled externally if needed.

  - **Respond with HTML**  
    - Type: RespondToWebhook (HTTP Response)  
    - Configuration: Sends the HTML content of the upload page back to the client.  
    - Inputs: Triggered by Serve Upload Page node.  
    - Outputs: HTTP response with HTML content.  
    - Edge Cases: Large HTML content size may affect response time; ensure HTML is properly sanitized and minimal.

---

#### 2.2 Start Upload Session Block

- **Overview:**  
  This block starts a new Dropbox upload session upon receiving a start request from the UI and returns a session ID needed for subsequent chunk uploads.

- **Nodes Involved:**  
  - Start Session Webhook (Webhook)  
  - Dropbox Start Session (HTTP Request)  
  - Respond Session ID (RespondToWebhook)

- **Node Details:**

  - **Start Session Webhook**  
    - Type: Webhook  
    - Configuration: Listens at webhook ID `start-session` for HTTP POST requests containing initial file metadata.  
    - Inputs: Incoming requests from UI signaling upload start.  
    - Outputs: Passes data to Dropbox Start Session node.  
    - Edge Cases: Missing or invalid metadata could cause errors.

  - **Dropbox Start Session**  
    - Type: HTTP Request  
    - Configuration: Uses Dropbox API to initiate an upload session (`/files/upload_session/start` endpoint).  
    - Credentials: Requires Dropbox OAuth2 credentials.  
    - Method: POST with relevant headers and payload.  
    - Inputs: Receives file metadata from Start Session Webhook.  
    - Outputs: Returns session ID from Dropbox API response.  
    - Edge Cases: Auth failures (expired token), network timeouts, API rate limits.

  - **Respond Session ID**  
    - Type: RespondToWebhook  
    - Configuration: Returns the Dropbox upload session ID to the UI to be used in chunk uploads.  
    - Inputs: Receives session ID from Dropbox Start Session.  
    - Outputs: HTTP response to UI.  
    - Edge Cases: Missing session ID in response or malformed JSON.

---

#### 2.3 Append Chunk Block

- **Overview:**  
  Handles reception of each uploaded chunk, appends it to the existing Dropbox upload session, and confirms successful append to the UI.

- **Nodes Involved:**  
  - Append Chunk Webhook (Webhook)  
  - Dropbox Append Chunk (HTTP Request)  
  - Respond Chunk OK (RespondToWebhook)

- **Node Details:**

  - **Append Chunk Webhook**  
    - Type: Webhook  
    - Configuration: Listens at webhook ID `append-chunk` for POST requests containing chunk data and session info.  
    - Inputs: Chunk data from UI.  
    - Outputs: Sends chunk data to Dropbox Append Chunk node.  
    - Edge Cases: Partial or corrupted chunk data, missing session info.

  - **Dropbox Append Chunk**  
    - Type: HTTP Request  
    - Configuration: Calls Dropbox API `/files/upload_session/append_v2` endpoint to append chunk.  
    - Credentials: Dropbox OAuth2 credentials required.  
    - Inputs: Receives chunk binary data and session ID.  
    - Outputs: Confirmation response.  
    - Edge Cases: Session expired, chunk out of order, API call failures.

  - **Respond Chunk OK**  
    - Type: RespondToWebhook  
    - Configuration: Sends an acknowledgment back to the UI confirming successful chunk append.  
    - Inputs: Triggered by Dropbox Append Chunk success.  
    - Outputs: HTTP response to UI.  
    - Edge Cases: Failure to confirm append may cause UI retries or errors.

---

#### 2.4 Finish Upload Session Block

- **Overview:**  
  Completes the upload session by committing all appended chunks into a single file in Dropbox, then notifies the UI of successful completion.

- **Nodes Involved:**  
  - Finish Session Webhook (Webhook)  
  - Dropbox Finish Session (HTTP Request)  
  - Respond Complete (RespondToWebhook)

- **Node Details:**

  - **Finish Session Webhook**  
    - Type: Webhook  
    - Configuration: Listens at webhook ID `finish-session` for POST requests signaling upload completion.  
    - Inputs: Final chunk info and session ID from UI.  
    - Outputs: Passes data to Dropbox Finish Session node.  
    - Edge Cases: Missing or incorrect session data.

  - **Dropbox Finish Session**  
    - Type: HTTP Request  
    - Configuration: Calls Dropbox API `/files/upload_session/finish` endpoint to commit the chunks.  
    - Credentials: Dropbox OAuth2 credentials required.  
    - Inputs: Receives session ID and final commit parameters.  
    - Outputs: Final file metadata from Dropbox API.  
    - Edge Cases: API errors, session already closed, network issues.

  - **Respond Complete**  
    - Type: RespondToWebhook  
    - Configuration: Sends upload completion confirmation back to the UI.  
    - Inputs: Triggered by Dropbox Finish Session success.  
    - Outputs: HTTP response to UI.  
    - Edge Cases: Failure to respond or delayed response may confuse UI state.

---

#### 2.5 Sticky Notes Documentation Block

- **Overview:**  
  This block consists of sticky notes scattered throughout the workflow providing contextual documentation and UI design notes.

- **Nodes Involved:**  
  - Main Documentation  
  - UI Background  
  - UI Section Header  
  - UI Detail  
  - Start Background  
  - Start Section Header  
  - Start Detail  
  - Append Background  
  - Append Section Header  
  - Append Detail  
  - Finish Background  
  - Finish Section Header  
  - Finish Detail  
  - Credentials Detail

- **Node Details:**

  - **Sticky Notes**  
    - Type: StickyNote  
    - Configuration: Each contains textual descriptions or UI-related comments.  
    - Inputs/Outputs: None; purely for documentation and visual aid.  
    - Edge Cases: None; purely informational.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                 | Input Node(s)           | Output Node(s)          | Sticky Note                                     |
|------------------------|----------------------|--------------------------------|------------------------|-------------------------|------------------------------------------------|
| Main Documentation     | StickyNote           | Documentation                  | -                      | -                       |                                                |
| UI Background          | StickyNote           | UI design notes                | -                      | -                       |                                                |
| UI Section Header      | StickyNote           | UI design notes                | -                      | -                       |                                                |
| UI Detail              | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Start Background       | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Start Section Header   | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Start Detail           | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Append Background      | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Append Section Header  | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Append Detail          | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Finish Background      | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Finish Section Header  | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Finish Detail          | StickyNote           | UI design notes                | -                      | -                       |                                                |
| Credentials Detail     | StickyNote           | Credentials information        | -                      | -                       |                                                |
| Serve Upload Page      | Webhook              | Serves upload page UI          | -                      | Respond with HTML       |                                                |
| Respond with HTML      | RespondToWebhook     | Sends upload page HTML         | Serve Upload Page       | -                       |                                                |
| Start Session Webhook  | Webhook              | Receives start session request | -                      | Dropbox Start Session   |                                                |
| Dropbox Start Session  | HTTP Request         | Starts Dropbox upload session  | Start Session Webhook   | Respond Session ID      | Requires Dropbox OAuth2 credentials             |
| Respond Session ID     | RespondToWebhook     | Returns session ID             | Dropbox Start Session   | -                       |                                                |
| Append Chunk Webhook   | Webhook              | Receives chunk data            | -                      | Dropbox Append Chunk    |                                                |
| Dropbox Append Chunk   | HTTP Request         | Appends chunk to Dropbox       | Append Chunk Webhook    | Respond Chunk OK        | Requires Dropbox OAuth2 credentials             |
| Respond Chunk OK       | RespondToWebhook     | Confirms chunk append          | Dropbox Append Chunk    | -                       |                                                |
| Finish Session Webhook | Webhook              | Receives finish session request| -                      | Dropbox Finish Session  |                                                |
| Dropbox Finish Session | HTTP Request         | Finishes upload session        | Finish Session Webhook  | Respond Complete        | Requires Dropbox OAuth2 credentials             |
| Respond Complete       | RespondToWebhook     | Confirms upload completion     | Dropbox Finish Session  | -                       |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Serve Upload Page" Node**  
   - Type: Webhook  
   - Webhook ID: `upload-ui`  
   - Method: GET  
   - Purpose: Listen for HTTP GET requests from UI clients requesting the upload page.

2. **Create "Respond with HTML" Node**  
   - Type: RespondToWebhook  
   - Connect input from "Serve Upload Page" node.  
   - Configure to respond with the HTML content of the upload page (file selection UI).  
   - Ensure valid HTML content is provided here.

3. **Create "Start Session Webhook" Node**  
   - Type: Webhook  
   - Webhook ID: `start-session`  
   - Method: POST  
   - Purpose: Receive requests to start a Dropbox upload session with file metadata.

4. **Create "Dropbox Start Session" Node**  
   - Type: HTTP Request  
   - Connect input from "Start Session Webhook" node.  
   - Set HTTP Method: POST  
   - URL: Dropbox API endpoint `/files/upload_session/start`  
   - Authentication: Use Dropbox OAuth2 credentials (setup in n8n credentials panel).  
   - Headers: Include `Dropbox-API-Arg` JSON with necessary parameters like close=false.  
   - Body: Binary data or empty as per Dropbox API for session start.  
   - Response: Capture session ID.

5. **Create "Respond Session ID" Node**  
   - Type: RespondToWebhook  
   - Connect input from "Dropbox Start Session" node.  
   - Configure to send back the session ID from Dropbox to the client.

6. **Create "Append Chunk Webhook" Node**  
   - Type: Webhook  
   - Webhook ID: `append-chunk`  
   - Method: POST  
   - Purpose: Receive chunk data and session info from the client.

7. **Create "Dropbox Append Chunk" Node**  
   - Type: HTTP Request  
   - Connect input from "Append Chunk Webhook" node.  
   - HTTP Method: POST  
   - URL: Dropbox API endpoint `/files/upload_session/append_v2`  
   - Authentication: Dropbox OAuth2 credentials.  
   - Headers: Include correct `Dropbox-API-Arg` with session ID and offset.  
   - Body: Binary chunk data from the webhook.  
   - Handle response for success/failure.

8. **Create "Respond Chunk OK" Node**  
   - Type: RespondToWebhook  
   - Connect input from "Dropbox Append Chunk" node.  
   - Configure to send HTTP 200 OK response confirming chunk append.

9. **Create "Finish Session Webhook" Node**  
   - Type: Webhook  
   - Webhook ID: `finish-session`  
   - Method: POST  
   - Purpose: Receive the final chunk info and commit request from client.

10. **Create "Dropbox Finish Session" Node**  
    - Type: HTTP Request  
    - Connect input from "Finish Session Webhook" node.  
    - HTTP Method: POST  
    - URL: Dropbox API endpoint `/files/upload_session/finish`  
    - Authentication: Dropbox OAuth2 credentials.  
    - Headers: Include `Dropbox-API-Arg` JSON with session ID, cursor, and commit info (path, mode).  
    - Body: Final chunk binary data if any.

11. **Create "Respond Complete" Node**  
    - Type: RespondToWebhook  
    - Connect input from "Dropbox Finish Session" node.  
    - Configure to send confirmation of upload completion to client.

12. **Add Sticky Notes**  
    - Add notes to document sections visually in the canvas to assist understanding and UI design.  
    - Use sticky notes with descriptive content at appropriate positions.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                   |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Dropbox API documentation is essential for understanding upload session endpoints and parameters. | https://www.dropbox.com/developers/documentation/http/documentation#files-upload_session-start |
| Dropbox OAuth2 credentials must be configured in n8n before running HTTP Request nodes.            | Configure under n8n credentials panel.            |
| The UI HTML page served should implement chunked file reading and POST each chunk to the append webhook. | Frontend implementation dependent, not included. |
| Handle network errors and retries on the client side to ensure reliable chunk upload.               | Best practice for large file uploads.             |

---

**Disclaimer:** The provided text is exclusively extracted from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.