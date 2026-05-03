Move data between JSON and spreadsheets

https://n8nworkflows.xyz/workflows/move-data-between-json-and-spreadsheets-1739


# Move data between JSON and spreadsheets

### 1. Workflow Overview

This workflow demonstrates how to move and convert data between JSON and spreadsheet formats using n8n nodes. It targets use cases involving data extraction from an API, transformation and formatting, and loading into Google Sheets or local files, as well as sending JSON data as email attachments.

The workflow is logically divided into four main blocks:

- **1.1 JSON to Google Sheets:** Fetches user data from a public API, extracts relevant fields, and appends them to a Google Sheets document.
- **1.2 JSON to CSV File:** Converts the extracted JSON data to CSV format and stores it as a local spreadsheet file.
- **1.3 CSV to JSON File:** Converts the CSV data back into JSON format and writes it as a local JSON file.
- **1.4 JSON File to Google Sheets via Email:** Reads the JSON file, sends it via Gmail as an attachment, and then appends the data to another Google Sheets document from the JSON file received.

Each block contains specific nodes that handle HTTP requests, data transformation, file conversion, file writing, email sending, and spreadsheet operations.

---

### 2. Block-by-Block Analysis

#### 2.1 JSON to Google Sheets

- **Overview:**  
This block fetches random user data from an external API, extracts the user’s first and last name and country, then appends this data as rows to a Google Sheets spreadsheet.

- **Nodes Involved:**  
  - HTTP Request  
  - Set  
  - Google Sheets  
  - Sticky Note ("JSON > Google Sheets")

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Fetches random user data from https://randomuser.me/api/.  
    - Configuration: Simple GET request without authentication or additional options.  
    - Expressions: None.  
    - Input: None (triggered externally or manually).  
    - Output: JSON data containing user details.  
    - Edge Cases: API downtime, network errors, unexpected response structure.  
    - Version: 2.

  - **Set**  
    - Type: Set  
    - Role: Extracts and formats the JSON response to create simplified data with `name` and `country` fields.  
    - Configuration: Uses expressions to concatenate first and last names and extract country from the JSON response (`{{$json["results"][0]["name"]["first"]}} {{$json["results"][0]["name"]["last"]}}` for name, `{{$json["results"][0]["location"]["country"]}}` for country). Keeps only these fields.  
    - Input: JSON from HTTP Request node.  
    - Output: JSON with only `name` and `country`.  
    - Edge Cases: Missing fields in API response causing expression failures.  
    - Version: 1.

  - **Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends the simplified data to a Google Sheets document in columns A to C.  
    - Configuration:  
      - Operation: Append  
      - Range: A:C  
      - Options: `usePathForKeyRow` set to true (keys in JSON used as headers).  
      - Authentication: OAuth2 with Google Sheets credentials.  
      - Sheet ID: "qwertz" (placeholder or example Sheet ID).  
    - Input: JSON output from the Set node.  
    - Output: Confirmation of append operation.  
    - Edge Cases: Authentication errors, quota limits, invalid sheet ID, range conflicts.  
    - Version: 1.

  - **Sticky Note ("JSON > Google Sheets")**  
    - Content: Labels this block as converting JSON data to Google Sheets.

---

#### 2.2 JSON to CSV File

- **Overview:**  
Converts the simplified JSON data into a CSV file format and prepares it for further processing or storage.

- **Nodes Involved:**  
  - Spreadsheet File  
  - Spreadsheet File1  
  - Sticky Note ("JSON > CSV")

- **Node Details:**

  - **Spreadsheet File**  
    - Type: Spreadsheet File  
    - Role: Converts JSON data into a CSV file.  
    - Configuration:  
      - Operation: toFile  
      - File Format: csv  
      - File Name: "users_spreadsheet" (logical name, actual extension is CSV).  
    - Input: JSON output from Set node (via Google Sheets node connection).  
    - Output: Binary CSV file data.  
    - Edge Cases: Data formatting errors, encoding issues.  
    - Version: 1.

  - **Spreadsheet File1**  
    - Type: Spreadsheet File  
    - Role: Further processes or passes the CSV binary data.  
    - Configuration: Default operation (implied to read or transform the CSV binary).  
    - Input: Binary CSV data from Spreadsheet File.  
    - Output: Binary data forwarded to next node.  
    - Edge Cases: Misinterpretation of CSV data, format mismatch.  
    - Version: 1.

  - **Sticky Note ("JSON > CSV")**  
    - Content: Labels this block as converting JSON data into CSV format.

---

#### 2.3 CSV to JSON File

- **Overview:**  
Transforms the CSV binary data back into JSON format and writes it to a local JSON file.

- **Nodes Involved:**  
  - Move Binary Data1  
  - Write Binary File  
  - Sticky Note ("CSV > JSON file")

- **Node Details:**

  - **Move Binary Data1**  
    - Type: Move Binary Data  
    - Role: Converts JSON data in binary form to be compatible with file writing.  
    - Configuration:  
      - Mode: jsonToBinary (convert JSON to binary format for file write).  
    - Input: Binary CSV from Spreadsheet File1.  
    - Output: Binary data prepared for file writing.  
    - Edge Cases: Conversion errors if input data is malformed.  
    - Version: 1.

  - **Write Binary File**  
    - Type: Write Binary File  
    - Role: Writes the binary data to a local file named "randomusers.json".  
    - Configuration:  
      - File Name: "randomusers.json"  
    - Input: Binary data from Move Binary Data1.  
    - Output: File path or confirmation of file creation.  
    - Edge Cases: Filesystem access errors, permission issues, disk space.  
    - Version: 1.

  - **Sticky Note ("CSV > JSON file")**  
    - Content: Labels this block as converting CSV data into a JSON file.

---

#### 2.4 JSON File to Google Sheets via Email

- **Overview:**  
Reads the saved JSON file, sends it as an email attachment via Gmail, extracts the attachment again, and appends its content to another Google Sheets spreadsheet.

- **Nodes Involved:**  
  - Gmail1  
  - Move Binary Data2  
  - Google Sheets2  
  - Sticky Note ("JSON file > Google Sheets")

- **Node Details:**

  - **Gmail1**  
    - Type: Gmail  
    - Role: Sends an email with the JSON file attached.  
    - Configuration:  
      - Subject: "JSON file with users"  
      - Message: "Hello, attached is a JSON file with random user information."  
      - Attachments: Uses binary property "data" (from Write Binary File output) as attachment.  
      - Authentication: OAuth2 for Gmail.  
    - Input: Binary JSON file data from Write Binary File node.  
    - Output: Email send confirmation, including attachment metadata.  
    - Edge Cases: Authentication errors, attachment size limits, network issues.  
    - Version: 1.

  - **Move Binary Data2**  
    - Type: Move Binary Data  
    - Role: Extracts the attachment binary data from the email node output to prepare for Google Sheets.  
    - Configuration:  
      - Source Key: "attachment_0" (first attachment in email).  
    - Input: Email node output with attachment metadata.  
    - Output: Binary JSON data for Google Sheets2.  
    - Edge Cases: Missing attachment, incorrect source key.  
    - Version: 1.

  - **Google Sheets2**  
    - Type: Google Sheets  
    - Role: Appends JSON data from the attachment to the Google Sheets spreadsheet.  
    - Configuration:  
      - Operation: Append  
      - Range: A:C  
      - Options: `usePathForKeyRow` true.  
      - Authentication: Google Sheets OAuth2.  
      - Sheet ID: "qwertz" (same or different sheet as first Google Sheets node).  
    - Input: Binary JSON data from Move Binary Data2.  
    - Output: Append confirmation.  
    - Edge Cases: Same as first Google Sheets node, plus data format compatibility.  
    - Version: 1.

  - **Sticky Note ("JSON file > Google Sheets")**  
    - Content: Labels this block as importing JSON file data back into Google Sheets.

---

### 3. Summary Table

| Node Name          | Node Type            | Functional Role                             | Input Node(s)     | Output Node(s)       | Sticky Note                 |
|--------------------|----------------------|--------------------------------------------|-------------------|----------------------|-----------------------------|
| HTTP Request       | HTTP Request         | Fetch user JSON data from API               | None              | Set                  |                             |
| Set                | Set                  | Extract name and country from JSON          | HTTP Request      | Google Sheets, Spreadsheet File |                             |
| Google Sheets      | Google Sheets        | Append extracted data to Google Sheets      | Set               | Spreadsheet File     | JSON > Google Sheets         |
| Spreadsheet File   | Spreadsheet File     | Convert JSON data to CSV file                | Google Sheets     | Spreadsheet File1    | JSON > CSV                   |
| Spreadsheet File1  | Spreadsheet File     | Process CSV binary data                      | Spreadsheet File  | Move Binary Data1    | JSON > CSV                   |
| Move Binary Data1  | Move Binary Data     | Convert JSON to binary for file write       | Spreadsheet File1 | Write Binary File    | CSV > JSON file              |
| Write Binary File  | Write Binary File    | Write JSON binary data to local file         | Move Binary Data1  | Gmail1               | CSV > JSON file              |
| Gmail1             | Gmail                | Send JSON file as email attachment           | Write Binary File  | Move Binary Data2    | JSON file > Google Sheets    |
| Move Binary Data2  | Move Binary Data     | Extract attachment binary data from email    | Gmail1            | Google Sheets2       | JSON file > Google Sheets    |
| Google Sheets2     | Google Sheets        | Append JSON file data to Google Sheets       | Move Binary Data2  | None                 | JSON file > Google Sheets    |
| Note               | Sticky Note          | Label for JSON to Google Sheets block        | None              | None                 | JSON > Google Sheets         |
| Note1              | Sticky Note          | Label for JSON to CSV block                   | None              | None                 | JSON > CSV                   |
| Note2              | Sticky Note          | Label for CSV to JSON file block              | None              | None                 | CSV > JSON file              |
| Note3              | Sticky Note          | Label for JSON file to Google Sheets block   | None              | None                 | JSON file > Google Sheets    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create HTTP Request node:**  
   - Type: HTTP Request (version 2)  
   - URL: `https://randomuser.me/api/`  
   - Method: GET (default)  
   - No authentication or special options.

2. **Create Set node:**  
   - Type: Set (version 1)  
   - Add two string fields:  
     - `name` with value expression: `{{$json["results"][0]["name"]["first"]}} {{$json["results"][0]["name"]["last"]}}`  
     - `country` with value expression: `{{$json["results"][0]["location"]["country"]}}`  
   - Enable "Keep Only Set" to remove other fields.  
   - Connect HTTP Request node output to Set node input.

3. **Create Google Sheets node (first):**  
   - Type: Google Sheets (version 1)  
   - Operation: Append  
   - Range: A:C  
   - Options: Enable "Use Path For Key Row"  
   - Sheet ID: `qwertz` (replace with your actual sheet ID)  
   - Authentication: OAuth2 with Google Sheets credentials configured in n8n.  
   - Connect Set node output to Google Sheets input.

4. **Create Spreadsheet File node:**  
   - Type: Spreadsheet File (version 1)  
   - Operation: toFile  
   - File Format: CSV  
   - File Name: `users_spreadsheet`  
   - Connect Google Sheets node output to Spreadsheet File node input.

5. **Create second Spreadsheet File node (Spreadsheet File1):**  
   - Type: Spreadsheet File (version 1)  
   - Default operation (reads or processes CSV binary)  
   - Connect Spreadsheet File node output to Spreadsheet File1 input.

6. **Create Move Binary Data1 node:**  
   - Type: Move Binary Data (version 1)  
   - Mode: jsonToBinary  
   - Connect Spreadsheet File1 output to Move Binary Data1 input.

7. **Create Write Binary File node:**  
   - Type: Write Binary File (version 1)  
   - File Name: `randomusers.json`  
   - Connect Move Binary Data1 output to Write Binary File input.

8. **Create Gmail node:**  
   - Type: Gmail (version 1)  
   - Subject: "JSON file with users"  
   - Message: "Hello, attached is a JSON file with random user information."  
   - Attachments: Attach binary property `data` (comes from Write Binary File output)  
   - Authentication: OAuth2 using Gmail credentials configured in n8n.  
   - Connect Write Binary File output to Gmail node input.

9. **Create Move Binary Data2 node:**  
   - Type: Move Binary Data (version 1)  
   - Source Key: `attachment_0` (to extract first attachment from email output)  
   - Connect Gmail output to Move Binary Data2 input.

10. **Create Google Sheets node (second):**  
    - Type: Google Sheets (version 1)  
    - Operation: Append  
    - Range: A:C  
    - Options: Enable "Use Path For Key Row"  
    - Sheet ID: `qwertz` (use same or different sheet ID as required)  
    - Authentication: OAuth2 with Google Sheets credentials.  
    - Connect Move Binary Data2 output to Google Sheets2 input.

11. **Add Sticky Notes:**  
    - Place sticky notes near each block with the following content:  
      - "JSON > Google Sheets" near HTTP Request, Set, Google Sheets nodes.  
      - "JSON > CSV" near Spreadsheet File and Spreadsheet File1 nodes.  
      - "CSV > JSON file" near Move Binary Data1 and Write Binary File nodes.  
      - "JSON file > Google Sheets" near Gmail, Move Binary Data2, and Google Sheets2 nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The placeholder `sheetId` "qwertz" must be replaced with an actual Google Sheets ID for your sheets.| Google Sheets API documentation for finding sheet IDs: https://developers.google.com/sheets/api     |
| OAuth2 credentials for Google Sheets and Gmail must be pre-configured in n8n credentials manager.   | n8n Docs on OAuth2 credential setup: https://docs.n8n.io/credentials/google/                        |
| The randomuser.me API is a public API for generating fake users, ideal for testing and demos.       | https://randomuser.me/                                                                             |
| For large data sets, watch out for Google Sheets API rate limits and file size limits on Gmail.     | Google API quotas: https://developers.google.com/sheets/api/limits                                |
| Expressions extracting nested JSON fields must be updated if the API response structure changes.    | n8n expression docs: https://docs.n8n.io/nodes/expressions/                                        |

---

This comprehensive reference document provides a clear understanding of the workflow’s structure, node configurations, data flow, and precautions needed for successful operation and possible extension.