Generate Dynamic Line Chart from JSON Data to Upload to Google Drive

https://n8nworkflows.xyz/workflows/generate-dynamic-line-chart-from-json-data-to-upload-to-google-drive-3578


# Generate Dynamic Line Chart from JSON Data to Upload to Google Drive

### 1. Workflow Overview

This workflow dynamically generates a line chart image from JSON data and uploads the resulting image to Google Drive. It is designed for use cases such as automated report generation, data visualization from APIs or databases, and embedding charts into internal tools or notifications. The workflow is structured into four main logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Preparation:** Setting up sample JSON data containing chart labels and sales data.
- **1.3 Chart Generation:** Using the QuickChart node to create a line chart image based on the JSON data.
- **1.4 Upload to Google Drive:** Uploading the generated chart image to a specified Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually, allowing the user to trigger the process on demand.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)

- **Node Details:**

  - **Node Name:** When clicking ‘Test workflow’  
    - **Type:** Manual Trigger  
    - **Role:** Starts the workflow manually when the user clicks 'Test workflow'.  
    - **Configuration:** Default manual trigger with no parameters.  
    - **Expressions/Variables:** None.  
    - **Input Connections:** None (entry point).  
    - **Output Connections:** Connects to `Edit Fields: Set JSON data to test`.  
    - **Version Requirements:** Compatible with all n8n versions supporting manual triggers.  
    - **Potential Failures:** None expected; manual trigger is stable.  
    - **Sub-workflow:** None.

---

#### 1.2 Data Preparation

- **Overview:**  
  This block prepares sample JSON data representing chart labels and sales data, simulating input data for the chart generation.

- **Nodes Involved:**  
  - `Edit Fields: Set JSON data to test` (Set Node)

- **Node Details:**

  - **Node Name:** Edit Fields: Set JSON data to test  
    - **Type:** Set  
    - **Role:** Defines a JSON object named `jsonData` with fields for report title, labels, and sales data arrays.  
    - **Configuration:**  
      - Sets a single field `jsonData` of type Object with the value:  
        ```json
        {
          "reportTitle": "Quarterly Sales",
          "labels": ["Q1", "Q2", "Q3", "Q4"],
          "salesData": [1250, 1800, 1550, 2100]
        }
        ```  
      - No additional options or transformations.  
    - **Expressions/Variables:** The JSON object is assigned directly as an expression.  
    - **Input Connections:** Receives from `When clicking ‘Test workflow’`.  
    - **Output Connections:** Connects to `QuickChart`.  
    - **Version Requirements:** Uses Set node version 3.4.  
    - **Potential Failures:**  
      - Misformatted JSON could cause errors downstream.  
      - Mismatch in array lengths between `labels` and `salesData` may cause chart inconsistencies.  
    - **Sub-workflow:** None.

---

#### 1.3 Chart Generation

- **Overview:**  
  Generates a line chart image dynamically using the QuickChart node, based on the JSON data prepared earlier.

- **Nodes Involved:**  
  - `QuickChart`

- **Node Details:**

  - **Node Name:** QuickChart  
    - **Type:** QuickChart  
    - **Role:** Creates a line chart image using data and labels from the JSON object.  
    - **Configuration:**  
      - `Chart Type`: `line`  
      - `Labels Mode`: `array` (labels are provided as an array)  
      - `Labels Array`: Expression `={{ $json.jsonData.labels }}` to dynamically read labels  
      - `Data`: Expression `={{ $json.jsonData.salesData }}` to dynamically read sales data points  
      - `Chart Options`: Empty (default appearance)  
      - `Dataset Options`: Empty (single dataset configured via top-level Data field)  
    - **Expressions/Variables:**  
      - Labels and data are dynamically pulled from the JSON object using expressions.  
    - **Input Connections:** Receives from `Edit Fields: Set JSON data to test`.  
    - **Output Connections:** Connects to `Google Drive: Upload File`.  
    - **Version Requirements:** Compatible with QuickChart node version 1.  
    - **Potential Failures:**  
      - If `labels` and `salesData` arrays differ in length, chart rendering may fail or be incorrect.  
      - Expression errors if `jsonData` is missing or malformed.  
      - Network or API errors if QuickChart service is unreachable.  
    - **Sub-workflow:** None.

---

#### 1.4 Upload to Google Drive

- **Overview:**  
  Uploads the generated chart image binary data to a specified folder in Google Drive, naming the file dynamically based on its extension.

- **Nodes Involved:**  
  - `Google Drive: Upload File`

- **Node Details:**

  - **Node Name:** Google Drive: Upload File  
    - **Type:** Google Drive  
    - **Role:** Uploads the binary image file to Google Drive.  
    - **Configuration:**  
      - `Name`: Expression `=chart.{{ $binary.data.fileExtension }}` dynamically names the file with the appropriate extension (e.g., `chart.png`).  
      - `Drive ID`: Selected as "My Drive" (default Google Drive root).  
      - `Folder ID`: Selected as "root" (default root folder).  
      - `Options`: Default (no additional options).  
    - **Credentials:** Uses OAuth2 credentials named "Google Drive account" (must be configured by user).  
    - **Expressions/Variables:** Uses binary data field `data` from QuickChart node output.  
    - **Input Connections:** Receives from `QuickChart`.  
    - **Output Connections:** None (end of workflow).  
    - **Version Requirements:** Google Drive node version 3.  
    - **Potential Failures:**  
      - Authentication errors if credentials are invalid or expired.  
      - Permission errors if folder access is restricted.  
      - Network or API rate limiting errors.  
      - File naming conflicts or invalid characters in file name.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                       | Node Type          | Functional Role                          | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                                          |
|--------------------------------|--------------------|----------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger     | Starts workflow manually                | None                           | Edit Fields: Set JSON data to test |                                                                                                                                      |
| Edit Fields: Set JSON data to test | Set                | Defines sample JSON data for chart      | When clicking ‘Test workflow’  | QuickChart                    |                                                                                                                                      |
| QuickChart                     | QuickChart         | Generates line chart image from JSON data | Edit Fields: Set JSON data to test | Google Drive: Upload File      |                                                                                                                                      |
| Google Drive: Upload File       | Google Drive       | Uploads chart image to Google Drive     | QuickChart                    | None                          |                                                                                                                                      |
| Sticky Note                    | Sticky Note        | Provides detailed usage and customization instructions | None                           | None                          | ## Chart Generator\n**Generate Dynamic Line Chart from JSON Data to Upload to Google Drive\n### How to Use & Customize\n\n* **Change Input Data:** Modify the `labels` and `salesData` arrays within the `Edit Fields: Set JSON data to test` node to use your own data. Ensure the number of labels matches the number of data points.\n* **Use Real Data Sources:** Replace the `Edit Fields: Set JSON data to test` node with nodes that fetch data from real sources like:\n    * HTTP Request (APIs)\n    * Postgres / MongoDB nodes (Databases)\n    * Google Sheets node\n    * Ensure the output data from your source node is formatted similarly (providing `labels` and `salesData` arrays). You might need another Set node to structure the data correctly before the QuickChart node.\n* **Change Chart Type:** In the QuickChart node, modify the `Chart Type` parameter (e.g., change from `line` to `bar`, `pie`, `doughnut`, etc.).\n* **Customize Chart Appearance:** Explore the `Chart Options` parameter within the QuickChart node to add titles, change colors, modify axes, etc., using QuickChart's standard JSON configuration options.\n* **Use Datasets (Recommended for Complex Charts):** For multiple lines/bars or more control, configure datasets explicitly in the QuickChart node:\n    * Remove the expression from the top-level `Data` field.\n    * Go to `Dataset Options` -&gt; `Add option` -&gt; `Add dataset`.\n    * Set the `Data` field within the dataset using an expression like `{{ $json.jsonData.salesData }}`.\n    * You can add multiple datasets this way.\n* **Change Output Destination:** Replace the `Google Drive: Upload File` node with other nodes to handle the chart image differently:\n    * `Write Binary File`: Save the chart to the local filesystem where n8n is running.\n    * `Slack` / `Discord` / `Telegram`: Send the chart to messaging platforms.\n    * `Move Binary Data`: Convert the image to Base64 to embed in HTML or return via webhook response. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node.  
   - Leave default settings. This node will start the workflow manually.

2. **Create Set Node to Define JSON Data**  
   - Add a **Set** node named `Edit Fields: Set JSON data to test`.  
   - Add a new field:  
     - Name: `jsonData`  
     - Type: `Object`  
     - Value (as JSON):  
       ```json
       {
         "reportTitle": "Quarterly Sales",
         "labels": ["Q1", "Q2", "Q3", "Q4"],
         "salesData": [1250, 1800, 1550, 2100]
       }
       ```  
   - No additional options needed.

3. **Create QuickChart Node to Generate Chart**  
   - Add a **QuickChart** node.  
   - Configure parameters:  
     - `Chart Type`: Select `line`.  
     - `Labels Mode`: Select `array`.  
     - `Labels Array`: Set expression to `{{ $json.jsonData.labels }}`.  
     - `Data`: Set expression to `{{ $json.jsonData.salesData }}`.  
     - Leave `Chart Options` and `Dataset Options` empty for default appearance.  
   - This node outputs the chart image as binary data in the `data` field.

4. **Create Google Drive Node to Upload Chart**  
   - Add a **Google Drive** node named `Google Drive: Upload File`.  
   - Set parameters:  
     - `Name`: Use expression `=chart.{{ $binary.data.fileExtension }}` to dynamically name the file with the correct extension.  
     - `Drive ID`: Select `My Drive` (default).  
     - `Folder ID`: Select `root` (default root folder).  
     - Leave other options default.  
   - Under **Credentials**, select or create Google Drive OAuth2 credentials with appropriate permissions to upload files.

5. **Connect Nodes in Sequence**  
   - Connect **Manual Trigger** → **Set Node** → **QuickChart** → **Google Drive**.

6. **Save and Test**  
   - Save the workflow.  
   - Click 'Test workflow' on the manual trigger node to run.  
   - Verify the chart image is generated and uploaded to your Google Drive root folder.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow demonstrates dynamic chart generation using QuickChart and uploading to Google Drive, suitable for automated reporting and data visualization tasks. You can customize the input data, chart type, appearance, and output destination as needed. For complex charts, use the dataset options in QuickChart for multiple datasets. Replace the sample data node with real data sources like HTTP requests, databases, or Google Sheets.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Workflow description and customization instructions embedded in the Sticky Note node.           |
| Google Drive credentials must be configured with OAuth2 and appropriate scopes to allow file uploads. Ensure the credentials are valid and have permission to write to the selected Drive and folder.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Credential setup requirement for Google Drive node.                                              |
| QuickChart supports various chart types and extensive customization via JSON options. Refer to [QuickChart documentation](https://quickchart.io/documentation/) for advanced chart configurations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | QuickChart node customization and documentation.                                                |
| For alternative output destinations, consider nodes like Write Binary File (local storage), Slack, Discord, Telegram (messaging platforms), or Move Binary Data (for Base64 encoding).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Suggestions for modifying output destination.                                                   |

---

This documentation provides a complete understanding of the workflow structure, node configurations, potential failure points, and instructions to reproduce or customize the workflow effectively.