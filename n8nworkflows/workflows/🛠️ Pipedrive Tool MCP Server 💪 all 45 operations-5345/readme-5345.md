🛠️ Pipedrive Tool MCP Server 💪 all 45 operations

https://n8nworkflows.xyz/workflows/----pipedrive-tool-mcp-server----all-45-operations-5345


# 🛠️ Pipedrive Tool MCP Server 💪 all 45 operations

### 1. Workflow Overview

This workflow, titled **🛠️ Pipedrive Tool MCP Server**, serves as a multi-operation control point (MCP) server for the Pipedrive CRM system, exposing 45 distinct operations across various Pipedrive resources. It is designed to enable AI agents or external systems to invoke any supported Pipedrive API operation through a single webhook endpoint. The key use cases include creating, reading, updating, deleting, searching, and managing multiple CRM entities such as activities, deals, deal activities, deal products, files, leads, notes, organizations, people, and products.

The workflow is logically divided into blocks, each corresponding to a Pipedrive resource and grouping all related operations:

- **1.1 MCP Trigger & Overview**: Entry point and setup information.
- **1.2 Activity Operations**: CRUD operations on activities.
- **1.3 Deal Operations**: CRUD, duplicate, search, and listing of deals.
- **1.4 Deal Activity Operations**: Listing deal activities.
- **1.5 Deal Product Operations**: Manage products attached to deals.
- **1.6 File Operations**: Manage files including download.
- **1.7 Lead Operations**: CRUD and listing of leads.
- **1.8 Note Operations**: CRUD and listing of notes.
- **1.9 Organization Operations**: CRUD, search, and listing of organizations.
- **1.10 Person Operations**: CRUD, search, and listing of people.
- **1.11 Product Operations**: Listing products.

Each operation node is pre-configured to receive parameters dynamically from AI agents via `$fromAI()` expressions, allowing automated and zero-configuration execution. The MCP trigger node listens on the webhook path `pipedrive-tool-mcp` and routes incoming requests to the appropriate operation node.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger & Overview

- **Overview:** Entry point for all MCP requests and documentation sticky note explaining workflow purpose, setup, and usage.
- **Nodes Involved:**  
  - `Pipedrive Tool MCP Server` (MCP Trigger)  
  - `Workflow Overview 0` (Sticky Note)

- **Node Details:**

| Node Name                  | Details                                                                                                                          |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Pipedrive Tool MCP Server  | - Type: `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP trigger node) <br> - Listens on webhook path `pipedrive-tool-mcp` <br> - Receives all external calls and triggers corresponding downstream nodes <br> - No input connections <br> - Outputs linked to each operation node via ai_tool connection <br> - Potential failures: webhook connection issues, authentication misconfiguration |
| Workflow Overview 0         | - Type: Sticky Note <br> - Contains detailed explanation of available operations (45 total), setup instructions, and help links <br> - No connections <br> - Serves as user reference for workflow usage |

---

#### 1.2 Activity Operations

- **Overview:** Manage activities with operations: create, delete, get, get many, update.
- **Nodes Involved:**  
  - `Create an activity`  
  - `Delete an activity`  
  - `Get an activity`  
  - `Get many activities`  
  - `Update an activity`  
  - `Sticky Note 1`

- **Node Details:**

| Node Name           | Details                                                                                                                            |
|---------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Create an activity   | - Type: `pipedriveTool` <br> - Operation: create activity <br> - Parameters: dynamically populated fields `done`, `type`, `subject` from AI input <br> - Uses no additional fields by default <br> - Input: MCP Trigger <br> - Output: response to MCP <br> - Failure: invalid parameters, API rate limits, auth errors |
| Delete an activity   | - Type: `pipedriveTool` <br> - Operation: delete activity <br> - Parameter: `activityId` from AI input <br> - Input: MCP Trigger <br> - Output: response to MCP <br> - Failure: invalid ID, not found, permission denied |
| Get an activity      | - Type: `pipedriveTool` <br> - Operation: get activity <br> - Parameters: `activityId` and optional `resolveProperties` boolean from AI <br> - Input: MCP Trigger <br> - Output: response to MCP <br> - Failure: invalid ID, network errors |
| Get many activities  | - Type: `pipedriveTool` <br> - Operation: getAll activities <br> - Parameters: optional `resolveProperties` <br> - Input: MCP Trigger <br> - Output: response to MCP <br> - Failure: large data sets may cause timeout |
| Update an activity   | - Type: `pipedriveTool` <br> - Operation: update activity <br> - Parameters: `activityId`, `updateFields` (empty by default), `encodeProperties` boolean <br> - Input: MCP Trigger <br> - Output: response to MCP <br> - Failure: invalid ID, invalid update data |

---

#### 1.3 Deal Operations

- **Overview:** Group of nodes to create, delete, duplicate, get, list, search, and update deals.
- **Nodes Involved:**  
  - `Create a deal`  
  - `Delete a deal`  
  - `Duplicate a deal`  
  - `Get a deal`  
  - `Get many deals`  
  - `Search a deal`  
  - `Update a deal`  
  - `Sticky Note 2`

- **Node Details:**

| Node Name          | Details                                                                                                                          |
|--------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Create a deal      | - Type: `pipedriveTool` <br> - Operation: create deal <br> - Parameters: `title`, `org_id`, `person_id`, `associateWith` from AI input <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: missing required fields, auth errors |
| Delete a deal      | - Type: `pipedriveTool` <br> - Operation: delete deal <br> - Parameter: `dealId` from AI <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid deal ID |
| Duplicate a deal   | - Type: `pipedriveTool` <br> - Operation: duplicate deal <br> - Parameter: `dealId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid deal ID |
| Get a deal         | - Type: `pipedriveTool` <br> - Operation: get deal <br> - Parameters: `dealId`, optional `resolveProperties` boolean <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid ID |
| Get many deals     | - Type: `pipedriveTool` <br> - Operation: getAll deals <br> - Parameters: empty filter object by default, optional `resolveProperties` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: timeout on large data sets |
| Search a deal      | - Type: `pipedriveTool` <br> - Operation: search deal <br> - Parameters: `term`, `exactMatch` boolean <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: empty or invalid search term |
| Update a deal      | - Type: `pipedriveTool` <br> - Operation: update deal <br> - Parameters: `dealId`, `updateFields` empty by default, `encodeProperties` boolean <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid update data |

---

#### 1.4 Deal Activity Operations

- **Overview:** Retrieve all activities associated with a deal.
- **Nodes Involved:**  
  - `Get many deal activities`  
  - `Sticky Note 3`

- **Node Details:**

| Node Name            | Details                                                                                                                  |
|----------------------|--------------------------------------------------------------------------------------------------------------------------|
| Get many deal activities | - Type: `pipedriveTool` <br> - Operation: getAll dealActivity <br> - Parameter: `dealId` from AI input <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid deal ID |

---

#### 1.5 Deal Product Operations

- **Overview:** Add, get, remove, and update products attached to deals.
- **Nodes Involved:**  
  - `Add a deal product`  
  - `Get many deal products`  
  - `Remove a deal product`  
  - `Update a deal product`  
  - `Sticky Note 4`

- **Node Details:**

| Node Name           | Details                                                                                                                   |
|---------------------|---------------------------------------------------------------------------------------------------------------------------|
| Add a deal product  | - Type: `pipedriveTool` <br> - Operation: add dealProduct <br> - Parameters: `dealId`, `quantity`, `productId`, `item_price` from AI input <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: missing or invalid product/deal IDs |
| Get many deal products | - Type: `pipedriveTool` <br> - Operation: getAll dealProduct <br> - Parameter: `dealId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid deal ID |
| Remove a deal product | - Type: `pipedriveTool` <br> - Operation: remove dealProduct <br> - Parameters: `dealId`, `productAttachmentId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid attachment ID |
| Update a deal product | - Type: `pipedriveTool` <br> - Operation: update dealProduct <br> - Parameters: `dealId`, `productAttachmentId`, `updateFields` empty by default <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid update data |

---

#### 1.6 File Operations

- **Overview:** Create, delete, download, get, and update files.
- **Nodes Involved:**  
  - `Create a file`  
  - `Delete a file`  
  - `Download a file`  
  - `Get a file`  
  - `update details of a file`  
  - `Sticky Note 5`

- **Node Details:**

| Node Name          | Details                                                                                                                      |
|--------------------|------------------------------------------------------------------------------------------------------------------------------|
| Create a file      | - Type: `pipedriveTool` <br> - Operation: create file <br> - Parameters: binaryPropertyName (from AI), empty additionalFields <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: missing binary data, invalid property name |
| Delete a file      | - Type: `pipedriveTool` <br> - Operation: delete file <br> - Parameter: `fileId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid file ID |
| Download a file    | - Type: `pipedriveTool` <br> - Operation: download file <br> - Parameters: `fileId`, `binaryPropertyName` <br> - Input: MCP Trigger <br> - Output: binary file data <br> - Failure: invalid file ID, binary data issues |
| Get a file         | - Type: `pipedriveTool` <br> - Operation: get file <br> - Parameter: `fileId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid file ID |
| update details of a file | - Type: `pipedriveTool` <br> - Operation: update file <br> - Parameters: `fileId`, `updateFields` (empty by default) <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid update data |

---

#### 1.7 Lead Operations

- **Overview:** Create, delete, get, list, and update leads.
- **Nodes Involved:**  
  - `Create a lead`  
  - `Delete a lead`  
  - `Get a lead`  
  - `Get many leads`  
  - `Update a lead`  
  - `Sticky Note 6`

- **Node Details:**

| Node Name          | Details                                                                                                                |
|--------------------|------------------------------------------------------------------------------------------------------------------------|
| Create a lead      | - Type: `pipedriveTool` <br> - Operation: create lead <br> - Parameters: `title`, `person_id`, `associateWith`, `organization_id` from AI <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: missing required fields |
| Delete a lead      | - Type: `pipedriveTool` <br> - Operation: delete lead <br> - Parameter: `leadId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid lead ID |
| Get a lead         | - Type: `pipedriveTool` <br> - Operation: get lead <br> - Parameter: `leadId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid ID |
| Get many leads     | - Type: `pipedriveTool` <br> - Operation: getAll leads <br> - Parameters: empty filters <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: timeout on large data |
| Update a lead      | - Type: `pipedriveTool` <br> - Operation: update lead <br> - Parameters: `leadId`, `updateFields` empty by default <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid update data |

---

#### 1.8 Note Operations

- **Overview:** Create, delete, get, list, and update notes.
- **Nodes Involved:**  
  - `Create a note`  
  - `Delete a note`  
  - `Get a note`  
  - `Get many notes`  
  - `Update a note`  
  - `Sticky Note 7`

- **Node Details:**

| Node Name          | Details                                                                                                              |
|--------------------|----------------------------------------------------------------------------------------------------------------------|
| Create a note      | - Type: `pipedriveTool` <br> - Operation: create note <br> - Parameters: `content` from AI <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: missing content |
| Delete a note      | - Type: `pipedriveTool` <br> - Operation: delete note <br> - Parameter: `noteId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid note ID |
| Get a note         | - Type: `pipedriveTool` <br> - Operation: get note <br> - Parameter: `noteId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid note ID |
| Get many notes     | - Type: `pipedriveTool` <br> - Operation: getAll notes <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: timeout on large data |
| Update a note      | - Type: `pipedriveTool` <br> - Operation: update note <br> - Parameters: `noteId`, `updateFields` empty by default <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid update data |

---

#### 1.9 Organization Operations

- **Overview:** Manage organizations with create, delete, get, list, search, and update operations.
- **Nodes Involved:**  
  - `Create an organization`  
  - `Delete an organization`  
  - `Get an organization`  
  - `Get many organizations`  
  - `Search an organization`  
  - `Update an organization`  
  - `Sticky Note 8`

- **Node Details:**

| Node Name               | Details                                                                                                                 |
|-------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Create an organization  | - Type: `pipedriveTool` <br> - Operation: create organization <br> - Parameter: `name` from AI <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: missing name |
| Delete an organization  | - Type: `pipedriveTool` <br> - Operation: delete organization <br> - Parameter: `organizationId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid ID |
| Get an organization     | - Type: `pipedriveTool` <br> - Operation: get organization <br> - Parameters: `organizationId`, optional `resolveProperties` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid ID |
| Get many organizations  | - Type: `pipedriveTool` <br> - Operation: getAll organizations <br> - Parameters: empty filters, optional `resolveProperties` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: large data timeout |
| Search an organization  | - Type: `pipedriveTool` <br> - Operation: search organization <br> - Parameter: `term` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: empty search term |
| Update an organization  | - Type: `pipedriveTool` <br> - Operation: update organization <br> - Parameters: `organizationId`, `updateFields` empty, `encodeProperties` boolean <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid update data |

---

#### 1.10 Person Operations

- **Overview:** Manage people with create, delete, get, list, search, and update operations.
- **Nodes Involved:**  
  - `Create a person`  
  - `Delete a person`  
  - `Get a person`  
  - `Get many people`  
  - `Search a person`  
  - `Update a person`  
  - `Sticky Note 9`

- **Node Details:**

| Node Name           | Details                                                                                                                   |
|---------------------|---------------------------------------------------------------------------------------------------------------------------|
| Create a person     | - Type: `pipedriveTool` <br> - Operation: create person <br> - Parameter: `name` from AI <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: missing name |
| Delete a person     | - Type: `pipedriveTool` <br> - Operation: delete person <br> - Parameter: `personId` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid ID |
| Get a person        | - Type: `pipedriveTool` <br> - Operation: get person <br> - Parameters: `personId`, optional `resolveProperties` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid ID |
| Get many people     | - Type: `pipedriveTool` <br> - Operation: getAll people <br> - Parameters: empty additionalFields, optional `resolveProperties` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: large data timeout |
| Search a person     | - Type: `pipedriveTool` <br> - Operation: search person <br> - Parameters: `term` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: empty search term |
| Update a person     | - Type: `pipedriveTool` <br> - Operation: update person <br> - Parameters: `personId`, `updateFields` empty, `encodeProperties` boolean <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: invalid update data |

---

#### 1.11 Product Operations

- **Overview:** Retrieve all products (read-only).
- **Nodes Involved:**  
  - `Get many products`  
  - `Sticky Note 10`

- **Node Details:**

| Node Name          | Details                                                                                          |
|--------------------|------------------------------------------------------------------------------------------------|
| Get many products  | - Type: `pipedriveTool` <br> - Operation: getAll product <br> - Parameter: optional `resolveProperties` <br> - Input: MCP Trigger <br> - Output: response <br> - Failure: timeout on large data |

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                  | Input Node(s)               | Output Node(s) | Sticky Note                                                                                                                          |
|---------------------------|----------------------------------|---------------------------------|-----------------------------|----------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0       | Sticky Note                      | Documentation overview          | None                        | None           | Contains full description, operations list, setup instructions, and support links                                                    |
| Pipedrive Tool MCP Server | MCP Trigger                     | Entry trigger for all requests  | None                        | All operation nodes |                                                                                                                                    |
| Create an activity        | Pipedrive Tool Node             | Create activity                 | MCP Trigger                 | MCP response   | Covered by Sticky Note 1                                                                                                             |
| Delete an activity        | Pipedrive Tool Node             | Delete activity                 | MCP Trigger                 | MCP response   | Covered by Sticky Note 1                                                                                                             |
| Get an activity           | Pipedrive Tool Node             | Retrieve single activity        | MCP Trigger                 | MCP response   | Covered by Sticky Note 1                                                                                                             |
| Get many activities       | Pipedrive Tool Node             | Retrieve multiple activities    | MCP Trigger                 | MCP response   | Covered by Sticky Note 1                                                                                                             |
| Update an activity        | Pipedrive Tool Node             | Update activity                 | MCP Trigger                 | MCP response   | Covered by Sticky Note 1                                                                                                             |
| Sticky Note 1             | Sticky Note                    | Activity block label            | None                        | None           | ## Activity                                                                                                                         |
| Create a deal             | Pipedrive Tool Node             | Create deal                    | MCP Trigger                 | MCP response   | Covered by Sticky Note 2                                                                                                             |
| Delete a deal             | Pipedrive Tool Node             | Delete deal                    | MCP Trigger                 | MCP response   | Covered by Sticky Note 2                                                                                                             |
| Duplicate a deal          | Pipedrive Tool Node             | Duplicate deal                 | MCP Trigger                 | MCP response   | Covered by Sticky Note 2                                                                                                             |
| Get a deal                | Pipedrive Tool Node             | Retrieve single deal           | MCP Trigger                 | MCP response   | Covered by Sticky Note 2                                                                                                             |
| Get many deals            | Pipedrive Tool Node             | Retrieve multiple deals        | MCP Trigger                 | MCP response   | Covered by Sticky Note 2                                                                                                             |
| Search a deal             | Pipedrive Tool Node             | Search deals by term           | MCP Trigger                 | MCP response   | Covered by Sticky Note 2                                                                                                             |
| Update a deal             | Pipedrive Tool Node             | Update deal                   | MCP Trigger                 | MCP response   | Covered by Sticky Note 2                                                                                                             |
| Sticky Note 2             | Sticky Note                    | Deal block label               | None                        | None           | ## Deal                                                                                                                             |
| Get many deal activities  | Pipedrive Tool Node             | List deal activities           | MCP Trigger                 | MCP response   | Covered by Sticky Note 3                                                                                                             |
| Sticky Note 3             | Sticky Note                    | Dealactivity block label       | None                        | None           | ## Dealactivity                                                                                                                     |
| Add a deal product        | Pipedrive Tool Node             | Attach product to deal         | MCP Trigger                 | MCP response   | Covered by Sticky Note 4                                                                                                             |
| Get many deal products    | Pipedrive Tool Node             | List deal products             | MCP Trigger                 | MCP response   | Covered by Sticky Note 4                                                                                                             |
| Remove a deal product     | Pipedrive Tool Node             | Remove product from deal       | MCP Trigger                 | MCP response   | Covered by Sticky Note 4                                                                                                             |
| Update a deal product     | Pipedrive Tool Node             | Update deal product details    | MCP Trigger                 | MCP response   | Covered by Sticky Note 4                                                                                                             |
| Sticky Note 4             | Sticky Note                    | Dealproduct block label        | None                        | None           | ## Dealproduct                                                                                                                     |
| Create a file             | Pipedrive Tool Node             | Upload a file                 | MCP Trigger                 | MCP response   | Covered by Sticky Note 5                                                                                                             |
| Delete a file             | Pipedrive Tool Node             | Delete a file                 | MCP Trigger                 | MCP response   | Covered by Sticky Note 5                                                                                                             |
| Download a file           | Pipedrive Tool Node             | Download file binary          | MCP Trigger                 | MCP response   | Covered by Sticky Note 5                                                                                                             |
| Get a file                | Pipedrive Tool Node             | Retrieve file info            | MCP Trigger                 | MCP response   | Covered by Sticky Note 5                                                                                                             |
| update details of a file  | Pipedrive Tool Node             | Update file metadata          | MCP Trigger                 | MCP response   | Covered by Sticky Note 5                                                                                                             |
| Sticky Note 5             | Sticky Note                    | File block label              | None                        | None           | ## File                                                                                                                             |
| Create a lead             | Pipedrive Tool Node             | Create lead                  | MCP Trigger                 | MCP response   | Covered by Sticky Note 6                                                                                                             |
| Delete a lead             | Pipedrive Tool Node             | Delete lead                  | MCP Trigger                 | MCP response   | Covered by Sticky Note 6                                                                                                             |
| Get a lead                | Pipedrive Tool Node             | Retrieve single lead         | MCP Trigger                 | MCP response   | Covered by Sticky Note 6                                                                                                             |
| Get many leads            | Pipedrive Tool Node             | List leads                   | MCP Trigger                 | MCP response   | Covered by Sticky Note 6                                                                                                             |
| Update a lead             | Pipedrive Tool Node             | Update lead                  | MCP Trigger                 | MCP response   | Covered by Sticky Note 6                                                                                                             |
| Sticky Note 6             | Sticky Note                    | Lead block label             | None                        | None           | ## Lead                                                                                                                             |
| Create a note             | Pipedrive Tool Node             | Create note                  | MCP Trigger                 | MCP response   | Covered by Sticky Note 7                                                                                                             |
| Delete a note             | Pipedrive Tool Node             | Delete note                  | MCP Trigger                 | MCP response   | Covered by Sticky Note 7                                                                                                             |
| Get a note                | Pipedrive Tool Node             | Retrieve single note         | MCP Trigger                 | MCP response   | Covered by Sticky Note 7                                                                                                             |
| Get many notes            | Pipedrive Tool Node             | List notes                   | MCP Trigger                 | MCP response   | Covered by Sticky Note 7                                                                                                             |
| Update a note             | Pipedrive Tool Node             | Update note                  | MCP Trigger                 | MCP response   | Covered by Sticky Note 7                                                                                                             |
| Sticky Note 7             | Sticky Note                    | Note block label             | None                        | None           | ## Note                                                                                                                             |
| Create an organization    | Pipedrive Tool Node             | Create organization          | MCP Trigger                 | MCP response   | Covered by Sticky Note 8                                                                                                             |
| Delete an organization    | Pipedrive Tool Node             | Delete organization          | MCP Trigger                 | MCP response   | Covered by Sticky Note 8                                                                                                             |
| Get an organization       | Pipedrive Tool Node             | Retrieve single organization | MCP Trigger                 | MCP response   | Covered by Sticky Note 8                                                                                                             |
| Get many organizations    | Pipedrive Tool Node             | List organizations           | MCP Trigger                 | MCP response   | Covered by Sticky Note 8                                                                                                             |
| Search an organization    | Pipedrive Tool Node             | Search organizations         | MCP Trigger                 | MCP response   | Covered by Sticky Note 8                                                                                                             |
| Update an organization    | Pipedrive Tool Node             | Update organization          | MCP Trigger                 | MCP response   | Covered by Sticky Note 8                                                                                                             |
| Sticky Note 8             | Sticky Note                    | Organization block label     | None                        | None           | ## Organization                                                                                                                     |
| Create a person           | Pipedrive Tool Node             | Create person                | MCP Trigger                 | MCP response   | Covered by Sticky Note 9                                                                                                             |
| Delete a person           | Pipedrive Tool Node             | Delete person                | MCP Trigger                 | MCP response   | Covered by Sticky Note 9                                                                                                             |
| Get a person              | Pipedrive Tool Node             | Retrieve single person       | MCP Trigger                 | MCP response   | Covered by Sticky Note 9                                                                                                             |
| Get many people           | Pipedrive Tool Node             | List people                  | MCP Trigger                 | MCP response   | Covered by Sticky Note 9                                                                                                             |
| Search a person           | Pipedrive Tool Node             | Search people                | MCP Trigger                 | MCP response   | Covered by Sticky Note 9                                                                                                             |
| Update a person           | Pipedrive Tool Node             | Update person                | MCP Trigger                 | MCP response   | Covered by Sticky Note 9                                                                                                             |
| Sticky Note 9             | Sticky Note                    | Person block label           | None                        | None           | ## Person                                                                                                                           |
| Get many products         | Pipedrive Tool Node             | List products                | MCP Trigger                 | MCP response   | Covered by Sticky Note 10                                                                                                            |
| Sticky Note 10            | Sticky Note                    | Product block label          | None                        | None           | ## Product                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Node type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set webhook path to `pipedrive-tool-mcp`  
   - No parameters needed  
   - This node acts as the entry point for all requests.

2. **Add Sticky Note for Workflow Overview**  
   - Type: Sticky Note  
   - Content: Paste the detailed overview content including all 45 operations, setup instructions, and help links.

3. **Create Activity Nodes (5 total)**  
   For each operation create a `Pipedrive Tool` node configured as follows:  
   - **Create an activity:**  
     - Resource: `activity`  
     - Operation: `create`  
     - Parameters: Use `$fromAI()` for `done`, `type`, and `subject` fields.  
   - **Delete an activity:**  
     - Resource: `activity`  
     - Operation: `delete`  
     - Parameter: `activityId` from AI  
   - **Get an activity:**  
     - Resource: `activity`  
     - Operation: `get`  
     - Parameters: `activityId`, `resolveProperties` from AI  
   - **Get many activities:**  
     - Resource: `activity`  
     - Operation: `getAll`  
     - Parameter: `resolveProperties` from AI  
   - **Update an activity:**  
     - Resource: `activity`  
     - Operation: `update`  
     - Parameters: `activityId`, `updateFields` (empty), `encodeProperties` from AI

   - Connect all these nodes' `ai_tool` inputs to the MCP trigger node output.

4. **Add Sticky Note “Activity” near these nodes**

5. **Repeat similar steps for each resource block:**  
   - **Deal Operations:** Create nodes for create, delete, duplicate, get, get many, search, update with appropriate parameters from AI.  
   - **Deal Activity:** Create node to get many deal activities.  
   - **Deal Product:** Create nodes for add, get many, remove, update deal products.  
   - **File Operations:** Create nodes for create, delete, download, get, update files.  
   - **Lead Operations:** Create nodes for create, delete, get, get many, update leads.  
   - **Note Operations:** Create nodes for create, delete, get, get many, update notes.  
   - **Organization Operations:** Create nodes for create, delete, get, get many, search, update organizations.  
   - **Person Operations:** Create nodes for create, delete, get, get many, search, update persons.  
   - **Product Operations:** Create node for get many products.

   For each block, add a corresponding Sticky Note with the block name.

6. **Set all parameter values in each Pipedrive Tool node to use `$fromAI()` expressions for dynamic input, matching the original fields exactly.**

7. **Ensure all operation nodes have their `resource` and `operation` parameters correctly set per the original workflow.**

8. **Connect each operation node to the MCP trigger node's output via the `ai_tool` input connection.**

9. **Configure Pipedrive credentials:**  
   - Add Pipedrive Tool credentials to one node (e.g., the "Create an activity" node).  
   - Open and close all other Pipedrive Tool nodes to inherit this credential.  
   - Ensure credentials are valid and have appropriate permissions.

10. **Save and Activate the Workflow.**  
    - Once activated, the webhook URL from the MCP trigger node is used by AI agents to invoke any of the 45 operations dynamically.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Zero configuration - all 45 operations pre-built and exposed via a single MCP webhook endpoint.                                     | Workflow Overview sticky note                                                                                   |
| AI agents automatically populate parameters with `$fromAI()` expressions for flexible input handling.                              | Workflow Overview sticky note                                                                                   |
| Native n8n error handling and response formatting ensures robustness in production use.                                             | Workflow Overview sticky note                                                                                   |
| Modify any tool node's default parameters to customize behavior or set static values as needed.                                     | Workflow Overview sticky note                                                                                   |
| MCP trigger node webhook URL must be copied and configured in AI agents or external clients to enable usage.                       | Workflow Overview sticky note                                                                                   |
| For help or customization, visit [n8n documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) or join [discord](https://discord.me/cfomodz). | Workflow Overview sticky note                                                                                   |

---

This documentation provides a complete and structured understanding of the **Pipedrive Tool MCP Server** workflow, enabling reproduction, modification, and integration with external AI agents or systems.