Save Typeform survey results to Airtable

https://n8nworkflows.xyz/workflows/save-typeform-survey-results-to-airtable-384


# Save Typeform survey results to Airtable

### 1. Workflow Overview

This workflow automates the process of capturing survey responses from a Typeform form and saving them directly into an Airtable base. It is designed for use cases where real-time collection and storage of survey data is required, such as event registrations, feedback collection, or data aggregation for projects.

The workflow consists of two main logical blocks:

- **1.1 Input Reception:** Listening for new survey submissions from Typeform.
- **1.2 Data Storage:** Appending the received survey data into an Airtable table.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new responses submitted to a specific Typeform form. When a user submits a response, the node triggers the workflow and outputs the response data for further processing.

- **Nodes Involved:**  
  - Typeform Trigger

- **Node Details:**

  **Typeform Trigger**  
  - **Type and Technical Role:**  
    Event trigger node that listens for new survey responses from a specified Typeform form.  
  - **Configuration Choices:**  
    - `formId`: The unique identifier of the Typeform form to monitor (currently empty, must be set for operation).  
    - Credentials: Uses the "Typeform" API credentials to authenticate requests.  
  - **Key Expressions or Variables Used:**  
    - Emits the full response data payload from Typeform upon each submission.  
  - **Input and Output Connections:**  
    - Input: None (trigger node).  
    - Output: Connected to the Airtable node for downstream processing.  
  - **Version-Specific Requirements:**  
    Compatible with n8n version 1.x; no special version constraints noted.  
  - **Edge Cases or Potential Failure Types:**  
    - Authentication or credential errors if API keys are invalid or expired.  
    - Missing or incorrect `formId` leading to no triggers firing.  
    - Network timeouts or Typeform API downtime.  
  - **Sub-workflow Reference:** None.

#### 1.2 Data Storage

- **Overview:**  
  This block receives the survey response data from Typeform and appends it into a specified Airtable table, effectively storing the data for later use or analysis.

- **Nodes Involved:**  
  - Airtable

- **Node Details:**

  **Airtable**  
  - **Type and Technical Role:**  
    Data node that appends records to an Airtable base's table.  
  - **Configuration Choices:**  
    - `application`: The Airtable application or base identifier to specify where data will be stored (currently empty).  
    - `table`: The specific Airtable table to which new records are appended (currently empty).  
    - `operation`: Set to "append" to add new records rather than update or delete.  
    - Credentials: Uses the "Airtable" API credentials for authenticated access.  
  - **Key Expressions or Variables Used:**  
    - Receives the data payload directly from the Typeform Trigger node.  
  - **Input and Output Connections:**  
    - Input: Connected from the Typeform Trigger node output.  
    - Output: None (endpoint node).  
  - **Version-Specific Requirements:**  
    Compatible with n8n version 1.x; no special version constraints noted.  
  - **Edge Cases or Potential Failure Types:**  
    - Authentication failure if API key is invalid.  
    - Missing or incorrect `application` or `table` settings causing data not to be saved.  
    - Schema mismatch if Typeform data fields do not align with Airtable columns.  
    - Network errors or Airtable API rate limiting.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name       | Node Type                 | Functional Role           | Input Node(s)   | Output Node(s) | Sticky Note                     |
|-----------------|---------------------------|---------------------------|-----------------|----------------|--------------------------------|
| Typeform Trigger| Typeform Trigger          | Receive survey responses  | None            | Airtable       |                                |
| Airtable        | Airtable                  | Append data to Airtable   | Typeform Trigger| None           |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add the "Typeform Trigger" node:**  
   - Node Type: Typeform Trigger  
   - Key Configuration:  
     - Set the `formId` parameter to the ID of the Typeform form you want to monitor.  
   - Credentials:  
     - Create or select existing Typeform API credentials with access rights to the form.  
   - Position the node (e.g., at coordinates [450, 250]) for clarity.

3. **Add the "Airtable" node:**  
   - Node Type: Airtable  
   - Key Configuration:  
     - Set the `application` field to the Airtable base ID where data will be stored.  
     - Set the `table` field to the specific Airtable table name.  
     - Set the operation to `append` so new records are added.  
     - Map the incoming data fields from Typeform to Airtable columns as needed (this mapping is implicit in the node).  
   - Credentials:  
     - Create or select existing Airtable API credentials with write access to the target base and table.  
   - Position the node (e.g., at [660, 250]).

4. **Connect the nodes:**  
   - Connect the output of the Typeform Trigger node to the input of the Airtable node.

5. **Activate the workflow:**  
   - Ensure both credentials are valid and the form/table IDs are correctly set.  
   - Activate the workflow to listen for new Typeform submissions and append them to Airtable.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| Ensure that the Typeform API credentials have permission to access the specific form indicated by `formId`.                                           | n8n Typeform node documentation                          |
| Airtable schema must match the data format sent by Typeform for seamless data appending; consider using the "Set" node if field transformation is needed. | Airtable API documentation                               |
| This workflow is in an inactive state by default; activate it to start processing submissions.                                                        | n8n workflow activation guide                            |

---

This document fully describes the "Save Typeform survey results to Airtable" workflow, enabling users and automation agents to understand, reproduce, and maintain it effectively.