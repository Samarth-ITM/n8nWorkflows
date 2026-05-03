Extract and Save Invoice Data from Google Drive to Sheets with Dumpling AI

https://n8nworkflows.xyz/workflows/extract-and-save-invoice-data-from-google-drive-to-sheets-with-dumpling-ai-4059


# Extract and Save Invoice Data from Google Drive to Sheets with Dumpling AI

### 1. Workflow Overview

This workflow automates the extraction and recording of invoice data from newly uploaded invoice files in a Google Drive folder into a structured Google Sheet. It is designed for operations teams, accountants, e-commerce businesses, or finance managers who regularly handle digital invoices and seek to reduce manual data entry errors and save processing time.

The workflow is logically divided into the following blocks:

- **1.1 Google Drive Monitoring & File Download**: Watches a specific Google Drive folder for new invoice files and downloads them.
- **1.2 File Conversion & AI Processing**: Converts the downloaded invoice PDF to Base64 format and sends it to Dumpling AI’s extraction API for structured data extraction.
- **1.3 Response Parsing & Data Splitting**: Parses the JSON response from Dumpling AI and splits the invoice line items into individual entries.
- **1.4 Data Saving to Google Sheets**: Appends each invoice line item along with invoice header metadata into a preformatted Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Google Drive Monitoring & File Download

**Overview:**  
This block triggers the workflow when a new file is created in a specified Google Drive folder and downloads that file for further processing.

**Nodes Involved:**  
- Google Drive Trigger – Watch Folder for New Files  
- Download Invoice File

**Node Details:**

- **Google Drive Trigger – Watch Folder for New Files**  
  - *Type:* Trigger node  
  - *Role:* Watches a specific folder in Google Drive for newly created files every minute.  
  - *Configuration:*  
    - Event: `fileCreated`  
    - Folder to watch: Specified by folder ID (configured with Google Drive folder "invoice-n8n")  
    - Polling interval: Every minute  
  - *Input:* None (trigger node)  
  - *Output:* JSON object representing the newly created file metadata (including file ID)  
  - *Credentials:* Google Drive OAuth2  
  - *Potential Failures:* Authentication errors (expired OAuth token), network issues, permission errors on the folder  
  - *Version:* 1

- **Download Invoice File**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the file from Google Drive using the file ID from the trigger node.  
  - *Configuration:*  
    - Operation: `download`  
    - File ID: dynamically taken from the trigger node’s output (`{{$json.id}}`)  
  - *Input:* File metadata from trigger node  
  - *Output:* Binary file data of the invoice  
  - *Credentials:* Google Drive OAuth2  
  - *Potential Failures:* File not found, permission denied, download timeout  
  - *Version:* 3

---

#### 2.2 File Conversion & AI Processing

**Overview:**  
Converts the downloaded invoice file into Base64 encoded string, then sends it via HTTP POST request to Dumpling AI’s document extraction API with a detailed prompt to extract structured invoice data.

**Nodes Involved:**  
- Convert invoice File to Base64  
- Send file to Dumpling AI for Data Extraction

**Node Details:**

- **Convert invoice File to Base64**  
  - *Type:* Extract From File node  
  - *Role:* Converts binary file data into a Base64 string property (`data`) usable in HTTP requests.  
  - *Configuration:*  
    - Operation: `binaryToProperty` (extract Base64 string from binary)  
  - *Input:* Binary invoice file from Google Drive download  
  - *Output:* JSON with Base64 string property named `data`  
  - *Potential Failures:* Binary data missing or corrupted, extraction errors  
  - *Version:* 1

- **Send file to Dumpling AI for Data Extraction**  
  - *Type:* HTTP Request node  
  - *Role:* Sends a JSON POST request to Dumpling AI’s `extract-document` API endpoint, submitting the Base64 invoice file and a prompt for data extraction.  
  - *Configuration:*  
    - URL: `https://app.dumplingai.com/api/v1/extract-document`  
    - Method: POST  
    - Body (JSON):  
      ```json
      {
        "inputMethod": "base64",
        "files": ["{{ $json.data }}"],
        "prompt": "Extract the order number, document date, PO number, sold to name and address, ship to name and address, list of items with model, quantity, unit price, and total price, and the final total amount including tax.",
        "jsonMode": "true"
      }
      ```  
    - Authentication: HTTP header with API key (generic header auth)  
  - *Input:* JSON with Base64 file string  
  - *Output:* JSON response containing extracted invoice data as a string property `results`  
  - *Credentials:* HTTP Header Auth (Dumpling AI API key)  
  - *Potential Failures:* Authentication errors (invalid API key), request timeouts, malformed JSON responses, API rate limits  
  - *Version:* 4.2

---

#### 2.3 Response Parsing & Data Splitting

**Overview:**  
Parses the JSON string response from Dumpling AI into usable JSON, then splits the invoice's line items array into individual entries for separate processing.

**Nodes Involved:**  
- Parse Dumpling AI JSON Response  
- Split line Items from Invoice

**Node Details:**

- **Parse Dumpling AI JSON Response**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Parses the JSON string from `results` field into a JSON object for downstream nodes.  
  - *Configuration:*  
    - Code snippet:  
      ```javascript
      const raw = $input.first().json.results;
      const parsed = JSON.parse(raw);
      return [{ json: parsed }];
      ```  
  - *Input:* Dumpling AI JSON response (string inside `results`)  
  - *Output:* Parsed JSON object containing invoice header and line items  
  - *Potential Failures:* Invalid JSON string causing parse exceptions  
  - *Version:* 2

- **Split line Items from Invoice**  
  - *Type:* Split Out node  
  - *Role:* Splits the `items` array from the parsed JSON into individual items, emitting one item per execution output.  
  - *Configuration:*  
    - Field to split out: `items`  
  - *Input:* Parsed invoice JSON with `items` array  
  - *Output:* Multiple items each representing one invoice line item  
  - *Potential Failures:* `items` field missing or not an array, empty arrays resulting in no output  
  - *Version:* 1

---

#### 2.4 Data Saving to Google Sheets

**Overview:**  
Appends each individual invoice line item along with invoice header metadata into a Google Sheet row by row, ensuring structured record-keeping.

**Nodes Involved:**  
- Save Data to Google Sheet

**Node Details:**

- **Save Data to Google Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends rows to the specified Google Sheet with mapped invoice data fields.  
  - *Configuration:*  
    - Operation: `append`  
    - Document ID: Spreadsheet ID where invoice data is saved  
    - Sheet Name: `Sheet1`  
    - Columns mapped explicitly with expressions referencing the current line item fields (`model`, `quantity`, etc.) and header metadata fields from the parsed AI response (accessed via node reference `Parse Dumpling AI JSON Response`):  
      - Order number, Document Date, Po_number, Sold to name, Sold to address, Ship to name, Ship to address, Model, Description, Quantity, Unity price, Total price  
    - Data type conversion disabled (fields saved as strings)  
  - *Input:* Individual invoice line item (from Split node), with references to header fields from parsed JSON  
  - *Output:* Confirmation of appended rows  
  - *Credentials:* Google Sheets OAuth2  
  - *Potential Failures:* Authentication errors, invalid sheet ID, API quota limits, schema mismatch  
  - *Version:* 4.5

---

### 3. Summary Table

| Node Name                              | Node Type                     | Functional Role                           | Input Node(s)                           | Output Node(s)                           | Sticky Note                                                                                                                                                                                                                                                                               |
|--------------------------------------|-------------------------------|-----------------------------------------|---------------------------------------|-----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger – Watch Folder for New Files | Google Drive Trigger          | Triggers workflow on new Drive files     | None                                  | Download Invoice File                    |                                                                                                                                                                                                                                                                                           |
| Download Invoice File                 | Google Drive                  | Downloads new invoice file                | Google Drive Trigger                   | Convert invoice File to Base64           |                                                                                                                                                                                                                                                                                           |
| Convert invoice File to Base64        | Extract From File             | Converts binary file to Base64 string    | Download Invoice File                  | Send file to Dumpling AI for Data Extraction |                                                                                                                                                                                                                                                                                           |
| Send file to Dumpling AI for Data Extraction | HTTP Request                 | Sends Base64 file and prompt to Dumpling AI | Convert invoice File to Base64         | Parse Dumpling AI JSON Response           |                                                                                                                                                                                                                                                                                           |
| Parse Dumpling AI JSON Response       | Code                         | Parses AI JSON string response into JSON | Send file to Dumpling AI for Data Extraction | Split line Items from Invoice             |                                                                                                                                                                                                                                                                                           |
| Split line Items from Invoice         | Split Out                    | Splits invoice line items into individual entries | Parse Dumpling AI JSON Response        | Save Data to Google Sheet                |                                                                                                                                                                                                                                                                                           |
| Save Data to Google Sheet             | Google Sheets                | Appends invoice data rows to Google Sheet | Split line Items from Invoice          | None                                    |                                                                                                                                                                                                                                                                                           |
| Sticky Note                          | Sticky Note                  | Workflow overview and node summary       | None                                  | None                                    | ### 📌 **Workflow Overview: Extract & Save Invoice Data from Google Drive with Dumpling AI**  This workflow monitors a Google Drive folder for new files (invoices), extracts structured invoice data using Dumpling AI, and appends it to a Google Sheet. See section 1 for detailed steps.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure:  
     - Event: `fileCreated`  
     - Trigger on: Specific folder  
     - Folder to watch: Enter your Google Drive folder ID where invoices are uploaded  
     - Poll interval: Every minute  
   - Credentials: Google Drive OAuth2 with appropriate permissions  

2. **Add Google Drive Node to Download File**  
   - Type: Google Drive  
   - Operation: `download`  
   - File ID: Use expression `{{$json.id}}` from trigger node output  
   - Credentials: Same Google Drive OAuth2 credentials  

3. **Add Extract From File Node to Convert File to Base64**  
   - Type: Extract From File  
   - Operation: `binaryToProperty`  
   - Input: Binary data from Google Drive download node  
   - Output property: `data` (default)  

4. **Add HTTP Request Node to Send to Dumpling AI**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/extract-document`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "inputMethod": "base64",
       "files": ["{{ $json.data }}"],
       "prompt": "Extract the order number, document date, PO number, sold to name and address, ship to name and address, list of items with model, quantity, unit price, and total price, and the final total amount including tax.",
       "jsonMode": "true"
     }
     ```  
   - Authentication: HTTP Header Auth (add your Dumpling AI API key in header)  
   - Credentials: Create HTTP Header Auth credential with your Dumpling AI API key  

5. **Add Code Node to Parse Dumpling AI Response**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const raw = $input.first().json.results;
     const parsed = JSON.parse(raw);
     return [{ json: parsed }];
     ```  

6. **Add Split Out Node to Split Line Items**  
   - Type: Split Out  
   - Field to split out: `items` (from the parsed JSON)  

7. **Add Google Sheets Node to Save Data**  
   - Type: Google Sheets  
   - Operation: `append`  
   - Document ID: Your target Google Sheet ID  
   - Sheet Name: `Sheet1` (or your preferred sheet)  
   - Columns Mapping (map fields from the current item and header metadata):  
     - Model: `={{ $json.model }}`  
     - Quantity: `={{ $json.quantity }}`  
     - Po_number: `={{ $('Parse Dumpling AI JSON Response').item.json.PO_number }}`  
     - Description: `={{ $json.description }}`  
     - Total price: `={{ $json.total_price }}`  
     - Unity price: `={{ $json.unit_price }}`  
     - Order number: `={{ $('Parse Dumpling AI JSON Response').item.json.order_number }}`  
     - Ship to name: `={{ $('Parse Dumpling AI JSON Response').item.json.ship_to_name }}`  
     - Sold to name: `={{ $('Parse Dumpling AI JSON Response').item.json.sold_to_name }}`  
     - Document Date: `={{ $('Parse Dumpling AI JSON Response').item.json.document_date }}`  
     - Ship to address: `={{ $('Parse Dumpling AI JSON Response').item.json.ship_to_address }}`  
     - Sold to address: `={{ $('Parse Dumpling AI JSON Response').item.json.sold_to_address }}`  
   - Credentials: Google Sheets OAuth2 with write access  

8. **Connect the nodes in the following order:**  
   - Google Drive Trigger → Download Invoice File → Convert invoice File to Base64 → Send file to Dumpling AI for Data Extraction → Parse Dumpling AI JSON Response → Split line Items from Invoice → Save Data to Google Sheet  

9. **Test the workflow:** Upload an invoice PDF file to the Google Drive folder and verify the data appears correctly in the Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                           |
|---------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Dumpling AI API and signup: Generate your API key at https://www.dumplingai.com/                                                | Dumpling AI official site                                  |
| Google Drive folder ID can be found in the URL when you open the folder in Google Drive.                                        | Setup instruction                                         |
| Google Sheets must have predefined headers matching the columns mapped in the workflow to ensure data appends correctly.       | Setup instruction                                         |
| Customize the Dumpling AI prompt to extract additional invoice fields as needed.                                                | Workflow customization                                    |
| You can replace Google Sheets node with Airtable or database nodes for alternative data storage options.                       | Customization tip                                         |
| For email-based invoice processing, replace Google Drive Trigger with an Email Trigger node and adjust file input accordingly. | Alternative trigger option                                |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation platform. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and publicly accessible.