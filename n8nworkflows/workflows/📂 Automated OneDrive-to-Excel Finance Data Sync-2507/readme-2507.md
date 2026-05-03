📂 Automated OneDrive-to-Excel Finance Data Sync

https://n8nworkflows.xyz/workflows/---automated-onedrive-to-excel-finance-data-sync-2507


# 📂 Automated OneDrive-to-Excel Finance Data Sync

### 1. Workflow Overview

This workflow automates the synchronization of CSV files uploaded to a designated Microsoft OneDrive folder with an Excel worksheet. It is designed for scenarios such as portfolio tracking, inventory management, or any use case requiring continuous, automated updates from CSV data files to a structured Excel table.

The workflow is logically divided into the following blocks:

- **1.1 OneDrive File Monitoring and Retrieval:** Watches a OneDrive folder for new files, retrieves metadata and downloads the files.
- **1.2 File Validation and Data Extraction:** Verifies the file format (CSV) and extracts the data contained within.
- **1.3 Data Cleaning and Preparation:** Cleans and transforms extracted CSV data to match the Excel table schema.
- **1.4 Excel Data Update:** Updates an Excel worksheet with new or modified records based on a unique identifier.
- **1.5 Post-Processing and Cleanup:** Handles logging and housekeeping tasks such as deleting processed files or handling non-CSV files gracefully.

---

### 2. Block-by-Block Analysis

#### 2.1 OneDrive File Monitoring and Retrieval

**Overview:**  
This block continuously monitors a specified OneDrive folder for new files. Upon detection, it fetches file metadata and downloads the file for further processing.

**Nodes Involved:**  
- Triggers if new file in watched folder  
- Logs the update on sheet  
- Gets the new file infos  
- Downloads the new file  

**Node Details:**

- **Triggers if new file in watched folder**  
  - *Type:* Microsoft OneDrive Trigger  
  - *Role:* Initiates workflow when a new file appears in the monitored OneDrive folder.  
  - *Configuration:* Default trigger settings, monitors a specific folder at a frequency of every minute.  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Metadata of newly added file  
  - *Failure Modes:* Network issues, authorization errors with OneDrive account.  
  - *Sub-Workflow:* None  

- **Logs the update on sheet**  
  - *Type:* Microsoft Excel  
  - *Role:* Logs the event of new file detection into an Excel sheet for audit or tracking purposes.  
  - *Configuration:* Points to a specific Excel workbook and worksheet for logging.  
  - *Inputs:* Trigger output (file metadata)  
  - *Outputs:* Confirmation of logging  
  - *Failure Modes:* Excel API errors, file permission issues.  
  - *Sub-Workflow:* None  

- **Gets the new file infos**  
  - *Type:* Microsoft OneDrive  
  - *Role:* Retrieves detailed information about the detected file, such as its path and properties.  
  - *Configuration:* Uses file ID from trigger to query OneDrive API.  
  - *Inputs:* Output of logging node  
  - *Outputs:* Full file metadata  
  - *Failure Modes:* File may have been deleted or moved after trigger, API rate limits.  
  - *Sub-Workflow:* None  

- **Downloads the new file**  
  - *Type:* Microsoft OneDrive  
  - *Role:* Downloads the actual file content for processing.  
  - *Configuration:* Uses file metadata to download the file content.  
  - *Inputs:* File metadata from previous node  
  - *Outputs:* Binary data of the file  
  - *Failure Modes:* Large file download timeout, network issues, file access restrictions.  
  - *Sub-Workflow:* None  

---

#### 2.2 File Validation and Data Extraction

**Overview:**  
This block confirms whether the downloaded file is a CSV. If so, it extracts the data; if not, it terminates the workflow with an error.

**Nodes Involved:**  
- Checks if it's CSV format  
- Extracts data from csv  
- Not CSV  

**Node Details:**

- **Checks if it's CSV format**  
  - *Type:* If node (conditional branching)  
  - *Role:* Checks the file extension or MIME type to confirm CSV format.  
  - *Configuration:* Condition checks the file's extension equals ".csv" or MIME type indicating CSV.  
  - *Inputs:* Binary file from "Downloads the new file" node  
  - *Outputs:*  
    - True branch: proceed to data extraction  
    - False branch: go to error termination  
  - *Failure Modes:* Incorrect file metadata leading to false negatives, malformed file names.  
  - *Sub-Workflow:* None  

- **Extracts data from csv**  
  - *Type:* Extract From File  
  - *Role:* Parses the CSV file content into structured data rows and columns.  
  - *Configuration:* Default CSV parsing settings; assumes headers present in the first row.  
  - *Inputs:* Binary CSV file from previous node  
  - *Outputs:* JSON array of rows with column headers as keys  
  - *Failure Modes:* Malformed CSV, encoding issues, large file parsing delays.  
  - *Sub-Workflow:* None  

- **Not CSV**  
  - *Type:* Stop and Error  
  - *Role:* Terminates the workflow with an error message if the file is not CSV.  
  - *Configuration:* Default error message indicating invalid file type.  
  - *Inputs:* False branch from "Checks if it's CSV format"  
  - *Outputs:* None (ends workflow)  
  - *Failure Modes:* N/A (designed to handle failures gracefully)  
  - *Sub-Workflow:* None  

---

#### 2.3 Data Cleaning and Preparation

**Overview:**  
Cleans extracted CSV data (e.g., removes irrelevant headers, normalizes fields) and prepares it for Excel insertion by mapping to the required schema.

**Nodes Involved:**  
- Cleans the output  
- Prepares the fields to put in the excel table  

**Node Details:**

- **Cleans the output**  
  - *Type:* Code (JavaScript)  
  - *Role:* Performs custom data transformation and cleaning on the extracted CSV data.  
  - *Configuration:* Custom script that removes unnecessary headers, trims whitespace, filters invalid rows, or formats fields as needed.  
  - *Inputs:* JSON data from "Extracts data from csv"  
  - *Outputs:* Cleaned and normalized data ready for Excel  
  - *Failure Modes:* Script errors, unexpected data formats causing exceptions.  
  - *Sub-Workflow:* None  

- **Prepares the fields to put in the excel table**  
  - *Type:* Set  
  - *Role:* Maps cleaned data fields to the exact column names and formats expected by the Excel table.  
  - *Configuration:* Defines key-value pairs or expressions to set fields such as "Ticker/ISIN" and other columns.  
  - *Inputs:* Cleaned data from previous node  
  - *Outputs:* Data structured to precisely match the Excel table schema  
  - *Failure Modes:* Incorrect mappings leading to data misalignment, missing required fields.  
  - *Sub-Workflow:* None  

---

#### 2.4 Excel Data Update

**Overview:**  
Updates the Excel worksheet with the prepared data, adding only new or changed rows based on the unique identifier ("Ticker/ISIN"). Also manages file and data synchronization states.

**Nodes Involved:**  
- Updates the excel table  
- Microsoft OneDrive2  
- No Operation, do nothing  

**Node Details:**

- **Updates the excel table**  
  - *Type:* Microsoft Excel  
  - *Role:* Upserts (updates or inserts) rows into the Excel table based on unique keys.  
  - *Configuration:* Target workbook and worksheet specified; key field set as "Ticker/ISIN" to detect changes; data from prepared fields node.  
  - *Inputs:* Structured data from "Prepares the fields to put in the excel table"  
  - *Outputs:* Confirmation of update, updated rows info  
  - *Failure Modes:* Excel API errors, data conflicts, rate limits.  
  - *Sub-Workflow:* None  

- **Microsoft OneDrive2**  
  - *Type:* Microsoft OneDrive  
  - *Role:* Handles post-update actions such as file deletion or moving processed files (configured to continue on errors).  
  - *Configuration:* May be set to delete or archive processed files to keep OneDrive organized.  
  - *Inputs:* Output from "Updates the excel table"  
  - *Outputs:* Confirmation or error status  
  - *Failure Modes:* Deletion failures due to permissions or file locks.  
  - *Sub-Workflow:* None  

- **No Operation, do nothing**  
  - *Type:* NoOp (No Operation)  
  - *Role:* Acts as a placeholder or end node for alternative flow branches.  
  - *Configuration:* No parameters; simply passes data through or ends branch.  
  - *Inputs:* Secondary output from "Microsoft OneDrive2"  
  - *Outputs:* None  
  - *Failure Modes:* N/A  
  - *Sub-Workflow:* None  

---

#### 2.5 Post-Processing and Cleanup

**Overview:**  
Logs updates back to OneDrive and Excel for monitoring and traceability, and manages workflow termination states for non-CSV files or errors.

**Nodes Involved:**  
- Not CSV (already covered in 2.2)  
- Sticky Notes (contextual guidance)  

**Node Details:**

- **Sticky Notes**  
  - *Type:* Sticky Note  
  - *Role:* Provide internal documentation or reminders to users about the workflow steps or configuration.  
  - *Configuration:* Contains no content in this workflow but present for potential annotations.  
  - *Inputs:* None  
  - *Outputs:* None  
  - *Failure Modes:* N/A  
  - *Sub-Workflow:* None  

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                          | Input Node(s)                       | Output Node(s)                       | Sticky Note |
|-----------------------------------|----------------------------|----------------------------------------|-----------------------------------|------------------------------------|-------------|
| Triggers if new file in watched folder | Microsoft OneDrive Trigger  | Watches OneDrive folder for new files  | None                              | Logs the update on sheet            |             |
| Logs the update on sheet           | Microsoft Excel             | Logs new file detection event           | Triggers if new file in watched folder | Gets the new file infos             |             |
| Gets the new file infos            | Microsoft OneDrive          | Retrieves detailed file metadata        | Logs the update on sheet           | Downloads the new file              |             |
| Downloads the new file             | Microsoft OneDrive          | Downloads the file content               | Gets the new file infos            | Checks if it's CSV format           |             |
| Checks if it's CSV format          | If                         | Validates file format is CSV             | Downloads the new file             | Extracts data from csv (true), Not CSV (false) |             |
| Extracts data from csv             | Extract From File           | Parses CSV content into structured data | Checks if it's CSV format (true)  | Cleans the output                  |             |
| Cleans the output                  | Code                       | Cleans and normalizes CSV data           | Extracts data from csv             | Prepares the fields to put in the excel table |             |
| Prepares the fields to put in the excel table | Set                        | Maps data to Excel table schema          | Cleans the output                  | Updates the excel table             |             |
| Updates the excel table            | Microsoft Excel             | Upserts data into Excel table             | Prepares the fields to put in the excel table | Microsoft OneDrive2                 |             |
| Microsoft OneDrive2                | Microsoft OneDrive          | Manages post-update file actions         | Updates the excel table            | No Operation, do nothing (secondary) |             |
| No Operation, do nothing           | NoOp                       | Placeholder / end node                    | Microsoft OneDrive2 (secondary output) | None                           |             |
| Not CSV                           | Stop and Error             | Stops workflow if file is not CSV         | Checks if it's CSV format (false) | None                              |             |
| Sticky Note                       | Sticky Note                | Workflow annotation (empty)               | None                             | None                              |             |
| Sticky Note1                      | Sticky Note                | Workflow annotation (empty)               | None                             | None                              |             |
| Sticky Note2                      | Sticky Note                | Workflow annotation (empty)               | None                             | None                              |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the OneDrive Trigger Node:**  
   - Add a *Microsoft OneDrive Trigger* node named "Triggers if new file in watched folder".  
   - Configure it to monitor the designated OneDrive folder for new files, set trigger frequency to every minute.  
   - Connect no inputs as it is the start of the workflow.

2. **Add Microsoft Excel Node for Logging:**  
   - Add a *Microsoft Excel* node named "Logs the update on sheet".  
   - Configure it to append a new row logging file detection data including timestamp and file metadata.  
   - Connect output of the trigger node to this node.

3. **Add OneDrive Metadata Retrieval Node:**  
   - Add a *Microsoft OneDrive* node named "Gets the new file infos".  
   - Configure it to retrieve detailed metadata using the file ID from the logging node output.  
   - Connect logging node output to this node.

4. **Add OneDrive Download Node:**  
   - Add a *Microsoft OneDrive* node named "Downloads the new file".  
   - Configure it to download the file binary content using the metadata.  
   - Connect "Gets the new file infos" output to this node.

5. **Add Conditional Node to Check File Format:**  
   - Add an *If* node named "Checks if it's CSV format".  
   - Configure condition to check if the file extension or MIME type equals "csv".  
   - Connect "Downloads the new file" output to this node.

6. **Add Extract From File Node:**  
   - Add an *Extract From File* node named "Extracts data from csv".  
   - Configure it to parse CSV content assuming headers are present.  
   - Connect the true output branch of the "Checks if it's CSV format" node here.

7. **Add Code Node to Clean Data:**  
   - Add a *Code* node named "Cleans the output".  
   - Implement JavaScript code to remove irrelevant headers, trim fields, filter invalid rows, or normalize data.  
   - Connect "Extracts data from csv" output here.

8. **Add Set Node to Prepare Data for Excel:**  
   - Add a *Set* node named "Prepares the fields to put in the excel table".  
   - Map cleaned data fields to match the exact Excel columns, including the unique key "Ticker/ISIN".  
   - Connect "Cleans the output" to this node.

9. **Add Microsoft Excel Node to Update Table:**  
   - Add a *Microsoft Excel* node named "Updates the excel table".  
   - Configure it to upsert (insert or update) rows based on the "Ticker/ISIN" unique identifier.  
   - Specify workbook, worksheet, and table to update.  
   - Connect "Prepares the fields to put in the excel table" output here.

10. **Add Microsoft OneDrive Node for Post-Processing:**  
    - Add a *Microsoft OneDrive* node named "Microsoft OneDrive2".  
    - Configure it to delete or archive the processed file to keep OneDrive tidy.  
    - Set error handling to "Continue on Error" to prevent workflow failure if deletion fails.  
    - Connect "Updates the excel table" output here.

11. **Add No Operation Node:**  
    - Add a *NoOp* node named "No Operation, do nothing".  
    - Connect the secondary output of "Microsoft OneDrive2" to this node to end the workflow branch gracefully.

12. **Add Stop and Error Node for Invalid Files:**  
    - Add a *Stop and Error* node named "Not CSV".  
    - Configure it with an error message indicating the file is not CSV and workflow will stop.  
    - Connect the false output branch of "Checks if it's CSV format" here.

13. **(Optional) Add Sticky Notes:**  
    - Add *Sticky Note* nodes at strategic points for documentation or reminders.  
    - Populate them with any workflow-specific notes or instructions for future maintainers.

14. **Credential Setup:**  
    - For all Microsoft OneDrive and Excel nodes, configure credentials using OAuth2 authentication with your Microsoft account.  
    - Ensure the connected account has permissions to read/write files and modify Excel workbooks.

15. **Testing and Validation:**  
    - Test the workflow by uploading CSV and non-CSV files to the watched OneDrive folder.  
    - Verify logs are created, data is correctly inserted into Excel, invalid files cause errors, and processed files are deleted or archived.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                    |
|------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow is optimized for OneDrive and Excel online integration via Microsoft Graph API.   | n8n Documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.microsoftOneDrive/ |
| Ensure the unique key field "Ticker/ISIN" is consistent between CSV and Excel for accurate updates. | Workflow setup best practice                                    |
| Monitor Microsoft API rate limits to avoid throttling during frequent file syncs.               | Microsoft Graph API limits: https://learn.microsoft.com/en-us/graph/throttling |
| Use the built-in error handling in OneDrive nodes to continue workflow execution on minor failures. | Prevents workflow stoppage on temporary network or API issues   |

---

This comprehensive analysis and reconstruction guide should allow any advanced user or AI agent to understand, reproduce, and maintain the "Automated OneDrive-to-Excel Finance Data Sync" workflow with confidence and precision.