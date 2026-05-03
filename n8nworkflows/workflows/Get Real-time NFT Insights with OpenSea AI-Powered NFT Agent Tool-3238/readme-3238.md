Get Real-time NFT Insights with OpenSea AI-Powered NFT Agent Tool

https://n8nworkflows.xyz/workflows/get-real-time-nft-insights-with-opensea-ai-powered-nft-agent-tool-3238


# Get Real-time NFT Insights with OpenSea AI-Powered NFT Agent Tool

### 1. Workflow Overview

The **OpenSea NFT Agent Tool** is an AI-powered n8n workflow designed to provide real-time, structured insights on NFTs by integrating OpenSea’s API with GPT-4o-mini AI. It targets traders, collectors, and investors who need instant access to NFT metadata, ownership, collections, traits, contracts, and payment token details.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives user queries and session data to initiate processing.
- **1.2 AI Processing Core:** Uses GPT-4o-mini to interpret queries and maintain conversational context.
- **1.3 OpenSea API Tools:** Multiple HTTP Request nodes configured as AI tools to fetch specific NFT data from OpenSea endpoints.
- **1.4 AI Agent Orchestration:** An AI agent node that routes requests to the appropriate OpenSea API tool based on the interpreted user query.
- **1.5 Memory Management:** A memory buffer node to maintain session context across multiple interactions.
- **1.6 Documentation & Guidance:** Sticky notes embedded in the workflow provide detailed usage instructions, examples, and error handling guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures incoming user queries and session identifiers to feed into the AI agent for processing.

- **Nodes Involved:**  
  - Workflow Input Trigger

- **Node Details:**  
  - **Workflow Input Trigger**  
    - Type: `executeWorkflowTrigger`  
    - Role: Entry point for external inputs (e.g., Telegram, webhook, or UI).  
    - Configuration: Accepts two inputs — `message` (user query text) and `sessionId` (session context identifier).  
    - Inputs: External trigger  
    - Outputs: Connected to "OpenSea NFT Agent" node  
    - Edge Cases: Missing or malformed inputs may cause the agent to fail or respond incorrectly.

#### 2.2 AI Processing Core

- **Overview:**  
  This block interprets user queries using GPT-4o-mini, enabling natural language understanding and decision-making on which API tool to invoke.

- **Nodes Involved:**  
  - NFT Agent Brain

- **Node Details:**  
  - **NFT Agent Brain**  
    - Type: `lmChatOpenAi` (LangChain OpenAI Chat Model)  
    - Role: Processes the user query text and generates AI-driven instructions or API call parameters.  
    - Configuration: Uses GPT-4o-mini model with default options.  
    - Credentials: OpenAI API key configured.  
    - Inputs: Receives text from "OpenSea NFT Agent" node (AI agent).  
    - Outputs: Feeds into "OpenSea NFT Agent" for further processing.  
    - Edge Cases: API rate limits, network timeouts, or malformed prompts may cause failures.

#### 2.3 AI Agent Orchestration

- **Overview:**  
  This is the core AI agent node that receives user queries, interprets them, and dynamically routes requests to the appropriate OpenSea API tool nodes.

- **Nodes Involved:**  
  - OpenSea NFT Agent

- **Node Details:**  
  - **OpenSea NFT Agent**  
    - Type: `agent` (LangChain AI Agent)  
    - Role: Acts as the orchestrator, parsing the input message and selecting the correct API tool to call based on the query intent.  
    - Configuration:  
      - Input text expression: `={{ $json.message }}` (uses incoming message)  
      - System message: Detailed instructions defining the agent’s role, available OpenSea API tools, usage rules, blockchain validation, and example queries.  
      - Tools: Connected to multiple OpenSea API HTTP Request nodes.  
    - Inputs: Receives input from "Workflow Input Trigger" and AI model/memory nodes.  
    - Outputs: Sends requests to specific OpenSea API tool nodes.  
    - Edge Cases: Incorrect or ambiguous queries may cause wrong tool selection or API errors. Strict blockchain name validation is enforced to avoid errors.

#### 2.4 OpenSea API Tools

- **Overview:**  
  This block consists of multiple HTTP Request nodes configured as AI tools, each targeting a specific OpenSea API endpoint to retrieve NFT-related data.

- **Nodes Involved:**  
  - OpenSea Get Account  
  - OpenSea Get Collection  
  - OpenSea Get Collections  
  - OpenSea Get Contract  
  - OpenSea Get NFT  
  - OpenSea Get NFTs by Account  
  - OpenSea Get NFTs by Collection  
  - OpenSea Get NFTs by Contract  
  - OpenSea Get Payment Token  
  - OpenSea Get Traits

- **Node Details:**  
  Each node shares similar configuration patterns:

  - Type: `toolHttpRequest` (LangChain HTTP Request Tool)  
  - Role: Executes HTTP GET requests to specific OpenSea API endpoints.  
  - Authentication: Uses HTTP Header Authentication with OpenSea API key.  
  - Headers: `Accept: application/json` (and `x-api-key` for traits node).  
  - URL: Parameterized with placeholders for required path variables (e.g., `{address_or_username}`, `{collection_slug}`, `{chain}`, `{address}`, `{identifier}`).  
  - Query Parameters: Some nodes accept optional query parameters like `limit`, `next`, `collection`, `creator_username`, `order_by`.  
  - Inputs: Called by the AI agent node with parameters extracted from the user query.  
  - Outputs: Return JSON data from OpenSea API to the AI agent for further processing or response generation.  
  - Edge Cases:  
    - API errors (400, 404, 500) due to invalid parameters or server issues.  
    - Blockchain name validation is critical; `polygon` must be replaced with `matic`.  
    - Rate limiting or API key restrictions may cause failures.  
  - Notable: The "OpenSea Get Traits" node explicitly includes the `x-api-key` header with the API key.

#### 2.5 Memory Management

- **Overview:**  
  Maintains conversational context and session data across multiple user queries to enable coherent multi-turn interactions.

- **Nodes Involved:**  
  - NFT Agent Memory

- **Node Details:**  
  - Type: `memoryBufferWindow` (LangChain Memory Node)  
  - Role: Stores recent conversation history and relevant session data for context preservation.  
  - Configuration: Default buffer window settings (no custom parameters).  
  - Inputs: Connected to "OpenSea NFT Agent" node as AI memory.  
  - Outputs: Feeds back into the AI agent for context-aware processing.  
  - Edge Cases: Memory overflow or excessive context length may degrade performance or cause truncation.

#### 2.6 Documentation & Guidance

- **Overview:**  
  Provides embedded documentation, usage instructions, example queries, and error handling tips for users and developers.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**  
  - Type: `stickyNote`  
  - Role: Contains detailed textual information about workflow features, API endpoints, usage examples, error codes, and support links.  
  - Content Highlights:  
    - Workflow overview and key features.  
    - Step-by-step usage instructions.  
    - Common API queries and example URLs.  
    - Error handling guide with HTTP status codes.  
    - Contact information for support (LinkedIn link).  
  - Position: Placed for easy reference within the workflow editor.  
  - Edge Cases: None (informational only).

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                              | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|----------------------------|--------------------------------------|----------------------------------------------|-------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Workflow Input Trigger      | executeWorkflowTrigger                | Receives user query and session input        | External trigger        | OpenSea NFT Agent       |                                                                                                |
| NFT Agent Brain             | lmChatOpenAi                         | AI model processing user queries              | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| NFT Agent Memory            | memoryBufferWindow                   | Maintains session context                      | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea NFT Agent           | agent                              | Orchestrates AI query interpretation and routes to API tools | Workflow Input Trigger, NFT Agent Brain, NFT Agent Memory | Multiple OpenSea API Tools |                                                                                                |
| OpenSea Get Account         | toolHttpRequest                     | Fetches OpenSea user profile                   | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get Collection      | toolHttpRequest                     | Retrieves NFT collection metadata              | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get Collections     | toolHttpRequest                     | Lists NFT collections with filters             | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get Contract        | toolHttpRequest                     | Retrieves smart contract details                | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get NFT             | toolHttpRequest                     | Fetches metadata and traits of a single NFT    | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get NFTs by Account | toolHttpRequest                     | Lists all NFTs owned by a wallet                | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get NFTs by Collection | toolHttpRequest                  | Retrieves NFTs in a specific collection         | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get NFTs by Contract | toolHttpRequest                    | Retrieves NFTs linked to a smart contract       | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get Payment Token   | toolHttpRequest                     | Fetches payment token details                    | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| OpenSea Get Traits          | toolHttpRequest                     | Retrieves trait categories and attributes       | OpenSea NFT Agent       | OpenSea NFT Agent       |                                                                                                |
| Sticky Note                | stickyNote                         | Workflow overview and feature summary           | —                       | —                       | # OpenSea NFT Agent Tool (n8n Workflow) Guide (overview, features, nodes)                      |
| Sticky Note1               | stickyNote                         | Usage instructions and example API queries      | —                       | —                       | How to use the workflow, common API queries & examples                                        |
| Sticky Note2               | stickyNote                         | Error handling and support contact info         | —                       | —                       | Error codes, troubleshooting tips, LinkedIn support link                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Workflow:** Name it "OpenSea NFT Agent Tool".

2. **Add Input Trigger Node:**  
   - Type: `executeWorkflowTrigger`  
   - Configure inputs: Add two parameters named `message` (string) and `sessionId` (string).  
   - Position: Start node.

3. **Add AI Language Model Node (NFT Agent Brain):**  
   - Type: `lmChatOpenAi`  
   - Model: Select `gpt-4o-mini`  
   - Credentials: Configure OpenAI API credentials.  
   - Connect input from "OpenSea NFT Agent" node (to be created).  
   - Position: Right of input trigger.

4. **Add AI Memory Node (NFT Agent Memory):**  
   - Type: `memoryBufferWindow`  
   - Default buffer settings.  
   - Connect input/output to "OpenSea NFT Agent" node.  
   - Position: Near AI Language Model node.

5. **Add AI Agent Node (OpenSea NFT Agent):**  
   - Type: `agent`  
   - Parameters:  
     - Text input expression: `={{ $json.message }}`  
     - System message: Paste the detailed system prompt describing the agent’s role, available OpenSea API tools, usage rules, blockchain validation, and example queries (as provided in the workflow).  
     - Tools: Add all OpenSea API HTTP Request nodes as tools (see below).  
   - Connect input from "Workflow Input Trigger" node.  
   - Connect AI model and memory nodes as AI language model and AI memory respectively.  
   - Position: Center-right.

6. **Add OpenSea API HTTP Request Nodes:** For each of the following, configure:

   - Type: `toolHttpRequest`  
   - Authentication: HTTP Header Authentication with OpenSea API key (create credential with header `x-api-key: YOUR_OPENSEA_API_KEY`).  
   - Headers: Add `Accept: application/json`. For "OpenSea Get Traits" node, also add `x-api-key` header explicitly.  
   - URL and parameters as follows:

   | Node Name                  | URL Template                                                                                   | Query Parameters (optional)                          | Description                                                 |
   |----------------------------|-----------------------------------------------------------------------------------------------|-----------------------------------------------------|-------------------------------------------------------------|
   | OpenSea Get Account         | `https://api.opensea.io/api/v2/accounts/{address_or_username}`                                | None                                                | Fetch user profile by address or username                    |
   | OpenSea Get Collection      | `https://api.opensea.io/api/v2/collections/{collection_slug}`                                | None                                                | Fetch collection metadata                                    |
   | OpenSea Get Collections     | `https://api.opensea.io/api/v2/collections`                                                  | `chain`, `creator_username`, `include_hidden`, `limit`, `next`, `order_by` | List collections with filters                                |
   | OpenSea Get Contract        | `https://api.opensea.io/api/v2/chain/{chain}/contract/{address}`                             | None                                                | Fetch smart contract details                                 |
   | OpenSea Get NFT             | `https://api.opensea.io/api/v2/chain/{chain}/contract/{address}/nfts/{identifier}`           | None                                                | Fetch single NFT metadata and traits                         |
   | OpenSea Get NFTs by Account | `https://api.opensea.io/api/v2/chain/{chain}/account/{address}/nfts`                         | `collection`, `limit`, `next`                        | List NFTs owned by account                                   |
   | OpenSea Get NFTs by Collection | `https://api.opensea.io/api/v2/collection/{collection_slug}/nfts`                          | `limit`, `next`                                     | List NFTs in a collection                                    |
   | OpenSea Get NFTs by Contract | `https://api.opensea.io/api/v2/chain/{chain}/contract/{address}/nfts`                       | `limit`, `next`                                     | List NFTs under a contract                                  |
   | OpenSea Get Payment Token   | `https://api.opensea.io/api/v2/chain/{chain}/payment_token/{address}`                        | None                                                | Fetch payment token details                                 |
   | OpenSea Get Traits          | `https://api.opensea.io/api/v2/traits/{collection_slug}`                                     | None                                                | Fetch trait categories and attributes                        |

7. **Connect all OpenSea API HTTP Request nodes as tools to the "OpenSea NFT Agent" node.**

8. **Connect the "NFT Agent Brain" node as AI language model and "NFT Agent Memory" node as AI memory to the "OpenSea NFT Agent" node.**

9. **Add Sticky Note nodes for documentation:**  
   - Add three sticky notes with content covering workflow overview, usage instructions, example queries, error handling, and support links.  
   - Position them for easy reference.

10. **Configure Credentials:**  
    - OpenAI API credentials for the AI nodes.  
    - HTTP Header Authentication credential for OpenSea API key (used by all HTTP Request nodes).

11. **Validate Blockchain Inputs:**  
    - Ensure that any user input for blockchain names replaces `polygon` with `matic` to comply with OpenSea API requirements.

12. **Test Workflow:**  
    - Trigger the workflow with sample queries such as:  
      - "Get OpenSea profile for 0xA5f49655E6814d9262fb656d92f17D7874d5Ac7E."  
      - "Retrieve details for the 'Azuki' NFT collection."  
      - "Fetch metadata for NFT #5678 from 'Bored Ape Yacht Club'."  
    - Verify that the AI agent correctly routes to the appropriate API tool and returns structured insights.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow strictly enforces blockchain naming conventions; `polygon` must be replaced with `matic`.        | OpenSea API documentation: https://docs.opensea.io/reference/api-keys                           |
| For successful API calls, an OpenSea API key is required and must be configured in n8n HTTP Header Auth.      | OpenSea API Key signup: https://docs.opensea.io/reference/api-keys                               |
| Example user queries and API endpoints are embedded in the system prompt for the AI agent to understand usage.| See Sticky Note and system message in "OpenSea NFT Agent" node                                  |
| Error handling includes HTTP status codes 200, 400, 404, 500 with recommended troubleshooting steps.           | See Sticky Note2 content for detailed error handling and support contact info                   |
| Support and custom automation services are available via LinkedIn contact: Don Jayamaha (http://linkedin.com/in/donjayamahajr) | Support contact link provided in Sticky Note2                                                   |

---

This documentation provides a comprehensive, structured reference for understanding, reproducing, and modifying the OpenSea NFT Agent Tool workflow in n8n, ensuring robust integration with OpenSea’s API and AI-powered query processing.