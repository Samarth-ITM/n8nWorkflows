Remove Duplicate Entries & Update Google Sheets Based on Profile URLs

https://n8nworkflows.xyz/workflows/remove-duplicate-entries---update-google-sheets-based-on-profile-urls-8132


# Remove Duplicate Entries & Update Google Sheets Based on Profile URLs

### 1. Workflow Overview

This workflow automates the process of cleaning and updating a Google Sheet by removing duplicate entries based on the "profileUrl" field. It is designed primarily for users managing lists or databases in Google Sheets where duplicate profile URLs need to be eliminated to ensure data integrity. The workflow performs the following logical blocks:

- **1.1 Input Trigger**: Manual initiation of the workflow.
- **1.2 Data Retrieval**: Fetching rows from a specified Google Sheet.
- **1.3 Duplicate Removal**: Identifying and removing duplicate entries based on the "profileUrl" column.
- **1.4 Data Conversion and Update**: Converting the cleaned data into a file format and updating the original Google Sheet with this deduplicated data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview**: This block initiates the workflow when a user manually clicks the "Execute workflow" button in n8n.
- **Nodes Involved**: `When clicking ‘Execute workflow’`
- **Node Details**:
  - **Type & Role**: Manual Trigger Node; starts the workflow on user command.
  - **Configuration**: No parameters required; default manual trigger.
  - **Expressions/Variables**: None.
  - **Connections**: Outputs to `Get row(s) in sheet`.
  - **Version Requirements**: Compatible with n8n version 0.150+.
  - **Potential Failures**: None typically, as it is manually triggered.
  - **Sub-workflow Reference**: None.

#### 2.2 Data Retrieval

- **Overview**: Retrieves all rows from a specific Google Sheet tab to process.
- **Nodes Involved**: `Get row(s) in sheet`
- **Node Details**:
  - **Type & Role**: Google Sheets Node; reads data from a spreadsheet.
  - **Configuration**:
    - Document ID: `1Kb2MV9Py1dWSp4fAoTDkIl1ha3j61CkgN7MalT5b4gU`
    - Sheet Name: Tab with ID `778504646` named "With Email/Unused"
    - Operation: Default (Get Rows)
  - **Expressions/Variables**: Uses a dynamic reference for document and sheet selection with caching for performance.
  - **Connections**: Input from manual trigger; output to `Remove Duplicates`.
  - **Version Requirements**: Uses Google Sheets OAuth2 credentials; requires valid authentication.
  - **Potential Failures**:
    - Authentication errors if OAuth token expires or is revoked.
    - API rate limits if called excessively.
    - Sheet/tab not found if document ID or sheet name changes.
  - **Sub-workflow Reference**: None.

#### 2.3 Duplicate Removal

- **Overview**: Processes the rows to identify and remove duplicate entries based on the "profileUrl" field.
- **Nodes Involved**: `Remove Duplicates`
- **Node Details**:
  - **Type & Role**: Remove Duplicates Node; filters out duplicate items in the dataset.
  - **Configuration**:
    - Comparison method: Selected fields.
    - Fields to Compare: `profileUrl`
  - **Expressions/Variables**: Field `profileUrl` must exist in the input data.
  - **Connections**: Input from `Get row(s) in sheet`; output to `Convert to File`.
  - **Version Requirements**: Requires n8n version that supports Remove Duplicates node v2 or later.
  - **Potential Failures**:
    - If `profileUrl` field is missing or inconsistently named, duplicates may not be correctly identified.
    - Data type inconsistencies in `profileUrl` field could cause false negatives.
  - **Sub-workflow Reference**: None.

#### 2.4 Data Conversion and Update

- **Overview**: Converts the deduplicated data into a file format and updates the original Google Sheet with this cleaned dataset.
- **Nodes Involved**: `Convert to File`, `Update file`
- **Node Details**:

  - **Convert to File**:
    - **Type & Role**: Convert to File Node; converts JSON data into a downloadable file.
    - **Configuration**: Default options; output likely CSV by default.
    - **Expressions/Variables**: Receives data from `Remove Duplicates`.
    - **Connections**: Input from `Remove Duplicates`; output to `Update file`.
    - **Version Requirements**: Compatible with n8n version 0.150+.
    - **Potential Failures**:
      - Data format inconsistencies could cause conversion issues.
      - Large datasets might cause performance delays.

  - **Update file**:
    - **Type & Role**: Google Drive Node; updates an existing Google Sheet with the new file content.
    - **Configuration**:
      - File ID: Same as the original Google Sheet (`1Kb2MV9Py1dWSp4fAoTDkIl1ha3j61CkgN7MalT5b4gU`)
      - Operation: Update file content.
      - ChangeFileContent: Enabled (true).
    - **Expressions/Variables**: Uses OAuth2 credentials for Google Drive (`smartaistructure`).
    - **Connections**: Input from `Convert to File`.
    - **Version Requirements**: Requires Google Drive OAuth2 credentials; API version 3 or above.
    - **Potential Failures**:
      - Authentication errors related to Google Drive OAuth2.
      - File lock or permissions issues (file edited or locked by another user).
      - Overwriting the spreadsheet with incompatible file format could corrupt data.
    - **Sub-workflow Reference**: None.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                       | Input Node(s)              | Output Node(s)           | Sticky Note                                        |
|----------------------------|-------------------------|-------------------------------------|----------------------------|--------------------------|---------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger           | Initiate workflow manually          |                            | Get row(s) in sheet       |                                                   |
| Get row(s) in sheet        | Google Sheets            | Fetch rows from Google Sheet        | When clicking ‘Execute workflow’ | Remove Duplicates         |                                                   |
| Remove Duplicates           | Remove Duplicates        | Remove duplicate rows by profileUrl | Get row(s) in sheet        | Convert to File           | ## Remove Duplicates from Google Sheets           |
| Convert to File            | Convert to File          | Convert data to file format          | Remove Duplicates          | Update file               | ## Download data as a CSV and update the Google sheets |
| Update file                | Google Drive             | Update Google Sheet with cleaned data | Convert to File            |                          |                                                   |
| Sticky Note                | Sticky Note              | Documentation note                   |                            |                          | ## Remove Duplicates from Google Sheets           |
| Sticky Note1               | Sticky Note              | Documentation note                   |                            |                          | ## Download data as a CSV and update the Google sheets |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Name: `When clicking ‘Execute workflow’`
   - No special parameters needed.
   - Position: Start node.

2. **Add Google Sheets Node to Get Rows**
   - Type: Google Sheets
   - Name: `Get row(s) in sheet`
   - Operation: Get Rows
   - Document ID: `1Kb2MV9Py1dWSp4fAoTDkIl1ha3j61CkgN7MalT5b4gU`
   - Sheet Name/ID: `778504646` ("With Email/Unused")
   - Credential: Connect with OAuth2 credentials authorized to access the sheet (e.g., `smartaistructure@gmail.com`).
   - Connect output from Manual Trigger.

3. **Add Remove Duplicates Node**
   - Type: Remove Duplicates
   - Name: `Remove Duplicates`
   - Comparison: Selected fields
   - Fields to Compare: `profileUrl`
   - Connect input to `Get row(s) in sheet`.

4. **Add Convert to File Node**
   - Type: Convert to File
   - Name: `Convert to File`
   - Default settings (usually converts JSON to CSV).
   - Connect input to `Remove Duplicates`.

5. **Add Google Drive Node to Update File**
   - Type: Google Drive
   - Name: `Update file`
   - Operation: Update
   - File ID: Same as Google Sheet (`1Kb2MV9Py1dWSp4fAoTDkIl1ha3j61CkgN7MalT5b4gU`)
   - Change File Content: Enabled (true)
   - Credential: Use Google Drive OAuth2 credentials (e.g., `smartaistructure`).
   - Connect input to `Convert to File`.

6. **Add Sticky Notes (Optional for Documentation)**
   - Add two Sticky Notes:
     - One covering nodes from manual trigger through Remove Duplicates titled "Remove Duplicates from Google Sheets".
     - One covering Convert to File and Update file nodes titled "Download data as a CSV and update the Google sheets".

7. **Save and Test Workflow**
   - Run manually to ensure the process:
     - Fetches rows.
     - Removes duplicates based on `profileUrl`.
     - Converts data to file.
     - Updates the Google Sheet with cleaned data.

---

### 5. General Notes & Resources

| Note Content                                           | Context or Link                                                                                 |
|--------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow title: "Remove Duplicate Entries & Update Google Sheets Based on Profile URLs" |                                                                                                 |
| Workflow handles data integrity by eliminating duplicate profile URLs in Google Sheets |                                                                                                 |
| Google Sheets and Google Drive OAuth2 credentials must have sufficient permissions to read and update the spreadsheet |                                                                                                 |
| Sticky notes provide contextual documentation within the workflow editor for clarity |                                                                                                 |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.