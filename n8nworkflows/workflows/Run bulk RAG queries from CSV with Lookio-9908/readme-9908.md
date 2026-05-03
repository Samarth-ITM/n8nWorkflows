Run bulk RAG queries from CSV with Lookio

https://n8nworkflows.xyz/workflows/run-bulk-rag-queries-from-csv-with-lookio-9908


# Run bulk RAG queries from CSV with Lookio

### 1. Workflow Overview

This workflow enables bulk retrieval-augmented generation (RAG) queries against a Lookio knowledge assistant using input CSV files. Users upload a CSV file containing a column named **Query** via a web form. The workflow extracts each query row, sends it to the Lookio API, collects the responses, and generates an enriched CSV file with an added **Response** column containing Lookio's answers. Finally, the user can download the enriched CSV from the form completion screen.

**Target Use Cases:**  
- Bulk knowledge retrieval from Lookio assistants using CSV input  
- Automating mass queries for data enrichment or analysis  
- Seamless integration of Lookio’s AI-powered responses into CSV workflows  

**Logical Blocks:**  
- **1.1 Input Reception:** Handles user CSV upload through a form and initial file extraction.  
- **1.2 CSV Validation and Splitting:** Checks for the required "Query" column and splits CSV rows for batch processing.  
- **1.3 Query Preparation:** Isolates the Query field from each row for API consumption.  
- **1.4 Lookio API Interaction:** Sends individual queries to Lookio and receives AI-generated responses.  
- **1.5 Response Aggregation and Output:** Aggregates responses, constructs enriched CSV, and provides file download via form completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives CSV input from a user via a web form trigger node, then extracts file contents for processing.

**Nodes Involved:**  
- On form submission  
- Extract all rows  
- Aggregate rows  

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point accepting CSV upload named "Bulk Lookio queries"  
  - *Configuration:*  
    - Form with single required file field accepting `.csv` files only  
    - Response type set to redirect (for better UX after submission)  
    - Form description instructs to upload CSV with a "Query" column  
  - *Input:* User-submitted HTTP request with CSV file  
  - *Output:* Binary CSV data attached to JSON for downstream processing  
  - *Edge cases:* Missing file upload, file format errors  
  - *Version:* 2.3 (supports form trigger features)  

- **Extract all rows**  
  - *Type:* Extract from File  
  - *Role:* Parses CSV binary data into JSON rows  
  - *Configuration:* Reads CSV from binary property named "CSV" (from form)  
  - *Input:* Binary CSV file from form submission node  
  - *Output:* JSON array with all CSV rows under `data` property  
  - *Edge cases:* Malformed CSV, parsing errors  
  - *Version:* 1  

- **Aggregate rows**  
  - *Type:* Aggregate  
  - *Role:* Flattens extracted CSV rows into a single JSON array (all items)  
  - *Configuration:* Aggregation mode set to “aggregate all item data”  
  - *Input:* Multiple items from "Extract all rows" node  
  - *Output:* Single aggregated JSON array for easier processing  
  - *Edge cases:* Empty input array  

---

#### 1.2 CSV Validation and Splitting

**Overview:**  
Validates presence of the “Query” column in the CSV and splits the data into individual rows for iteration.

**Nodes Involved:**  
- Valid Query column? (IF)  
- Error message  
- Split Out  

**Node Details:**  

- **Valid Query column?**  
  - *Type:* IF  
  - *Role:* Checks if the first row contains a non-empty "Query" field  
  - *Configuration:*  
    - Expression checks existence of `$json.data[0].Query` (case-sensitive)  
    - Condition: string exists and is non-empty  
  - *Input:* Aggregated CSV rows from previous block  
  - *Output:* Two branches: true (valid) and false (invalid)  
  - *Edge cases:* CSV without "Query" column, empty "Query" values, case sensitivity issues  
  - *Version:* 2.2  

- **Error message**  
  - *Type:* Form  
  - *Role:* Returns error feedback to user if CSV lacks "Query" column  
  - *Configuration:*  
    - Completion operation with title “Error” and message “No "query" column has been found”  
  - *Input:* False branch from validation node  
  - *Output:* User-facing error response  
  - *Edge cases:* None beyond upstream validation failure  
  - *Version:* 2.3  

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Extracts the `data` array from aggregated JSON to individual items (rows)  
  - *Configuration:* Field to split out set to “data”  
  - *Input:* True branch from validation node (aggregated rows)  
  - *Output:* One item per CSV row for processing  
  - *Edge cases:* Empty data array  

---

#### 1.3 Query Preparation

**Overview:**  
Selects only the “Query” field from each CSV row to prepare for API calls.

**Nodes Involved:**  
- Isolate the Query column  
- Loop Over Queries  

**Node Details:**  

- **Isolate the Query column**  
  - *Type:* Set  
  - *Role:* Keeps only the "Query" property for each item  
  - *Configuration:* Assigns a new string field "Query" with value from `$json.Query`  
  - *Input:* Individual CSV rows from Split Out  
  - *Output:* Simplified JSON containing only the Query string  
  - *Edge cases:* Missing Query field in any row (should not happen due to validation)  
  - *Version:* 3.4  

- **Loop Over Queries**  
  - *Type:* Split In Batches  
  - *Role:* Processes queries one by one or in small batches to control API load  
  - *Configuration:* Default batch size (unspecified, defaults to 1)  
  - *Input:* Prepared Query-only items  
  - *Output:* Single or batch items for API call  
  - *Edge cases:* Large CSVs may cause slow processing or rate limits  
  - *Version:* 3  

---

#### 1.4 Lookio API Interaction

**Overview:**  
Calls Lookio API with each query and obtains the corresponding response.

**Nodes Involved:**  
- Lookio API call  
- Prepare output  

**Node Details:**  

- **Lookio API call**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST requests to Lookio API webhook endpoint for each query  
  - *Configuration:*  
    - URL: `https://api.lookio.app/webhook/query`  
    - Method: POST  
    - Body Parameters:  
      - `query`: current query text from `$json.Query`  
      - `assistant_id`: placeholder `<your-assistant-id>` (must be replaced)  
      - `query_mode`: set to `"flash"` (fast query mode)  
    - Header Parameters:  
      - `api_key`: placeholder `<your-lookio-api-key>` (must be replaced)  
    - Sends both body and headers as form parameters  
  - *Input:* Individual query items from Loop Over Queries  
  - *Output:* JSON response with AI-generated output in `$json.Output`  
  - *Edge cases:*  
    - Authentication failure (invalid API key)  
    - Network timeouts  
    - API errors or rate limits  
  - *Version:* 4.2  

- **Prepare output**  
  - *Type:* Set  
  - *Role:* Combines original Query and Lookio response into one item for final output  
  - *Configuration:*  
    - Sets "Query" to the original query (`$('Loop Over Queries').item.json.Query`)  
    - Sets "Response" to Lookio API response output (`$json.Output`)  
  - *Input:* Lookio API call response  
  - *Output:* JSON with both Query and Response fields for CSV enrichment  
  - *Edge cases:* Missing or malformed API response fields  
  - *Version:* 3.4  

---

#### 1.5 Response Aggregation and Output

**Overview:**  
Generates an enriched CSV with the original queries and Lookio responses and returns it to the user.

**Nodes Involved:**  
- Generate enriched CSV  
- Form ending and file download  

**Node Details:**  

- **Generate enriched CSV**  
  - *Type:* Convert To File  
  - *Role:* Converts JSON array of query-response objects into CSV file format  
  - *Configuration:*  
    - Filename set to “Lookio bulk result.csv”  
  - *Input:* Items from Loop Over Queries (after Prepare output)  
  - *Output:* Binary CSV file ready for download  
  - *Edge cases:* Special characters in text fields, empty responses  
  - *Version:* 1.1  

- **Form ending and file download**  
  - *Type:* Form  
  - *Role:* Final form node that returns the enriched CSV file as a downloadable response  
  - *Configuration:*  
    - Operation: completion  
    - Respond with binary file (`returnBinary`)  
    - Completion title: “Your enriched file is ready”  
    - Completion message: informs user about added “Response” column  
  - *Input:* Binary CSV file from Generate enriched CSV  
  - *Output:* HTTP response with downloadable CSV file  
  - *Edge cases:* File size limits, network interruptions  
  - *Version:* 2.3  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                                | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                         |
|---------------------------|---------------------|-----------------------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| On form submission        | Form Trigger        | Entry point for CSV upload via form             | —                            | Extract all rows               | # CSV bulk RAG queries with Lookio<br>Upload a CSV with a column named **Query** and get back a CSV with new **Response** column populated by Lookio.<br>Setup instructions included. |
| Extract all rows          | Extract From File   | Parses uploaded CSV file into JSON rows        | On form submission           | Aggregate rows                |                                                                                                                     |
| Aggregate rows            | Aggregate          | Aggregates rows into a single JSON array       | Extract all rows             | Valid Query column?           |                                                                                                                     |
| Valid Query column?       | IF                 | Validates presence of "Query" column           | Aggregate rows               | Split Out, Error message      |                                                                                                                     |
| Error message             | Form               | Returns error if "Query" column missing         | Valid Query column? (false)  | —                            |                                                                                                                     |
| Split Out                 | Split Out          | Extracts individual rows from aggregated data  | Valid Query column? (true)   | Isolate the Query column      |                                                                                                                     |
| Isolate the Query column  | Set                | Selects only the "Query" field                  | Split Out                   | Loop Over Queries             |                                                                                                                     |
| Loop Over Queries         | Split In Batches   | Iterates over queries one by one                 | Isolate the Query column     | Lookio API call, Generate enriched CSV |                                                                                                                     |
| Lookio API call           | HTTP Request       | Sends query to Lookio API and receives response | Loop Over Queries            | Prepare output                | ## Lookio for knowledge retrieval<br>- Add your [Lookio](https://www.lookio.app/) API key<br>- Specify the ID of the Lookio assistant to query |
| Prepare output            | Set                | Combines Query and Lookio API response          | Lookio API call              | Loop Over Queries             |                                                                                                                     |
| Generate enriched CSV     | Convert To File    | Converts JSON data to enriched CSV file          | Loop Over Queries            | Form ending and file download |                                                                                                                     |
| Form ending and file download | Form           | Returns enriched CSV file to user for download  | Generate enriched CSV        | —                            |                                                                                                                     |
| Sticky Note               | Sticky Note        | Documentation note with workflow overview & setup | —                            | —                            | # CSV bulk RAG queries with Lookio<br>Upload a CSV with a column named **Query** and get back a CSV with new **Response** column populated by Lookio.<br>Setup instructions included. |
| Sticky Note1              | Sticky Note        | Reminder to configure Lookio API key and assistant ID | —                            | —                            | ## Lookio for knowledge retrieval<br>- Add your [Lookio](https://www.lookio.app/) API key<br>- Specify the ID of the Lookio assistant to query |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the form trigger node ("On form submission"):**  
   - Type: Form Trigger  
   - Title: "Bulk Lookio queries"  
   - Add one required file input field: label "CSV", accept only `.csv` files, single file upload  
   - Set response to redirect after submission  
   - Add a form description: "Import a CSV with a column named 'Query' to have your Lookio assistant populate answers in a new 'Response' column."  
   - Save and expose webhook URL if needed.

2. **Add "Extract all rows" node:**  
   - Type: Extract From File  
   - Set binary property name to "CSV" (matches form file field)  
   - Connect output of form trigger to this node.

3. **Add "Aggregate rows" node:**  
   - Type: Aggregate  
   - Set aggregation mode to "aggregate all item data"  
   - Connect output of "Extract all rows" to this node.

4. **Add "Valid Query column?" IF node:**  
   - Type: IF  
   - Condition: check if `$json.data[0].Query` exists and is non-empty (case-sensitive)  
   - Connect output of "Aggregate rows" to this node.

5. **Add "Error message" form node:**  
   - Type: Form  
   - Operation: Completion  
   - Completion title: "Error"  
   - Completion message: `No "query" column has been found`  
   - Connect IF node's false branch to this node.

6. **Add "Split Out" node:**  
   - Type: Split Out  
   - Field to split out: "data"  
   - Connect IF node's true branch to this node.

7. **Add "Isolate the Query column" node:**  
   - Type: Set  
   - Keep only the field "Query" from `$json.Query`  
   - Connect output of "Split Out" to this node.

8. **Add "Loop Over Queries" node:**  
   - Type: Split In Batches  
   - Default batch size (1 is default)  
   - Connect output of "Isolate the Query column" to this node.

9. **Add "Lookio API call" node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.lookio.app/webhook/query`  
   - Body parameters:  
     - `query`: expression `$json.Query`  
     - `assistant_id`: your Lookio assistant ID (replace placeholder)  
     - `query_mode`: "flash"  
   - Header parameters:  
     - `api_key`: your Lookio API key (replace placeholder)  
   - Connect output of "Loop Over Queries" to this node.

10. **Add "Prepare output" node:**  
    - Type: Set  
    - Assign:  
      - "Query" = expression `$('Loop Over Queries').item.json.Query`  
      - "Response" = expression `$json.Output` (from Lookio API response)  
    - Connect output of "Lookio API call" to this node.

11. **Connect output of "Prepare output" back to "Loop Over Queries" node:**  
    - This creates the iteration loop to accumulate prepared output.

12. **Add "Generate enriched CSV" node:**  
    - Type: Convert To File  
    - File name: "Lookio bulk result.csv"  
    - Connect output of "Loop Over Queries" (second output) to this node.

13. **Add "Form ending and file download" node:**  
    - Type: Form  
    - Operation: Completion  
    - Respond with: Return binary file  
    - Completion title: "Your enriched file is ready"  
    - Completion message: `A new column "Response" has been added and contains the result from your Lookio Assistant`  
    - Connect output of "Generate enriched CSV" to this node.

14. **Final Connections:**  
    - Form trigger → Extract all rows → Aggregate rows → Valid Query column?  
    - Valid Query column? (true) → Split Out → Isolate the Query column → Loop Over Queries  
    - Loop Over Queries → Lookio API call → Prepare output → Loop Over Queries (loop back)  
    - Loop Over Queries (second output) → Generate enriched CSV → Form ending and file download  
    - Valid Query column? (false) → Error message

15. **Credential Setup:**  
    - Configure HTTP Request node with proper authentication if needed (none specified beyond API key header)  
    - Replace `<your-lookio-api-key>` and `<your-assistant-id>` with actual values from your Lookio account.

16. **Testing:**  
    - Activate the workflow  
    - Submit a test CSV via the form with at least one "Query" column  
    - Verify that the enriched CSV downloads with responses from Lookio

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                        | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Upload a CSV with a column named **Query** and get back a CSV with a new **Response** column populated by your Lookio assistant. Setup steps include creating a Lookio assistant, replacing API key and assistant ID. | Main sticky note in workflow documentation                    |
| Add your [Lookio](https://www.lookio.app/) API key and specify the Lookio assistant ID in the HTTP Request node headers and body parameters.                                                                        | Sticky Note1 on Lookio API call node                           |
| Lookio documentation and assistant creation: https://www.lookio.app/                                                                                                                                                 | Official Lookio website                                        |
| This workflow template was created by Guillaume Duvernay.                                                                                                                                                            | Workflow author credit                                        |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.