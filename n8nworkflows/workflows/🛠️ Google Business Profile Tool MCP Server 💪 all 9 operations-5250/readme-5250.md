🛠️ Google Business Profile Tool MCP Server 💪 all 9 operations

https://n8nworkflows.xyz/workflows/----google-business-profile-tool-mcp-server----all-9-operations-5250


# 🛠️ Google Business Profile Tool MCP Server 💪 all 9 operations

### 1. Workflow Overview

This workflow, titled **"Google Business Profile Tool MCP Server"**, is designed to serve as a multi-operation server for managing Google Business Profile (GBP) content and interactions. It supports all nine primary operations related to Google Business Profile posts and reviews, facilitating integration with the Google Business Profile API through n8n’s Google Business Profile Tool nodes. The workflow is triggered via a LangChain MCP (Multi-Channel Processing) trigger node, enabling it to handle requests dynamically and route them to the appropriate GBP operation.

**Target Use Cases:**  
- Automated management of Google Business Profile posts (create, update, delete, retrieve single/multiple posts)  
- Automated management of reviews (get single/multiple reviews, reply to reviews, delete replies)  
- Serving as a backend server for a multi-channel platform that requires Google Business Profile integrations  

**Logical Blocks:**  
- **1.1 Trigger Input Reception:** Single MCP trigger node that initiates the workflow upon receiving requests.  
- **1.2 Posts Management Operations:** Nodes handling post creation, retrieval, update, deletion, and bulk retrieval.  
- **1.3 Reviews Management Operations:** Nodes handling review retrieval, bulk retrieval, replying to reviews, and deleting review replies.  
- **1.4 Informational Sticky Notes:** Two sticky notes grouping nodes visually for clarity (no active logic).

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Input Reception

- **Overview:**  
  This block contains the MCP trigger node that acts as the entry point for all incoming requests. It listens for calls to the workflow and passes the request context (including operation type and parameters) to downstream nodes.

- **Nodes Involved:**  
  - Google Business Profile Tool MCP Server (MCP Trigger)

- **Node Details:**  
  - **Google Business Profile Tool MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Workflow entry trigger for multi-channel processing requests.  
    - Configuration: Default MCP trigger with a webhook ID configured for external calls. No additional parameters set.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to all GBP operation nodes via `ai_tool` output.  
    - Edge Cases / Failures:  
      - Webhook connectivity issues or missing webhook ID.  
      - Malformed or missing request payload resulting in downstream failures.  
      - Authentication or permission failures if credentials are misconfigured (handled downstream).  
    - Sub-workflow: None.

---

#### 1.2 Posts Management Operations

- **Overview:**  
  This block manages all operations related to Google Business Profile posts, including creating, retrieving, updating, deleting single posts, and listing multiple posts.

- **Nodes Involved:**  
  - Create post  
  - Delete post  
  - Get post  
  - Get many posts  
  - Update a post  
  - Sticky Note 1 (visual grouping)

- **Node Details:**  

  - **Create post**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Creates a new post on a Google Business Profile.  
    - Configuration: Uses default Google Business Profile Tool node with operation set to "create post". Parameters (such as post content, media, and location) must be supplied dynamically or via previous nodes (not shown in this workflow).  
    - Inputs: Connected from MCP trigger node output.  
    - Outputs: None specified (end operation).  
    - Edge Cases / Failures:  
      - Invalid or incomplete post data.  
      - API quota exceeded or authentication errors.  
      - Network timeouts.  
    - Sub-workflow: None.

  - **Delete post**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Deletes a specified post from a Google Business Profile.  
    - Configuration: Operation set as "delete post," requires post identifier as input.  
    - Inputs: From MCP trigger node.  
    - Outputs: None specified.  
    - Edge Cases / Failures:  
      - Attempting to delete a non-existent post.  
      - Permission denied errors.  
      - Network/API issues.  
    - Sub-workflow: None.

  - **Get post**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Retrieves details of a single post by ID.  
    - Configuration: Operation "get post." Requires post ID as input.  
    - Inputs: MCP trigger node.  
    - Outputs: None specified.  
    - Edge Cases / Failures:  
      - Post ID not found.  
      - API limit errors.  
    - Sub-workflow: None.

  - **Get many posts**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Retrieves multiple posts, typically with pagination.  
    - Configuration: Operation "get many posts." May accept parameters like page size or filters.  
    - Inputs: MCP trigger node.  
    - Outputs: None specified.  
    - Edge Cases / Failures:  
      - No posts found.  
      - Pagination misconfiguration.  
    - Sub-workflow: None.

  - **Update a post**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Updates content or metadata of an existing post.  
    - Configuration: Operation "update post" with required post ID and update data.  
    - Inputs: MCP trigger node.  
    - Outputs: None specified.  
    - Edge Cases / Failures:  
      - Invalid update fields.  
      - Post ID not found or locked.  
    - Sub-workflow: None.

  - **Sticky Note 1**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Role: Visual separator for post-related nodes.  
    - Content: Empty.  
    - No logic impact.

---

#### 1.3 Reviews Management Operations

- **Overview:**  
  This block handles Google Business Profile review operations, enabling retrieval, replying, and deleting replies for reviews.

- **Nodes Involved:**  
  - Delete a reply to a review  
  - Get review  
  - Get many reviews  
  - Reply to review  
  - Sticky Note 2 (visual grouping)

- **Node Details:**  

  - **Delete a reply to a review**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Deletes a reply made to a specific review.  
    - Configuration: Operation "delete reply," requires review ID and reply ID.  
    - Inputs: From MCP trigger node.  
    - Outputs: None specified.  
    - Edge Cases / Failures:  
      - Reply or review not found.  
      - Permission errors.  
    - Sub-workflow: None.

  - **Get review**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Retrieves a specific review by ID.  
    - Configuration: Operation "get review," requires review ID.  
    - Inputs: MCP trigger node.  
    - Outputs: None specified.  
    - Edge Cases / Failures:  
      - Review not found.  
      - API quota or authentication issues.  
    - Sub-workflow: None.

  - **Get many reviews**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Retrieves multiple reviews, supporting pagination or filtering.  
    - Configuration: Operation "get many reviews."  
    - Inputs: MCP trigger node.  
    - Outputs: None specified.  
    - Edge Cases / Failures:  
      - No reviews available.  
      - Pagination parameters missing or invalid.  
    - Sub-workflow: None.

  - **Reply to review**  
    - Type: `n8n-nodes-base.googleBusinessProfileTool`  
    - Role: Posts a reply to a specific review.  
    - Configuration: Operation "reply to review," requires review ID and reply content.  
    - Inputs: MCP trigger node.  
    - Outputs: None specified.  
    - Edge Cases / Failures:  
      - Reply content invalid or empty.  
      - Review ID not found.  
    - Sub-workflow: None.

  - **Sticky Note 2**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Role: Visual separator for review-related nodes.  
    - Content: Empty.  
    - No logic impact.

---

#### 1.4 Informational Sticky Notes

- **Overview:**  
  Two sticky notes are placed near the post and review nodes respectively, presumably for organizational or annotation purposes. Both contain empty content.

- **Nodes Involved:**  
  - Workflow Overview 0  
  - Sticky Note 1  
  - Sticky Note 2

- **Node Details:**  
  - All sticky notes have empty content and serve no functional role beyond visual grouping.

---

### 3. Summary Table

| Node Name                      | Node Type                                | Functional Role                          | Input Node(s)                     | Output Node(s)                    | Sticky Note |
|--------------------------------|-----------------------------------------|----------------------------------------|----------------------------------|---------------------------------|-------------|
| Workflow Overview 0             | Sticky Note                             | Visual/organizational                   | None                             | None                            |             |
| Google Business Profile Tool MCP Server | MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`) | Entry point for all GBP operations     | None                             | All GBP operation nodes         |             |
| Create post                    | Google Business Profile Tool            | Create a new GBP post                   | MCP Trigger                     | None                            |             |
| Delete post                   | Google Business Profile Tool            | Delete a specified GBP post             | MCP Trigger                     | None                            |             |
| Get post                      | Google Business Profile Tool            | Retrieve details of a single post       | MCP Trigger                     | None                            |             |
| Get many posts                | Google Business Profile Tool            | Retrieve multiple posts                  | MCP Trigger                     | None                            |             |
| Update a post                 | Google Business Profile Tool            | Update an existing post                  | MCP Trigger                     | None                            |             |
| Sticky Note 1                 | Sticky Note                            | Visual grouping for post nodes           | None                             | None                            |             |
| Delete a reply to a review     | Google Business Profile Tool            | Delete a reply to a review               | MCP Trigger                     | None                            |             |
| Get review                   | Google Business Profile Tool            | Retrieve a single review                  | MCP Trigger                     | None                            |             |
| Get many reviews             | Google Business Profile Tool            | Retrieve multiple reviews                 | MCP Trigger                     | None                            |             |
| Reply to review              | Google Business Profile Tool            | Reply to a specific review                | MCP Trigger                     | None                            |             |
| Sticky Note 2                 | Sticky Note                            | Visual grouping for review nodes          | None                             | None                            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set the timezone to America/New_York.

2. **Add MCP Trigger Node**  
   - Name: `Google Business Profile Tool MCP Server`  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Configuration: Default. Ensure webhook ID is generated (or specify one). This node serves as the entry point for all requests.  
   - No credentials required here as this is a trigger.

3. **Add Google Business Profile Tool Nodes for Post Operations**  
   For each operation below, add a node of type `Google Business Profile Tool` and configure the operation accordingly:  
   
   - **Create post**  
     - Operation: Create Post  
     - Configure parameters as needed (e.g., Post content, media, location) to accept dynamic input from MCP trigger.  
     - Connect input from the MCP Trigger node’s output.

   - **Delete post**  
     - Operation: Delete Post  
     - Requires Post ID parameter.  
     - Connect input from MCP Trigger node.

   - **Get post**  
     - Operation: Get Post  
     - Requires Post ID parameter.  
     - Connect input from MCP Trigger node.

   - **Get many posts**  
     - Operation: Get Many Posts  
     - Optional parameters for filtering or pagination.  
     - Connect input from MCP Trigger node.

   - **Update a post**  
     - Operation: Update Post  
     - Requires Post ID and update fields.  
     - Connect input from MCP Trigger node.

4. **Add Google Business Profile Tool Nodes for Review Operations**  

   - **Delete a reply to a review**  
     - Operation: Delete Reply  
     - Requires Review ID and Reply ID.  
     - Connect input from MCP Trigger node.

   - **Get review**  
     - Operation: Get Review  
     - Requires Review ID.  
     - Connect input from MCP Trigger node.

   - **Get many reviews**  
     - Operation: Get Many Reviews  
     - Optional parameters for pagination/filtering.  
     - Connect input from MCP Trigger node.

   - **Reply to review**  
     - Operation: Reply to Review  
     - Requires Review ID and Reply content.  
     - Connect input from MCP Trigger node.

5. **Set up Google Business Profile credentials** in n8n credentials manager:  
   - Use OAuth2 credentials with appropriate scopes to access Google Business Profile API.  
   - Assign the credentials to all Google Business Profile Tool nodes.

6. **Add Sticky Notes for Visual Grouping** (optional)  
   - Add sticky notes near post nodes and review nodes to visually separate logic blocks.

7. **Test the workflow** by triggering the MCP node with various operation requests and verify correct execution of each Google Business Profile operation.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow serves as a comprehensive server for Google Business Profile operations via n8n and MCP.    | Project description                              |
| Google Business Profile Tool node requires proper OAuth2 credentials with Google Business Profile API access. | Credential setup instructions                    |
| MCP Trigger node enables multi-channel integration and flexible request handling.                         | n8n LangChain MCP documentation                  |
| Workflow timezone is set to America/New_York for consistent time-based operations.                        | Workflow settings                                |

---

**Disclaimer:**  
The provided description and documentation are solely based on an automated n8n workflow. All content adheres to legal and ethical guidelines and does not contain any unauthorized or protected data.