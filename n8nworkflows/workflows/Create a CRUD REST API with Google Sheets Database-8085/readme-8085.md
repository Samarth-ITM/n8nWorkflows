Create a CRUD REST API with Google Sheets Database

https://n8nworkflows.xyz/workflows/create-a-crud-rest-api-with-google-sheets-database-8085


# Create a CRUD REST API with Google Sheets Database

### 1. Workflow Overview

This workflow implements a **simple REST API backend using n8n** with **Google Sheets as a no-code database**. It supports all fundamental CRUD (Create, Read, Update, Delete) operations on records stored in a Google Sheet, enabling easy integration for small applications, prototypes, or internal tools without writing custom backend code.

The workflow is logically divided into the following blocks:

- **1.1 API Endpoint Reception:** Multiple webhook nodes listen for HTTP methods (POST, GET, PUT, DELETE) on specified API paths, serving as REST API endpoints.
- **1.2 Create Operation:** Appends new rows to the Google Sheet when receiving POST requests.
- **1.3 Read Operations:** Retrieves either all rows or a specific row from the Google Sheet based on GET requests.
- **1.4 Update Operation:** Updates existing rows in the Google Sheet using PUT requests.
- **1.5 Delete Operation:** Deletes rows from the Google Sheet on DELETE requests.
- **1.6 Response Handling:** Each operation ends with a node that sends an appropriate JSON response back to the API caller.
- **1.7 Helper Logic:** A node prepares data for the update operation to merge query parameters and request body fields into a single update object.
- **1.8 Documentation Sticky Note:** Contains detailed usage instructions, example curl commands, and setup guidelines.

---

### 2. Block-by-Block Analysis

---

#### 1.1 API Endpoint Reception

**Overview:**  
These webhook nodes act as entry points for the REST API, each configured for specific HTTP methods and endpoints corresponding to CRUD operations.

**Nodes Involved:**  
- Webhook: Create  
- Webhook: Read All  
- Webhook: Read  
- Webhook: Update  
- Webhook: Delete  

**Node Details:**

- **Webhook: Create**  
  - Type: Webhook  
  - Role: Listens for HTTP POST requests at `/items` to create new records.  
  - HTTP Method: POST  
  - Response Mode: Uses a separate response node.  
  - Input: Receives JSON request body with fields `name`, `email`, `status`.  
  - Output: Connected to "Append row in sheet".  
  - Edge cases: Missing or malformed JSON body; invalid field types; authentication issues are minimal since it's a public webhook unless protected externally.

- **Webhook: Read All**  
  - Type: Webhook  
  - Role: Listens for HTTP GET requests at `/items/all` to fetch all records.  
  - HTTP Method: GET  
  - Response Mode: Uses a separate response node.  
  - Output: Connected to "Get rows in sheet".  
  - Edge cases: Large data sets may cause performance issues; no query parameters expected.

- **Webhook: Read**  
  - Type: Webhook  
  - Role: Listens for HTTP GET requests at `/items` with query parameter `id` to fetch a single record.  
  - HTTP Method: GET  
  - Response Mode: Uses a separate response node.  
  - Input: Expects `id` query parameter indicating row number.  
  - Output: Connected to "Get row in sheet".  
  - Edge cases: Missing or invalid `id` query param; row not found.

- **Webhook: Update**  
  - Type: Webhook  
  - Role: Listens for HTTP PUT requests at `/items` with query `id` and JSON body to update a record.  
  - HTTP Method: PUT  
  - Response Mode: Uses a separate response node.  
  - Input: `id` query parameter for row to update; JSON body with fields to modify.  
  - Output: Connected to "Prepare Fields for Update".  
  - Edge cases: Missing or invalid `id`; empty or invalid update body.

- **Webhook: Delete**  
  - Type: Webhook  
  - Role: Listens for HTTP DELETE requests at `/items` with query parameter `id` to delete a row.  
  - HTTP Method: DELETE  
  - Response Mode: Uses a separate response node.  
  - Input: `id` query parameter for row to delete.  
  - Output: Connected to "Delete rows or columns from sheet".  
  - Edge cases: Missing or invalid `id`; attempting to delete non-existent row.

---

#### 1.2 Create Operation

**Overview:**  
Appends a new row to the Google Sheet database using fields received in the POST request.

**Nodes Involved:**  
- Append row in sheet  
- Respond to Webhook: Create  

**Node Details:**

- **Append row in sheet**  
  - Type: Google Sheets node  
  - Role: Appends a new row with values from the webhook's JSON body (`name`, `email`, `status`).  
  - Configuration:  
    - Operation: Append  
    - Sheet: "Sheet1" (gid=0) in Google Sheet with ID `1bQyl8pGVutkq1LRwK_-6TAAcXwNj4_TipeWHi-qmK1Q`  
    - Columns explicitly mapped with expressions: e.g. `={{ $json.body.name }}`  
    - Authentication: Google Service Account credential  
  - Inputs: Receives webhook POST data.  
  - Outputs: Connects to response node.  
  - Edge cases: API quota limits; malformed input data; Google Sheets service errors.

- **Respond to Webhook: Create**  
  - Type: Respond to Webhook  
  - Role: Sends JSON success message after row appended.  
  - Configuration: Responds with JSON `{ "status": "success", "message": "Record created." }`.  
  - Inputs: Output from append node.  
  - Outputs: HTTP response to client.

---

#### 1.3 Read Operations

**Overview:**  
Fetches records from the Google Sheet: either all rows or a single row by ID.

**Nodes Involved:**  
- Get rows in sheet  
- Get row in sheet  
- Respond to Webhook: Read All  
- Respond to Webhook: Read  

**Node Details:**

- **Get rows in sheet**  
  - Type: Google Sheets node  
  - Role: Retrieves all rows from the sheet.  
  - Configuration:  
    - Operation: Get All Rows (default)  
    - Sheet and document as above  
    - Authentication: Google Service Account  
  - Input: Triggered by Read All webhook.  
  - Output: Connects to "Respond to Webhook: Read All".  
  - Edge cases: Large datasets may cause performance issues or timeouts.

- **Get row in sheet**  
  - Type: Google Sheets node  
  - Role: Retrieves a single row filtered by row number (from query `id`).  
  - Configuration:  
    - Operation: Get Row with filter on column `row_number` equals `={{ $json.query.id }}`  
    - Sheet and document as above  
    - Authentication: Google Service Account  
  - Input: Triggered by Read webhook.  
  - Output: Connects to "Respond to Webhook: Read".  
  - Edge cases: Invalid or missing `id`; row not found.

- **Respond to Webhook: Read All**  
  - Type: Respond to Webhook  
  - Role: Returns all rows fetched as JSON.  
  - Configuration: Responds with all incoming items.  
  - Input: Output from "Get rows in sheet".  
  - Output: HTTP response.

- **Respond to Webhook: Read**  
  - Type: Respond to Webhook  
  - Role: Returns the single row fetched as JSON.  
  - Configuration: Default response options.  
  - Input: Output from "Get row in sheet".  
  - Output: HTTP response.

---

#### 1.4 Update Operation

**Overview:**  
Updates an existing record in the Google Sheet based on the row ID and supplied fields in the PUT request body.

**Nodes Involved:**  
- Prepare Fields for Update  
- Update row in sheet  
- Respond to Webhook: Update  

**Node Details:**

- **Prepare Fields for Update**  
  - Type: Set node  
  - Role: Constructs a JSON object combining the row number (from query `id`) and request body fields to prepare for update.  
  - Configuration:  
    - Mode: Raw JSON  
    - Expression:  
      ```js
      {
        "row_number": {{ $json.query.id }}
        {{ $if($json.body.keys().length > 0, ', ' + $json.body.toJsonString().replace('{', '').replace('}', ''), '') }}
      }
      ```  
    - This dynamically includes all fields from the body merged with row_number.  
  - Input: From Update webhook node.  
  - Output: To "Update row in sheet".  
  - Edge cases: Empty bodies; invalid JSON; missing `id`.

- **Update row in sheet**  
  - Type: Google Sheets node  
  - Role: Updates the specified row with new data.  
  - Configuration:  
    - Operation: Update  
    - Matching column: `row_number`  
    - Columns: name, email, status, row_number mapped automatically from input data  
    - Sheet and document as above  
    - Authentication: Google Service Account  
  - Input: Prepared update data.  
  - Output: To "Respond to Webhook: Update".  
  - Edge cases: Invalid row ID; Google API errors.

- **Respond to Webhook: Update**  
  - Type: Respond to Webhook  
  - Role: Sends JSON confirmation of successful update.  
  - Configuration: JSON `{ "status": "success", "message": "Record updated." }`.  
  - Input: From update node.  
  - Output: HTTP response.

---

#### 1.5 Delete Operation

**Overview:**  
Deletes a row from the Google Sheet based on the given row ID.

**Nodes Involved:**  
- Delete rows or columns from sheet  
- Respond to Webhook: Delete  

**Node Details:**

- **Delete rows or columns from sheet**  
  - Type: Google Sheets node  
  - Role: Deletes a row by index.  
  - Configuration:  
    - Operation: Delete  
    - Sheet and document as above  
    - `startIndex` set to `={{ $json.query.id }}` (row to delete)  
    - Authentication: Google Service Account  
  - Input: From Delete webhook.  
  - Output: To "Respond to Webhook: Delete".  
  - Edge cases: Invalid or missing `id`; deleting rows that might shift indexes for subsequent operations.

- **Respond to Webhook: Delete**  
  - Type: Respond to Webhook  
  - Role: Sends JSON confirmation of deletion.  
  - Configuration: JSON `{ "status": "success", "message": "Record deleted." }`.  
  - Input: From delete node.  
  - Output: HTTP response.

---

#### 1.6 Documentation Sticky Note

**Overview:**  
Contains detailed explanations, setup instructions, example API calls, and a link to an external blog post for further reading.

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note (non-functional, documentation only)  
  - Content includes:  
    - Explanation of the workflow purpose  
    - Setup instructions with Google Sheet preparation and credential setup  
    - Example curl commands for each CRUD operation  
    - Link to blog post: [Build a Simple REST API in 10 Minutes with n8n & Google Sheets](https://n8nplaybook.com/post/2025/08/n8n-google-sheets-rest-api/)  
  - Positioned off to the side for reference.

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                     | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                                                                                                                                      |
|------------------------------|----------------------------|-----------------------------------|------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook: Create               | Webhook                    | API entry for creating records    | -                      | Append row in sheet            | This workflow template demonstrates how to quickly and easily create a simple REST API using n8n and a Google Sheet as a no-code database.                                                                        |
| Append row in sheet           | Google Sheets              | Append new row to sheet           | Webhook: Create         | Respond to Webhook: Create     |                                                                                                                                                                                                                  |
| Respond to Webhook: Create    | Respond to Webhook         | Respond success for create        | Append row in sheet     | -                             |                                                                                                                                                                                                                  |
| Webhook: Read All             | Webhook                    | API entry for reading all records | -                      | Get rows in sheet             |                                                                                                                                                                                                                  |
| Get rows in sheet             | Google Sheets              | Retrieve all rows                 | Webhook: Read All       | Respond to Webhook: Read All   |                                                                                                                                                                                                                  |
| Respond to Webhook: Read All  | Respond to Webhook         | Respond with all records          | Get rows in sheet       | -                             |                                                                                                                                                                                                                  |
| Webhook: Read                | Webhook                    | API entry for reading one record  | -                      | Get row in sheet              |                                                                                                                                                                                                                  |
| Get row in sheet              | Google Sheets              | Retrieve single row by id          | Webhook: Read           | Respond to Webhook: Read       |                                                                                                                                                                                                                  |
| Respond to Webhook: Read      | Respond to Webhook         | Respond with single record        | Get row in sheet        | -                             |                                                                                                                                                                                                                  |
| Webhook: Update              | Webhook                    | API entry for updating records    | -                      | Prepare Fields for Update      |                                                                                                                                                                                                                  |
| Prepare Fields for Update     | Set                        | Prepare update data object        | Webhook: Update         | Update row in sheet            |                                                                                                                                                                                                                  |
| Update row in sheet           | Google Sheets              | Update row in sheet               | Prepare Fields for Update | Respond to Webhook: Update   |                                                                                                                                                                                                                  |
| Respond to Webhook: Update    | Respond to Webhook         | Respond success for update        | Update row in sheet     | -                             |                                                                                                                                                                                                                  |
| Webhook: Delete              | Webhook                    | API entry for deleting records    | -                      | Delete rows or columns from sheet |                                                                                                                                                                                                                  |
| Delete rows or columns from sheet | Google Sheets          | Delete row by index               | Webhook: Delete         | Respond to Webhook: Delete     |                                                                                                                                                                                                                  |
| Respond to Webhook: Delete    | Respond to Webhook         | Respond success for delete        | Delete rows or columns from sheet | -                       |                                                                                                                                                                                                                  |
| Sticky Note                  | Sticky Note                | Documentation and instructions    | -                      | -                             | This workflow template demonstrates how to quickly and easily create a simple REST API using n8n and a Google Sheet as a no-code database. Usage instructions, example curl calls, and blog post link included. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node for Create (POST /items):**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `items`  
   - Response Mode: Response Node  
   - Leave other options default.

2. **Create Google Sheets Node to Append Row:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your Google Sheet ID (e.g., `1bQyl8pGVutkq1LRwK_-6TAAcXwNj4_TipeWHi-qmK1Q`)  
   - Sheet Name: `Sheet1` (or your sheet name, gid=0)  
   - Authentication: Connect Google Service Account credentials  
   - Columns: Define `name`, `email`, `status` columns with expressions:  
     - name: `={{ $json.body.name }}`  
     - email: `={{ $json.body.email }}`  
     - status: `={{ $json.body.status }}`  

3. **Create Respond to Webhook Node for Create Response:**  
   - Type: Respond to Webhook  
   - Respond With: JSON  
   - Response Body: `{ "status": "success", "message": "Record created." }`  

4. **Connect Webhook Create → Append row in sheet → Respond to Webhook Create**

5. **Create Webhook Node for Read All (GET /items/all):**  
   - Type: Webhook  
   - HTTP Method: GET  
   - Path: `items/all`  
   - Response Mode: Response Node  

6. **Create Google Sheets Node to Get All Rows:**  
   - Type: Google Sheets  
   - Operation: Get All Rows  
   - Document ID and Sheet Name as above  
   - Authentication: Service Account  

7. **Create Respond to Webhook Node for Read All Response:**  
   - Type: Respond to Webhook  
   - Respond With: All Incoming Items  

8. **Connect Webhook Read All → Get rows in sheet → Respond to Webhook Read All**

9. **Create Webhook Node for Read Single (GET /items?id=):**  
   - Type: Webhook  
   - HTTP Method: GET  
   - Path: `items`  
   - Response Mode: Response Node  

10. **Create Google Sheets Node to Get Row by Filter:**  
    - Type: Google Sheets  
    - Operation: Get Row  
    - Filter Rows: Lookup column `row_number` equals `={{ $json.query.id }}`  
    - Document ID, Sheet Name, and Authentication as above  

11. **Create Respond to Webhook Node for Read Single Response:**  
    - Type: Respond to Webhook  
    - Default response settings  

12. **Connect Webhook Read → Get row in sheet → Respond to Webhook Read**

13. **Create Webhook Node for Update (PUT /items?id=):**  
    - Type: Webhook  
    - HTTP Method: PUT  
    - Path: `items`  
    - Response Mode: Response Node  

14. **Create Set Node to Prepare Update Fields:**  
    - Type: Set  
    - Mode: Raw JSON  
    - Expression:  
      ```js
      {
        "row_number": {{ $json.query.id }}
        {{ $if($json.body.keys().length > 0, ', ' + $json.body.toJsonString().replace('{', '').replace('}', ''), '') }}
      }
      ```  
    - This merges the query ID with the request body fields for update.

15. **Create Google Sheets Node to Update Row:**  
    - Type: Google Sheets  
    - Operation: Update  
    - Matching Columns: `row_number`  
    - Columns: name, email, status, row_number (auto-mapped)  
    - Document ID, Sheet Name, Authentication as above  

16. **Create Respond to Webhook Node for Update Response:**  
    - Type: Respond to Webhook  
    - Respond With: JSON  
    - Response Body: `{ "status": "success", "message": "Record updated." }`  

17. **Connect Webhook Update → Prepare Fields for Update → Update row in sheet → Respond to Webhook Update**

18. **Create Webhook Node for Delete (DELETE /items?id=):**  
    - Type: Webhook  
    - HTTP Method: DELETE  
    - Path: `items`  
    - Response Mode: Response Node  

19. **Create Google Sheets Node to Delete Row:**  
    - Type: Google Sheets  
    - Operation: Delete  
    - Sheet and Document as above  
    - Start Index: `={{ $json.query.id }}`  
    - Authentication: Service Account  

20. **Create Respond to Webhook Node for Delete Response:**  
    - Type: Respond to Webhook  
    - Respond With: JSON  
    - Response Body: `{ "status": "success", "message": "Record deleted." }`  

21. **Connect Webhook Delete → Delete rows or columns from sheet → Respond to Webhook Delete**

22. **Set Credentials:**  
    - Create or use an existing Google Service Account credential in n8n with appropriate scopes for Google Sheets API access.  
    - Assign this credential in all Google Sheets nodes.

23. **Test the workflow:**  
    - Activate the workflow.  
    - Use `curl`, Postman, or similar tools with the example commands in the Sticky Note to verify operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                              | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow template demonstrates how to quickly and easily create a simple REST API using n8n and a Google Sheet as a no-code database. It handles all CRUD operations with a single Google Sheet backend.              | Included in Sticky Note node content                                                                               |
| Setup requires creating a Google Sheet with columns `name`, `email`, and `status`, and configuring Google Service Account credentials for n8n.                                                                           | See Sticky Note instructions                                                                                        |
| Example curl commands for each CRUD operation are provided to test the API endpoints.                                                                                                                                    | See Sticky Note content                                                                                            |
| For detailed instructions and extended explanation, see the blog post "Build a Simple REST API in 10 Minutes with n8n & Google Sheets".                                                                                   | https://n8nplaybook.com/post/2025/08/n8n-google-sheets-rest-api/                                                  |
| Google Sheets API quota limits and API errors may affect large datasets or frequent requests. Consider pagination or rate limiting for production use.                                                                    | General best practice                                                                                              |
| Row indexing in Google Sheets starts at 0 for deletion; ensure correct `id` parameter is passed to avoid deleting wrong rows.                                                                                             | Important for Delete operation                                                                                      |
| The update operation merges query parameter `id` and JSON body fields dynamically; malformed JSON or missing `id` will cause failures.                                                                                   | Edge case warning                                                                                                  |
| No authentication is built into the webhooks; protect your n8n instance or secure webhooks to prevent unauthorized access.                                                                                               | Security note                                                                                                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.