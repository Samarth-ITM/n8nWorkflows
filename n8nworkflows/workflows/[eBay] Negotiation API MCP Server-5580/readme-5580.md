[eBay] Negotiation API MCP Server

https://n8nworkflows.xyz/workflows/-ebay--negotiation-api-mcp-server-5580


# [eBay] Negotiation API MCP Server

---

### 1. Workflow Overview

This workflow implements an MCP (Multi-Channel Platform) server interface for eBay's Negotiation API, enabling AI agents to interact with eBay seller discount offer operations. It targets sellers who want to programmatically send discount offers to buyers who have shown interest in their listings, facilitating automated negotiation workflows via AI.

The workflow is logically divided into these blocks:

- **1.1 Setup & Documentation**: Provides setup instructions, usage notes, and overview information for users deploying and integrating the MCP server.
- **1.2 MCP Trigger Endpoint**: Hosts the MCP trigger node acting as the AI agent's webhook endpoint, accepting incoming requests that specify which Negotiation API operation to execute.
- **1.3 Offer Operations**: Contains two HTTP Request Tool nodes that handle the main API calls:
  - *Find Eligible Listings*: Retrieves listings eligible for sending discount offers.
  - *Send Discount Offer*: Sends discount offers to buyers interested in eligible listings.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Documentation

- **Overview:**  
  This block provides comprehensive setup instructions, usage notes, and a workflow overview in sticky notes to assist users in importing, configuring, and customizing the workflow.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)

- **Node Details:**

  - **Setup Instructions (Sticky Note)**
    - *Type & Role:* Visual note node for documentation and user guidance.
    - *Configuration:* Contains detailed stepwise instructions on importing the workflow, configuring OAuth2 credentials, activating the workflow, obtaining the MCP webhook URL, and connecting AI agents. Also includes usage tips, customization advice, and a Discord support link.
    - *Key Expressions:* None.
    - *Inputs/Outputs:* None; standalone informational node.
    - *Edge Cases:* None.

  - **Workflow Overview (Sticky Note)**
    - *Type & Role:* Visual note node summarizing the workflow purpose, operation details, and usage.
    - *Configuration:* Describes the Negotiation API's function (sending discount offers), how the workflow acts as an MCP server, and lists the two available API operations exposed.
    - *Key Expressions:* None.
    - *Inputs/Outputs:* None; standalone informational node.
    - *Edge Cases:* None.

---

#### 2.2 MCP Trigger Endpoint

- **Overview:**  
  This block exposes the main entry point for AI agent requests via an MCP trigger node that acts as a webhook server endpoint, routing requests to the appropriate API operation nodes.

- **Nodes Involved:**  
  - Negotiation MCP Server (MCP Trigger)

- **Node Details:**

  - **Negotiation MCP Server (MCP Trigger)**
    - *Type & Role:* MCP trigger node — serves as the webhook endpoint for AI agents to send operation requests.
    - *Configuration:*  
      - Webhook path set to `/negotiation-mcp`.  
      - Waits for AI agent requests and routes to downstream HTTP Request Tool nodes based on the requested operation.
    - *Key Expressions:* None directly; it handles AI-driven operation dispatch.
    - *Input Connections:* None (trigger node).
    - *Output Connections:* Connected to both "Find Eligible Listings" and "Send Discount Offer" HTTP Request Tool nodes.
    - *Version Requirements:* Requires n8n version supporting MCP Trigger nodes (v1.95+).
    - *Edge Cases:*  
      - Webhook authentication failures if OAuth2 is misconfigured.  
      - Timeout or network errors when forwarding requests.  
      - Invalid or incomplete AI agent requests may cause routing issues.
    - *Sub-workflow Reference:* None.

---

#### 2.3 Offer Operations

- **Overview:**  
  This block contains the two core HTTP request nodes that execute the Negotiation API operations. Each node dynamically populates parameters from AI inputs and sends HTTP requests to eBay's API.

- **Nodes Involved:**  
  - Find Eligible Listings (HTTP Request Tool)  
  - Send Discount Offer (HTTP Request Tool)  
  - Offer (Sticky Note)

- **Node Details:**

  - **Find Eligible Listings (HTTP Request Tool)**
    - *Type & Role:* HTTP Request Tool node to query eBay's `/find_eligible_items` endpoint.
    - *Configuration:*  
      - HTTP GET request to `https://api.ebay.com{basePath}/find_eligible_items`.  
      - Query parameters `limit` and `offset` are dynamically populated using `$fromAI()` expressions, allowing the AI agent to specify pagination (limit: max 200, default 10; offset: default 0).  
      - Header `X-EBAY-C-MARKETPLACE-ID` is also dynamically set via `$fromAI()`, specifying the eBay marketplace.  
      - Uses generic HTTP header authentication with OAuth2 credentials configured externally.
      - Maintains original API response structure.
    - *Key Expressions:*  
      - `={{ $fromAI('limit', 'description', 'string') }}` for limit.  
      - `={{ $fromAI('offset', 'description', 'string') }}` for offset.  
      - `={{ $fromAI('X-EBAY-C-MARKETPLACE-ID', 'description', 'string') }}` for marketplace ID.
    - *Input Connections:* From MCP Trigger node.
    - *Output Connections:* None; returns response to MCP.
    - *Version Requirements:* HTTP Request Tool v4.2 or later recommended.
    - *Edge Cases:*  
      - API rate limits or quota exceeded.  
      - Invalid marketplace ID or missing headers causes API errors.  
      - Network errors, timeouts.  
      - Incorrect pagination parameters causing empty or partial results.
    - *Sub-workflow Reference:* None.

  - **Send Discount Offer (HTTP Request Tool)**
    - *Type & Role:* HTTP Request Tool node to POST discount offers via `/send_offer_to_interested_buyers` endpoint.
    - *Configuration:*  
      - HTTP POST request to `https://api.ebay.com{basePath}/send_offer_to_interested_buyers`.  
      - Header `X-EBAY-C-MARKETPLACE-ID` dynamically populated via `$fromAI()`.  
      - Authentication via generic HTTP header with OAuth2 credentials.  
      - Body and other parameters are expected to be populated by AI agent via MCP interface.
      - Returns API response to MCP endpoint.
    - *Key Expressions:*  
      - `={{ $fromAI('X-EBAY-C-MARKETPLACE-ID', 'description', 'string') }}` for marketplace ID.
    - *Input Connections:* From MCP Trigger node.
    - *Output Connections:* None; returns response to MCP.
    - *Version Requirements:* HTTP Request Tool v4.2 or later recommended.
    - *Edge Cases:*  
      - Missing or invalid marketplace ID headers.  
      - API errors if no eligible buyers found.  
      - Network or timeout failures.  
      - Incorrectly structured offer data from AI agent.
    - *Sub-workflow Reference:* None.

  - **Offer (Sticky Note)**
    - *Type & Role:* Visual note marking the "Offer" operations section.
    - *Content:* Simple heading "## Offer".
    - *Inputs/Outputs:* None.
    - *Edge Cases:* None.

---

### 3. Summary Table

| Node Name             | Node Type                 | Functional Role                          | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                                                                                                                                                                             |
|-----------------------|---------------------------|----------------------------------------|-------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions    | Sticky Note               | Setup & user guidance                   | —                       | —                             | Contains detailed setup instructions, usage tips, customization advice, and support links including Discord and official n8n docs.                                                                                                                                      |
| Workflow Overview     | Sticky Note               | Workflow purpose and operation summary | —                       | —                             | Describes the Negotiation API's use case, MCP server function, and lists two available API operations: Find Eligible Listings, Send Discount Offer.                                                                                                                     |
| Negotiation MCP Server | MCP Trigger               | AI agent request endpoint               | —                       | Find Eligible Listings, Send Discount Offer |                                                                                                                                                                                                                                                                         |
| Offer                 | Sticky Note               | Visual section header for Offer block  | —                       | —                             | Simple note with heading "## Offer".                                                                                                                                                                                                                                    |
| Find Eligible Listings | HTTP Request Tool         | Retrieves eligible listings for offers | Negotiation MCP Server   | —                             | Sends GET request to eBay's `/find_eligible_items` with dynamic pagination and marketplace ID headers. Parameters auto-populated using AI expressions.                                                                                                                  |
| Send Discount Offer    | HTTP Request Tool         | Sends discount offers to interested buyers | Negotiation MCP Server   | —                             | Sends POST request to eBay's `/send_offer_to_interested_buyers` with dynamic marketplace ID header. Parameters expected from AI.                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note "Setup Instructions"**  
   - Add a Sticky Note node.  
   - Paste the setup instructions content covering import, OAuth2 credential setup, activation, MCP URL, AI agent connection, usage notes, customization tips, and support links including Discord and n8n documentation.  
   - Set position to approximately [-1360, -240].

2. **Create Sticky Note "Workflow Overview"**  
   - Add another Sticky Note node.  
   - Input content describing the Negotiation API purpose, MCP server functionality, and list of two available endpoints: "Find Eligible Listings" and "Send Discount Offer".  
   - Set size width=340, height=780, position [-1100, -240].

3. **Add MCP Trigger Node "Negotiation MCP Server"**  
   - Add an MCP Trigger node.  
   - Set webhook path to `negotiation-mcp`.  
   - This node serves as the server endpoint for AI agent requests.  
   - Position around [-700, -240].

4. **Add Sticky Note "Offer"**  
   - Add a Sticky Note with content "## Offer".  
   - Position near [-740, -100].

5. **Add HTTP Request Tool Node "Find Eligible Listings"**  
   - Add HTTP Request Tool node.  
   - Set HTTP Method to GET.  
   - URL: `https://api.ebay.com{basePath}/find_eligible_items`.  
   - Enable sending query and headers.  
   - Add two query parameters:  
     - `limit` with value: `={{ $fromAI('limit', 'This query parameter specifies the maximum number of items to return from the result set on a page in the paginated response. Minimum: 1   Maximum: 200 Default: 10', 'string') }}`  
     - `offset` with value: `={{ $fromAI('offset', 'This query parameter specifies the number of results to skip in the result set before returning the first result in the paginated response... Default: 0', 'string') }}`  
   - Add header parameter:  
     - `X-EBAY-C-MARKETPLACE-ID` with value: `={{ $fromAI('X-EBAY-C-MARKETPLACE-ID', 'The eBay marketplace on which you want to search for eligible listings...', 'string') }}`  
   - Authentication: Use generic HTTP header auth; link to OAuth2 credentials configured for eBay API.  
   - Position at approximately [-600, -60].  
   - Connect input from MCP Trigger node.

6. **Add HTTP Request Tool Node "Send Discount Offer"**  
   - Add HTTP Request Tool node.  
   - Set HTTP Method to POST.  
   - URL: `https://api.ebay.com{basePath}/send_offer_to_interested_buyers`.  
   - Enable sending headers.  
   - Add header parameter:  
     - `X-EBAY-C-MARKETPLACE-ID` with value: `={{ $fromAI('X-EBAY-C-MARKETPLACE-ID', 'The eBay marketplace on which your listings with "eligible" buyers appear...', 'string') }}`  
   - Authentication: Use generic HTTP header auth; link to OAuth2 credentials configured for eBay API.  
   - Position at approximately [-400, -60].  
   - Connect input from MCP Trigger node.

7. **Configure OAuth2 Credentials**  
   - In n8n credentials, create or configure OAuth2 credentials for eBay API with appropriate client ID, secret, and token URLs.  
   - Assign these credentials to the HTTP Request Tool nodes under "Authentication" → "Generic Credential Type" → "HTTP Header Auth".

8. **Activate Workflow**  
   - Save and activate the workflow.  
   - Copy the webhook URL from the MCP trigger node for use in AI agent configuration.

9. **Optional Customizations**  
   - Add data transformation nodes between MCP trigger and HTTP requests if needed.  
   - Add error handling nodes (e.g., Error Trigger, IF nodes) for graceful failure responses.  
   - Add logging or monitoring nodes to track requests and responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| For integration guidance and custom automations, contact the workflow author on Discord: https://discord.me/cfomodz                                                                                                                                                                                               | Discord support                                                                                                                          |
| Official n8n documentation for MCP triggers and Langchain integration can be found here: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                                                                                                                              | n8n Documentation                                                                                                                        |
| The Negotiation API requires proper eBay marketplace IDs; refer to eBay’s Negotiation API requirements and restrictions documentation for valid values and usage.                                                                                                                                                   | eBay API documentation                                                                                                                  |
| This workflow uses `$fromAI()` expressions to dynamically populate parameters from AI agent inputs, facilitating seamless AI integration without manual parameter entry.                                                                                                                                             | n8n AI expressions                                                                                                                       |
| The workflow maintains original API response structures to ensure AI agents receive consistent data formats for downstream processing.                                                                                                                                                                           | Design note                                                                                                                             |

---

**Disclaimer:** The provided text derives exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.