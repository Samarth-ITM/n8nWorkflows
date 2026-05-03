🛠️ Raindrop Tool MCP Server 💪 all 13 operations

https://n8nworkflows.xyz/workflows/----raindrop-tool-mcp-server----all-13-operations-5346


# 🛠️ Raindrop Tool MCP Server 💪 all 13 operations

### 1. Workflow Overview

This workflow, titled **"Raindrop Tool MCP Server"**, orchestrates all 13 core operations available in the Raindrop Tool integration within n8n. It serves as a backend server-like setup that listens for incoming Multi-Command Protocol (MCP) triggers and executes the corresponding Raindrop API operation based on the request. The workflow is designed to provide a complete API interface for managing bookmarks, collections, tags, and user data within Raindrop.

The workflow’s logic is structured into the following functional blocks:

- **1.1 MCP Trigger Reception:**  
  The entry point that listens for incoming MCP commands and routes them to the appropriate Raindrop operation nodes.

- **1.2 Bookmark Operations:**  
  Nodes handling creation, deletion, retrieval (single and multiple), and updates of bookmarks.

- **1.3 Collection Operations:**  
  Nodes managing creation, deletion, retrieval (single and multiple), and updates of collections.

- **1.4 Tag Operations:**  
  Nodes enabling deletion and retrieval of tags.

- **1.5 User Operations:**  
  Node retrieving user information.

Each node within these blocks is triggered by the MCP trigger node, allowing external systems or clients to call any operation via MCP commands.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Reception

- **Overview:**  
  This block listens for incoming MCP requests, acting as the workflow’s entry point. It receives commands specifying which Raindrop operation to perform.

- **Nodes Involved:**  
  - Raindrop Tool MCP Server

- **Node Details:**  
  - **Raindrop Tool MCP Server**  
    - *Type & Role:* MCP Trigger node from the LangChain MCP integration, acts as a webhook trigger to receive MCP commands remotely.  
    - *Configuration:* Default configuration with a unique webhook ID. No additional parameters set.  
    - *Input/Output:* No input; outputs data to all subsequent Raindrop Tool nodes.  
    - *Edge cases:* Webhook connectivity issues, malformed MCP commands, or unsupported operations can cause failures. Authentication issues depend on Raindrop credentials configured in the called nodes.  
    - *Version Requirements:* Requires n8n version supporting `@n8n/n8n-nodes-langchain.mcpTrigger`.  
    - *Sub-workflow:* None.

#### 2.2 Bookmark Operations

- **Overview:**  
  Handles all bookmark-related API calls including create, delete, get single, get multiple, and update bookmarks.

- **Nodes Involved:**  
  - Create a bookmark  
  - Delete a bookmark  
  - Get a bookmark  
  - Get many bookmarks  
  - Update a bookmark

- **Node Details:**  
  All nodes use the **Raindrop Tool** node type specialized for Raindrop API operations.

  - **Create a bookmark**  
    - *Role:* Creates a new bookmark in Raindrop.  
    - *Configuration:* No explicit parameters shown, but typically requires bookmark URL, title, collection ID, tags, etc.  
    - *Input:* Triggered by MCP Server node.  
    - *Output:* Returns created bookmark data.  
    - *Edge cases:* Invalid URL, missing required fields, or authentication failure.

  - **Delete a bookmark**  
    - *Role:* Deletes a specified bookmark by ID.  
    - *Input:* MCP Server node.  
    - *Output:* Confirmation of deletion.  
    - *Edge cases:* Bookmark ID not found, permission denied.

  - **Get a bookmark**  
    - *Role:* Retrieves details of a single bookmark by ID.  
    - *Input:* MCP Server node.  
    - *Output:* Bookmark data.  
    - *Edge cases:* Bookmark not found, access denied.

  - **Get many bookmarks**  
    - *Role:* Retrieves multiple bookmarks, optionally filtered by parameters such as collection or tags.  
    - *Input:* MCP Server node.  
    - *Output:* Array of bookmarks.  
    - *Edge cases:* Large data sets causing timeouts, invalid filters.

  - **Update a bookmark**  
    - *Role:* Updates properties of a bookmark by ID.  
    - *Input:* MCP Server node.  
    - *Output:* Updated bookmark data.  
    - *Edge cases:* Invalid update fields, bookmark not found.

#### 2.3 Collection Operations

- **Overview:**  
  Manages collections in Raindrop, including creating, deleting, fetching (single/multiple), and updating collections.

- **Nodes Involved:**  
  - Create a collection  
  - Delete a collection  
  - Get a collection  
  - Get many collections  
  - Update a collection

- **Node Details:**  
  Each node is a **Raindrop Tool** node configured for collection operations.

  - **Create a collection**  
    - *Role:* Creates a new collection.  
    - *Input:* MCP Server node.  
    - *Output:* New collection details.  
    - *Edge cases:* Missing collection name, permission errors.

  - **Delete a collection**  
    - *Role:* Deletes a collection by ID.  
    - *Input:* MCP Server node.  
    - *Output:* Deletion confirmation.  
    - *Edge cases:* Collection not found, cannot delete non-empty collection.

  - **Get a collection**  
    - *Role:* Retrieves a collection’s details by ID.  
    - *Input:* MCP Server node.  
    - *Output:* Collection data.  
    - *Edge cases:* Collection not found.

  - **Get many collections**  
    - *Role:* Lists collections, possibly with paging.  
    - *Input:* MCP Server node.  
    - *Output:* Array of collections.  
    - *Edge cases:* Large result sets, timeout.

  - **Update a collection**  
    - *Role:* Updates collection properties by ID.  
    - *Input:* MCP Server node.  
    - *Output:* Updated collection info.  
    - *Edge cases:* Invalid update data.

#### 2.4 Tag Operations

- **Overview:**  
  Provides tag deletion and retrieval capabilities.

- **Nodes Involved:**  
  - Delete a tag  
  - Get many tags

- **Node Details:**  
  Both are **Raindrop Tool** nodes.

  - **Delete a tag**  
    - *Role:* Deletes a tag by ID or name.  
    - *Input:* MCP Server node.  
    - *Output:* Confirmation of deletion.  
    - *Edge cases:* Tag not found.

  - **Get many tags**  
    - *Role:* Retrieves a list of tags.  
    - *Input:* MCP Server node.  
    - *Output:* Array of tags.  
    - *Edge cases:* Large tag sets.

#### 2.5 User Operations

- **Overview:**  
  Retrieves authenticated user information from Raindrop.

- **Nodes Involved:**  
  - Get a user

- **Node Details:**  
  - **Get a user**  
    - *Role:* Fetches current user profile details.  
    - *Input:* MCP Server node.  
    - *Output:* User information object.  
    - *Edge cases:* Authentication token invalid or expired.

---

### 3. Summary Table

| Node Name             | Node Type                    | Functional Role                         | Input Node(s)           | Output Node(s)            | Sticky Note                           |
|-----------------------|------------------------------|---------------------------------------|------------------------|---------------------------|-------------------------------------|
| Workflow Overview 0   | Sticky Note                  | Workflow title and overview placeholder| None                   | None                      |                                     |
| Raindrop Tool MCP Server | MCP Trigger (LangChain MCP) | Entry point, receives MCP commands    | None                   | All Raindrop Tool nodes   |                                     |
| Create a bookmark     | Raindrop Tool                | Creates a new bookmark                 | Raindrop Tool MCP Server | None                      |                                     |
| Delete a bookmark     | Raindrop Tool                | Deletes a bookmark                     | Raindrop Tool MCP Server | None                      |                                     |
| Get a bookmark        | Raindrop Tool                | Retrieves a single bookmark            | Raindrop Tool MCP Server | None                      |                                     |
| Get many bookmarks    | Raindrop Tool                | Retrieves multiple bookmarks           | Raindrop Tool MCP Server | None                      |                                     |
| Update a bookmark     | Raindrop Tool                | Updates a bookmark                     | Raindrop Tool MCP Server | None                      |                                     |
| Sticky Note 1         | Sticky Note                  | Bookmark operations group placeholder  | None                   | None                      |                                     |
| Create a collection   | Raindrop Tool                | Creates a new collection               | Raindrop Tool MCP Server | None                      |                                     |
| Delete a collection   | Raindrop Tool                | Deletes a collection                   | Raindrop Tool MCP Server | None                      |                                     |
| Get a collection      | Raindrop Tool                | Retrieves a single collection          | Raindrop Tool MCP Server | None                      |                                     |
| Get many collections  | Raindrop Tool                | Retrieves multiple collections         | Raindrop Tool MCP Server | None                      |                                     |
| Update a collection   | Raindrop Tool                | Updates a collection                   | Raindrop Tool MCP Server | None                      |                                     |
| Sticky Note 2         | Sticky Note                  | Collection operations group placeholder| None                   | None                      |                                     |
| Delete a tag          | Raindrop Tool                | Deletes a tag                         | Raindrop Tool MCP Server | None                      |                                     |
| Get many tags         | Raindrop Tool                | Retrieves multiple tags                | Raindrop Tool MCP Server | None                      |                                     |
| Sticky Note 3         | Sticky Note                  | Tag operations group placeholder       | None                   | None                      |                                     |
| Get a user            | Raindrop Tool                | Retrieves user profile information     | Raindrop Tool MCP Server | None                      |                                     |
| Sticky Note 4         | Sticky Note                  | User operations group placeholder      | None                   | None                      |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the MCP Trigger node:**  
   - Node type: `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP Trigger)  
   - Configure webhook ID automatically or keep default.  
   - This node acts as the entry point for all incoming MCP commands.

3. **Add Raindrop Tool nodes for Bookmark operations:**  
   For each operation below, add a node of type `Raindrop Tool` and configure the operation accordingly. Connect each node’s input to the MCP Trigger node.

   - **Create a bookmark:** Set operation to "Create Bookmark". Configure required parameters such as URL, title, collection ID, tags, etc.  
   - **Delete a bookmark:** Set operation to "Delete Bookmark". Parameter: Bookmark ID.  
   - **Get a bookmark:** Set operation to "Get Bookmark". Parameter: Bookmark ID.  
   - **Get many bookmarks:** Set operation to "Get Many Bookmarks". Optional filters like collection or tags can be configured.  
   - **Update a bookmark:** Set operation to "Update Bookmark". Parameters: Bookmark ID plus fields to update.

4. **Add Raindrop Tool nodes for Collection operations:**  
   Similarly, create nodes for:

   - Create a collection  
   - Delete a collection  
   - Get a collection  
   - Get many collections  
   - Update a collection

   Connect each node’s input to the MCP Trigger node. Configure operation type accordingly.

5. **Add Raindrop Tool nodes for Tag operations:**  
   Add nodes for:

   - Delete a tag (parameter: tag ID or name)  
   - Get many tags (optionally with filters)

   Connect inputs to MCP Trigger node.

6. **Add Raindrop Tool node for User operation:**  
   - Node: Get a user  
   - Connect input to MCP Trigger node.

7. **Set Credentials:**  
   - For all Raindrop Tool nodes, configure Raindrop API credentials (API token).  
   - Ensure the API token has sufficient permissions for all operations.

8. **Optional: Add Sticky Notes for clarity:**  
   - Group bookmark operation nodes with a sticky note labeled "Bookmark Operations".  
   - Group collection operations similarly with "Collection Operations".  
   - Add sticky notes for tags and user operations if desired.

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                |
|------------------------------------------------------------------------------|-----------------------------------------------|
| MCP Trigger node enables remote invocation by external clients via MCP protocol.| n8n MCP Trigger documentation                  |
| Raindrop Tool integration supports all core API operations: bookmarks, collections, tags, and user profile.| Raindrop API docs: https://developer.raindrop.io/ |
| Ensure Raindrop API token has appropriate scopes for all operations to prevent authorization errors.| Raindrop API token management                   |
| This workflow is designed as a centralized backend server for Raindrop operations, enabling external automation.| Use case: API gateway or MCP server            |

---

**Disclaimer:** The content provided here stems exclusively from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.