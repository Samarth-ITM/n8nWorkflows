CSV to HubSpot Uploader with Dynamic Field Mapping and Google Sheets Integration

https://n8nworkflows.xyz/workflows/csv-to-hubspot-uploader-with-dynamic-field-mapping-and-google-sheets-integration-3785


# CSV to HubSpot Uploader with Dynamic Field Mapping and Google Sheets Integration

### 1. Workflow Overview

This workflow automates the process of uploading CSV data into HubSpot CRM with dynamic field mapping and Google Sheets integration. It targets Customer Success Managers, marketers, sales teams, and data administrators who frequently import or update CRM records and want to reduce manual effort and errors.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initial Processing**  
  Handles CSV file upload, extracts headers and content, and identifies the type of HubSpot object for import.

- **1.2 HubSpot Properties Retrieval and Preparation**  
  Fetches the list of HubSpot properties for selected CRM objects, cleans and prepares them, and stores them in Google Sheets for reference.

- **1.3 Field Validation and Dynamic Mapping**  
  Checks if all CSV fields exist in HubSpot properties; if not, prompts the user to map unmatched fields via a dynamic form.

- **1.4 Data Transformation and Preparation for Import**  
  Applies the field mappings to transform CSV data into the correct format for HubSpot import.

- **1.5 HubSpot Data Import Execution**  
  Splits the transformed data into individual records and uploads them to HubSpot via API calls.

- **1.6 User Interaction and Feedback**  
  Provides form-based interaction for uploading CSV files and mapping fields, and confirms successful import.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initial Processing

- **Overview:**  
  This block manages user input through a form trigger, extracts CSV content and headers, and prepares the data for processing.

- **Nodes Involved:**  
  - File upload form  
  - To get the first line of file  
  - Set the real fields  
  - Get the content of file  

- **Node Details:**

  - **File upload form**  
    - Type: Form Trigger  
    - Role: Receives CSV file upload and object type selection from user.  
    - Configuration: Accepts single CSV file; dropdown for selecting HubSpot object type (Companies, Contacts, Leads, Deals, Tickets).  
    - Input: User form submission  
    - Output: Uploaded file and selected object type  
    - Edge Cases: File format issues, missing file, unsupported object type.

  - **To get the first line of file**  
    - Type: Extract From File  
    - Role: Extracts the first line (header) from the uploaded CSV file as text.  
    - Configuration: Operation set to "text".  
    - Input: File from form trigger  
    - Output: First line of CSV file  
    - Edge Cases: Empty file, encoding problems.

  - **Set the real fields**  
    - Type: Set  
    - Role: Parses the first line into an array of field names, splitting by semicolon delimiter.  
    - Configuration: Expression splits first line string by "\n" then by ";" to create array `real_fields`.  
    - Input: First line text  
    - Output: JSON with `real_fields` array  
    - Edge Cases: Incorrect delimiter, malformed header.

  - **Get the content of file**  
    - Type: Extract From File  
    - Role: Extracts full CSV content with header row, delimiter ";", UTF-8 encoding.  
    - Configuration: Header row enabled, delimiter set to semicolon, encoding UTF-8.  
    - Input: Uploaded file  
    - Output: Array of CSV rows as JSON objects  
    - Edge Cases: Encoding mismatches, delimiter inconsistencies.

---

#### 2.2 HubSpot Properties Retrieval and Preparation

- **Overview:**  
  Retrieves HubSpot properties for each object type, cleans the list, and stores the properties in a Google Sheet for reference.

- **Nodes Involved:**  
  - Start here to update your field list (manual trigger)  
  - Erase Google sheet  
  - Define array of objects  
  - Split by object  
  - Fetch properties from Hubspot  
  - Define crm_type  
  - Split results  
  - Remove hidden and starting with hs_ props fields  
  - Transforms the results  
  - Append to Google sheet  

- **Node Details:**

  - **Start here to update your field list**  
    - Type: Manual Trigger  
    - Role: Entry point to refresh and update HubSpot properties list.  
    - Input: Manual activation  
    - Output: Triggers properties update flow.

  - **Erase Google sheet**  
    - Type: Google Sheets  
    - Role: Clears existing content in the Google Sheet used for storing HubSpot properties.  
    - Configuration: Clears sheet "gid=0" in the specified Google Sheet document.  
    - Input: Trigger from manual node  
    - Output: Empty sheet ready for new data  
    - Edge Cases: Google Sheets API errors, credential issues.

  - **Define array of objects**  
    - Type: Set  
    - Role: Defines the list of HubSpot CRM objects to fetch properties for: ["companies", "contacts", "deals", "leads", "tickets"].  
    - Output: JSON array of object types.

  - **Split by object**  
    - Type: Split Out  
    - Role: Splits the array of objects into individual items to process separately.  
    - Input: Array of objects  
    - Output: Single object per item.

  - **Fetch properties from Hubspot**  
    - Type: HTTP Request  
    - Role: Calls HubSpot API to fetch properties for the current object type.  
    - Configuration: URL uses dynamic object name; OAuth2 credentials for HubSpot.  
    - Input: Object name from split node  
    - Output: JSON with properties list  
    - Edge Cases: API rate limits, authentication failures, network errors.

  - **Define crm_type**  
    - Type: Code  
    - Role: Adds a `crm_type` field to each property result to identify the object type, removes `options` field for simplicity.  
    - Input: API response JSON  
    - Output: Cleaned and annotated properties list.

  - **Split results**  
    - Type: Split Out  
    - Role: Splits the properties array into individual property items.  
    - Input: Properties array  
    - Output: Single property per item.

  - **Remove hidden and starting with hs_ props fields**  
    - Type: Filter  
    - Role: Filters out properties that are hidden or whose name starts with "hs_".  
    - Input: Individual property items  
    - Output: Filtered properties only.

  - **Transforms the results**  
    - Type: Code  
    - Role: Transforms property data as needed (currently returns `results` field) for Google Sheets formatting.  
    - Input: Filtered properties  
    - Output: Transformed properties.

  - **Append to Google sheet**  
    - Type: Google Sheets  
    - Role: Appends the cleaned and transformed properties to the Google Sheet for reference.  
    - Configuration: Maps all relevant property fields automatically; appends to sheet "gid=0".  
    - Input: Transformed properties  
    - Output: Updated Google Sheet  
    - Edge Cases: Google Sheets API limits, credential issues.

---

#### 2.3 Field Validation and Dynamic Mapping

- **Overview:**  
  Validates if all CSV input fields exist in HubSpot properties; if not, dynamically generates a form for user to map unmatched fields to existing HubSpot fields.

- **Nodes Involved:**  
  - Get the fields from the sheet  
  - Check if all fields from input are defined  
  - If all fields are defined  
  - Creates the correspondance table  
  - Form to set the correponding field for each input field  
  - Set the values for each field  
  - Set the values for each field1  

- **Node Details:**

  - **Get the fields from the sheet**  
    - Type: Google Sheets  
    - Role: Retrieves HubSpot properties filtered by the selected CRM object type from the Google Sheet.  
    - Configuration: Filters on `crm_type` matching the lowercase selected object type from form.  
    - Input: Object type from form trigger  
    - Output: Properties for selected object.

  - **Check if all fields from input are defined**  
    - Type: Code  
    - Role: Compares CSV header fields (`real_fields`) with HubSpot properties (`props`), returns boolean `response` indicating if all are defined.  
    - Input: CSV fields and properties  
    - Output: Boolean flag, keys, and properties arrays.

  - **If all fields are defined**  
    - Type: If  
    - Role: Branches workflow based on whether all CSV fields are recognized in HubSpot properties.  
    - Input: Boolean `response` from previous node  
    - Output: True branch for direct mapping, False branch for user mapping.

  - **Creates the correspondance table**  
    - Type: Code  
    - Role: For unmatched CSV fields, creates a dynamic form schema with dropdowns listing HubSpot properties to map each field.  
    - Input: CSV fields and properties  
    - Output: JSON form schema and mapping object.

  - **Form to set the correponding field for each input field**  
    - Type: Form  
    - Role: Presents the dynamic form to user to map unmatched CSV fields to HubSpot properties.  
    - Configuration: Form title and description explain mapping purpose.  
    - Input: Form schema from previous node  
    - Output: User field mappings.

  - **Set the values for each field**  
    - Type: Code  
    - Role: Applies direct mapping when all CSV fields match HubSpot properties, preparing data for import.  
    - Input: CSV data and properties  
    - Output: Transformed data array.

  - **Set the values for each field1**  
    - Type: Code  
    - Role: Applies user-defined field mappings from the form to transform CSV data accordingly.  
    - Input: CSV data and user mappings  
    - Output: Transformed data array.

---

#### 2.4 Data Transformation and Preparation for Import

- **Overview:**  
  Splits transformed data into individual records and packages them for HubSpot API import.

- **Nodes Involved:**  
  - Merge fields and data  
  - Split all records to import  
  - Define properties  

- **Node Details:**

  - **Merge fields and data**  
    - Type: Merge  
    - Role: Combines CSV header fields and HubSpot properties data streams for validation and mapping.  
    - Input: CSV header and content extraction nodes  
    - Output: Merged data for field validation.

  - **Split all records to import**  
    - Type: Split Out  
    - Role: Splits the array of transformed records into individual items for sequential API calls.  
    - Input: Transformed data array  
    - Output: Single record per item.

  - **Define properties**  
    - Type: Set  
    - Role: Wraps each individual record into an object under `def.properties` key for HubSpot API format.  
    - Input: Single record  
    - Output: JSON object ready for API.

---

#### 2.5 HubSpot Data Import Execution

- **Overview:**  
  Uploads each prepared record to HubSpot CRM using the appropriate API endpoint.

- **Nodes Involved:**  
  - Uploads to Hubspot  
  - Form response  

- **Node Details:**

  - **Uploads to Hubspot**  
    - Type: HTTP Request  
    - Role: Sends POST requests to HubSpot CRM API to create or update objects (default to "companies" endpoint).  
    - Configuration: URL fixed to companies endpoint; JSON body set dynamically from input; OAuth2 credentials used.  
    - Input: Single record JSON  
    - Output: API response  
    - Edge Cases: API rate limits, validation errors, authentication failures.

  - **Form response**  
    - Type: Form  
    - Role: Displays completion message to user confirming successful import.  
    - Configuration: Completion title set to "Your Data has been imported successfully".  
    - Input: Successful API calls  
    - Output: User notification.

---

#### 2.6 User Interaction and Feedback

- **Overview:**  
  Provides user guidance and feedback through sticky notes and forms.

- **Nodes Involved:**  
  - Sticky Note (multiple instances)  

- **Node Details:**

  - Sticky Notes provide instructions, explanations, and contact information at various workflow stages.  
  - Examples include:  
    - Workflow purpose and update instructions  
    - Import workflow instructions for file upload and object selection  
    - Property processor explanation  
    - Import execution notes  
    - Contact information for support and template links.

---

### 3. Summary Table

| Node Name                                | Node Type             | Functional Role                                      | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                                           |
|-----------------------------------------|-----------------------|-----------------------------------------------------|----------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| File upload form                        | Form Trigger          | Receives CSV file and object type from user         | -                                | To get the first line of file, Get the content of file | Sticky Note2: Explains form uploader usage and CSV format requirements                                                |
| To get the first line of file           | Extract From File     | Extracts CSV header line                             | File upload form                 | Set the real fields                  |                                                                                                                       |
| Set the real fields                     | Set                   | Parses CSV header into array                         | To get the first line of file    | Merge fields and data (via Get content of file) |                                                                                                                       |
| Get the content of file                 | Extract From File     | Extracts full CSV content                            | File upload form                 | Merge fields and data                |                                                                                                                       |
| Merge fields and data                   | Merge                 | Combines CSV header and content                      | Set the real fields, Get the content of file | Get the fields from the sheet       |                                                                                                                       |
| Get the fields from the sheet           | Google Sheets         | Retrieves HubSpot properties for selected object    | Merge fields and data            | Check if all fields from input are defined |                                                                                                                       |
| Check if all fields from input are defined | Code                  | Validates CSV fields against HubSpot properties     | Get the fields from the sheet    | If all fields are defined            | Sticky Note3: Explains property processor logic                                                                       |
| If all fields are defined               | If                    | Branches based on field validation                   | Check if all fields from input are defined | Set the values for each field, Creates the correspondance table |                                                                                                                       |
| Creates the correspondance table       | Code                  | Generates form schema for unmatched fields           | If all fields are defined (false branch) | Form to set the correponding field for each input field |                                                                                                                       |
| Form to set the correponding field for each input field | Form                  | Collects user mappings for unmatched fields          | Creates the correspondance table | Set the values for each field1       |                                                                                                                       |
| Set the values for each field           | Code                  | Maps CSV data directly when all fields match         | If all fields are defined (true branch) | Split all records to import          |                                                                                                                       |
| Set the values for each field1          | Code                  | Applies user-defined mappings to CSV data            | Form to set the correponding field for each input field | Split all records to import          |                                                                                                                       |
| Split all records to import             | Split Out             | Splits transformed data into individual records      | Set the values for each field, Set the values for each field1 | Define properties                   |                                                                                                                       |
| Define properties                      | Set                   | Packages each record into HubSpot API format          | Split all records to import      | Uploads to Hubspot                   |                                                                                                                       |
| Uploads to Hubspot                     | HTTP Request          | Sends data to HubSpot API                             | Define properties               | Form response                       | Sticky Note5: Notes about importing values into HubSpot                                                              |
| Form response                         | Form                  | Confirms successful import to user                    | Uploads to Hubspot              | -                                   |                                                                                                                       |
| Start here to update your field list   | Manual Trigger        | Initiates properties update workflow                  | -                                | Erase Google sheet                  | Sticky Note: "Update the properties by object Workflow"                                                               |
| Erase Google sheet                    | Google Sheets         | Clears Google Sheet for fresh properties data         | Start here to update your field list | Define array of objects             | Sticky Note9: Instructions to create an empty Google Sheet                                                           |
| Define array of objects               | Set                   | Defines HubSpot CRM objects to fetch properties for   | Erase Google sheet              | Split by object                    | Sticky Note6: Lists objects to import                                                                                  |
| Split by object                      | Split Out             | Splits array of objects for individual processing     | Define array of objects          | Fetch properties from Hubspot       |                                                                                                                       |
| Fetch properties from Hubspot         | HTTP Request          | Fetches properties from HubSpot API                   | Split by object                 | Define crm_type                    |                                                                                                                       |
| Define crm_type                      | Code                  | Adds crm_type field and cleans properties             | Fetch properties from Hubspot    | Split results                    | Sticky Note8: Notes on filtering properties                                                                           |
| Split results                      | Split Out             | Splits properties array into individual properties    | Define crm_type                 | Remove hidden and starting with hs_ props fields |                                                                                                                       |
| Remove hidden and starting with hs_ props fields | Filter                | Filters out hidden or system properties                | Split results                  | Transforms the results             |                                                                                                                       |
| Transforms the results               | Code                  | Prepares properties for Google Sheets                  | Remove hidden and starting with hs_ props fields | Append to Google sheet            |                                                                                                                       |
| Append to Google sheet               | Google Sheets         | Appends properties to Google Sheet                      | Transforms the results          | -                                 |                                                                                                                       |
| Sticky Note (multiple)               | Sticky Note           | Provides instructions and contact info                  | -                              | -                                 | Multiple sticky notes with workflow explanations and contact info                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "Start here to update your field list" to initiate the properties update process.

2. **Add a Google Sheets node** named "Erase Google sheet"  
   - Operation: Clear  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: "gid=0" (Sheet1)  
   - Credentials: Google Sheets OAuth2

3. **Add a Set node** named "Define array of objects"  
   - Define field `object` as array: `["companies","contacts","deals","leads","tickets"]`

4. **Add a Split Out node** named "Split by object"  
   - Field to split out: `object`

5. **Add an HTTP Request node** named "Fetch properties from Hubspot"  
   - Method: GET  
   - URL: `https://api.hubapi.com/crm/v3/properties/{{ $json.object }}`  
   - Authentication: HubSpot OAuth2 credentials

6. **Add a Code node** named "Define crm_type"  
   - For each property in `$json.results`, add `crm_type` = current object  
   - Remove `options` field  
   - Return cleaned results

7. **Add a Split Out node** named "Split results"  
   - Field to split out: `results`

8. **Add a Filter node** named "Remove hidden and starting with hs_ props fields"  
   - Conditions:  
     - Property `name` does NOT start with "hs_"  
     - Property `hidden` is false

9. **Add a Code node** named "Transforms the results"  
   - Return the property JSON as is or transform as needed for Google Sheets

10. **Add a Google Sheets node** named "Append to Google sheet"  
    - Operation: Append  
    - Document ID and Sheet Name as above  
    - Map all property fields automatically  
    - Credentials: Google Sheets OAuth2

11. **Create a Form Trigger node** named "File upload form"  
    - Form fields:  
      - File upload field (accept .csv, single file)  
      - Dropdown for "Type of import" with options: Companies, Contacts, Leads, Deals, Tickets

12. **Add an Extract From File node** named "To get the first line of file"  
    - Operation: Text  
    - Input: File from form

13. **Add a Set node** named "Set the real fields"  
    - Parse first line text into array `real_fields` by splitting on `\n` and then `;`

14. **Add an Extract From File node** named "Get the content of file"  
    - Operation: Table  
    - Encoding: UTF-8  
    - Delimiter: `;`  
    - Header row: true

15. **Add a Merge node** named "Merge fields and data"  
    - Merge the outputs of "Set the real fields" and "Get the content of file"

16. **Add a Google Sheets node** named "Get the fields from the sheet"  
    - Operation: Lookup with filter on `crm_type` equals lowercase selected object type from form  
    - Document ID and Sheet Name as above  
    - Credentials: Google Sheets OAuth2

17. **Add a Code node** named "Check if all fields from input are defined"  
    - Compare CSV fields (`real_fields`) with HubSpot properties (`props`)  
    - Return boolean `response` indicating if all CSV fields are recognized

18. **Add an If node** named "If all fields are defined"  
    - Condition: `$json.response === true`

19. **Add a Code node** named "Creates the correspondance table" (for false branch)  
    - For each unmatched CSV field, create a dropdown form element with HubSpot properties as options

20. **Add a Form node** named "Form to set the correponding field for each input field"  
    - Use the JSON form schema from previous node  
    - Execute once

21. **Add two Code nodes** for data transformation:  
    - "Set the values for each field" for direct mapping (true branch)  
    - "Set the values for each field1" for user-defined mapping (false branch)

22. **Add a Split Out node** named "Split all records to import"  
    - Field to split out: `out` (transformed data array)

23. **Add a Set node** named "Define properties"  
    - Wrap each record in an object with key `def.properties`

24. **Add an HTTP Request node** named "Uploads to Hubspot"  
    - Method: POST  
    - URL: `https://api.hubapi.com/crm/v3/objects/companies` (adjust dynamically if needed)  
    - Body Content Type: JSON  
    - Body: `={{ $json.def }}`  
    - Authentication: HubSpot OAuth2 credentials

25. **Add a Form node** named "Form response"  
    - Operation: Completion  
    - Completion Title: "Your Data has been imported successfully"

26. **Connect nodes accordingly following the workflow logic and dependencies.**

27. **Add Sticky Notes** at appropriate places to provide user guidance and documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Contact for workflow modifications, help, or custom workflow development: [thomas@pollup.net](mailto:thomas@pollup.net)                                 | Sticky Note7                                                                                        |
| Check other n8n templates by the author: [https://n8n.io/creators/zeerobug/](https://n8n.io/creators/zeerobug/)                                         | Sticky Note7                                                                                        |
| CSV file requirements: UTF-8 encoding, semicolon delimiter, header row with field names. Can be customized in "Get content of file" node.               | Sticky Note2                                                                                        |
| Define your list of HubSpot objects in "Define array of objects" node to customize import targets.                                                      | Sticky Note6                                                                                        |
| Create an empty Google Sheet with appropriate columns before running the properties update workflow to store HubSpot properties.                        | Sticky Note9                                                                                        |
| Filter properties to exclude hidden and system fields starting with "hs_" to avoid importing unsupported fields.                                        | Sticky Note8                                                                                        |
| Use the manual trigger "Start here to update your field list" to refresh HubSpot properties in Google Sheets before importing new data.                 | Sticky Note                                                                                         |
| The workflow assumes HubSpot API credentials with sufficient scopes and Google Sheets OAuth2 credentials are configured correctly in n8n credentials.   | Credential configuration note                                                                       |

---

This documentation provides a detailed and structured understanding of the "CSV to HubSpot Uploader with Dynamic Field Mapping and Google Sheets Integration" workflow, enabling users and AI agents to reproduce, modify, and troubleshoot it effectively.