Connect AI Agents to Beanstream Payments API for Payment Processing & Management

https://n8nworkflows.xyz/workflows/connect-ai-agents-to-beanstream-payments-api-for-payment-processing---management-5538


# Connect AI Agents to Beanstream Payments API for Payment Processing & Management

---

## 1. Workflow Overview

This workflow, titled **"Connect AI Agents to Beanstream Payments API for Payment Processing & Management"**, serves as a Multi-Channel Platform (MCP) server interface that exposes 15 key Beanstream Payments API endpoints as callable tools for AI agents. It enables AI-driven automation systems to interact programmatically with Beanstream's payment processing, profile management, reporting, and tokenization services through a unified webhook endpoint.

The workflow is logically divided into the following blocks:

- **1.1 Setup & Documentation**: Contains sticky notes with setup instructions, workflow overview, and operational descriptions for easy user orientation.

- **1.2 MCP Server Trigger**: The entry point of the workflow accepting incoming AI agent requests via a webhook.

- **1.3 Payments Operations**: Implements 5 payment-related API calls (make payment, get payment, complete pre-authorization, return payment, void transaction).

- **1.4 Profiles Operations**: Implements 8 profile and card management API calls (create, get, delete, update profiles; add, get, update, delete cards).

- **1.5 Reporting Operation**: Supports a single reporting endpoint to query reports.

- **1.6 Tokenization Operation**: Provides credit card tokenization functionality.

Each block consists primarily of HTTP Request nodes configured to call specific Beanstream Payments API endpoints, using AI-driven expressions to populate parameters dynamically.

---

## 2. Block-by-Block Analysis

### 2.1 Setup & Documentation Block

**Overview:**  
This block provides user-facing documentation and setup guidance via sticky notes. It explains workflow usage, authentication, available API operations, and customization tips.

**Nodes Involved:**  
- Setup Instructions (sticky note)  
- Workflow Overview (sticky note)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Purpose: Provides step-by-step setup instructions including importing the workflow, activating it, copying the MCP webhook URL, and connecting AI agents.  
  - Content includes usage notes and customization suggestions.  
  - No input or output connections (informational node).  

- **Workflow Overview**  
  - Type: Sticky Note  
  - Purpose: Summarizes the workflow's purpose as a Beanstream Payments MCP-compatible interface, lists API base URL, explains the MCP trigger role, and enumerates 15 available operations grouped by functional area (Payments, Profiles, Reporting, Tokenization).  
  - No input or output connections.  

**Edge Cases / Failures:**  
- No runtime failures as these are static notes.

---

### 2.2 MCP Server Trigger Block

**Overview:**  
This block contains the MCP Trigger node that acts as the HTTP webhook endpoint where AI agents send requests to invoke any of the 15 Beanstream API operations.

**Nodes Involved:**  
- Beanstream Payments MCP Server (MCP Trigger)

**Node Details:**

- **Beanstream Payments MCP Server**  
  - Type: MCP Trigger (LangChain n8n node)  
  - Configuration:  
    - Webhook path: `/beanstream-payments-mcp`  
    - Acts as a server endpoint for AI agent requests.  
  - Input: External HTTP POST requests from AI agents (MCP protocol).  
  - Output: Passes requests to connected HTTP Request nodes corresponding to operations.  
  - Version: Requires n8n LangChain MCP node support (n8n v1.95+ recommended).  
  - Failure Modes:  
    - Webhook not reachable if workflow inactive or URL changed.  
    - Malformed MCP requests may cause errors downstream.  

---

### 2.3 Payments Operations Block

**Overview:**  
This block provides 5 HTTP Request nodes implementing the Payments API endpoints: make payment, get payment, complete pre-authorization, return payment, and void transaction.

**Nodes Involved:**  
- Sticky Note (Payments)  
- Make Payment  
- Get payment  
- Complete pre-auth  
- Return payment  
- Void Transaction

**Node Details:**

- **Sticky Note (Payments)**  
  - Type: Sticky Note  
  - Purpose: Label block as Payments.

- **Make Payment**  
  - Type: HTTP Request Tool  
  - Method: POST  
  - URL: `https://www.beanstream.com/api/v1/payments`  
  - Authentication: Generic HTTP Header Auth credential (expects API key or token in headers)  
  - Parameters: Body parameters auto-populated via AI expressions `$fromAI()` (dynamic)  
  - Output: API response with payment creation result.  
  - Errors: Auth failures if credentials invalid, API errors for invalid payment data.

- **Get payment**  
  - Type: HTTP Request Tool  
  - Method: GET (default)  
  - URL: `https://www.beanstream.com/api/v1/payments/{{ $fromAI('transId', 'The transaction id.', 'number') }}`  
  - Path Parameter: `transId` dynamically provided by AI agent request.  
  - Authentication: Generic HTTP Header Auth.  
  - Errors: Missing or invalid transId, 404 if payment not found.

- **Complete pre-auth**  
  - Type: HTTP Request Tool  
  - Method: POST  
  - URL: `https://www.beanstream.com/api/v1/payments/{{ $fromAI('transId', 'The transaction id.', 'number') }}/completions`  
  - Path Parameter: `transId` needed to complete a pre-authorization.  
  - Authentication: Generic HTTP Header Auth.  
  - Errors: Invalid transaction ID, or transaction not eligible for completion.

- **Return payment**  
  - Type: HTTP Request Tool  
  - Method: POST  
  - URL: `https://www.beanstream.com/api/v1/payments/{{ $fromAI('transId', 'The transaction id.', 'number') }}/returns`  
  - Path Parameter: `transId` for the payment to return/refund.  
  - Authentication: Generic HTTP Header Auth.  
  - Errors: Invalid or non-refundable transactions.

- **Void Transaction**  
  - Type: HTTP Request Tool  
  - Method: POST  
  - URL: `https://www.beanstream.com/api/v1/payments/{{ $fromAI('transId', 'The transaction id to void.', 'number') }}/void`  
  - Path Parameter: `transId` for transaction voiding.  
  - Authentication: Generic HTTP Header Auth.  
  - Errors: Transaction not voidable, or invalid ID.

**Common Edge Cases:**  
- Authentication errors if API keys missing or incorrect.  
- Path parameter missing or malformed (expression evaluation failure).  
- API endpoint downtime or timeout.  
- Request payload validation errors.

---

### 2.4 Profiles Operations Block

**Overview:**  
This block implements 8 profile and card management endpoints, enabling creation, retrieval, updating, and deletion of customer profiles and their payment cards.

**Nodes Involved:**  
- Sticky Note2 (Profiles)  
- Create Profile  
- Delete profile  
- Get profile  
- Update Profile  
- Get cards  
- Add card  
- Delete card  
- Update card

**Node Details:**

- **Sticky Note2 (Profiles)**  
  - Type: Sticky Note  
  - Purpose: Label block as Profiles.

- **Create Profile**  
  - Type: HTTP Request Tool  
  - Method: POST  
  - URL: `https://www.beanstream.com/api/v1/profiles`  
  - Authentication: Generic HTTP Header Auth  
  - Body parameters: Populated dynamically via AI expressions.  
  - Errors: Invalid profile data or missing required fields.

- **Delete profile**  
  - Type: HTTP Request Tool  
  - Method: DELETE  
  - URL: `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}`  
  - Path Parameter: `profileId` required to identify profile.  
  - Authentication: Generic HTTP Header Auth  
  - Errors: Profile not found or deletion not allowed.

- **Get profile**  
  - Type: HTTP Request Tool  
  - Method: GET (default)  
  - URL: `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}`  
  - Authentication: Generic HTTP Header Auth  
  - Errors: Missing or invalid profileId.

- **Update Profile**  
  - Type: HTTP Request Tool  
  - Method: PUT  
  - URL: `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}`  
  - Authentication: Generic HTTP Header Auth  
  - Body parameters: updated profile data from AI.  
  - Errors: Invalid data or profile not found.

- **Get cards**  
  - Type: HTTP Request Tool  
  - Method: GET  
  - URL: `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}/cards`  
  - Authentication: Generic HTTP Header Auth  
  - Errors: Profile not found or no cards available.

- **Add card**  
  - Type: HTTP Request Tool  
  - Method: POST  
  - URL: `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}/cards`  
  - Authentication: Generic HTTP Header Auth  
  - Body parameters: Card details dynamically populated.  
  - Errors: Invalid card data.

- **Delete card**  
  - Type: HTTP Request Tool  
  - Method: DELETE  
  - URL: `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}/cards/{{ $fromAI('cardId', 'The card id.', 'number') }}`  
  - Path Parameters: `profileId` and `cardId` required.  
  - Authentication: Generic HTTP Header Auth  
  - Errors: Card or profile not found.

- **Update card**  
  - Type: HTTP Request Tool  
  - Method: PUT  
  - URL: `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}/cards/{{ $fromAI('cardId', 'The card id.', 'number') }}`  
  - Authentication: Generic HTTP Header Auth  
  - Body parameters: Updated card data.  
  - Errors: Invalid or missing card/profile.

**Common Edge Cases:**  
- Expression failures if `profileId` or `cardId` parameters missing or invalid.  
- API errors due to invalid payloads or authorization failures.  
- Deletion of non-existent profiles/cards.

---

### 2.5 Reporting Operations Block

**Overview:**  
This block handles reporting via a single endpoint that accepts search queries for report data.

**Nodes Involved:**  
- Sticky Note3 (Reporting)  
- Search Query

**Node Details:**

- **Sticky Note3 (Reporting)**  
  - Type: Sticky Note  
  - Purpose: Label block as Reporting.

- **Search Query**  
  - Type: HTTP Request Tool  
  - Method: POST  
  - URL: `https://www.beanstream.com/api/v1/reports`  
  - Authentication: Generic HTTP Header Auth  
  - Body: Search query parameters dynamically populated by AI.  
  - Errors: Invalid query structure or no data found.

---

### 2.6 Tokenization Operations Block

**Overview:**  
This block includes a single node to tokenize credit card information securely.

**Nodes Involved:**  
- Sticky Note4 (Token Iz Ation)  
- Tokenize credit card

**Node Details:**

- **Sticky Note4 (Token Iz Ation)**  
  - Type: Sticky Note  
  - Purpose: Label block as Tokenization.

- **Tokenize credit card**  
  - Type: HTTP Request Tool  
  - Method: POST  
  - URL: `https://www.beanstream.com/api/v1/scripts/tokenization/tokens`  
  - Authentication: Generic HTTP Header Auth  
  - Body parameters: Credit card data dynamically passed by AI expressions.  
  - Errors: Invalid card data or tokenization failure.

---

## 3. Summary Table

| Node Name              | Node Type                 | Functional Role                     | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                             |
|------------------------|---------------------------|-----------------------------------|-------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------|
| Setup Instructions     | Sticky Note               | Setup and usage instructions      | None                          | None                            | ⚙️ Setup Instructions with import, authentication, activation, usage, and customization guidance.     |
| Workflow Overview      | Sticky Note               | Workflow summary and API overview | None                          | None                            | 🛠️ Beanstream Payments MCP Server overview and available operations list.                             |
| Beanstream Payments MCP Server | MCP Trigger              | Entry webhook for AI agent calls  | External HTTP requests        | HTTP Request nodes (all ops)    |                                                                                                       |
| Sticky Note (Payments) | Sticky Note               | Label payments block              | None                          | None                            | ## Payments                                                                                            |
| Make Payment           | HTTP Request Tool         | Create a new payment              | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Get payment            | HTTP Request Tool         | Retrieve payment details          | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Complete pre-auth      | HTTP Request Tool         | Complete pre-authorization        | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Return payment         | HTTP Request Tool         | Return/refund a payment           | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Void Transaction       | HTTP Request Tool         | Void a transaction                | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Sticky Note2 (Profiles)| Sticky Note               | Label profiles block              | None                          | None                            | ## Profiles                                                                                            |
| Create Profile         | HTTP Request Tool         | Create customer profile           | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Delete profile         | HTTP Request Tool         | Delete a customer profile         | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Get profile            | HTTP Request Tool         | Get customer profile details      | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Update Profile         | HTTP Request Tool         | Update customer profile           | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Get cards              | HTTP Request Tool         | List cards for a profile          | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Add card               | HTTP Request Tool         | Add card to profile               | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Delete card            | HTTP Request Tool         | Delete card from profile          | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Update card            | HTTP Request Tool         | Update card details               | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Sticky Note3 (Reporting)| Sticky Note              | Label reporting block             | None                          | None                            | ## Reporting                                                                                           |
| Search Query           | HTTP Request Tool         | Search reports                   | MCP Trigger                   | MCP Trigger                     |                                                                                                       |
| Sticky Note4 (Token Iz Ation) | Sticky Note           | Label tokenization block          | None                          | None                            | ## Token Iz Ation                                                                                      |
| Tokenize credit card   | HTTP Request Tool         | Tokenize credit card info         | MCP Trigger                   | MCP Trigger                     |                                                                                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Setup Instructions**  
   - Add sticky note node with color 4 (blue).  
   - Content: Setup steps for import, authentication (none required), activation, webhook URL retrieval, AI agent connection, usage notes, customization tips, and support links (Discord and n8n docs).  
   - Position: Around [-1380, -240].

2. **Create Sticky Note: Workflow Overview**  
   - Add sticky note node, color default, width 420, height 920.  
   - Content: Workflow description, API base URL, MCP trigger explanation, list of 15 operations.  
   - Position: [-1120, -240].

3. **Create MCP Trigger Node: Beanstream Payments MCP Server**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Set webhook path: `beanstream-payments-mcp`.  
   - Position: [-620, -240].  
   - No authentication required on this node.

4. **Create Sticky Note: Payments Label**  
   - Sticky note with content "## Payments".  
   - Color 2 (green), width 1100, height 200.  
   - Position: [-660, -100].

5. **Create HTTP Request Tool Nodes for Payments APIs:** For each node:  
   - Authentication: Use a "genericCredentialType" with HTTP Header Auth (configure credentials with API key/token).  
   - Use expressions to dynamically populate path parameters and bodies with `$fromAI()` expressions.

   - **Make Payment:**  
     - Method: POST  
     - URL: `https://www.beanstream.com/api/v1/payments`  
     - Tool Description: "Make Payment"  
     - Position: [-520, -60]

   - **Get payment:**  
     - Method: GET (default)  
     - URL: `https://www.beanstream.com/api/v1/payments/{{ $fromAI('transId', 'The transaction id.', 'number') }}`  
     - Tool Description: Includes parameter description for transId.  
     - Position: [-320, -60]

   - **Complete pre-auth:**  
     - Method: POST  
     - URL: `https://www.beanstream.com/api/v1/payments/{{ $fromAI('transId', 'The transaction id.', 'number') }}/completions`  
     - Tool Description: Include transId parameter.  
     - Position: [-120, -60]

   - **Return payment:**  
     - Method: POST  
     - URL: `https://www.beanstream.com/api/v1/payments/{{ $fromAI('transId', 'The transaction id.', 'number') }}/returns`  
     - Tool Description: Include transId parameter.  
     - Position: [80, -60]

   - **Void Transaction:**  
     - Method: POST  
     - URL: `https://www.beanstream.com/api/v1/payments/{{ $fromAI('transId', 'The transaction id to void.', 'number') }}/void`  
     - Tool Description: Include transId parameter.  
     - Position: [280, -60]

6. **Create Sticky Note: Profiles Label**  
   - Content: "## Profiles"  
   - Color 3 (orange), width 1700, height 200  
   - Position: [-660, 140]

7. **Create HTTP Request Tool Nodes for Profile APIs:** With similar authentication and dynamic parameters:

   - **Create Profile:**  
     - POST to `https://www.beanstream.com/api/v1/profiles`  
     - Position: [-520, 180]

   - **Delete profile:**  
     - DELETE to `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}`  
     - Position: [-320, 180]

   - **Get profile:**  
     - GET to `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}`  
     - Position: [-120, 180]

   - **Update Profile:**  
     - PUT to `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}`  
     - Position: [80, 180]

   - **Get cards:**  
     - GET to `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}/cards`  
     - Position: [280, 180]

   - **Add card:**  
     - POST to `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}/cards`  
     - Position: [480, 180]

   - **Delete card:**  
     - DELETE to `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}/cards/{{ $fromAI('cardId', 'The card id.', 'number') }}`  
     - Position: [680, 180]

   - **Update card:**  
     - PUT to `https://www.beanstream.com/api/v1/profiles/{{ $fromAI('profileId', 'The profile id. (aka CustomerCode)', 'string') }}/cards/{{ $fromAI('cardId', 'The card id.', 'number') }}`  
     - Position: [880, 180]

8. **Create Sticky Note: Reporting Label**  
   - Content: "## Reporting"  
   - Color 4 (blue), width 300, height 200  
   - Position: [-660, 380]

9. **Create HTTP Request Tool Node: Search Query**  
   - POST to `https://www.beanstream.com/api/v1/reports`  
   - Authentication: Generic HTTP Header Auth  
   - Position: [-520, 420]

10. **Create Sticky Note: Tokenization Label**  
    - Content: "## Token Iz Ation"  
    - Color 5 (purple), width 300, height 200  
    - Position: [-660, 620]

11. **Create HTTP Request Tool Node: Tokenize credit card**  
    - POST to `https://www.beanstream.com/api/v1/scripts/tokenization/tokens`  
    - Authentication: Generic HTTP Header Auth  
    - Position: [-520, 660]

12. **Set Up Credentials**  
    - Create and configure Generic HTTP Header Auth credentials in n8n with the required Beanstream API key or token.  
    - Assign this credential to all HTTP Request Tool nodes.

13. **Connect Nodes**  
    - All HTTP Request Tool nodes connect their `ai_tool` input to the MCP Trigger node.  
    - No sequential chaining between HTTP Request nodes (all independent endpoints).

14. **Activate Workflow**  
    - Enable the workflow to expose the MCP webhook endpoint.  
    - Share the webhook URL `/beanstream-payments-mcp` with AI agents for integration.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| For integration guidance or custom automation help, contact on Discord: https://discord.me/cfomodz                                                              | Support and community help                                                                                        |
| Official n8n documentation on MCP and LangChain nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/             | Reference for MCP Trigger node and LangChain integration                                                        |
| Workflow converts Beanstream Payments API v1 (https://www.beanstream.com/api/v1) into MCP-compatible AI tools                                                    | Beanstream Payments API documentation                                                                             |
| Parameters for API calls are dynamically populated using `$fromAI()` expressions to enable AI-driven requests                                                    | Expression usage detail                                                                                            |
| No authentication required for the webhook endpoint itself; API credentials managed via Generic HTTP Header Auth credentials in each HTTP Request node          | Security and authentication approach                                                                              |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---