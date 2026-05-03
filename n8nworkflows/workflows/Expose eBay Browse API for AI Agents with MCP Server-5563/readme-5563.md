Expose eBay Browse API for AI Agents with MCP Server

https://n8nworkflows.xyz/workflows/expose-ebay-browse-api-for-ai-agents-with-mcp-server-5563


# Expose eBay Browse API for AI Agents with MCP Server

### 1. Workflow Overview

This workflow, titled **"[eBay] Browse API MCP Server"**, is designed to expose eBay's Browse API functionalities through an MCP (Multi-Channel Platform) server interface tailored for AI agents. It acts as a backend service that listens for AI tool requests and performs various eBay Browse API operations, such as searching for items, retrieving item details, managing shopping carts, and checking item compatibility.

The workflow is logically divided into the following functional blocks:

- **1.1 MCP Server Trigger**: The entry point that listens for AI-driven requests via the MCP trigger node.
- **1.2 eBay Browse API Request Handlers**: Multiple HTTP Request Tool nodes handle different types of eBay Browse API calls, each performing a specific operation (e.g., retrieving item details or searching by image).
- **1.3 Informational Notes**: Sticky notes scattered throughout the workflow provide setup instructions, workflow overview, and contextual commentary for developers or maintainers.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger

- **Overview:**  
  This block serves as the input reception point that listens for incoming AI agent requests routed through the MCP server trigger node. It acts as the gateway for all subsequent API calls.

- **Nodes Involved:**  
  - Browse MCP Server

- **Node Details:**

  - **Browse MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Listens to incoming MCP requests from AI agents; triggers the workflow when a request arrives.  
    - Configuration: Default MCP trigger settings with a unique webhook ID for external communication.  
    - Input: None (webhook trigger)  
    - Output: Routes requests to all connected eBay Browse API HTTP request nodes via the `ai_tool` output connection.  
    - Edge Cases: Potential issues include webhook authentication errors, timeouts if AI agent requests are delayed, or payload parsing errors.  
    - Version Requirements: Requires n8n version supporting LangChain MCP Trigger nodes.  
    - Sub-workflow: None.

---

#### 2.2 eBay Browse API Request Handlers

- **Overview:**  
  This block contains multiple HTTP Request Tool nodes, each responsible for executing a specific eBay Browse API endpoint operation depending on the AI agent’s request type. These nodes process the request, call eBay’s API, and return the relevant data.

- **Nodes Involved:**  
  - Retrieve Item Details  
  - Retrieve Item by Legacy ID  
  - Retrieve Items in Group  
  - Retrieve Item by ID  
  - Check Item Compatibility  
  - Search Item Summaries  
  - Search Items by Image  
  - Retrieve Shopping Cart  
  - Add Item to Cart  
  - Remove Item from Cart  
  - Update Cart Item Quantity

- **Node Details:**

  - **Retrieve Item Details**  
    - Type: HTTP Request Tool  
    - Role: Calls eBay Browse API to fetch detailed information about a specific item using the current API format.  
    - Configuration: Configured to make a GET request to the appropriate eBay Browse API endpoint with authentication headers and item identifiers passed from the MCP server trigger.  
    - Inputs: Receives item identifiers and parameters from MCP trigger node.  
    - Outputs: Returns item detail data to the MCP server node for response to the AI agent.  
    - Edge Cases: Invalid item IDs, API rate limits, authentication failures, or network timeouts.

  - **Retrieve Item by Legacy ID**  
    - Type: HTTP Request Tool  
    - Role: Retrieves item data using eBay's legacy item ID format.  
    - Configuration: Similar GET request with legacy ID parameters.  
    - Inputs/Outputs: Same as above.  
    - Edge Cases: Legacy ID format errors, deprecated endpoint issues.

  - **Retrieve Items in Group**  
    - Type: HTTP Request Tool  
    - Role: Fetches all items grouped under a specific category or collection.  
    - Configuration: GET request with group identifiers.  
    - Edge Cases: Empty groups, invalid group IDs.

  - **Retrieve Item by ID**  
    - Type: HTTP Request Tool  
    - Role: Alternative or supplementary method to fetch item details using item ID.  
    - Edge Cases: Similar to Retrieve Item Details.

  - **Check Item Compatibility**  
    - Type: HTTP Request Tool  
    - Role: Checks if an item is compatible with a given product or requirement (e.g., vehicle parts compatibility).  
    - Configuration: Calls compatibility endpoint with query parameters specifying compatibility criteria.  
    - Edge Cases: Invalid compatibility parameters, missing required data.

  - **Search Item Summaries**  
    - Type: HTTP Request Tool  
    - Role: Executes search queries to retrieve summarized listings based on search criteria.  
    - Configuration: GET request with query parameters for keywords, filters, pagination, etc.  
    - Edge Cases: Empty search results, malformed search queries.

  - **Search Items by Image**  
    - Type: HTTP Request Tool  
    - Role: Enables reverse image search to find items matching an uploaded image.  
    - Configuration: Likely POST request with image data or URL.  
    - Edge Cases: Unsupported image formats, image size limits, API restrictions.

  - **Retrieve Shopping Cart**  
    - Type: HTTP Request Tool  
    - Role: Retrieves the current state of the user’s shopping cart.  
    - Edge Cases: Cart session expired, empty cart.

  - **Add Item to Cart**  
    - Type: HTTP Request Tool  
    - Role: Adds specified item to the user’s shopping cart.  
    - Edge Cases: Item out of stock, invalid quantity.

  - **Remove Item from Cart**  
    - Type: HTTP Request Tool  
    - Role: Removes a specific item from the shopping cart.  
    - Edge Cases: Item not present in cart.

  - **Update Cart Item Quantity**  
    - Type: HTTP Request Tool  
    - Role: Updates the quantity of an existing item in the cart.  
    - Edge Cases: Quantity exceeds stock, invalid quantity values.

  - All HTTP Request Tool nodes receive their inputs from the MCP trigger node via the `ai_tool` output connection. They process the API call and return data to the MCP server node, which then communicates back to the AI agent.

---

#### 2.3 Informational Notes

- **Overview:**  
  Sticky Note nodes distributed in the workflow provide textual guidance, setup instructions, or workflow overview information for users and maintainers.

- **Nodes Involved:**  
  - Setup Instructions  
  - Workflow Overview  
  - Sticky Note (near Browse MCP Server)  
  - Sticky Note2 (near Search Item Summaries)  
  - Sticky Note3 (near Search Items by Image)  
  - Sticky Note4 (near Retrieve Shopping Cart)

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Contains instructions on configuring the workflow environment, API credentials, or prerequisites.

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Describes the purpose and general flow of the workflow.

  - Other Sticky Notes  
    - Provide contextual comments or reminders at specific workflow sections for clarity.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                           | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                   |
|----------------------------|----------------------------------|------------------------------------------|-----------------------|----------------------|-----------------------------------------------------------------------------------------------|
| Setup Instructions         | Sticky Note                      | Setup guidance                          | None                  | None                 |                                                                                               |
| Workflow Overview          | Sticky Note                      | Workflow description                    | None                  | None                 |                                                                                               |
| Browse MCP Server          | MCP Trigger                     | Entry point; listens for AI agent requests | None                  | All HTTP Request Tools |                                                                                               |
| Sticky Note               | Sticky Note                      | Contextual comment near MCP Server     | None                  | None                 |                                                                                               |
| Retrieve Item Details      | HTTP Request Tool                | Fetch detailed item info                | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Retrieve Item by Legacy ID | HTTP Request Tool                | Fetch item info by legacy ID            | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Retrieve Items in Group    | HTTP Request Tool                | Fetch items in a group                  | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Retrieve Item by ID        | HTTP Request Tool                | Fetch item info by ID                   | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Check Item Compatibility   | HTTP Request Tool                | Check product compatibility             | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Sticky Note2              | Sticky Note                      | Comment near Search Item Summaries      | None                  | None                 |                                                                                               |
| Search Item Summaries      | HTTP Request Tool                | Search for items with summary info     | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Sticky Note3              | Sticky Note                      | Comment near Search Items by Image      | None                  | None                 |                                                                                               |
| Search Items by Image      | HTTP Request Tool                | Search items using image input          | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Sticky Note4              | Sticky Note                      | Comment near Shopping Cart operations   | None                  | None                 |                                                                                               |
| Retrieve Shopping Cart     | HTTP Request Tool                | Get current shopping cart contents     | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Add Item to Cart           | HTTP Request Tool                | Add an item to shopping cart            | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Remove Item from Cart      | HTTP Request Tool                | Remove item from shopping cart          | Browse MCP Server      | Browse MCP Server     |                                                                                               |
| Update Cart Item Quantity  | HTTP Request Tool                | Modify quantity of an item in cart      | Browse MCP Server      | Browse MCP Server     |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named `[eBay] Browse API MCP Server`.

2. **Add Sticky Note nodes**:
   - Create a node named `Setup Instructions` with setup guidance content.
   - Create a node named `Workflow Overview` describing the workflow’s purpose.
   - Add additional sticky notes near respective API call nodes for comments and context.

3. **Add MCP Trigger Node**:
   - Add `Browse MCP Server` node of type `MCP Trigger`.
   - Configure it with a unique webhook ID or leave default for auto-generation.
   - This node listens for AI agent requests.

4. **Add HTTP Request Tool Nodes** for each eBay Browse API operation:
   - **Retrieve Item Details**
     - Method: GET
     - URL: eBay Browse API endpoint for item details (e.g., `/buy/browse/v1/item/{item_id}`)
     - Authentication: Use eBay API credentials (OAuth2 or API Key)
     - Parameters: Accept item ID from MCP trigger input data.
   - **Retrieve Item by Legacy ID**
     - Similar GET request with legacy item ID format.
   - **Retrieve Items in Group**
     - GET request to fetch items by group/category.
   - **Retrieve Item by ID**
     - GET request to fetch item by ID.
   - **Check Item Compatibility**
     - GET request with compatibility parameters.
   - **Search Item Summaries**
     - GET request with search query parameters.
   - **Search Items by Image**
     - POST request with image data or URL.
   - **Retrieve Shopping Cart**
     - GET request for current cart state.
   - **Add Item to Cart**
     - POST request with item ID and quantity.
   - **Remove Item from Cart**
     - DELETE or POST request to remove item.
   - **Update Cart Item Quantity**
     - PATCH or POST request to update quantity.

5. **Set up credentials** for eBay API:
   - Create OAuth2 or API key credentials inside n8n for eBay Browse API.
   - Assign the credentials to each HTTP Request Tool node.

6. **Connect nodes**:
   - Connect `Browse MCP Server` node output (`ai_tool` channel) to the input of each HTTP Request Tool node.
   - Connect each HTTP Request Tool node output back to the `Browse MCP Server` node to return responses.

7. **Configure error handling** on HTTP Request Tool nodes:
   - Set retry logic or error workflows to handle API failures, timeouts, or invalid inputs.

8. **Validate webhook and API permissions**:
   - Ensure the MCP trigger webhook is publicly accessible.
   - Confirm API keys have sufficient rights for Browse API operations.

9. **Test the workflow**:
   - Simulate AI agent requests through the MCP webhook with various operation requests.
   - Verify correct API calls and responses.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                            |
|------------------------------------------------------------------------------|-------------------------------------------|
| The workflow uses the LangChain MCP Trigger node to interface with AI agents.| Requires n8n version supporting LangChain MCP. |
| eBay Browse API documentation: https://developer.ebay.com/api-docs/buy/browse/ | Official API reference for endpoint details. |
| Ensure eBay OAuth2 credentials are properly configured in n8n credentials.   | Credential setup instructions.             |
| Sticky notes in the workflow serve as inline documentation and setup hints.  | Aid for maintainers and developers.        |

---

**Disclaimer:**  
The provided description and analysis are based exclusively on the automated n8n workflow JSON. All data and operations comply with applicable policies and legal guidelines.