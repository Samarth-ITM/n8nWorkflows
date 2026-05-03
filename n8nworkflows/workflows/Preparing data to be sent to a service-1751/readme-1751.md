Preparing data to be sent to a service

https://n8nworkflows.xyz/workflows/preparing-data-to-be-sent-to-a-service-1751


# Preparing data to be sent to a service

### 1. Workflow Overview

This workflow is designed to prepare and format data correctly before sending it to a destination service such as a database, spreadsheet, or CRM. It ensures the data fields match the expected schema of the target system by renaming fields, filtering out unnecessary data, and adding additional required fields.

**Target Use Cases:**
- Transform incoming data fields to match those required by a spreadsheet or database.
- Filter out irrelevant fields, keeping only those needed for the destination.
- Add computed or timestamp fields required by the destination.

**Logical Blocks:**
- **1.1 Input Trigger:** Manual execution trigger to start the workflow.
- **1.2 Data Generation:** Simulate or retrieve sample customer data.
- **1.3 Data Preparation:** Rename and filter fields, add new fields.
- **1.4 Data Insertion:** Upsert the prepared data into a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:** This block serves as the starting point of the workflow, manually triggered by the user to initiate the data processing.
- **Nodes Involved:**  
  - On clicking 'execute'
- **Node Details:**

  - **Node Name:** On clicking 'execute'  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution manually; no input required.  
    - Configuration: Default manual trigger, no parameters set.  
    - Input Connections: None (starting node).  
    - Output Connections: Passes trigger event to "Customer Datastore - Generate some data".  
    - Edge Cases: User must manually trigger; no automatic scheduling or event-based triggers.  
    - Version Requirements: Compatible with all n8n versions supporting manual triggers.

#### 1.2 Data Generation

- **Overview:** Generates or retrieves a dataset representing customer records to be processed downstream.
- **Nodes Involved:**  
  - Customer Datastore - Generate some data
- **Node Details:**

  - **Node Name:** Customer Datastore - Generate some data  
    - Type: n8nTrainingCustomerDatastore (Training node to simulate customer data)  
    - Role: Produces an array of customer records with fields like `id`, `name`, `email`.  
    - Configuration: Operation set to `getAllPeople` to retrieve all simulated customer data.  
    - Input Connections: Triggered by "On clicking 'execute'".  
    - Output Connections: Passes data to "Set - Prepare fields".  
    - Edge Cases: Data may be empty or malformed if upstream simulation fails.  
    - Version Requirements: Must have the n8n training nodes installed; otherwise, this node will fail.

#### 1.3 Data Preparation

- **Overview:** Transforms the incoming data to match the destination schema by renaming fields, filtering only required ones, and adding a timestamp.
- **Nodes Involved:**  
  - Set - Prepare fields
- **Node Details:**

  - **Node Name:** Set - Prepare fields  
    - Type: Set  
    - Role: Renames fields, drops unwanted fields, adds a new field with the current timestamp.  
    - Configuration:  
      - Keeps only fields explicitly set (with `keepOnlySet` enabled).  
      - Sets:  
        - `ID` mapped from `id`  
        - `Full name` mapped from `name`  
        - `Email` mapped from `email`  
        - `Created time` set to current timestamp (`{{$now}}`).  
    - Input Connections: Receives data from "Customer Datastore - Generate some data".  
    - Output Connections: Sends formatted data to "Create or Update record in Google Sheet".  
    - Edge Cases: If any of the expected fields (`id`, `name`, `email`) are missing, resulting output will have empty or null values. Expression failures if source fields are undefined.  
    - Version Requirements: `keepOnlySet` option available from n8n v0.152.0 and later.  
    - Notes: This node is central to data formatting, ensuring compatibility with the Google Sheets destination.

#### 1.4 Data Insertion

- **Overview:** Inserts or updates the formatted records into a Google Sheet, matching on existing rows to prevent duplicates.
- **Nodes Involved:**  
  - Create or Update record in Google Sheet
- **Node Details:**

  - **Node Name:** Create or Update record in Google Sheet  
    - Type: Google Sheets  
    - Role: Upserts data into a specified Google Sheet range.  
    - Configuration:  
      - Operation: `upsert` (updates existing or inserts new records).  
      - Range: `A:C` (columns A to C targeted).  
      - Sheet ID: `13_bAEYNTzVXVY6SfAkBa9ijtJGSxPd8D-hcXXwXtdDo` (specific Google Sheet).  
      - Authentication: OAuth2 with stored Google Sheets credentials.  
    - Input Connections: Receives prepared data from "Set - Prepare fields".  
    - Output Connections: None (terminal node).  
    - Edge Cases:  
      - Authentication errors if credentials expire or are revoked.  
      - API quota limits or network timeouts.  
      - Mismatch between data fields and sheet columns could cause insertion errors.  
    - Version Requirements: Requires OAuth2 credentials; Google Sheets node must support upsert operation (available in recent versions).  

---

### 3. Summary Table

| Node Name                        | Node Type                     | Functional Role                      | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                                                          |
|---------------------------------|-------------------------------|------------------------------------|--------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'            | Manual Trigger                | Initiates workflow manually        | None                           | Customer Datastore - Generate some data |                                                                                                                                      |
| Customer Datastore - Generate some data | n8nTrainingCustomerDatastore | Simulates or retrieves customer data | On clicking 'execute'          | Set - Prepare fields             |                                                                                                                                      |
| Set - Prepare fields             | Set                           | Renames, filters, and formats data | Customer Datastore - Generate some data | Create or Update record in Google Sheet | Prepare fields: rename `name` to `Full name`, keep only `ID`, `Email`, add `Created time` field                                      |
| Create or Update record in Google Sheet | Google Sheets                | Upserts data into Google Sheet     | Set - Prepare fields           | None                            |                                                                                                                                      |
| Note                            | Sticky Note                   | Explains the use of Set node       | None                           | None                            | Very often your data is not in the right format to insert in a node. you can use the set node to fix it. Click the `Execute Workflow` button and double click on the nodes to see the input and output items. |
| Note1                           | Sticky Note                   | Explains data formatting for Sheets | None                           | None                            | This is where we put the data in the format that Google Sheets expect. This means changing the field name from `name` to `Full name`, dropping all fields except `ID`, `Email` and adding a `Created time` field |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - Leave default settings. Name it "On clicking 'execute'".

2. **Add Customer Datastore Node**  
   - Add a node of type **n8nTrainingCustomerDatastore** (ensure training nodes are installed).  
   - Set **Operation** to `getAllPeople`.  
   - Name it "Customer Datastore - Generate some data".  
   - Connect the output of "On clicking 'execute'" to this node's input.

3. **Add Set Node for Data Preparation**  
   - Add a **Set** node named "Set - Prepare fields".  
   - Enable **Keep Only Set** option.  
   - Add the following fields under the "Values to Set":  
     - Number type: `ID` with expression `{{$json["id"]}}`  
     - String type: `Full name` with expression `{{$json["name"]}}`  
     - String type: `Email` with expression `{{$json["email"]}}`  
     - String type: `Created time` with expression `{{$now}}` (current timestamp)  
   - Connect the output of "Customer Datastore - Generate some data" to this node's input.

4. **Add Google Sheets Node for Upsert**  
   - Add a **Google Sheets** node named "Create or Update record in Google Sheet".  
   - Set **Operation** to `upsert`.  
   - Set **Range** to `A:C`.  
   - Enter the **Sheet ID**: `13_bAEYNTzVXVY6SfAkBa9ijtJGSxPd8D-hcXXwXtdDo`.  
   - Select or create Google Sheets OAuth2 credentials with the required permissions.  
   - Connect the output of "Set - Prepare fields" to this node's input.

5. **Add Sticky Notes for Documentation (Optional)**  
   - Add a sticky note near the start node explaining the purpose of using the Set node:  
     > "Very often your data is not in the right format to insert in a node. you can use the set node to fix it. Click the `Execute Workflow` button and double click on the nodes to see the input and output items."
   - Add another sticky note near the Set node explaining the field transformation:  
     > "This is where we put the data in the format that Google Sheets expect. This means changing the field name from `name` to `Full name`, dropping all fields except `ID`, `Email` and adding a `Created time` field."

6. **Test Execution**  
   - Save the workflow and click the **Execute Workflow** button to run the process manually.  
   - Inspect the output of each node by double-clicking them to ensure data is formatted and inserted correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Using the Set node is a common and effective way to prepare data for downstream nodes in n8n workflows. | General best practice for data transformation before database or API insertion.                    |
| OAuth2 credentials for Google Sheets must have necessary scopes for reading/writing sheets.              | [Google Sheets API OAuth2 setup](https://developers.google.com/sheets/api/guides/authorizing)     |
| Upsert operation in Google Sheets node matches existing rows to update or inserts new rows if none found.| Useful for avoiding duplicates and maintaining data consistency in spreadsheets.                   |

---

This completes the detailed, structured analysis and documentation of the workflow "Preparing data to be sent to a service". It provides all necessary information to understand, replicate, and troubleshoot the workflow.