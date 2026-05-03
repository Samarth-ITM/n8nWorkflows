Import CSV Contacts to Notion Database from Google Drive

https://n8nworkflows.xyz/workflows/import-csv-contacts-to-notion-database-from-google-drive-5612


# Import CSV Contacts to Notion Database from Google Drive

### 1. Workflow Overview

This workflow automates the import of contact data from a CSV file stored on Google Drive into a Notion database. It is tailored for users who maintain contact lists externally (e.g., contact managers exporting CSV files) and wish to synchronize or migrate these contacts into Notion for centralized management.

The workflow is logically divided into these functional blocks:

- **1.1 Input Trigger and File Retrieval:** Waits for manual execution, then downloads the specified CSV file from Google Drive.
- **1.2 Data Extraction and Transformation:** Extracts CSV content, parses and normalizes contact fields to prepare data.
- **1.3 Notion Database Page Creation:** Creates or updates pages in a Notion database with the transformed contact information.
- **1.4 Documentation Block:** A sticky note providing usage instructions and configuration guidelines.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and File Retrieval

- **Overview:**  
  This block initiates the workflow manually and downloads a specific CSV file from Google Drive that contains the contact data to be imported.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Download file

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point that allows the user to start the workflow manually within n8n.  
    - Configuration: No parameters; triggers on manual execution only.  
    - Connections: Output → "Download file" node  
    - Edge cases: No input data; failure unlikely unless n8n system is down.

  - **Download file**  
    - Type: Google Drive node (operation: download)  
    - Role: Downloads the CSV file from Google Drive using a specified file ID.  
    - Configuration:  
      - File ID set to "1qYi9QBfBFu9jWXLoR7fdTn7FOwbceiL5" (hardcoded file identifier)  
      - Binary property name: "data" (the file content is stored as binary data for downstream use)  
    - Credentials: Google Drive OAuth2 (requires valid OAuth2 connection)  
    - Connections: Output → "Extract from File" node  
    - Edge cases:  
      - Authentication failure if OAuth token expired or invalid  
      - File not found or access denied if file ID or permissions are incorrect  
      - Network timeouts or Google Drive API rate limits  

#### 1.2 Data Extraction and Transformation

- **Overview:**  
  Extracts the CSV content from the downloaded file, parses it into JSON objects, and transforms each contact's data to match the expected structure for Notion.

- **Nodes Involved:**  
  - Extract from File  
  - Code

- **Node Details:**

  - **Extract from File**  
    - Type: Extract From File  
    - Role: Converts the binary CSV file data into structured JSON data representing rows and columns.  
    - Configuration: Default extraction options (no custom delimiters or encoding specified)  
    - Input: Binary data named "data" from "Download file"  
    - Output: JSON array of contact records  
    - Connections: Output → "Code" node  
    - Edge cases:  
      - Malformed CSV files could cause parse errors  
      - Empty file results in empty output  
      - Encoding issues if CSV is not UTF-8  

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Normalizes and restructures contact data into a simplified JSON format suitable for Notion page creation.  
    - Configuration:  
      - Extracts fields: "First Name", "Last Name", "Display Name", "E-mail Address", "Mobile Phone", "Home Phone", and "Company"  
      - Logic:  
        - Constructs full name from "Display Name" or concatenates first and last name  
        - Chooses phone number priority: Mobile Phone over Home Phone  
        - Defaults to empty strings when fields are missing  
      - Outputs an array of JSON items, each representing one contact with properties: name, email, phone, company  
    - Input: JSON array of contacts from "Extract from File"  
    - Output: Transformed contact JSON array  
    - Connections: Output → "Create a database page" node  
    - Edge cases:  
      - Missing expected CSV columns results in empty strings for those fields  
      - Unexpected data types in CSV columns could cause runtime errors (unlikely given defensive coding)  
      - Large CSV files may affect performance  

#### 1.3 Notion Database Page Creation

- **Overview:**  
  Creates new pages in a specified Notion database for each contact with their respective details.

- **Nodes Involved:**  
  - Create a database page

- **Node Details:**

  - **Create a database page**  
    - Type: Notion node (resource: databasePage, operation: create)  
    - Role: Inserts each contact as a new page in the Notion database specified by its ID.  
    - Configuration:  
      - Database ID: "22243337-8e81-80a8-bc67-c2494034f1b4" (hardcoded target database)  
      - Title property: Set to contact's full name (`{{$json.name}}`)  
      - Properties mapped:  
        - Email Address (email property): populated from `{{$json.email}}` or null if empty  
        - Phone Number (phone_number property): populated from `{{$json.phone}}` or null if empty  
        - Company (rich_text property): populated from `{{$json.company}}` or "N/A" if empty  
    - Credentials: Notion API OAuth2 connection required  
    - Input: JSON contact data from "Code" node  
    - Output: Response from Notion API for page creation  
    - Edge cases:  
      - Authentication errors if Notion OAuth token is invalid or expired  
      - API rate limits from Notion if many contacts processed too quickly  
      - Property mismatch errors if Notion database schema changes  
      - Empty required fields might cause creation errors depending on Notion property settings  

#### 1.4 Documentation Block

- **Overview:**  
  A sticky note node provides instructions and context to users for using the workflow effectively.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Documents usage steps including required setup for API keys, Notion database setup, and data fields handled.  
    - Content Summary:  
      - Instructions to get a CSV contacts file from any contact manager app  
      - Notes on setting API keys for Google Drive and Notion  
      - Reminder to create a Notion database for contacts before running  
      - Current fields transferred: full name, email, phone, and company  
    - No input/output connections (informational only)  

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                        | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                                                                            |
|---------------------------|-------------------------|-------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger           | Manual start of the workflow          | —                           | Download file              |                                                                                                                                                                      |
| Download file             | Google Drive            | Download CSV file from Google Drive  | When clicking ‘Execute workflow’ | Extract from File          |                                                                                                                                                                      |
| Extract from File          | Extract From File       | Parse CSV binary to JSON              | Download file               | Code                       |                                                                                                                                                                      |
| Code                      | Code                    | Transform and normalize contact data | Extract from File           | Create a database page      |                                                                                                                                                                      |
| Create a database page     | Notion                  | Create pages in Notion database       | Code                        | —                          |                                                                                                                                                                      |
| Sticky Note               | Sticky Note             | Usage instructions                    | —                           | —                          | ## How to use 1. Get a .csv file with your contacts (you can download this from any contact manager app) 2. Set API key for Google Drive API, and Notion (you need to create a "connection" in Notion) 3. Create Database for your contacts in Notion 4. Choose which properties to extract from the .csv and pass it in to the Notion database. Right now, it transfer 4 pieces of information: full name, email, phone, and company. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name it: "When clicking ‘Execute workflow’"  
   - No special parameters needed.

3. **Add a Google Drive node:**  
   - Name it: "Download file"  
   - Set operation to "Download"  
   - Configure the fileId parameter with the Google Drive file ID containing the CSV (e.g., "1qYi9QBfBFu9jWXLoR7fdTn7FOwbceiL5")  
   - Set "Binary Property Name" to "data"  
   - Add Google Drive OAuth2 credentials (create and authorize if not existing).  
   - Connect output of the manual trigger node to this node.

4. **Add an Extract From File node:**  
   - Name it: "Extract from File"  
   - Leave default options (handles CSV automatically)  
   - Connect output of "Download file" node to this node.

5. **Add a Code node:**  
   - Name it: "Code"  
   - Paste the following JavaScript code to transform contacts:

```javascript
const contacts = [];

for (const contact of items) {
  const firstName = contact.json["First Name"] || "";
  const lastName = contact.json["Last Name"] || "";
  const fullName = contact.json["Display Name"] || `${firstName} ${lastName}`.trim();
  const email = contact.json["E-mail Address"] || "";
  const phone = contact.json["Mobile Phone"] || contact.json["Home Phone"] || "";

  contacts.push({
    json: {
      name: fullName,
      email: email,
      phone: phone,
      company: contact.json["Company"] || ""
    }
  });
}

return contacts;
```

   - Connect output of "Extract from File" node to this node.

6. **Add a Notion node:**  
   - Name it: "Create a database page"  
   - Set resource to "databasePage" and operation to "create"  
   - Set Database ID to your target Notion database ID (e.g., "22243337-8e81-80a8-bc67-c2494034f1b4")  
   - Configure properties mapping:  
     - Title property: `={{ $json.name }}`  
     - Email Address (email type): `={{ $json.email ? $json.email : null }}`  
     - Phone Number (phone_number type): `={{ $json.phone ? $json.phone : null }}`  
     - Company (rich_text type): `={{ $json.company ? $json.company : "N/A" }}`  
   - Add Notion API OAuth2 credentials (create and authorize if not existing).  
   - Connect output of "Code" node to this node.

7. **(Optional) Add a Sticky Note node for documentation:**  
   - Name it: "Sticky Note"  
   - Add usage instructions as content:  
     ```
     ## How to use
     1. Get a .csv file with your contacts (you can download this from any contact manager app)
     2. Set API key for Google Drive API, and Notion (you need to create a "connection" in Notion)
     3. Create Database for your contacts in Notion
     4. Choose which properties to extract from the .csv and pass it in to the Notion database. Right now, it transfer 4 pieces of information: full name, email, phone, and company.
     ```
   - Position it visually near the workflow nodes.

8. **Connect nodes in this order:**  
   Manual Trigger → Download file → Extract from File → Code → Create a database page

9. **Save and activate your workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                     | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To use this workflow, ensure you have valid OAuth2 credentials configured in n8n for both Google Drive and Notion APIs.                                                       | n8n credential setup interface                                                                  |
| The Notion database must have properties named exactly as used in the workflow for email, phone number, company, and title to avoid property mismatch errors.                 | Notion database schema requirements                                                            |
| CSV column headers expected: "First Name", "Last Name", "Display Name", "E-mail Address", "Mobile Phone", "Home Phone", "Company". Customize the Code node if your CSV differs.| CSV format and customization instructions                                                      |
| This workflow does not currently handle updates or deduplication; it creates a new page per contact every run. Consider adding logic for update detection if needed.          | Enhancement suggestion                                                                          |
| Sticky note provides clear, embedded workflow usage instructions for maintainers and end-users.                                                                                  | Workflow internal documentation                                                                |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a low-code integration and automation tool. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.