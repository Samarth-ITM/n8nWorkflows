Template-Based Google Drive Folder Generation with Forms and Apps Script

https://n8nworkflows.xyz/workflows/template-based-google-drive-folder-generation-with-forms-and-apps-script-11663


# Template-Based Google Drive Folder Generation with Forms and Apps Script

### 1. Workflow Overview

This workflow automates the creation of a structured Google Drive folder system based on a predefined template folder. It is designed for use cases requiring consistent folder organization, such as agencies, freelancers, or teams managing client or project files. The workflow is triggered by a user submitting a form with a folder name, then it creates a new folder, duplicates the template folder structure inside it via a Google Apps Script, and replaces all `{{NAME}}` placeholders with the submitted name.

**Logical blocks:**

- **1.1 Form Input:** Captures folder name input from the user via a form.
- **1.2 Create Folder in Google Drive:** Creates a new folder in Google Drive with the submitted name under a specified parent folder.
- **1.3 Duplicate Template Structure:** Calls a Google Apps Script web app to duplicate the template folder structure inside the newly created folder and replace placeholders.

---

### 2. Block-by-Block Analysis

#### 2.1 Block: Form Input

- **Overview:**  
  This block captures user input through a form trigger node. The user submits the desired folder name to initiate the folder creation process.

- **Nodes Involved:**  
  - Folder Request Form

- **Node Details:**  

  - **Folder Request Form**  
    - **Type & Role:** `formTrigger` node; starts the workflow upon form submission.  
    - **Configuration:**  
      - Form titled "Create New Folder"  
      - Single required field: "Name" (placeholder example: "e.g. Acme Corporation")  
      - Form description prompts user to enter a name to generate the folder structure  
    - **Expressions/Variables:** Outputs JSON with the key `Name` containing the user input.  
    - **Connections:** Outputs to "Create Main Folder" node.  
    - **Version Requirements:** Uses typeVersion 2.2, standard for `formTrigger`.  
    - **Edge Cases / Failures:**  
      - Missing or empty name input (form enforced required)  
      - Webhook downtime or connectivity issues blocking form submission  
    - **Sub-workflow:** None

#### 2.2 Block: Create Folder in Google Drive

- **Overview:**  
  Creates a new folder in Google Drive, named after the user’s input, inside a specified parent folder.

- **Nodes Involved:**  
  - Create Main Folder  
  - Wait for Drive

- **Node Details:**  

  - **Create Main Folder**  
    - **Type & Role:** `googleDrive` node; creates a folder resource.  
    - **Configuration:**  
      - Folder name set dynamically from the form input: `={{ $json.Name }}`  
      - Drive specified as "My Drive" by name  
      - Parent folder ID set via a configured credential or environment variable placeholder `DESTINATION_PARENT_FOLDER_ID` (must be updated before use)  
      - Resource type: folder  
    - **Expressions/Variables:** Uses direct JSON path from form input for name.  
    - **Connections:** Outputs to "Wait for Drive"  
    - **Version Requirements:** typeVersion 3 for Google Drive node  
    - **Edge Cases / Failures:**  
      - Invalid or missing parent folder ID  
      - Insufficient permissions for folder creation  
      - API quota limits or network errors  
    - **Sub-workflow:** None

  - **Wait for Drive**  
    - **Type & Role:** `wait` node; introduces a pause to ensure folder creation is fully processed before further steps.  
    - **Configuration:** No parameters; default wait behavior (duration implicitly minimal or webhook-based sync)  
    - **Connections:** Outputs to "Duplicate Template" node  
    - **Version Requirements:** typeVersion 1.1  
    - **Edge Cases / Failures:**  
      - If wait is too short, subsequent steps may fail due to propagation delays  
    - **Sub-workflow:** None

#### 2.3 Block: Duplicate Template Structure

- **Overview:**  
  Calls a Google Apps Script web app to duplicate the predefined template folder structure inside the newly created folder and replace all `{{NAME}}` placeholders with the submitted folder name.

- **Nodes Involved:**  
  - Duplicate Template

- **Node Details:**  

  - **Duplicate Template**  
    - **Type & Role:** `httpRequest` node; invokes a Google Apps Script web app via HTTP POST.  
    - **Configuration:**  
      - URL must be set to the deployed Apps Script Web App URL (`YOUR_APPS_SCRIPT_URL`)  
      - HTTP method: POST  
      - Query parameters:  
        - `templateFolderId`: ID of the template folder with placeholders (`YOUR_TEMPLATE_FOLDER_ID`)  
        - `name`: dynamically set from form input `={{ $('Folder Request Form').item.json.Name }}`  
        - `destinationFolderId`: ID of the new folder created, `={{ $('Create Main Folder').item.json.id }}`  
      - Timeout set to 300000 ms (5 minutes) to allow for potentially long folder duplication  
    - **Expressions/Variables:** Uses references to previous nodes for dynamic values  
    - **Connections:** None (workflow ends here)  
    - **Version Requirements:** typeVersion 4.2  
    - **Edge Cases / Failures:**  
      - Incorrect or expired Apps Script URL or deployment errors  
      - Network timeouts or HTTP errors  
      - Insufficient permissions for Apps Script to access Drive folders  
      - Template folder ID invalid or missing placeholders  
      - Response errors from Apps Script (e.g., quota limits, scripting errors)  
    - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name           | Node Type        | Functional Role                    | Input Node(s)       | Output Node(s)     | Sticky Note                                                                                                           |
|---------------------|------------------|----------------------------------|---------------------|--------------------|-----------------------------------------------------------------------------------------------------------------------|
| Main Overview       | stickyNote       | Workflow purpose & instructions  | -                   | -                  | Auto-Generate Folder Structure from Template; setup steps and customization notes                                     |
| Warning - Apps Script| stickyNote       | Apps Script deployment instructions| -                   | -                  | ⚠️ Apps Script Setup (Required) - detailed deployment and setup steps with link to script.google.com                 |
| Section 1           | stickyNote       | Form Input section header         | -                   | -                  | ## 1. Form Input                                                                                                       |
| Section 2           | stickyNote       | Create Drive Folder section header| -                   | -                  | ## 2. Create Folder in Drive                                                                                           |
| Section 3           | stickyNote       | Duplicate Template section header | -                   | -                  | ## 3. Duplicate Template Structure                                                                                     |
| Folder Request Form  | formTrigger      | Capture folder name input         | -                   | Create Main Folder  | Enter a name to generate the folder structure                                                                         |
| Create Main Folder   | googleDrive      | Create new folder in Drive        | Folder Request Form  | Wait for Drive      | -                                                                                                                     |
| Wait for Drive      | wait             | Delay to ensure folder creation   | Create Main Folder   | Duplicate Template  | -                                                                                                                     |
| Duplicate Template   | httpRequest      | Call Apps Script to duplicate template folder structure| Wait for Drive | -               | -                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Sticky Note node named "Main Overview":**  
   - Paste the overview text describing the workflow’s purpose and setup steps.

3. **Add a Sticky Note node named "Warning - Apps Script":**  
   - Paste the detailed Apps Script deployment instructions and link to [script.google.com](https://script.google.com).  
   - Position this visibly near the duplication block.

4. **Add Sticky Note nodes "Section 1", "Section 2", "Section 3":**  
   - Label them to mark the workflow blocks.

5. **Add a Form Trigger node named "Folder Request Form":**  
   - Configure form title as "Create New Folder"  
   - Add a single required field named "Name" with placeholder text "e.g. Acme Corporation"  
   - Add form description "Enter a name to generate the folder structure"  
   - Save and note the webhook URL generated (used for form submissions).

6. **Add a Google Drive node named "Create Main Folder":**  
   - Set operation to create a folder.  
   - For the folder name, use expression: `={{ $json.Name }}` to dynamically name the folder from the form input.  
   - Select "My Drive" as the drive.  
   - Set the parent folder ID to the ID of the folder where all new folders should be created (replace `DESTINATION_PARENT_FOLDER_ID` with your actual folder ID).  
   - Ensure Google Drive credentials with write permissions are configured.

7. **Connect "Folder Request Form" node output to "Create Main Folder" node input.**

8. **Add a Wait node named "Wait for Drive":**  
   - No special configuration; default wait is sufficient to allow synchronization.

9. **Connect "Create Main Folder" node output to "Wait for Drive" node input.**

10. **Add an HTTP Request node named "Duplicate Template":**  
    - Set HTTP Method to POST.  
    - Set URL to your deployed Google Apps Script Web App URL (`YOUR_APPS_SCRIPT_URL`).  
    - Under Query Parameters, add:  
      - `templateFolderId` = your template folder ID (`YOUR_TEMPLATE_FOLDER_ID`)  
      - `name` = expression referencing form input: `={{ $('Folder Request Form').item.json.Name }}`  
      - `destinationFolderId` = expression referencing created folder ID: `={{ $('Create Main Folder').item.json.id }}`  
    - Set timeout to 300000 ms (5 minutes) to allow for longer processing times.  
    - Ensure no authentication is required or configure if Apps Script deployment requires it.

11. **Connect "Wait for Drive" node output to "Duplicate Template" node input.**

12. **Save the workflow and do a test submission via the form webhook to verify the entire folder creation and duplication process.**

13. **Update all placeholder values:**
    - `DESTINATION_PARENT_FOLDER_ID` with your Google Drive folder ID where new folders will be created.  
    - `YOUR_APPS_SCRIPT_URL` with your deployed Apps Script URL.  
    - `YOUR_TEMPLATE_FOLDER_ID` with your template folder ID containing placeholder folders/files.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| For Apps Script deployment, visit [script.google.com](https://script.google.com) to create and deploy the web app | Apps Script setup detailed in the "Warning - Apps Script" sticky note                                  |
| The workflow supports customization by adding more form fields, notifications (Slack, Email), or CRM updates  | Suggested in the "Main Overview" sticky note                                                           |
| Ensure Google Drive credentials have appropriate scopes for folder creation and access to involved folders    | Credential setup critical for Google Drive node and Apps Script access                                 |
| The Apps Script code must handle placeholder replacement in folder and file names, as well as content if needed| Provided separately in the workflow description and Apps Script deployment instructions                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.