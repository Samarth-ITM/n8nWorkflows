🛠️ Clearbit Tool MCP Server

https://n8nworkflows.xyz/workflows/----clearbit-tool-mcp-server-5321


# 🛠️ Clearbit Tool MCP Server

---

### 1. Workflow Overview

This workflow, titled **🛠️ Clearbit Tool MCP Server**, acts as a server providing three main Clearbit API operations via a single MCP (Multi-Channel Platform) trigger. It is designed for integration with AI agents, enabling them to call Clearbit’s company and person enrichment or autocomplete services seamlessly.

**Target Use Cases:**
- AI-powered applications needing real-time company autocomplete suggestions.
- Enriching company data by domain.
- Enriching person data by email address.

**Logical Blocks:**

- **1.1 MCP Trigger Input:**  
  Receives requests from AI agents via a webhook, serving as the entry point.

- **1.2 Company Operations:**  
  Handles Clearbit company-related API calls:
  - Autocomplete a company by name.
  - Enrich company data by domain.

- **1.3 Person Operations:**  
  Handles Clearbit person data enrichment by email.

- **1.4 Documentation and Setup Guidance (Sticky Notes):**  
  Provides embedded instructions and contextual notes for users configuring or maintaining the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger Input

- **Overview:**  
  This block listens for incoming HTTP requests from AI agents or other clients, triggering the workflow and routing requests to the appropriate Clearbit operation.

- **Nodes Involved:**  
  - Clearbit Tool MCP Server

- **Node Details:**

  - **Clearbit Tool MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Entry trigger node for MCP server requests; listens on a webhook path.  
    - Configuration:  
      - Webhook path set to `"clearbit-tool-mcp"`.  
      - Automatically routes input to downstream Clearbit tool nodes based on the requested operation.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to all Clearbit tool nodes handling specific operations.  
    - Requirements: Requires the workflow to be activated to listen for incoming requests.  
    - Edge Cases:  
      - Webhook URL misconfiguration or network issues prevent triggering.  
      - Invalid or malformed incoming request payloads.  
      - Authentication failure if Clearbit credentials are missing or invalid (propagated downstream).  
    - Sub-workflow: None.

#### 1.2 Company Operations

- **Overview:**  
  This block executes Clearbit’s company autocomplete and enrichment operations based on input parameters provided by the AI agent.

- **Nodes Involved:**  
  - Autocomplete a company  
  - Enrich a company  
  - Sticky Note 1 ("Company")

- **Node Details:**

  - **Autocomplete a company**  
    - Type: `n8n-nodes-base.clearbitTool`  
    - Role: Calls Clearbit API to autocomplete company names based on partial input.  
    - Configuration:  
      - Operation set to `"autocomplete"`.  
      - Parameter `"name"` dynamically populated from AI input via expression: `{{$fromAI('Name', ``, 'string')}}`.  
      - Credentials: Uses Clearbit API credentials (user must set credential ID).  
    - Inputs: Connected downstream from MCP trigger.  
    - Outputs: Provides autocomplete suggestions for companies.  
    - Edge Cases:  
      - Empty or invalid "Name" input leads to no results or errors.  
      - API rate limits or credential errors.  
      - Expression failure if `$fromAI()` input key missing.  

  - **Enrich a company**  
    - Type: `n8n-nodes-base.clearbitTool`  
    - Role: Calls Clearbit API to enrich company data given a domain.  
    - Configuration:  
      - Operation defaults to `"enrich"`.  
      - Parameter `"domain"` dynamically populated via `$fromAI('Domain', '', 'string')`.  
      - Additional fields empty (default).  
      - Credentials: Same Clearbit API credentials as above.  
    - Inputs: Connected downstream from MCP trigger.  
    - Outputs: Provides enriched company information.  
    - Edge Cases:  
      - Missing or invalid domain strings cause errors or empty results.  
      - Credential or network issues.  
      - Expression failures on AI input.  

  - **Sticky Note 1 ("Company")**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Provides a visual label grouping the company-related nodes for clarity.  
    - No technical role in execution.

#### 1.3 Person Operations

- **Overview:**  
  This block handles enrichment of person data via Clearbit, using an email address provided by the AI agent.

- **Nodes Involved:**  
  - Enrich a person  
  - Sticky Note 2 ("Person")

- **Node Details:**

  - **Enrich a person**  
    - Type: `n8n-nodes-base.clearbitTool`  
    - Role: Calls Clearbit API to enrich person data by email.  
    - Configuration:  
      - Operation: `"enrich"` (resource `"person"` explicitly set).  
      - Parameter `"email"` dynamically populated via `$fromAI('Email', '', 'string')`.  
      - Additional fields empty.  
      - Credentials: Same Clearbit API credentials as other tool nodes (user must configure).  
    - Inputs: Connected downstream from MCP trigger.  
    - Outputs: Provides enriched person information.  
    - Edge Cases:  
      - Missing or invalid email input causes errors or no data.  
      - Credential or API limit issues.  
      - Expression failures on AI input.  

  - **Sticky Note 2 ("Person")**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Visual label for person-related nodes.  

#### 1.4 Documentation and Setup Guidance

- **Overview:**  
  Provides detailed textual instructions and overview of the workflow’s features, usage, and setup steps to assist users in configuring and using the workflow.

- **Nodes Involved:**  
  - Workflow Overview 0 (sticky note)

- **Node Details:**

  - **Workflow Overview 0**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content:  
      - Lists available operations supported: company autocomplete, company enrich, person enrich.  
      - Provides step-by-step setup instructions: import, credential setup, activation, webhook URL retrieval, and AI agent connection.  
      - Highlights features such as zero configuration, AI parameter binding via `$fromAI()`, full operation coverage, error handling, and customization.  
      - Contains helpful links to official n8n documentation and Discord support.  
    - No execution role; purely informational.

---

### 3. Summary Table

| Node Name               | Node Type                               | Functional Role            | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                              |
|-------------------------|---------------------------------------|----------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow Overview 0     | stickyNote                            | Setup instructions & overview | None                        | None                        | ## 🛠️ Clearbit Tool MCP Server ... See full instructions and links in the node content (too long to fit here) |
| Clearbit Tool MCP Server | @n8n/n8n-nodes-langchain.mcpTrigger | Entry point / triggers workflow | None                        | Autocomplete a company, Enrich a company, Enrich a person |                                                                                                        |
| Autocomplete a company  | clearbitTool                         | Company autocomplete API call | Clearbit Tool MCP Server    | None                        |                                                                                                        |
| Enrich a company        | clearbitTool                         | Company enrichment API call  | Clearbit Tool MCP Server    | None                        |                                                                                                        |
| Sticky Note 1           | stickyNote                          | Visual label for company nodes | None                        | None                        | ## Company                                                                                             |
| Enrich a person         | clearbitTool                         | Person enrichment API call   | Clearbit Tool MCP Server    | None                        |                                                                                                        |
| Sticky Note 2           | stickyNote                          | Visual label for person nodes  | None                        | None                        | ## Person                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add node: Type `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name it `Clearbit Tool MCP Server`  
   - Set webhook path to `clearbit-tool-mcp` (or any preferred unique path)  
   - No credentials needed here  
   - This node initiates the workflow on incoming HTTP requests.

2. **Add Clearbit Credentials**  
   - In n8n credentials, add new credentials of type `Clearbit API`  
   - Provide your Clearbit API key  
   - Name the credential (example: `Clearbit Credential`)

3. **Add Company Autocomplete Node**  
   - Add node: Type `Clearbit Tool` (`n8n-nodes-base.clearbitTool`)  
   - Name it `Autocomplete a company`  
   - Set operation to `autocomplete`  
   - Set parameter `name` to use expression: `{{$fromAI('Name', '', 'string')}}`  
   - Assign Clearbit credentials created in step 2  
   - Connect input from `Clearbit Tool MCP Server` node

4. **Add Company Enrichment Node**  
   - Add node: Type `Clearbit Tool`  
   - Name it `Enrich a company`  
   - Operation defaults to `enrich` for company resource  
   - Set parameter `domain` via expression: `{{$fromAI('Domain', '', 'string')}}`  
   - Leave additional fields empty/default  
   - Assign Clearbit credentials  
   - Connect input from `Clearbit Tool MCP Server` node

5. **Add Person Enrichment Node**  
   - Add node: Type `Clearbit Tool`  
   - Name it `Enrich a person`  
   - Set resource explicitly to `person`  
   - Operation: `enrich`  
   - Set parameter `email` via expression: `{{$fromAI('Email', '', 'string')}}`  
   - Leave additional fields empty/default  
   - Assign Clearbit credentials  
   - Connect input from `Clearbit Tool MCP Server` node

6. **Add Sticky Notes for Clarity (Optional)**  
   - Add sticky notes to visually group company and person nodes  
   - Example:  
     - Sticky Note labeled "Company" near company nodes  
     - Sticky Note labeled "Person" near person node  
     - Sticky Note labeled "Workflow Overview" with setup instructions for user guidance

7. **Activate the Workflow**  
   - Save and activate the workflow to enable webhook listening  
   - Copy webhook URL from `Clearbit Tool MCP Server` node to integrate with AI agents or clients

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow supports 3 Clearbit operations: company autocomplete, company enrich, person enrich                    | Workflow overview sticky note                                                                   |
| AI agents populate parameters using `$fromAI()` expressions automatically                                       | Enables zero-configuration integration with AI agents                                          |
| Setup instructions include importing workflow, adding credentials, activating workflow, and copying webhook URL | Included in Workflow Overview sticky note                                                      |
| Official n8n MCP documentation for tool node integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Useful for customizing and troubleshooting MCP triggers                                        |
| Discord support for MCP integration and customizations: https://discord.me/cfomodz                             | Community and developer support channel                                                        |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created using n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.

---