🛠️ Mautic Tool MCP Server 💪 all 20 operations

https://n8nworkflows.xyz/workflows/----mautic-tool-mcp-server----all-20-operations-5184


# 🛠️ Mautic Tool MCP Server 💪 all 20 operations

### 1. Workflow Overview

This workflow is designed to provide a comprehensive integration with Mautic, a marketing automation platform, by exposing 20 different operations accessible via a single webhook trigger. The workflow targets users who want to automate and manage contacts, companies, campaigns, segments, and email sending within Mautic through n8n.  

The logic is organized into clearly defined functional blocks, each grouping related Mautic operations by entity type or feature:

- **1.1 Trigger Input Reception:** Receives incoming requests to invoke Mautic operations.
- **1.2 Campaign Contact Management:** Handles adding and removing contacts from campaigns.
- **1.3 Company Management:** Supports creation, retrieval, updating, deletion, and contact management for companies.
- **1.4 Contact Management:** Covers creation, retrieval, updating, deletion, point editing, and managing Do Not Contact lists.
- **1.5 Segment Management:** Adds or removes contacts from segments and sends emails to segments.
- **1.6 Email Sending:** Sends emails to single contacts.

All Mautic operations are triggered conditionally based on the incoming request routed from the centralized webhook node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception Block

- **Overview:**  
  This block serves as the entry point of the workflow. It receives incoming HTTP requests via an MCP webhook trigger and routes them to the appropriate Mautic operation node.

- **Nodes Involved:**  
  - Mautic Tool MCP Server (Webhook trigger)

- **Node Details:**

  - **Mautic Tool MCP Server:**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger` (Webhook trigger specialized for Mautic operations)  
    - Configuration: No custom parameters specified; binds to webhookId `f2c1eef6-0614-45e9-addc-138ccf95bf46`.  
    - Expressions/Variables: N/A  
    - Inputs: None (start node)  
    - Outputs: Connects to all Mautic Tool nodes via the "ai_tool" connection.  
    - Version: 1  
    - Edge Cases: Webhook invocation failure, invalid or malformed HTTP requests, security/authentication issues if not externally controlled.  
    - Notes: Acts as a centralized trigger for all 20 Mautic operations.

#### 2.2 Campaign Contact Management Block

- **Overview:**  
  Manages adding and removing contacts from campaigns.

- **Nodes Involved:**  
  - Add a campaign contact  
  - Remove a campaign contact

- **Node Details:**

  - **Add a campaign contact:**  
    - Type: `n8n-nodes-base.mauticTool`  
    - Role: Adds a contact to a specified campaign in Mautic.  
    - Configuration: Default, expects campaign and contact identifiers.  
    - Inputs: From MCP Server via ai_tool connection  
    - Outputs: None (end nodes in this context)  
    - Edge Cases: Invalid campaign/contact ID, API connectivity issues, permission errors.

  - **Remove a campaign contact:**  
    - Type: `n8n-nodes-base.mauticTool`  
    - Role: Removes a contact from a campaign.  
    - Configuration: Similar to Add node, expects campaign and contact identifiers.  
    - Inputs/Outputs: Same as above  
    - Edge Cases: Same as above.

#### 2.3 Company Management Block

- **Overview:**  
  Covers CRUD operations for companies and manages contacts within companies.

- **Nodes Involved:**  
  - Create a company  
  - Delete a company  
  - Get a company  
  - Get many companies  
  - Update a company  
  - Add a company contact  
  - Remove a company contact

- **Node Details:**

  - **Create a company:**  
    - Type: `mauticTool`  
    - Role: Creates a new company entity in Mautic.  
    - Config: Requires company details like name, address, etc.  
    - Inputs: MCP Server  
    - Outputs: None  
    - Edge Cases: Missing required fields, API limits.

  - **Delete a company:**  
    - Role: Deletes a company by ID.  
    - Edge Cases: Non-existent company ID, dependencies blocking deletion.

  - **Get a company:**  
    - Role: Fetches details of a single company by ID.  
    - Edge Cases: Wrong ID, API timeout.

  - **Get many companies:**  
    - Role: Fetches multiple companies with optional filters.  
    - Edge Cases: Large dataset handling, pagination.

  - **Update a company:**  
    - Role: Updates company details.  
    - Edge Cases: Partial updates, invalid fields.

  - **Add a company contact:**  
    - Role: Links a contact to a company.  
    - Edge Cases: Non-existent contact or company.

  - **Remove a company contact:**  
    - Role: Unlinks a contact from a company.  
    - Edge Cases: Contact not linked to company.

#### 2.4 Contact Management Block

- **Overview:**  
  Manages contacts with operations to create, delete, update, retrieve, modify points, and manage Do Not Contact status.

- **Nodes Involved:**  
  - Create a contact  
  - Delete a contact  
  - Edit a contact's points  
  - Add/remove contacts from/to the do not contact list  
  - Get a contact  
  - Get many contacts  
  - Send email to a contact  
  - Update a contact

- **Node Details:**

  - **Create a contact:**  
    - Role: Adds a new contact to Mautic.  
    - Config: Requires standard contact fields (email, name, etc.)  
    - Edge Cases: Duplicate email, invalid data.

  - **Delete a contact:**  
    - Role: Removes a contact by ID.  
    - Edge Cases: Non-existent ID.

  - **Edit a contact's points:**  
    - Role: Adjusts loyalty or scoring points for a contact.  
    - Edge Cases: Invalid point values, concurrency issues.

  - **Add/remove contacts from/to the do not contact list:**  
    - Role: Toggles Do Not Contact status.  
    - Edge Cases: Status conflicts.

  - **Get a contact:**  
    - Role: Retrieves single contact details.  
    - Edge Cases: Wrong ID.

  - **Get many contacts:**  
    - Role: Retrieves multiple contacts with filters.  
    - Edge Cases: Pagination, large data sets.

  - **Send email to a contact:**  
    - Role: Sends a designated email to one contact.  
    - Edge Cases: Missing email template, SMTP issues.

  - **Update a contact:**  
    - Role: Updates contact information.  
    - Edge Cases: Partial updates, invalid fields.

#### 2.5 Segment Management Block

- **Overview:**  
  Manages contact membership in segments and sending emails to entire segments.

- **Nodes Involved:**  
  - Add a contact to a segment  
  - Remove a contact from a segment  
  - Send an email to a segment

- **Node Details:**

  - **Add a contact to a segment:**  
    - Role: Includes a contact in a marketing segment.  
    - Edge Cases: Contact or segment not found.

  - **Remove a contact from a segment:**  
    - Role: Excludes a contact from a segment.  
    - Edge Cases: Contact not in segment.

  - **Send an email to a segment:**  
    - Role: Sends an email campaign to all contacts in a segment.  
    - Edge Cases: Large segment size, email throttling, template errors.

#### 2.6 Email Sending Block

- **Overview:**  
  Facilitates sending emails directly to individual contacts.

- **Nodes Involved:**  
  - Send email to a contact (also part of Contact Management Block)

- **Node Details:**  
  Covered already above under Contact Management.

---

### 3. Summary Table

| Node Name                                    | Node Type                              | Functional Role                                 | Input Node(s)          | Output Node(s)         | Sticky Note                      |
|----------------------------------------------|--------------------------------------|------------------------------------------------|------------------------|------------------------|---------------------------------|
| Workflow Overview 0                           | Sticky Note                          | Workflow overview placeholder                   | -                      | -                      |                                 |
| Mautic Tool MCP Server                        | MCP Trigger (webhook)                | Central webhook receiver triggering all ops    | -                      | All Mautic Tool nodes   |                                 |
| Add a campaign contact                        | Mautic Tool                         | Add contact to campaign                          | Mautic Tool MCP Server  | -                      |                                 |
| Remove a campaign contact                     | Mautic Tool                         | Remove contact from campaign                     | Mautic Tool MCP Server  | -                      |                                 |
| Sticky Note 1                                 | Sticky Note                          | Placeholder near campaign contact nodes         | -                      | -                      |                                 |
| Create a company                              | Mautic Tool                         | Create a new company                             | Mautic Tool MCP Server  | -                      |                                 |
| Delete a company                              | Mautic Tool                         | Delete a company                                | Mautic Tool MCP Server  | -                      |                                 |
| Get a company                                 | Mautic Tool                         | Retrieve company details                        | Mautic Tool MCP Server  | -                      |                                 |
| Get many companies                            | Mautic Tool                         | Retrieve multiple companies                     | Mautic Tool MCP Server  | -                      |                                 |
| Update a company                              | Mautic Tool                         | Update company details                          | Mautic Tool MCP Server  | -                      |                                 |
| Sticky Note 2                                 | Sticky Note                          | Placeholder near company nodes                   | -                      | -                      |                                 |
| Add a company contact                         | Mautic Tool                         | Link contact to a company                        | Mautic Tool MCP Server  | -                      |                                 |
| Remove a company contact                      | Mautic Tool                         | Unlink contact from a company                    | Mautic Tool MCP Server  | -                      |                                 |
| Sticky Note 3                                 | Sticky Note                          | Placeholder near company contact nodes           | -                      | -                      |                                 |
| Create a contact                              | Mautic Tool                         | Create a contact                                | Mautic Tool MCP Server  | -                      |                                 |
| Delete a contact                              | Mautic Tool                         | Delete a contact                                | Mautic Tool MCP Server  | -                      |                                 |
| Edit a contact's points                       | Mautic Tool                         | Modify loyalty/scoring points                    | Mautic Tool MCP Server  | -                      |                                 |
| Add/remove contacts from/to the do not contact list | Mautic Tool                         | Manage Do Not Contact status                      | Mautic Tool MCP Server  | -                      |                                 |
| Get a contact                                 | Mautic Tool                         | Retrieve contact details                        | Mautic Tool MCP Server  | -                      |                                 |
| Get many contacts                             | Mautic Tool                         | Retrieve multiple contacts                      | Mautic Tool MCP Server  | -                      |                                 |
| Send email to a contact                       | Mautic Tool                         | Send email to a single contact                   | Mautic Tool MCP Server  | -                      |                                 |
| Update a contact                              | Mautic Tool                         | Update contact details                          | Mautic Tool MCP Server  | -                      |                                 |
| Sticky Note 4                                 | Sticky Note                          | Placeholder near contact nodes                    | -                      | -                      |                                 |
| Add a contact to a segment                    | Mautic Tool                         | Add contact to segment                           | Mautic Tool MCP Server  | -                      |                                 |
| Remove a contact from a segment               | Mautic Tool                         | Remove contact from segment                      | Mautic Tool MCP Server  | -                      |                                 |
| Sticky Note 5                                 | Sticky Note                          | Placeholder near segment nodes                    | -                      | -                      |                                 |
| Send an email to a segment                    | Mautic Tool                         | Send email campaign to segment                   | Mautic Tool MCP Server  | -                      |                                 |
| Sticky Note 6                                 | Sticky Note                          | Placeholder near segment email node               | -                      | -                      |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Webhook Trigger Node:**  
   - Add node: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: `Mautic Tool MCP Server`  
   - Configure webhook ID (auto-generated or custom)  
   - No additional parameters needed.

2. **Add Campaign Contact Nodes:**  
   - Add `mauticTool` node, name it `Add a campaign contact`  
   - Configure operation to add a contact to campaign (set required parameters: campaign ID, contact ID)  
   - Connect input from `Mautic Tool MCP Server` node.  
   - Add another `mauticTool` node named `Remove a campaign contact` with the corresponding removal operation. Connect similarly.

3. **Add Company Management Nodes:**  
   - Add nodes for: `Create a company`, `Delete a company`, `Get a company`, `Get many companies`, `Update a company`, `Add a company contact`, `Remove a company contact`.  
   - For each, configure the appropriate Mautic operation and required parameters.  
   - Connect each node’s input to `Mautic Tool MCP Server`.

4. **Add Contact Management Nodes:**  
   - Add nodes for: `Create a contact`, `Delete a contact`, `Edit a contact's points`, `Add/remove contacts from/to the do not contact list`, `Get a contact`, `Get many contacts`, `Send email to a contact`, `Update a contact`.  
   - Configure each node’s operation and parameters as per Mautic API specs.  
   - Connect inputs from the MCP Server node.

5. **Add Segment Management Nodes:**  
   - Add `Add a contact to a segment`, `Remove a contact from a segment`, and `Send an email to a segment` nodes.  
   - Configure each node with segment IDs, contact IDs, and email templates as needed.  
   - Connect inputs from the MCP Server.

6. **Add Sticky Note Nodes:**  
   - Insert Sticky Note nodes as visual placeholders near related groups of nodes if desired for clarity.

7. **Credential Setup:**  
   - For all `mauticTool` nodes, configure Mautic API credentials (OAuth2 or API key) to enable authenticated communication.  
   - Ensure the MCP Trigger node is accessible externally for webhook calls.

8. **Test Workflow:**  
   - Trigger the webhook with appropriate payloads specifying the operation to invoke.  
   - Verify responses and correct operation routing.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                   |
|------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow title: "🛠️ Mautic Tool MCP Server 💪 all 20 operations"                              | Workflow identification                           |
| All Mautic operations are centralized via a single MCP webhook node for unified external access | Design decision for API gateway pattern           |
| Mautic Tool nodes require properly configured API credentials in n8n to function correctly     | Credential management in n8n                       |

---

**Disclaimer:**  
The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly follows applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.