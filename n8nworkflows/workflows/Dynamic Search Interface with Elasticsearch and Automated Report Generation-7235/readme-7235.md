Dynamic Search Interface with Elasticsearch and Automated Report Generation

https://n8nworkflows.xyz/workflows/dynamic-search-interface-with-elasticsearch-and-automated-report-generation-7235


# Dynamic Search Interface with Elasticsearch and Automated Report Generation

### 1. Workflow Overview

This workflow implements a **Dynamic Search Interface** targeting banking transaction data indexed in Elasticsearch. It allows users to specify search criteria via a web form, performs a time-bound and amount-filtered query on the bank transactions index, and generates automated reports in either text or CSV formats. The reports summarize suspicious transactions matching the criteria and are saved as timestamped files on the server.

**Target Use Cases:**  
- Fraud detection or transaction monitoring teams searching for suspicious banking activities  
- Automated generation of audit or compliance reports  
- Dynamic ad-hoc querying with flexible time ranges and optional customer filters

**Logical Blocks:**

- **1.1 Input Reception:** User input collected via a dynamic web form with validation.  
- **1.2 Query Construction:** Transform user input into a structured Elasticsearch JSON query.  
- **1.3 Data Retrieval:** Execute the query against Elasticsearch securely and retrieve results.  
- **1.4 Report Generation:** Format the retrieved data into human-readable text or CSV reports, including metadata.  
- **1.5 File Storage:** Save the generated report as a binary file on disk with a timestamped filename.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures user input through a web form that dynamically requests search parameters including minimum amount, time range, optional customer ID, and report format. This serves as the workflow entry point.

- **Nodes Involved:**  
  - *Search Form*  
  - *Sticky Note* (entry point description)

- **Node Details:**

  - **Search Form**  
    - *Type:* Form Trigger  
    - *Role:* Entry point for user interaction, exposes a dynamic form at a webhook URL.  
    - *Configuration:*  
      - Path: Unique webhook ID-based path  
      - Form Fields:  
        - Minimum Amount ($) [number, optional, defaults handled later]  
        - Time Range [dropdown, required, options: Last 1 Hour, 6 Hours, 24 Hours, 3 Days]  
        - Customer ID (optional) [text input]  
        - Report Format [dropdown, required, options: Text Report, CSV Export]  
      - Response Mode: Immediate response node  
      - Form Description: "Search for suspicious transactions in your banking database"  
    - *Inputs:* None (trigger node)  
    - *Outputs:* JSON with form data  
    - *Edge Cases:*  
      - Missing required fields (auto-validated)  
      - User input invalid types (number expected for amount)  
      - Empty optional Customer ID  

  - **Sticky Note** (Entry Point)  
    - Describes the purpose and inputs of the form.  
    - No functional role, purely documentation.

#### 1.2 Query Construction

- **Overview:**  
  Processes the user form inputs and converts them into a well-structured Elasticsearch query JSON. Handles default values, time range conversion, and conditional inclusion of customer filters.

- **Nodes Involved:**  
  - *Build Search Query*  
  - *Sticky Note1* (explains query building)

- **Node Details:**

  - **Build Search Query**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Build Elasticsearch query object and metadata from form input.  
    - *Configuration Highlights:*  
      - Extracts form parameters: Minimum Amount (default 1000), Time Range (default Last 24 Hours), Customer ID (optional), Report Format (default Text Report)  
      - Converts time range labels to Elasticsearch relative time strings (e.g., "now-24h")  
      - Constructs a `bool.must` query array with:  
        - Timestamp range filter (gte)  
        - Amount range filter (gte)  
        - Optional term filter on `customer_id.keyword` if Customer ID provided  
      - Sorts results by timestamp descending, limits size to 100  
      - Returns `elasticsearchQuery` (to be sent to ES) and `searchParams` (for downstream metadata)  
    - *Inputs:* Form data JSON from Search Form node  
    - *Outputs:* JSON with ES query and parameters for reporting  
    - *Edge Cases:*  
      - Missing or malformed form inputs (handled with defaults)  
      - Customer ID whitespace trimming  
      - Unexpected time range values (default fallback)  

  - **Sticky Note1**  
    - Explains conversion logic and validation inside this node.  
    - Notes on time range mapping and filter composition.

#### 1.3 Data Retrieval

- **Overview:**  
  Executes the constructed Elasticsearch query via HTTP POST to the bank_transactions index on a local ES instance, using HTTP Basic Authentication.

- **Nodes Involved:**  
  - *Search Elasticsearch*  
  - *Sticky Note2* (explains ES search details)

- **Node Details:**

  - **Search Elasticsearch**  
    - *Type:* HTTP Request  
    - *Role:* Query Elasticsearch cluster with JSON body from prior node  
    - *Configuration:*  
      - URL: `https://localhost:9220/bank_transactions/_search` (local ES instance)  
      - Method: POST  
      - Body: JSON (ES query from Build Search Query)  
      - Authentication: HTTP Basic Auth (credential stored in n8n)  
      - Allow unauthorized certificates enabled (for local HTTPS)  
      - Returns JSON ES search results  
    - *Inputs:* Elasticsearch query JSON  
    - *Outputs:* Raw Elasticsearch response JSON  
    - *Edge Cases:*  
      - Connection errors (ES down, network issues)  
      - Authentication failures (invalid credentials)  
      - Malformed query errors (unlikely, controlled by previous node)  
      - Empty or zero hits responses  

  - **Sticky Note2**  
    - Describes the ES index and authentication setup.  
    - Notes on max results and sorting.

#### 1.4 Report Generation

- **Overview:**  
  Converts raw Elasticsearch hits into user-friendly reports, supporting two formats: human-readable text summary or CSV export. Generates filenames and MIME types accordingly and converts content to binary for file operations.

- **Nodes Involved:**  
  - *Format Report*  
  - *Sticky Note3* (explains report generation logic)

- **Node Details:**

  - **Format Report**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Build report content and metadata from ES search results and parameters  
    - *Configuration Highlights:*  
      - Extracts hits and total from ES response  
      - Uses searchParams from Build Search Query node for report metadata  
      - Generates ISO date-based filename: `report_YYYY-MM-DD.txt` or `.csv`  
      - For CSV: outputs header row and transaction rows with fields: Transaction_ID, Customer_ID, Amount, Merchant_Category, Timestamp  
      - For Text: outputs a formatted summary with search criteria and detailed listing of each transaction  
      - Converts report content string into binary buffer for file writing  
      - Returns JSON with filename, mimeType, content, and binary data  
    - *Inputs:* Raw ES results + metadata from prior nodes  
    - *Outputs:* Composite JSON + binary data object for file writing  
    - *Edge Cases:*  
      - No transactions found (outputs friendly message)  
      - Missing fields in hits (gracefully handled)  
      - Invalid format selection (defaults handled upstream)  

  - **Sticky Note3**  
    - Describes the two report formats and output details.

#### 1.5 File Storage

- **Overview:**  
  Saves the generated report content as a file on the local server disk under `/tmp`, using the generated filename and binary data.

- **Nodes Involved:**  
  - *Read/Write Files from Disk*  
  - *Sticky Note4* (explains file saving)

- **Node Details:**

  - **Read/Write Files from Disk**  
    - *Type:* Read/Write File node  
    - *Role:* Write report binary data to file system  
    - *Configuration:*  
      - Operation: Write  
      - Filename: dynamic `/tmp/{{ $json.filename }}` from prior node  
      - Uses binary data input named `data` (from Format Report node)  
    - *Inputs:* Binary data + filename JSON  
    - *Outputs:* Confirmation of write operation (file metadata)  
    - *Edge Cases:*  
      - Disk permission errors  
      - Disk space issues  
      - Filename conflicts (timestamped to mitigate)  

  - **Sticky Note4**  
    - Describes location and use cases for saved report files.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                     | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                 |
|---------------------------|---------------------|-----------------------------------|---------------------|--------------------------|---------------------------------------------------------------------------------------------|
| Search Form               | Form Trigger        | User input form entry point       | None                | Build Search Query       | 🎯 ENTRY POINT: User fills dynamic search form with filters and output format; auto-validates |
| Build Search Query        | Code                | Build ES query from form input    | Search Form         | Search Elasticsearch     | 🔧 QUERY BUILDER: Converts inputs to ES query JSON with filters and time range mapping      |
| Search Elasticsearch      | HTTP Request        | Query ES index with built query   | Build Search Query  | Format Report            | 🎯 DATA HUNTER: Sends POST to ES with basic auth; max 100 results, newest first             |
| Format Report             | Code                | Format ES results to report format| Search Elasticsearch| Read/Write Files from Disk| 📝 REPORT GENERATOR: Creates text or CSV report, generates timestamped filename, binary data|
| Read/Write Files from Disk| Read/Write File     | Save report file on server disk   | Format Report       | None                     | 📁 FILE SAVER: Writes report to /tmp with binary data, ready for download or further use    |
| Sticky Note               | Sticky Note         | Documentation                     | None                | None                     | 🎯 ENTRY POINT                                                                           |
| Sticky Note1              | Sticky Note         | Documentation                     | None                | None                     | 🔧 QUERY BUILDER                                                                        |
| Sticky Note2              | Sticky Note         | Documentation                     | None                | None                     | 🎯 DATA HUNTER                                                                         |
| Sticky Note3              | Sticky Note         | Documentation                     | None                | None                     | 📝 REPORT GENERATOR                                                                    |
| Sticky Note4              | Sticky Note         | Documentation                     | None                | None                     | 📁 FILE SAVER                                                                          |
| Sticky Note5              | Sticky Note         | Documentation                     | None                | None                     | 🎯 DYNAMIC SEARCH PIPELINE summary: User Form → Query Builder → ES Search → Report Format → File Save |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Add **Form Trigger** node named "Search Form".  
   - Set webhook path to unique ID (or custom path).  
   - Configure form title: "🔍 Dynamic Search".  
   - Add form fields:  
     - "Minimum Amount ($)" as number, optional.  
     - "Time Range" as dropdown, required, options: "Last 1 Hour", "Last 6 Hours", "Last 24 Hours", "Last 3 Days".  
     - "Customer ID (Optional)" as text input, optional.  
     - "Report Format" as dropdown, required, options: "Text Report", "CSV Export".  
   - Set form description: "Search for suspicious transactions in your banking database".  
   - Set response mode to immediate response node.

2. **Create Code Node for Query Building:**  
   - Add **Code** node named "Build Search Query".  
   - Connect input from "Search Form".  
   - Paste JavaScript code to:  
     - Extract form inputs with defaults.  
     - Map time range to ES relative time string.  
     - Build ES query with `bool.must` including timestamp range and amount range filters.  
     - Conditionally add customer ID filter if provided.  
     - Sort by timestamp descending, limit size 100.  
     - Return `elasticsearchQuery` and `searchParams` JSON.  

3. **Create HTTP Request Node for ES Search:**  
   - Add **HTTP Request** node named "Search Elasticsearch".  
   - Connect input from "Build Search Query".  
   - Set method to POST.  
   - URL: `https://localhost:9220/bank_transactions/_search`.  
   - Enable "Allow Unauthorized Certificates" (for local HTTPS).  
   - Set body to JSON, pass `elasticsearchQuery` from previous node.  
   - Configure HTTP Basic Auth credentials with valid ES user/pass.  
   - Set output to JSON.

4. **Create Code Node for Report Formatting:**  
   - Add **Code** node named "Format Report".  
   - Connect input from "Search Elasticsearch".  
   - Paste JavaScript code to:  
     - Extract ES response hits and metadata.  
     - Determine report format.  
     - Generate timestamped filename with `.txt` or `.csv`.  
     - For CSV: create header and rows.  
     - For Text: create human-readable summary with search criteria and transactions.  
     - Convert content string to binary buffer.  
     - Return JSON + binary data for file writing.

5. **Create Read/Write File Node:**  
   - Add **Read/Write Files from Disk** node named "Read/Write Files from Disk".  
   - Connect input from "Format Report".  
   - Set operation to "write".  
   - Set filename to `/tmp/{{ $json.filename }}` (dynamic from previous node).  
   - Use binary data field `data` from previous node.  

6. **Add Sticky Notes (Optional, for documentation):**  
   - Add sticky notes describing each block as per the original workflow.  
   - Position notes near respective nodes for clarity.

7. **Set Execution Order and Activate Workflow:**  
   - Ensure connections follow this sequence:  
     Search Form → Build Search Query → Search Elasticsearch → Format Report → Read/Write Files from Disk  
   - Activate workflow and test by submitting the form.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Form auto-validates required fields to prevent malformed submissions.                         | Search Form node configuration                                                                            |
| Elasticsearch query limits results to 100 newest transactions sorted descending by timestamp. | Query Builder and Search Elasticsearch nodes                                                             |
| File saving location is `/tmp/` with timestamped filenames for easy retrieval and audit.      | Read/Write Files from Disk node                                                                           |
| The workflow typically executes within 2-5 seconds depending on ES response times.            | Sticky Note5 summary                                                                                       |
| Requires valid Elasticsearch HTTP Basic Auth credentials configured in n8n credentials.      | Search Elasticsearch node                                                                                  |
| Allows optional filtering by customer ID to narrow down suspicious transactions.              | Build Search Query node                                                                                     |
| Reports can be downloaded, attached, or processed further after file writing.                  | General use case of file output                                                                             |

---

**Disclaimer:**  
The provided text is exclusively generated from an n8n automated workflow export. The workflow strictly complies with content policies and contains no illegal or protected elements. All data handled is legal and publicly accessible.