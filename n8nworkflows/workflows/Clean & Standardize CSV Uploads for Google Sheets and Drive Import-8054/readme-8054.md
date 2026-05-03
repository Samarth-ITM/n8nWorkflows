Clean & Standardize CSV Uploads for Google Sheets and Drive Import

https://n8nworkflows.xyz/workflows/clean---standardize-csv-uploads-for-google-sheets-and-drive-import-8054


# Clean & Standardize CSV Uploads for Google Sheets and Drive Import

### 1. Workflow Overview

This workflow is designed to handle CSV file uploads, clean and standardize their contents, and optionally save the processed data to Google Drive and/or Google Sheets. It targets use cases where raw CSV data is received via HTTP POST requests (e.g., from forms or API clients), and needs to be prepared for further use or archival in Google Workspace services.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives CSV file uploads through a webhook and extracts the CSV content.
- **1.2 CSV Parsing:** Parses the raw CSV content into structured rows with normalized headers.
- **1.3 Data Cleaning and Standardization:** Applies cleaning rules to remove duplicates, empty rows, and standardize formats (emails, phones, names, general text). It also scores data quality.
- **1.4 CSV Regeneration:** Generates a cleaned CSV file content based on the cleaned data.
- **1.5 Output to Google Workspace:** Saves the cleaned CSV file to Google Drive and imports the cleaned data into a Google Sheet (optional).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block accepts CSV file uploads via an HTTP POST webhook, supports multiple encoding methods (binary upload, base64, or direct CSV content), validates the input, and prepares metadata.

- **Nodes Involved:**  
  - CSV Upload Webhook  
  - Extract CSV Content

- **Node Details:**  

  - **CSV Upload Webhook**  
    - Type: Webhook  
    - Role: Entry point node that listens for POST requests at path `/csv-upload`.  
    - Configuration: Accepts HTTP POST with form-data or JSON payloads. No special response suppression.  
    - Input: HTTP request with CSV file or content  
    - Output: Request data (binary or JSON) forwarded to next node  
    - Edge cases: Missing or malformed request bodies, unsupported encodings, file size constraints (recommend max 10MB).  

  - **Extract CSV Content**  
    - Type: Code (JavaScript)  
    - Role: Extracts CSV content from the webhook payload in various formats and validates it.  
    - Configuration:  
      - Checks presence of binary data under key `data`, or JSON keys `csv_content` or `file_base64`.  
      - Converts base64 to UTF-8 string.  
      - Throws errors if no CSV content or if content is too short (<10 characters).  
      - Calculates initial row count by splitting on newline.  
    - Input: Webhook output  
    - Output: JSON with properties: `filename`, `original_content`, `file_size_bytes`, `initial_row_count`, timestamps.  
    - Edge cases: Missing CSV content, invalid encoding, empty or near-empty files.

#### 2.2 CSV Parsing

- **Overview:**  
  Parses the raw CSV text into structured JSON rows with normalized headers, removing empty lines and ensuring column consistency.

- **Nodes Involved:**  
  - Parse CSV Data

- **Node Details:**  

  - **Parse CSV Data**  
    - Type: Code  
    - Role: Splits CSV content into lines, normalizes headers to lowercase snake_case, and builds JSON objects per row.  
    - Configuration:  
      - Trims lines and removes empty lines.  
      - Parses first line as headers; strips quotes and whitespace.  
      - Skips rows with inconsistent column count or entirely empty values.  
    - Input: JSON from Extract CSV Content  
    - Output: JSON enriched with `headers`, `raw_rows` (array of objects), and parsed row count.  
    - Edge cases: CSVs with inconsistent row lengths, completely empty files, unusual characters in headers.

#### 2.3 Data Cleaning and Standardization

- **Overview:**  
  Applies cleaning transformations on each row to standardize emails, phone numbers, names, and general text fields, calculates a data quality score, and filters out low-quality rows.

- **Nodes Involved:**  
  - Clean & Standardize Data

- **Node Details:**  

  - **Clean & Standardize Data**  
    - Type: Code  
    - Role: Cleans each field according to its type inferred by key names (`email`, `phone`, `name`), and computes a quality score based on filled fields.  
    - Configuration:  
      - Email: Lowercase, trimmed, validated with regex; invalid emails replaced with empty string.  
      - Phone: Digits only, non-digit characters removed.  
      - Name: Capitalizes each word, trims whitespace.  
      - Others: Trimmed and condensed spaces.  
      - Adds `_data_quality_score` as percentage of filled fields.  
      - Filters rows with score >=30%.  
    - Input: Parsed CSV JSON data  
    - Output: JSON with `cleaned_rows`, summary statistics (`original_count`, `after_cleaning`, `average_quality_score`), and completion timestamp.  
    - Edge cases: Rows missing key columns, malformed emails or phones, all-empty rows.

#### 2.4 CSV Regeneration

- **Overview:**  
  Rebuilds a CSV string from the cleaned data rows and headers, prepares binary content for file saving, and generates a timestamped filename.

- **Nodes Involved:**  
  - Generate Clean CSV

- **Node Details:**  

  - **Generate Clean CSV**  
    - Type: Code  
    - Role: Constructs CSV text from cleaned JSON rows using original headers; prepares binary data for output.  
    - Configuration:  
      - Joins headers with commas, then appends each cleaned row.  
      - Filename format: `cleaned_<timestamp>_<original_filename>`.  
      - Sets binary data properties for file name and MIME type.  
    - Input: Cleaned JSON data  
    - Output: JSON plus binary containing cleaned CSV file content  
    - Edge cases: Empty cleaned rows, missing headers (unlikely), Unicode characters in data.

#### 2.5 Output to Google Workspace

- **Overview:**  
  Optionally saves the cleaned CSV file to Google Drive and updates a Google Sheet with the cleaned data by clearing existing content and appending new rows.

- **Nodes Involved:**  
  - Save to Google Drive  
  - Clear Existing Sheet  
  - Prepare for Sheets  
  - Import to Google Sheets

- **Node Details:**  

  - **Save to Google Drive**  
    - Type: Google Drive node  
    - Role: Uploads the cleaned CSV file to a specified folder in Google Drive.  
    - Configuration:  
      - Uses OAuth2 credentials for Google Drive.  
      - Target folder is set to "My Drive" root folder by default; can be customized.  
      - File name dynamically set from cleaned filename.  
    - Input: Binary CSV content from Generate Clean CSV  
    - Output: Google Drive file metadata  
    - Edge cases: Authentication failure, insufficient permissions, file size limits.

  - **Clear Existing Sheet**  
    - Type: Google Sheets node  
    - Role: Clears all data in the specified sheet ("Sheet1") in the target Google Sheet document.  
    - Configuration:  
      - Document ID must be replaced with actual Google Sheet ID.  
      - Uses OAuth2 credentials for Google Sheets.  
    - Input: Triggered after saving file to Drive  
    - Output: Confirmation of clearing  
    - Edge cases: Missing or incorrect Sheet ID, permission issues.

  - **Prepare for Sheets**  
    - Type: Code  
    - Role: Converts cleaned rows JSON array to individual items for Google Sheets append operation.  
    - Configuration:  
      - Maps each cleaned row object to an item with `json` property.  
    - Input: Output of Clear Existing Sheet node (passes data from previous nodes)  
    - Output: Array of JSON items for import  
    - Edge cases: Empty cleaned data.

  - **Import to Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends the cleaned data rows to the specified Google Sheet document and sheet.  
    - Configuration:  
      - Document ID placeholder must be replaced.  
      - Operation: append  
      - Sheet name: "Sheet1"  
    - Input: Prepared JSON items from Prepare for Sheets  
    - Output: Confirmation of append operation  
    - Edge cases: API limits, permissions, invalid Sheet ID.

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                 | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                          |
|-----------------------|----------------------|--------------------------------|-----------------------|------------------------|----------------------------------------------------------------------------------------------------|
| Setup Instructions    | Sticky Note          | Setup guidance and instructions | —                     | —                      | 🧹 **SETUP REQUIRED:** 1. Upload Method: webhook, form-data/base64, max 10MB; 2. Google Sheets OAuth; 3. Google Drive OAuth; 4. Cleaning rules overview. |
| CSV Upload Webhook    | Webhook              | Receives CSV upload via HTTP   | —                     | Extract CSV Content     |                                                                                                    |
| Extract CSV Content   | Code                 | Extracts and validates CSV text | CSV Upload Webhook     | Parse CSV Data          |                                                                                                    |
| Parse CSV Data        | Code                 | Parses CSV to JSON rows         | Extract CSV Content    | Clean & Standardize Data |                                                                                                    |
| Clean & Standardize Data | Code               | Cleans and standardizes data    | Parse CSV Data         | Generate Clean CSV      |                                                                                                    |
| Generate Clean CSV    | Code                 | Builds cleaned CSV binary       | Clean & Standardize Data | Save to Google Drive, Clear Existing Sheet |                                                                                                    |
| Save to Google Drive  | Google Drive         | Saves cleaned CSV to Drive      | Generate Clean CSV     | Clear Existing Sheet    |                                                                                                    |
| Clear Existing Sheet  | Google Sheets        | Clears existing sheet content   | Save to Google Drive   | Prepare for Sheets      |                                                                                                    |
| Prepare for Sheets    | Code                 | Prepares data format for Sheets | Clear Existing Sheet   | Import to Google Sheets |                                                                                                    |
| Import to Google Sheets | Google Sheets       | Appends cleaned data to Sheet   | Prepare for Sheets     | —                      |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note node named "Setup Instructions"**  
   - Paste the content describing setup steps and cleaning rules.  
   - Position it visibly near the workflow start as a guide.

2. **Create a Webhook node named "CSV Upload Webhook"**  
   - Set HTTP Method: POST  
   - Path: `csv-upload`  
   - Disable "No Response Body" (default false)  
   - This node is the workflow entry point.

3. **Create a Code node named "Extract CSV Content"**  
   - Connect input from "CSV Upload Webhook" node.  
   - Paste the provided JavaScript code to extract CSV content from binary, JSON, or base64 fields.  
   - This node outputs metadata and raw CSV content.

4. **Create a Code node named "Parse CSV Data"**  
   - Connect input from "Extract CSV Content".  
   - Paste the JavaScript code that splits CSV lines, normalizes headers to lowercase snake_case, filters empty rows, and builds JSON objects for each row.  
   - Output includes headers array and raw rows.

5. **Create a Code node named "Clean & Standardize Data"**  
   - Connect input from "Parse CSV Data".  
   - Paste the JavaScript code that cleans each field type (email, phone, name, text), computes a quality score, filters low-quality rows, and generates cleaning summary.  
   - Outputs cleaned rows and summary.

6. **Create a Code node named "Generate Clean CSV"**  
   - Connect input from "Clean & Standardize Data".  
   - Paste the JavaScript code to rebuild CSV text from cleaned rows and headers, generate a cleaned filename, and prepare binary data with proper MIME type.

7. **Create a Google Drive node named "Save to Google Drive"**  
   - Connect input from "Generate Clean CSV".  
   - Set Operation: Upload File  
   - Name: use expression `={{ $json.cleaned_filename }}`  
   - Folder ID: set your desired folder or use "root"  
   - Credentials: Configure Google Drive OAuth2 credentials with appropriate permissions.

8. **Create a Google Sheets node named "Clear Existing Sheet"**  
   - Connect input from "Save to Google Drive".  
   - Operation: Clear  
   - Sheet Name: "Sheet1"  
   - Document ID: replace `"YOUR_GOOGLE_SHEET_ID"` with your actual Google Sheet ID  
   - Credentials: Configure Google Sheets OAuth2 credentials.

9. **Create a Code node named "Prepare for Sheets"**  
   - Connect input from "Clear Existing Sheet".  
   - Paste the JavaScript code that maps cleaned rows array to individual JSON items for Sheets append.

10. **Create a Google Sheets node named "Import to Google Sheets"**  
    - Connect input from "Prepare for Sheets".  
    - Operation: Append  
    - Sheet Name: "Sheet1"  
    - Document ID: same as in Clear Existing Sheet node  
    - Credentials: Google Sheets OAuth2 credentials.

11. **Verify all connections:**  
    - CSV Upload Webhook → Extract CSV Content → Parse CSV Data → Clean & Standardize Data → Generate Clean CSV → Save to Google Drive → Clear Existing Sheet → Prepare for Sheets → Import to Google Sheets.

12. **Set credentials:**  
    - Google Drive OAuth2 (for Save to Google Drive node)  
    - Google Sheets OAuth2 (for Clear Existing Sheet and Import to Google Sheets)  
    - Ensure OAuth scopes allow file upload and sheet editing.

13. **Test the workflow:**  
    - Send a POST request to `your-n8n-instance-url/webhook/csv-upload` with a CSV file in form-data or base64 JSON.  
    - Confirm cleaned file appears in Google Drive and data updates in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow supports CSV file uploads up to approximately 10MB; larger files may require adjustments.       | Setup Instructions sticky note                                                                    |
| When integrating with Google Sheets, replace `"YOUR_GOOGLE_SHEET_ID"` with the actual Sheet ID in nodes.      | Setup Instructions sticky note                                                                    |
| For Google Drive uploads, configure destination folder ID as needed; default is root folder.                   | Setup Instructions sticky note                                                                    |
| Cleaning rules include email validation via regex, phone number digit extraction, and capitalization of names. | Setup Instructions sticky note                                                                    |
| Workflow designed to be extensible for additional cleaning rules or other output destinations.                 | General design consideration                                                                      |

---

**Disclaimer:** The text provided is exclusively derived from an n8n workflow automation. It complies strictly with all content policies and does not contain illegal, offensive, or protected material. All handled data is legal and public.