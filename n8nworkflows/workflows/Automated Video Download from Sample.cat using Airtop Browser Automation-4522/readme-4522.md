Automated Video Download from Sample.cat using Airtop Browser Automation

https://n8nworkflows.xyz/workflows/automated-video-download-from-sample-cat-using-airtop-browser-automation-4522


# Automated Video Download from Sample.cat using Airtop Browser Automation

### 1. Workflow Overview

This workflow automates the download of a video file from the website [Sample.cat](https://sample.cat/en/webm) using Airtop Browser Automation within n8n. It is designed for users who want to programmatically retrieve media files from web pages without manual intervention, useful for bulk downloads, backups, or dataset compilation.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger:** Initiates the workflow manually by user action.
- **1.2 Airtop Browser Session Management:** Starts and eventually terminates a browser session to interact with the target webpage.
- **1.3 Web Navigation & Interaction:** Opens the specified URL and simulates clicking the download button for a specific video file.
- **1.4 File Availability & Download:** Waits for the file to become ready, checks its availability, then downloads it.
- **1.5 Cleanup:** Terminates the browser session to release resources.

Each block is connected sequentially, ensuring a robust and linear flow from trigger to file download completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger

**Overview:**  
This block provides the entry point to start the workflow manually, enabling user-controlled execution.

**Nodes Involved:**  
- When clicking ‘Test workflow’

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user action via n8n UI.  
  - Configuration: No parameters; default manual trigger.  
  - Inputs: None  
  - Outputs: Connects to the "Session" node to start the browser session.  
  - Edge Cases: None specific; workflow won't start unless triggered.  

---

#### 2.2 Airtop Browser Session Management

**Overview:**  
Manages the lifecycle of the Airtop browser session used to navigate and interact with the webpage.

**Nodes Involved:**  
- Session  
- Terminate  

**Node Details:**  

- **Session**  
  - Type: Airtop Node (Session resource)  
  - Role: Initializes a new Airtop browser session, providing a session ID for downstream interactions.  
  - Configuration: Default with Airtop API credentials; no additional parameters.  
  - Inputs: Triggered by the manual trigger node.  
  - Outputs: Connects to "Window" to open the webpage and also to "Terminate" for session cleanup.  
  - Credentials: Airtop API key (official organization).  
  - Edge Cases: API key invalidity, network issues causing session creation failure.  

- **Terminate**  
  - Type: Airtop Node (Session resource)  
  - Role: Ends the Airtop browser session to free resources.  
  - Configuration: Operation set to "terminate".  
  - Inputs: Connected from "Session" node (parallel branch), triggered after the main operations complete.  
  - Outputs: None (end of session lifecycle).  
  - Credentials: Airtop API key.  
  - Edge Cases: Session already closed or invalid session ID may cause termination failure but typically safe to ignore.  

---

#### 2.3 Web Navigation & Interaction

**Overview:**  
Navigates to the target webpage and simulates the user interaction needed to trigger the video file download.

**Nodes Involved:**  
- Window  
- Click on download button  

**Node Details:**  

- **Window**  
  - Type: Airtop Node (Window resource)  
  - Role: Opens the browser window at the specified URL.  
  - Configuration:  
    - URL: `https://sample.cat/en/webm`  
    - Resource: Window  
    - Options: Live view enabled, resizing disabled for consistent UI.  
  - Inputs: From "Session" node.  
  - Outputs: Connects to "Click on download button".  
  - Credentials: Airtop API key.  
  - Edge Cases: Navigation failure due to network issues, URL changes, or site downtime.  

- **Click on download button**  
  - Type: Airtop Node (Interaction resource)  
  - Role: Simulates a click on the blue download button associated with the video titled “SD 640x360 (Seawater, drone view video, 30 FPS)”.  
  - Configuration:  
    - Resource: Interaction  
    - Element Description: Describes the UI element to click (blue download button for the specific video).  
  - Inputs: From "Window".  
  - Outputs: Connects to "Wait".  
  - Credentials: Airtop API key.  
  - Edge Cases: Button not found if page layout changes, button not visible, or interaction timing issues.  

---

#### 2.4 File Availability & Download

**Overview:**  
Waits for the file to become available, retrieves metadata to confirm readiness, and downloads the video file.

**Nodes Involved:**  
- Wait  
- Get file data  
- Download file  
- Sticky Note (explaining wait strategy)  

**Node Details:**  

- **Wait**  
  - Type: Wait Node  
  - Role: Pauses workflow execution for 10 seconds to allow file processing on the server side.  
  - Configuration: Amount set to 10 seconds.  
  - Inputs: From "Click on download button".  
  - Outputs: Connects to "Get file data".  
  - Edge Cases: Fixed wait may be insufficient or excessive; better approach is looping with status check (not implemented).  

- **Get file data**  
  - Type: Airtop Node (File resource)  
  - Role: Retrieves metadata for files available in the current session, limited to the latest one.  
  - Configuration:  
    - Limit: 1  
    - Resource: File  
    - Session ID: Dynamically uses the session ID from the "Session" node output.  
    - OutputSingleItem: false (returns array of files).  
  - Inputs: From "Wait".  
  - Outputs: Connects to "Download file".  
  - Credentials: Airtop API key.  
  - Edge Cases: No files found, session ID invalid, or API call failure.  

- **Download file**  
  - Type: Airtop Node (File resource)  
  - Role: Downloads the file as binary data using the file ID obtained from "Get file data".  
  - Configuration:  
    - Operation: get  
    - File ID: Dynamically set from the `$json.id` of the previous node's output.  
    - OutputBinaryFile: true (downloads file content).  
  - Inputs: From "Get file data".  
  - Outputs: None (file is downloaded as binary output).  
  - Credentials: Airtop API key.  
  - Edge Cases: File ID invalid, download failure, or binary output issues.  

- **Sticky Note** (associated with this block)  
  - Content: Advises waiting a few seconds to ensure the file is ready before download, or alternatively looping until the file status is "available". This is a recommended improvement for robustness.  
  - Covers the waiting and file availability nodes.  

---

#### 2.5 Cleanup

**Overview:**  
Ensures the Airtop browser session is terminated after the download process, freeing system resources.

**Nodes Involved:**  
- Terminate (also part of 2.2)  

**Node Details:**  
See section 2.2 for details on the Terminate node.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                           | Input Node(s)                | Output Node(s)            | Sticky Note                                                                                       |
|---------------------------|---------------------|-----------------------------------------|-----------------------------|---------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts the workflow manually             | None                        | Session                   |                                                                                                 |
| Session                   | Airtop (Session)    | Creates Airtop browser session           | When clicking ‘Test workflow’ | Window, Terminate          |                                                                                                 |
| Window                    | Airtop (Window)     | Opens specified URL in browser           | Session                     | Click on download button   |                                                                                                 |
| Click on download button  | Airtop (Interaction)| Clicks download button on webpage        | Window                      | Wait                      |                                                                                                 |
| Wait                      | Wait                | Pauses execution to allow file readiness | Click on download button    | Get file data             | Wait a few seconds to make sure the file is ready for download. Alternatively, loop until file status `available`. |
| Get file data             | Airtop (File)       | Fetches metadata of downloadable files   | Wait                        | Download file             |                                                                                                 |
| Download file             | Airtop (File)       | Downloads the actual file binary          | Get file data               | None                      |                                                                                                 |
| Terminate                 | Airtop (Session)    | Ends the Airtop browser session           | Session                     | None                      |                                                                                                 |
| Sticky Note               | Sticky Note         | Provides usage notes and instructions     | None                        | None                      | README with detailed explanation of workflow purpose, steps, and next steps for enhancements.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: To start the workflow manually.  

2. **Create an Airtop Session node**  
   - Name: "Session"  
   - Resource: Session  
   - Operation: Create (default)  
   - Credentials: Select your Airtop API key (must be set up before).  
   - Connect the Manual Trigger output to this node.  

3. **Create an Airtop Window node**  
   - Name: "Window"  
   - Resource: Window  
   - URL: `https://sample.cat/en/webm`  
   - Options: Enable Live View, Disable Resize  
   - Credentials: Use the same Airtop API credentials.  
   - Connect "Session" output to this node.  

4. **Create an Airtop Interaction node**  
   - Name: "Click on download button"  
   - Resource: Interaction  
   - Element Description: "Blue download button for file \"SD 640x360 (Seawater, drone view video, 30 FPS)\""  
   - Credentials: Same Airtop API credentials.  
   - Connect "Window" output to this node.  

5. **Create a Wait node**  
   - Name: "Wait"  
   - Amount: 10 seconds  
   - Connect "Click on download button" output to this node.  

6. **Create an Airtop Get File Data node**  
   - Name: "Get file data"  
   - Resource: File  
   - Operation: List  
   - Limit: 1  
   - Session IDs: Expression referencing session ID from "Session" node output: `={{ $('Session').item.json.sessionId }}`  
   - Credentials: Airtop API credentials.  
   - Connect "Wait" output to this node.  

7. **Create an Airtop Download File node**  
   - Name: "Download file"  
   - Resource: File  
   - Operation: Get  
   - File ID: Expression referencing the ID from the previous node: `={{ $json.id }}`  
   - Output Binary File: true  
   - Credentials: Airtop API credentials.  
   - Connect "Get file data" output to this node.  

8. **Create an Airtop Terminate node**  
   - Name: "Terminate"  
   - Resource: Session  
   - Operation: Terminate  
   - Credentials: Airtop API credentials.  
   - Connect "Session" output to this node in parallel (to ensure session ends after workflow completes).  

9. **(Optional) Create Sticky Notes for documentation**  
   - Add a sticky note near the Wait and file retrieval nodes reminding to wait adequately or implement looping for availability checks.  
   - Add a sticky note with README content explaining workflow purpose, use case, and next steps.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| README with detailed explanation of workflow purpose, step-by-step logic, and suggestions for enhancements such as retry logic and dynamic URL/button handling. Includes setup requirement for Airtop API key and next steps for integration with storage.              | Included as Sticky Note in the workflow. Links to https://portal.airtop.ai/api-keys for API keys.   |
| Recommendation to improve robustness by implementing a loop to check for file status `available` instead of fixed wait time.                                                                                                                                             | Sticky Note attached near Wait and Get file data nodes.                                             |
| Airtop API key required to authenticate all Airtop nodes. Register and retrieve keys at Airtop's developer portal.                                                                                                                                                      | https://portal.airtop.ai/api-keys                                                                     |

---

**Disclaimer:** This documentation is based exclusively on the provided n8n workflow JSON. All data processed and actions performed comply with applicable content policies and legal use.