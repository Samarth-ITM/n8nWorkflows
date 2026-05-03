🛠️ X (Formerly Twitter) Tool MCP Server 💪 all 8 operations

https://n8nworkflows.xyz/workflows/----x--formerly-twitter--tool-mcp-server----all-8-operations-5076


# 🛠️ X (Formerly Twitter) Tool MCP Server 💪 all 8 operations

### 1. Workflow Overview

This workflow, titled **"🛠️ X (Formerly Twitter) Tool MCP Server 💪 all 8 operations"**, is designed to serve as a multi-operation control point (MCP) for managing various user interactions and content operations on the social platform X (formerly Twitter). It is intended to facilitate automated handling of eight distinct Twitter operations through a centralized webhook trigger, enabling scalable and modular automation.

The logical structure can be divided into the following blocks:

- **1.1 Input Reception:**  
  Reception of incoming requests via the MCP trigger node, which acts as the single entry point to the workflow.

- **1.2 Twitter API Operation Nodes:**  
  Eight separate Twitter Tool nodes, each dedicated to a specific Twitter API operation:
  - Create Direct Message
  - Add Member to List
  - Create Tweet
  - Delete Tweet
  - Like Tweet
  - Retweet Tweet
  - Search Tweets
  - Get User

Each operation node is connected directly from the MCP trigger node, which routes the incoming request payload to the appropriate Twitter operation.

- **1.3 Annotation Blocks:**  
  Several sticky note nodes are placed near operation clusters for documentation or explanation purposes (content currently empty).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives and triggers the workflow upon HTTP webhook calls. It serves as the command and control entry point for all downstream Twitter operations.

- **Nodes Involved:**  
  - X (Formerly Twitter) Tool MCP Server

- **Node Details:**  

  - **Node Name:** X (Formerly Twitter) Tool MCP Server  
  - **Type:** MCP Trigger (@n8n/n8n-nodes-langchain.mcpTrigger)  
  - **Configuration:**  
    - Configured with a webhook ID to accept inbound HTTP requests.  
    - No additional parameters specified, implying the node expects a dynamic input payload to route commands.  
  - **Expressions/Variables:**  
    - Not explicitly configured with expressions; presumably, the incoming payload contains an operation identifier and parameters.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connected to all eight Twitter Tool nodes via the `ai_tool` connection property, enabling dynamic routing of operations.  
  - **Version Requirements:** MCP Trigger node requires n8n version supporting Langchain MCP nodes (v0.156+).  
  - **Potential Failure Points:**  
    - Webhook authorization or connectivity errors.  
    - Malformed or missing payload data causing downstream errors.  
  - **Sub-workflow:** None.

#### 1.2 Twitter API Operation Nodes

This block contains eight nodes, each responsible for invoking a specific Twitter API operation via the official Twitter Tool integration.

- **Overview:**  
  Each node represents an atomic Twitter operation. They are all activated by the MCP trigger node, which passes relevant parameters. This modular approach allows for flexible routing and independent operation handling.

- **Nodes Involved:**  
  - Create Direct Message  
  - Add Member to List  
  - Create Tweet  
  - Delete Tweet  
  - Like Tweet  
  - Retweet Tweet  
  - Search Tweets  
  - Get User

- **Node Details:**

  Each node shares similar characteristics but targets different Twitter API endpoints. Below are their detailed descriptions:

  1. **Create Direct Message**  
     - **Type:** Twitter Tool node (v2)  
     - **Role:** Sends a direct message to a specified user.  
     - **Configuration:** No explicit parameters set in JSON; expects input data from MCP trigger with recipient and message content.  
     - **Connections:** Input from MCP trigger node. No further outputs.  
     - **Failure Points:** Authentication errors (OAuth token), invalid recipient ID, message length violations, rate limits.  

  2. **Add Member to List**  
     - **Type:** Twitter Tool node (v2)  
     - **Role:** Adds a user to a specified Twitter List.  
     - **Configuration:** Parameters expected from input data, such as list ID and user ID.  
     - **Connections:** Input from MCP trigger node.  
     - **Failure Points:** Authorization issues, non-existent list or user, rate limits.  

  3. **Create Tweet**  
     - **Type:** Twitter Tool node (v2)  
     - **Role:** Posts a new tweet on authenticated user’s timeline.  
     - **Configuration:** Requires tweet content from input.  
     - **Connections:** Input from MCP trigger node.  
     - **Failure Points:** Tweet content invalid or too long, auth issues, rate limits.  

  4. **Delete Tweet**  
     - **Type:** Twitter Tool node (v2)  
     - **Role:** Deletes a specified tweet by tweet ID.  
     - **Configuration:** Expects tweet ID from input data.  
     - **Connections:** Input from MCP trigger node.  
     - **Failure Points:** Tweet not found, permission denied, auth errors.  

  5. **Like Tweet**  
     - **Type:** Twitter Tool node (v2)  
     - **Role:** Likes a tweet specified by tweet ID on behalf of the authenticated user.  
     - **Configuration:** Requires tweet ID.  
     - **Connections:** Input from MCP trigger node.  
     - **Failure Points:** Tweet ID invalid, already liked, auth errors.  

  6. **Retweet Tweet**  
     - **Type:** Twitter Tool node (v2)  
     - **Role:** Retweets a specified tweet.  
     - **Configuration:** Requires tweet ID input.  
     - **Connections:** Input from MCP trigger node.  
     - **Failure Points:** Tweet not found, retweet already exists, auth errors.  

  7. **Search Tweets**  
     - **Type:** Twitter Tool node (v2)  
     - **Role:** Searches for tweets matching specified criteria.  
     - **Configuration:** Query parameters expected from input (e.g., keywords, filters).  
     - **Connections:** Input from MCP trigger node.  
     - **Failure Points:** Invalid query syntax, rate limits, auth errors.  

  8. **Get User**  
     - **Type:** Twitter Tool node (v2)  
     - **Role:** Retrieves user profile information by username or user ID.  
     - **Configuration:** User identifier expected from input.  
     - **Connections:** Input from MCP trigger node.  
     - **Failure Points:** User not found, auth errors.

#### 1.3 Annotation Blocks (Sticky Notes)

- **Overview:**  
  Sticky Notes are positioned near clusters of nodes—most appear empty but are intended for descriptive comments or instructions.

- **Nodes Involved:**  
  - Workflow Overview 0  
  - Sticky Note 1 (near Create Direct Message)  
  - Sticky Note 2 (near Add Member to List)  
  - Sticky Note 3 (near Create Tweet and related nodes)  
  - Sticky Note 4 (near Get User)  

- **Node Details:**  
  These nodes are visual aids only and contain no operational code or expressions.

---

### 3. Summary Table

| Node Name                    | Node Type                        | Functional Role                      | Input Node(s)                | Output Node(s)                | Sticky Note                                   |
|------------------------------|---------------------------------|------------------------------------|-----------------------------|------------------------------|-----------------------------------------------|
| Workflow Overview 0           | Sticky Note                     | Documentation                      | None                        | None                         |                                               |
| X (Formerly Twitter) Tool MCP Server | MCP Trigger (@n8n/nodes-langchain.mcpTrigger) | Entry point, routes operation requests | None                        | Create Direct Message, Add Member to List, Create Tweet, Delete Tweet, Like Tweet, Retweet Tweet, Search Tweets, Get User |                                               |
| Create Direct Message         | Twitter Tool (v2)               | Send direct message                 | X (Formerly Twitter) Tool MCP Server | None                         |                                               |
| Add Member to List            | Twitter Tool (v2)               | Add user to Twitter list           | X (Formerly Twitter) Tool MCP Server | None                         |                                               |
| Create Tweet                 | Twitter Tool (v2)               | Post a tweet                       | X (Formerly Twitter) Tool MCP Server | None                         |                                               |
| Delete Tweet                 | Twitter Tool (v2)               | Delete tweet                      | X (Formerly Twitter) Tool MCP Server | None                         |                                               |
| Like Tweet                   | Twitter Tool (v2)               | Like a tweet                      | X (Formerly Twitter) Tool MCP Server | None                         |                                               |
| Retweet Tweet                | Twitter Tool (v2)               | Retweet a tweet                   | X (Formerly Twitter) Tool MCP Server | None                         |                                               |
| Search Tweets                | Twitter Tool (v2)               | Search tweets                    | X (Formerly Twitter) Tool MCP Server | None                         |                                               |
| Get User                    | Twitter Tool (v2)               | Retrieve user profile             | X (Formerly Twitter) Tool MCP Server | None                         |                                               |
| Sticky Note 1                | Sticky Note                     | Documentation                     | None                        | None                         |                                               |
| Sticky Note 2                | Sticky Note                     | Documentation                     | None                        | None                         |                                               |
| Sticky Note 3                | Sticky Note                     | Documentation                     | None                        | None                         |                                               |
| Sticky Note 4                | Sticky Note                     | Documentation                     | None                        | None                         |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node:**  
   - Add node: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: `X (Formerly Twitter) Tool MCP Server`  
   - Configure webhook with a unique webhook ID or leave default to auto-generate.  
   - No additional parameters needed.  
   - This node will be the single entry point for all Twitter operations.

2. **Add Twitter Tool Nodes for Each Operation:**

   For each operation below, create a node of type `Twitter Tool (v2)`, configure the credentials for Twitter OAuth2, and set parameters to be received from input data:

   - **Create Direct Message**  
     - Node Name: `Create Direct Message`  
     - Set parameters to accept recipient user ID and message text dynamically from input.  
     - Connect input from `X (Formerly Twitter) Tool MCP Server`.  

   - **Add Member to List**  
     - Node Name: `Add Member to List`  
     - Accept list ID and user ID as dynamic input parameters.  
     - Connect input from MCP trigger node.  

   - **Create Tweet**  
     - Node Name: `Create Tweet`  
     - Accept tweet content dynamically.  
     - Connect input from MCP trigger node.  

   - **Delete Tweet**  
     - Node Name: `Delete Tweet`  
     - Accept tweet ID dynamically.  
     - Connect input from MCP trigger node.  

   - **Like Tweet**  
     - Node Name: `Like Tweet`  
     - Accept tweet ID dynamically.  
     - Connect input from MCP trigger node.  

   - **Retweet Tweet**  
     - Node Name: `Retweet Tweet`  
     - Accept tweet ID dynamically.  
     - Connect input from MCP trigger node.  

   - **Search Tweets**  
     - Node Name: `Search Tweets`  
     - Accept search query parameters dynamically (e.g., keywords, filters).  
     - Connect input from MCP trigger node.  

   - **Get User**  
     - Node Name: `Get User`  
     - Accept username or user ID dynamically.  
     - Connect input from MCP trigger node.  

3. **Set Twitter Credentials:**  
   - For each Twitter Tool node, assign valid Twitter OAuth2 credentials with required scopes for the respective operations (e.g., tweet.read, tweet.write, users.read, dm.write).  
   - Verify credential validity before deploying.  

4. **Add Sticky Notes:**  
   - Optionally add sticky notes near each cluster of operations for documentation or instructions.  
   - Their content can be customized as needed; currently, notes are empty.  

5. **Test the Workflow:**  
   - Trigger the workflow via the MCP webhook with payloads specifying the operation and parameters.  
   - Confirm each Twitter Tool node performs the correct API call and handles errors gracefully.  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow leverages the official Twitter Tool integration for n8n supporting Twitter API v2 endpoints. | n8n Documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.twitter/  |
| MCP Trigger node is part of n8n's Langchain integration enabling multi-operation workflows.   | n8n MCP Nodes Documentation: https://docs.n8n.io/integrations/builtin/app-nodes/mcp/             |
| Ensure Twitter Developer account and app have necessary elevated access for these operations. | Twitter Developer Portal: https://developer.twitter.com/en/portal/dashboard                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.