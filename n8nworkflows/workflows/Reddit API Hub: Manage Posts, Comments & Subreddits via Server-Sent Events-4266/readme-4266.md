Reddit API Hub: Manage Posts, Comments & Subreddits via Server-Sent Events

https://n8nworkflows.xyz/workflows/reddit-api-hub--manage-posts--comments---subreddits-via-server-sent-events-4266


# Reddit API Hub: Manage Posts, Comments & Subreddits via Server-Sent Events

### 1. Workflow Overview

This workflow, titled **"Reddit API Hub: Manage Posts, Comments & Subreddits via Server-Sent Events"**, provides a comprehensive interface to interact programmatically with Reddit. It supports creating, retrieving, searching, deleting, and replying to posts and comments, as well as fetching subreddit information and rules. The workflow is triggered by an external request or another workflow, then routes the operation request to the appropriate logic branch.

**Target use cases include**:  
- Automated Reddit post and comment management  
- Bulk retrieval and filtering of posts, comments, and subreddits  
- Integration with external applications or AI agents for Reddit content operations  
- Dynamic handling of various Reddit API functionalities via a unified interface

**Logical blocks grouped by functionality and node dependencies**:  

- **1.1 Input Reception & Operation Routing**  
  Receives JSON input describing the desired Reddit operation and routes requests to the correct functional block (post, comment, subreddit).

- **1.2 Post Operations Block**  
  Handles CRUD operations on Reddit posts, including creating, deleting, retrieving many posts, getting post by ID, and searching posts.

- **1.3 Comment Operations Block**  
  Manages comment creation, deletion, retrieval, and replying on Reddit posts.

- **1.4 Subreddit Operations Block**  
  Retrieves subreddit information, rules, and lists subreddits by filters.

- **1.5 Data Mapping and Output Preparation**  
  Maps raw Reddit API responses into structured JSON outputs and splits arrays for downstream processing or output.

- **1.6 Sub-Workflow Integration Nodes**  
  These nodes invoke sub-workflows for post, comment, and subreddit operations, encapsulating and modularizing logic.

- **1.7 Trigger & Auxiliary Nodes**  
  The MCP Server Trigger node receives inbound requests; switch nodes route operations based on input commands; sticky notes provide documentation and guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Operation Routing

**Overview:**  
This block accepts input JSON that specifies the Reddit operation to perform, then uses a switch node to route the flow to either post, comment, or subreddit operations.

**Nodes Involved:**  
- MCP Server Trigger  
- When Executed by Another Workflow  
- operation_switch (Switch)

**Node Details:**  

- **MCP Server Trigger**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Entry point accepting Server-Sent Events JSON requests  
  - Configuration: webhook path configured, receives operation JSON  
  - Inputs: External HTTP/Webhook requests or AI tool inputs  
  - Outputs: JSON with operation parameters  
  - Failure Modes: Invalid/malformed JSON, webhook errors, auth failures

- **When Executed by Another Workflow**  
  - Type: `executeWorkflowTrigger`  
  - Role: Enables triggering this workflow from another workflow, passing JSON example with all possible parameters  
  - Configuration: Example JSON schema for operation parameters provided  
  - Inputs: Triggered by other workflows  
  - Outputs: JSON with operation parameters  
  - Failure Modes: Invalid input schema or missing parameters

- **operation_switch**  
  - Type: `Switch` (v3.2)  
  - Role: Routes operation types to one of three main flow paths: post_flow, comment_flow, subreddit_flow  
  - Configuration: Checks if `$json.operation` starts with "post_", "comment_", or "=subreddit_" respectively  
  - Inputs: JSON with `operation` field  
  - Outputs: Main outputs with keys `post_flow`, `comment_flow`, `subreddit_flow`  
  - Failure Modes: Operation string missing or does not match any prefix, causing no valid output path

---

#### 1.2 Post Operations Block

**Overview:**  
This block manages Reddit post-related operations (create, get many, delete, get by id, search). It uses a switch node to select the exact post operation, executes the Reddit API node, then maps and formats results.

**Nodes Involved:**  
- post_switch (Switch)  
- create_post  
- get_many_posts  
- delete_post  
- get_post_by_id  
- search_posts  
- map_post_get_many  
- map_post_get_search  
- posts_split_out  
- post_operation (Sub-workflow tool)

**Node Details:**  

- **post_switch**  
  - Type: `Switch` (v3.2)  
  - Role: Determines specific post operation by matching exact `$json.operation` values (e.g., "post_create")  
  - Outputs: Routes to corresponding Reddit API nodes or search node  
  - Failure Modes: Unknown operation strings, no matching output

- **create_post**  
  - Type: `Reddit` node  
  - Role: Creates a Reddit post on a specified subreddit with title and text  
  - Config: `subreddit`, `title`, and `text` fields dynamically set from input JSON  
  - Credentials: Reddit OAuth2  
  - Inputs: JSON with `postTitle`, `postText`, `subReddit`  
  - Outputs: Created post info JSON  
  - Failures: OAuth token expiry, invalid subreddit, quota limits

- **get_many_posts**  
  - Type: `Reddit` node  
  - Role: Retrieves multiple posts from a subreddit filtered by category (hot, top, etc.) and limited by count  
  - Config: Uses `$json.limit`, `$json.filtersCategory`, and `$json.subReddit`  
  - Outputs: Array of post objects  
  - Failures: Invalid category, rate limiting

- **delete_post**  
  - Type: `Reddit` node  
  - Role: Deletes a Reddit post by its ID  
  - Config: `postId` from `$json.postPostId`  
  - Failures: Unauthorized deletion, invalid post ID

- **get_post_by_id**  
  - Type: `Reddit` node  
  - Role: Fetches a single post by ID and subreddit  
  - Config: `postId` and `subreddit` from JSON  
  - Failures: Post not found, invalid ID

- **search_posts**  
  - Type: `Reddit` node  
  - Role: Searches posts in a subreddit by keyword and limit  
  - Config: `filtersKeyword`, `limit`, `subReddit` from JSON  
  - Failures: Empty search terms, rate limiting

- **map_post_get_many** and **map_post_get_search**  
  - Type: `Set` node  
  - Role: Restructures arrays of posts into simplified JSON objects with selected fields for output  
  - Key fields: subreddit, selftext, author, title, awards, flair, score, thumbnail, created, comments, permalink  
  - Inputs: Output from Reddit API nodes  
  - Outputs: JSON object with key `"posts"` containing mapped array

- **posts_split_out**  
  - Type: `SplitOut` node  
  - Role: Splits each post in the `"posts"` array into separate items for downstream processing  
  - Inputs: JSON with `"posts"` array  
  - Outputs: Individual post JSONs

- **post_operation**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Sub-workflow encapsulating all post-related operations; invoked as AI tool  
  - Inputs: Defined parameters like operation type, subreddit, limit, post ID, etc.  
  - Failure Modes: Sub-workflow errors, parameter mismatches

---

#### 1.3 Comment Operations Block

**Overview:**  
Handles Reddit comment operations including creation, retrieval, deletion, and replying to comments on posts.

**Nodes Involved:**  
- comment_switch (Switch)  
- create_comment  
- get_many_comments  
- delete_comment  
- reply_comment  
- map_comment_get_many  
- split_out_get_many_comments  
- comment_operation (Sub-workflow tool)

**Node Details:**  

- **comment_switch**  
  - Type: `Switch` (v3.2)  
  - Role: Routes exact comment operation (comment_create, comment_get_many, etc.) to relevant nodes  
  - Inputs: JSON with `operation` field  
  - Outputs: Mapped to comment CRUD nodes  
  - Failures: Unknown operations

- **create_comment**  
  - Type: `Reddit` node  
  - Role: Creates a comment on a Reddit post  
  - Config: `postId` and `commentText` from input JSON  
  - Failures: Invalid post ID, permission errors

- **get_many_comments**  
  - Type: `Reddit` node  
  - Role: Retrieves multiple comments of a post with limit and subreddit  
  - Config: `limit`, `postId`, `subreddit` from JSON  
  - Outputs: Array of comment objects  
  - Failures: No comments found, rate limits

- **delete_comment**  
  - Type: `Reddit` node  
  - Role: Deletes a comment by comment ID  
  - Config: `commentId` from JSON  
  - Failures: Unauthorized, invalid comment ID

- **reply_comment**  
  - Type: `Reddit` node  
  - Role: Replies to a specific comment by comment ID  
  - Config: `commentId` and `replyText` from JSON  
  - Failures: Invalid comment ID, permissions

- **map_comment_get_many**  
  - Type: `Set` node  
  - Role: Maps comment array to simplified JSON with selected fields for output  
  - Fields: subreddit, author, author_is_blocked, body, score, awards, created, permalink, replies  
  - Outputs: JSON object with `"comments"` array

- **split_out_get_many_comments**  
  - Type: `SplitOut` node  
  - Role: Splits each comment in `"comments"` array into individual items  
  - Inputs: JSON with `"comments"` array  
  - Outputs: Separate comment JSONs

- **comment_operation**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Sub-workflow encapsulating comment operations for modular use  
  - Inputs: Comment operation parameters similar to main flow

---

#### 1.4 Subreddit Operations Block

**Overview:**  
Enables retrieval of subreddit metadata, rules, and lists subreddits matching filters.

**Nodes Involved:**  
- subreddit_switch (Switch)  
- get_subreddit_about  
- subreddit_get_many  
- get_subreddit_rules  
- subReddit_operation (Sub-workflow tool)

**Node Details:**  

- **subreddit_switch**  
  - Type: `Switch` (v3.2)  
  - Role: Routes subreddit operations (subreddit_get_about, subreddit_get_many, subreddit_get_rules)  
  - Inputs: `$json.operation`  
  - Outputs: Mapped to respective Reddit API nodes  
  - Failures: Unknown operations

- **get_subreddit_about**  
  - Type: `Reddit` node  
  - Role: Retrieves detailed info about a subreddit  
  - Config: `subreddit` from input JSON  
  - Outputs: Subreddit metadata  
  - Failures: Subreddit not found, rate limits

- **subreddit_get_many**  
  - Type: `Reddit` node  
  - Role: Retrieves multiple subreddits filtered by keyword and trending boolean  
  - Config: `limit`, `filtersKeyword`, `filtersTrendig` from JSON  
  - Outputs: Array of subreddit objects  
  - Failures: Empty filters, rate limits

- **get_subreddit_rules**  
  - Type: `Reddit` node  
  - Role: Fetches the rules of a subreddit  
  - Config: `subreddit` and content type "rules"  
  - Outputs: Subreddit rules JSON  
  - Failures: Subreddit without rules, permission issues

- **subReddit_operation**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Sub-workflow managing subreddit operations  
  - Inputs: Operation type, subreddit, filters, limit

---

#### 1.5 Data Mapping and Output Preparation

**Overview:**  
Maps raw API data into clean, structured JSON, and splits arrays for individual item processing or streaming.

**Nodes Involved:**  
- map_post_get_many  
- map_post_get_search  
- map_comment_get_many  
- posts_split_out  
- split_out_get_many_comments

**Node Details:**  

- **Mapping Nodes (Set)**  
  - Purpose: Transform raw response arrays into structured JSON with selected, relevant fields  
  - Use JavaScript expressions to process arrays from Reddit nodes  
  - Output JSON with keys like `"posts"` or `"comments"` containing arrays of mapped objects

- **SplitOut Nodes**  
  - Purpose: Convert array fields (`"posts"`, `"comments"`) into individual flow items for further processing, output, or streaming  
  - Important for workflows expecting item-by-item handling or SSE streaming

---

#### 1.6 Sub-Workflow Integration Nodes

**Overview:**  
These nodes act as AI tool invocations that encapsulate and trigger sub-workflows to perform operations modularly.

**Nodes Involved:**  
- post_operation  
- comment_operation  
- subReddit_operation

**Node Details:**  

- All three nodes are of type `@n8n/n8n-nodes-langchain.toolWorkflow`  
- Each has parameters defining the sub-workflow to invoke (all point to the same underlying workflow, MCP_reddit)  
- They accept defined inputs and provide descriptions for their API contract  
- Role: Encapsulate logic for posts, comments, and subreddit operations, enabling reuse and modularity  
- Failure Modes: Sub-workflow invocation errors, incorrect or incomplete parameters

---

#### 1.7 Trigger & Auxiliary Nodes

**Overview:**  
These nodes support the workflow operation and provide documentation for users.

**Nodes Involved:**  
- MCP Server Trigger (already covered)  
- Sticky Notes (several)  

**Sticky Notes Content Summary:**  
- Input explanation and routing rules  
- List of possible operations supported  
- Blocks for Post CRUD, Comment CRUD, and Subreddit read operations

---

### 3. Summary Table

| Node Name                  | Node Type                                   | Functional Role                        | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                              |
|----------------------------|---------------------------------------------|-------------------------------------|----------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| MCP Server Trigger          | @n8n/n8n-nodes-langchain.mcpTrigger         | Entry point receiving JSON requests | post_operation, comment_operation, subReddit_operation (AI tool outputs) | operation_switch              | ## Input and switch to right operation type. The MCP server receives a JSON that describes the type of operation you want. |
| When Executed by Another Workflow | executeWorkflowTrigger                  | Triggered by other workflows with JSON input | N/A                              | operation_switch              |                                                                                                        |
| operation_switch            | Switch (v3.2)                               | Routes operation type to post/comment/subreddit flows | MCP Server Trigger, When Executed by Another Workflow | post_switch, comment_switch, subreddit_switch |                                                                                                        |
| post_switch                | Switch (v3.2)                               | Routes specific post operation       | operation_switch                 | create_post, get_many_posts, delete_post, get_post_by_id, search_posts |                                                                                                        |
| create_post                | Reddit                                      | Creates a new Reddit post            | post_switch                     | N/A                           | ## Post CRUD                                                                                            |
| get_many_posts             | Reddit                                      | Retrieves multiple posts             | post_switch                     | map_post_get_many             | ## Post CRUD                                                                                            |
| delete_post                | Reddit                                      | Deletes a Reddit post                | post_switch                     | N/A                           | ## Post CRUD                                                                                            |
| get_post_by_id             | Reddit                                      | Retrieves a post by ID               | post_switch                     | N/A                           | ## Post CRUD                                                                                            |
| search_posts               | Reddit                                      | Searches posts by keyword            | post_switch                     | map_post_get_search           | ## Post CRUD                                                                                            |
| map_post_get_many          | Set                                         | Restructures post list output        | get_many_posts                  | posts_split_out               | ## Post CRUD                                                                                            |
| map_post_get_search        | Set                                         | Restructures search results          | search_posts                   | posts_split_out               | ## Post CRUD                                                                                            |
| posts_split_out            | SplitOut                                    | Splits posts array into individual items | map_post_get_many, map_post_get_search | N/A                       |                                                                                                        |
| comment_switch             | Switch (v3.2)                               | Routes specific comment operation    | operation_switch                 | create_comment, get_many_comments, delete_comment, reply_comment | ## Comment CRUD                                                                                         |
| create_comment            | Reddit                                      | Creates a comment on a post          | comment_switch                  | N/A                           | ## Comment CRUD                                                                                         |
| get_many_comments          | Reddit                                      | Retrieves multiple comments          | comment_switch                  | map_comment_get_many          | ## Comment CRUD                                                                                         |
| delete_comment             | Reddit                                      | Deletes a comment                    | comment_switch                  | N/A                           | ## Comment CRUD                                                                                         |
| reply_comment             | Reddit                                      | Replies to a comment                 | comment_switch                  | N/A                           | ## Comment CRUD                                                                                         |
| map_comment_get_many       | Set                                         | Restructures comments list output    | get_many_comments               | split_out_get_many_comments   | ## Comment CRUD                                                                                         |
| split_out_get_many_comments | SplitOut                                  | Splits comments array into individual items | map_comment_get_many            | N/A                           |                                                                                                        |
| subreddit_switch           | Switch (v3.2)                               | Routes specific subreddit operation  | operation_switch                 | get_subreddit_about, subreddit_get_many, get_subreddit_rules | ## Subreddit read's operation                                                                            |
| get_subreddit_about        | Reddit                                      | Retrieves subreddit info             | subreddit_switch                | N/A                           | ## Subreddit read's operation                                                                            |
| subreddit_get_many         | Reddit                                      | Retrieves multiple subreddits        | subreddit_switch                | N/A                           | ## Subreddit read's operation                                                                            |
| get_subreddit_rules        | Reddit                                      | Retrieves subreddit rules            | subreddit_switch                | N/A                           | ## Subreddit read's operation                                                                            |
| post_operation             | @n8n/n8n-nodes-langchain.toolWorkflow       | Sub-workflow encapsulating post ops | N/A (AI tool input)             | MCP Server Trigger (AI tool)  |                                                                                                        |
| comment_operation          | @n8n/n8n-nodes-langchain.toolWorkflow       | Sub-workflow encapsulating comment ops | N/A (AI tool input)             | MCP Server Trigger (AI tool)  |                                                                                                        |
| subReddit_operation        | @n8n/n8n-nodes-langchain.toolWorkflow       | Sub-workflow encapsulating subreddit ops | N/A (AI tool input)             | MCP Server Trigger (AI tool)  |                                                                                                        |
| Sticky Note                | Sticky Note                                 | Documentation on input and routing  | N/A                            | N/A                           | ## Input and switch to right operation type. The MCP server receives a JSON that describes the type of operation you want. |
| Sticky Note1               | Sticky Note                                 | Lists all possible operations       | N/A                            | N/A                           | ## Possible operations: post_create, post_delete, post_get_many, post_get_by_id, post_search, comment_create, comment_delete, comment_get_many, comment_reply, subreddit_get_about, subreddit_get_rules, subreddit_get_many |
| Sticky Note2               | Sticky Note                                 | Marks Post CRUD block                | N/A                            | N/A                           | ## Post CRUD                                                                                            |
| Sticky Note3               | Sticky Note                                 | Marks Comment CRUD block             | N/A                            | N/A                           | ## Comment CRUD                                                                                         |
| Sticky Note4               | Sticky Note                                 | Marks Subreddit read operations block | N/A                            | N/A                           | ## Subreddit read's operation                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node:**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Configure webhook path to receive JSON requests describing the Reddit operation.  
   - This node acts as the main entry point for the workflow.

2. **Add Execute Workflow Trigger Node:**  
   - Type: `executeWorkflowTrigger`  
   - Configure to accept JSON input from other workflows with an example payload including all operation parameters.  
   - Connect its output to `operation_switch`.

3. **Create `operation_switch` Node:**  
   - Type: `Switch` node (version 3.2)  
   - Configure three outputs:  
     - `post_flow` if `$json.operation` starts with `"post_"`  
     - `comment_flow` if `$json.operation` starts with `"comment_"`  
     - `subreddit_flow` if `$json.operation` starts with `"=subreddit_"`  
   - Connect outputs to respective `post_switch`, `comment_switch`, and `subreddit_switch` nodes.

4. **Post Operations Setup:**  
   - Create `post_switch` node (Switch v3.2) with exact matching rules for:  
     - `post_create`, `post_get_many`, `post_delete`, `post_get_by_id`, `post_search`  
   - Create Reddit nodes:  
     - `create_post` node configured with `subreddit`, `title`, and `text` from incoming JSON fields.  
     - `get_many_posts` node configured with `limit`, `filtersCategory`, and `subreddit`.  
     - `delete_post` node configured with `postId`.  
     - `get_post_by_id` node configured with `postId` and `subreddit`.  
     - `search_posts` node configured with `limit`, `keyword`, and `subreddit`.  
   - Connect `post_switch` outputs to the respective Reddit nodes.  
   - Create `map_post_get_many` and `map_post_get_search` nodes of type `Set` to map the output arrays from `get_many_posts` and `search_posts` respectively.  
   - Use JavaScript expressions within `Set` nodes to map relevant fields into an array under key `"posts"`.  
   - Connect outputs of mapping nodes to a `posts_split_out` node (SplitOut) to split array into individual items.

5. **Comment Operations Setup:**  
   - Create `comment_switch` node (Switch v3.2) with exact matching rules for:  
     - `comment_create`, `comment_get_many`, `comment_delete`, `comment_reply`  
   - Create Reddit nodes for comment operations:  
     - `create_comment` (postId, commentText)  
     - `get_many_comments` (limit, postId, subreddit)  
     - `delete_comment` (commentId)  
     - `reply_comment` (commentId, replyText)  
   - Connect `comment_switch` outputs to respective Reddit nodes.  
   - Create `map_comment_get_many` node of type `Set` to map comment arrays to `"comments"` key with selected fields.  
   - Connect to `split_out_get_many_comments` node (SplitOut) to split comments array into individual items.

6. **Subreddit Operations Setup:**  
   - Create `subreddit_switch` node (Switch v3.2) with exact matching rules for:  
     - `subreddit_get_about`, `subreddit_get_many`, `subreddit_get_rules`  
   - Create Reddit nodes configured accordingly:  
     - `get_subreddit_about` with `subreddit`  
     - `subreddit_get_many` with `limit`, `filtersKeyword`, `filtersTrendig`  
     - `get_subreddit_rules` with `subreddit` and content set to "rules"  
   - Connect outputs of `subreddit_switch` to these Reddit nodes.

7. **Sub-Workflow Tool Nodes:**  
   - Create three `@n8n/n8n-nodes-langchain.toolWorkflow` nodes named `post_operation`, `comment_operation`, `subReddit_operation`.  
   - Configure each with the workflow ID of this main workflow (MCP_reddit) for modular invocation.  
   - Define inputs and descriptions for each tool according to the allowed operations and parameters.

8. **Connect AI tool outputs to `MCP Server Trigger` node's AI tool input** to enable round-trip invocation.

9. **Add Sticky Notes for Documentation:**  
   - Add notes describing input format, possible operations, and marking the logical blocks (Post CRUD, Comment CRUD, Subreddit operations).

10. **Set Credentials:**  
    - Configure Reddit OAuth2 credentials for all Reddit nodes to enable authenticated API calls.

11. **Configure Execution Order:**  
    - Ensure execution order is set to "v1" (sequential) as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                               |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| The workflow supports Server-Sent Events (SSE) style JSON input for dynamic Reddit API interaction across posts, comments, and subreddits. | Workflow purpose                                                             |
| Supported Reddit operations are comprehensive: post_create, post_delete, post_get_many, post_get_by_id, post_search, comment_create, comment_delete, comment_get_many, comment_reply, subreddit_get_about, subreddit_get_rules, subreddit_get_many. | Sticky Note1 content                                                         |
| Modular design allows invoking the workflow as a tool from AI agents or other workflows via `toolWorkflow` nodes for posts, comments, and subreddits. | Sub-workflow nodes description                                               |
| Reddit OAuth2 credentials must be valid and have sufficient permissions to perform all operations. Token expiration or scope issues may cause errors. | Credentials note                                                             |
| API rate limiting, invalid IDs, missing parameters, or malformed JSON inputs are primary failure points to handle in integration.  | General error considerations                                                 |

---

**Disclaimer:** The above documentation is based exclusively on the provided n8n workflow JSON. It complies with all current content policies and does not contain illegal, offensive, or protected elements. All managed data are legal and public.