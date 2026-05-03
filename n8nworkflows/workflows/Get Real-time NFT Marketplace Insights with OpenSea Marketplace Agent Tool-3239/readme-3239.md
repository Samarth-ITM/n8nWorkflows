Get Real-time NFT Marketplace Insights with OpenSea Marketplace Agent Tool

https://n8nworkflows.xyz/workflows/get-real-time-nft-marketplace-insights-with-opensea-marketplace-agent-tool-3239


# Get Real-time NFT Marketplace Insights with OpenSea Marketplace Agent Tool

### 1. Workflow Overview

The **OpenSea Marketplace Agent Tool** is an advanced n8n workflow designed to provide real-time, AI-enhanced insights into the NFT marketplace via OpenSea's API. It targets NFT traders, collectors, and investors who want to monitor listings, offers, orders, and trait-based pricing across multiple blockchains. The workflow leverages OpenAI's GPT-4o-mini model to interpret user queries and intelligently route requests to the appropriate OpenSea API endpoints, delivering structured, actionable data.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**  
  Captures user queries and session context.

- **1.2 AI Processing & Memory**  
  Uses GPT-4o-mini to parse queries, maintain session memory, and determine which marketplace API tool to invoke.

- **1.3 OpenSea API Integration Tools**  
  A suite of HTTP Request nodes configured as AI tools, each targeting a specific OpenSea API endpoint to fetch listings, offers, orders, or trait-based data.

- **1.4 Agent Orchestration**  
  The OpenSea Marketplace Agent node acts as the central AI agent, connecting user input, AI brain, memory, and API tools to execute the correct data retrieval and response generation.

- **1.5 Documentation & Notes**  
  Sticky notes provide detailed guidance, usage instructions, API endpoint descriptions, and error handling tips.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives external input data (user queries and session IDs) to initiate the workflow.

**Nodes Involved:**  
- Workflow Input Trigger

**Node Details:**  
- **Workflow Input Trigger**  
  - Type: `executeWorkflowTrigger` (n8n base node)  
  - Role: Entry point for the workflow, accepts parameters named `message` (user query) and `sessionId` (to maintain conversation context).  
  - Configuration: Expects JSON input with keys `message` and `sessionId`.  
  - Input: External trigger (e.g., HTTP webhook or manual execution).  
  - Output: Passes data to the OpenSea Marketplace Agent node.  
  - Edge Cases: Missing or malformed input could cause downstream errors; no explicit validation here.

---

#### 1.2 AI Processing & Memory

**Overview:**  
Processes user queries with GPT-4o-mini, maintains session memory for context continuity, and interprets which API tool to call.

**Nodes Involved:**  
- Marketplace Agent Brain  
- Marketplace Agent Memory

**Node Details:**  
- **Marketplace Agent Brain**  
  - Type: `lmChatOpenAi` (LangChain OpenAI node)  
  - Role: Acts as the AI language model brain, interpreting user queries and generating instructions for the agent.  
  - Configuration: Uses GPT-4o-mini model, no additional options set.  
  - Input: Receives raw user queries or intermediate prompts.  
  - Output: Provides AI-generated text responses or instructions.  
  - Credentials: Requires OpenAI API key configured in n8n.  
  - Edge Cases: API rate limits, malformed prompts, or network issues may cause failures.

- **Marketplace Agent Memory**  
  - Type: `memoryBufferWindow` (LangChain memory node)  
  - Role: Stores recent conversation history to maintain context across multiple user queries in the same session.  
  - Configuration: Default buffer window memory, no custom parameters.  
  - Input: Receives conversation messages and AI responses.  
  - Output: Provides context to the AI brain for coherent dialogue.  
  - Edge Cases: Memory overflow or session ID mismatches could cause context loss.

---

#### 1.3 OpenSea API Integration Tools

**Overview:**  
A collection of HTTP Request nodes configured as AI tools to query specific OpenSea API endpoints for NFT marketplace data.

**Nodes Involved:**  
- OpenSea Get All Listings by Collection  
- OpenSea Get All Offers by Collection  
- OpenSea Get Best Listing by NFT  
- OpenSea Get Best Listings by Collection  
- OpenSea Get Best Offer by NFT  
- OpenSea Get Collection Offers  
- OpenSea Get Item Offers  
- OpenSea Get Listings  
- OpenSea Get Order  
- OpenSea Get Trait Offers

**Node Details:**  
Each node shares common characteristics:

- Type: `toolHttpRequest` (LangChain HTTP Request node)  
- Role: Executes HTTP GET requests to OpenSea API endpoints with dynamic URL parameters and query strings.  
- Authentication: Uses HTTP Header Authentication with OpenSea API key credential.  
- Headers: Sets `Accept: application/json`.  
- Query Parameters: Configured to accept optional parameters such as `limit`, `next` (pagination), `include_private_listings`, and others depending on the endpoint.  
- Input: Receives parameters from the AI agent (e.g., collection slug, NFT identifier, chain, protocol, order hash).  
- Output: Returns JSON responses from OpenSea API for further processing or delivery.  
- Edge Cases:  
  - Incorrect or missing parameters (e.g., wrong collection slug, invalid chain name) cause 400 or 404 errors.  
  - API rate limits or server errors (500) require retry logic or error handling.  
  - Chain names must follow OpenSea's allowed list (e.g., use `matic` instead of `polygon`).  
  - Protocol must be `"seaport"` where required.  
  - Fixed protocol address for order queries must be used exactly.  
- Version Specifics: Compatible with n8n LangChain nodes v1.1.

Individual nodes:

- **OpenSea Get All Listings by Collection**  
  - Endpoint: `/api/v2/listings/collection/{collection_slug}/all`  
  - Retrieves all active listings for a collection with pagination.

- **OpenSea Get All Offers by Collection**  
  - Endpoint: `/api/v2/offers/collection/{collection_slug}/all`  
  - Retrieves all valid offers for a collection.

- **OpenSea Get Best Listing by NFT**  
  - Endpoint: `/api/v2/listings/collection/{collection_slug}/nfts/{identifier}/best`  
  - Retrieves the cheapest listing for a specific NFT.

- **OpenSea Get Best Listings by Collection**  
  - Endpoint: `/api/v2/listings/collection/{collection_slug}/best`  
  - Retrieves cheapest listings for a collection.

- **OpenSea Get Best Offer by NFT**  
  - Endpoint: `/api/v2/offers/collection/{collection_slug}/nfts/{identifier}/best`  
  - Retrieves highest offer for a specific NFT.

- **OpenSea Get Collection Offers**  
  - Endpoint: `/api/v2/offers/collection/{collection_slug}`  
  - Retrieves all active collection-wide offers.

- **OpenSea Get Item Offers**  
  - Endpoint: `/api/v2/orders/{chain}/{protocol}/offers`  
  - Retrieves individual active offers filtered by chain and protocol.

- **OpenSea Get Listings**  
  - Endpoint: `/api/v2/orders/{chain}/{protocol}/listings`  
  - Retrieves active listings filtered by chain and protocol.

- **OpenSea Get Order**  
  - Endpoint: `/api/v2/orders/chain/{chain}/protocol/0x0000000000000068f116a894984e2db1123eb395/{order_hash}`  
  - Retrieves a specific order by order hash.

- **OpenSea Get Trait Offers**  
  - Endpoint: `/api/v2/offers/collection/{collection_slug}/traits`  
  - Retrieves offers for specific NFT traits.

---

#### 1.4 Agent Orchestration

**Overview:**  
The OpenSea Marketplace Agent node orchestrates the entire workflow by connecting the AI brain, memory, and all API tools. It interprets user queries, selects the appropriate API tool, and returns structured insights.

**Nodes Involved:**  
- OpenSea Marketplace Agent

**Node Details:**  
- Type: `agent` (LangChain agent node)  
- Role: Central AI agent that receives user input (`message`), uses the AI brain and memory to understand intent, and routes requests to the correct OpenSea API tool node.  
- Configuration:  
  - Input text expression: `={{ $json.message }}` to dynamically pass user queries.  
  - System message includes detailed instructions and usage guidelines for all OpenSea API tools, including parameter requirements, allowed chains, protocols, and example queries.  
  - Prompt type: `define` (custom system prompt).  
- Input: Receives user queries from the Workflow Input Trigger.  
- Output: Sends requests to one or more API tools based on parsed intent, collects responses, and generates final output.  
- Connections:  
  - Connected to all OpenSea API tool nodes as AI tools.  
  - Connected to Marketplace Agent Brain as AI language model.  
  - Connected to Marketplace Agent Memory for session context.  
- Edge Cases:  
  - Misinterpretation of user queries may lead to incorrect API calls.  
  - Missing or invalid parameters cause API errors.  
  - API downtime or rate limits affect response availability.  
  - Complex queries requiring multiple API calls may increase latency.

---

#### 1.5 Documentation & Notes

**Overview:**  
Sticky Notes provide detailed documentation, usage instructions, error handling guidance, and support links embedded within the workflow canvas.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**  
- **Sticky Note**  
  - Content: Comprehensive workflow guide including overview, key features, node descriptions, and API endpoint summaries.  
  - Purpose: Serves as an embedded reference manual for users and developers.

- **Sticky Note1**  
  - Content: Step-by-step usage instructions and common example API queries with endpoint URLs.  
  - Purpose: Helps users understand how to interact with the workflow and what inputs to provide.

- **Sticky Note2**  
  - Content: Error codes, troubleshooting tips, and contact information for support.  
  - Purpose: Assists in diagnosing common issues and provides a support channel.

---

### 3. Summary Table

| Node Name                      | Node Type                                  | Functional Role                            | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                              |
|--------------------------------|--------------------------------------------|--------------------------------------------|---------------------------|------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow Input Trigger          | executeWorkflowTrigger                      | Entry point for user queries                | External trigger          | OpenSea Marketplace Agent     | See Sticky Note1 for input usage and example queries                                                    |
| Marketplace Agent Brain         | lmChatOpenAi                               | AI language model processing user queries  | OpenSea Marketplace Agent | OpenSea Marketplace Agent     | See Sticky Note for AI brain description                                                               |
| Marketplace Agent Memory        | memoryBufferWindow                         | Maintains session context                    | OpenSea Marketplace Agent | OpenSea Marketplace Agent     | See Sticky Note for memory role                                                                         |
| OpenSea Marketplace Agent       | agent                                     | Orchestrates AI processing and API calls    | Workflow Input Trigger, Marketplace Agent Brain, Marketplace Agent Memory | All OpenSea API tools               | See Sticky Note for agent overview and API tool usage                                                  |
| OpenSea Get All Listings by Collection | toolHttpRequest                      | Fetches all active listings for a collection | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get All Offers by Collection | toolHttpRequest                      | Fetches all active offers for a collection  | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get Best Listing by NFT | toolHttpRequest                           | Fetches cheapest listing for a specific NFT | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get Best Listings by Collection | toolHttpRequest                    | Fetches cheapest listings for a collection  | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get Best Offer by NFT   | toolHttpRequest                           | Fetches highest offer for a specific NFT    | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get Collection Offers   | toolHttpRequest                           | Fetches all active collection-wide offers   | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get Item Offers         | toolHttpRequest                           | Fetches individual active offers by chain/protocol | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get Listings            | toolHttpRequest                           | Fetches active listings by chain/protocol   | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get Order               | toolHttpRequest                           | Fetches order details by order hash          | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| OpenSea Get Trait Offers        | toolHttpRequest                           | Fetches active offers for specific NFT traits | OpenSea Marketplace Agent |                              | See Sticky Note for API endpoint details                                                               |
| Sticky Note                    | stickyNote                                | Documentation and workflow guide             | None                      | None                         | Contains comprehensive workflow overview and node descriptions                                         |
| Sticky Note1                   | stickyNote                                | Usage instructions and example API queries  | None                      | None                         | Contains step-by-step usage and example API calls                                                     |
| Sticky Note2                   | stickyNote                                | Error handling and support information       | None                      | None                         | Contains error codes, troubleshooting tips, and LinkedIn contact link                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Workflow Input Trigger Node**  
   - Type: `executeWorkflowTrigger`  
   - Parameters: Define workflow inputs `message` (string) and `sessionId` (string).  
   - Position: Place near the left side as entry point.

2. **Create Marketplace Agent Brain Node**  
   - Type: `lmChatOpenAi` (LangChain OpenAI node)  
   - Parameters:  
     - Model: Select `gpt-4o-mini`  
     - No additional options needed.  
   - Credentials: Link to your OpenAI API key credential.  
   - Position: Left-center.

3. **Create Marketplace Agent Memory Node**  
   - Type: `memoryBufferWindow` (LangChain memory node)  
   - Parameters: Use default settings (buffer window memory).  
   - Position: Near Marketplace Agent Brain.

4. **Create OpenSea Marketplace Agent Node**  
   - Type: `agent` (LangChain agent node)  
   - Parameters:  
     - Text: Set expression `={{ $json.message }}` to pass user query dynamically.  
     - System Message: Paste detailed system prompt describing all OpenSea API tools, usage rules, allowed chains, protocols, and example queries (as provided in the original system message).  
     - Prompt Type: Set to `define`.  
   - Position: Center-right.  
   - Connect inputs:  
     - Connect `Workflow Input Trigger` main output to this node’s main input.  
     - Connect `Marketplace Agent Brain` as `ai_languageModel` input.  
     - Connect `Marketplace Agent Memory` as `ai_memory` input.

5. **Create OpenSea API Tool Nodes**  
   For each OpenSea API endpoint, create a `toolHttpRequest` node with the following setup:

   - **Common Settings:**  
     - Authentication: HTTP Header Authentication  
     - Credential: Use OpenSea API key credential (configured with your API key).  
     - Headers: Add header `Accept: application/json`.  
     - Method: GET  
     - Send Query and Headers: Enabled  
     - Parameters Query: Add parameters as per endpoint requirements (e.g., `limit`, `next`, `collection_slug`, `identifier`, `chain`, `protocol`, `order_hash`). Use expression or modelOptional where applicable.

   - **Individual Nodes:**  
     - **OpenSea Get All Listings by Collection**  
       URL: `https://api.opensea.io/api/v2/listings/collection/{collection_slug}/all`  
       Parameters: `limit`, `next` (optional)  
     - **OpenSea Get All Offers by Collection**  
       URL: `https://api.opensea.io/api/v2/offers/collection/{collection_slug}/all`  
       Parameters: `limit`, `next` (optional)  
     - **OpenSea Get Best Listing by NFT**  
       URL: `https://api.opensea.io/api/v2/listings/collection/{collection_slug}/nfts/{identifier}/best`  
       Parameters: `include_private_listings` (optional)  
     - **OpenSea Get Best Listings by Collection**  
       URL: `https://api.opensea.io/api/v2/listings/collection/{collection_slug}/best`  
       Parameters: `include_private_listings`, `limit`, `next` (optional)  
     - **OpenSea Get Best Offer by NFT**  
       URL: `https://api.opensea.io/api/v2/offers/collection/{collection_slug}/nfts/{identifier}/best`  
       No query parameters required.  
     - **OpenSea Get Collection Offers**  
       URL: `https://api.opensea.io/api/v2/offers/collection/{collection_slug}`  
       No query parameters required.  
     - **OpenSea Get Item Offers**  
       URL: `https://api.opensea.io/api/v2/orders/{chain}/{protocol}/offers`  
       Parameters: Multiple optional filters (e.g., `asset_contract_address`, `cursor`, `limit`, etc.)  
     - **OpenSea Get Listings**  
       URL: `https://api.opensea.io/api/v2/orders/{chain}/{protocol}/listings`  
       Parameters: Similar to Item Offers node.  
     - **OpenSea Get Order**  
       URL: `https://api.opensea.io/api/v2/orders/chain/{chain}/protocol/0x0000000000000068f116a894984e2db1123eb395/{order_hash}`  
       Parameters: `chain`, `order_hash`  
     - **OpenSea Get Trait Offers**  
       URL: `https://api.opensea.io/api/v2/offers/collection/{collection_slug}/traits`  
       Parameters: `float_value`, `int_value`, `type`, `value` (optional)

   - Position: Arrange horizontally or vertically for clarity.

6. **Connect OpenSea API Tool Nodes to OpenSea Marketplace Agent**  
   - For each API tool node, connect its input as an AI tool to the OpenSea Marketplace Agent node’s `ai_tool` input.  
   - This allows the agent to invoke these tools dynamically based on user queries.

7. **Add Sticky Notes for Documentation** (Optional but recommended)  
   - Create `stickyNote` nodes with content describing workflow overview, usage instructions, error handling, and support links as per the original workflow.  
   - Position them clearly for user reference.

8. **Configure Credentials**  
   - Add OpenAI API key credential for the Marketplace Agent Brain node.  
   - Add OpenSea API key credential for all HTTP Request nodes using HTTP Header Authentication.

9. **Test the Workflow**  
   - Trigger the workflow with sample input JSON:  
     ```json
     { "message": "Show me the 10 cheapest listings for Bored Ape Yacht Club.", "sessionId": "test-session-1" }
     ```  
   - Verify that the AI agent routes the request to the correct API tool and returns structured data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow requires an OpenSea API key, which can be obtained by signing up at https://docs.opensea.io/reference/api-keys          | OpenSea API Key Setup                                                                              |
| Use `matic` instead of `polygon` for chain parameters to avoid API errors.                                                           | Chain Naming Conventions                                                                           |
| Protocol must always be set to `"seaport"` for item and listing queries.                                                              | Protocol Requirement                                                                              |
| For order hash queries, the protocol address must be fixed to `0x0000000000000068f116a894984e2db1123eb395`.                           | Order Hash Query Protocol Address                                                                 |
| Common HTTP error codes returned by OpenSea API: 200 (Success), 400 (Bad Request), 404 (Not Found), 500 (Server Error)                | Error Handling                                                                                     |
| LinkedIn contact for support and custom automation: Don Jayamaha - http://linkedin.com/in/donjayamahajr                                | Support Contact                                                                                   |
| Example queries include: "Get the best listing for BAYC #5678", "Find the highest bid for CryptoPunk #1234", "Retrieve order details" | Example User Queries                                                                              |
| This workflow supports multi-chain NFT data retrieval including Ethereum, Polygon (Matic), Arbitrum, Optimism, and others.             | Multi-Chain Support                                                                               |

---

This documentation provides a complete, structured understanding of the OpenSea Marketplace Agent Tool workflow, enabling advanced users and AI agents to comprehend, reproduce, and extend the system confidently.