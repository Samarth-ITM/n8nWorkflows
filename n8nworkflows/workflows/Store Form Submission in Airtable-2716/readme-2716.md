Store Form Submission in Airtable

https://n8nworkflows.xyz/workflows/store-form-submission-in-airtable-2716


# Store Form Submission in Airtable

### 1. Workflow Overview

This workflow, titled **"Automated Form Submission Data Storage in Airtable"**, is designed to automate the capture and storage of user-submitted form data into an Airtable base. It targets businesses or teams that want to eliminate manual data entry from web forms, ensuring data is accurately and efficiently recorded in a structured database for easy access and management.

The workflow is logically divided into two main blocks:

- **1.1 Input Reception:** Detects and receives new form submissions automatically.
- **1.2 Data Storage:** Maps and stores the received form data into Airtable.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow whenever a user submits the specified form. It acts as the entry point, capturing all form field data and metadata such as submission time.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **Node Name:** On form submission  
    - **Type:** Form Trigger (n8n-nodes-base.formTrigger)  
    - **Technical Role:** Listens for new submissions on a specific form and initiates the workflow with the submitted data.  
    - **Configuration:**  
      - Form Title: "Create User"  
      - Form Description: "Provide the necessary information here"  
      - Form Fields:  
        - Name (text, required)  
        - Age (number, required)  
        - Email (email, required)  
        - Address (text, optional)  
        - You have Subscription? (dropdown with options "Yes" or "No", required)  
      - Webhook ID assigned for external form submission integration.  
    - **Key Expressions/Variables:**  
      - Captures all form fields as JSON properties, e.g., `$json.Name`, `$json.Age`, `$json.email`, `$json.address`, `$json['You have Subscription ?']`  
      - Includes submission timestamp as `$json.submittedAt`  
    - **Input Connections:** None (trigger node)  
    - **Output Connections:** Connected to "User Data Storage" node  
    - **Version Requirements:** Uses version 2.2 of the form trigger node  
    - **Potential Failure Modes:**  
      - Webhook misconfiguration or network issues preventing trigger activation  
      - Missing required fields in submission causing incomplete data  
      - Form title mismatch if form is renamed or deleted  
    - **Sub-workflow:** None

#### 1.2 Data Storage

- **Overview:**  
  This block receives the form data from the trigger node and inserts it as a new record into a specified Airtable base and table, mapping each form field to the corresponding Airtable column.

- **Nodes Involved:**  
  - User Data Storage

- **Node Details:**

  - **Node Name:** User Data Storage  
    - **Type:** Airtable (n8n-nodes-base.airtable)  
    - **Technical Role:** Creates a new record in Airtable with mapped form data fields.  
    - **Configuration:**  
      - Operation: Create record  
      - Base and Table: Configured via URL mode (values not shown, must be set by user)  
      - Columns mapped as follows:  
        - Name → `$json.Name`  
        - Age → `$json.Age`  
        - Email → `$json.email`  
        - Address → `$json.address`  
        - Subscription → `$json['You have Subscription ?']`  
        - Submitted → `$json.submittedAt` (timestamp of submission)  
      - Mapping Mode: Explicitly defined below  
    - **Key Expressions/Variables:** Uses direct JSON path expressions to map form data to Airtable columns.  
    - **Input Connections:** Receives data from "On form submission" node  
    - **Output Connections:** None (end node)  
    - **Version Requirements:** Uses version 2.1 of the Airtable node  
    - **Credentials:** Requires Airtable API token configured under "airtableTokenApi"  
    - **Potential Failure Modes:**  
      - Invalid or missing Airtable API credentials causing authentication failure  
      - Incorrect base or table URL leading to failed record creation  
      - Data type mismatches if Airtable columns expect different formats  
      - Network or API rate limit errors from Airtable  
    - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                 | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                      |
|---------------------|-----------------------|--------------------------------|---------------------|---------------------|------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger           | Trigger workflow on form submit| None                | User Data Storage   | Create User Form                                                                                |
| User Data Storage    | Airtable               | Store form data in Airtable    | On form submission  | None                | Store User Data                                                                                |
| Sticky Note         | Sticky Note            | Workflow title display          | None                | None                | Automated Form Submission Data Storage in Airtable                                            |
| Sticky Note1        | Sticky Note            | Workflow description display    | None                | None                | This workflow automatically captures data submitted through a form and stores it in Airtable. By using a form submission trigger, the workflow ensures that every time a form is filled out, the data is instantly recorded in Airtable without manual effort. This streamlines data management, making it easy to store and organize form data in a structured database for future reference. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Automated Form Submission Data Storage in Airtable".

2. **Add a Form Trigger node:**
   - Set the node name to "On form submission".
   - Configure the form:
     - Title: "Create User"
     - Description: "Provide the necessary information here"
     - Fields:
       - Name: Text field, required, placeholder "Enter Your Name"
       - Age: Number field, required, placeholder "Enter Your Age"
       - Email: Email field, required, placeholder "Enter Your Email"
       - Address: Text field, optional, placeholder "Enter Your Address"
       - You have Subscription ?: Dropdown field, required, options: "Yes", "No"
   - Save the node. This node will generate a webhook URL to receive form submissions.

3. **Add an Airtable node:**
   - Name it "User Data Storage".
   - Set the operation to "Create".
   - Configure the Airtable base and table:
     - Use the URL mode to specify the Airtable base and table URLs (these must be obtained from your Airtable account).
   - Map the columns explicitly:
     - Name → `{{$json.Name}}`
     - Age → `{{$json.Age}}`
     - Email → `{{$json.email}}`
     - Address → `{{$json.address}}`
     - Subscription → `{{$json['You have Subscription ?']}}`
     - Submitted → `{{$json.submittedAt}}`
   - Set the mapping mode to "Define Below".
   - Configure Airtable credentials:
     - Create or select an Airtable API token credential with appropriate permissions.

4. **Connect the nodes:**
   - Link the output of "On form submission" node to the input of "User Data Storage" node.

5. **Add sticky notes (optional):**
   - Add a sticky note titled "Automated Form Submission Data Storage in Airtable" for workflow identification.
   - Add another sticky note describing the workflow purpose and automation benefits.

6. **Save and activate the workflow:**
   - Ensure the workflow is active to start listening for form submissions.
   - Test by submitting the form and verifying data appears in Airtable.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link              |
|----------------------------------------------------------------------------------------------------------------------------------|-----------------------------|
| Workflow developed by AI developers at WeblineIndia to automate form data capture and storage in Airtable.                      | [WeblineIndia](https://weblineindia.com) |
| This workflow eliminates manual data entry by automatically recording form submissions in Airtable, streamlining data management.| Workflow description sticky note in the workflow |
| Airtable API credentials must be configured properly to enable data storage functionality.                                        | Airtable official docs: https://airtable.com/api |
| The form trigger node requires a webhook URL; ensure your n8n instance is accessible externally or via tunneling for testing.    | n8n webhook documentation: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/ |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the "Automated Form Submission Data Storage in Airtable" workflow.