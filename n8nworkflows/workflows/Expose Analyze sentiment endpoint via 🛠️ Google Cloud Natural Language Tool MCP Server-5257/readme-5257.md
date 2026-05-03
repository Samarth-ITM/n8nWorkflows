Expose Analyze sentiment endpoint via 🛠️ Google Cloud Natural Language Tool MCP Server

https://n8nworkflows.xyz/workflows/expose-analyze-sentiment-endpoint-via-----google-cloud-natural-language-tool-mcp-server-5257


# Expose Analyze sentiment endpoint via 🛠️ Google Cloud Natural Language Tool MCP Server

### 1. Workflow Overview

This workflow exposes an API endpoint via n8n’s MCP (Multi-Cluster Process) Server to perform sentiment analysis on text documents using the Google Cloud Natural Language Tool. It is designed to be integrated with AI agents or other services that require sentiment analysis capabilities via a webhook URL.

Logical blocks:

- **1.1 MCP Server Trigger**: Receives HTTP webhook requests and routes them into the workflow.
- **1.2 Sentiment Analysis Processing**: Uses the Google Cloud Natural Language Tool node to analyze sentiment of provided text.
- **1.3 Documentation and Configuration Notes**: Sticky notes providing usage instructions and operational overview.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger

- **Overview:**  
  This block listens for incoming HTTP requests on a defined webhook path, acting as the entry point for the sentiment analysis service. It supports MCP server features for AI agent integration and automatic parameter population.

- **Nodes Involved:**  
  - Google Cloud Natural Language Tool MCP Server (MCP Trigger)

- **Node Details:**  

  - **Google Cloud Natural Language Tool MCP Server**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.mcpTrigger` — MCP server trigger node that exposes a webhook endpoint for incoming requests.  
    - *Configuration:*  
      - Webhook path set to `google-cloud-natural-language-tool-mcp`.  
      - Webhook ID provided by n8n for managing the endpoint.  
    - *Expressions/Variables:* None explicitly used; the node accepts parameters dynamically from AI agents via `$fromAI()` expressions at runtime.  
    - *Input/Output:*  
      - Input: HTTP request from external client or AI agent.  
      - Output: Passes data to the "Analyze sentiment" node via the named connection `ai_tool`.  
    - *Version Requirements:* Requires n8n version supporting MCP triggers and LangChain nodes.  
    - *Potential Failures:*  
      - Webhook URL misconfiguration or path conflicts.  
      - Network or firewall issues blocking HTTP requests.  
      - Authentication errors if downstream credentials are invalid.  
    - *Sub-workflow:* This node is the main trigger and entry point, no sub-workflows invoked.

#### 1.2 Sentiment Analysis Processing

- **Overview:**  
  This block performs the sentiment analysis on the incoming document text using Google Cloud’s Natural Language API. It processes the request data and returns structured sentiment analysis results.

- **Nodes Involved:**  
  - Analyze sentiment (Google Cloud Natural Language Tool node)

- **Node Details:**  

  - **Analyze sentiment**  
    - *Type & Role:* `n8n-nodes-base.googleCloudNaturalLanguageTool` — Calls Google Cloud Natural Language API to analyze sentiment on provided text documents.  
    - *Configuration:*  
      - Default parameters used (no special options set).  
      - Credentials are linked via OAuth2 for Google Cloud Natural Language API access. The credential ID needs to be set by the user (`SET_YOUR_CREDENTIAL_ID_HERE`).  
    - *Expressions/Variables:* None hard-coded; parameters dynamically populated by MCP server trigger via `$fromAI()` expressions at runtime.  
    - *Input/Output:*  
      - Input: Receives input from the MCP trigger node under the connection named `ai_tool`.  
      - Output: Returns sentiment analysis result to the MCP server node, which formats the response.  
    - *Version Requirements:* Compatible with n8n versions supporting the Google Cloud Natural Language Tool node and OAuth2 credential flow.  
    - *Potential Failures:*  
      - Credential errors or expired tokens causing authentication failure.  
      - API rate limits or quota exceeded on Google Cloud side.  
      - Malformed input documents causing API errors.  
      - Network or timeout errors.  
    - *Sub-workflow:* None.

#### 1.3 Documentation and Configuration Notes

- **Overview:**  
  Provides users with instructions on workflow setup, available operations, and useful links. These are informational nodes that do not affect workflow execution.

- **Nodes Involved:**  
  - Workflow Overview 0 (Sticky Note)  
  - Sticky Note 1 (Sticky Note)

- **Node Details:**  

  - **Workflow Overview 0**  
    - *Type & Role:* `n8n-nodes-base.stickyNote` — Contains detailed setup instructions, feature descriptions, and helpful links for users deploying the workflow.  
    - *Configuration:* Large note with multi-step setup instructions including credential configuration, activation, and usage.  
    - *Input/Output:* None (informational only).  
    - *Potential Issues:* N/A.

  - **Sticky Note 1**  
    - *Type & Role:* `n8n-nodes-base.stickyNote` — A small note titled “Document” presumably indicating the type of resource analyzed.  
    - *Input/Output:* None.  
    - *Potential Issues:* N/A.

---

### 3. Summary Table

| Node Name                            | Node Type                                | Functional Role                      | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                                 |
|------------------------------------|----------------------------------------|------------------------------------|---------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0                 | Sticky Note                            | Setup instructions and overview    | None                            | None                            | ## 🛠️ Google Cloud Natural Language Tool MCP Server<br>### 📋 Available Operations (1 total)<br>**Document**: analyze sentiment<br>### ⚙️ Setup Instructions<br>1. Import Workflow<br>2. Add Credentials<br>3. Activate<br>4. Get URL<br>5. Connect AI agents<br>### ✨ Features<br>Zero config, AI parameter auto-population, error handling<br>### 💬 Help links: [n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/), [Discord](https://discord.me/cfomodz) |
| Google Cloud Natural Language Tool MCP Server | MCP Trigger                            | Webhook entry point for API        | None                            | Analyze sentiment               |                                                                                                                             |
| Analyze sentiment                  | Google Cloud Natural Language Tool Node | Performs sentiment analysis         | Google Cloud Natural Language Tool MCP Server | None                            |                                                                                                                             |
| Sticky Note 1                     | Sticky Note                            | Document label                     | None                            | None                            | ## Document                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Add node: Type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Set `path` parameter to `google-cloud-natural-language-tool-mcp`.  
   - This node will expose a webhook URL for external HTTP POST requests.  
   - No credentials needed for this node.  

2. **Create Google Cloud Natural Language Tool Node**  
   - Add node: Type `Google Cloud Natural Language Tool`.  
   - Operation: Default (Analyze Sentiment on Document resource).  
   - Configure OAuth2 credentials:  
     - Create Google Cloud OAuth2 credential with necessary scopes for Natural Language API.  
     - Assign this credential to the node under `googleCloudNaturalLanguageOAuth2Api`.  
   - Leave options empty or customize if needed (e.g., language hints).  

3. **Connect Nodes**  
   - Connect the MCP Server Trigger node’s output to the input of the Google Cloud Natural Language Tool node.  
   - Use the connection named `ai_tool` to ensure proper MCP handling.  

4. **Add Sticky Notes for Documentation (Optional)**  
   - Create a sticky note with setup instructions and usage notes as per the original workflow.  
   - Create a small sticky note labeled “Document” near the Google Cloud Natural Language Tool node for clarity.  

5. **Activate Workflow**  
   - Save and activate the workflow.  
   - Copy the webhook URL from the MCP Server Trigger node for external use.  

6. **Testing and Integration**  
   - Send POST requests with JSON body containing the text document to analyze.  
   - Use AI agents or tools that support MCP integration with automatic parameter injection via `$fromAI()`.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow uses n8n’s MCP server capabilities to expose AI agent-integrated endpoints with zero configuration required on the client side.         | Workflow Overview Sticky Note                                                                          |
| For detailed MCP node usage and customization, consult the official n8n documentation.                                                                | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/           |
| Community support and customizations are available via Discord channel linked in the workflow notes.                                                  | https://discord.me/cfomodz                                                                              |
| Ensure Google Cloud Natural Language API is enabled in your GCP project, and OAuth2 credentials have the correct scopes before running the workflow.  | Google Cloud Console, API & Services > Credentials                                                    |

---

**Disclaimer:** The content provided is derived exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.