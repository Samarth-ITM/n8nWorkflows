Bulk JSON File Downloader from Google Sheet to Local Folders

https://n8nworkflows.xyz/workflows/bulk-json-file-downloader-from-google-sheet-to-local-folders-6366


# Bulk JSON File Downloader from Google Sheet to Local Folders

### 1. Workflow Overview

This workflow automates the process of downloading multiple JSON files listed in a Google Sheet and saving them into local folders. It is designed for bulk operations where each row in the sheet provides a JSON file URL and a corresponding name or folder structure. The workflow logically divides into the following blocks:

- **1.1 Trigger & Data Retrieval:** Initiates the flow manually and fetches the rows from Google Sheets containing the JSON file information.
- **1.2 Data Extraction & Preparation:** Extracts relevant fields (name, URL) from the sheet data and transforms the URL for download.
- **1.3 Batch Processing & HTTP Download:** Loops over the prepared items in batches, performs HTTP requests to download JSON content.
- **1.4 File Writing:** Saves the downloaded JSON content into local disk files, organized per the specified names or folders.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Retrieval

- **Overview:**  
  This block starts the workflow manually and retrieves all relevant rows from the configured Google Sheet.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Get row(s) in sheet

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point initiated by user interaction  
    - *Config:* Default manual trigger with no parameters  
    - *Connections:* Outputs to "Get row(s) in sheet"  
    - *Potential Failures:* None typical, except UI failure or manual start omissions

  - **Get row(s) in sheet**  
    - *Type:* Google Sheets node (v4.6)  
    - *Role:* Retrieves rows from a specified Google Sheet  
    - *Config:* Parameters not explicitly shown, but typically includes Sheet ID, Sheet Name, and range or filtering to get JSON URLs and names  
    - *Connections:* Outputs to "Take name and URL"  
    - *Potential Failures:* Authentication errors, API rate limits, empty or malformed sheet data

#### 2.2 Data Extraction & Preparation

- **Overview:**  
  Extracts the file name and URL from each row, then processes the URL to generate a proper download URL.

- **Nodes Involved:**  
  - Take name and URL  
  - url to downloadurl

- **Node Details:**

  - **Take name and URL**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Parses input data to select only the necessary fields (e.g., file name and JSON URL)  
    - *Config:* Contains code to transform raw Google Sheets data into structured objects with name and URL keys  
    - *Connections:* Outputs to "url to downloadurl"  
    - *Potential Failures:* Code errors if input data structure changes or missing keys

  - **url to downloadurl**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Converts or cleans the provided URL to a valid download URL format  
    - *Config:* Contains URL transformation logic (e.g., replacing strings, appending query params)  
    - *Connections:* Outputs to "Loop Over Items"  
    - *Potential Failures:* Invalid URL formats, string manipulation errors

#### 2.3 Batch Processing & HTTP Download

- **Overview:**  
  Processes items in batches to efficiently download each JSON file via HTTP requests.

- **Nodes Involved:**  
  - Loop Over Items  
  - HTTP Request  
  - Code2

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* SplitInBatches node (v3)  
    - *Role:* Controls batch size and iteration over the list of URLs to download  
    - *Config:* Likely configured with batch size (default or user-defined)  
    - *Connections:*  
      - Main output (batch items) to "HTTP Request"  
      - Secondary output (empty) to handle batch completion or parallel flow  
    - *Potential Failures:* Batch size misconfiguration, empty input arrays

  - **HTTP Request**  
    - *Type:* HTTP Request node (v4.2)  
    - *Role:* Makes GET requests to download JSON content from each URL  
    - *Config:*  
      - HTTP Method: GET  
      - URL: Set dynamically from input data (download URL)  
      - Response Format: Likely JSON or raw content  
      - Authentication: None specified, assuming public URLs  
    - *Connections:* Outputs to "Code2"  
    - *Potential Failures:* Network errors, 404 or 403 HTTP errors, timeouts, invalid JSON responses

  - **Code2**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Processes the HTTP response, possibly formatting or preparing data for file writing  
    - *Config:* Contains code to handle response data, may rename fields or prepare metadata  
    - *Connections:* Outputs to "Read/Write Files from Disk"  
    - *Potential Failures:* Parsing errors if response data is malformed or unexpected

#### 2.4 File Writing

- **Overview:**  
  Saves the downloaded JSON data into the local file system, organizing files based on the given names and folder structure.

- **Nodes Involved:**  
  - Read/Write Files from Disk

- **Node Details:**

  - **Read/Write Files from Disk**  
    - *Type:* ReadWriteFile node (v1)  
    - *Role:* Writes JSON content to local disk files using the specified path and filename  
    - *Config:* Parameters likely include:  
      - Operation: Write  
      - File path: Derived from the name or folder structure extracted earlier  
      - Content: JSON data from previous node  
      - Encoding: UTF-8 or similar  
    - *Connections:* Outputs back to "Loop Over Items" (to iterate next batch)  
    - *Potential Failures:* File system permissions, invalid file paths, disk space issues

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                       | Input Node(s)              | Output Node(s)              | Sticky Note                    |
|----------------------------|------------------------|------------------------------------|----------------------------|----------------------------|-------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Workflow entry point                | —                          | Get row(s) in sheet         |                               |
| Get row(s) in sheet        | Google Sheets (v4.6)    | Retrieve rows from Google Sheet     | When clicking ‘Execute workflow’ | Take name and URL            |                               |
| Take name and URL          | Code (v2)               | Extract file name and URL           | Get row(s) in sheet         | url to downloadurl          |                               |
| url to downloadurl         | Code (v2)               | Transform URL for download          | Take name and URL           | Loop Over Items             |                               |
| Loop Over Items            | SplitInBatches (v3)     | Batch processing of download items | url to downloadurl          | HTTP Request, (secondary empty output) |                               |
| HTTP Request               | HTTP Request (v4.2)     | Download JSON content via HTTP      | Loop Over Items             | Code2                      |                               |
| Code2                      | Code (v2)               | Process HTTP response data          | HTTP Request               | Read/Write Files from Disk  |                               |
| Read/Write Files from Disk | ReadWriteFile (v1)      | Save JSON files locally             | Code2                      | Loop Over Items             |                               |
| Sticky Note                | Sticky Note             | —                                  | —                          | —                          |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Google Sheets Node ("Get row(s) in sheet"):**  
   - Connect from Manual Trigger.  
   - Configure credentials for Google API (OAuth2 or service account).  
   - Set Spreadsheet ID and Sheet Name containing JSON file info.  
   - Choose to get all rows or specific range where URLs and names are stored.

3. **Add Code Node ("Take name and URL"):**  
   - Connect from Google Sheets node.  
   - Write JavaScript code to extract necessary fields such as file name and URL from each row.  
   - Example logic: Map input items to objects with `name` and `url` properties.

4. **Add Code Node ("url to downloadurl"):**  
   - Connect from "Take name and URL".  
   - Write code to transform URLs to valid download URLs if needed (e.g., replacing parts of the string).  
   - Output the updated URLs for download.

5. **Add SplitInBatches Node ("Loop Over Items"):**  
   - Connect from "url to downloadurl".  
   - Configure batch size (e.g., 1 for sequential downloads or larger for parallelism).  
   - This node manages processing each JSON URL in batches.

6. **Add HTTP Request Node ("HTTP Request"):**  
   - Connect from "Loop Over Items" (main output).  
   - Set method to GET.  
   - Use expression to set URL dynamically from input data (e.g., `{{$json["url"]}}`).  
   - Ensure no authentication unless required.  
   - Set response format to JSON or raw (depending on expected content).

7. **Add Code Node ("Code2"):**  
   - Connect from "HTTP Request".  
   - Write code to handle the HTTP response data, possibly preparing file content or metadata for writing.  
   - Pass on the file name and content.

8. **Add ReadWriteFile Node ("Read/Write Files from Disk"):**  
   - Connect from "Code2".  
   - Set operation to "Write".  
   - Configure file path using the file name from input data, optionally including folders.  
   - Set file content as the JSON data received.  
   - Encoding should be UTF-8.

9. **Connect "Read/Write Files from Disk" back to "Loop Over Items":**  
   - Connect main output to the secondary input of "Loop Over Items" to continue processing next batch.

10. **Set Credentials:**  
    - Google Sheets node: Google API OAuth2 or Service Account credentials.  
    - HTTP Request node: Usually no credentials required unless URLs require auth.

11. **Test the workflow:**  
    - Execute manually and verify JSON files are downloaded and saved correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow assumes public or accessible URLs for JSON files; private URLs may require authentication setup. | HTTP Request node configuration section.                                                                 |
| The batch processing loop enables controlled downloading to avoid API rate limits or network overload.          | SplitInBatches node documentation.                                                                        |
| Ensure local disk permissions allow write operations to the target folder paths used by the workflow.           | Operating system file system permissions.                                                                 |
| Google Sheets API quota and authorization must be properly configured to prevent runtime errors.                | Google Sheets API official docs: https://developers.google.com/sheets/api                                     |
| For advanced use, folder structure creation can be implemented via additional code nodes before file writing.   | n8n community forum and documentation: https://docs.n8n.io/nodes/n8n-nodes-base.readWriteFile/             |

---

*Disclaimer: The text provided originates solely from an n8n automated workflow and complies fully with content policies. It contains no illegal, offensive, or protected material. All data processed is legal and public.*