Get email notifications for newly uploaded Google Drive files

https://n8nworkflows.xyz/workflows/get-email-notifications-for-newly-uploaded-google-drive-files-1283


# Get email notifications for newly uploaded Google Drive files

### 1. Workflow Overview

This workflow automates email notifications whenever a new file is uploaded to a specific Google Drive folder. It is designed for users who want to stay informed about file additions without manually checking their Google Drive. The workflow comprises two main logical blocks:

- **1.1 Google Drive Event Detection:** Monitors a specific Google Drive folder and triggers the workflow when a new file is created.
- **1.2 Email Notification Dispatch:** Sends an email alert containing details about the newly uploaded file.

---

### 2. Block-by-Block Analysis

#### 1.1 Google Drive Event Detection

- **Overview:**  
  This block listens for new file creation events in a designated Google Drive folder. It initiates the workflow upon detection of such an event.

- **Nodes Involved:**  
  - Google Drive Trigger

- **Node Details:**

  - **Google Drive Trigger**  
    - **Type & Role:** Trigger node that activates the workflow when a new file is created in Google Drive.  
    - **Configuration:**  
      - Event set to `"fileCreated"` to detect new files.  
      - Trigger scope limited to `"specificFolder"`.  
      - Watches the folder with ID `"1HwOAKkkgveLji8vVpW9Xrg1EsBskwMNb"`.  
    - **Expressions/Variables:** None used in parameters; event-driven trigger.  
    - **Input/Output:**  
      - Input: None (trigger node).  
      - Output: Emits JSON data representing the newly created file, including file metadata such as `"name"`.  
    - **Version Requirements:** Compatible with n8n version supporting Google Drive OAuth2 credentials and trigger nodes.  
    - **Potential Failures:**  
      - Authentication errors if OAuth2 token expires or is invalid.  
      - Permission issues if folder ID is incorrect or inaccessible.  
      - Connectivity timeouts or API rate limits from Google Drive.  
    - **Sub-workflow:** Not applicable.

#### 1.2 Email Notification Dispatch

- **Overview:**  
  This block sends an email notification with information about the new file detected by the Google Drive Trigger node.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**

  - **Send Email**  
    - **Type & Role:** Action node that sends an SMTP email.  
    - **Configuration:**  
      - Email subject fixed as `"File Update"`.  
      - Email body text uses an expression to dynamically insert the file name: `"A file in your Google Drive file folder has been created: {{$json[\"name\"]}}"`.  
      - Email recipient set to `"mutedjam@n8n.io"`.  
      - Email sender set to `"mutedjam@n8n.io"`.  
    - **Expressions/Variables:**  
      - Uses data from the Google Drive Trigger node, specifically the `"name"` property of the file JSON object.  
    - **Input/Output:**  
      - Input: Receives data from Google Drive Trigger node.  
      - Output: None (final action node).  
    - **Version Requirements:** Requires SMTP credentials configured in n8n.  
    - **Potential Failures:**  
      - SMTP authentication failures.  
      - Email delivery problems (e.g., invalid recipient address, SMTP server unavailability).  
      - Expression evaluation errors if input data is missing or malformed.  
    - **Sub-workflow:** Not applicable.

---

### 3. Summary Table

| Node Name           | Node Type                    | Functional Role                     | Input Node(s)          | Output Node(s)      | Sticky Note                                          |
|---------------------|------------------------------|-----------------------------------|------------------------|---------------------|------------------------------------------------------|
| Google Drive Trigger | Google Drive Trigger          | Detect new file creations in folder | None                   | Send Email          |                                                      |
| Send Email          | Email Send                   | Send notification email about new file | Google Drive Trigger   | None                |                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Drive Trigger node:**  
   - Add a new node of type **Google Drive Trigger**.  
   - Set the **Event** parameter to `"fileCreated"`.  
   - For the **Trigger On** option, select `"specificFolder"`.  
   - Enter the **Folder To Watch** with the folder ID `"1HwOAKkkgveLji8vVpW9Xrg1EsBskwMNb"`.  
   - Configure **Google Drive OAuth2 API credentials** by selecting or creating an account with appropriate permissions to access the folder.

2. **Create the Send Email node:**  
   - Add a new node of type **Email Send**.  
   - Set the **To Email** field to `"mutedjam@n8n.io"`.  
   - Set the **From Email** field to `"mutedjam@n8n.io"`.  
   - Set the **Subject** field to `"File Update"`.  
   - In the **Text** field, enter the expression:  
     `=A file in your Google Drive file folder has been created: {{$json["name"]}}`  
     This dynamically inserts the name of the newly created file from the trigger data.  
   - Configure **SMTP credentials** with valid SMTP server details for sending emails.

3. **Connect nodes:**  
   - Connect the output of **Google Drive Trigger** node to the input of the **Send Email** node.

4. **Save and activate the workflow:**  
   - Ensure both credentials are correctly set and tested.  
   - Activate the workflow to enable automatic triggering upon file creation.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                         |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| The Google Drive Trigger node requires proper OAuth2 setup with access to the specified folder.         | n8n Docs: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googledrive#trigger |
| SMTP credentials must be configured with a valid SMTP server to successfully send emails.                | n8n Docs: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.email-send              |
| Folder ID can be found in the Google Drive folder URL after `/folders/` segment.                         | Google Drive UI                                           |
| Workflow image source: ![](https://docs.n8n.io/assets/img/workflow.44f43fab.png)                         | Provided in workflow description                          |