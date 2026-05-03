Migrate Pinecone Index to Weaviate Class with Airtable Pagination

https://n8nworkflows.xyz/workflows/migrate-pinecone-index-to-weaviate-class-with-airtable-pagination-8445


# Migrate Pinecone Index to Weaviate Class with Airtable Pagination

### 1. Workflow Overview

This workflow automates the migration of vector data from a Pinecone index to a Weaviate class, using Airtable for pagination state management. It is designed for scenarios where large Pinecone vector indexes need to be incrementally transferred to Weaviate, handling API pagination limits and ensuring seamless continuation across executions.

The workflow is logically divided into the following blocks:

- **1.1 Parameter Initialization:** Sets up required parameters for the migration, including source and target endpoints, namespaces, and batch limits.
- **1.2 Pagination State Management:** Uses Airtable to store and retrieve the pagination token to handle vector retrieval in batches.
- **1.3 Vector Retrieval from Pinecone:** Queries Pinecone API for vectors, paginated by the stored token.
- **1.4 Vector Preparation and Formatting:** Processes raw vectors to extract IDs and prepares the data structure compliant with Weaviate’s object schema.
- **1.5 Vector Upload to Weaviate:** Sends formatted vectors to the Weaviate cluster via HTTP API.
- **1.6 Pagination Token Update and Iteration:** Updates the Airtable record with the new pagination token and loops back if more data is available.
- **1.7 Completion Handling:** Marks the migration as completed when no more pages are available.

---

### 2. Block-by-Block Analysis

#### 1.1 Parameter Initialization

- **Overview:** Defines all necessary configuration parameters for the migration process such as Pinecone source index URL, namespace, batch size limit, Weaviate target class, and cluster endpoint.
- **Nodes Involved:** `Parameters`

##### Node: Parameters
- **Type:** Set node
- **Role:** Holds static or user-defined inputs required for API calls and workflow logic.
- **Configuration:**
  - `sourceIndex`: URL endpoint of the Pinecone index.
  - `sourceNamespace`: Pinecone namespace to query vectors from.
  - `batchLimit`: Maximum number of vectors to retrieve per request (default 100).
  - `targetCollection`: Destination Weaviate class name.
  - `weaviateCluster`: Weaviate REST API endpoint URL.
- **Expressions:** Values are hardcoded strings for user replacement.
- **Connections:**
  - Output to `Get Next Page Token`
- **Edge Cases:** Missing or incorrect URLs/parameters will cause API call failures downstream.
- **Version:** n8n Set node v3.4

---

#### 1.2 Pagination State Management

- **Overview:** Manages the pagination token using Airtable to keep track of which batch of vectors to fetch next.
- **Nodes Involved:** `Get Next Page Token`, `Save Next Page Token`, `Save Next Page Token1`
  
##### Node: Get Next Page Token
- **Type:** Airtable node
- **Role:** Retrieves the current pagination token from Airtable, filtering for the record where `Number = 0`.
- **Configuration:**
  - Base and table IDs linked to an Airtable containing pagination tokens.
  - Operation: Search with filter `{Number} = 0`.
- **Input:** Receives parameters from `Parameters`.
- **Output:** Pagination token in field `Name`.
- **Edge Cases:** No matching record found or Airtable API errors would break pagination continuity.
- **Version:** Airtable v2.1

##### Node: Save Next Page Token
- **Type:** Airtable node
- **Role:** Updates Airtable with the new pagination token after fetching a batch.
- **Configuration:**
  - Upserts record where `Number = 0`.
  - Stores the pagination token in the `Name` field.
- **Input:** Receives pagination token from Pinecone API response.
- **Edge Cases:** Airtable API failures or schema mismatches can cause token loss.
- **Version:** Airtable v2.1

##### Node: Save Next Page Token1
- Similar configuration and role as `Save Next Page Token`, used after fetching subsequent pages.

---

#### 1.3 Vector Retrieval from Pinecone

- **Overview:** Retrieves vectors from Pinecone API in paginated batches, using the pagination token stored in Airtable.
- **Nodes Involved:** `First Iteration?`, `Get Record First Page`, `Get Record Next Page`

##### Node: First Iteration?
- **Type:** If node
- **Role:** Determines if this is the first iteration by checking if the pagination token equals "INIT".
- **Input:** Pagination token from Airtable.
- **Output:** 
  - True branch: triggers first page fetch.
  - False branch: triggers next page fetch.
- **Edge Cases:** Token value mismatch could cause incorrect branching.
- **Version:** If node v2.2

##### Node: Get Record First Page
- **Type:** HTTP Request node
- **Role:** Fetches the first batch of vectors from Pinecone without a pagination token.
- **Configuration:**
  - URL constructed from `sourceIndex` parameter + `/vectors/list`.
  - Query parameters: namespace, limit.
  - Authentication: Pinecone API credentials.
- **Output:** Pinecone response including vectors and next pagination token.
- **Edge Cases:** API rate limits, invalid credentials, or empty responses.

##### Node: Get Record Next Page
- **Type:** HTTP Request node
- **Role:** Fetches the next batch using pagination token.
- **Configuration:** Similar to `Get Record First Page` but includes `pagination_token` query param.
- **Edge Cases:** Invalid or expired pagination token may cause errors.

---

#### 1.4 Vector Preparation and Formatting

- **Overview:** Processes raw vector data from Pinecone, extracts IDs, and formats them for Weaviate ingestion.
- **Nodes Involved:** `Select Ids`, `Select Ids1`, `Prepare Fetch Body`, `Fetch Vectors`, `Format2Weaviate`

##### Node: Select Ids / Select Ids1
- **Type:** Set node
- **Role:** Extracts vectors array and namespace from Pinecone response JSON.
- **Input:** Pinecone API response.
- **Output:** Structured JSON with `vectors` array and `namespace` string.
- **Edge Cases:** Missing or malformed vectors array may cause downstream errors.

##### Node: Prepare Fetch Body
- **Type:** Code node (JavaScript)
- **Role:** Validates vectors array, extracts vector IDs, and constructs fetch body for detailed vector retrieval.
- **Key Logic:**
  - Ensures input is array or single object.
  - Throws errors if `vectors` field is missing or empty.
  - Returns object with `ids` array and `namespace`.
- **Input:** Output from `Select Ids` or `Select Ids1`.
- **Output:** JSON payload for vector fetch.
- **Edge Cases:** Throws exceptions on invalid input structure.

##### Node: Fetch Vectors
- **Type:** HTTP Request node
- **Role:** Calls Pinecone `/vectors/fetch` endpoint with vector IDs to get full vector data including values and metadata.
- **Configuration:**
  - URL from `sourceIndex` + `/vectors/fetch`.
  - Authenticated with Pinecone API credentials.
  - JSON body includes `ids`, `namespace`, `include_values:true`, `include_metadata:true`.
- **Edge Cases:** API failures, missing vectors, auth issues.

##### Node: Format2Weaviate
- **Type:** Code node (JavaScript)
- **Role:** Transforms Pinecone vector objects into Weaviate-compatible objects.
- **Key Logic:**
  - Maps each vector to a JSON object with:
    - `class`: target Weaviate class name.
    - `id`: vector ID.
    - `vector`: vector values array.
    - `properties`: selected metadata fields mapped explicitly.
- **Input:** Detailed vector data from `Fetch Vectors`.
- **Output:** Array of Weaviate object JSONs.
- **Edge Cases:** Missing metadata fields may result in undefined properties.

---

#### 1.5 Vector Upload to Weaviate

- **Overview:** Uploads the formatted vector objects to the Weaviate cluster.
- **Nodes Involved:** `LoadWeAviate`

##### Node: LoadWeAviate
- **Type:** HTTP Request node
- **Role:** Posts vector objects to Weaviate REST API `/v1/objects`.
- **Configuration:**
  - URL built from `weaviateCluster` parameter.
  - Method: POST.
  - Uses HTTP Bearer authentication credential.
  - Sends JSON body as received from `Format2Weaviate`.
- **Edge Cases:** Auth failures, API rate limiting, or schema validation errors.

---

#### 1.6 Pagination Token Update and Iteration

- **Overview:** Saves the new pagination token after vector upload and triggers the next iteration if more data exists.
- **Nodes Involved:** `Save Next Page Token`, `Save Next Page Token1`, connected to pagination management and vector retrieval nodes.

- **Edge Cases:** Failure to update Airtable token can halt migration or cause duplicates.

---

#### 1.7 Completion Handling

- **Overview:** Detects when no further pagination token exists and ends the migration process.
- **Nodes Involved:** `Is Next Pagination Token null?`, `Migration completed`

##### Node: Is Next Pagination Token null?
- **Type:** If node
- **Role:** Checks if the pagination token length is greater than zero to decide continuation.
- **Output:** 
  - True: continue iteration.
  - False: triggers migration completion.
- **Edge Cases:** Incorrect token evaluation may cause infinite loops or premature ending.

##### Node: Migration completed
- **Type:** NoOp node
- **Role:** Acts as a logical endpoint for the workflow.
- **Edge Cases:** None.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                     | Input Node(s)           | Output Node(s)             | Sticky Note                                      |
|------------------------|---------------------|-----------------------------------|------------------------|----------------------------|-------------------------------------------------|
| Parameters             | Set                 | Defines migration parameters       | Schedule Trigger       | Get Next Page Token         |                                                 |
| Schedule Trigger       | Schedule Trigger    | Triggers workflow periodically     |                        | Parameters                 |                                                 |
| Get Next Page Token    | Airtable            | Retrieves current pagination token | Parameters             | Is Next Pagination Token null? |                                                 |
| Is Next Pagination Token null? | If            | Checks if pagination token exists  | Get Next Page Token    | First Iteration?, Migration completed |                                                 |
| First Iteration?       | If                  | Branches initial vs subsequent fetch | Is Next Pagination Token null? | Get Record First Page, Get Record Next Page |                                                 |
| Get Record First Page  | HTTP Request        | Fetches first batch from Pinecone  | First Iteration? (true) | Save Next Page Token        |                                                 |
| Get Record Next Page   | HTTP Request        | Fetches next batch from Pinecone   | First Iteration? (false) | Save Next Page Token1       |                                                 |
| Save Next Page Token   | Airtable            | Updates pagination token (first)   | Get Record First Page  | Select Ids                 |                                                 |
| Save Next Page Token1  | Airtable            | Updates pagination token (next)    | Get Record Next Page   | Select Ids1                |                                                 |
| Select Ids             | Set                 | Extracts vectors and namespace     | Save Next Page Token   | Prepare Fetch Body         |                                                 |
| Select Ids1            | Set                 | Extracts vectors and namespace     | Save Next Page Token1  | Prepare Fetch Body         |                                                 |
| Prepare Fetch Body     | Code                | Validates & prepares fetch request | Select Ids, Select Ids1 | Fetch Vectors             |                                                 |
| Fetch Vectors          | HTTP Request        | Retrieves vector details from Pinecone | Prepare Fetch Body  | Format2Weaviate            |                                                 |
| Format2Weaviate        | Code                | Formats vectors for Weaviate upload | Fetch Vectors          | LoadWeAviate               |                                                 |
| LoadWeAviate           | HTTP Request        | Uploads vectors to Weaviate        | Format2Weaviate        |                            |                                                 |
| Migration completed    | NoOp                | Marks migration end                | Is Next Pagination Token null? (false) |                      |                                                 |
| Sticky Note            | Sticky Note         | Explains workflow purpose and parameters |                      |                            | ## Tool for Transferring an Index from Pinecone to Weaviate ... |
| Sticky Note2           | Sticky Note         | Labels iterations section          |                        |                            | ## Iterations                                    |
| Sticky Note3           | Sticky Note         | Labels migration completion        |                        |                            | ## Migration Completed                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Type: Schedule Trigger
   - Parameters: Interval set to every 15 seconds
   - Position: Starting point of the workflow

2. **Create a Set node named "Parameters"**
   - Type: Set
   - Parameters:
     - `sourceIndex`: URL of Pinecone index (string)
     - `sourceNamespace`: Pinecone namespace (string)
     - `batchLimit`: Number of vectors per batch (default "100")
     - `targetCollection`: Weaviate class name (string)
     - `weaviateCluster`: Weaviate REST API endpoint (string)
   - Connect Schedule Trigger → Parameters

3. **Create an Airtable node "Get Next Page Token"**
   - Type: Airtable (v2.1)
   - Operation: Search
   - Base: Your Airtable base ID (e.g., "app4dxOyxKbkYOFfL")
   - Table: Your pagination table (e.g., "tblIBrPDIVMwtUMJr")
   - Filter formula: `{Number} = 0`
   - Connect Parameters → Get Next Page Token

4. **Create an If node "Is Next Pagination Token null?"**
   - Type: If (v2.2)
   - Condition: Check if length of trimmed `Name` > 0 (pagination token exists)
   - Connect Get Next Page Token → Is Next Pagination Token null?

5. **Create an If node "First Iteration?"**
   - Type: If (v2.2)
   - Condition: Check if trimmed pagination token equals "INIT"
   - Connect "Is Next Pagination Token null?" True output → First Iteration?

6. **Create HTTP Request node "Get Record First Page"**
   - Type: HTTP Request (v4.2)
   - URL: `={{ $parameter.sourceIndex }}/vectors/list`
   - Query params: namespace, limit (from Parameters)
   - Authentication: Pinecone API credentials (predefined)
   - Connect First Iteration? True → Get Record First Page

7. **Create HTTP Request node "Get Record Next Page"**
   - Type: HTTP Request (v4.2)
   - URL: `={{ $parameter.sourceIndex }}/vectors/list`
   - Query params: namespace, limit, pagination_token (from Airtable token)
   - Authentication: Pinecone API credentials (predefined)
   - Connect First Iteration? False → Get Record Next Page

8. **Create Airtable node "Save Next Page Token"**
   - Type: Airtable (v2.1)
   - Operation: Upsert
   - Base/Table same as Step 3
   - Map `Name` to `pagination.next` from Pinecone response or empty string
   - Match on `Number = 0`
   - Connect Get Record First Page → Save Next Page Token

9. **Create Airtable node "Save Next Page Token1"**
   - Same as Step 8, connected from Get Record Next Page

10. **Create Set nodes "Select Ids" and "Select Ids1"**
    - Extract `vectors` and `namespace` fields from Pinecone response JSON.
    - Connect Save Next Page Token → Select Ids
    - Connect Save Next Page Token1 → Select Ids1

11. **Create a Code node "Prepare Fetch Body"**
    - JavaScript to:
      - Validate presence of `vectors` array
      - Extract vector IDs
      - Return `{ids: [...], namespace: ...}`
    - Connect Select Ids and Select Ids1 → Prepare Fetch Body

12. **Create HTTP Request node "Fetch Vectors"**
    - URL: `={{ $parameter.sourceIndex }}/vectors/fetch`
    - Method: POST
    - Auth: Pinecone API credentials
    - JSON Body includes vector IDs and namespace, with `include_values` and `include_metadata` true
    - Connect Prepare Fetch Body → Fetch Vectors

13. **Create Code node "Format2Weaviate"**
    - JavaScript to map Pinecone vectors to Weaviate objects with:
      - `class`, `id`, `vector`, and `properties` from metadata fields
    - Connect Fetch Vectors → Format2Weaviate

14. **Create HTTP Request node "LoadWeAviate"**
    - URL: `https://{{ $parameter.weaviateCluster }}/v1/objects`
    - Method: POST
    - Auth: HTTP Bearer Token (Weaviate credentials)
    - Send JSON body from Format2Weaviate output
    - Connect Format2Weaviate → LoadWeAviate

15. **Create If node "Migration completed"**
    - NoOp node named “Migration completed”
    - Connect Is Next Pagination Token null? False output → Migration completed

16. **Configure credentials**
    - Pinecone API: Provide API key, set as predefined credential for HTTP nodes.
    - Weaviate API: Provide Bearer token credential for Weaviate HTTP node.
    - Airtable API: Provide API key with read/write access to the base.

17. **Initialize Airtable table**
    - Create a record with Name = "INIT" and Number = 0 before starting the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow migrates vector data from Pinecone to Weaviate with pagination control via Airtable.                                | Workflow purpose                                       |
| Airtable table must be prepared with columns: Name (string), Number (number), and initialized with record (INIT,0).               | Airtable setup instructions                            |
| The workflow runs incrementally every 15 seconds, suitable for large datasets and rate-limited APIs.                              | Scheduling rationale                                  |
| Pinecone API limit: max 100 vectors per batch, reflected in batchLimit parameter.                                                 | Pinecone API docs                                      |
| Metadata fields mapped explicitly: issue_customer, issue_id, issue_key, summary, text, triage. Ensure these exist in Pinecone.   | Vector metadata mapping                                |
| Weaviate REST endpoint expects HTTP Bearer token authentication.                                                                  | Weaviate API authentication                            |
| Sticky Note in the workflow explains the overall process and parameter requirements.                                              | Workflow node Sticky Note                              |

---

**Disclaimer:** The provided content is generated from an n8n workflow automation and adheres strictly to legal and content policies.