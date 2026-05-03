🛠️ Storyblok Tool MCP Server 💪 all 7 operations

https://n8nworkflows.xyz/workflows/----storyblok-tool-mcp-server----all-7-operations-5361


# 🛠️ Storyblok Tool MCP Server 💪 all 7 operations

### 1. Workflow Overview

This workflow, titled **"Storyblok Tool MCP Server"**, is designed to serve as a multi-command processing (MCP) server for Storyblok CMS operations. Its purpose is to expose a single entry point that can handle multiple Storyblok-related operations — specifically, the 7 core CRUD and publishing actions on stories within Storyblok.  

The workflow logically divides into the following blocks:

- **1.1 MCP Trigger Reception:**  
  The workflow’s entry point listens for incoming MCP commands, serving as a webhook endpoint that receives requests specifying which Storyblok operation should be performed.

- **1.2 Storyblok Operations Execution:**  
  Based on the MCP command received, the workflow routes the execution to one of the six Storyblok Tool nodes, each representing a distinct Storyblok operation:  
  - Get a story  
  - Get many stories  
  - Delete a story  
  - Publish a story  
  - Unpublish a story  

The workflow integrates these operations through direct node-to-node connections from the MCP trigger node to each Storyblok Tool node, allowing dynamic execution depending on the requested operation.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Reception

- **Overview:**  
  This block initiates the workflow by receiving MCP commands via a webhook. It acts as a centralized trigger that interprets incoming requests and routes them to the appropriate Storyblok operation node.

- **Nodes Involved:**  
  - Storyblok Tool MCP Server (MCP Trigger)

- **Node Details:**

  - **Node Name:** Storyblok Tool MCP Server  
    - **Type:** MCP Trigger (from n8n-nodes-langchain package)  
    - **Technical Role:** Webhook trigger designed to receive multi-command processing requests, parse them, and provide an AI tool interface for routing commands.  
    - **Configuration:**  
      - No additional parameters set; uses default webhook setup with a unique webhook ID.  
      - Exposes a webhook endpoint for external MCP command calls.  
    - **Key Expressions/Variables:** None explicitly set; acts as entry point.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Connects to all Storyblok operation nodes ("Get a story", "Get many stories", "Delete a story", "Publish a story", "Unpublish a story").  
    - **Version Requirements:** Requires n8n environment capable of supporting MCP Trigger nodes (likely n8n version 1.95+ due to Langchain package usage).  
    - **Potential Failure Modes:**  
      - Webhook authentication or permission errors (if enabled externally).  
      - Network connectivity issues causing webhook unavailability.  
      - Invalid or malformed MCP command payloads leading to routing failure.  
    - **Sub-workflow:** None.

---

#### 2.2 Storyblok Operations Execution

- **Overview:**  
  This block contains all the Storyblok Tool nodes that perform specific operations on Storyblok stories. Each node corresponds to one of the supported operations and executes that operation based on the input routed from the MCP Trigger.

- **Nodes Involved:**  
  - Get a story  
  - Get many stories  
  - Delete a story  
  - Publish a story  
  - Unpublish a story

- **Node Details:**

  - **Node Name:** Get a story  
    - **Type:** Storyblok Tool node  
    - **Technical Role:** Retrieves a single story from Storyblok CMS based on provided parameters (e.g., story ID or slug).  
    - **Configuration:** No parameters preset; expects dynamic input from MCP trigger.  
    - **Input Connections:** Receives input from "Storyblok Tool MCP Server" node.  
    - **Output Connections:** None (end node).  
    - **Version Requirements:** Requires Storyblok Tool node installed and configured with valid Storyblok API credentials.  
    - **Potential Failure Modes:**  
      - API authentication failure due to invalid credentials.  
      - Story not found errors if story ID/slug does not exist.  
      - Rate limiting or API timeout errors.  

  - **Node Name:** Get many stories  
    - **Type:** Storyblok Tool node  
    - **Technical Role:** Fetches multiple stories from Storyblok CMS, e.g., for listing or batch processing.  
    - **Configuration:** No preset parameters; expects request data from MCP trigger.  
    - **Input Connections:** From "Storyblok Tool MCP Server".  
    - **Output Connections:** None.  
    - **Version Requirements:** Same as above.  
    - **Potential Failure Modes:** Similar to "Get a story" but may include pagination issues or large data handling.  

  - **Node Name:** Delete a story  
    - **Type:** Storyblok Tool node  
    - **Technical Role:** Deletes a specified story in Storyblok CMS.  
    - **Configuration:** Expects story identifier from MCP trigger input.  
    - **Input Connections:** From MCP Trigger.  
    - **Output Connections:** None.  
    - **Version Requirements:** Same as above.  
    - **Potential Failure Modes:**  
      - Permission denied if API token lacks delete rights.  
      - Story not found errors.  
      - API rate limiting or timeout.  

  - **Node Name:** Publish a story  
    - **Type:** Storyblok Tool node  
    - **Technical Role:** Publishes a story, making it live in Storyblok.  
    - **Configuration:** Parameters expected from MCP trigger.  
    - **Input Connections:** From MCP trigger.  
    - **Output Connections:** None.  
    - **Version Requirements:** Same as above.  
    - **Potential Failure Modes:**  
      - API permission errors.  
      - Story version conflicts.  
      - Network/API errors.  

  - **Node Name:** Unpublish a story  
    - **Type:** Storyblok Tool node  
    - **Technical Role:** Unpublishes a live story in Storyblok, reverting it to draft or unpublished state.  
    - **Configuration:** Input from MCP trigger.  
    - **Input Connections:** From MCP trigger.  
    - **Output Connections:** None.  
    - **Version Requirements:** Same as above.  
    - **Potential Failure Modes:** Similar to publish operation.

---

### 3. Summary Table

| Node Name                 | Node Type                    | Functional Role                                | Input Node(s)               | Output Node(s)              | Sticky Note                      |
|---------------------------|------------------------------|-----------------------------------------------|-----------------------------|-----------------------------|---------------------------------|
| Workflow Overview 0       | Sticky Note                  | Informational / overview placeholder          |                             |                             |                                 |
| Storyblok Tool MCP Server | MCP Trigger                  | Entry point receiving MCP commands             |                             | Get a story, Get many stories, Delete a story, Publish a story, Unpublish a story |                                 |
| Get a story               | Storyblok Tool               | Retrieves a single story                        | Storyblok Tool MCP Server    |                             |                                 |
| Get many stories          | Storyblok Tool               | Retrieves multiple stories                      | Storyblok Tool MCP Server    |                             |                                 |
| Delete a story            | Storyblok Tool               | Deletes a specified story                       | Storyblok Tool MCP Server    |                             |                                 |
| Publish a story           | Storyblok Tool               | Publishes a story                              | Storyblok Tool MCP Server    |                             |                                 |
| Unpublish a story         | Storyblok Tool               | Unpublishes a story                            | Storyblok Tool MCP Server    |                             |                                 |
| Sticky Note 1             | Sticky Note                  | Placeholder or note (empty content)             |                             |                             |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node:**
   - Add a new node of type **MCP Trigger** (from the Langchain n8n nodes collection).  
   - Leave parameters at default; this node acts as the webhook entry point.  
   - Note the webhook URL it generates; this will be used to send MCP requests.

2. **Add Storyblok Tool Nodes for Operations:**
   For each Storyblok operation below, add a **Storyblok Tool** node configured as follows:

   - **Get a story:**  
     - Node type: Storyblok Tool  
     - Operation: Get a story (ensure this is selected or set internally)  
     - Leave other parameters empty/default as they will be set dynamically from MCP trigger input  
     - Connect input from MCP Trigger node's output.

   - **Get many stories:**  
     - Same as above, set to "Get many stories" operation.

   - **Delete a story:**  
     - Set operation to "Delete a story".

   - **Publish a story:**  
     - Set operation to "Publish a story".

   - **Unpublish a story:**  
     - Set operation to "Unpublish a story".

3. **Connect MCP Trigger Node Output to Each Storyblok Tool Node's Input:**
   - From the MCP Trigger node, create output connections to each Storyblok Tool node.

4. **Configure Credentials for Storyblok Tool Nodes:**
   - For each Storyblok Tool node, set up credentials using a valid Storyblok API token with appropriate permissions (read, write, delete, publish as needed).  
   - The credentials should be stored in n8n and assigned to each Storyblok Tool node.

5. **Optional Sticky Notes:**
   - Add any sticky notes for documentation or overview purposes.

6. **Save and Activate Workflow:**
   - Save the workflow and activate it.  
   - Test by sending MCP requests to the MCP Trigger webhook URL specifying the desired Storyblok operation and parameters.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                   |
|----------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow requires the n8n Langchain package for the MCP Trigger node.                               | n8n Langchain integration documentation           |
| Storyblok Tool nodes require valid API credentials with necessary scopes (read, write, delete, publish). | Storyblok API docs: https://www.storyblok.com/docs/api/content-delivery |
| MCP (Multi-Command Processing) Trigger is suitable for handling multiple commands dynamically in one workflow. | n8n MCP Trigger documentation                      |
| Ensure the webhook endpoint exposed by the MCP Trigger is secured if exposed publicly (e.g., via API keys or IP whitelisting). | Security best practices for n8n webhooks          |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. It strictly adheres to content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.