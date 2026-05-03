🛠️ PagerDuty Tool MCP Server 💪 all 9 operations

https://n8nworkflows.xyz/workflows/----pagerduty-tool-mcp-server----all-9-operations-5343


# 🛠️ PagerDuty Tool MCP Server 💪 all 9 operations

### 1. Workflow Overview

This workflow, titled **🛠️ PagerDuty Tool MCP Server**, functions as a multi-communication platform (MCP) server that exposes a comprehensive set of PagerDuty operations via an n8n MCP trigger node. It facilitates 9 distinct PagerDuty API operations, enabling AI agents or external clients to interact programmatically with PagerDuty incidents, incident notes, log entries, and user data. The workflow is designed for seamless integration with AI-driven automation, where parameters are dynamically populated through AI expressions.

**Logical blocks:**

- 1.1 MCP Trigger and Overview  
  The entry point accepting incoming requests and an overview sticky note describing the workflow and usage instructions.

- 1.2 Incident Management  
  Nodes handling creation, retrieval (single and multiple), and updating of PagerDuty incidents.

- 1.3 Incident Note Management  
  Nodes managing creation and retrieval of incident notes associated with incidents.

- 1.4 Log Entry Retrieval  
  Nodes for fetching single or multiple log entries related to PagerDuty incidents.

- 1.5 User Retrieval  
  Node for fetching user information by user ID.

Each node in these blocks connects back to the MCP trigger node, which manages the API interactions and AI-driven parameter injection.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger and Overview

- **Overview:**  
  This block initializes the workflow by accepting incoming MCP requests on a webhook and provides a detailed sticky note outlining the workflow's purpose, available operations, and setup instructions.

- **Nodes Involved:**  
  - PagerDuty Tool MCP Server (MCP trigger)  
  - Workflow Overview 0 (sticky note)

- **Node Details:**

  - **PagerDuty Tool MCP Server**  
    - Type: MCP Trigger node (@n8n/n8n-nodes-langchain.mcpTrigger)  
    - Role: Entry point webhook for MCP server requests  
    - Configuration: Webhook path set to "pagerduty-tool-mcp"  
    - Inputs: External HTTP requests from clients or AI agents  
    - Outputs: Connected downstream to all operation nodes via ai_tool connection  
    - Notes: Requires MCP setup, webhook URL use for AI agent integration  
    - Edge cases: Webhook not reachable, invalid requests, authentication failures

  - **Workflow Overview 0 (Sticky Note)**  
    - Type: Sticky note, documentation node  
    - Role: Provides user-facing overview, instructions, and feature list  
    - Configuration: Large note with markdown content describing the workflow, operations, setup, and support links  
    - Inputs/Outputs: None (informational only)  
    - Edge cases: None (static content)

#### 1.2 Incident Management

- **Overview:**  
  This block manages all incident-related PagerDuty operations: creating new incidents, retrieving one or many incidents, and updating existing incidents. It allows dynamic population of parameters via AI expressions.

- **Nodes Involved:**  
  - Create an incident  
  - Get an incident  
  - Get many incidents  
  - Update an incident  
  - Sticky Note 1 ("Incident")

- **Node Details:**

  - **Create an incident**  
    - Type: PagerDuty Tool node  
    - Role: Creates a new incident in PagerDuty  
    - Configuration:  
      - Resource: "incident"  
      - Operation: "create"  
      - Parameters dynamically filled from AI: Email, Title, Service_Id  
      - Authentication: API Token (configured once in one node, reused)  
      - DescriptionType and conferenceBridge set to auto/default  
    - Inputs: MCP trigger node (ai_tool connection)  
    - Outputs: Incident creation response  
    - Edge cases: Invalid or missing service IDs, API token auth failures, invalid email format, empty required fields

  - **Get an incident**  
    - Type: PagerDuty Tool node  
    - Role: Retrieves a specific incident by ID  
    - Configuration:  
      - Operation: "get"  
      - IncidentId from AI parameter  
    - Inputs: MCP trigger node  
    - Outputs: Incident data  
    - Edge cases: Incident ID not found, auth errors, malformed ID

  - **Get many incidents**  
    - Type: PagerDuty Tool node  
    - Role: Retrieves multiple incidents with optional limit or returnAll flag  
    - Configuration:  
      - Operation: "getAll"  
      - Limit and Return_All dynamically from AI  
    - Inputs: MCP trigger node  
    - Outputs: List of incidents  
    - Edge cases: Invalid limit values, paging errors, auth failures

  - **Update an incident**  
    - Type: PagerDuty Tool node  
    - Role: Updates an incident identified by Incident_Id  
    - Configuration:  
      - Operation: "update"  
      - IncidentId, Email dynamically from AI  
      - UpdateFields empty by default but modifiable  
      - Auth: API Token  
    - Inputs: MCP trigger node  
    - Outputs: Update response  
    - Edge cases: Incident not found, invalid update data, auth issues

  - **Sticky Note 1 ("Incident")**  
    - Type: Sticky note  
    - Role: Visually groups and labels the incident-related nodes  
    - Inputs/Outputs: None

#### 1.3 Incident Note Management

- **Overview:**  
  Handles creation of incident notes and retrieval of multiple incident notes linked to specific incidents.

- **Nodes Involved:**  
  - Create an incident note  
  - Get many incident notes  
  - Sticky Note 2 ("Incidentnote")

- **Node Details:**

  - **Create an incident note**  
    - Type: PagerDuty Tool node  
    - Role: Adds a note to an incident  
    - Configuration:  
      - Resource: "incidentNote"  
      - Operation: "create"  
      - Parameters from AI: Email, Content, Incident_Id  
    - Inputs: MCP trigger node  
    - Outputs: Note creation response  
    - Edge cases: Incident not found, empty content, auth errors

  - **Get many incident notes**  
    - Type: PagerDuty Tool node  
    - Role: Retrieves multiple notes for an incident  
    - Configuration:  
      - Operation: "getAll"  
      - Limit, Return_All, Incident_Id from AI  
    - Inputs: MCP trigger node  
    - Outputs: List of notes  
    - Edge cases: Invalid incident ID, paging issues, auth failures

  - **Sticky Note 2 ("Incidentnote")**  
    - Type: Sticky note  
    - Role: Visual grouping for incident note operations

#### 1.4 Log Entry Retrieval

- **Overview:**  
  Retrieves details about PagerDuty log entries either individually or in bulk.

- **Nodes Involved:**  
  - Get a log entry  
  - Get many log entries  
  - Sticky Note 3 ("Logentry")

- **Node Details:**

  - **Get a log entry**  
    - Type: PagerDuty Tool node  
    - Role: Fetches a specific log entry by ID  
    - Configuration:  
      - Resource: "logEntry"  
      - logEntryId from AI  
    - Inputs: MCP trigger node  
    - Outputs: Single log entry data  
    - Edge cases: Invalid or missing log entry ID, auth errors

  - **Get many log entries**  
    - Type: PagerDuty Tool node  
    - Role: Fetches multiple log entries with limit and returnAll flags  
    - Configuration:  
      - Operation: "getAll"  
      - Parameters from AI: Limit, Return_All  
    - Inputs: MCP trigger node  
    - Outputs: List of log entries  
    - Edge cases: Invalid limits, paging, auth failures

  - **Sticky Note 3 ("Logentry")**  
    - Type: Sticky note  
    - Role: Visual label for log entry nodes

#### 1.5 User Retrieval

- **Overview:**  
  Retrieves PagerDuty user details by user ID.

- **Nodes Involved:**  
  - Get a user  
  - Sticky Note 4 ("User")

- **Node Details:**

  - **Get a user**  
    - Type: PagerDuty Tool node  
    - Role: Retrieves user information by user ID  
    - Configuration:  
      - Resource: "user"  
      - userId from AI  
    - Inputs: MCP trigger node  
    - Outputs: User data  
    - Edge cases: User ID not found, auth errors

  - **Sticky Note 4 ("User")**  
    - Type: Sticky note  
    - Role: Visual grouping for user retrieval

---

### 3. Summary Table

| Node Name            | Node Type                  | Functional Role                     | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                   |
|----------------------|----------------------------|-----------------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0  | Sticky Note                | Workflow documentation and setup  | None                  | None                  | ## 🛠️ PagerDuty Tool MCP Server - Available 9 operations, setup instructions, support links                   |
| PagerDuty Tool MCP Server | MCP Trigger (@n8n/n8n-nodes-langchain.mcpTrigger) | Entry webhook for MCP requests    | External HTTP/Webhook  | All PagerDuty tool nodes (ai_tool connection) |                                                                                                               |
| Create an incident    | PagerDuty Tool             | Create new PagerDuty incident     | MCP Trigger           | None                  | ## Incident                                                                                                   |
| Get an incident       | PagerDuty Tool             | Retrieve a single incident        | MCP Trigger           | None                  | ## Incident                                                                                                   |
| Get many incidents    | PagerDuty Tool             | Retrieve multiple incidents       | MCP Trigger           | None                  | ## Incident                                                                                                   |
| Update an incident    | PagerDuty Tool             | Update an existing incident       | MCP Trigger           | None                  | ## Incident                                                                                                   |
| Sticky Note 1         | Sticky Note                | Visual label for Incident block   | None                  | None                  | ## Incident                                                                                                   |
| Create an incident note | PagerDuty Tool           | Create a note for an incident     | MCP Trigger           | None                  | ## Incidentnote                                                                                               |
| Get many incident notes | PagerDuty Tool           | Retrieve multiple incident notes  | MCP Trigger           | None                  | ## Incidentnote                                                                                               |
| Sticky Note 2         | Sticky Note                | Visual label for Incidentnote block | None                  | None                  | ## Incidentnote                                                                                               |
| Get a log entry       | PagerDuty Tool             | Retrieve a specific log entry     | MCP Trigger           | None                  | ## Logentry                                                                                                   |
| Get many log entries  | PagerDuty Tool             | Retrieve multiple log entries     | MCP Trigger           | None                  | ## Logentry                                                                                                   |
| Sticky Note 3         | Sticky Note                | Visual label for Logentry block   | None                  | None                  | ## Logentry                                                                                                   |
| Get a user            | PagerDuty Tool             | Retrieve user information         | MCP Trigger           | None                  | ## User                                                                                                       |
| Sticky Note 4         | Sticky Note                | Visual label for User block       | None                  | None                  | ## User                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Name: "PagerDuty Tool MCP Server"  
   - Type: "@n8n/n8n-nodes-langchain.mcpTrigger"  
   - Parameters: Set Webhook Path to `pagerduty-tool-mcp`  
   - Save credentials later after creating one PagerDuty node  
   - Position: Center top (e.g., [-400, -100])

2. **Add Sticky Note: Workflow Overview**  
   - Name: "Workflow Overview 0"  
   - Content: Markdown describing the workflow, operations, setup instructions, and support links (refer to Section 1.1)  
   - Size: Width 420, Height 840  
   - Position: Left side (e.g., [-1460, 120])

3. **Create Incident Management Nodes**  
   - Create four PagerDuty Tool nodes for incidents:  
     a. "Create an incident"  
        - Resource: incident  
        - Operation: create  
        - Parameters:  
          - Email: `{{$fromAI('Email','', 'string')}}`  
          - Title: `{{$fromAI('Title','', 'string')}}`  
          - ServiceId: `{{$fromAI('Service_Id','', 'string')}}`  
        - Authentication: apiToken  
        - DescriptionType: auto  
     b. "Get an incident"  
        - Operation: get  
        - IncidentId: `{{$fromAI('Incident_Id','', 'string')}}`  
     c. "Get many incidents"  
        - Operation: getAll  
        - Limit: `{{$fromAI('Limit','', 'number')}}`  
        - ReturnAll: `{{$fromAI('Return_All','', 'boolean')}}`  
     d. "Update an incident"  
        - Operation: update  
        - IncidentId: `{{$fromAI('Incident_Id','', 'string')}}`  
        - Email: `{{$fromAI('Email','', 'string')}}`  
        - UpdateFields: empty object by default  
        - Authentication: apiToken  
   - Position nodes horizontally near each other (e.g., [-800,140], [-580,140], [-360,140], [-140,140])  
   - Connect each node's input to the MCP trigger node with ai_tool connection

4. **Add Sticky Note for Incident Block**  
   - Name: "Sticky Note 1"  
   - Content: "## Incident"  
   - Position: Left of incident nodes (e.g., [-1000, 120])  
   - Color: 4 (blue)

5. **Create Incident Note Management Nodes**  
   - Two PagerDuty Tool nodes:  
     a. "Create an incident note"  
        - Resource: incidentNote  
        - Operation: create  
        - Parameters:  
          - Email: `{{$fromAI('Email','', 'string')}}`  
          - Content: `{{$fromAI('Content','', 'string')}}`  
          - IncidentId: `{{$fromAI('Incident_Id','', 'string')}}`  
     b. "Get many incident notes"  
        - Operation: getAll  
        - Limit: `{{$fromAI('Limit','', 'number')}}`  
        - ReturnAll: `{{$fromAI('Return_All','', 'boolean')}}`  
        - IncidentId: `{{$fromAI('Incident_Id','', 'string')}}`  
   - Position near each other (e.g., [-800,380], [-580,380])  
   - Connect each input to MCP trigger node

6. **Add Sticky Note for Incidentnote Block**  
   - Name: "Sticky Note 2"  
   - Content: "## Incidentnote"  
   - Position: Left of incident note nodes (e.g., [-1000, 360])  
   - Color: 5 (purple)

7. **Create Log Entry Retrieval Nodes**  
   - Two PagerDuty Tool nodes:  
     a. "Get a log entry"  
        - Resource: logEntry  
        - logEntryId: `{{$fromAI('Log_Entry_Id','', 'string')}}`  
     b. "Get many log entries"  
        - Operation: getAll  
        - Limit: `{{$fromAI('Limit','', 'number')}}`  
        - ReturnAll: `{{$fromAI('Return_All','', 'boolean')}}`  
   - Position horizontally (e.g., [-800,620], [-580,620])  
   - Connect inputs to MCP trigger node

8. **Add Sticky Note for Logentry Block**  
   - Name: "Sticky Note 3"  
   - Content: "## Logentry"  
   - Position: Left of log entry nodes (e.g., [-1000, 600])  
   - Color: 6 (orange)

9. **Create User Retrieval Node**  
   - One PagerDuty Tool node:  
     - Name: "Get a user"  
     - Resource: user  
     - userId: `{{$fromAI('User_Id','', 'string')}}`  
   - Position: (e.g., [-800,860])  
   - Connect input to MCP trigger node

10. **Add Sticky Note for User Block**  
    - Name: "Sticky Note 4"  
    - Content: "## User"  
    - Position: Left of user node (e.g., [-1000, 840])  
    - Color: 7 (green)

11. **Configure Credentials**  
    - In one PagerDuty Tool node (e.g., "Create an incident"), add PagerDuty API Token credentials using the n8n credentials manager  
    - Open and close other PagerDuty nodes to inherit the credential configuration automatically

12. **Save and Activate Workflow**  
    - Save workflow  
    - Activate to enable webhook trigger  
    - Copy webhook URL from "PagerDuty Tool MCP Server" node for external use or AI agent configuration

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Zero configuration - all 9 PagerDuty operations pre-built, ready to use with AI agents automatically populating parameters              | Workflow Overview sticky note content                                                                             |
| For setup and customization help, consult [n8n MCP documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) | Workflow Overview sticky note                                                                                     |
| Direct Discord support available at https://discord.me/cfomodz for MCP integration guidance                                              | Workflow Overview sticky note                                                                                     |
| PagerDuty Tool node requires valid API Token credential setup once per workflow                                                         | Credential configuration section                                                                                  |
| Use the webhook URL from MCP trigger node as endpoint for AI agents or external applications to invoke PagerDuty operations             | MCP Trigger node configuration                                                                                     |

---

**Disclaimer:** The provided documentation exclusively reflects the analyzed n8n workflow automation and complies with all applicable content policies. It contains no illegal, offensive, or copyrighted materials and operates only on public or authorized data.