Send Webflow form data to Google Sheets

https://n8nworkflows.xyz/workflows/send-webflow-form-data-to-google-sheets-2555


# Send Webflow form data to Google Sheets

### 1. Workflow Overview

This workflow automates the process of capturing form submissions from a Webflow site and logging them into a Google Sheets spreadsheet. It is designed to streamline data collection by eliminating manual copying and pasting, while also ensuring data is structured and timestamped appropriately.

The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Triggered by a Webflow form submission via the Webflow webhook node.
- **1.2 Data Preparation:** Processes raw form data, extracting relevant fields and adding a submission timestamp.
- **1.3 Data Logging:** Dynamically appends the processed data as a new row into a specified Google Sheets document, automatically mapping columns as needed.

Additional elements include sticky notes that provide guidance on setup, configuration tips, and usage instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new form submissions from a specified Webflow site. It serves as the entry point for the workflow and triggers subsequent data processing.

- **Nodes Involved:**  
  - On Form Submission

- **Node Details:**  

  **On Form Submission**  
  - Type: Webflow Trigger (Webhook)  
  - Role: Receives live form submission data from Webflow.  
  - Configuration:  
    - Connected to a specific Webflow site by Site ID (`640cfc01791fc750653436fd`).  
    - Uses OAuth2 credentials for authentication with Webflow API (named “Webflow account”).  
  - Inputs: None (trigger node).  
  - Outputs: Emits incoming form submission JSON payload for further processing.  
  - Version: 2  
  - Potential Failures:  
    - OAuth token expiration or invalid credentials.  
    - Webhook misconfiguration or disabled legacy APIs on Webflow side (noted in sticky notes).  
    - Network or API downtime.  
  - Notes: Disabling legacy API is crucial to ensure webhook data format compatibility (see Sticky Note2).  

#### 1.2 Data Preparation

- **Overview:**  
  Extracts the form data from the Webflow webhook payload, normalizes the data structure by flattening the fields, and appends a submission date for logging purposes.

- **Nodes Involved:**  
  - Prepare Fields

- **Node Details:**  

  **Prepare Fields**  
  - Type: Code node (JavaScript)  
  - Role: Transforms the raw payload into a structured object suitable for Google Sheets insertion.  
  - Configuration:  
    - Extracts `payload.data` from the webhook JSON, which contains the form fields.  
    - Extracts `payload.submittedAt` timestamp or uses current date/time if missing.  
    - Returns a new object spreading all form fields plus a new `Date` field.  
  - Key expressions:  
    ```js
    const formData = $input.all()[0].json.payload.data;
    const Date = $input.all()[0].json.payload.submittedAt || new Date();
    return { ...formData, Date };
    ```  
  - Inputs: Receives JSON from "On Form Submission" node.  
  - Outputs: JSON object with individual form fields and a date field.  
  - Version: 2  
  - Potential Failures:  
    - Malformed or missing `payload.data` or `submittedAt` fields.  
    - Runtime exceptions in JavaScript if input structure changes.  
  - Sticky Note1 references the simplicity of this code snippet.  

#### 1.3 Data Logging

- **Overview:**  
  Appends the prepared form data as a new row into a Google Sheets spreadsheet. It dynamically maps columns based on the incoming data and can create columns if they do not exist yet.

- **Nodes Involved:**  
  - Append New Row

- **Node Details:**  

  **Append New Row**  
  - Type: Google Sheets node  
  - Role: Appends a new row with form data into a specified Google Sheets document.  
  - Configuration:  
    - Operation: `append` (add new row).  
    - Sheet Name: `Sheet1` (gid=0).  
    - Document ID: `1gLJ5I4ZJ9FQHJH56lunUKnHUBUsIms9PciIkJYi8SJE` (Google Sheets document).  
    - Columns: Maps `Name`, `Email`, and `Message` fields explicitly from the JSON path `$json.data.FieldName`.  
    - Mapping Mode: `autoMapInputData` — automatically creates missing columns and maps input data.  
  - Inputs: Receives prepared JSON from "Prepare Fields" node.  
  - Outputs: Operation result (row appended confirmation).  
  - Credentials: Uses Google Sheets OAuth2 account credentials.  
  - Version: 4.5  
  - Potential Failures:  
    - Authentication/authorization errors with Google API.  
    - Incorrect or missing spreadsheet ID or sheet name.  
    - Data type mismatches or too large payloads.  
    - API rate limiting or quota exceeded errors.  
  - Sticky Note3 highlights the capability to auto-create columns and append data even on empty sheets.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                 | Input Node(s)       | Output Node(s)    | Sticky Note                                                                                   |
|---------------------|---------------------|--------------------------------|---------------------|-------------------|-----------------------------------------------------------------------------------------------|
| On Form Submission  | Webflow Trigger     | Input reception from Webflow   | None                | Prepare Fields    | "Make sure to disable legacy API" with screenshot (Sticky Note2)                             |
| Prepare Fields      | Code                | Data extraction and timestamp  | On Form Submission  | Append New Row    | "1 line of code to take the data object (adding date as a plus)" (Sticky Note1)              |
| Append New Row      | Google Sheets       | Append processed data to sheet | Prepare Fields      | None              | "Automatically create column names and append data (works even on empty sheets)" (Sticky Note3) |
| Sticky Note1        | Sticky Note         | Guidance on Prepare Fields code| None                | None              | 1 line of code to take the data object (adding date as a plus)                               |
| Sticky Note2        | Sticky Note         | Setup tip for Webflow           | None                | None              | Make sure to disable legacy API ![](https://imgur.com/0tebypt.png)                           |
| Sticky Note3        | Sticky Note         | Google Sheets dynamic columns   | None                | None              | Automatically create column names and append data (works even on empty sheets)                |
| Sticky Note4        | Sticky Note         | Setup guidance for Webflow OAuth2 | None             | None              | Detailed instructions on obtaining client ID and secret from Webflow dashboard with images   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webflow Trigger Node**
   - Add a “Webflow Trigger” node.
   - Configure it with your Webflow Site ID (found in Webflow dashboard).
   - Authenticate using OAuth2 credentials linked to your Webflow account.
   - Ensure that legacy API support is disabled in your Webflow project settings (see instructions in Sticky Note2).
   - This node will listen for form submissions and trigger the workflow.

2. **Add a Code Node Named “Prepare Fields”**
   - Add a “Code” node immediately downstream of the Webflow Trigger.
   - Use JavaScript to extract the form fields and add a submission timestamp:
     ```js
     const formData = $input.all()[0].json.payload.data;
     const Date = $input.all()[0].json.payload.submittedAt || new Date();
     return { ...formData, Date };
     ```
   - This node normalizes the data for Google Sheets.

3. **Create a Google Sheets Node Named “Append New Row”**
   - Add a “Google Sheets” node connected after “Prepare Fields”.
   - Set operation to `append`.
   - Choose or enter your Google Sheets Document ID and Sheet Name (`Sheet1` or your target).
   - Set the column mapping mode to `autoMapInputData` to allow dynamic column creation.
   - Map fields explicitly for `Name`, `Email`, and `Message` from the JSON path `$json.data.FieldName`.
   - Authenticate with Google Sheets OAuth2 credentials.

4. **Connect Nodes in Order**
   - Connect “On Form Submission” node’s output to “Prepare Fields” node’s input.
   - Connect “Prepare Fields” node’s output to “Append New Row” node’s input.

5. **Add Sticky Notes for Documentation (Optional but Recommended)**
   - Add sticky notes to provide setup tips:
     - How to obtain Webflow OAuth2 credentials (client ID and secret). Refer to Sticky Note4 content with images.
     - Remind to disable legacy API in Webflow (Sticky Note2).
     - One-line code explanation for Prepare Fields (Sticky Note1).
     - Google Sheets dynamic column creation feature (Sticky Note3).

6. **Test the Workflow**
   - Submit a form on your Webflow site.
   - Verify that data appears correctly in your Google Sheets.
   - Check logs for errors such as authentication failures or schema mismatches.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| How to obtain Webflow Client ID and Client Secret for OAuth2: Navigate to Webflow Dashboard > Apps & Integrations > Create an App, enable REST API and form access, then copy credentials to n8n. | See Sticky Note4 with annotated screenshots: https://imgur.com/IX2ruVB.png, https://imgur.com/J0be6lz.png, https://imgur.com/Uiwo7vp.png |
| Disable Legacy API in Webflow to ensure webhook data compatibility.                                            | Sticky Note2 includes a screenshot and instructions: https://imgur.com/0tebypt.png                  |
| Google Sheets node supports automatic creation of columns when appending data to an empty or pre-existing sheet. | Sticky Note3                                                                                       |
| Estimated setup time: 5-10 minutes.                                                                            | Workflow description                                                                               |

---

This documentation provides a complete, structured, and clear reference for understanding, reproducing, and troubleshooting the “Send Webflow form data to Google Sheets” workflow in n8n.