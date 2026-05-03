Generate n8n Forms from Airtable and BaseRow Tables 

https://n8nworkflows.xyz/workflows/generate-n8n-forms-from-airtable-and-baserow-tables--2603


# Generate n8n Forms from Airtable and BaseRow Tables 

### 1. Workflow Overview

This workflow dynamically generates n8n forms based on the schema of tables from two popular data sources: Airtable and Baserow. It allows users to select a database and table, then automatically builds an n8n form reflecting the table’s fields. Upon form submission, the workflow converts the submitted data into the appropriate API format and creates a new row in the selected table. Special handling is implemented for file/attachment fields, which are processed separately due to API constraints.

The workflow is logically divided into two parallel branches, one for Airtable and one for Baserow, each containing similar functional blocks:

- **1.1 Input Reception:** Form triggers that initiate the workflow and let users select the database and table.
- **1.2 Schema Retrieval:** Fetching the schema (fields metadata) of the selected Airtable base or Baserow table.
- **1.3 Schema Processing and Filtering:** Breaking down the schema, filtering unsupported field types, and converting it into n8n form fields schema.
- **1.4 Form Rendering:** Dynamically rendering the n8n form using the generated JSON schema.
- **1.5 Data Preparation:** After form submission, preparing the submitted data by filtering out file fields and typecasting booleans.
- **1.6 Row Creation:** Creating a new row/record in Airtable or Baserow using the prepared data.
- **1.7 File Handling:** Extracting submitted files, uploading them (for Baserow), and updating the row with file references.
- **1.8 Completion:** Displaying a submission completion message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Starts the workflow upon an n8n form submission where the user selects the target Airtable base/table or Baserow table.

**Nodes Involved:**  
- On form submission (Airtable)  
- On form submission1 (Baserow)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Configurations:  
    - Path: `airtable-n8n-form`  
    - Form Title: "Airtable to n8n Form"  
    - Form Fields: Dropdowns for BaseId (Airtable base) and TableId (Airtable table) with preset options  
  - Output: JSON with selected BaseId and TableId  
  - Edge Cases: Invalid or missing selections, webhook connectivity issues

- **On form submission1**  
  - Type: Form Trigger  
  - Configurations:  
    - Path: `baserow-n8n-form`  
    - Form Title: "Baserow to n8n Form"  
    - Form Fields: Dropdown for TableId with preset option  
  - Output: JSON with selected TableId  
  - Edge Cases: Same as above

---

#### 2.2 Schema Retrieval

**Overview:**  
Fetches the schema metadata of the selected table from Airtable or Baserow, which includes field definitions.

**Nodes Involved:**  
- Get Base Schema (Airtable)  
- Baserow List Fields (Baserow)

**Node Details:**

- **Get Base Schema**  
  - Type: Airtable node (resource: base, operation: getSchema)  
  - Configured with BaseId from form submission  
  - Retrieves all tables and fields metadata for the base  
  - Outputs large JSON including tables array

- **Baserow List Fields**  
  - Type: HTTP Request  
  - Method: GET  
  - URL: `https://api.baserow.io/api/database/fields/table/{{ $json.TableId }}/`  
  - Auth: HTTP Header Auth (Baserow credentials)  
  - Fetches fields metadata for the specified Baserow table

- Edge Cases: API authentication errors, invalid BaseId/TableId, network timeouts

---

#### 2.3 Schema Processing and Filtering

**Overview:**  
Processes the raw schema data, filters out unsupported field types, and converts the schema into n8n’s form fields JSON schema format.

**Nodes Involved:**  
- Filter Table (Airtable)  
- Fields to List (Airtable)  
- Covert to n8n Form Fields (Airtable)  
- Filter Unsupported FieldTypes (Airtable)  
- Combine Fields (Airtable)  
- Filter Unsupported FieldTypes1 (Baserow)  
- Covert to n8n Form Fields1 (Baserow)  
- Filter Table (Baserow) — Not explicitly named, but filtering done in code  
- Combine Fields1 (Baserow)

**Node Details:**

- **Filter Table** (Airtable)  
  - Type: Filter  
  - Filters tables array to the selected TableId  
  - Output: Single table JSON with fields array

- **Fields to List** (Airtable)  
  - Type: SplitOut  
  - Splits fields array into individual items for processing

- **Covert to n8n Form Fields** (Airtable)  
  - Type: Code  
  - Converts Airtable field types to n8n form field types (e.g., singleLineText → text, multipleAttachments → file)  
  - Handles options for dropdowns, required fields, placeholders, multi-select, and file properties  
  - Edge Cases: Unsupported or unknown field types are ignored (return empty object)

- **Filter Unsupported FieldTypes** (Airtable)  
  - Type: Filter  
  - Removes empty or unsupported fields after conversion

- **Combine Fields** (Airtable)  
  - Type: Aggregate  
  - Combines filtered fields back into a single JSON array for form rendering

- **Covert to n8n Form Fields1** (Baserow)  
  - Type: Code  
  - Similar conversion logic adjusted for Baserow’s field type and option names  
  - Maps Baserow types like text, long_text, single_select to n8n form types

- **Filter Unsupported FieldTypes1** (Baserow)  
  - Same as above, for Baserow

- **Combine Fields1** (Baserow)  
  - Combines filtered Baserow fields for form rendering

- Edge Cases: Missing or malformed schema, empty fields array, field type mismatches

---

#### 2.4 Form Rendering

**Overview:**  
Renders the n8n form dynamically by defining its fields via JSON schema constructed in previous steps.

**Nodes Involved:**  
- Render Form (Airtable)  
- Render Form1 (Baserow)

**Node Details:**

- **Render Form**  
  - Type: Form  
  - Configured to define form using JSON (input from Combine Fields node)  
  - Webhook to receive form data upon user submission  
  - Output: JSON and binary data from form submission

- **Render Form1**  
  - Same as above for Baserow

- Edge Cases: Form rendering failures, webhook routing errors, malformed JSON schema

---

#### 2.5 Data Preparation

**Overview:**  
Prepares the submitted form data for API insertion by filtering out file fields (which are processed separately) and casting boolean values correctly.

**Nodes Involved:**  
- Prep Data for Insert (Airtable)  
- Prep Data for Insert1 (Baserow)

**Node Details:**

- **Prep Data for Insert**  
  - Type: Code  
  - Filters out fields of type 'multipleAttachments'  
  - Casts checkbox fields to boolean  
  - Builds a sanitized JSON object for Airtable API

- **Prep Data for Insert1**  
  - Similar for Baserow  
  - Filters out 'file' type fields  
  - Casts 'boolean' type fields to boolean

- Edge Cases: Missing fields, type mismatches, unexpected data formats

---

#### 2.6 Row Creation

**Overview:**  
Creates a new row/record in the selected Airtable base/table or Baserow table using the prepared data.

**Nodes Involved:**  
- Airtable Create Record (Airtable)  
- Baserow Create Row (Baserow)

**Node Details:**

- **Airtable Create Record**  
  - Type: Airtable node (operation: create)  
  - Uses BaseId and TableId from form submission  
  - Uses auto mapping for fields with manual JSON expressions for values  
  - Does not include file fields (handled separately)  
  - Output: JSON with created record ID

- **Baserow Create Row**  
  - HTTP Request POST to Baserow API for creating row  
  - URL includes TableId  
  - Sends JSON body prepared from Prep Data for Insert1  
  - Auth via HTTP header credentials  
  - Output includes created row ID

- Edge Cases: API auth failures, validation errors, rate limits

---

#### 2.7 File Handling

**Overview:**  
Processes file or attachment fields submitted via the form, uploading files and updating the created row with file references.

**Nodes Involved:**  
- Files To List (Airtable)  
- Update Airtable Row (Airtable)  
- Files To List1 (Baserow)  
- Baserow Upload File (Baserow)  
- Group By FieldName (Baserow)  
- Baserow Update Row (Baserow)

**Node Details:**

- **Files To List** (Airtable)  
  - Code node that extracts file-type fields from form binary data  
  - Returns list of file objects for uploading

- **Update Airtable Row**  
  - Airtable node for uploading attachments to a specific record’s attachment field  
  - Uses binary data from Files To List  
  - Airtable API appends attachments rather than replacing

- **Files To List1** (Baserow)  
  - Same as Files To List but for Baserow fields

- **Baserow Upload File**  
  - HTTP POST to Baserow’s user-files upload endpoint with multipart form data  
  - Uploads each file and gets back file metadata

- **Group By FieldName**  
  - Code node groups uploaded files by their form field name, preparing JSON structure for updating row

- **Baserow Update Row**  
  - HTTP PATCH to update the created row with uploaded file references  
  - Baserow replaces file arrays on update, so all files must be sent at once

- Edge Cases: File size limits, upload failures, binary data corruption, API limits

---

#### 2.8 Completion

**Overview:**  
Displays a completion message to the user after form submission and all processing is finished.

**Nodes Involved:**  
- Show Completion! (Baserow)  
- Show Completion!1 (Airtable)

**Node Details:**

- Form node configured for completion operation  
- Shows "Submission Complete!" title and a thank you message  
- Executes once per submission

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                           | Input Node(s)              | Output Node(s)             | Sticky Note                                      |
|-------------------------|----------------------|-----------------------------------------|----------------------------|----------------------------|-------------------------------------------------|
| On form submission      | Form Trigger         | Start Airtable branch, user selects base/table | -                          | Get Base Schema            |                                                 |
| Get Base Schema         | Airtable             | Fetch Airtable base schema               | On form submission         | Filter Table               | See Sticky Note1 about Airtable schema           |
| Filter Table            | Filter               | Select specific table from base schema  | Get Base Schema            | Fields to List             |                                                 |
| Fields to List          | SplitOut             | Split table fields into individual items | Filter Table              | Covert to n8n Form Fields  |                                                 |
| Covert to n8n Form Fields| Code                 | Convert Airtable fields to n8n form fields | Fields to List            | Filter Unsupported FieldTypes| See Sticky Note2 for conversion details          |
| Filter Unsupported FieldTypes| Filter           | Remove unsupported or empty fields      | Covert to n8n Form Fields  | Combine Fields             |                                                 |
| Combine Fields          | Aggregate            | Aggregate all valid fields for form     | Filter Unsupported FieldTypes | Render Form             |                                                 |
| Render Form             | Form                 | Render dynamic n8n form for Airtable    | Combine Fields             | Prep Data for Insert       | See Sticky Note3 about dynamic form rendering    |
| Prep Data for Insert    | Code                 | Prepare submitted data (exclude files)  | Render Form                | Airtable Create Record     | See Sticky Note4 about data preparation           |
| Airtable Create Record  | Airtable             | Create new record in Airtable            | Prep Data for Insert       | Files To List              | See Sticky Note4 about creating row               |
| Files To List           | Code                 | Extract files from form submission       | Airtable Create Record     | Update Airtable Row        | See Sticky Note5 about file upload process        |
| Update Airtable Row     | HTTP Request (Airtable) | Upload files to Airtable record          | Files To List              | Show Completion!1          |                                                 |
| Show Completion!1       | Form                 | Show submission complete message (Airtable) | Update Airtable Row       | -                          |                                                 |
| On form submission1     | Form Trigger         | Start Baserow branch, user selects table | -                          | Baserow List Fields        |                                                 |
| Baserow List Fields     | HTTP Request         | Get fields metadata for Baserow table   | On form submission1        | Covert to n8n Form Fields1 | See Sticky Note1 about Baserow schema             |
| Covert to n8n Form Fields1| Code               | Convert Baserow fields to n8n form fields| Baserow List Fields       | Filter Unsupported FieldTypes1| See Sticky Note2 for conversion details          |
| Filter Unsupported FieldTypes1| Filter           | Remove unsupported or empty fields (Baserow) | Covert to n8n Form Fields1 | Combine Fields1          |                                                 |
| Combine Fields1         | Aggregate            | Aggregate all valid fields for form (Baserow) | Filter Unsupported FieldTypes1 | Render Form1            |                                                 |
| Render Form1            | Form                 | Render dynamic n8n form for Baserow     | Combine Fields1            | Prep Data for Insert1      | See Sticky Note3 about dynamic form rendering    |
| Prep Data for Insert1   | Code                 | Prepare submitted data (exclude files)  | Render Form1               | Baserow Create Row         | See Sticky Note4 about data preparation           |
| Baserow Create Row      | HTTP Request         | Create new row in Baserow table          | Prep Data for Insert1      | Files To List1             |                                                 |
| Files To List1          | Code                 | Extract files from form submission (Baserow) | Baserow Create Row        | Baserow Upload File        | See Sticky Note5 about file upload process        |
| Baserow Upload File     | HTTP Request         | Upload files to Baserow user-files       | Files To List1             | Group By FieldName         |                                                 |
| Group By FieldName      | Code                 | Group uploaded files by field name       | Baserow Upload File        | Baserow Update Row         |                                                 |
| Baserow Update Row      | HTTP Request         | Update Baserow row with uploaded files   | Group By FieldName         | Show Completion!           |                                                 |
| Show Completion!        | Form                 | Show submission complete message (Baserow) | Baserow Update Row        | -                          |                                                 |
| Sticky Note             | Sticky Note          | Instructions and overview                 | -                          | -                          | See content in Sticky Note                         |
| Sticky Note1            | Sticky Note          | Explains schema retrieval differences     | -                          | -                          | Covers Airtable and Baserow schema retrieval      |
| Sticky Note2            | Sticky Note          | Explains conversion to n8n form schema   | -                          | -                          |                                                 |
| Sticky Note3            | Sticky Note          | Explains rendering dynamic n8n form      | -                          | -                          |                                                 |
| Sticky Note4            | Sticky Note          | Explains data preparation and row creation| -                          | -                          |                                                 |
| Sticky Note5            | Sticky Note          | Explains file upload process differences  | -                          | -                          |                                                 |
| Sticky Note6            | Sticky Note          | Airtable example warning                   | -                          | -                          |                                                 |
| Sticky Note7            | Sticky Note          | Baserow example warning                    | -                          | -                          |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Airtable branch:**

   1.1. Add a **Form Trigger** node named "On form submission":
   - Set webhook path to `airtable-n8n-form`.
   - Add dropdown form fields for `BaseId` and `TableId` with appropriate options (e.g., "appfP15Xd0aVZR9xV" for BaseId).
   - Title the form "Airtable to n8n Form".

   1.2. Add an **Airtable** node named "Get Base Schema":
   - Set resource: `base`, operation: `getSchema`.
   - Use the `BaseId` from the form trigger (`{{$json.BaseId}}`) as input.
   - Connect the output of "On form submission" to this node.

   1.3. Add a **Filter** node "Filter Table":
   - Condition: Filter tables where `id` equals the `TableId` from form submission.
   - Connect from "Get Base Schema".

   1.4. Add a **SplitOut** node "Fields to List":
   - Field to split out: `fields`.
   - Connect from "Filter Table".

   1.5. Add a **Code** node "Covert to n8n Form Fields":
   - Run once per item.
   - Implement conversion logic mapping Airtable field types to n8n form field types.
   - Connect from "Fields to List".

   1.6. Add a **Filter** node "Filter Unsupported FieldTypes":
   - Filter out items where `fieldType` does not exist or is empty.
   - Connect from "Covert to n8n Form Fields".

   1.7. Add an **Aggregate** node "Combine Fields":
   - Aggregate all item data into one array.
   - Connect from "Filter Unsupported FieldTypes".

   1.8. Add a **Form** node "Render Form":
   - Define form using JSON.
   - Use expression to set JSON schema from aggregated fields (`{{$json.data}}`).
   - Set webhook to receive user input.
   - Connect from "Combine Fields".

   1.9. Add a **Code** node "Prep Data for Insert":
   - Run once per item.
   - Filter out `multipleAttachments` fields from submission.
   - Cast checkbox fields to booleans.
   - Connect from "Render Form".

   1.10. Add an **Airtable** node "Airtable Create Record":
   - Operation: `create` record.
   - Use BaseId and TableId from original form submission.
   - Map fields from prepared data.
   - Connect from "Prep Data for Insert".

   1.11. Add a **Code** node "Files To List":
   - Extract files from binary data corresponding to `file` fields.
   - Connect from "Airtable Create Record".

   1.12. Add an **HTTP Request** node "Update Airtable Row":
   - POST to Airtable attachments upload endpoint.
   - Use record ID from created record.
   - Attach files from binary data.
   - Connect from "Files To List".

   1.13. Add a **Form** node "Show Completion!1":
   - Operation: completion.
   - Display "Submission Complete!" and thank you message.
   - Connect from "Update Airtable Row".

2. **Create the Baserow branch:**

   2.1. Add a **Form Trigger** node "On form submission1":
   - Webhook path: `baserow-n8n-form`
   - Dropdown for TableId with appropriate options.
   - Title "Baserow to n8n Form".

   2.2. Add an **HTTP Request** node "Baserow List Fields":
   - GET `https://api.baserow.io/api/database/fields/table/{{ $json.TableId }}/`.
   - Use HTTP Header Auth credentials.
   - Connect from "On form submission1".

   2.3. Add a **Code** node "Covert to n8n Form Fields1":
   - Run once per item.
   - Map Baserow field types to n8n form field types.
   - Connect from "Baserow List Fields".

   2.4. Add a **Filter** node "Filter Unsupported FieldTypes1":
   - Filter out empty or unsupported fields.
   - Connect from "Covert to n8n Form Fields1".

   2.5. Add an **Aggregate** node "Combine Fields1":
   - Aggregate filtered fields.
   - Connect from "Filter Unsupported FieldTypes1".

   2.6. Add a **Form** node "Render Form1":
   - Define form using JSON from combined fields.
   - Webhook to receive user input.
   - Connect from "Combine Fields1".

   2.7. Add a **Code** node "Prep Data for Insert1":
   - Run once per item.
   - Filter out `file` type fields.
   - Cast boolean fields.
   - Connect from "Render Form1".

   2.8. Add an **HTTP Request** node "Baserow Create Row":
   - POST to `https://api.baserow.io/api/database/rows/table/{{ TableId }}/?user_field_names=true`.
   - JSON body from prepared data.
   - Auth: HTTP Header Auth.
   - Connect from "Prep Data for Insert1".

   2.9. Add a **Code** node "Files To List1":
   - Extract files from binary data matching `file` fields.
   - Connect from "Baserow Create Row".

   2.10. Add an **HTTP Request** node "Baserow Upload File":
   - POST to `https://api.baserow.io/api/user-files/upload-file/`.
   - Multipart form-data upload.
   - Auth: HTTP Header Auth.
   - Connect from "Files To List1".

   2.11. Add a **Code** node "Group By FieldName":
   - Group uploaded files by field label.
   - Format for Baserow row update.
   - Connect from "Baserow Upload File".

   2.12. Add an **HTTP Request** node "Baserow Update Row":
   - PATCH to update the created row with file references.
   - URL includes TableId and row ID.
   - Auth: HTTP Header Auth.
   - Connect from "Group By FieldName".

   2.13. Add a **Form** node "Show Completion!":
   - Completion message.
   - Connect from "Baserow Update Row".

3. **Credential Setup:**

   - Airtable Credential: Personal Access Token with appropriate scopes for base access.
   - Baserow Credential: HTTP Header Auth with API token configured.

4. **Webhook Setup:**

   - Ensure paths `airtable-n8n-form` and `baserow-n8n-form` are accessible and correctly routed.
   - The form nodes use webhook functionality for dynamic form rendering and submission.

5. **Testing:**

   - Test form rendering by accessing the webhook URL.
   - Submit test data, including files.
   - Verify record creation and file uploads in Airtable/Baserow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This template demonstrates replacing Airtable or Baserow forms with dynamically generated n8n forms using JSON definition.                | Overview from Sticky Note                                                                                |
| Check out the example Airtable base used in this workflow: https://airtable.com/appfP15Xd0aVZR9xV/shrGFgXLyQ4Jg58SU                        | Provided link in Sticky Note and description                                                            |
| New to Airtable? Sign up here: https://airtable.com/invite/r/cKzxFYVc                                                                        | Signup link for Airtable                                                                                  |
| Join the n8n community for help: [Discord](https://discord.com/invite/XPKeKXeB7d) or [Forum](https://community.n8n.io/)                      | Community support links                                                                                   |
| Airtable API uploads attachments by appending to the existing attachments array; Baserow replaces the file references on update.           | Important API behavior difference explained in Sticky Note5                                              |
| Baserow requires separate two-step file upload and row update process; Airtable can handle attachment upload in one step with special call.| See Sticky Note5                                                                                           |
| The workflow is a great exercise in dynamic form creation but may have limited practical use since Airtable and Baserow offer native forms.  | General note from description                                                                             |

---

This document fully captures the workflow’s structure, node roles, configurations, and logic to support understanding, modification, and reconstruction.