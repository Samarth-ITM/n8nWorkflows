Create an Onfleet task when a file in Google Drive is updated

https://n8nworkflows.xyz/workflows/create-an-onfleet-task-when-a-file-in-google-drive-is-updated-1547


# Create an Onfleet task when a file in Google Drive is updated

### 1. Workflow Overview

This workflow automates the creation of a delivery task in Onfleet whenever a specific file in Google Drive is updated. It is designed for last-mile delivery operations that want to trigger task creation based on changes in a monitored document or file stored in Google Drive.

**Use Case:**  
Automatically initiate or update delivery tasks in Onfleet based on changes detected in a Google Drive file, eliminating manual task entry and improving operational efficiency.

**Logical Blocks:**  
- **1.1 Input Reception:** Detect updates on a designated Google Drive file using a trigger node.  
- **1.2 Task Creation:** Upon detection, create a new task in Onfleet using the Onfleet API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception — Google Drive Trigger

- **Overview:**  
This block listens for update events on a specific file in Google Drive, polling every minute. It serves as the event source that initiates the workflow.

- **Nodes Involved:**  
  - Google Drive Trigger

- **Node Details:**  
  - **Type and Role:** `googleDriveTrigger` — event trigger node that polls Google Drive for file changes.  
  - **Configuration Choices:**  
    - Poll frequency set to every minute (`pollTimes` with mode `everyMinute`).  
    - Trigger condition set to `specificFile`, monitoring a particular file identified by `fileToWatch` (file ID or URL).  
  - **Key Expressions or Variables:**  
    - `fileToWatch`: User-supplied Google Drive file ID or URL to monitor.  
  - **Input and Output Connections:**  
    - No input nodes, as this is a trigger.  
    - Outputs to the Onfleet node upon detecting updates.  
  - **Version-specific Requirements:**  
    - Uses typeVersion 1 of the Google Drive Trigger node.  
  - **Potential Failure Types & Edge Cases:**  
    - Authentication errors if Google credentials expire or are invalid.  
    - File ID invalid or inaccessible due to permissions.  
    - Network timeouts or API rate limits on Google Drive API.  
  - **Sub-workflow Reference:** None.

#### 1.2 Task Creation — Onfleet Task Node

- **Overview:**  
This block creates a new delivery task in Onfleet each time the Google Drive Trigger outputs an update event.

- **Nodes Involved:**  
  - Onfleet

- **Node Details:**  
  - **Type and Role:** `onfleet` node, used to interact with the Onfleet API.  
  - **Configuration Choices:**  
    - Operation set to `create`, which creates a new task in Onfleet.  
    - `additionalFields` left empty, implying default task creation parameters or minimal required data.  
  - **Key Expressions or Variables:**  
    - No dynamic expressions configured by default; user may customize additional fields for task details.  
  - **Input and Output Connections:**  
    - Input from Google Drive Trigger node.  
    - No further output nodes connected.  
  - **Version-specific Requirements:**  
    - Uses typeVersion 1 of the Onfleet node.  
  - **Potential Failure Types & Edge Cases:**  
    - Authentication errors if Onfleet API key is invalid or missing.  
    - API rate limits or network issues.  
    - Missing required task parameters if not configured properly.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name          | Node Type                 | Functional Role                | Input Node(s)        | Output Node(s) | Sticky Note                                                                                                  |
|--------------------|---------------------------|-------------------------------|----------------------|----------------|--------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger| googleDriveTrigger        | Detect file updates in Google Drive | None                 | Onfleet        | Connect to Google Drive with your own credentials. Specify poll frequency and file ID/URL to monitor.       |
| Onfleet            | onfleet                   | Create a delivery task in Onfleet | Google Drive Trigger | None           | Update with your Onfleet API key. Register at https://onfleet.com/signup to obtain API key.                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node:**  
   - Add a new node of type `Google Drive Trigger`.  
   - Configure credentials with your Google account OAuth2 credentials.  
   - Set `Poll Times` to `Every Minute` to define polling frequency.  
   - Set `Trigger On` to `specificFile`.  
   - Enter the `File URL or ID` of the Google Drive file you want to monitor.  
   - Save the node.

2. **Create Onfleet Node:**  
   - Add a new node of type `Onfleet`.  
   - Configure credentials using your Onfleet API key (register at https://onfleet.com/signup if needed).  
   - Set the `Operation` parameter to `Create` to create a new delivery task.  
   - Leave `Additional Fields` empty or configure as needed to specify task details (e.g., destination, worker, etc.).  
   - Save the node.

3. **Connect Nodes:**  
   - Connect the output of the Google Drive Trigger node to the input of the Onfleet node.

4. **Activate Workflow:**  
   - Ensure both nodes have valid credentials and configurations.  
   - Activate the workflow to enable polling and automation.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                |
|----------------------------------------------------------------------------------------------|-----------------------------------------------|
| Onfleet is a last-mile delivery platform providing route planning, dispatch, and analytics. | https://onfleet.com                            |
| You can customize which Onfleet entity to interact with by modifying the Onfleet node config.| Workflow description                          |
| To register for an Onfleet API key, visit:                                                  | https://onfleet.com/signup                     |
| Google Drive Trigger node requires proper OAuth2 credentials with Drive access scope.        | n8n Google Drive Trigger documentation         |