Let AI Agents Get Campaigns with 🛠️ Google Ads Tool MCP Server

https://n8nworkflows.xyz/workflows/let-ai-agents-get-campaigns-with-----google-ads-tool-mcp-server-5262


# Let AI Agents Get Campaigns with 🛠️ Google Ads Tool MCP Server

### 1. Workflow Overview

This workflow, titled **"Google Ads Tool MCP Server"**, serves as a minimal control point (MCP) server designed to expose Google Ads campaign operations via an AI agent-friendly interface. It targets use cases where AI agents or external systems need to dynamically request Google Ads campaign information without manual API calls or complex configuration. The workflow logically divides into two main blocks:

- **1.1 MCP Server Setup & Trigger:** Handles incoming requests from AI agents or external clients using an MCP trigger node and manages authentication and routing.
- **1.2 Google Ads Campaign Operations:** Executes Google Ads API calls to either retrieve all campaigns or get details of a specific campaign based on parameters dynamically populated by AI expressions.

The workflow includes built-in zero-configuration support for two campaign-related operations ("get all campaigns" and "get a campaign") with parameters automatically injected by the AI agents using `$fromAI()` expressions. It also features native n8n error handling, response formatting, and minimal user setup requirements.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Setup & Trigger

**Overview:**  
This block listens for incoming HTTP webhook requests that represent AI agent calls. It acts as the entry point and dispatcher for Google Ads campaign operations.

**Nodes Involved:**  
- Google Ads Tool MCP Server (MCP Trigger node)  
- Workflow Overview 0 (Sticky Note)

**Node Details:**

- **Google Ads Tool MCP Server**  
  - *Type:* `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - *Role:* MCP server trigger node designed to receive and handle AI agent requests at a specific webhook path.  
  - *Configuration:*  
    - Path set to `"google-ads-tool-mcp"` for webhook access.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connected as an AI tool to downstream Google Ads Tool nodes for operation execution.  
  - *Version Requirements:* Requires n8n version supporting MCP nodes and LangChain integration.  
  - *Edge Cases / Failure Modes:*  
    - Webhook URL misconfiguration may cause unreachable endpoint.  
    - Authentication issues on downstream Google Ads Tool nodes propagate errors.  
    - Invalid or malformed AI agent requests may cause runtime errors or empty responses.  
  - *Sub-workflow:* None.

- **Workflow Overview 0 (Sticky Note)**  
  - Contains detailed instructions on workflow usage, setup, and feature summary.  
  - Serves as documentation within the workflow for users.  
  - No technical processing role.

---

#### 2.2 Google Ads Campaign Operations

**Overview:**  
This block executes the core Google Ads operations: retrieving multiple campaigns or a specific campaign. It uses Google Ads Tool nodes configured to accept parameters dynamically via AI expressions.

**Nodes Involved:**  
- Get many campaigns (Google Ads Tool node)  
- Get a campaign (Google Ads Tool node)  
- Sticky Note 1 (Sticky Note labeled "Campaign")

**Node Details:**

- **Get many campaigns**  
  - *Type:* `n8n-nodes-base.googleAdsTool`  
  - *Role:* Retrieves a list of campaigns from Google Ads based on AI-provided parameters.  
  - *Configuration:*  
    - Operation defaults to "get all" campaigns implicitly (no explicit operation parameter set).  
    - Parameters injected dynamically via `$fromAI()` expressions:  
      - `campaigsNotice` (likely a typo, expected "campaignsNotice")  
      - `clientCustomerId`  
      - `managerCustomerId`  
    - Empty `requestOptions` and `additionalOptions` objects for extensibility.  
    - Credential: Configured with Google Ads OAuth2 credentials (placeholder ID to be replaced).  
  - *Inputs:* Receives AI tool requests from MCP Server node.  
  - *Outputs:* Returns campaign list data to MCP node for response.  
  - *Version Requirements:* Requires Google Ads Tool node and OAuth2 credentials setup.  
  - *Edge Cases / Failure Modes:*  
    - OAuth2 token expiration or invalid credentials cause auth errors.  
    - Missing or malformed client or manager customer IDs cause API call failures.  
    - Network timeouts or Google Ads API rate limits may interrupt operation.  
  - *Sub-workflow:* None.

- **Get a campaign**  
  - *Type:* `n8n-nodes-base.googleAdsTool`  
  - *Role:* Fetches details of a single campaign by campaign ID.  
  - *Configuration:*  
    - Operation explicitly set to `"get"`.  
    - Parameters injected dynamically via `$fromAI()` expressions:  
      - `campaignId` (required for getting a specific campaign)  
      - `campaigsNotice` (likely a typo)  
      - `clientCustomerId`  
      - `managerCustomerId`  
    - Empty `requestOptions`.  
    - Credential: Same Google Ads OAuth2 credential as "Get many campaigns" node.  
  - *Inputs:* Receives AI tool requests from MCP Server node.  
  - *Outputs:* Returns single campaign data to MCP node for response.  
  - *Version Requirements:* Same as above.  
  - *Edge Cases / Failure Modes:*  
    - Missing or invalid `campaignId` results in not found or error response.  
    - Same authentication and API limitations as above.  
  - *Sub-workflow:* None.

- **Sticky Note 1**  
  - Label: "Campaign"  
  - Provides a visual grouping and clarity for the campaign-related operations block.

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                               | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                   |
|-------------------------|------------------------------------|-----------------------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0      | Sticky Note                        | Provides workflow documentation and instructions | None                     | None                    | ## 🛠️ Google Ads Tool MCP Server... See detailed setup & usage instructions including links to n8n docs and Discord support. |
| Google Ads Tool MCP Server | MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`) | Entry point for AI agent requests, MCP server webhook | None                     | Get many campaigns, Get a campaign | See Workflow Overview 0 for setup details.                                                                                     |
| Get many campaigns       | Google Ads Tool (`n8n-nodes-base.googleAdsTool`) | Retrieves list of Google Ads campaigns         | Google Ads Tool MCP Server | None                    | Grouped under "Campaign" block (Sticky Note 1).                                                                                |
| Get a campaign          | Google Ads Tool (`n8n-nodes-base.googleAdsTool`) | Retrieves a single Google Ads campaign by ID   | Google Ads Tool MCP Server | None                    | Grouped under "Campaign" block (Sticky Note 1).                                                                                |
| Sticky Note 1            | Sticky Note                        | Visual grouping for campaign operations         | None                     | None                    | Label: "Campaign"                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note "Workflow Overview 0"**  
   - Add a Sticky Note node.  
   - Set width: 420, height: 760.  
   - Paste content with detailed workflow description, setup instructions, and helpful links as per the original note.  
   - Position it on the far left side for documentation.

2. **Add MCP Trigger Node "Google Ads Tool MCP Server"**  
   - Node type: `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Configure webhook path: `google-ads-tool-mcp`.  
   - No credentials needed here.  
   - Position it right of the Workflow Overview note.

3. **Add Google Ads Tool Node "Get many campaigns"**  
   - Node type: `n8n-nodes-base.googleAdsTool`.  
   - No explicit operation set, defaults to "get all campaigns".  
   - Configure parameters with expressions:  
     - `campaigsNotice`: `={{ $fromAI('Campaigs_Notice', ``, 'string') }}`  
     - `clientCustomerId`: `={{ $fromAI('Client_Customer_Id', ``, 'string') }}`  
     - `managerCustomerId`: `={{ $fromAI('Manager_Customer_Id', ``, 'string') }}`  
   - Leave `requestOptions` and `additionalOptions` empty.  
   - Set Google Ads OAuth2 credentials (create or select OAuth2 credential with appropriate scopes).  
   - Position below MCP Trigger node, to the left.

4. **Add Google Ads Tool Node "Get a campaign"**  
   - Node type: `n8n-nodes-base.googleAdsTool`.  
   - Set operation to `"get"`.  
   - Configure parameters with expressions:  
     - `campaignId`: `={{ $fromAI('Campaign_Id', ``, 'string') }}`  
     - `campaigsNotice`: same as above.  
     - `clientCustomerId`: same as above.  
     - `managerCustomerId`: same as above.  
   - Leave `requestOptions` empty.  
   - Use same Google Ads OAuth2 credentials as previous node.  
   - Position to the right of "Get many campaigns" node.

5. **Connect the MCP Trigger node to both Google Ads Tool nodes**  
   - Connect output of "Google Ads Tool MCP Server" to input of both "Get many campaigns" and "Get a campaign" nodes using the AI tool connection type.

6. **Add Sticky Note "Campaign"**  
   - Add a Sticky Note node near campaign-related nodes.  
   - Set content to "Campaign".  
   - Adjust size and position for visual grouping.

7. **Credential Setup**  
   - Create Google Ads OAuth2 credentials in n8n with required scopes to access campaigns.  
   - Replace placeholder credential ID in both Google Ads Tool nodes with your actual credential.

8. **Activate Workflow**  
   - Save and activate the workflow.  
   - Copy the webhook URL from the MCP Trigger node to use it as the AI agent endpoint.

9. **Usage**  
   - AI agents or external systems call the webhook URL with JSON requests containing parameters like `Campaign_Id`, `Client_Customer_Id`, etc., to invoke campaign operations seamlessly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Workflow supports two Google Ads campaign operations: get all campaigns and get a specific campaign.                          | Functional scope of the workflow.                                                                                   |
| AI agent parameters are injected using `$fromAI()` expressions, enabling zero-configuration dynamic input.                     | Enables seamless AI integration.                                                                                    |
| For detailed MCP integration documentation, see [n8n MCP Node Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) | Official n8n documentation for MCP nodes.                                                                           |
| For community support and customization help, join [n8n Discord](https://discord.me/cfomodz).                                   | Community support channel.                                                                                          |
| Ensure Google Ads OAuth2 credentials are properly set up with necessary permissions to avoid authentication errors.             | Credential management best practice.                                                                                 |

---

**Disclaimer:**  
The provided descriptions and analysis are strictly derived from the automated n8n workflow JSON exported from a legal and publicly authorized environment. No illegal or offensive content is present.