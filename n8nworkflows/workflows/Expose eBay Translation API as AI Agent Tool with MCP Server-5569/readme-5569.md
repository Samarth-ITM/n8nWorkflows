Expose eBay Translation API as AI Agent Tool with MCP Server

https://n8nworkflows.xyz/workflows/expose-ebay-translation-api-as-ai-agent-tool-with-mcp-server-5569


# Expose eBay Translation API as AI Agent Tool with MCP Server

---

### 1. Workflow Overview

This workflow exposes eBay’s Translation API as an AI Agent Tool using an MCP (Modular Connector Protocol) Server interface within n8n. It is designed to serve 3rd party AI agents by providing a standardized endpoint that translates text (item titles, descriptions, search queries) using eBay’s API.

The workflow logically divides into these functional blocks:

- **1.1 Setup & Documentation**: Provides setup instructions, usage notes, and overview information via sticky notes for user guidance.
- **1.2 MCP Server Trigger**: The entry point that acts as a server endpoint receiving AI agent requests via MCP.
- **1.3 Translation API Call**: Handles the HTTP request to eBay’s Translation API, translating input text as specified by AI parameters.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Documentation

- **Overview:**  
  This block provides users with essential setup instructions, usage notes, and a high-level overview of the workflow’s purpose and capabilities. It ensures users understand how to configure, activate, and utilize the workflow effectively.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  
  - Sticky Note (labeled “Language”)

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note (visual documentation)  
    - Content: Step-by-step setup guide including importing workflow, configuring OAuth2 credentials, activating workflow, obtaining MCP URL, and connecting AI agent. Also includes usage tips and customization suggestions.  
    - Input/Output: None (informational only)  
    - Edge cases: None (no runtime function)  

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Content: Describes available operations (1 endpoint for Language Translation), explains how the workflow functions with MCP trigger and HTTP request nodes, and highlights AI expression usage ($fromAI()).  
    - Input/Output: None  
    - Edge cases: None  

  - **Sticky Note (Language)**  
    - Type: Sticky Note  
    - Content: Simply labels the language translation section, serving as a visual separator.  
    - Input/Output: None  

---

#### 2.2 MCP Server Trigger

- **Overview:**  
  This block serves as the workflow’s entry point, exposing an MCP-compliant webhook endpoint for AI agents to send requests for translation services.

- **Nodes Involved:**  
  - Translation MCP Server

- **Node Details:**

  - **Translation MCP Server**  
    - Type: MCP Trigger node (LangChain)  
    - Configuration:  
      - Path: `translation-mcp` — defines the webhook endpoint path.  
      - This node listens for incoming AI agent requests compliant with MCP protocol.  
    - Input: External HTTP requests from AI agents  
    - Output: Passes data downstream to the API request node  
    - Version-specific: Requires n8n with LangChain MCP node support  
    - Edge cases:  
      - Failure to authenticate incoming requests if MCP security not configured properly  
      - Webhook URL misconfiguration  
      - Timeout if downstream nodes are slow  
    - Sub-workflow: None  

---

#### 2.3 Translation API Call

- **Overview:**  
  This block executes the actual translation request by calling eBay’s Translation API endpoint with parameters auto-populated via AI expressions from the MCP trigger.

- **Nodes Involved:**  
  - Translate Text

- **Node Details:**

  - **Translate Text**  
    - Type: HTTP Request Tool Node  
    - Configuration:  
      - URL: `https://api.ebay.com{basePath}/translate` — basePath expected to be resolved dynamically.  
      - Method: POST  
      - Authentication: HTTP Header Authentication using a generic OAuth2 credential (must be preconfigured)  
      - Tool Description: “Translates input text into a given language.”  
      - Parameters: Populated dynamically using `$fromAI()` expressions (implied from workflow notes)  
    - Input: Receives AI parameters from MCP trigger node such as source text, target language, and other translation options  
    - Output: Sends API response back to MCP node for returning to AI agent  
    - Version-specific: Requires n8n version supporting HTTP Request Tool node v4.2+  
    - Edge cases:  
      - Authentication errors if OAuth2 token invalid or expired  
      - API endpoint unreachable or network failures  
      - Malformed or missing parameters causing API errors  
      - Rate limiting or quota exceeded from eBay API  
    - Sub-workflow: None  

---

### 3. Summary Table

| Node Name            | Node Type                  | Functional Role                   | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                              |
|----------------------|----------------------------|---------------------------------|------------------------|------------------------|--------------------------------------------------------------------------------------------------------|
| Setup Instructions   | Sticky Note                | User setup & configuration guide | None                   | None                   | See detailed setup instructions and usage notes including OAuth2 setup, workflow activation, and MCP URL usage. [discord link](https://discord.me/cfomodz) & [n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) |
| Workflow Overview    | Sticky Note                | Workflow purpose & operation summary | None                   | None                   | Describes Translation MCP Server, MCP trigger use, auto-population of parameters with $fromAI(), and available endpoint info.                      |
| Sticky Note (Language) | Sticky Note              | Visual section label             | None                   | None                   |                                                                                                        |
| Translation MCP Server | MCP Trigger (LangChain)    | Receives AI agent requests via MCP | None                   | Translate Text          |                                                                                                        |
| Translate Text       | HTTP Request Tool          | Calls eBay Translation API       | Translation MCP Server  | None                   |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note “Setup Instructions”**  
   - Add Sticky Note node.  
   - Paste setup instructions including: import workflow, OAuth2 credential setup, workflow activation, MCP URL retrieval, AI agent connection.  
   - Add usage notes about `$fromAI()` expressions, API endpoint availability, response structure, and customization tips.  
   - Add contact info links: Discord (https://discord.me/cfomodz) and n8n documentation (https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/).

2. **Create Sticky Note “Workflow Overview”**  
   - Add Sticky Note node.  
   - Summarize the workflow purpose: exposing eBay Translation API as MCP server for AI agents.  
   - Describe MCP trigger, HTTP request nodes, AI expression usage, and available translation endpoint.

3. **Create Sticky Note “Language”**  
   - Add Sticky Note node.  
   - Add a simple label “Language” to visually separate translation logic.

4. **Create MCP Trigger Node**  
   - Add MCP Trigger node (LangChain).  
   - Set the webhook path to `translation-mcp`.  
   - This node will serve as the external HTTP endpoint for AI agent requests.

5. **Configure OAuth2 Credentials for eBay API**  
   - In n8n Credentials, create an OAuth2 credential for eBay API.  
   - Ensure it supports HTTP Header Authentication with valid client ID, secret, and access token management.

6. **Create HTTP Request Tool Node “Translate Text”**  
   - Add HTTP Request Tool node.  
   - Set URL to `https://api.ebay.com{basePath}/translate`.  
   - Set HTTP method to POST.  
   - Choose “Generic Credential Type” authentication, with HTTP Header Auth using the OAuth2 eBay credential created.  
   - Add tool description: “Translates input text into a given language.”  
   - Use `$fromAI()` expressions for parameters (e.g., source text, target language) to auto-populate from the MCP trigger input.  
   - Connect input from the MCP Trigger node.

7. **Connect Nodes**  
   - Connect MCP Trigger node’s AI tool output to the “Translate Text” node’s input.  
   - The HTTP Request Tool node outputs directly back to MCP node to return API response to AI agent.

8. **Activate Workflow**  
   - Save and activate the workflow in n8n.  
   - Retrieve the webhook URL from the MCP Trigger node for use by AI agents.

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Setup and customization instructions are embedded in the workflow via sticky notes for user convenience.                               | Internal workflow documentation                                                                              |
| MCP Server allows native integration with AI agents supporting MCP protocol, simplifying third-party API exposure.                      | See n8n LangChain MCP node documentation: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ |
| For integration help or custom automation support, users can reach out on Discord.                                                     | https://discord.me/cfomodz                                                                                   |
| Workflow assumes valid OAuth2 credentials for eBay API are configured prior to activation.                                              | Credential setup in n8n required                                                                              |
| Responses preserve original eBay API structure to maintain consistency and ease of parsing by AI agents.                               | Design choice for native API response handling                                                              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---