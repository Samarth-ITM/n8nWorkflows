Load data into spreadsheet or database

https://n8nworkflows.xyz/workflows/load-data-into-spreadsheet-or-database-980


# Load data into spreadsheet or database

### 1. Workflow Overview

This workflow demonstrates a generic process for loading structured data into a tabular data destination such as Google Sheets, Airtable, CSV files, or relational databases like MySQL. It is designed to transform raw or nested input data into a tabular format that matches the target destination’s column structure, ensuring smooth insertion or appending of rows.

**Target Use Cases:**  
- Importing CRM contact data into spreadsheets or databases  
- Formatting complex nested data structures into flat key-value pairs for tabular storage  
- Demonstrating standard data transformation steps before loading into tabular destinations

**Logical Blocks:**

- **1.1 Input Trigger:** Manual triggering of the workflow to initiate the data loading process.  
- **1.2 Data Generation (Mock Data):** Simulates retrieving contacts data resembling a HubSpot CRM response.  
- **1.3 Data Transformation:** Maps nested CRM contact properties (name and email) into flat key-value pairs that correspond to spreadsheet/database columns.  
- **1.4 Data Loading Placeholder:** Represents the final node where data would be appended or inserted into the destination spreadsheet or database.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  Starts the workflow manually, allowing users to execute the process on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Type & Role:** Manual Trigger node — initiates the workflow when the user clicks "Execute Workflow."  
  - **Configuration:** Default manual trigger with no parameters.  
  - **Expressions:** None.  
  - **Connections:** Outputs to "Mock data (CRM Contacts)."  
  - **Version Requirements:** Standard node, no special version needs.  
  - **Potential Failures:** None expected since this is a manual trigger.  
  - **Sub-workflow Reference:** None.

---

#### 1.2 Data Generation (Mock Data)

- **Overview:**  
  Creates mock CRM contacts data mimicking a HubSpot API response format, providing sample input data for transformation.

- **Nodes Involved:**  
  - Mock data (CRM Contacts)

- **Node Details:**  
  - **Type & Role:** Function node — used to generate static JSON objects representing contacts.  
  - **Configuration:** JavaScript code pushes two contact objects into the output array. Each contact includes nested properties such as firstname, lastname, and email stored in complex nested arrays and objects.  
  - **Key Expressions:**  
    - None dynamic; the function outputs hardcoded JSON mimicking HubSpot contacts.  
  - **Input:** Receives input from the Manual Trigger node.  
  - **Output:** Passes generated items to the "Set" node for transformation.  
  - **Version Requirements:** Compatible with standard n8n Function node behavior.  
  - **Potential Failures:**  
    - Coding errors in the JavaScript function could cause runtime exceptions.  
    - Large data sets may impact performance.  
  - **Sub-workflow Reference:** None.

---

#### 1.3 Data Transformation

- **Overview:**  
  Transforms the nested contact data into flat key-value pairs suitable for insertion into tabular destinations, specifically extracting full names and primary email addresses.

- **Nodes Involved:**  
  - Set

- **Node Details:**  
  - **Type & Role:** Set node — used to define the output data structure by setting specific fields "Name" and "Email" for each item.  
  - **Configuration:**  
    - "Name" is set by concatenating the `firstname` and `lastname` values found in the nested `properties` object.  
    - "Email" is extracted from the first identity profile’s first identity value (assumed to be the primary email).  
  - **Key Expressions:**  
    - `={{$json["properties"]["firstname"]["value"]}} {{$json["properties"]["lastname"]["value"]}}` for "Name"  
    - `={{$json["identity-profiles"][0]["identities"][0]["value"]}}` for "Email"  
  - **Input:** Receives mock contact data from the Function node.  
  - **Output:** The transformed flat data items go to the next node for loading.  
  - **Version Requirements:** None specific; uses standard expression syntax.  
  - **Potential Failures:**  
    - If the expected JSON structure is missing or fields are undefined, expressions can fail or produce empty values.  
    - Assumes the first identity profile and first identity always contain an email — may not hold true in all data.  
  - **Sub-workflow Reference:** None.

---

#### 1.4 Data Loading Placeholder

- **Overview:**  
  Represents the final action node where the transformed data would be appended to or inserted into a spreadsheet or database.

- **Nodes Involved:**  
  - Replace me (NoOp node)

- **Node Details:**  
  - **Type & Role:** NoOp (No Operation) node — serves as a placeholder to be replaced by a real Google Sheets, Airtable, CSV, or database node configured with append or "Add row" operations.  
  - **Configuration:** No parameters configured; user needs to replace this node with actual destination node.  
  - **Input:** Receives transformed data from the Set node.  
  - **Output:** None (end of workflow).  
  - **Version Requirements:** None.  
  - **Potential Failures:** None as it currently does nothing.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name              | Node Type        | Functional Role                            | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                  |
|------------------------|------------------|--------------------------------------------|-------------------------|------------------------|--------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger   | Initiates workflow manually                 | —                       | Mock data (CRM Contacts) |                                                                                                              |
| Mock data (CRM Contacts)| Function        | Generates mock CRM contacts data             | On clicking 'execute'   | Set                    | "Get contacts" data from Hubspot node.                                                                       |
| Set                    | Set             | Transforms nested data into flat key-value pairs for spreadsheet columns | Mock data (CRM Contacts) | Replace me             |                                                                                                              |
| Replace me             | NoOp            | Placeholder for spreadsheet/database append operation | Set                     | —                      | Google Sheet/ Airtable/ Database with an "append" or "Add row" operation                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `On clicking 'execute'`.  
   - No special configuration needed.

2. **Create Function Node for Mock Data**  
   - Add a **Function** node named `Mock data (CRM Contacts)`.  
   - Insert the following JavaScript code to simulate HubSpot contacts data:
     ```javascript
     var newItems = [];
     newItems.push({json:{
         "addedAt": 1606827045601,
         "vid": 1,
         "canonical-vid": 1,
         "merged-vids": [],
         "portal-id": 8924380,
         "is-contact": true,
         "profile-token": "AO_T-mMZqmgHPI5CLLlw2qE24AlgWOJUL0LdMb2CegxeMzQK1LXyh7iZAgjNd-00ZdPAfnFU9Lv_7nq6qlrKvfAh8hr_cw-VBH1RCCMgHHYQ06DOXoIGAlViWmMKY-0lF9dv7lBVOMf5",
         "profile-url": "https://app.hubspot.com/contacts/8924380/contact/1",
         "properties": {
           "firstname": {"value": "Maria"},
           "lastmodifieddate": {"value": "1606827057310"},
           "company": {"value": "HubSpot"},
           "lastname": {"value": "Johnson (Sample Contact)"}
         },
         "form-submissions": [],
         "identity-profiles": [
           {
             "vid": 1,
             "saved-at-timestamp": 1606827045478,
             "deleted-changed-timestamp": 0,
             "identities": [
               {"type": "EMAIL", "value": "emailmaria@hubspot.com", "timestamp": 1606827045444, "is-primary": true},
               {"type": "LEAD_GUID", "value": "cfa8b21f-164e-4c9a-aab1-1235c81a7d26", "timestamp": 1606827045475}
             ]
           }
         ],
         "merge-audits": []
       }});
     newItems.push({json:{
         "addedAt": 1606827045834,
         "vid": 51,
         "canonical-vid": 51,
         "merged-vids": [],
         "portal-id": 8924380,
         "is-contact": true,
         "profile-token": "AO_T-mMX1jbZjaachMJ8t1F2yRdvyAvsir5RMvooW7XjbPZTdAv8hc24U0Rnc_PDF1gp1qmc8Tg2hDytOaRXRiWVyg-Eg8rbPFEiXNdU6jfMneow46tsSiQH1yyRf03mMi5ALZXMVfyA",
         "profile-url": "https://app.hubspot.com/contacts/8924380/contact/51",
         "properties": {
           "firstname": {"value": "Brian"},
           "lastmodifieddate": {"value": "1606827060106"},
           "company": {"value": "HubSpot"},
           "lastname": {"value": "Halligan (Sample Contact)"}
         },
         "form-submissions": [],
         "identity-profiles": [
           {
             "vid": 51,
             "saved-at-timestamp": 1606827045720,
             "deleted-changed-timestamp": 0,
             "identities": [
               {"type": "EMAIL", "value": "bh@hubspot.com", "timestamp": 1606827045444, "is-primary": true},
               {"type": "LEAD_GUID", "value": "d3749acc-06e1-4511-84fd-7b0d847f6eff", "timestamp": 1606827045717}
             ]
           }
         ],
         "merge-audits": []
       }});
     return newItems;
     ```
   - Save the node.

3. **Connect Manual Trigger to Function Node**  
   - Connect the output of `On clicking 'execute'` to the input of `Mock data (CRM Contacts)`.

4. **Create Set Node for Data Transformation**  
   - Add a **Set** node named `Set`.  
   - Enable the option "Keep Only Set" to discard any other fields.  
   - Add two string fields:  
     - **Name** with the value expression:  
       `={{$json["properties"]["firstname"]["value"]}} {{$json["properties"]["lastname"]["value"]}}`  
     - **Email** with the value expression:  
       `={{$json["identity-profiles"][0]["identities"][0]["value"]}}`  
   - Save the node.

5. **Connect Function Node to Set Node**  
   - Connect the output of `Mock data (CRM Contacts)` to the input of the `Set` node.

6. **Create a Placeholder Node for Loading Data**  
   - Add a **NoOp** node named `Replace me`.  
   - This node is a placeholder; replace it with your actual Google Sheets, Airtable, CSV, or database node configured to append or add rows.  
   - Save the node.

7. **Connect Set Node to Placeholder Node**  
   - Connect the output of `Set` node to the input of `Replace me`.

8. **Configure Destination Node (Replacement for `Replace me`)**  
   - Replace the `Replace me` node with a real node matching your destination type:  
     - For Google Sheets: Use Google Sheets node with credentials configured; action set to Append or Add Row; specify spreadsheet and sheet name.  
     - For Airtable: Use Airtable node with API key; specify base and table; action set to Create record(s).  
     - For MySQL or other DB: Use corresponding database node; configure connection credentials; specify table and insert operation.  
   - Map incoming fields "Name" and "Email" to the destination columns.  
   - Test with small data sets to verify.

9. **Workflow Activation and Testing**  
   - Save and activate the workflow.  
   - Click "Execute Workflow" to run the manual trigger and observe data flowing through to the destination.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| For complex nested data, consider using the Function node to flatten arrays into separate items before loading into spreadsheets or databases.            | [Example in n8n community forums](https://community.n8n.io/t/getting-all-items-from-trello-api-call/4567/8) |
| Use the [Set node](https://docs.n8n.io/nodes/n8n-nodes-base.set/#set) for simple data field transformations when input data is already itemized.         | n8n Official Documentation                                                                     |
| Use the [Function node](https://docs.n8n.io/nodes/n8n-nodes-base.function/#function) for advanced transformations or mapping nested data to flat structure. | n8n Official Documentation                                                                     |
| Loading data into spreadsheets or databases requires that the JSON keys exactly match the column names in the destination.                                | See key concept summary in workflow overview                                                  |
| Video walkthrough of the workflow is available for a visual explanation.                                                                                   | [Video Walkthrough](https://www.loom.com/share/eb87068f35a14af095f7b0f020b62211)                |

---

This documentation enables users and automation agents to fully understand, recreate, and adapt the workflow for loading data into tabular destinations, handling key transformation steps and anticipating common issues with nested data formats and destination schema matching.