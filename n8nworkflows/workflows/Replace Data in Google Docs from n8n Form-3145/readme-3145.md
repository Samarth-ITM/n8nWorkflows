Replace Data in Google Docs from n8n Form

https://n8nworkflows.xyz/workflows/replace-data-in-google-docs-from-n8n-form-3145


# Replace Data in Google Docs from n8n Form

### 1. Workflow Overview

This workflow automates the process of generating customized Google Docs documents by replacing placeholder variables in a Google Docs template with data submitted through an n8n form. It is designed for users who want to quickly produce documents such as contracts, invoices, or reports without manual editing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures form submissions from an n8n form trigger node.
- **1.2 Template Preparation:** Copies a predefined Google Docs template file to create a new document instance.
- **1.3 Data Formatting:** Transforms the form data into a structured format suitable for Google Docs API text replacement requests.
- **1.4 Document Update:** Sends a batch update request to Google Docs API to replace template variables with actual form data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives user input from a web form created within n8n. It triggers the workflow when the form is submitted and collects the data for further processing.

- **Nodes Involved:**  
  - Form  
  - Sticky Note (authentication reminder)

- **Node Details:**

  - **Form**  
    - Type: Form Trigger  
    - Role: Entry point capturing form submissions  
    - Configuration:  
      - Form Title: "Form"  
      - Form Fields: One required field labeled "name"  
      - Webhook ID assigned for external access  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Copy template file" node  
    - Edge Cases:  
      - Missing required fields will prevent submission  
      - No authentication configured by default (security risk)  
    - Sticky Note attached: Advises adding Basic Auth to secure the form endpoint

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Instructional reminder to add Basic Authentication to the form trigger to prevent unauthorized access  
    - Position: Near the Form node for visibility

#### 1.2 Template Preparation

- **Overview:**  
  This block duplicates a Google Docs template file to create a new document instance named after the form input, preparing it for variable replacement.

- **Nodes Involved:**  
  - Copy template file

- **Node Details:**

  - **Copy template file**  
    - Type: Google Drive node  
    - Role: Copies a Google Docs template file to a new file named after the form input "name" field  
    - Configuration:  
      - Operation: Copy file  
      - Source File ID: Fixed Google Docs template file ID  
      - New File Name: Set dynamically to the submitted form field "name" value  
    - Inputs: Receives data from "Form" node  
    - Outputs: Connects to "Format form data" node  
    - Credentials: Uses Google Drive OAuth2 credentials  
    - Edge Cases:  
      - Invalid or revoked credentials cause authentication errors  
      - Incorrect file ID or permission issues cause copy failure  
      - Missing "name" field in form data leads to empty or invalid file name

#### 1.3 Data Formatting

- **Overview:**  
  This block processes the form data into a format compatible with the Google Docs API batchUpdate request, mapping each form field to a replaceAllText request.

- **Nodes Involved:**  
  - Format form data  
  - Format form data to Google Doc API

- **Node Details:**

  - **Format form data**  
    - Type: Code (JavaScript)  
    - Role: Extracts all key-value pairs from the form submission and converts them into an array of objects with keys "key" and "value"  
    - Configuration:  
      - Iterates over all form submission JSON properties dynamically  
      - Outputs an array named "webhook_data" containing all form fields  
    - Inputs: Receives data from "Copy template file" node  
    - Outputs: Connects to "Format form data to Google Doc API" node  
    - Edge Cases:  
      - Empty form submissions produce empty arrays  
      - Unexpected data types may cause runtime errors if not handled

  - **Format form data to Google Doc API**  
    - Type: Code (JavaScript)  
    - Role: Converts the array of form key-value pairs into Google Docs API "replaceAllText" request objects, excluding certain keys like "submittedAt" and "formMode"  
    - Configuration:  
      - For each key-value pair, creates an object with "replaceAllText" specifying the placeholder text (e.g., {{key}}) and replacement text (value)  
      - Outputs an array named "data" containing all replacement requests  
    - Inputs: Receives data from "Format form data" node  
    - Outputs: Connects to "Replace data in Google Doc" node  
    - Edge Cases:  
      - Keys with reserved names are excluded  
      - Missing or undefined values replaced with empty strings could cause unexpected document content

#### 1.4 Document Update

- **Overview:**  
  This block sends a batch update request to the Google Docs API to replace all placeholders in the copied document with the corresponding form data values.

- **Nodes Involved:**  
  - Replace data in Google Doc  
  - Sticky Note (authentication reminder)

- **Node Details:**

  - **Replace data in Google Doc**  
    - Type: HTTP Request  
    - Role: Calls Google Docs API batchUpdate endpoint to perform text replacements in the document  
    - Configuration:  
      - URL: Dynamically constructed using the copied document ID from "Copy template file" node  
      - Method: POST  
      - Body: JSON containing "requests" array with replaceAllText operations from previous node  
      - Authentication: Predefined Credential Type using Google Docs OAuth2 API credentials  
    - Inputs: Receives data from "Format form data to Google Doc API" node  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - Authentication failures if credentials expire or are revoked  
      - API rate limits or quota exceeded errors  
      - Invalid document ID or malformed requests cause API errors

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains that all form fields are automatically converted to Google Docs variables, e.g., a form field "address" corresponds to {{address}} in the template  
    - Positioned near the form and formatting nodes for user guidance

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Reminds users to select Predefined Credential Type and choose Google Docs OAuth API in the HTTP Request node authentication settings  
    - Positioned near the "Replace data in Google Doc" node

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                              | Input Node(s)         | Output Node(s)               | Sticky Note                                                                                       |
|-------------------------|---------------------|----------------------------------------------|-----------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Form                    | Form Trigger        | Captures form submissions                     | None                  | Copy template file          | Please add authentication to form by selecting Basic Auth to prevent unauthorized access to the form. |
| Sticky Note             | Sticky Note         | Reminder to add Basic Auth to form            | None                  | None                        | Please add authentication to form by selecting Basic Auth to prevent unauthorized access to the form. |
| Copy template file      | Google Drive        | Copies Google Docs template file               | Form                  | Format form data            |                                                                                                 |
| Format form data        | Code (JavaScript)   | Extracts form fields into key-value pairs     | Copy template file     | Format form data to Google Doc API |                                                                                                 |
| Format form data to Google Doc API | Code (JavaScript)   | Converts form data into Google Docs API requests | Format form data       | Replace data in Google Doc  |                                                                                                 |
| Replace data in Google Doc | HTTP Request       | Sends batchUpdate request to Google Docs API  | Format form data to Google Doc API | None                        | In Authentication, you need to select Predefined Credential Type and then choose Google Docs OAuth API. |
| Sticky Note1            | Sticky Note         | Explains variable replacement mechanism       | None                  | None                        | The workflow automatically fetches all form fields and converts them into variables in Google Doc. For example, if you add a text field to the form called "address," you can use the variable {{address}} in the Google Doc template. |
| Sticky Note2            | Sticky Note         | Authentication reminder for Google Docs API   | None                  | None                        | In Authentication, you need to select Predefined Credential Type and then choose Google Docs OAuth API. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a "Form" trigger node.  
   - Set the form title to "Form".  
   - Add a required text field labeled "name".  
   - Enable Basic Authentication on the webhook to secure the form endpoint (recommended).  
   - Save the node.

2. **Add Google Drive Node to Copy Template**  
   - Add a "Google Drive" node.  
   - Set operation to "Copy".  
   - Specify the source file ID of your Google Docs template (must be a Google Docs file with placeholders like {{variable}}).  
   - Set the new file name to the expression: `={{ $json.name }}` to name the copied file after the form input.  
   - Connect the output of the Form node to this node.  
   - Configure Google Drive OAuth2 credentials with appropriate permissions.

3. **Add Code Node to Format Form Data**  
   - Add a "Code" node.  
   - Use JavaScript to iterate over all form submission fields and convert them into an array of objects with keys "key" and "value".  
   - Example code snippet:  
     ```javascript
     const data = [];
     Object.keys($('Form').all().map((item) => {
       Object.keys(item.json).map((bodyProperty) => {
         data.push({
           key: bodyProperty,
           value: item.json[bodyProperty],
         });
       })
     }));
     return {
       webhook_data: data,
       pairedItem: 0,
     };
     ```  
   - Connect the output of the Google Drive copy node to this node.

4. **Add Code Node to Format Data for Google Docs API**  
   - Add another "Code" node.  
   - Transform the array of key-value pairs into Google Docs API batchUpdate "replaceAllText" requests, excluding keys like "submittedAt" and "formMode".  
   - Example code snippet:  
     ```javascript
     const result = [];
     $('Format form data').all().map((item) => {
       item.json.webhook_data.map((data) => {
         if ("submittedAt" !== data.key && "formMode" !== data.key) {
           result.push({
             "replaceAllText": {
               "containsText": {
                 "text": `{{${data.key}}}`, 
                 "matchCase": true
               },
               "replaceText": `${data.value}`
             },
           });
         }
       });
     });
     return {
       data: result,
       pairedItem: 0,
     };
     ```  
   - Connect the output of the previous Code node to this node.

5. **Add HTTP Request Node to Update Google Docs Document**  
   - Add an "HTTP Request" node.  
   - Set method to POST.  
   - Set URL to:  
     `https://docs.googleapis.com/v1/documents/{{ $('Copy template file').first().json.id }}:batchUpdate`  
   - Set body parameters with JSON containing the "requests" array from the previous node:  
     ```json
     {
       "requests": {{$json.data}}
     }
     ```  
   - Enable "Send Body" and set authentication to "Predefined Credential Type".  
   - Select Google Docs OAuth2 credentials with permission to edit documents.  
   - Connect the output of the second Code node to this node.

6. **Add Sticky Notes (Optional but Recommended)**  
   - Add sticky notes near the Form node reminding to enable Basic Auth for security.  
   - Add sticky notes near the HTTP Request node reminding to select the correct authentication method.  
   - Add a sticky note explaining the automatic mapping of form fields to Google Docs variables.

7. **Test the Workflow**  
   - Submit the form with sample data.  
   - Verify that a new Google Docs document is created with the form data replacing the placeholders.  
   - Check for errors in authentication or API calls and adjust credentials as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow automatically fetches all form fields and converts them into variables in Google Docs. For example, if you add a text field to the form called "address," you can use the variable {{address}} in the Google Doc template. | Sticky Note near form and formatting nodes                                                      |
| Please add authentication to form by selecting Basic Auth to prevent unauthorized access to the form.                 | Sticky Note near Form node                                                                      |
| In Authentication, you need to select Predefined Credential Type and then choose Google Docs OAuth API.                | Sticky Note near HTTP Request node                                                             |
| Google Docs API documentation for batchUpdate method: https://developers.google.com/docs/api/reference/rest/v1/documents/batchUpdate | Useful for understanding the API calls made by the HTTP Request node                           |
| Google Drive API documentation for file copy operation: https://developers.google.com/drive/api/v3/reference/files/copy | Useful for understanding the template file copy operation                                      |

---

This document provides a complete and detailed reference for understanding, reproducing, and customizing the "Replace Data in Google Docs from n8n Form" workflow. It covers all nodes, logic blocks, configuration details, and potential edge cases to ensure robust implementation and maintenance.