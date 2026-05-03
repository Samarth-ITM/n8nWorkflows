Validate emails in a table using Mailcheck

https://n8nworkflows.xyz/workflows/validate-emails-in-a-table-using-mailcheck-1055


# Validate emails in a table using Mailcheck

### 1. Workflow Overview

This workflow validates email addresses stored in an Airtable table using the Mailcheck node to verify their validity. It is designed for use cases where email quality assurance is required for a dataset maintained in Airtable or a similar table-based system.

The workflow consists of the following logical blocks:

- **1.1 Data Retrieval:** Fetches all records from the specified Airtable table.
- **1.2 Email Validation:** Uses the Mailcheck node to validate each email retrieved.
- **1.3 Data Preparation:** Sets and formats the output data, selecting only relevant fields to pass forward.
- **1.4 Data Update:** Updates the original Airtable records with the validation results.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Retrieval

- **Overview:**  
  This block lists all records from the Airtable table, extracting the email addresses to be validated.

- **Nodes Involved:**  
  - Airtable

- **Node Details:**  

  **Airtable**  
  - Type: Airtable node  
  - Role: Retrieves records from Airtable.  
  - Configuration:  
    - Operation: List all records from "Table 1".  
    - No filter or additional options specified, so all records are retrieved.  
  - Key Expressions: None used within this node.  
  - Inputs: None (starting node)  
  - Outputs: List of records, each containing a JSON object with fields including "Email" and an "id".  
  - Failure Modes:  
    - Authentication errors if Airtable API credentials are invalid or expired.  
    - Rate limiting or API errors from Airtable service.  
    - Empty table or missing "Email" field could cause downstream issues.  
  - Sub-workflow: Not applicable.

#### 1.2 Email Validation

- **Overview:**  
  Validates each email address obtained from Airtable using the Mailcheck API to verify if the email domain exists and is valid.

- **Nodes Involved:**  
  - Mailcheck

- **Node Details:**  

  **Mailcheck**  
  - Type: Mailcheck node  
  - Role: Checks the validity of email addresses.  
  - Configuration:  
    - Email parameter dynamically set via expression: `{{$json["fields"]["Email"]}}`. This extracts the "Email" field from each Airtable record.  
  - Credentials: Requires valid Mailcheck API credentials configured as "Mailcheck API Credentials".  
  - Inputs: Receives each record from Airtable node output.  
  - Outputs: Returns validation results, including a boolean field `mxExists` indicating if the email domain's MX records exist (a key validity indicator).  
  - Failure Modes:  
    - API authentication failure.  
    - Invalid email format leading to errors or false negatives.  
    - Network timeouts or API rate limits.  
  - Sub-workflow: Not applicable.

#### 1.3 Data Preparation

- **Overview:**  
  Prepares the data structure for updating Airtable by filtering and formatting only necessary data: the record ID and the validation boolean.

- **Nodes Involved:**  
  - Set

- **Node Details:**  

  **Set**  
  - Type: Set node  
  - Role: Maps and filters output data to keep only the record ID and validity flag.  
  - Configuration:  
    - Keeps only two fields:  
      - `ID`: set to `{{$node["Airtable"].json["id"]}}` (the record ID from the original Airtable node).  
      - `Valid`: set to `{{$json["mxExists"]}}` (boolean from Mailcheck result).  
    - Keeps only these set fields (`keepOnlySet` enabled), removing extraneous data.  
  - Inputs: Receives Mailcheck validation result with original record context.  
  - Outputs: Cleaned and simplified data object.  
  - Failure Modes:  
    - Expression errors if the expected fields are missing or named differently.  
  - Sub-workflow: Not applicable.

#### 1.4 Data Update

- **Overview:**  
  Updates the original Airtable records with the validation result in the "Valid" field based on the processed data.

- **Nodes Involved:**  
  - Airtable1

- **Node Details:**  

  **Airtable1**  
  - Type: Airtable node  
  - Role: Updates records in Airtable.  
  - Configuration:  
    - Operation: Update a single record by ID.  
    - Table: "Table 1" (dynamic via expression but effectively static here).  
    - Fields updated: "Valid" field only.  
    - Record ID: `{{$json["ID"]}}` passed from Set node.  
    - Application parameter copied from the first Airtable node to maintain connection context.  
    - `updateAllFields` set to false to update only specified fields.  
  - Credentials: Uses separate Airtable credentials "Airtable Credentials n8n".  
  - Inputs: Receives filtered data with record ID and validation boolean.  
  - Outputs: Confirmation of update per record.  
  - Failure Modes:  
    - Authentication failure.  
    - Record ID missing or invalid causing update failure.  
    - Rate limiting or API errors.  
  - Sub-workflow: Not applicable.

---

### 3. Summary Table

| Node Name   | Node Type           | Functional Role        | Input Node(s) | Output Node(s) | Sticky Note                                                                                                                                                                    |
|-------------|---------------------|-----------------------|---------------|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Airtable    | Airtable            | Retrieve records      | None          | Mailcheck      | This node will list all the records from a table. Based on your use case, you might want to replace this node.                                                                 |
| Mailcheck   | Mailcheck           | Validate email        | Airtable      | Set            | This node will check the emails that got returned by the previous node.                                                                                                        |
| Set         | Set                 | Prepare output data   | Mailcheck     | Airtable1      | We will use the Set node to ensure that only the data that we set in this node gets passed on to the next nodes in the workflow.                                              |
| Airtable1   | Airtable            | Update records        | Set           | None           | This node will update the Valid field in the table. Based on your use case, you might want to replace this node.                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Airtable Node (Retrieve Records):**  
   - Add an Airtable node named "Airtable".  
   - Credentials: Configure with your Airtable API key and base.  
   - Parameters:  
     - Operation: "List"  
     - Table: "Table 1" (or your relevant table name)  
   - No filters or additional options needed unless desired.  

2. **Create Mailcheck Node (Validate Emails):**  
   - Add a Mailcheck node named "Mailcheck".  
   - Credentials: Configure with your Mailcheck API credentials.  
   - Parameters:  
     - Email: Use expression `{{$json["fields"]["Email"]}}` to dynamically pull the email from Airtable output.  
   - Connect "Airtable" node output to "Mailcheck" node input.

3. **Create Set Node (Prepare Data):**  
   - Add a Set node named "Set".  
   - Parameters:  
     - Add two fields:  
       - Type: String, Name: "ID", Value: `{{$node["Airtable"].json["id"]}}`  
       - Type: Boolean, Name: "Valid", Value: `{{$json["mxExists"]}}`  
     - Enable "Keep Only Set" option to pass only these two fields.  
   - Connect "Mailcheck" node output to "Set" node input.

4. **Create Airtable Node (Update Records):**  
   - Add an Airtable node named "Airtable1".  
   - Credentials: Configure with your Airtable API key and base (can be same or different from the first).  
   - Parameters:  
     - Operation: "Update"  
     - Table: "Table 1"  
     - ID: Use expression `{{$json["ID"]}}` to specify which record to update.  
     - Fields to update: Check only "Valid".  
     - Update all fields: Set to false.  
     - Application: Use expression `{{$node["Airtable"].parameter["application"]}}` to inherit the application context from the first Airtable node.  
   - Connect "Set" node output to "Airtable1" node input.

5. **Save and Activate Workflow:**  
   - Ensure all credentials are valid and tested.  
   - Execute to validate emails and update records accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow screenshot visually demonstrates node connections and layout for clarity.                                     | Screenshot referenced in original workflow description (fileId:485)                                |
| Mailcheck API requires proper API credentials; ensure you have an active account and valid API key before using this node.  | Mailcheck official site/documentation for API key setup                                            |
| Airtable nodes can be replaced or adapted to other data sources depending on your use case (e.g., Google Sheets, databases).| Adaptation suggestion for different table sources                                                  |

---

This structured reference document provides a detailed understanding of the workflow’s components, logic, and configuration, enabling reproduction, customization, and troubleshooting for users and automation agents alike.