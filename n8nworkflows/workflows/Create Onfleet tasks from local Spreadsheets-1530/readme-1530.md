Create Onfleet tasks from local Spreadsheets

https://n8nworkflows.xyz/workflows/create-onfleet-tasks-from-local-spreadsheets-1530


# Create Onfleet tasks from local Spreadsheets

### 1. Workflow Overview

This workflow automates the creation of Onfleet delivery tasks by importing data from a local spreadsheet file. It is designed for users who manage last-mile delivery operations and want to bulk-import delivery tasks directly into Onfleet’s platform without manual entry.

**Target Use Case:**  
One-time or periodic batch import of delivery tasks into Onfleet from locally stored spreadsheets, enabling streamlined route planning and dispatch.

**Logical Blocks:**

- **1.1 Input Reception:** Reads a local spreadsheet file containing delivery task details.
- **1.2 Data Parsing:** Converts the binary file content into structured JSON data.
- **1.3 Onfleet Task Creation:** Creates delivery tasks in Onfleet using the parsed spreadsheet data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block reads the local spreadsheet file in binary format from disk to prepare for further processing.

- **Nodes Involved:**  
  - Read Binary File

- **Node Details:**

  - **Node Name:** Read Binary File  
    - **Type:** `readBinaryFile`  
    - **Role:** Reads a file from the local file system into binary data for subsequent processing.  
    - **Configuration:**  
      - `filePath` set to the absolute path of the spreadsheet file (e.g., `/Users/jamesli/Downloads/Onfleet Import Google Sheet.xlsx`).  
    - **Key Expressions/Variables:** None.  
    - **Input Connections:** None (starting point).  
    - **Output Connections:** Connected to `Spreadsheet File1`.  
    - **Version Requirements:** Standard node, compatible with n8n v0.150+.  
    - **Potential Failures:**  
      - File not found or path incorrect → node fails.  
      - Permission denied on file → node fails.  
      - Unsupported file types → downstream parsing may fail.  
    - **Notes:** Requires the workflow runner to have access to the specified local file path.

#### 1.2 Data Parsing

- **Overview:**  
  Converts the binary spreadsheet file to structured JSON data representing each row as an object with columns as keys.

- **Nodes Involved:**  
  - Spreadsheet File1

- **Node Details:**

  - **Node Name:** Spreadsheet File1  
    - **Type:** `spreadsheetFile`  
    - **Role:** Parses binary spreadsheet data (XLSX format) into JSON objects for processing.  
    - **Configuration:**  
      - Default options, no filters or sheet selection specified (assumes first sheet).  
    - **Key Expressions/Variables:** None.  
    - **Input Connections:** Receives binary data from `Read Binary File`.  
    - **Output Connections:** Feeds JSON to `Onfleet`.  
    - **Version Requirements:** Ensure compatibility with XLSX parsing in n8n v0.150+.  
    - **Potential Failures:**  
      - Corrupted or malformed spreadsheet → parsing error.  
      - Unsupported spreadsheet format → error.  
      - Large files may cause performance issues.  
    - **Notes:** Users can add options for sheet selection or range if needed.

#### 1.3 Onfleet Task Creation

- **Overview:**  
  Creates delivery tasks in Onfleet for each row of parsed spreadsheet data, mapping spreadsheet columns to Onfleet task properties.

- **Nodes Involved:**  
  - Onfleet

- **Node Details:**

  - **Node Name:** Onfleet  
    - **Type:** `onfleet` (n8n Onfleet node)  
    - **Role:** Uses Onfleet API to create delivery tasks with specified destination and recipient info.  
    - **Configuration:**  
      - Operation: `create` task.  
      - Destination address is composed by concatenating spreadsheet columns:  
        `Address_Line1`, `Address_Line2`, `City/Town`, `State/Province`, `Country`, `Postal_Code`.  
      - AddressNotes is empty.  
      - AddressApartment mapped to `Address_Line2`.  
      - Additional fields:  
        - `notes` mapped from `Task_Details`.  
        - `recipientName` mapped from `Recipient_Name`.  
        - `recipientNotes` mapped from `Recipient_Notes`.  
        - `recipientPhone` formatted as `+1` concatenated with `Recipient_Phone` (assumes US country code).  
      - Credentials: Uses stored Onfleet API Key credential.  
    - **Key Expressions/Variables:**  
      - Expression syntax e.g., `={{$json["Address_Line1"]}}` etc., to map spreadsheet columns dynamically.  
    - **Input Connections:** Receives JSON objects from `Spreadsheet File1`.  
    - **Output Connections:** None (end node).  
    - **Version Requirements:** Onfleet API version compatible with n8n Onfleet node v1.  
    - **Potential Failures:**  
      - Authentication errors if API key invalid.  
      - Address validation errors if address fields incomplete or malformed.  
      - Phone number format issues if input inconsistent.  
      - API rate limiting if large batch executed at once.  
      - Missing required fields in input JSON → API call fails.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role               | Input Node(s)     | Output Node(s)   | Sticky Note                                                                                      |
|--------------------|---------------------|------------------------------|-------------------|------------------|-------------------------------------------------------------------------------------------------|
| Read Binary File    | readBinaryFile      | Reads local spreadsheet file | None              | Spreadsheet File1 | Update the `Read Binary File` node with the absolute file path to your local spreadsheet.        |
| Spreadsheet File1   | spreadsheetFile     | Parses spreadsheet to JSON   | Read Binary File  | Onfleet          |                                                                                                 |
| Onfleet            | onfleet             | Creates Onfleet delivery tasks | Spreadsheet File1 | None             | Update Onfleet node with your Onfleet API key. Register at https://onfleet.com/signup              |
|                    |                     |                              |                   |                  | For import templates visit https://support.onfleet.com/hc/en-us/articles/360023910131-Task-Overview#h_4667f289-d298-49bc-9faa-00898a922dab |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node:** `Read Binary File`  
   - Type: `Read Binary File`  
   - Parameters:  
     - `filePath`: Set to the absolute path of your local spreadsheet file (e.g., `/Users/yourname/Documents/OnfleetImport.xlsx`).  
   - No credentials required.  

2. **Create Node:** `Spreadsheet File1`  
   - Type: `Spreadsheet File`  
   - Parameters:  
     - Leave default options unless you need to specify a particular sheet or range.  
   - Connect output of `Read Binary File` node to input of this node.

3. **Create Node:** `Onfleet`  
   - Type: `Onfleet` node (requires Onfleet API credentials).  
   - Credentials: Configure using your Onfleet API Key credential. Create this credential in n8n by providing your Onfleet API key from https://onfleet.com/signup.  
   - Parameters:  
     - Operation: `create` task  
     - Destination:  
       - Address: Use expression to concatenate address fields from spreadsheet data:  
         `={{$json["Address_Line1"]}}, {{$json["Address_Line2"]}}, {{$json["City/Town"]}} {{$json["State/Province"]}}, {{$json["Country"]}}, {{$json["Postal_Code"]}}`  
       - Address Apartment: `={{$json["Address_Line2"]}}`  
       - Address Notes: leave empty or set as needed  
     - Additional Fields:  
       - Notes: `={{$json["Task_Details"]}}`  
       - Recipient:  
         - Name: `={{$json["Recipient_Name"]}}`  
         - Notes: `={{$json["Recipient_Notes"]}}`  
         - Phone: `=+1{{$json["Recipient_Phone"]}}` (assumes US country code; adjust if needed)  
   - Connect output of `Spreadsheet File1` node to input of this node.

4. **Workflow Activation:**  
   - After setting up nodes and connections, save the workflow.  
   - Trigger manually or via webhook to run once and create Onfleet tasks from the spreadsheet data.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| To get an Onfleet API key, sign up at https://onfleet.com/signup                                | Onfleet API credentials                                                                                             |
| For Onfleet task import templates and field mapping, see Onfleet Support documentation          | https://support.onfleet.com/hc/en-us/articles/360023910131-Task-Overview#h_4667f289-d298-49bc-9faa-00898a922dab    |
| Adjust `recipientPhone` formatting if your delivery region is outside the US (+1)               | Phone number formatting is hardcoded with +1 country code; modify expressions accordingly for other countries.     |
| Ensure the n8n instance has read access to the local file path specified in `Read Binary File`  | Local file system permissions                                                                                       |

---

This documentation fully captures the workflow's design, node configurations, data flow, and integration points for both manual rebuilding and automated processing.