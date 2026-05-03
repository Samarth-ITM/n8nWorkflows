XML to SQL database import

https://n8nworkflows.xyz/workflows/xml-to-sql-database-import-1948


# XML to SQL database import

### 1. Workflow Overview

This workflow automates the process of importing data from an XML file into an SQL database table. It is designed primarily for use cases where structured XML data representing product information needs to be parsed and inserted into a MySQL database. The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger and File Loading:** Receives a manual execution trigger and loads the XML file in binary format from the server.
- **1.2 Data Extraction and Conversion:** Extracts raw text from the binary XML file and converts the XML content into a JSON object.
- **1.3 Data Splitting:** Splits the JSON structure into individual product records for database insertion.
- **1.4 Database Insertion:** Inserts each product record into a predefined SQL table.
- **1.5 (Optional) Table Creation:** A disabled MySQL node that can create and truncate the target SQL table based on a sample database schema.

This modular design facilitates straightforward modification or extension, such as adapting to different XML schemas or SQL databases.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger and File Loading

**Overview:**  
This block initiates the workflow manually and loads the XML file as binary data from the file system.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)  
- Read Binary Files

**Node Details:**

- **When clicking "Execute Workflow"**  
  - *Type & Role:* Manual trigger node; initiates workflow execution upon user action.  
  - *Configuration:* No parameters required; default manual trigger.  
  - *Inputs:* None (start node).  
  - *Outputs:* Connects to Read Binary Files node.  
  - *Failure Cases:* None typical; manual trigger dependent.  
  - *Version:* 1.

- **Read Binary Files**  
  - *Type & Role:* Reads a file from disk as binary data.  
  - *Configuration:* File path set to `/home/node/.n8n/intermediate.xml`. This path must exist and contain the XML file.  
  - *Inputs:* Receives trigger from manual node.  
  - *Outputs:* Passes binary file data to next node.  
  - *Failure Cases:* File not found, permission errors, or inaccessible file path could cause failure.  
  - *Version:* 1.

---

#### 2.2 Data Extraction and Conversion

**Overview:**  
Converts the binary XML data into a UTF-8 string and then parses this string into a JSON object representing the XML structure.

**Nodes Involved:**  
- Extract binary data (Code)  
- XML to JSON

**Node Details:**

- **Extract binary data**  
  - *Type & Role:* Code node that manually converts binary buffer to UTF-8 string.  
  - *Configuration:* JavaScript code uses `this.helpers.getBinaryDataBuffer(0, 'data')` to access binary data, then converts to string.  
  - *Key Expression:*  
    ```js
    let binaryDataBufferItem = await this.helpers.getBinaryDataBuffer(0, 'data');
    var data = binaryDataBufferItem.toString('utf8');
    return {"data": data};
    ```  
  - *Inputs:* Takes binary file from Read Binary Files node.  
  - *Outputs:* Outputs JSON with key `"data"` containing XML string.  
  - *Failure Cases:* Expression errors if binary data missing or malformed; encoding issues possible.  
  - *Version:* 2.

- **XML to JSON**  
  - *Type & Role:* Parses XML string into a JSON object.  
  - *Configuration:*  
    - `trim`: false (preserves whitespace)  
    - `attrkey`: `$` (attributes stored under `$` key)  
    - Other options set to false to prevent normalization or merging of attributes.  
  - *Inputs:* Receives XML string from previous node.  
  - *Outputs:* JSON object representing XML structure.  
  - *Failure Cases:* XML parse errors if input malformed or incomplete.  
  - *Version:* 1.

---

#### 2.3 Data Splitting

**Overview:**  
Breaks down the JSON array of products into individual items for insertion.

**Nodes Involved:**  
- Item Lists

**Node Details:**

- **Item Lists**  
  - *Type & Role:* Splits input JSON array into separate items based on specified field.  
  - *Configuration:* Splits on the field path `Products.Product` which corresponds to the array of products in JSON.  
  - *Inputs:* JSON object from XML to JSON node.  
  - *Outputs:* One item per product, enabling individual database insertion.  
  - *Failure Cases:* If the field path does not exist or is malformed, splitting will fail or produce no items.  
  - *Version:* 3.

---

#### 2.4 Database Insertion

**Overview:**  
Inserts each product record into the SQL table `new_table` with mapped columns.

**Nodes Involved:**  
- Add new records

**Node Details:**

- **Add new records**  
  - *Type & Role:* MySQL node performing inserts into a specified table.  
  - *Configuration:*  
    - Table: `new_table` (exists or needs to be created beforehand)  
    - Data mode: Define below with explicit column mappings:  
      - `productCode` ← `$json.Code`  
      - `productName` ← `$json.Name`  
      - `productLine` ← `$json.Line`  
      - `productScale` ← `$json.Scale`  
      - `productDescription` ← `$json.Description`  
      - `MSRP` ← `$json.Price`  
      - `productVendor` ← `"NA"` (default hardcoded)  
      - `quantityInStock` ← `0` (default hardcoded)  
      - `buyPrice` ← `0` (default hardcoded)  
    - Credentials: Uses configured MySQL credential named `db4free MySQL`.  
    - Option enabled for detailed output.  
  - *Inputs:* One product item per execution from Item Lists.  
  - *Outputs:* SQL insert result details.  
  - *Failure Cases:* SQL errors if table does not exist, schema mismatch, or connection issues. Data type mismatches could cause errors.  
  - *Version:* 2.2.

---

#### 2.5 (Optional) Table Creation

**Overview:**  
A disabled MySQL node that, when enabled and executed, creates the `new_table` by copying the schema of an existing `products` table and truncates it.

**Nodes Involved:**  
- Create new table

**Node Details:**

- **Create new table**  
  - *Type & Role:* Executes raw SQL queries to prepare the database table.  
  - *Configuration:*  
    - Query:  
      ```
      CREATE TABLE IF NOT EXISTS new_table AS SELECT * FROM products;
      TRUNCATE new_table;
      ```  
    - Operation: Execute query (no data retrieval).  
    - Credentials: Same `db4free MySQL` credential.  
  - *Inputs:* None (standalone).  
  - *Outputs:* Query execution result.  
  - *Failure Cases:* SQL permission errors, missing `products` table, or syntax errors.  
  - *Version:* 2.2.  
  - *Note:* Disabled by default; recommended to run only once or when schema changes.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                   | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                       |
|------------------------|---------------------|---------------------------------|-----------------------------|---------------------------|-------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger      | Initiates workflow execution     | —                           | Read Binary Files          |                                                                                                 |
| Read Binary Files       | Read Binary Files   | Loads XML file from disk          | When clicking "Execute Workflow" | Extract binary data        |                                                                                                 |
| Extract binary data     | Code                | Converts binary to UTF-8 string   | Read Binary Files            | XML to JSON               |                                                                                                 |
| XML to JSON            | XML                  | Parses XML string to JSON         | Extract binary data          | Item Lists                |                                                                                                 |
| Item Lists             | Item Lists           | Splits JSON array into items      | XML to JSON                 | Add new records           |                                                                                                 |
| Add new records        | MySQL                | Inserts product data into database | Item Lists                  | —                         |                                                                                                 |
| Create new table       | MySQL (disabled)     | Creates and truncates target table | —                           | —                         | Activate and execute this node only when needed. CREATE TABLE IF NOT EXISTS new_table AS SELECT * FROM products; TRUNCATE new_table; |
| Sticky Note            | Sticky Note          | Describes Create new table node   | —                           | —                         | Activate and execute this node only when needed. CREATE TABLE IF NOT EXISTS new_table AS SELECT * FROM products; TRUNCATE new_table; |
| Sticky Note1           | Sticky Note          | Contains example XML file content | —                           | —                         | This is a content of the example XML file. Please use it if the file was not already created [XML content shown] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node. No parameters needed. This will start the workflow manually.

2. **Create Read Binary Files Node**  
   - Add a **Read Binary Files** node.  
   - Set `File Selector` to the XML file path, e.g., `/home/node/.n8n/intermediate.xml`.  
   - Connect output of Manual Trigger to this node.

3. **Create Code Node to Extract Binary Data**  
   - Add a **Code** node.  
   - Set language to JavaScript.  
   - Paste the following code:  
     ```js
     let binaryDataBufferItem = await this.helpers.getBinaryDataBuffer(0, 'data');
     var data = binaryDataBufferItem.toString('utf8');
     return {"data": data};
     ```  
   - Connect output of Read Binary Files node to this node.

4. **Create XML Node to Convert XML to JSON**  
   - Add an **XML** node.  
   - Set options:  
     - `Trim` = false  
     - `Attr Key` = `$`  
     - `Normalize` = false  
     - `Merge Attrs` = true  
     - `Ignore Attrs` = false  
     - `Normalize Tags` = false  
   - Connect output of Code node to this node.

5. **Create Item Lists Node to Split Products**  
   - Add an **Item Lists** node.  
   - Set `Field To Split Out` to `Products.Product` (matches the XML structure).  
   - Connect output of XML node to this node.

6. **Create MySQL Node to Insert Records**  
   - Add a **MySQL** node.  
   - Set operation to `Insert`.  
   - Select or type target table: `new_table`.  
   - Set Data Mode to `Define Below`.  
   - Map columns with values:  
     - productCode: `={{ $json.Code }}`  
     - productName: `={{ $json.Name }}`  
     - productLine: `={{ $json.Line }}`  
     - productScale: `={{ $json.Scale }}`  
     - productDescription: `={{ $json.Description }}`  
     - MSRP: `={{ $json.Price }}`  
     - productVendor: `NA` (fixed string)  
     - quantityInStock: `0` (fixed number)  
     - buyPrice: `0` (fixed number)  
   - Connect output of Item Lists node to this node.  
   - Configure MySQL credentials with your database connection (e.g., `db4free MySQL`).

7. **(Optional) Create MySQL Node for Table Setup**  
   - Add a **MySQL** node.  
   - Set operation to `Execute Query`.  
   - Enter SQL query:  
     ```
     CREATE TABLE IF NOT EXISTS new_table AS SELECT * FROM products;
     TRUNCATE new_table;
     ```  
   - Configure to use the same MySQL credentials.  
   - Disable this node by default.  
   - Use this node only when you need to create or reset the target table schema.

8. **(Optional) Add Sticky Notes**  
   - Add Sticky Note nodes to document the table creation SQL and example XML content for reference.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Activate and execute the table creation node only when needed to create/reset the `new_table`.           | See the disabled MySQL node "Create new table".     |
| Example XML file content is provided in a sticky note for reference and to create the XML file if missing.| The XML represents a product list structure.        |
| Sample SQL database used: https://www.mysqltutorial.org/mysql-sample-database.aspx                        | Reference for the source schema of the `products` table. |

---

This completes the comprehensive documentation of the "XML to SQL database import" n8n workflow. It enables understanding, reproduction, and troubleshooting of each step involved in importing XML data into an SQL table.