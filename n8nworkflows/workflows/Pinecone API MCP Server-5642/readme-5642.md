Pinecone API MCP Server

https://n8nworkflows.xyz/workflows/pinecone-api-mcp-server-5642


# Pinecone API MCP Server

### 1. Workflow Overview

This workflow, titled **"Pinecone API MCP Server"**, serves as an integrated server to manage Pinecone vector database operations via an n8n MCP (Multi-Channel Platform) trigger. It is designed for use cases involving vector database management including index and collection lifecycle management, vector data operations, and statistics retrieval. The workflow acts as an API server endpoint that receives MCP-triggered requests and performs corresponding Pinecone API HTTP calls.

The workflow logic is organized into the following functional blocks:

- **1.1 MCP Trigger Reception:** Entry point receiving MCP API calls.
- **1.2 Pinecone API Operations:** Multiple HTTP Request nodes handling discrete Pinecone API operations, grouped by function:
  - Collection management (list, create, delete, describe)
  - Index management (list, create, delete, describe, configure)
  - Vector data operations (fetch, upsert, update, delete)
  - Query execution and statistics retrieval
- **1.3 Documentation & Instructions:** Sticky notes providing setup instructions and workflow overview.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger Reception

**Overview:**  
This block is the workflow's entry point, waiting for MCP requests to trigger vector database operations.

**Nodes Involved:**  
- Pinecone MCP Server

**Node Details:**  
- **Name:** Pinecone MCP Server  
- **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP Trigger)  
- **Technical Role:** Listens for incoming MCP API calls and triggers workflow execution accordingly.  
- **Configuration:** Default settings; webhook ID assigned to receive external requests.  
- **Input:** External MCP API calls via webhook.  
- **Output:** Routes execution flow to HTTP request nodes to perform the requested Pinecone API operation.  
- **Version-specific requirements:** Requires n8n environment with MCP trigger support and proper webhook exposure.  
- **Potential Failures:** Webhook not reachable, invalid MCP request payloads, authentication failures if credentials are missing downstream.  
- **Sub-workflow Reference:** This node acts as the main trigger; no sub-workflow invoked.

---

#### 1.2 Pinecone API Operations

**Overview:**  
This block contains multiple HTTP Request nodes, each configured to perform a specific Pinecone API call as requested by the MCP trigger. These nodes interact with Pinecone’s REST API to manage collections, indexes, vectors, queries, and statistics.

**Nodes Involved:**  
- List Collections  
- Create Collection 1  
- Delete Collection 1  
- Describe Collection  
- List Indexes  
- Create Index  
- Delete Index  
- Describe Index  
- Configure Index  
- Retrieve Index Stats  
- Execute Query  
- Delete Vectors  
- Fetch Vectors  
- Update Vectors  
- Upsert Vectors

**Node Details:**  

- **List Collections**  
  - Type: HTTP Request  
  - Role: Retrieves all collections in Pinecone environment.  
  - Config: HTTP GET, endpoint `/collections`, uses Pinecone API credentials.  
  - Input: Triggered from MCP Server node.  
  - Output: JSON list of collections.  
  - Edge cases: API rate limits, network issues.

- **Create Collection 1**  
  - Type: HTTP Request  
  - Role: Creates a new collection.  
  - Config: HTTP POST, endpoint `/collections`, body with collection parameters.  
  - Input: Triggered from MCP Server node.  
  - Output: Confirmation of creation.  
  - Edge cases: Duplicate name errors, invalid parameters.

- **Delete Collection 1**  
  - Type: HTTP Request  
  - Role: Deletes specified collection.  
  - Config: HTTP DELETE, endpoint `/collections/{name}`.  
  - Input: Triggered from MCP Server node.  
  - Output: Deletion confirmation.  
  - Edge cases: Trying to delete non-existent collection, permission errors.

- **Describe Collection**  
  - Type: HTTP Request  
  - Role: Retrieves metadata about a collection.  
  - Config: HTTP GET, endpoint `/collections/{name}`.  
  - Input: Triggered from MCP Server node.  
  - Output: Collection details.  
  - Edge cases: Collection not found.

- **List Indexes**  
  - Type: HTTP Request  
  - Role: Lists all indexes.  
  - Config: HTTP GET, endpoint `/databases`.  
  - Input: Triggered from MCP Server node.  
  - Output: List of indexes.  
  - Edge cases: API errors, network latency.

- **Create Index**  
  - Type: HTTP Request  
  - Role: Creates a new index.  
  - Config: HTTP POST, endpoint `/databases`, with index config in body.  
  - Input: Triggered from MCP Server node.  
  - Output: Confirmation and index details.  
  - Edge cases: Invalid config, duplicate index errors.

- **Delete Index**  
  - Type: HTTP Request  
  - Role: Deletes an index.  
  - Config: HTTP DELETE, endpoint `/databases/{name}`.  
  - Input: Triggered from MCP Server node.  
  - Output: Deletion success.  
  - Edge cases: Index not found, permission issues.

- **Describe Index**  
  - Type: HTTP Request  
  - Role: Gets details about an index.  
  - Config: HTTP GET, endpoint `/databases/{name}`.  
  - Input: Triggered from MCP Server node.  
  - Output: Index metadata.  
  - Edge cases: Index missing.

- **Configure Index**  
  - Type: HTTP Request  
  - Role: Updates index configuration.  
  - Config: HTTP PATCH or PUT, endpoint `/databases/{name}`, body with updated config.  
  - Input: Triggered from MCP Server node.  
  - Output: Updated index confirmation.  
  - Edge cases: Invalid config, conflicts.

- **Retrieve Index Stats**  
  - Type: HTTP Request  
  - Role: Fetches usage and performance statistics of an index.  
  - Config: HTTP GET, endpoint `/databases/{name}/stats`.  
  - Input: Triggered from MCP Server node.  
  - Output: Stats JSON.  
  - Edge cases: Index offline or unreachable.

- **Execute Query**  
  - Type: HTTP Request  
  - Role: Performs vector similarity query.  
  - Config: HTTP POST, endpoint `/query`, body with query vector and params.  
  - Input: Triggered from MCP Server node.  
  - Output: Query results.  
  - Edge cases: Invalid query vectors, timeout.

- **Delete Vectors**  
  - Type: HTTP Request  
  - Role: Deletes vector(s) from an index.  
  - Config: HTTP POST or DELETE, endpoint `/vectors/delete`, body with vector IDs.  
  - Input: Triggered from MCP Server node.  
  - Output: Deletion confirmation.  
  - Edge cases: Vector IDs not found.

- **Fetch Vectors**  
  - Type: HTTP Request  
  - Role: Retrieves vector data by IDs.  
  - Config: HTTP POST, endpoint `/vectors/fetch`, body with vector IDs.  
  - Input: Triggered from MCP Server node.  
  - Output: Vectors data.  
  - Edge cases: Missing vectors, partial fetch.

- **Update Vectors**  
  - Type: HTTP Request  
  - Role: Updates existing vectors.  
  - Config: HTTP POST or PATCH, endpoint `/vectors/update`, with updated vector data.  
  - Input: Triggered from MCP Server node.  
  - Output: Update confirmation.  
  - Edge cases: Vector not found, invalid data format.

- **Upsert Vectors**  
  - Type: HTTP Request  
  - Role: Inserts or updates vectors in batch.  
  - Config: HTTP POST, endpoint `/vectors/upsert`, with vector batch data.  
  - Input: Triggered from MCP Server node.  
  - Output: Upsert results.  
  - Edge cases: Partial success, batch size limits.

---

#### 1.3 Documentation & Instructions

**Overview:**  
Sticky notes are used to provide setup instructions and a brief overview within the n8n editor interface.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)  
- Sticky Note (generic)  
- Sticky Note2 (generic)

**Node Details:**  
- Type: Sticky Note  
- Role: Documentation placeholders for users to add setup steps, workflow explanations, or comments.  
- Configuration: Empty content in current version; intended for manual editing.  
- Input/Output: None; informational only.  
- Edge cases: None.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                     | Input Node(s)         | Output Node(s)                   | Sticky Note                     |
|---------------------|----------------------------------|-----------------------------------|-----------------------|---------------------------------|--------------------------------|
| Setup Instructions   | Sticky Note                      | Workflow documentation             | -                     | -                               |                                |
| Workflow Overview    | Sticky Note                      | Workflow documentation             | -                     | -                               |                                |
| Pinecone MCP Server  | MCP Trigger                     | Entry point for MCP API requests  | -                     | All HTTP Request nodes          |                                |
| Sticky Note         | Sticky Note                      | Documentation                     | -                     | -                               |                                |
| List Collections     | HTTP Request                    | Retrieve Pinecone collections     | Pinecone MCP Server    | -                               |                                |
| Create Collection 1  | HTTP Request                    | Create new collection             | Pinecone MCP Server    | -                               |                                |
| Delete Collection 1  | HTTP Request                    | Delete existing collection        | Pinecone MCP Server    | -                               |                                |
| Describe Collection  | HTTP Request                    | Get info on specific collection   | Pinecone MCP Server    | -                               |                                |
| List Indexes         | HTTP Request                    | List all indexes                  | Pinecone MCP Server    | -                               |                                |
| Create Index         | HTTP Request                    | Create new index                  | Pinecone MCP Server    | -                               |                                |
| Delete Index         | HTTP Request                    | Delete index                     | Pinecone MCP Server    | -                               |                                |
| Describe Index       | HTTP Request                    | Get index details                | Pinecone MCP Server    | -                               |                                |
| Configure Index      | HTTP Request                    | Update index configuration       | Pinecone MCP Server    | -                               |                                |
| Sticky Note2         | Sticky Note                      | Documentation                     | -                     | -                               |                                |
| Retrieve Index Stats | HTTP Request                    | Fetch index statistics           | Pinecone MCP Server    | -                               |                                |
| Execute Query        | HTTP Request                    | Run vector similarity queries    | Pinecone MCP Server    | -                               |                                |
| Delete Vectors       | HTTP Request                    | Remove vectors from index        | Pinecone MCP Server    | -                               |                                |
| Fetch Vectors        | HTTP Request                    | Retrieve vectors by ID           | Pinecone MCP Server    | -                               |                                |
| Update Vectors       | HTTP Request                    | Update existing vectors          | Pinecone MCP Server    | -                               |                                |
| Upsert Vectors       | HTTP Request                    | Insert or update vectors batch  | Pinecone MCP Server    | -                               |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set the timezone to America/New_York.

2. **Add MCP Trigger Node:**  
   - Node Type: `MCP Trigger` (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
   - Name it "Pinecone MCP Server".  
   - Leave default parameters.  
   - Copy the generated webhook URL for external calls.

3. **Add HTTP Request Nodes for Pinecone API Operations:** For each of the following nodes, do the following:

   - Node Type: HTTP Request  
   - Set HTTP Method and URL endpoint according to Pinecone API documentation (e.g., `/collections`, `/databases`, `/vectors/upsert` etc.).  
   - Use Pinecone API credentials (API key, environment URL) in the node’s authentication section (usually as HTTP Header Authorization Bearer token).  
   - Configure request body or query parameters as needed (JSON format).  
   - Name the node as specified below:

   The nodes to create (with typical configurations):

   - **List Collections:** GET `/collections`  
   - **Create Collection 1:** POST `/collections` with JSON body for collection details  
   - **Delete Collection 1:** DELETE `/collections/{collectionName}` (use expression to pass collection name)  
   - **Describe Collection:** GET `/collections/{collectionName}`  
   - **List Indexes:** GET `/databases`  
   - **Create Index:** POST `/databases` with JSON body for index config  
   - **Delete Index:** DELETE `/databases/{indexName}`  
   - **Describe Index:** GET `/databases/{indexName}`  
   - **Configure Index:** PATCH or PUT `/databases/{indexName}` with config body  
   - **Retrieve Index Stats:** GET `/databases/{indexName}/stats`  
   - **Execute Query:** POST `/query` with query vector payload  
   - **Delete Vectors:** POST or DELETE `/vectors/delete` with vector IDs in body  
   - **Fetch Vectors:** POST `/vectors/fetch` with vector IDs in body  
   - **Update Vectors:** POST or PATCH `/vectors/update` with updated vector data  
   - **Upsert Vectors:** POST `/vectors/upsert` with batch vector data  

4. **Connect each HTTP Request node’s input to the MCP Trigger node’s output** to handle requests routed by MCP.

5. **Add Sticky Notes as needed** for documentation inside the workflow editor:  
   - "Setup Instructions"  
   - "Workflow Overview"  
   - Additional generic sticky notes to separate logical blocks.

6. **Test each API node** by sending appropriate MCP requests to the webhook URL and verifying the Pinecone API responses.

7. **Ensure credentials are securely stored** in n8n credentials manager and linked to HTTP Request nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                  |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow is designed as a Pinecone vector database management API server using n8n MCP triggers. | Project description                                             |
| Refer to Pinecone API documentation for exact endpoint URLs and request body schemas:                | https://docs.pinecone.io/docs/overview                           |
| MCP Trigger node requires n8n environment with MCP support and webhook exposure.                      | n8n MCP documentation                                           |
| Use appropriate API key with Pinecone for authentication in HTTP Request nodes.                       | Pinecone API key management                                     |
| Sticky notes in the workflow serve as placeholders for adding setup instructions or workflow overview. | Editable in n8n workflow editor                                 |

---

**Disclaimer:** The provided text is exclusively generated from an n8n automated workflow export. It complies fully with content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.