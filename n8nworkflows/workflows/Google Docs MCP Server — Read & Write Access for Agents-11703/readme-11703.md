Google Docs MCP Server — Read & Write Access for Agents

https://n8nworkflows.xyz/workflows/google-docs-mcp-server---read---write-access-for-agents-11703


# Google Docs MCP Server — Read & Write Access for Agents

### 1. Workflow Overview

This workflow, titled **Google Docs MCP Server — Read & Write Access for Agents**, provides a programmatic interface for AI coding agents and other MCP-compatible clients to interact with Google Docs via n8n. Its main purpose is to bridge the gap between Google Drive’s file search capabilities and the ability to reliably read, write, update, and format Google Docs documents programmatically. This enables advanced automation scenarios such as AI-driven document editing, formatting, and content restructuring.

The workflow is organized into the following logical blocks:

- **1.1 MCP Input Reception:** Captures incoming structured MCP requests from agents.
- **1.2 Google Drive File Search:** Handles requests to search files in Google Drive.
- **1.3 Google Docs Document Management:** Supports creating new documents and retrieving existing documents.
- **1.4 Google Docs Content Editing:** Provides a variety of update operations including find & replace, text insertion, page breaks, and list/table formatting.
- **1.5 Documentation & Metadata:** Sticky notes explaining the workflow’s purpose, use cases, and operational flow.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Input Reception

- **Overview:**  
  This block listens for incoming MCP requests from coding agents and triggers the workflow. It accepts structured inputs defining the desired Google Docs or Google Drive operations.

- **Nodes Involved:**  
  - Google Drive MCP Server

- **Node Details:**  
  - **Node Name:** Google Drive MCP Server  
  - **Type:** MCP Trigger (Langchain n8n MCP Trigger node)  
  - **Configuration:**  
    - Acts as a webhook listener on path `a289c719-fb71-4b08-97c6-79d12645dc7e`  
    - Accepts structured MCP payloads from clients requesting operations such as file search, document creation, retrieval, and editing  
  - **Key Expressions/Variables:** None directly; it passes all incoming MCP requests to downstream nodes through `ai_tool` output  
  - **Input/Output:** No input, output routes MCP requests to various Google Docs and Drive nodes  
  - **Version Requirements:** Type version 1, requires Langchain n8n MCP nodes installed  
  - **Error/Edge Cases:**  
    - Invalid or malformed MCP requests could cause failures or no operation  
    - Webhook authentication and rate limiting should be considered externally  
  - **Sub-workflow:** None

---

#### 1.2 Google Drive File Search

- **Overview:**  
  This block performs file searches within the user’s Google Drive based on queries received from agents.

- **Nodes Involved:**  
  - Search Files from Gdrive

- **Node Details:**  
  - **Node Name:** Search Files from Gdrive  
  - **Type:** Google Drive Tool (fileFolder resource)  
  - **Configuration:**  
    - Search limited to "My Drive"  
    - Query string is dynamically set from the MCP input variable `Search_Query`  
    - Limits results to 10 files  
  - **Key Expressions/Variables:**  
    - `{{$fromAI('Search_Query', '', 'string')}}` dynamically provides the search query from the MCP input  
  - **Input:** Receives MCP requests routed from MCP Trigger  
  - **Output:** Returns search results back to MCP Trigger node to reply to agent  
  - **Version Requirements:** Type version 3 of Google Drive Tool  
  - **Edge Cases:**  
    - Empty or invalid queries may return no results  
    - Permission errors if Drive API credentials lack proper scopes  
  - **Sub-workflow:** None

---

#### 1.3 Google Docs Document Management

- **Overview:**  
  This block supports creating new Google Docs and retrieving existing documents by URL or ID.

- **Nodes Involved:**  
  - Create a document in Google Docs  
  - Get a document in Google Docs

- **Node Details:**  

  - **Create a document in Google Docs**  
    - Type: Google Docs Tool  
    - Operation: Create new document  
    - Config: Document title dynamically set from MCP input `Title`  
    - Folder ID fixed as `1L3CaQB_khbHGb4HZO-qyDDQCvZfut3jY` (target folder for new docs)  
    - Input from MCP Trigger  
    - Outputs document metadata to MCP Trigger for agent response  
    - Version 2 of Google Docs Tool  
    - Edge Cases: Title missing or invalid folder ID could cause failure  

  - **Get a document in Google Docs**  
    - Type: Google Docs Tool  
    - Operation: Get document content  
    - Config: Document URL or ID dynamically set from MCP input `Doc_ID_or_URL`  
    - Input from MCP Trigger  
    - Outputs document content to MCP Trigger  
    - Version 2  
    - Edge Cases: Invalid or inaccessible document URL/ID, permission errors  

---

#### 1.4 Google Docs Content Editing

- **Overview:**  
  This block implements multiple Google Docs update operations such as find & replace, text insertion (at end or indexed location), page breaks, and list formatting (checkbox, bullet, numbered) as well as table insertion.

- **Nodes Involved:**  
  - Update a document in Google Docs using Find and Replace  
  - Add Text to a document in Google Docs  
  - Add Page Break to the end of a document in Google Docs1  
  - Create Checkbox List in a document in Google Docs1  
  - Create Bullet List in a document in Google Docs  
  - Create Numbered List in a document in Google Docs  
  - Insert Empty Table to the end of a document in Google Docs1  
  - Insert Empty Table to a document in Google Docs using Index  
  - Insert Page Break to a document in Google Docs using Index  
  - Insert Text to the end of a document in Google Docs  
  - Insert Text to a document in Google Docs using Index  

- **Node Details:**  

  For each node, the configuration pattern is:

  - **Type:** Google Docs Tool  
  - **Operation:** Update  
  - **Document URL:** dynamic from MCP input `Doc_ID_or_URL`  
  - **Write Control Object:** revision ID string from MCP input `Revision_ID` for concurrency control  
  - **Actions UI:** array of action fields specifying the kind of update (e.g., replaceAll, insert text, insert page break, create paragraph bullets with checkbox/bullet/numbered presets, insert table) with parameters dynamically passed from MCP inputs such as text, match case, start/end index, row/column counts, etc.  
  - **Input:** MCP Trigger node routes requests to these nodes based on requested action type  
  - **Output:** Responses returned to MCP Trigger for agent consumption  
  - **Version:** Mostly type version 2  
  - **Edge Cases:**  
    - Revision ID must be valid and current to avoid concurrency conflicts  
    - Invalid indexes or out-of-range parameters cause errors  
    - Permission and quota limits on Google Docs API  
    - Empty or malformed text fields may result in no changes  
  - **Sub-workflow:** None

---

#### 1.5 Documentation & Metadata

- **Overview:**  
  Sticky notes nodes provide descriptive documentation directly in the workflow canvas, explaining the purpose, rationale, and operational flow of the MCP server.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**  
  - **Type:** Sticky Note (n8n built-in)  
  - Contain detailed textual explanations about:  
    - The role of this Google Docs MCP Server  
    - Why providing Docs editing via MCP matters for coding agents  
    - How the MCP server works step-by-step  
  - No inputs or outputs  
  - Serve as inline documentation for maintainers and users  
  - Positioned for easy visual reference near the MCP Trigger  

---

### 3. Summary Table

| Node Name                               | Node Type                    | Functional Role                                   | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                 |
|----------------------------------------|------------------------------|--------------------------------------------------|------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------|
| Google Drive MCP Server                 | MCP Trigger                  | Entry point; receives agent MCP requests         | None                   | All Google Docs & Drive Nodes | Covered by Sticky Note, Sticky Note1, Sticky Note2 explaining overall MCP server purpose and workflow logic |
| Search Files from Gdrive                | Google Drive Tool            | Searches Google Drive files based on query       | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Create a document in Google Docs       | Google Docs Tool             | Creates new Google Docs document                   | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Get a document in Google Docs          | Google Docs Tool             | Retrieves Google Doc content                       | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Update a document in Google Docs using Find and Replace | Google Docs Tool | Performs find & replace updates on a document     | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Add Text to a document in Google Docs  | Google Docs Tool             | Inserts text at the end of a document              | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Add Page Break to the end of a document in Google Docs1 | Google Docs Tool   | Inserts a page break at the document end           | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Create Checkbox List in a document in Google Docs1 | Google Docs Tool       | Converts paragraphs to a checkbox list             | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Create Bullet List in a document in Google Docs | Google Docs Tool         | Converts paragraphs to bullet list                  | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Create Numbered List in a document in Google Docs | Google Docs Tool        | Converts paragraphs to numbered list                | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Insert Empty Table to the end of a document in Google Docs1 | Google Docs Tool      | Inserts an empty table at the end of document      | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Insert Empty Table to a document in Google Docs using Index | Google Docs Tool      | Inserts an empty table at specified index          | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Insert Page Break to a document in Google Docs using Index | Google Docs Tool       | Inserts a page break at specified index             | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Insert Text to the end of a document in Google Docs | Google Docs Tool          | Inserts text at the end of the document             | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Insert Text to a document in Google Docs using Index | Google Docs Tool         | Inserts text at specified index                      | Google Drive MCP Server | Google Drive MCP Server  |                                                                                                             |
| Sticky Note                            | Sticky Note                  | Documentation: Google Docs MCP Access Server       | None                   | None                    | Explains purpose: enables programmatic reading/writing/editing of Google Docs for MCP clients               |
| Sticky Note1                           | Sticky Note                  | Documentation: Why this MCP server matters         | None                   | None                    | Explains limitations of Drive-only access and benefits of MCP for Docs                                      |
| Sticky Note2                           | Sticky Note                  | Documentation: How this MCP Server works           | None                   | None                    | Describes step-by-step workflow execution from agent request to response                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: `Google Drive MCP Server`  
   - Parameters: Set `path` to a unique webhook path (e.g., `a289c719-fb71-4b08-97c6-79d12645dc7e`)  
   - Credentials: None (the MCP server handles external authentication)  
   - Purpose: Receive structured MCP requests from coding agents

2. **Create Google Drive Search Node**  
   - Type: `Google Drive Tool` (resource: fileFolder)  
   - Name: `Search Files from Gdrive`  
   - Parameters:  
     - Limit: 10  
     - Filter: Drive ID mode: list, value: "My Drive"  
     - Query String: Expression: `{{$fromAI('Search_Query', '', 'string')}}`  
   - Credentials: Google Drive OAuth2 with Drive API scopes  
   - Connect output `ai_tool` of MCP Trigger to this node

3. **Create Google Docs Create Document Node**  
   - Type: `Google Docs Tool`  
   - Name: `Create a document in Google Docs`  
   - Parameters:  
     - Title: Expression: `{{$fromAI('Title', '', 'string')}}`  
     - Folder ID: Fixed folder ID (e.g., `1L3CaQB_khbHGb4HZO-qyDDQCvZfut3jY`)  
   - Credentials: Google Docs OAuth2 with Docs API scopes  
   - Connect output `ai_tool` of MCP Trigger to this node

4. **Create Google Docs Get Document Node**  
   - Type: `Google Docs Tool`  
   - Name: `Get a document in Google Docs`  
   - Parameters:  
     - Operation: get  
     - Document URL: Expression: `{{$fromAI('Doc_ID_or_URL', '', 'string')}}`  
   - Credentials: Google Docs OAuth2  
   - Connect output `ai_tool` of MCP Trigger to this node

5. **Create Google Docs Update Nodes**  
   For each update operation, create a Google Docs Tool node with operation `update` and configure as follows:

   - **Find and Replace**  
     - Name: `Update a document in Google Docs using Find and Replace`  
     - Actions: Array with one action field:  
       - action: replaceAll  
       - text: `{{$fromAI('actionFields0_Text', '', 'string')}}`  
       - matchCase: `{{$fromAI('actionFields0_Match_Case', '', 'boolean')}}`  
       - replaceText: `{{$fromAI('actionFields0_New_Text', '', 'string')}}`  
     - Document URL: `{{$fromAI('Doc_ID_or_URL', '', 'string')}}`

   - **Add Text**  
     - Name: `Add Text to a document in Google Docs`  
     - Action: insert text at end  
     - Text: `{{$fromAI('actionFields0_Text', '', 'string')}}`

   - **Add Page Break at End**  
     - Name: `Add Page Break to the end of a document in Google Docs1`  
     - Action: insert pageBreak object at end  

   - **Create Checkbox List**  
     - Name: `Create Checkbox List in a document in Google Docs1`  
     - Action: create paragraphBullets with bulletPreset `BULLET_CHECKBOX`  
     - Start and end index from MCP inputs

   - **Create Bullet List**  
     - Name: `Create Bullet List in a document in Google Docs`  
     - Action: create paragraphBullets (default preset)  
     - Start/end index from MCP inputs

   - **Create Numbered List**  
     - Name: `Create Numbered List in a document in Google Docs`  
     - Action: create paragraphBullets with bulletPreset `NUMBERED_DECIMAL_NESTED`  
     - Start/end index from MCP inputs

   - **Insert Empty Table at End**  
     - Name: `Insert Empty Table to the end of a document in Google Docs1`  
     - Action: insert table with rows and columns specified by MCP inputs

   - **Insert Empty Table Using Index**  
     - Name: `Insert Empty Table to a document in Google Docs using Index`  
     - Action: insert table at index with rows and columns from MCP inputs

   - **Insert Page Break Using Index**  
     - Name: `Insert Page Break to a document in Google Docs using Index`  
     - Action: insert pageBreak at index from MCP input

   - **Insert Text at End**  
     - Name: `Insert Text to the end of a document in Google Docs`  
     - Action: insert text at end with text from MCP input

   - **Insert Text Using Index**  
     - Name: `Insert Text to a document in Google Docs using Index`  
     - Action: insert text at index with text and index from MCP inputs

   - Credentials: Google Docs OAuth2 for all above nodes  
   - Document URL and Revision ID: dynamically set from MCP inputs for concurrency control  
   - Connect all these nodes’ input (`ai_tool`) from MCP Trigger node

6. **Add Sticky Notes**  
   - Create three sticky note nodes with the content provided in the original workflow, positioned near MCP Trigger for documentation purposes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This MCP server enables coding agents like Claude, ChatGPT (with MCP support), Clawed Code, and others to programmatically read, write, and format Google Docs documents via structured MCP requests.                         | Inline workflow documentation (Sticky Note)                     |
| The workflow solves the problem where coding agents can find Google Docs files via Drive search but cannot reliably open or edit them, by exposing Google Docs editing operations via MCP.                                   | Inline workflow documentation (Sticky Note1)                    |
| Workflow steps: 1) Agent sends MCP request; 2) MCP trigger receives structured input; 3) Google Drive and Docs nodes execute requested operations; 4) Results returned to agent in structured MCP response.                     | Inline workflow documentation (Sticky Note2)                    |
| Folder ID `1L3CaQB_khbHGb4HZO-qyDDQCvZfut3jY` is hardcoded for new document creation; update to your own folder ID as needed.                                                                                                | Configuration detail                                             |
| Google Docs API concurrency requires using `Revision_ID` for write control to avoid update conflicts.                                                                                                                        | Google Docs API best practice                                    |
| This workflow requires Google Drive and Google Docs OAuth2 credentials with appropriate scopes configured in n8n.                                                                                                           | Credential setup                                                 |
| MCP Trigger node requires Langchain MCP nodes installed in n8n.                                                                                                                                                              | Node requirement                                                 |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.