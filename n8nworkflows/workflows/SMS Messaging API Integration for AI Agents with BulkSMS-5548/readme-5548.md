SMS Messaging API Integration for AI Agents with BulkSMS

https://n8nworkflows.xyz/workflows/sms-messaging-api-integration-for-ai-agents-with-bulksms-5548


# SMS Messaging API Integration for AI Agents with BulkSMS

---
### 1. Workflow Overview

This workflow implements a comprehensive JSON REST MCP (Multi-Channel Provider) server interface for the BulkSMS API, enabling AI agents to interact with BulkSMS services via 15 distinct API operations. It is designed to provide automated SMS messaging functionalities such as sending messages, managing blocked numbers, handling account credits, dealing with attachments, and managing webhooks, all through AI-driven parameter generation.

**Target Use Cases:**  
- AI agents requiring SMS messaging capabilities integrated with BulkSMS.  
- Automated workflows needing to send, query, and manage SMS messages and related resources.  
- Systems requiring webhook management and user profile access for BulkSMS accounts.

**Logical Blocks:**  
- **1.1 Setup & Overview**: Instructions and documentation for users to configure and understand the workflow.  
- **1.2 MCP Trigger**: The entry point serving as a webhook endpoint for AI agent requests.  
- **1.3 Blocked Numbers Management**: Listing and blocking phone numbers.  
- **1.4 Credits Management**: Transferring account credits.  
- **1.5 Message Operations**: Retrieving message history, sending messages (new and quick), viewing message details, and listing related messages.  
- **1.6 Profile Retrieval**: Fetching user profile information.  
- **1.7 Attachment Handling**: Generating signed URLs for uploading attachments.  
- **1.8 Webhook Management**: Listing, registering, removing, viewing, and updating webhooks.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Overview

- **Overview:** Provides detailed user instructions, workflow purpose, and operation descriptions to guide setup and use.  
- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  

- **Node Details:**  
  - *Setup Instructions*  
    - Type: Sticky Note  
    - Role: User guidance on importing, configuring authentication, activating workflow, and usage tips.  
    - Content includes setup steps, customization advice, and support links (Discord and n8n documentation).  
    - No inputs or outputs; purely informational.

  - *Workflow Overview*  
    - Type: Sticky Note  
    - Role: Summary of BulkSMS API features and workflow design.  
    - Explains API base URL, request requirements, parameter handling, and lists all 15 available API operations grouped by category.  
    - No inputs or outputs; informational.

- **Edge Cases:** None (informational nodes).

---

#### 1.2 MCP Trigger

- **Overview:** Acts as the main webhook entry point for AI agents, receiving requests and routing them to appropriate HTTP request nodes.  
- **Nodes Involved:** BulkSMS JSON REST MCP Server  

- **Node Details:**  
  - Type: MCP Trigger (n8n-nodes-langchain.mcpTrigger)  
  - Configuration: Webhook path set to `bulksms-json-rest-mcp`.  
  - Role: Listens for incoming AI agent HTTP requests, triggering downstream nodes based on invoked API operation.  
  - Inputs: External HTTP requests from AI agents.  
  - Outputs: Routed to all HTTP request nodes handling BulkSMS API calls.  
  - Version: Requires n8n version supporting MCP Trigger nodes.  
  - Edge Cases: Possible webhook connectivity issues, authentication failure if credentials are misconfigured, malformed requests from AI agents.

---

#### 1.3 Blocked Numbers Management

- **Overview:** Enables listing currently blocked phone numbers and adding new blocked numbers via BulkSMS API.  
- **Nodes Involved:**  
  - Sticky Note (Blocked Numbers)  
  - List Blocked Numbers (HTTP Request)  
  - Block Phone Number (HTTP Request)  

- **Node Details:**  
  - *Sticky Note*  
    - Type: Sticky Note  
    - Role: Label for the block; no functional impact.

  - *List Blocked Numbers*  
    - Type: HTTP Request Tool  
    - Role: Retrieves blocked numbers list with optional filters.  
    - URL: `https://api.bulksms.com/v1/blocked-numbers` (GET)  
    - Query Parameters:  
      - `min-id`: Minimum record ID to return (default 0).  
      - `limit`: Max number of records (default 10000, max 10000).  
    - Parameters populated dynamically using `$fromAI()` expressions.  
    - Authentication: Basic HTTP header credentials.  
    - Outputs: JSON list of blocked numbers.  
    - Edge Cases: Authentication failure, invalid query parameters, API rate limits.

  - *Block Phone Number*  
    - Type: HTTP Request Tool  
    - Role: Adds a new blocked phone number.  
    - URL: `https://api.bulksms.com/v1/blocked-numbers` (POST)  
    - Body: Expects details for the number to block (parameters auto-filled by AI).  
    - Authentication: Basic HTTP header credentials.  
    - Outputs: Confirmation or error response.  
    - Edge Cases: Duplicate blocking, invalid phone number format, auth failures.

---

#### 1.4 Credits Management

- **Overview:** Facilitates transferring account credits to another BulkSMS account.  
- **Nodes Involved:**  
  - Sticky Note (Credits)  
  - Transfer Account Credits (HTTP Request)  

- **Node Details:**  
  - *Sticky Note*  
    - Informational label.

  - *Transfer Account Credits*  
    - Type: HTTP Request Tool  
    - Role: Initiates credit transfer between accounts.  
    - URL: `https://api.bulksms.com/v1/credit/transfer` (POST)  
    - Body: Transfer details populated via `$fromAI()`.  
    - Authentication: Basic HTTP header credentials.  
    - Outputs: Transfer confirmation or error.  
    - Edge Cases: Insufficient credits, invalid recipient account, auth errors.

---

#### 1.5 Message Operations

- **Overview:** Comprehensive message management including retrieval, sending, and related message querying.  
- **Nodes Involved:**  
  - Sticky Note (Message)  
  - Retrieve Message History (HTTP Request)  
  - Send New Message (HTTP Request)  
  - Send Quick Message (HTTP Request)  
  - View Message Details (HTTP Request)  
  - List Related Messages (HTTP Request)  

- **Node Details:**  
  - *Sticky Note*  
    - Label for message-related nodes.

  - *Retrieve Message History*  
    - URL: `https://api.bulksms.com/v1/messages` (GET)  
    - Query Parameters:  
      - `limit`, `filter`, `sortOrder` (all populated from AI).  
    - Returns paginated message history.  
    - Edge Cases: Large datasets, pagination handling, invalid filters.

  - *Send New Message*  
    - URL: `https://api.bulksms.com/v1/messages` (POST)  
    - Query Parameters:  
      - `deduplication-id`, `auto-unicode`, `schedule-date`, `schedule-description`  
      - All parameters dynamically filled by AI.  
    - Role: Sends new SMS with advanced options like scheduled sending and encoding controls.  
    - Edge Cases: Invalid scheduling format, duplicate messages, encoding issues.

  - *Send Quick Message*  
    - URL: `https://api.bulksms.com/v1/messages/send` (GET or POST)  
    - Query Parameters:  
      - `to`, `body`, `deduplication-id`  
      - Filled by AI.  
    - Role: Quick send SMS with minimal parameters.  
    - Edge Cases: Missing required parameters, invalid phone number.

  - *View Message Details*  
    - URL: `https://api.bulksms.com/v1/messages/{{id}}` (GET)  
    - Path Parameter: `id` of message from AI.  
    - Role: Retrieves a specific message's details.  
    - Edge Cases: Invalid or nonexistent message ID.

  - *List Related Messages*  
    - URL: `https://api.bulksms.com/v1/messages/{{id}}/relatedReceivedMessages` (GET)  
    - Path Parameter: `id` of sent message.  
    - Role: Lists messages related to a sent message.  
    - Edge Cases: Missing or invalid message ID.

---

#### 1.6 Profile Retrieval

- **Overview:** Fetches the BulkSMS user profile data.  
- **Nodes Involved:**  
  - Sticky Note (Profile)  
  - Retrieve User Profile (HTTP Request)  

- **Node Details:**  
  - *Sticky Note*  
    - Label node.

  - *Retrieve User Profile*  
    - URL: `https://api.bulksms.com/v1/profile` (GET)  
    - Role: Returns user account profile information.  
    - Authentication: Basic HTTP header credentials.  
    - Edge Cases: Auth failures, API unavailability.

---

#### 1.7 Attachment Handling

- **Overview:** Generates signed URLs to upload attachments securely to BulkSMS.  
- **Nodes Involved:**  
  - Sticky Note (Attachments)  
  - Generate Attachment Upload URL (HTTP Request)  

- **Node Details:**  
  - *Sticky Note*  
    - Label node.

  - *Generate Attachment Upload URL*  
    - URL: `https://api.bulksms.com/v1/rmm/pre-sign-attachment` (POST)  
    - Role: Requests a pre-signed URL for attachment upload.  
    - Authentication: Basic HTTP header credentials.  
    - Edge Cases: Invalid attachment metadata, auth issues.

---

#### 1.8 Webhook Management

- **Overview:** Full lifecycle management of BulkSMS webhooks including listing, creating, deleting, viewing, and updating.  
- **Nodes Involved:**  
  - Sticky Note (Web Hooks)  
  - List All Webhooks (HTTP Request)  
  - Register New Webhook (HTTP Request)  
  - Remove Webhook (HTTP Request)  
  - Get Webhook Details (HTTP Request)  
  - Update Webhook Settings (HTTP Request)  

- **Node Details:**  
  - *Sticky Note*  
    - Label node.

  - *List All Webhooks*  
    - URL: `https://api.bulksms.com/v1/webhooks` (GET)  
    - Role: Retrieves list of all configured webhooks.  
    - Edge Cases: Large lists, auth failure.

  - *Register New Webhook*  
    - URL: `https://api.bulksms.com/v1/webhooks` (POST)  
    - Role: Creates a new webhook with parameters from AI.  
    - Edge Cases: Duplicate URLs, invalid configurations.

  - *Remove Webhook*  
    - URL: `https://api.bulksms.com/v1/webhooks/{{id}}` (DELETE)  
    - Path Parameter: `id` of webhook.  
    - Role: Deletes an existing webhook.  
    - Edge Cases: Nonexistent webhook ID, auth errors.

  - *Get Webhook Details*  
    - URL: `https://api.bulksms.com/v1/webhooks/{{id}}` (GET)  
    - Role: Gets details of a specific webhook.  
    - Edge Cases: Invalid ID.

  - *Update Webhook Settings*  
    - URL: `https://api.bulksms.com/v1/webhooks/{{id}}` (POST)  
    - Role: Updates parameters of an existing webhook.  
    - Edge Cases: Invalid parameters, auth failure.

---

### 3. Summary Table

| Node Name                   | Node Type                    | Functional Role                   | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                                 |
|-----------------------------|------------------------------|---------------------------------|----------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------|
| Setup Instructions           | Sticky Note                  | Setup and usage instructions    | None                             | None                            | ⚙️ Setup Instructions with detailed configuration and usage guidance including links to Discord and docs.   |
| Workflow Overview            | Sticky Note                  | Workflow description and overview | None                             | None                            | BulkSMS API overview and list of 15 available operations grouped by category.                               |
| BulkSMS JSON REST MCP Server | MCP Trigger                  | Main webhook entry point        | External HTTP requests           | All HTTP Request nodes          |                                                                                                             |
| Sticky Note                 | Sticky Note                  | Label for Blocked Numbers block | None                             | None                            | ## Blocked Numbers                                                                                           |
| List Blocked Numbers         | HTTP Request Tool            | List blocked phone numbers      | MCP Trigger                     | None                            |                                                                                                             |
| Block Phone Number           | HTTP Request Tool            | Block a phone number            | MCP Trigger                     | None                            |                                                                                                             |
| Sticky Note2                | Sticky Note                  | Label for Credits block         | None                             | None                            | ## Credits                                                                                                  |
| Transfer Account Credits     | HTTP Request Tool            | Transfer account credits        | MCP Trigger                     | None                            |                                                                                                             |
| Sticky Note3                | Sticky Note                  | Label for Message block         | None                             | None                            | ## Message                                                                                                  |
| Retrieve Message History     | HTTP Request Tool            | Retrieve message history        | MCP Trigger                     | None                            |                                                                                                             |
| Send New Message            | HTTP Request Tool            | Send new SMS message            | MCP Trigger                     | None                            |                                                                                                             |
| Send Quick Message          | HTTP Request Tool            | Send quick SMS message          | MCP Trigger                     | None                            |                                                                                                             |
| View Message Details        | HTTP Request Tool            | View details of a message       | MCP Trigger                     | None                            |                                                                                                             |
| List Related Messages       | HTTP Request Tool            | List related messages to a sent message | MCP Trigger                | None                            |                                                                                                             |
| Sticky Note4                | Sticky Note                  | Label for Profile block         | None                             | None                            | ## Profile                                                                                                  |
| Retrieve User Profile       | HTTP Request Tool            | Retrieve user profile           | MCP Trigger                     | None                            |                                                                                                             |
| Sticky Note5                | Sticky Note                  | Label for Attachments block     | None                             | None                            | ## Attachments                                                                                              |
| Generate Attachment Upload URL | HTTP Request Tool          | Generate signed URL for attachments | MCP Trigger                  | None                            |                                                                                                             |
| Sticky Note6                | Sticky Note                  | Label for Web Hooks block       | None                             | None                            | ## Web Hooks                                                                                                |
| List All Webhooks           | HTTP Request Tool            | List all webhooks               | MCP Trigger                     | None                            |                                                                                                             |
| Register New Webhook        | HTTP Request Tool            | Create a new webhook            | MCP Trigger                     | None                            |                                                                                                             |
| Remove Webhook             | HTTP Request Tool            | Delete a webhook                | MCP Trigger                     | None                            |                                                                                                             |
| Get Webhook Details         | HTTP Request Tool            | Get webhook details             | MCP Trigger                     | None                            |                                                                                                             |
| Update Webhook Settings     | HTTP Request Tool            | Update webhook configuration    | MCP Trigger                     | None                            |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note Node: "Setup Instructions"**  
   - Content: Detailed setup steps including import, authentication, activation, and usage notes.  
   - Position: Top-left area for visibility.

2. **Create Sticky Note Node: "Workflow Overview"**  
   - Content: BulkSMS API overview, operation list (15 endpoints), and workflow design explanation.  
   - Position: Adjacent to Setup Instructions.

3. **Add MCP Trigger Node: "BulkSMS JSON REST MCP Server"**  
   - Type: MCP Trigger (Langchain MCP Trigger)  
   - Path: `bulksms-json-rest-mcp`  
   - Configure webhook authentication if needed.  
   - Position: Central, connected to all main HTTP request nodes.

4. **Blocked Numbers Block:**  
   - Add Sticky Note: "Blocked Numbers" label.  
   - Add HTTP Request Node: "List Blocked Numbers"  
     - Method: GET  
     - URL: `https://api.bulksms.com/v1/blocked-numbers`  
     - Query Parameters: `min-id`, `limit` with values using `$fromAI()` expressions.  
     - Authentication: Setup Basic HTTP Header credentials.  
   - Add HTTP Request Node: "Block Phone Number"  
     - Method: POST  
     - URL: `https://api.bulksms.com/v1/blocked-numbers`  
     - Body: Parameters for new block from AI input.  
     - Authentication: Basic HTTP Header credentials.

5. **Credits Block:**  
   - Add Sticky Note: "Credits" label.  
   - Add HTTP Request Node: "Transfer Account Credits"  
     - Method: POST  
     - URL: `https://api.bulksms.com/v1/credit/transfer`  
     - Body parameters from AI.  
     - Authentication: Basic HTTP Header credentials.

6. **Message Operations Block:**  
   - Add Sticky Note: "Message" label.  
   - Add HTTP Request Node: "Retrieve Message History"  
     - Method: GET  
     - URL: `https://api.bulksms.com/v1/messages`  
     - Query Parameters: `limit`, `filter`, `sortOrder` from AI.  
     - Authentication: Basic HTTP Header credentials.  
   - Add HTTP Request Node: "Send New Message"  
     - Method: POST  
     - URL: `https://api.bulksms.com/v1/messages`  
     - Query Parameters: `deduplication-id`, `auto-unicode`, `schedule-date`, `schedule-description` all from AI.  
     - Authentication: Basic HTTP Header credentials.  
   - Add HTTP Request Node: "Send Quick Message"  
     - Method: GET or POST  
     - URL: `https://api.bulksms.com/v1/messages/send`  
     - Query Parameters: `to`, `body`, `deduplication-id` from AI.  
     - Authentication: Basic HTTP Header credentials.  
   - Add HTTP Request Node: "View Message Details"  
     - Method: GET  
     - URL: `https://api.bulksms.com/v1/messages/{{ $fromAI('id') }}`  
     - Authentication: Basic HTTP Header credentials.  
   - Add HTTP Request Node: "List Related Messages"  
     - Method: GET  
     - URL: `https://api.bulksms.com/v1/messages/{{ $fromAI('id') }}/relatedReceivedMessages`  
     - Authentication: Basic HTTP Header credentials.

7. **Profile Block:**  
   - Add Sticky Note: "Profile" label.  
   - Add HTTP Request Node: "Retrieve User Profile"  
     - Method: GET  
     - URL: `https://api.bulksms.com/v1/profile`  
     - Authentication: Basic HTTP Header credentials.

8. **Attachments Block:**  
   - Add Sticky Note: "Attachments" label.  
   - Add HTTP Request Node: "Generate Attachment Upload URL"  
     - Method: POST  
     - URL: `https://api.bulksms.com/v1/rmm/pre-sign-attachment`  
     - Authentication: Basic HTTP Header credentials.

9. **Web Hooks Block:**  
   - Add Sticky Note: "Web Hooks" label.  
   - Add HTTP Request Node: "List All Webhooks"  
     - Method: GET  
     - URL: `https://api.bulksms.com/v1/webhooks`  
     - Authentication: Basic HTTP Header credentials.  
   - Add HTTP Request Node: "Register New Webhook"  
     - Method: POST  
     - URL: `https://api.bulksms.com/v1/webhooks`  
     - Authentication: Basic HTTP Header credentials.  
   - Add HTTP Request Node: "Remove Webhook"  
     - Method: DELETE  
     - URL: `https://api.bulksms.com/v1/webhooks/{{ $fromAI('id') }}`  
     - Authentication: Basic HTTP Header credentials.  
   - Add HTTP Request Node: "Get Webhook Details"  
     - Method: GET  
     - URL: `https://api.bulksms.com/v1/webhooks/{{ $fromAI('id') }}`  
     - Authentication: Basic HTTP Header credentials.  
   - Add HTTP Request Node: "Update Webhook Settings"  
     - Method: POST  
     - URL: `https://api.bulksms.com/v1/webhooks/{{ $fromAI('id') }}`  
     - Authentication: Basic HTTP Header credentials.

10. **Connect Each HTTP Request Node's AI Tool Input**  
    - For each HTTP Request node, connect the incoming AI agent request from the MCP Trigger node as an `ai_tool` input.  
    - This allows the MCP Trigger to route requests to the correct API operation node based on AI command.

11. **Configure Authentication Credentials**  
    - Create and assign Basic HTTP Header authentication credentials with BulkSMS API key or token.  
    - Assign these credentials to all HTTP Request nodes.

12. **Activate Workflow**  
    - Save and activate the workflow to listen for AI agent requests.  
    - Copy the MCP Trigger webhook URL for use in AI agent configuration.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                                               |
|--------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Discord channel for integration guidance and custom automations.                                                   | https://discord.me/cfomodz                                                                                                    |
| Official n8n documentation for MCP node and BulkSMS integration details.                                           | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                                  |
| BulkSMS API base URL: `https://api.bulksms.com/v1`                                                                 | All HTTP requests target this API base URL.                                                                                   |
| Parameters are dynamically populated using `$fromAI()` expressions, enabling AI-driven input handling.             | Workflow designed for AI agents to populate parameters automatically.                                                         |
| Responses maintain original API JSON structure, allowing clients to handle data natively.                          | No data transformation is performed by default, but customization is possible.                                                |
| Users should implement custom error handling, logging, or data transformations as needed for production scenarios. | The workflow provides a foundation and can be extended with additional nodes for robustness.                                  |

---

**Disclaimer:** This documentation is based exclusively on an automated workflow created using n8n, respecting all applicable content policies. The workflow contains no illegal or offensive content and processes only legal and public data.