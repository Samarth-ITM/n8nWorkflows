Form with Dynamic Dropdown Field

https://n8nworkflows.xyz/workflows/form-with-dynamic-dropdown-field-2712


# Form with Dynamic Dropdown Field

### 1. Workflow Overview

This workflow dynamically updates a customizable form’s dropdown field based on data fetched from an external source, exemplified here by Google Sheets. It is designed for use cases where form options need to reflect live or frequently changing datasets, such as event categories, product lists, or user roles.

The workflow is logically divided into the following blocks:

- **1.1 Data Source Polling:** Periodically triggers on changes in the Google Sheets document to fetch updated dropdown values.
- **1.2 Data Formatting:** Processes and formats the raw data from the source into a structure suitable for the form dropdown.
- **1.3 Workflow Retrieval and Update:** Retrieves the current workflow JSON, replaces the dropdown options with the new values, and updates the workflow to reflect these changes.
- **1.4 Form Submission Handling:** Listens for form submissions and triggers a sub-workflow to process the submitted data.
- **1.5 Configuration and Documentation:** Sticky notes provide setup instructions and contextual information for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Source Polling

- **Overview:**  
  This block triggers the workflow at regular intervals (every minute) when changes occur in the Google Sheets document containing dropdown options.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Get all values

- **Node Details:**

  - **Google Sheets Trigger**  
    - *Type:* Trigger node  
    - *Role:* Polls Google Sheets every minute for changes in the specified sheet and document.  
    - *Configuration:*  
      - Poll interval: every minute  
      - Document ID and Sheet Name set to a specific Google Sheets document and sheet (gid=0).  
      - Credentials: OAuth2 for Google Sheets Trigger.  
    - *Connections:* Outputs to "Get all values".  
    - *Potential Failures:* Authentication errors, API rate limits, connectivity issues.

  - **Get all values**  
    - *Type:* Google Sheets node  
    - *Role:* Reads all rows from the specified sheet to retrieve dropdown values.  
    - *Configuration:*  
      - Document ID and Sheet Name same as trigger.  
      - Credentials: OAuth2 for Google Sheets.  
    - *Connections:* Outputs to "Format to 'values'".  
    - *Potential Failures:* Authentication errors, empty or malformed sheet data.

---

#### 2.2 Data Formatting

- **Overview:**  
  Transforms raw data from Google Sheets into a standardized format with a key named `value` required for dropdown options.

- **Nodes Involved:**  
  - Format to 'values'  
  - Write JSON

- **Node Details:**

  - **Format to 'values'**  
    - *Type:* Set node  
    - *Role:* Maps the `title` field from each row to a new field named `value`.  
    - *Configuration:*  
      - Assigns `value` = `{{$json.title}}`  
    - *Connections:* Outputs to "Write JSON".  
    - *Potential Failures:* Missing `title` field in input data.

  - **Write JSON**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Constructs a nested JSON structure that fits the form field dropdown options format.  
    - *Configuration:*  
      - Maps each item’s `value` into an array of objects with `option` keys.  
      - Returns JSON structured to replace dropdown options in the workflow.  
    - *Connections:* Outputs to "n8n | get wf".  
    - *Potential Failures:* Code errors, empty input array.

---

#### 2.3 Workflow Retrieval and Update

- **Overview:**  
  Retrieves the current workflow JSON, replaces the dropdown options with the newly formatted values, and updates the workflow to apply changes.

- **Nodes Involved:**  
  - n8n | get wf  
  - Replace values  
  - n8n | update

- **Node Details:**

  - **n8n | get wf**  
    - *Type:* n8n node (workflow operation)  
    - *Role:* Retrieves the current workflow JSON by its own workflow ID.  
    - *Configuration:*  
      - Operation: get  
      - Workflow ID: current workflow’s ID (dynamic expression).  
      - Credentials: n8n API credentials.  
    - *Connections:* Outputs to "Replace values".  
    - *Potential Failures:* API authentication errors, workflow not found.

  - **Replace values**  
    - *Type:* Set node  
    - *Role:* Replaces the dropdown options in the retrieved workflow JSON with the new values from "Write JSON".  
    - *Configuration:*  
      - Uses a complex expression to find the dropdown field in the workflow JSON and replace its `fieldOptions.values` array.  
      - The replacement values come from the "Write JSON" node output.  
    - *Connections:* Outputs to "n8n | update".  
    - *Potential Failures:* Expression failures if the dropdown field is missing or JSON structure changes.

  - **n8n | update**  
    - *Type:* n8n node (workflow operation)  
    - *Role:* Updates the current workflow with the modified JSON containing new dropdown options.  
    - *Configuration:*  
      - Operation: update  
      - Workflow ID: from input JSON  
      - Workflow Object: stringified updated JSON  
      - Credentials: n8n API credentials.  
    - *Connections:* None (end of update chain).  
    - *Potential Failures:* API authentication errors, update conflicts.

---

#### 2.4 Form Submission Handling

- **Overview:**  
  Listens for form submissions and triggers the same workflow to process the submitted data.

- **Nodes Involved:**  
  - On form submission  
  - Execute Workflow

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger node  
    - *Role:* Exposes a webhook that serves a form with a dropdown field and listens for submissions.  
    - *Configuration:*  
      - Form title: "Example Title"  
      - Form fields: includes a text field and a dropdown field with default options (these will be replaced dynamically).  
      - Webhook ID: fixed UUID.  
    - *Connections:* Outputs to "Execute Workflow".  
    - *Potential Failures:* Webhook connectivity, form rendering issues.

  - **Execute Workflow**  
    - *Type:* Execute Workflow node  
    - *Role:* Calls the current workflow to process the form submission data.  
    - *Configuration:*  
      - Workflow ID: current workflow ID (dynamic expression).  
    - *Connections:* None (end of submission chain).  
    - *Potential Failures:* Workflow execution errors, infinite recursion if not handled carefully.

---

#### 2.5 Configuration and Documentation (Sticky Notes)

- **Overview:**  
  Provides user guidance on form setup, data source configuration, data formatting, nested properties, workflow retrieval, dropdown value replacement, and form update.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5  
  - Sticky Note6

- **Node Details:**

  - **Sticky Note**  
    - Content: Form setup instructions, emphasizing customization and production mode requirement.  
  - **Sticky Note1**  
    - Content: Data source setup instructions, highlighting Google Sheets connection and trigger timing.  
  - **Sticky Note2**  
    - Content: Data formatting explanation, importance of `value` field naming.  
  - **Sticky Note3**  
    - Content: Explanation of nested property transformation.  
  - **Sticky Note4**  
    - Content: Describes the "Get Workflow" node role.  
  - **Sticky Note5**  
    - Content: Explains replacing dropdown values in the form field.  
  - **Sticky Note6**  
    - Content: Describes the workflow update process with new dropdown values.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                          | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                      |
|---------------------|---------------------------|----------------------------------------|-----------------------|----------------------|------------------------------------------------------------------------------------------------|
| Google Sheets Trigger| Trigger                   | Polls Google Sheets for changes        | —                     | Get all values       | ## Data source setup - Connect to your Google Sheet containing dropdown values - Node can be replaced with any other data source (API, database) - Set timing trigger |
| Get all values       | Google Sheets             | Reads all rows from sheet               | Google Sheets Trigger  | Format to 'values'   |                                                                                                |
| Format to 'values'   | Set                       | Maps `title` to `value` field           | Get all values         | Write JSON           | ## Data formatting - Extracts needed data from source - Renames field to 'value' (do not change this name) |
| Write JSON           | Code                      | Constructs dropdown options JSON        | Format to 'values'     | n8n | get wf          | ## Nested properties - Transforms the data to the desired format                                |
| n8n | get wf          | n8n (workflow operation)  | Retrieves current workflow JSON         | Write JSON             | Replace values       | ## Get Workflow - Gets the current workflow data                                               |
| Replace values       | Set                       | Replaces dropdown options in workflow   | n8n | get wf              | n8n | update           | ## Add Dropdown Values - Replaces the nested parameters of the Dropdown Form Field with the nested properties sourced from the data. |
| n8n | update          | n8n (workflow operation)  | Updates workflow with new dropdown data | Replace values         | —                    | ## Update Form - Replaces the current workflow’s JSON with the updated JSON containing the new Dropdown values. |
| On form submission   | Form Trigger              | Serves form and listens for submissions| —                      | Execute Workflow     | ## Form setup - Customize your form fields. - The dropdown field will auto-update with values from your data source. - Other form fields can be added as needed (limited to one dropdown field). - Connect to your workflow that processes the submitted form data. - Form requires production mode for testing |
| Execute Workflow     | Execute Workflow          | Processes submitted form data           | On form submission     | —                    |                                                                                                |
| Sticky Note          | Sticky Note               | Documentation                          | —                      | —                    | See above for content                                                                           |
| Sticky Note1         | Sticky Note               | Documentation                          | —                      | —                    | See above for content                                                                           |
| Sticky Note2         | Sticky Note               | Documentation                          | —                      | —                    | See above for content                                                                           |
| Sticky Note3         | Sticky Note               | Documentation                          | —                      | —                    | See above for content                                                                           |
| Sticky Note4         | Sticky Note               | Documentation                          | —                      | —                    | See above for content                                                                           |
| Sticky Note5         | Sticky Note               | Documentation                          | —                      | —                    | See above for content                                                                           |
| Sticky Note6         | Sticky Note               | Documentation                          | —                      | —                    | See above for content                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node**  
   - Type: Google Sheets Trigger  
   - Configure to poll every minute.  
   - Set Document ID and Sheet Name to your Google Sheets document and sheet (e.g., gid=0).  
   - Use OAuth2 credentials for Google Sheets Trigger.

2. **Create Google Sheets node ("Get all values")**  
   - Type: Google Sheets  
   - Configure with the same Document ID and Sheet Name as the trigger.  
   - Use OAuth2 credentials for Google Sheets.  
   - Connect input from Google Sheets Trigger.

3. **Create Set node ("Format to 'values'")**  
   - Type: Set  
   - Add assignment: create a new field `value` with expression `{{$json.title}}` (adjust field name if your sheet uses a different column).  
   - Connect input from "Get all values".

4. **Create Code node ("Write JSON")**  
   - Type: Code (JavaScript)  
   - Paste the following code:
     ```javascript
     const inputArray = items.map(item => item.json);

     const output = [
       {
         nodes: [
           {
             parameters: {
               formFields: {
                 values: [
                   {
                     fieldOptions: {
                       values: inputArray.map(entry => ({ option: entry.value }))
                     }
                   }
                 ]
               }
             }
           }
         ]
       }
     ];

     return output.map(item => ({ json: item }));
     ```
   - Connect input from "Format to 'values'".

5. **Create n8n node ("n8n | get wf")**  
   - Type: n8n (workflow operation)  
   - Operation: get  
   - Workflow ID: set expression to current workflow ID `{{$workflow.id}}`  
   - Use n8n API credentials.  
   - Connect input from "Write JSON".

6. **Create Set node ("Replace values")**  
   - Type: Set  
   - Add an assignment with:  
     - Name: an expression that finds the dropdown field path in the workflow JSON (use the provided complex expression from the original workflow).  
     - Value: set to `{{$('Write JSON').item.json.nodes[0].parameters.formFields.values[0].fieldOptions.values}}`  
   - Enable "Include Other Fields" to preserve the rest of the JSON.  
   - Connect input from "n8n | get wf".

7. **Create n8n node ("n8n | update")**  
   - Type: n8n (workflow operation)  
   - Operation: update  
   - Workflow ID: expression `{{$json.id}}` (from input JSON)  
   - Workflow Object: expression `{{ JSON.stringify($json) }}`  
   - Use n8n API credentials.  
   - Connect input from "Replace values".

8. **Create Form Trigger node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form title and fields:  
     - Add a text field labeled "Example text field".  
     - Add a dropdown field labeled "Example dropdown" with placeholder options (these will be replaced dynamically).  
   - Set webhook ID (can be auto-generated or fixed).  
   - No input connections.

9. **Create Execute Workflow node ("Execute Workflow")**  
   - Type: Execute Workflow  
   - Workflow ID: expression `{{$workflow.id}}` (calls the same workflow)  
   - Connect input from "On form submission".

10. **Add Sticky Notes**  
    - Add sticky notes with the content provided in the original workflow for documentation and setup guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Form requires production mode for testing.                                                                           | Sticky Note on form setup.                                                                         |
| Customize your form fields; dropdown field auto-updates with data source values; limited to one dropdown field.       | Sticky Note on form setup.                                                                         |
| Connect to your Google Sheet containing dropdown values; node can be replaced with any other data source (API, DB).  | Sticky Note on data source setup.                                                                 |
| Extracts needed data from source and renames field to `value` (do not change this name).                              | Sticky Note on data formatting.                                                                   |
| Transforms the data to the desired nested format for dropdown options.                                               | Sticky Note on nested properties.                                                                 |
| Gets the current workflow data to enable dynamic update of dropdown options.                                         | Sticky Note on workflow retrieval.                                                                |
| Replaces the nested parameters of the dropdown form field with the new values sourced from the data.                 | Sticky Note on dropdown value replacement.                                                        |
| Updates the current workflow JSON with the new dropdown values to apply changes.                                     | Sticky Note on workflow update.                                                                   |

---

This documentation fully describes the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.