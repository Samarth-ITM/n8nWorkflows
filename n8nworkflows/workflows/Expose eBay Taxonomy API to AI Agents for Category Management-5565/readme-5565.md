Expose eBay Taxonomy API to AI Agents for Category Management

https://n8nworkflows.xyz/workflows/expose-ebay-taxonomy-api-to-ai-agents-for-category-management-5565


# Expose eBay Taxonomy API to AI Agents for Category Management

### 1. Workflow Overview

This workflow, titled **"[eBay] Taxonomy API MCP Server"**, serves as an integration layer exposing eBay's Taxonomy API functionalities to AI agents via a Multi-Channel Processing (MCP) trigger. Its core purpose is to facilitate category management operations such as fetching category trees, retrieving category aspects, compatibility properties, and suggestions, enabling AI-driven automation for eBay category handling.

The workflow is logically divided into two main blocks based on functional roles:

- **1.1 MCP Server Trigger Input:**  
  This block receives incoming AI agent requests via an MCP trigger node, acting as the entry point for category-related queries.

- **1.2 eBay Taxonomy API Query Handlers:**  
  This block contains several HTTP Request Tool nodes that handle specific API calls to eBay's Taxonomy endpoints, such as fetching category trees, subtree data, category suggestions, item aspects, compatibility properties, and compatibility values. Each node corresponds to a distinct API endpoint and is connected directly to the MCP trigger to serve requests dynamically.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger Input

- **Overview:**  
  This block initializes the workflow by listening for incoming requests from AI agents. It acts as the main gateway for all subsequent eBay Taxonomy API calls.

- **Nodes Involved:**  
  - Taxonomy MCP Server

- **Node Details:**  

  - **Taxonomy MCP Server**  
    - **Type and Technical Role:**  
      `@n8n/n8n-nodes-langchain.mcpTrigger` — a specialized trigger node designed to receive requests from AI agents via the MCP protocol.  
    - **Configuration Choices:**  
      Configured with a webhook ID (`64664d6b-5d78-4108-80e1-7311a074c81b`) to expose a unique HTTP endpoint. No additional parameters configured, relying on dynamic input to direct processing.  
    - **Key Expressions or Variables:**  
      Accepts input data specifying the type of taxonomy operation requested, which controls downstream routing implicitly.  
    - **Input and Output Connections:**  
      No input connections (trigger node). Outputs connect to all HTTP request nodes handling eBay API calls.  
    - **Version-Specific Requirements:**  
      Requires n8n version supporting `mcpTrigger` nodes and webhook integration.  
    - **Edge Cases / Potential Failures:**  
      - Webhook security/authentication misconfiguration could expose the endpoint.  
      - Payload format errors from AI agents may cause request parsing failures.  
      - Network or server downtime leading to unavailability.  
    - **Sub-workflow Reference:**  
      This node is the entry point and does not invoke sub-workflows.

#### 1.2 eBay Taxonomy API Query Handlers

- **Overview:**  
  This block contains multiple HTTP Request Tool nodes, each responsible for invoking a specific eBay Taxonomy API endpoint. They operate in parallel, all triggered by the MCP Server node to fulfill distinct category management queries.

- **Nodes Involved:**  
  - Fetch Category Tree  
  - Retrieve Leaf Category Aspects  
  - Fetch Category Subtree  
  - Get Category Suggestions  
  - Retrieve Compatibility Properties  
  - Fetch Compatibility Values  
  - Get Item Aspects by Category  
  - Fetch Default Category Tree ID

- **Node Details:**  

  - **Fetch Category Tree**  
    - **Type and Technical Role:**  
      HTTP Request Tool; fetches the full eBay category tree.  
    - **Configuration Choices:**  
      Configured for eBay API endpoint returning the category tree structure. Uses appropriate authentication credentials (not shown in JSON but required).  
    - **Input and Output Connections:**  
      Input from `Taxonomy MCP Server` node; outputs the category tree JSON.  
    - **Edge Cases:**  
      - API rate limiting or authentication failures.  
      - Large response payloads causing timeouts.  

  - **Retrieve Leaf Category Aspects**  
    - **Type and Technical Role:**  
      HTTP Request Tool; retrieves specific aspects for leaf categories.  
    - **Configuration Choices:**  
      Calls eBay API endpoint for leaf category details.  
    - **Connections:**  
      Input from MCP Server; output returns aspects data.  
    - **Edge Cases:**  
      - Invalid category IDs leading to 404 errors.  
      - Inconsistent or incomplete aspect data from API.  

  - **Fetch Category Subtree**  
    - **Type and Technical Role:**  
      HTTP Request Tool; fetches a subtree within the category tree.  
    - **Configuration Choices:**  
      Targets subtree endpoint with category ID as parameter.  
    - **Connections:**  
      Input from MCP Server.  
    - **Edge Cases:**  
      - Invalid or missing subtree ID parameters.  

  - **Get Category Suggestions**  
    - **Type and Technical Role:**  
      HTTP Request Tool; provides category suggestions based on input keywords.  
    - **Configuration Choices:**  
      Calls suggestion endpoint with query parameters.  
    - **Connections:**  
      Input from MCP Server.  
    - **Edge Cases:**  
      - Empty or invalid search terms.  
      - API returning no suggestions.  

  - **Retrieve Compatibility Properties**  
    - **Type and Technical Role:**  
      HTTP Request Tool; retrieves compatibility properties for categories.  
    - **Configuration Choices:**  
      Queries API endpoint returning compatibility-related metadata.  
    - **Connections:**  
      Input from MCP Server.  
    - **Edge Cases:**  
      - Missing or unsupported category IDs.  

  - **Fetch Compatibility Values**  
    - **Type and Technical Role:**  
      HTTP Request Tool; fetches compatibility values associated with properties.  
    - **Configuration Choices:**  
      Targets compatibility values endpoint with required parameters.  
    - **Connections:**  
      Input from MCP Server.  
    - **Edge Cases:**  
      - Network latency or incomplete data.  

  - **Get Item Aspects by Category**  
    - **Type and Technical Role:**  
      HTTP Request Tool; fetches item aspects specific to a category.  
    - **Configuration Choices:**  
      Calls API endpoint returning aspects metadata.  
    - **Connections:**  
      Input from MCP Server.  
    - **Edge Cases:**  
      - Outdated category IDs causing API errors.  

  - **Fetch Default Category Tree ID**  
    - **Type and Technical Role:**  
      HTTP Request Tool; retrieves the default category tree ID.  
    - **Configuration Choices:**  
      Calls API endpoint returning the default tree identifier.  
    - **Connections:**  
      Input from MCP Server.  
    - **Edge Cases:**  
      - Missing or unexpected API responses.

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                           | Input Node(s)       | Output Node(s)     | Sticky Note |
|------------------------------|-----------------------------------|------------------------------------------|---------------------|--------------------|-------------|
| Setup Instructions            | Sticky Note                       | Documentation placeholder                 | —                   | —                  |             |
| Workflow Overview             | Sticky Note                       | Documentation placeholder                 | —                   | —                  |             |
| Taxonomy MCP Server           | MCP Trigger                      | Entry point receiving AI agent requests  | —                   | All HTTP Request nodes |             |
| Sticky Note                  | Sticky Note                       | Documentation placeholder                 | —                   | —                  |             |
| Fetch Category Tree           | HTTP Request Tool                | Fetch full eBay category tree             | Taxonomy MCP Server  | —                  |             |
| Retrieve Leaf Category Aspects| HTTP Request Tool                | Retrieve aspects for leaf categories      | Taxonomy MCP Server  | —                  |             |
| Fetch Category Subtree        | HTTP Request Tool                | Fetch category subtree data                | Taxonomy MCP Server  | —                  |             |
| Get Category Suggestions      | HTTP Request Tool                | Provide category suggestions               | Taxonomy MCP Server  | —                  |             |
| Retrieve Compatibility Properties | HTTP Request Tool            | Retrieve compatibility properties          | Taxonomy MCP Server  | —                  |             |
| Fetch Compatibility Values    | HTTP Request Tool                | Fetch compatibility values                 | Taxonomy MCP Server  | —                  |             |
| Get Item Aspects by Category  | HTTP Request Tool                | Fetch item aspects metadata                 | Taxonomy MCP Server  | —                  |             |
| Fetch Default Category Tree ID| HTTP Request Tool                | Retrieve default category tree ID          | Taxonomy MCP Server  | —                  |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation (Optional):**  
   - Create three Sticky Note nodes named:  
     - `Setup Instructions`  
     - `Workflow Overview`  
     - `Sticky Note`  
   - Position them for clarity; content can be added later as needed.

2. **Add MCP Trigger Node:**  
   - Create a node of type `@n8n/n8n-nodes-langchain.mcpTrigger`, named `Taxonomy MCP Server`.  
   - Configure the webhook ID or let n8n generate a new webhook URL. This node serves as the main entry point for AI agents.  
   - Leave parameters empty unless specific routing is required.

3. **Add HTTP Request Tool Nodes for eBay Taxonomy API Endpoints:**  
   For each of the following nodes, create an HTTP Request Tool node with these configurations:  
   - **Authentication:** Configure with valid eBay API credentials (OAuth2 or API key as required).  
   - **HTTP Method:** `GET` (assumed for taxonomy queries).  
   - **URL:** Use the appropriate eBay Taxonomy API endpoint for each node (refer to eBay developer documentation).  
   - **Parameters:** Map input parameters dynamically from the MCP trigger payload (e.g., category IDs, keywords).  
   - **Headers:** Include necessary headers such as `Authorization` and `Accept: application/json`.

   Nodes to create:  
   - `Fetch Category Tree`  
   - `Retrieve Leaf Category Aspects`  
   - `Fetch Category Subtree`  
   - `Get Category Suggestions`  
   - `Retrieve Compatibility Properties`  
   - `Fetch Compatibility Values`  
   - `Get Item Aspects by Category`  
   - `Fetch Default Category Tree ID`

4. **Connect Nodes:**  
   - Connect the output of `Taxonomy MCP Server` node to the input of **each** HTTP Request Tool node listed above, enabling parallel processing of different API calls based on the AI agent's request.

5. **Set Up Credentials:**  
   - Ensure eBay API credentials are created and configured in n8n credentials manager.  
   - Assign these credentials to all HTTP Request Tool nodes requiring authentication.

6. **Configure Parameters for API Calls:**  
   - For each HTTP Request node, configure parameters dynamically using expressions to extract required inputs from the MCP trigger data.  
   - For example, category IDs or keywords should be mapped from the incoming request data fields.

7. **Test Workflow:**  
   - Deploy the workflow and test by sending sample AI agent requests to the MCP webhook URL specifying different taxonomy operations.  
   - Verify each HTTP Request node successfully returns expected data.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                               |
|------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| eBay Taxonomy API documentation is essential for correct endpoint URLs and parameter details.              | https://developer.ebay.com/api-docs/static/taxonomy-api-overview.html |
| MCP Trigger nodes enable AI agents to invoke workflows dynamically; ensure secure webhook exposure.        | n8n documentation on MCP Trigger nodes        |
| Proper credential handling (OAuth2 tokens refresh) is critical to avoid authentication failures.           | n8n OAuth2 Credential Setup                     |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created in n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.