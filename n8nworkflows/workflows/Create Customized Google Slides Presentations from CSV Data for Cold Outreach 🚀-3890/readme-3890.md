Create Customized Google Slides Presentations from CSV Data for Cold Outreach 🚀

https://n8nworkflows.xyz/workflows/create-customized-google-slides-presentations-from-csv-data-for-cold-outreach----3890


# Create Customized Google Slides Presentations from CSV Data for Cold Outreach 🚀

### 1. Workflow Overview

This workflow automates the creation of customized Google Slides presentations for cold outreach campaigns by processing CSV lead data. It targets sales and marketing teams who want to generate personalized slide decks efficiently and without manual editing for each prospect.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Detects new CSV files uploaded to Google Drive, filters for CSV file types, and downloads the CSV content.
- **1.2 Data Preparation:** Extracts lead data from the CSV, creates a new Google Sheet to log leads, and merges CSV data into the Sheet.
- **1.3 Presentation Generation:** Retrieves all leads from the Sheet, moves processed files to an archive folder, duplicates a master Google Slides template, customizes each copy with lead-specific data, and updates the Sheet with generated presentation IDs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block monitors a specified Google Drive folder for new files, filters to process only CSV files, and downloads the CSV content for further processing.

**Nodes Involved:**  
- New Leads Arrived  
- File Type?  
- Download by ID

**Node Details:**

- **New Leads Arrived**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches a specific Google Drive folder for new file creation events every minute.  
  - *Configuration:* Monitors folder ID `1GYT9Z8_BnqqY9dqsMpFWJqjeNVsq_xTY` (placeholder).  
  - *Input/Output:* Trigger node; outputs metadata of new files.  
  - *Edge Cases:* Possible missed triggers if polling interval too long; permissions errors if credentials lack folder access.

- **File Type?**  
  - *Type:* Switch  
  - *Role:* Filters incoming files by MIME type to separate CSV (`text/csv`) from XLSX files.  
  - *Configuration:* Two outputs: `csv` and `xlsx`, only CSV path is used.  
  - *Input:* File metadata from trigger.  
  - *Output:* Routes CSV files to download node.  
  - *Edge Cases:* Files with incorrect MIME types or extensions might be skipped or misrouted.

- **Download by ID**  
  - *Type:* Google Drive  
  - *Role:* Downloads the CSV file content based on the file ID from the trigger.  
  - *Configuration:* Uses OAuth2 credentials; file ID taken dynamically from trigger output.  
  - *Input:* File ID from File Type? node.  
  - *Output:* Binary data of the CSV file.  
  - *Edge Cases:* Download failures due to revoked credentials, file not found, or network issues.

---

#### 2.2 Data Preparation

**Overview:**  
This block extracts lead data from the CSV, creates a new Google Sheet to store the leads, merges the CSV data into the Sheet, and prepares data for presentation generation.

**Nodes Involved:**  
- Extract Information from CSV file  
- Create new Sheet  
- Combine Empty New Document with CSV Data  
- Merge Data for new Lead Document

**Node Details:**

- **Extract Information from CSV file**  
  - *Type:* Extract From File  
  - *Role:* Parses the downloaded CSV file to extract structured JSON data.  
  - *Configuration:* UTF-8 encoding, comma delimiter, header row enabled.  
  - *Input:* Binary CSV data from Download by ID.  
  - *Output:* JSON array of lead records.  
  - *Edge Cases:* Malformed CSV files causing parse errors.

- **Create new Sheet**  
  - *Type:* Google Sheets  
  - *Role:* Creates a new Google Spreadsheet to log lead data.  
  - *Configuration:* Spreadsheet title includes current date in Europe/Berlin timezone; creates a sheet named `sample_data`.  
  - *Input:* Triggered after CSV extraction.  
  - *Output:* Spreadsheet metadata including spreadsheetId and sheetId.  
  - *Edge Cases:* API quota limits, permission errors.

- **Combine Empty New Document with CSV Data**  
  - *Type:* Merge  
  - *Role:* Combines outputs from Create new Sheet and Extract Information nodes to synchronize data flow.  
  - *Configuration:* Uses data from second input (CSV data) for further processing.  
  - *Input:* Two inputs: new Sheet metadata and extracted CSV data.  
  - *Output:* Combined data for appending to Sheet.  
  - *Edge Cases:* Synchronization issues if one input is missing.

- **Merge Data for new Lead Document**  
  - *Type:* Google Sheets  
  - *Role:* Appends lead data extracted from CSV into the newly created Google Sheet.  
  - *Configuration:* Auto-maps CSV columns to Sheet columns; appends rows to the first sheet.  
  - *Input:* Combined data from previous node.  
  - *Output:* Confirmation of appended rows.  
  - *Edge Cases:* Data mismatches, rate limits, or append failures.

---

#### 2.3 Presentation Generation

**Overview:**  
This block retrieves all leads from the Google Sheet, archives the processed CSV file, duplicates a master Slides template for each lead, customizes the copied presentation by replacing placeholders with lead data, and updates the Sheet with the new presentation IDs.

**Nodes Involved:**  
- Get all Leads  
- MoveToLeadListFolder  
- Copy Presentation Template  
- Create Custom Presentation  
- Add Presentation ID to Lead List  
- Sticky Note (informational)

**Node Details:**

- **Get all Leads**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves all lead rows from the Google Sheet for processing.  
  - *Configuration:* Reads from the sheet created earlier using dynamic spreadsheet and sheet IDs.  
  - *Input:* Triggered after data merge.  
  - *Output:* JSON array of lead records.  
  - *Edge Cases:* Empty sheets, read permission errors.

- **MoveToLeadListFolder**  
  - *Type:* Google Drive  
  - *Role:* Moves the processed Google Sheet file to an archive folder to keep the workspace organized.  
  - *Configuration:* Moves file to folder ID `1-oMQTyijYXNmt-Dwh748JFjlDZVCu6ii` (placeholder).  
  - *Input:* Spreadsheet ID from Create new Sheet node.  
  - *Output:* Confirmation of move operation.  
  - *Edge Cases:* Move failures due to permission or folder not found.

- **Copy Presentation Template**  
  - *Type:* Google Drive  
  - *Role:* Copies a master Google Slides template to create a new presentation per lead.  
  - *Configuration:* Uses a fixed template ID `1FtMBECbZY-9gSW6L6DtioOOJv-V9LXduEjZO0pp-NmA` (placeholder); names new file with lead company and current date; places copy into a specific folder `1Yu2s72rgOJlz1-tMuzlaeN8UZezYftRT`.  
  - *Input:* Triggered after moving the lead list file.  
  - *Output:* New presentation file metadata including ID.  
  - *Edge Cases:* Copy failures, quota limits.

- **Create Custom Presentation**  
  - *Type:* Google Slides  
  - *Role:* Replaces placeholder tokens in the copied Slides presentation with lead-specific data.  
  - *Configuration:* Replaces `{COMPANYNAME}`, `{Testdurchgestrichen}`, and `{nichtdurchgestrichen}` with respective lead fields `Company`, `Full Name`, and `First Name`.  
  - *Input:* Presentation ID from copied template and lead data from Get all Leads.  
  - *Output:* Confirmation of text replacement.  
  - *Edge Cases:* Placeholder not found, API rate limits.

- **Add Presentation ID to Lead List**  
  - *Type:* Google Sheets  
  - *Role:* Updates the Google Sheet row for each lead with the URL or ID of the generated presentation.  
  - *Configuration:* Matches rows by `Email` column; writes `PresentationID` field.  
  - *Input:* Presentation ID from Create Custom Presentation and lead email from Get all Leads.  
  - *Output:* Confirmation of update.  
  - *Edge Cases:* Update conflicts, mismatched rows.

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Provides visual documentation inside the workflow canvas titled "Duplicate Template and Create Custom Presentations".  
  - *Input/Output:* None; purely informational.

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                                   | Input Node(s)                   | Output Node(s)                      | Sticky Note                                      |
|-------------------------------|------------------------|-------------------------------------------------|--------------------------------|-----------------------------------|-------------------------------------------------|
| New Leads Arrived              | Google Drive Trigger   | Detect new CSV uploads in Drive                  | -                              | File Type?                        |                                                 |
| File Type?                    | Switch                 | Filter for `.csv` files only                      | New Leads Arrived              | Download by ID                   |                                                 |
| Download by ID                | Google Drive           | Download the CSV content                          | File Type?                    | Extract Information from CSV file, Create new Sheet |                                                 |
| Extract Information from CSV file | Extract From File      | Parse CSV content into JSON                       | Download by ID                | Combine Empty New Document with CSV Data |                                                 |
| Create new Sheet              | Google Sheets          | Create a new Google Sheet for lead data          | Download by ID                | Combine Empty New Document with CSV Data | Sticky Note1: "# Create New Google Sheets and Insert Data from CSV file" |
| Combine Empty New Document with CSV Data | Merge                  | Combine new Sheet metadata with extracted CSV data | Extract Information from CSV file, Create new Sheet | Merge Data for new Lead Document |                                                 |
| Merge Data for new Lead Document | Google Sheets          | Append CSV lead data into Google Sheet            | Combine Empty New Document with CSV Data | Get all Leads                  |                                                 |
| Get all Leads                | Google Sheets          | Retrieve all leads from the Sheet                  | Merge Data for new Lead Document | MoveToLeadListFolder            |                                                 |
| MoveToLeadListFolder          | Google Drive           | Move processed Sheet to archive folder             | Get all Leads                | Copy Presentation Template       |                                                 |
| Copy Presentation Template    | Google Drive           | Duplicate master Slides template                    | MoveToLeadListFolder          | Create Custom Presentation       | Sticky Note: "# Duplicate Template and Create Custom Presentations" |
| Create Custom Presentation    | Google Slides          | Replace placeholders with lead-specific data       | Copy Presentation Template    | Add Presentation ID to Lead List |                                                 |
| Add Presentation ID to Lead List | Google Sheets          | Update Sheet with generated presentation IDs       | Create Custom Presentation, Get all Leads | -                             |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node ("New Leads Arrived")**  
   - Type: Google Drive Trigger  
   - Configure to watch a specific folder for new files (use your CSV upload folder ID).  
   - Set event to `fileCreated` and polling interval to every minute.  
   - Connect output to a Switch node.

2. **Add Switch Node ("File Type?")**  
   - Type: Switch  
   - Configure two outputs:  
     - Output `csv` for MIME type `text/csv`  
     - Output `xlsx` for MIME type `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` (optional)  
   - Connect `csv` output to Google Drive node for download.

3. **Add Google Drive Node ("Download by ID")**  
   - Operation: Download  
   - File ID: Use expression to get file ID from trigger node (`{{$json.id}}`).  
   - Connect output to Extract From File and Create Google Sheets nodes.

4. **Add Extract From File Node ("Extract Information from CSV file")**  
   - Configure encoding as UTF-8, delimiter as comma, header row enabled.  
   - Input: Binary data from Download node.  
   - Output: JSON lead data.

5. **Add Google Sheets Node ("Create new Sheet")**  
   - Operation: Create Spreadsheet  
   - Title: Use expression with current date (e.g., `Leads_{{ $now.setZone('Europe/Berlin').toFormat('yyyy-dd-MM') }}`).  
   - Create one sheet named `sample_data`.  
   - Output: Spreadsheet metadata.

6. **Add Merge Node ("Combine Empty New Document with CSV Data")**  
   - Mode: Choose Branch  
   - Use data from second input (CSV data).  
   - Connect inputs from Extract From File and Create new Sheet nodes.

7. **Add Google Sheets Node ("Merge Data for new Lead Document")**  
   - Operation: Append  
   - Document ID and Sheet ID: Use expressions from Create new Sheet node outputs.  
   - Mapping Mode: Auto-map input data to sheet columns.  
   - Input: Output from Merge node.

8. **Add Google Sheets Node ("Get all Leads")**  
   - Operation: Read Rows  
   - Document ID and Sheet ID: Use expressions from Create new Sheet node outputs.  
   - Connect output to Google Drive node for moving file.

9. **Add Google Drive Node ("MoveToLeadListFolder")**  
   - Operation: Move  
   - File ID: Use spreadsheet ID from Create new Sheet node.  
   - Folder ID: Set to your archive folder ID.  
   - Connect output to Google Drive node for copying presentation template.

10. **Add Google Drive Node ("Copy Presentation Template")**  
    - Operation: Copy  
    - File ID: Set to your master Google Slides template ID.  
    - Name: Use expression to include lead company and current date.  
    - Folder ID: Set to your presentations folder ID.  
    - Connect output to Google Slides node.

11. **Add Google Slides Node ("Create Custom Presentation")**  
    - Operation: Replace Text  
    - Presentation ID: Use copied file ID from previous node.  
    - Define text replacements mapping placeholders (e.g., `{COMPANYNAME}`) to lead data fields (e.g., `Company`).  
    - Connect output to Google Sheets node for updating lead list.

12. **Add Google Sheets Node ("Add Presentation ID to Lead List")**  
    - Operation: Update  
    - Document ID and Sheet ID: Use expressions from Create new Sheet node.  
    - Matching Column: `Email`  
    - Columns to update: `PresentationID` with the new presentation ID.  
    - Connect output to end of workflow.

13. **Add Sticky Notes (Optional)**  
    - Add visual sticky notes for documentation inside the workflow editor.

14. **Credentials Setup**  
    - Configure OAuth2 credentials for Google Drive, Sheets, and Slides nodes with appropriate scopes.  
    - Ensure credentials have access to all relevant folders and files.

15. **Testing**  
    - Upload a sample CSV file with headers matching your placeholders (e.g., Name, Company, Email).  
    - Monitor workflow execution and verify Sheets and Slides outputs.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The workflow uses placeholder variables for file and folder IDs to avoid exposing actual IDs.      | Security best practice                                                                                   |
| Disable “Keep Binary Data” in Copy Slides Template node to conserve memory during execution.        | Performance optimization                                                                                 |
| Add AI/natural-language nodes before slide creation to generate personalized content dynamically.  | Customization tip                                                                                         |
| Use SplitInBatches node to throttle API calls and avoid Google API rate limits.                     | Scalability and reliability enhancement                                                                 |
| Add error-handling branches to capture and log failed operations for robustness.                    | Recommended for production workflows                                                                     |
| Workflow credits: Designed for sales and marketing teams automating cold outreach presentations.    | Provided by workflow author                                                                               |
| Google Slides API documentation: https://developers.google.com/slides/api                           | Useful for advanced customization and troubleshooting                                                   |
| Google Sheets API documentation: https://developers.google.com/sheets/api                           | Useful for understanding data operations                                                                |
| Google Drive API documentation: https://developers.google.com/drive/api                             | Useful for file management operations                                                                    |

---

This comprehensive reference document captures the full structure, logic, and configuration details of the workflow "Create Customized Google Slides Presentations from CSV Data for Cold Outreach 🚀" to facilitate understanding, reproduction, and modification.