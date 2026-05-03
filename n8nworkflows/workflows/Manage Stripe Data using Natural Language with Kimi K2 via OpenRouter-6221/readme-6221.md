Manage Stripe Data using Natural Language with Kimi K2 via OpenRouter

https://n8nworkflows.xyz/workflows/manage-stripe-data-using-natural-language-with-kimi-k2-via-openrouter-6221


# Manage Stripe Data using Natural Language with Kimi K2 via OpenRouter

### 1. Workflow Overview

This workflow enables natural language interaction with Stripe data using the Kimi K2 AI model via OpenRouter. It allows users to query and manipulate Stripe data by sending chat messages, which are processed by an AI agent empowered with specific Stripe API capabilities. The workflow supports both reading data (customers, charges, coupons, balance) and writing data (creating coupons), strictly enforcing permission boundaries for security and clarity.

Logical blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users.
- **1.2 AI Processing:** Uses an AI agent powered by Kimi K2 (via OpenRouter) to interpret user requests and orchestrate Stripe API calls.
- **1.3 Context Management:** Maintains chat context with a memory buffer to provide coherent conversation flow.
- **1.4 Stripe Read Operations:** Retrieves data from Stripe such as customers, charges, coupons, and balance.
- **1.5 Stripe Write Operations:** Performs authorized write actions, specifically creating coupons in Stripe.
- **1.6 Documentation & Requirements:** Provides notes on required credentials and usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures incoming chat messages to trigger the workflow.
- **Nodes Involved:** `When chat message received`
- **Node Details:**
  - **Type:** Chat Trigger (LangChain)
  - **Role:** Listens for incoming chat messages via webhook to start the workflow.
  - **Configuration:** Default options; webhook ID assigned.
  - **Inputs:** External HTTP/webhook requests with chat messages.
  - **Outputs:** Passes the message to `AI Agent`.
  - **Edge Cases:** Webhook misconfigurations, message format errors, network issues.
  - **Version:** 1.1

#### 2.2 AI Processing

- **Overview:** Processes user input through an AI agent that understands and executes authorized Stripe operations.
- **Nodes Involved:** `AI Agent`, `OpenRouter Chat Model`, `Simple Memory`
- **Node Details:**

  - **AI Agent**
    - **Type:** LangChain AI Agent Node
    - **Role:** Core logic interpreting chat messages, deciding which Stripe operations to invoke.
    - **Configuration:** 
      - System message defines strict access rights: read customers, charges, coupons, balance; write only for creating coupons.
      - Enforces concise, accurate responses and user confirmation for write actions.
    - **Key Expressions:** None dynamic; system prompt is static.
    - **Inputs:** Receives chat messages from `When chat message received`.
    - **Outputs:** Calls Stripe tool nodes or returns response to chat.
    - **Edge Cases:** Misinterpretation, unauthorized requests, API call failures.
    - **Version:** 2

  - **OpenRouter Chat Model**
    - **Type:** LangChain OpenRouter Chat Model
    - **Role:** Provides the language model backend (Kimi K2) for AI Agent.
    - **Configuration:** Model set to `moonshotai/kimi-k2`.
    - **Credentials:** Uses OpenRouter API key.
    - **Connections:** Feeds AI Agent’s language model interface.
    - **Edge Cases:** API quota limits, network timeouts, invalid credentials.
    - **Version:** 1

  - **Simple Memory**
    - **Type:** LangChain Memory Buffer Window
    - **Role:** Maintains context of last 20 messages for coherent conversations.
    - **Configuration:** Context window length set to 20 messages.
    - **Connections:** Memory linked to AI Agent.
    - **Edge Cases:** Memory overflow or loss, context desynchronization.
    - **Version:** 1.3

#### 2.3 Stripe Read Operations

- **Overview:** Retrieves various Stripe data based on AI Agent’s decision.
- **Nodes Involved:** `Get many customers in Stripe`, `Get many charges in Stripe`, `Get many coupons in Stripe`, `Get a balance in Stripe`
- **Node Details:**

  - **Get many customers in Stripe**
    - **Type:** Stripe Tool Node
    - **Role:** Retrieves a list of customers from Stripe.
    - **Configuration:** 
      - Resource: Customer
      - Operation: Get All
      - Limit: Dynamically set via AI override variable `Limit` (number).
    - **Credentials:** Stripe API key.
    - **Inputs:** Invoked by AI Agent.
    - **Outputs:** Returns customer list.
    - **Edge Cases:** API rate limits, invalid limit values, authentication errors.
    - **Version:** 1

  - **Get many charges in Stripe**
    - **Type:** Stripe Tool Node
    - **Role:** Retrieves a list of charge transactions.
    - **Configuration:** 
      - Resource: Charge
      - Operation: Get All
      - Limit: From AI override variable `Limit`.
    - **Credentials:** Stripe API key.
    - **Inputs:** Invoked by AI Agent.
    - **Outputs:** List of charges.
    - **Edge Cases:** Same as above.
    - **Version:** 1

  - **Get many coupons in Stripe**
    - **Type:** Stripe Tool Node
    - **Role:** Retrieves available coupons.
    - **Configuration:** 
      - Resource: Coupon
      - Operation: Get All
      - Limit: From AI override `Limit`.
    - **Credentials:** Stripe API key.
    - **Inputs:** Invoked by AI Agent.
    - **Outputs:** List of coupons.
    - **Edge Cases:** API failures, invalid limits.
    - **Version:** 1

  - **Get a balance in Stripe**
    - **Type:** Stripe Tool Node
    - **Role:** Retrieves current Stripe balance.
    - **Configuration:** No parameters needed.
    - **Credentials:** Stripe API key.
    - **Inputs:** Invoked by AI Agent.
    - **Outputs:** Balance details.
    - **Edge Cases:** API availability, credentials.
    - **Version:** 1

#### 2.4 Stripe Write Operations

- **Overview:** Executes authorized write actions, specifically creating coupons.
- **Nodes Involved:** `Create a coupon in Stripe`
- **Node Details:**

  - **Create a coupon in Stripe**
    - **Type:** Stripe Tool Node
    - **Role:** Creates a new coupon with a percentage discount.
    - **Configuration:** 
      - Resource: Coupon
      - Percent Off: Dynamically set from AI override variable `Percent_Off` (number).
    - **Credentials:** Stripe API key.
    - **Inputs:** Invoked by AI Agent after user confirmation.
    - **Outputs:** Details of created coupon.
    - **Edge Cases:** Invalid percentage, API errors, permission issues.
    - **Version:** 1

#### 2.5 Documentation & Requirements

- **Overview:** Provides essential setup instructions and notes.
- **Nodes Involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`
- **Node Details:**
  - **Sticky Note**
    - Content: "## Read data"
    - Positioning: Above Stripe read nodes, grouping them visually.
  - **Sticky Note1**
    - Content: "## Write data"
    - Positioned near coupon creation node.
  - **Sticky Note2**
    - Content: 
      ```
      ## Requirements

      - An **Stripe** account and API key credentials enabled. n8n docs: https://docs.n8n.io/integrations/builtin/credentials/stripe/ 
      - An **[OpenRouter](https://openrouter.ai)** account and API credentials enabled. Set up an API key at https://openrouter.ai/settings/keys 
      ```
    - Color code: 5 (distinct)
    - Positioned at workflow start to indicate setup necessities.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                   | Input Node(s)            | Output Node(s)            | Sticky Note                              |
|----------------------------|----------------------------------|---------------------------------|-------------------------|---------------------------|-----------------------------------------|
| When chat message received | LangChain Chat Trigger            | Entry point for incoming chats  | —                       | AI Agent                  |                                         |
| AI Agent                   | LangChain AI Agent                | Orchestrates Stripe API calls   | When chat message received, Simple Memory, OpenRouter Chat Model | Stripe Tool nodes        |                                         |
| OpenRouter Chat Model      | LangChain OpenRouter Model        | Provides AI language model      | —                       | AI Agent                  |                                         |
| Simple Memory              | LangChain Memory Buffer           | Maintains chat context          | —                       | AI Agent                  |                                         |
| Get many customers in Stripe | Stripe Tool                     | Reads Stripe customer data      | AI Agent                 | AI Agent                  | "## Read data"                          |
| Get many charges in Stripe | Stripe Tool                      | Reads Stripe charge data        | AI Agent                 | AI Agent                  | "## Read data"                          |
| Get many coupons in Stripe | Stripe Tool                      | Reads Stripe coupon data        | AI Agent                 | AI Agent                  | "## Read data"                          |
| Get a balance in Stripe    | Stripe Tool                      | Reads Stripe balance            | AI Agent                 | AI Agent                  | "## Read data"                          |
| Create a coupon in Stripe  | Stripe Tool                      | Writes (creates) coupon         | AI Agent                 | AI Agent                  | "## Write data"                         |
| Sticky Note                | Sticky Note                     | Visual grouping for read data   | —                       | —                         | "## Read data"                          |
| Sticky Note1               | Sticky Note                     | Visual grouping for write data  | —                       | —                         | "## Write data"                         |
| Sticky Note2               | Sticky Note                     | Setup instructions and notes    | —                       | —                         | Requirements for Stripe and OpenRouter |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add `When chat message received` (LangChain Chat Trigger).
   - Configure with default options; obtain webhook URL for external chat input.

2. **Add AI Agent Node:**
   - Add `AI Agent` (LangChain AI Agent).
   - Set system message as:
     ```
     You are an AI agent with authorized access to a defined set of Stripe tools...
     [Full system message as in workflow]
     ```
   - Connect input from `When chat message received`.
   - Connect AI Agent to AI language model, memory, and Stripe tools.

3. **Add OpenRouter Chat Model Node:**
   - Add `OpenRouter Chat Model` (LangChain OpenRouter model).
   - Set model to `moonshotai/kimi-k2`.
   - Add OpenRouter API credentials:
     - Obtain API key from https://openrouter.ai/settings/keys.
     - Create credential in n8n for OpenRouter with this key.
   - Connect output to `AI Agent` as language model.

4. **Add Memory Node:**
   - Add `Simple Memory` (LangChain Memory Buffer Window).
   - Set context window length to 20 messages.
   - Connect memory to `AI Agent`.

5. **Add Stripe Credentials in n8n:**
   - Obtain Stripe API key.
   - Create Stripe API credential in n8n.

6. **Add Stripe Tool Nodes for Reading Data:**
   - For customers:
     - Add `Stripe Tool` node.
     - Set resource to `customer`, operation to `getAll`.
     - For `limit`, set expression: `{{$fromAI('Limit', '', 'number')}}`.
     - Assign Stripe credentials.
     - Connect to AI Agent via `ai_tool`.
   - Repeat similarly for:
     - Charges: resource `charge`, operation `getAll`.
     - Coupons: resource `coupon`, operation `getAll`.
     - Balance: resource `balance`, no parameters.

7. **Add Stripe Tool Node for Writing Data:**
   - Add `Stripe Tool` node for creating coupon.
   - Set resource to `coupon`.
   - Set `percentOff` parameter to expression: `{{$fromAI('Percent_Off', '', 'number')}}`.
   - Assign Stripe credentials.
   - Connect to AI Agent via `ai_tool`.

8. **Connect Nodes:**
   - `When chat message received` → `AI Agent` (main).
   - `AI Agent` → each Stripe Tool node (`ai_tool`).
   - `OpenRouter Chat Model` → `AI Agent` (`ai_languageModel`).
   - `Simple Memory` → `AI Agent` (`ai_memory`).

9. **Add Sticky Notes (Optional for Clarity):**
   - Add sticky notes to group read and write nodes.
   - Add sticky note with requirements and links for setting up Stripe and OpenRouter credentials.

10. **Activate Workflow:**
    - Test webhook endpoint by sending chat messages.
    - Confirm AI agent processes requests and triggers corresponding Stripe API calls.
    - Handle errors by checking logs and API responses.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                           |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Stripe API credentials setup documentation                                                    | https://docs.n8n.io/integrations/builtin/credentials/stripe/ |
| OpenRouter account and API key setup instructions                                            | https://openrouter.ai/settings/keys                       |
| The AI agent enforces strict permission boundaries to protect sensitive financial operations  | Security best practices for financial data access         |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It complies with content policies and contains no illegal or protected elements. All data handled is legal and public.