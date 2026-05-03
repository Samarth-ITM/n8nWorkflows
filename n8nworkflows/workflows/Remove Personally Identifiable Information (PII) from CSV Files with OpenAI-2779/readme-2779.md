Remove Personally Identifiable Information (PII) from CSV Files with OpenAI

https://n8nworkflows.xyz/workflows/remove-personally-identifiable-information--pii--from-csv-files-with-openai-2779


# Remove Personally Identifiable Information (PII) from CSV Files with OpenAI

### 1. Workflow Overview

This workflow automates the process of removing Personally Identifiable Information (PII) from CSV files uploaded to a monitored Google Drive folder. It is designed for organizations or users who need to sanitize datasets by identifying and excluding PII columns before further processing or sharing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Detects new CSV files uploaded to a specific Google Drive folder and retrieves their metadata and content.
- **1.2 Data Extraction:** Extracts tabular data from the downloaded CSV file for analysis.
- **1.3 PII Identification via AI:** Uses OpenAI to analyze the data and identify columns containing PII.
- **1.4 Data Sanitization:** Removes the identified PII columns from the dataset and reconstructs a sanitized CSV file.
- **1.5 Output Upload:** Uploads the cleaned CSV file back to a designated Google Drive folder with a modified filename to preserve the original.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow when a new file is created in a monitored Google Drive folder. It retrieves the file's metadata and downloads the file content for further processing.

**Nodes Involved:**  
- Google Drive Trigger  
- Get filename  
- Google Drive

**Node Details:**

- **Google Drive Trigger**  
  - *Type:* Trigger node for Google Drive  
  - *Role:* Watches a specific Google Drive folder for newly created files (polls every minute).  
  - *Configuration:*  
    - Event: `fileCreated`  
    - Folder to watch: Specified by folder ID `"1-hRMnBRYgY6iVJ_youKMyPz83k9GAVYu"`  
    - Polling interval: Every minute  
  - *Input:* None (trigger)  
  - *Output:* File metadata (including file ID and name)  
  - *Potential Failures:* Authentication errors, API rate limits, folder permission issues.

- **Get filename**  
  - *Type:* SplitOut node  
  - *Role:* Extracts the original filename from the file metadata for later use.  
  - *Configuration:*  
    - Field to split out: `name` (filename)  
    - Destination field: `originalFilename`  
    - Executes once per trigger event  
  - *Input:* Output from Google Drive Trigger  
  - *Output:* JSON containing `originalFilename`  
  - *Potential Failures:* Missing or malformed filename field.

- **Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the file content using the file ID from the trigger.  
  - *Configuration:*  
    - Operation: `download`  
    - File ID: Dynamically set from trigger output (`{{$json.id}}`)  
    - Binary property name: `data` (where the file content is stored)  
  - *Input:* File metadata from Google Drive Trigger  
  - *Output:* Binary file content  
  - *Potential Failures:* File not found, permission denied, network issues.

---

#### 2.2 Data Extraction

**Overview:**  
Extracts structured tabular data from the downloaded CSV file to prepare it for AI analysis.

**Nodes Involved:**  
- Extract from File

**Node Details:**

- **Extract from File**  
  - *Type:* ExtractFromFile node  
  - *Role:* Parses the binary CSV file content into JSON objects representing rows and columns.  
  - *Configuration:* Default options (assumes CSV format)  
  - *Input:* Binary file content from Google Drive node  
  - *Output:* JSON array of rows with column headers as keys  
  - *Potential Failures:* Malformed CSV, unsupported file format, empty file.

---

#### 2.3 PII Identification via AI

**Overview:**  
Uses OpenAI's GPT model to analyze the extracted tabular data and identify columns containing PII. The output is a comma-separated list of PII column names.

**Nodes Involved:**  
- OpenAI  
- Get result  
- Merge

**Node Details:**

- **OpenAI**  
  - *Type:* OpenAI node (LangChain integration)  
  - *Role:* Sends a prompt with sample tabular data to GPT-4o-mini model to identify PII columns.  
  - *Configuration:*  
    - Model: `gpt-4o-mini`  
    - Messages:  
      - System message instructing to analyze tabular data and return only PII column names separated by commas, key named `content`.  
      - User message dynamically constructed with:  
        - Headers: extracted from JSON keys (`Object.keys($json)`)  
        - Example row: extracted from JSON values (`Object.values($json)`)  
    - JSON output enabled to parse response as JSON  
    - Executes once per run  
  - *Input:* JSON data from Extract from File node (first row)  
  - *Output:* JSON with key `content` containing comma-separated PII column names  
  - *Potential Failures:* API key issues, rate limits, malformed prompt or response, model unavailability.

- **Get result**  
  - *Type:* SplitOut node  
  - *Role:* Extracts the PII column names string from the OpenAI response.  
  - *Configuration:*  
    - Field to split out: `message.content.content` (the nested content field)  
    - Destination field: `data`  
  - *Input:* OpenAI node output  
  - *Output:* JSON with `data` containing PII columns string  
  - *Potential Failures:* Missing or malformed response field.

- **Merge**  
  - *Type:* Merge node  
  - *Role:* Combines multiple inputs into a single data stream for further processing.  
  - *Configuration:*  
    - Number of inputs: 3  
  - *Input:*  
    - Input 1: PII columns string from Get result  
    - Input 2: Original filename from Get filename  
    - Input 3: Extracted tabular data from Extract from File  
  - *Output:* Merged JSON containing PII columns, filename, and data rows  
  - *Potential Failures:* Input mismatch, data synchronization issues.

---

#### 2.4 Data Sanitization

**Overview:**  
Removes the identified PII columns from the dataset and reconstructs a sanitized CSV file with a new filename.

**Nodes Involved:**  
- Remove PII columns

**Node Details:**

- **Remove PII columns**  
  - *Type:* Code node (JavaScript)  
  - *Role:*  
    - Parses the PII column names string into an array.  
    - Removes these columns from each data row.  
    - Converts sanitized data back into CSV format.  
    - Generates a new filename appending `_PII_removed` before the extension.  
  - *Configuration:*  
    - Custom JavaScript code (detailed below)  
  - *Key expressions/logic:*  
    - Extract PII columns from merged input (`input[0].json.data`)  
    - Extract original filename (`input[1].json.originalFilename`)  
    - Process data rows starting from the third item (`input.slice(2)`)  
    - Remove PII columns from each row  
    - Rebuild CSV content with updated headers and sanitized rows  
    - Create new filename with suffix `_PII_removed`  
  - *Input:* Merged data from Merge node  
  - *Output:* JSON with keys:  
    - `fileName`: new sanitized filename  
    - `content`: CSV content as plain text  
  - *Potential Failures:*  
    - Missing PII columns data  
    - No data rows to process  
    - CSV formatting errors  
    - Filename parsing errors

---

#### 2.5 Output Upload

**Overview:**  
Uploads the sanitized CSV file back to a specified Google Drive folder, preserving the original file and preventing workflow loops.

**Nodes Involved:**  
- Upload to Drive

**Node Details:**

- **Upload to Drive**  
  - *Type:* Google Drive node  
  - *Role:* Creates a new text file in Google Drive with sanitized CSV content.  
  - *Configuration:*  
    - Operation: `createFromText`  
    - File name: dynamically set from `fileName` field in input JSON  
    - Content: CSV content from `content` field in input JSON  
    - Drive ID: `My Drive` (default Google Drive)  
    - Folder ID: `"1F30Qu3csrmMhtcu_prMipeiGm-64VEdd"` (different folder from trigger to avoid loops)  
  - *Input:* Output from Remove PII columns node  
  - *Output:* Metadata of uploaded file  
  - *Potential Failures:* Permission errors, quota limits, network issues.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                          | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                                  |
|---------------------|--------------------------------|----------------------------------------|-----------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger | Google Drive Trigger            | Detect new files in monitored folder   | None                        | Get filename, Google Drive |                                                                                                              |
| Get filename        | SplitOut                       | Extract original filename               | Google Drive Trigger         | Merge                   |                                                                                                              |
| Google Drive        | Google Drive                   | Download file content                   | Google Drive Trigger         | Extract from File        |                                                                                                              |
| Extract from File   | ExtractFromFile                | Parse CSV content into JSON             | Google Drive                 | OpenAI, Merge            |                                                                                                              |
| OpenAI              | OpenAI (LangChain)             | Identify PII columns via AI             | Extract from File            | Get result               |                                                                                                              |
| Get result          | SplitOut                      | Extract PII columns string from AI output | OpenAI                      | Merge                   |                                                                                                              |
| Merge               | Merge                         | Combine PII columns, filename, and data | Get result, Get filename, Extract from File | Remove PII columns |                                                                                                              |
| Remove PII columns  | Code                          | Remove PII columns and rebuild CSV      | Merge                       | Upload to Drive          |                                                                                                              |
| Upload to Drive     | Google Drive                  | Upload sanitized CSV to Drive           | Remove PII columns           | None                    |                                                                                                              |
| Sticky Note         | Sticky Note                   | Workflow description and instructions   | None                        | None                    | ## Remove PII from CSV Files\nThis workflow monitors a Google Drive folder for new CSV files, identifies and removes PII columns using OpenAI, and uploads the sanitized file back to the drive. It requires Google Drive and OpenAI integrations with API access enabled. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**  
   - Set event to `fileCreated`.  
   - Configure to watch a specific folder by folder ID (e.g., `"1-hRMnBRYgY6iVJ_youKMyPz83k9GAVYu"`).  
   - Set polling interval to every minute.  
   - Connect Google Drive OAuth2 credentials.

2. **Create SplitOut node named "Get filename":**  
   - Set field to split out: `name` (filename from trigger).  
   - Set destination field name: `originalFilename`.  
   - Enable execute once per trigger.  
   - Connect input from Google Drive Trigger.

3. **Create Google Drive node to download file:**  
   - Operation: `download`.  
   - File ID: set dynamically to `{{$json.id}}` from trigger.  
   - Binary property name: `data`.  
   - Connect input from Google Drive Trigger.  
   - Use same Google Drive OAuth2 credentials.

4. **Create ExtractFromFile node:**  
   - Use default options (assumes CSV).  
   - Connect input from Google Drive node (downloaded file).  
   - This node converts binary CSV to JSON rows.

5. **Create OpenAI node:**  
   - Use OpenAI API credentials.  
   - Model: `gpt-4o-mini`.  
   - Set messages:  
     - System message instructing to identify PII columns and return only column names separated by commas, key named `content`.  
     - User message dynamically constructed with:  
       - Headers: `{{Object.keys($json)}}`  
       - Example row: `{{Object.values($json)}}`  
   - Enable JSON output.  
   - Set to execute once.  
   - Connect input from ExtractFromFile node.

6. **Create SplitOut node named "Get result":**  
   - Field to split out: `message.content.content` (extract PII columns string).  
   - Destination field: `data`.  
   - Connect input from OpenAI node.

7. **Create Merge node:**  
   - Set number of inputs to 3.  
   - Connect inputs:  
     - Input 1: from Get result (PII columns string).  
     - Input 2: from Get filename (original filename).  
     - Input 3: from ExtractFromFile (tabular data).  
   - This merges all required data for sanitization.

8. **Create Code node named "Remove PII columns":**  
   - Paste the provided JavaScript code that:  
     - Extracts PII columns from input.  
     - Removes those columns from data rows.  
     - Converts sanitized data back to CSV.  
     - Generates new filename with `_PII_removed` suffix.  
   - Connect input from Merge node.

9. **Create Google Drive node named "Upload to Drive":**  
   - Operation: `createFromText`.  
   - Name: set dynamically from `{{$json.fileName}}`.  
   - Content: set dynamically from `{{$json.content}}`.  
   - Drive ID: `My Drive`.  
   - Folder ID: set to a different folder than the trigger folder (e.g., `"1F30Qu3csrmMhtcu_prMipeiGm-64VEdd"`) to avoid loops.  
   - Connect input from Remove PII columns node.  
   - Use Google Drive OAuth2 credentials.

10. **(Optional) Add Sticky Note:**  
    - Add a sticky note describing the workflow purpose and prerequisites for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow requires Google Drive and OpenAI API credentials with appropriate permissions and API keys. | Setup instructions in the workflow description.                                                        |
| To avoid infinite loops, ensure the upload folder differs from the monitored folder in Google Drive.      | Workflow configuration best practice.                                                                  |
| Customize the OpenAI prompt to align with your organization's PII definitions and compliance requirements. | Modify the system and user messages in the OpenAI node accordingly.                                    |
| For more information on n8n Google Drive integration, visit: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googledrive/ | Official n8n documentation.                                                                             |
| For OpenAI API usage and prompt design, see: https://platform.openai.com/docs/                             | OpenAI official documentation.                                                                          |

---

This document fully describes the workflow’s structure, logic, and configuration, enabling both human users and automation agents to understand, reproduce, and modify the workflow effectively.