Generate Automated GST Reports with GST Insights API and Google Docs

https://n8nworkflows.xyz/workflows/generate-automated-gst-reports-with-gst-insights-api-and-google-docs-7698


# Generate Automated GST Reports with GST Insights API and Google Docs

### 1. Workflow Overview

This workflow automates the generation of GST (Goods and Services Tax) reports by integrating form input, an external GST Insights API, and Google Docs. Its primary purpose is to capture GST/PAN numbers from users via a form, fetch detailed GST data from an API, reformat the data into a readable structure, and update a Google Document with this information.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the GST/PAN number from a user-submitted form.
- **1.2 GST Data Retrieval:** Sends the GST/PAN number to the GST Insights API to retrieve detailed company GST information.
- **1.3 Data Formatting:** Processes and reformats the raw API response into a structured plain-text format suitable for document insertion.
- **1.4 Document Update:** Updates a Google Document with the formatted GST details for record-keeping and reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by listening for a GST/PAN form submission from a user. It collects the required GST/PAN number to trigger downstream processing.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **Node Name:** On form submission  
  - **Type:** Form Trigger (n8n-nodes-base.formTrigger)  
  - **Role:** Entry trigger for the workflow on form submission  
  - **Configuration:**  
    - Form title: "GST Insight"  
    - One required form field labeled "GST/PAN No"  
    - No additional options configured  
  - **Expressions/Variables:** Captures the submitted "GST/PAN No" value as `$json["GST/PAN No"]`  
  - **Connections:** Outputs to the "GST Insights" HTTP Request node  
  - **Version:** 2.2  
  - **Edge Cases / Potential Failures:**  
    - Missing or malformed GST/PAN number input would prevent further processing (required flag mitigates this)  
    - Network or webhook errors affecting trigger reliability  
  - **Sub-workflow:** None

#### 1.2 GST Data Retrieval

- **Overview:**  
  This block sends the GST/PAN number received from the form to the GST Insights API via a POST request to fetch detailed GST-related company information.

- **Nodes Involved:**  
  - GST Insights

- **Node Details:**  
  - **Node Name:** GST Insights  
  - **Type:** HTTP Request (n8n-nodes-base.httpRequest)  
  - **Role:** Fetch GST details by sending the PAN number to an external API  
  - **Configuration:**  
    - HTTP Method: POST  
    - URL: https://gst-insights.p.rapidapi.com/index.php  
    - Content Type: multipart/form-data  
    - Headers:  
      - `x-rapidapi-host`: gst-insights.p.rapidapi.com  
      - `x-rapidapi-key`: user must provide their own API key ("your key" placeholder)  
    - Body Parameters: includes "pan" field with value from the form input: `={{ $json["GST/PAN No"] }}`  
    - Send headers enabled  
  - **Expressions/Variables:** Uses expression to insert PAN number from the form submission  
  - **Connections:** Outputs to the "Reformat" node  
  - **Version:** 4.2  
  - **Edge Cases / Potential Failures:**  
    - API authentication failure due to missing/invalid `x-rapidapi-key`  
    - API downtime or network timeouts  
    - Unexpected API response structure or errors  
    - Malformed PAN input causing API rejection  
  - **Sub-workflow:** None

#### 1.3 Data Formatting

- **Overview:**  
  This code node reformats the complex JSON response from the GST Insights API into a readable plain-text summary designed for insertion into a Google Document.

- **Nodes Involved:**  
  - Reformat

- **Node Details:**  
  - **Node Name:** Reformat  
  - **Type:** Code (JavaScript) (n8n-nodes-base.code)  
  - **Role:** Transform API JSON data into formatted plain text  
  - **Configuration:**  
    - JavaScript code extracts key fields such as company name, GST number, state, GST codes, addresses, status, trade name, registration dates, and e-invoice status  
    - Constructs a multi-line string with labels and corresponding values  
    - Returns an object with `docContent` property containing the formatted text  
  - **Expressions/Variables:** Uses `$input.first().json` to access API response data  
  - **Connections:** Outputs to the "Google Docs" node  
  - **Version:** 2  
  - **Edge Cases / Potential Failures:**  
    - API response missing expected fields causing undefined errors  
    - Data structure changes in API response may break extraction code  
    - Null or empty values in nested properties  
  - **Sub-workflow:** None

#### 1.4 Document Update

- **Overview:**  
  This block updates a specified Google Document by inserting the formatted GST details, enabling record-keeping and further sharing.

- **Nodes Involved:**  
  - Google Docs

- **Node Details:**  
  - **Node Name:** Google Docs  
  - **Type:** Google Docs (n8n-nodes-base.googleDocs)  
  - **Role:** Update the content of a Google Document with the formatted GST details  
  - **Configuration:**  
    - Operation: Update  
    - Action: Insert text (`={{ $json.docContent }}`)  
    - Document URL: (empty in JSON, must be specified)  
    - Authentication: Service Account using configured Google API credentials  
  - **Expressions/Variables:** Inserts text from `docContent` property output by the "Reformat" node  
  - **Connections:** Terminal node (no outputs)  
  - **Version:** 2  
  - **Edge Cases / Potential Failures:**  
    - Missing or incorrect Google Document URL  
    - Service account lacks permission to edit the document  
    - Network errors or quota limits on Google API usage  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name          | Node Type            | Functional Role                | Input Node(s)      | Output Node(s)    | Sticky Note                                                    |
|--------------------|----------------------|-------------------------------|--------------------|-------------------|----------------------------------------------------------------|
| On form submission | Form Trigger          | Capture GST/PAN form input    | -                  | GST Insights      | This node triggers the workflow when the user submits the GST/PAN form. It collects the required "GST/PAN No" and starts the automation process. |
| GST Insights       | HTTP Request          | Fetch GST details from API    | On form submission | Reformat          | Sends a POST request to the GST Insights API to retrieve GST details based on the submitted PAN. The API response includes company details such as GST number, registration date, and e-invoice status. |
| Reformat           | Code (JavaScript)     | Format API data into text     | GST Insights       | Google Docs       | Reformats the data fetched from the API into a plain-text format for document insertion. The node extracts and structures key company information like GST number, company name, and trade name. |
| Google Docs        | Google Docs           | Update document with GST info | Reformat           | -                 | Reformats the data fetched from the API into a plain-text format for document insertion. The node extracts and structures key company information like GST number, company name, and trade name. |
| Sticky Note        | Sticky Note           | Documentation notes           | -                  | -                 | Flow overview and block explanations                             |
| Sticky Note1       | Sticky Note           | Documentation note            | -                  | -                 | This node triggers the workflow when the user submits the GST/PAN form. It collects the required "GST/PAN No" and starts the automation process. |
| Sticky Note2       | Sticky Note           | Documentation note            | -                  | -                 | Sends a POST request to the GST Insights API to retrieve GST details based on the submitted PAN. The API response includes company details such as GST number, registration date, and e-invoice status. |
| Sticky Note3       | Sticky Note           | Documentation note            | -                  | -                 | Reformats the data fetched from the API into a plain-text format for document insertion. The node extracts and structures key company information like GST number, company name, and trade name. |
| Sticky Note4       | Sticky Note           | Documentation note            | -                  | -                 | Reformats the data fetched from the API into a plain-text format for document insertion. The node extracts and structures key company information like GST number, company name, and trade name. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "On form submission" node:**  
   - Node Type: Form Trigger  
   - Configure:  
     - Form Title: "GST Insight"  
     - Add one form field:  
       - Label: "GST/PAN No"  
       - Required: Yes  
   - This node acts as the trigger when the form is submitted.

2. **Create the "GST Insights" HTTP Request node:**  
   - Node Type: HTTP Request  
   - Set HTTP Method: POST  
   - Set URL: https://gst-insights.p.rapidapi.com/index.php  
   - Content Type: multipart/form-data  
   - Enable "Send Headers"  
   - Add headers:  
     - `x-rapidapi-host`: gst-insights.p.rapidapi.com  
     - `x-rapidapi-key`: *Your RapidAPI key here*  
   - Add Body Parameters (multipart form-data):  
     - Name: pan  
     - Value: Expression from previous node input: `={{ $json["GST/PAN No"] }}`  
   - Connect "On form submission" node output to this node’s input.

3. **Create the "Reformat" Code node:**  
   - Node Type: Code (JavaScript)  
   - Paste the following JavaScript code (adjust if necessary for your API response structure):  
     ```javascript
     const data = $input.first().json;

     const companyDetails = [
       { label: "Company Name", value: data.company_name },
       { label: "GST Number", value: data.gst_number },
       { label: "State", value: data.state },
       { label: "GST Code", value: data.gst_details.stjCd },
       { label: "GST Type", value: data.gst_details.dty },
       { label: "GST Location", value: data.gst_details.stj },
       { label: "GSTIN", value: data.gst_details.gstin },
       { label: "Status", value: data.sts },
       { label: "Trade Name", value: data.tradeNam },
       { label: "Supplier Address", value: `${data.gst_details.adadr[0].addr.bnm}, ${data.gst_details.adadr[0].addr.loc}, ${data.gst_details.adadr[0].addr.st}, ${data.gst_details.adadr[0].addr.pncd}` },
       { label: "Private Address", value: `${data.gst_details.pradr.addr.bnm}, ${data.gst_details.pradr.addr.loc}, ${data.gst_details.pradr.addr.st}, ${data.gst_details.pradr.addr.pncd}` },
       { label: "GST Update Date", value: data.gst_details.lstupdt },
       { label: "Company Type", value: data.gst_details.ctb },
       { label: "GST Registration Date", value: data.gst_details.rgdt },
       { label: "E-invoice Status", value: data.gst_details.einvoiceStatus }
     ];

     let docContent = `Company GST Details\n\n`;

     companyDetails.forEach(detail => {
       docContent += `${detail.label}: ${detail.value}\n`;
     });

     return [{ json: { docContent } }];
     ```  
   - Connect "GST Insights" node output to this node’s input.

4. **Create the "Google Docs" node:**  
   - Node Type: Google Docs  
   - Operation: Update  
   - Action: Insert text  
   - Text to insert: Expression from previous node: `={{ $json.docContent }}`  
   - Specify the Google Document URL to update (must be set manually)  
   - Authentication: Use Service Account credentials configured in n8n  
   - Connect "Reformat" node output to this node’s input.

5. **Credential Setup:**  
   - For Google Docs: Configure Google API credentials with Service Account having edit permissions on the target Google Document.  
   - For GST Insights API: Obtain an API key from RapidAPI for the GST Insights API and insert it in the HTTP Request node header `x-rapidapi-key`.

6. **Workflow Execution:**  
   - Deploy and activate the workflow.  
   - Submit the GST/PAN form with a valid number to trigger the workflow.  
   - The workflow will fetch GST data, format it, and update the Google Document accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates GST data retrieval and documentation, significantly reducing manual data entry for GST reports.  | Project purpose and automation benefits                                                                                        |
| The GST Insights API requires a valid RapidAPI key and expects PAN format compliance.                                      | https://rapidapi.com/                                                                                                          |
| Google Docs node requires the target document URL and appropriate edit permissions for the service account used.          | https://developers.google.com/docs/api                                                                                         |
| The code node assumes a specific nested JSON structure returned by the GST Insights API. Validate with real API outputs.  | Adjust code if API response structure changes                                                                                   |
| Sticky notes within the workflow provide inline documentation for each major block and node.                               | n8n canvas sticky notes                                                                                                        |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.