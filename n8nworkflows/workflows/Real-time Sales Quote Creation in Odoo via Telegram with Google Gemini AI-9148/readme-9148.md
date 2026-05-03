Real-time Sales Quote Creation in Odoo via Telegram with Google Gemini AI

https://n8nworkflows.xyz/workflows/real-time-sales-quote-creation-in-odoo-via-telegram-with-google-gemini-ai-9148


# Real-time Sales Quote Creation in Odoo via Telegram with Google Gemini AI

### 1. Workflow Overview

This workflow automates the creation of real-time sales quotations in Odoo based on customer interactions via Telegram, leveraging Google Gemini AI for natural language understanding and LangChain agents for orchestration. It targets sales teams or businesses that want to streamline quotation generation through conversational AI on Telegram, integrating product pricing, customer verification, and order creation within Odoo.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Telegram Trigger receives customer messages.
- **1.2 AI Processing & Intent Handling:** Google Gemini Chat Models and LangChain Agents interpret customer inputs, classify feedback, and orchestrate next steps.
- **1.3 Customer Verification:** Nodes query Odoo to check if the customer exists or needs creation.
- **1.4 Product & Pricing Retrieval:** Odoo nodes fetch product data and pricing details.
- **1.5 Sales Order & Quotation Creation:** Creation and management of sales orders and quotation lines in Odoo.
- **1.6 Output & Telegram Response:** Prepares outputs and sends responses back to the customer via Telegram.
- **1.7 Workflow Control & Error Handling:** Switch and Code nodes control flow based on AI decisions and data validations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures incoming customer messages from Telegram to initiate the workflow.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram messages.  
    - Config: Uses a dedicated webhook ID to listen for Telegram updates.  
    - Inputs: None (triggered by incoming message).  
    - Outputs: Passes message data to AI Processing block.  
    - Edge Cases: Telegram API downtime, webhook failures, malformed messages.

---

#### 1.2 AI Processing & Intent Handling

- **Overview:**  
  Uses Google Gemini Chat Models and LangChain Agents to interpret messages, classify feedback, decide flow branches, and manage AI-driven logic.

- **Nodes Involved:**  
  - From AI  
  - Google Gemini Chat Model  
  - Window Buffer Memory1  
  - Switch  
  - AI Agent  
  - Google Gemini Chat Model1  
  - From AI1  
  - Google Gemini Chat Model3  
  - Check Feedback  
  - Google Gemini Chat Model5  
  - Code  
  - Code1  
  - Telegram2

- **Node Details:**

  - **From AI**  
    - Type: LangChain agent node processing initial AI interpretation.  
    - Config: Uses Google Gemini as language model (linked via ai_languageModel connection).  
    - Input: Telegram Trigger output.  
    - Output: Passes to Code node for further flow control.  
    - Edge Cases: AI timeout, invalid response, API quota limits.

  - **Google Gemini Chat Model**  
    - Type: Language model node using Google Gemini for chat completions.  
    - Config: Default settings, connected to From AI node.  
    - Role: Provides NLP capabilities for initial input understanding.

  - **Window Buffer Memory1**  
    - Type: Memory buffer to maintain conversation context across interactions.  
    - Config: Attached as ai_memory to From AI node.  
    - Role: Preserves dialogue context for better AI coherence.

  - **Switch**  
    - Type: Flow control based on AI output classification.  
    - Config: Routes to either From AI1, AI Agent, or discards input.  
    - Role: Determines next logical step after initial AI analysis.

  - **AI Agent**  
    - Type: LangChain agent for further AI-driven processing with retries.  
    - Config: Max 2 tries, 2 seconds wait between tries.  
    - Input: Switch node output for AI follow-up actions.  
    - Output: Sends result to Telegram2 for response.

  - **Google Gemini Chat Model1**  
    - Type: Language model node linked to AI Agent for chat completions.

  - **From AI1**  
    - Type: Secondary LangChain agent handling feedback classification.  
    - Config: Similar retry and wait settings.  
    - Output: Connects to Check Feedback node.

  - **Google Gemini Chat Model3**  
    - Type: Language model for From AI1.

  - **Check Feedback**  
    - Type: Text classifier node to analyze AI feedback and decide flow path.  
    - Output: Routes either to Code1 or Telegram1 node.

  - **Google Gemini Chat Model5**  
    - Type: Language model assisting Check Feedback node.

  - **Code**  
    - Type: JavaScript code node for custom logic after From AI.  
    - Role: Processes AI output to determine next steps.

  - **Code1**  
    - Type: JavaScript code node after Check Feedback for processing classified feedback.

  - **Telegram2**  
    - Type: Telegram node sending messages back to user based on AI Agent output.

- **Edge Cases:**  
  - AI model rate limits or failures.  
  - Misclassification by Switch or Check Feedback nodes.  
  - Telegram sending failures.

---

#### 1.3 Customer Verification

- **Overview:**  
  Checks if the customer exists in Odoo; creates a new contact if not.

- **Nodes Involved:**  
  - Execute Workflow Trigger  
  - Edit Fields3  
  - Switch2  
  - Switch1  
  - Search Customer  
  - Search Customer1  
  - Check Existing Customer  
  - Create Contact  
  - Code2  
  - Code6  
  - Code7

- **Node Details:**

  - **Execute Workflow Trigger**  
    - Type: Executes the whole workflow or sub-workflow on demand.  
    - Input: External triggers or internal calls.

  - **Edit Fields3**  
    - Type: Set node to prepare or modify input data for customer search.

  - **Switch2**  
    - Type: Branches based on whether customer data is found or not.

  - **Switch1**  
    - Type: Further branching for product data handling in search flow.

  - **Search Customer**  
    - Type: Odoo node querying customer records by parameters such as phone or email.  
    - Output: Passes data to Code2 for verification.

  - **Search Customer1**  
    - Type: Another Odoo customer search node, likely for different criteria or fallback.

  - **Check Existing Customer**  
    - Type: Switch node to decide if customer exists or if creation is needed.  
    - Output: Connects to Create Contact or Code7.

  - **Create Contact**  
    - Type: Odoo node to create new customer/contact record.  
    - Input: Customer details from earlier nodes.  
    - Output: Passes created contact to Search Customer1 for verification.

  - **Code2, Code6, Code7**  
    - Type: JavaScript code nodes for custom logic in customer verification and handling.

- **Edge Cases:**  
  - Odoo API authentication or connectivity errors.  
  - Customer duplicates or incomplete data leading to search failures.  
  - Contact creation failures due to validation errors.

---

#### 1.4 Product & Pricing Retrieval

- **Overview:**  
  Retrieves product information and pricing from Odoo to prepare lines for quotations.

- **Nodes Involved:**  
  - Product Data  
  - Edit Fields1  
  - Prepare output  
  - Get Product Pricing (Sub-workflow)  
  - Prepare Item Lines  
  - Split Out3  
  - Get Product Variant

- **Node Details:**

  - **Product Data**  
    - Type: Odoo node fetching product details based on input criteria.

  - **Edit Fields1**  
    - Type: Set node modifying product data for downstream nodes.

  - **Prepare output**  
    - Type: Code node formatting product data for quotation.

  - **Get Product Pricing**  
    - Type: LangChain toolWorkflow node that likely calls a sub-workflow to query pricing rules or AI-based pricing insights.  
    - Input: Connects to AI Agent for complex price determination.

  - **Prepare Item Lines**  
    - Type: Code node to prepare product lines for the sales order.

  - **Split Out3**  
    - Type: SplitOut node to handle multiple product line items individually.

  - **Get Product Variant**  
    - Type: Odoo node fetching specific product variant details.

- **Edge Cases:**  
  - Product not found or variants missing.  
  - Pricing data inaccessible or inconsistent.  
  - Sub-workflow failure or communication errors.

---

#### 1.5 Sales Order & Quotation Creation

- **Overview:**  
  Creates sales orders and lines in Odoo based on prepared product and customer data.

- **Nodes Involved:**  
  - Sale Order1  
  - Prepare Item Lines  
  - Split Out3  
  - Sale Order Line1  
  - Get Sales Order line Id  
  - Get Sales Order Details  
  - Prepare output3  
  - Sale Quotation1 (LangChain toolWorkflow)  
  - Create Sales Order SubWorkflow

- **Node Details:**

  - **Sale Order1**  
    - Type: Odoo node creating or updating sale orders.

  - **Prepare Item Lines**  
    - Type: Code node preparing item lines for order lines.

  - **Split Out3**  
    - Type: Splits items to process individually.

  - **Sale Order Line1**  
    - Type: Odoo node creating sales order lines.

  - **Get Sales Order line Id**  
    - Type: Odoo node fetching IDs of sales order lines for reference.

  - **Get Sales Order Details**  
    - Type: Odoo node retrieving full sales order details.

  - **Prepare output3**  
    - Type: Code node formatting the final output of sales order creation.

  - **Sale Quotation1**  
    - Type: LangChain toolWorkflow node, likely handling AI-assisted quotation generation logic.

  - **Create Sales Order SubWorkflow**  
    - Type: Sub-workflow node encapsulating sales order creation steps, called from Quote Creator agent.

- **Edge Cases:**  
  - Odoo API failures during order creation.  
  - Data mismatches causing order line failures.  
  - Sub-workflow invocation issues or timeouts.

---

#### 1.6 Output & Telegram Response

- **Overview:**  
  Formats outputs and sends responses back to the user on Telegram.

- **Nodes Involved:**  
  - Quote Creator  
  - Google Gemini Chat Model2  
  - Telegram3  
  - Telegram1

- **Node Details:**

  - **Quote Creator**  
    - Type: LangChain agent orchestrating final quotation message generation.

  - **Google Gemini Chat Model2**  
    - Type: Language model for Quote Creator.

  - **Telegram3**  
    - Type: Telegram node sending the quotation message to the user.

  - **Telegram1**  
    - Type: Telegram node sending messages on feedback cases (e.g., error or clarification).

- **Edge Cases:**  
  - Telegram API message sending errors.  
  - Message formatting issues or truncation.

---

#### 1.7 Workflow Control & Error Handling

- **Overview:**  
  Controls execution flow based on conditions and handles errors or alternate paths.

- **Nodes Involved:**  
  - Switch (multiple: Switch, Switch1, Switch2)  
  - Code (multiple: Code, Code1, Code2, Code6, Code7)  
  - Sticky Notes (various)

- **Node Details:**

  - **Switch nodes**  
    - Handle routing based on AI decisions, data presence, or validation results.

  - **Code nodes**  
    - Include custom JavaScript logic for data manipulation and flow control.

  - **Sticky Notes**  
    - Provide documentation or reminders in the workflow editor (content mostly empty or minimal here).

- **Edge Cases:**  
  - Incorrect or unexpected values causing flow to dead-end.  
  - Expression evaluation failures in code nodes.

---

### 3. Summary Table

| Node Name                  | Node Type                                   | Functional Role                              | Input Node(s)               | Output Node(s)               | Sticky Note |
|----------------------------|---------------------------------------------|----------------------------------------------|-----------------------------|------------------------------|-------------|
| Telegram Trigger           | Telegram Trigger                            | Receive Telegram messages                     | None                        | From AI                      |             |
| From AI                   | LangChain Agent                            | Initial AI message processing                 | Telegram Trigger            | Code                         |             |
| Code                      | Code Node                                 | Custom logic after AI processing               | From AI                     | Switch                       |             |
| Switch                    | Switch                                    | Decide flow path after AI processing           | Code                        | From AI1, AI Agent           |             |
| From AI1                  | LangChain Agent                            | Feedback classification                        | Switch                      | Check Feedback               |             |
| Check Feedback            | Text Classifier                           | Classifies AI feedback                          | From AI1                    | Code1, Telegram1             |             |
| Code1                     | Code Node                                 | Process classified feedback                     | Check Feedback              | Telegram2                    |             |
| AI Agent                  | LangChain Agent                            | AI follow-up processing                         | Switch                      | Telegram2                    |             |
| Telegram2                 | Telegram Node                             | Send AI response to user                        | AI Agent, Code1             |                              |             |
| Google Gemini Chat Model  | Language Model                            | Provide NLP capabilities                        |                             | From AI                      |             |
| Google Gemini Chat Model1 | Language Model                            | Provide NLP for AI Agent                        |                             | AI Agent                     |             |
| Google Gemini Chat Model3 | Language Model                            | Provide NLP for From AI1                         |                             | From AI1                     |             |
| Google Gemini Chat Model5 | Language Model                            | Provide NLP for Check Feedback                   |                             | Check Feedback               |             |
| Google Gemini Chat Model2 | Language Model                            | Provide NLP for Quote Creator                    |                             | Quote Creator                |             |
| Quote Creator             | LangChain Agent                            | Generate quotation messages                      | Code1                       | Telegram3                    |             |
| Telegram3                 | Telegram Node                             | Send quotation message                           | Quote Creator               |                              |             |
| Execute Workflow Trigger  | Execute Workflow Trigger                   | External workflow trigger                        |                             | Edit Fields3                 |             |
| Edit Fields3              | Set Node                                 | Prepare customer data for search                 | Execute Workflow Trigger    | Switch2                      |             |
| Switch2                   | Switch                                    | Route based on customer search results           | Edit Fields3                | Switch1, Search Customer     |             |
| Switch1                   | Switch                                    | Route based on product data availability          | Switch2                     | Product Data                 |             |
| Search Customer           | Odoo Node                                | Search for existing customer                      | Switch2                     | Code2                       |             |
| Code2                     | Code Node                                 | Process customer search results                   | Search Customer             | Check Existing Customer      |             |
| Check Existing Customer   | Switch                                    | Decide to create or reuse customer                 | Code2                       | Create Contact, Code7        |             |
| Create Contact            | Odoo Node                                | Create new customer contact                        | Check Existing Customer     | Search Customer1             |             |
| Search Customer1          | Odoo Node                                | Verify created customer                            | Create Contact              | Code6                       |             |
| Code6                     | Code Node                                 | Process newly created customer data                | Search Customer1            | Sale Order1                 |             |
| Code7                     | Code Node                                 | Handle existing customer case                      | Check Existing Customer     | Sale Order1                 |             |
| Sale Order1               | Odoo Node                                | Create or update sales order                         | Code6, Code7                | Prepare Item Lines           |             |
| Prepare Item Lines        | Code Node                                 | Prepare product lines for sale order                 | Sale Order1                 | Split Out3                  |             |
| Split Out3                | Split Out Node                            | Split multiple product lines                          | Prepare Item Lines          | Get Product Variant          |             |
| Get Product Variant       | Odoo Node                                | Fetch product variant details                         | Split Out3                  | Sale Order Line1             |             |
| Sale Order Line1          | Odoo Node                                | Create sales order line                               | Get Product Variant          | Get Sales Order line Id      |             |
| Get Sales Order line Id   | Odoo Node                                | Retrieve sales order line IDs                         | Sale Order Line1            | Get Sales Order Details      |             |
| Get Sales Order Details   | Odoo Node                                | Retrieve full sales order details                      | Get Sales Order line Id     | Prepare output3              |             |
| Prepare output3           | Code Node                                 | Format sales order details for output                  | Get Sales Order Details     |                              |             |
| Product Data              | Odoo Node                                | Retrieve product information                           | Switch1                     | Edit Fields1                |             |
| Edit Fields1              | Set Node                                 | Prepare product data for pricing                        | Product Data                | Prepare output              |             |
| Prepare output            | Code Node                                 | Format product data for pricing                          | Edit Fields1                |                              |             |
| Get Product Pricing       | LangChain ToolWorkflow                    | Retrieve product pricing via AI or API                  | AI Agent                    |                              |             |
| Sale Quotation1           | LangChain ToolWorkflow                    | AI-assisted sales quotation generation                 |                              |                              |             |
| Create Sales Order SubWorkflow | LangChain ToolWorkflow               | Sub-workflow for order creation                          | Quote Creator               |                              |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with Telegram Bot credentials and a webhook ID.  
   - This node listens for user messages.

2. **Add LangChain Agent Node "From AI"**  
   - Configure to use a Google Gemini Chat Model for language understanding.  
   - Attach a Window Buffer Memory node to maintain conversation context.  
   - Connect the Telegram Trigger output to this node.

3. **Add Google Gemini Chat Model node**  
   - Set it as the language model for the "From AI" LangChain agent.

4. **Add Code Node "Code"**  
   - Input: output of "From AI".  
   - Implement custom JS logic to interpret AI output and decide the next step.

5. **Add Switch Node**  
   - Connect from "Code" node.  
   - Configure branches to route output to either:  
     - "From AI1" (for feedback classification),  
     - "AI Agent" (for follow-up AI processing),  
     - or discard.

6. **Add LangChain Agent "From AI1" and Google Gemini Chat Model3**  
   - Configure as a pair similar to "From AI".  
   - Connect Switch output branch to "From AI1".

7. **Add Text Classifier Node "Check Feedback" and Google Gemini Chat Model5**  
   - Connect output of "From AI1" to "Check Feedback".  
   - Classify feedback to route to either Code1 or Telegram1.

8. **Add Code Node "Code1"**  
   - Input: "Check Feedback" positive branch.  
   - Implement feedback-specific processing.

9. **Add Telegram Node "Telegram2"**  
   - Connect outputs of "AI Agent" and "Code1" to "Telegram2".  
   - Configure with Telegram credentials for sending messages back.

10. **Add LangChain Agent "AI Agent" and Google Gemini Chat Model1**  
    - Connect Switch node to "AI Agent".  
    - Configure retries and wait times for robustness.

11. **Create Customer Verification Branch:**  
    - Add Execute Workflow Trigger for manual starts.  
    - Add Set Node "Edit Fields3" to prepare data.  
    - Add Switch Node "Switch2" to route based on customer existence.  
    - Add two Odoo Search Customer nodes ("Search Customer" and "Search Customer1") with appropriate queries.  
    - Add Switch "Check Existing Customer" to decide on creating a new contact or reusing existing.  
    - Add Odoo Create Contact node.  
    - Add Code nodes "Code2", "Code6", "Code7" for post-processing.  
    - Connect flows accordingly.

12. **Add Product & Pricing Retrieval:**  
    - Add Odoo "Product Data" node for fetching product info.  
    - Add Set Node "Edit Fields1" to prepare product data.  
    - Add Code node "Prepare output" for formatting.  
    - Add LangChain ToolWorkflow "Get Product Pricing" linked to AI Agent.  
    - Add Code "Prepare Item Lines" to prepare sale order lines.  
    - Add SplitOut node "Split Out3" to handle multiple products.  
    - Add Odoo node "Get Product Variant" to fetch variants.

13. **Add Sales Order & Quotation Creation:**  
    - Add Odoo node "Sale Order1" for creating sales orders.  
    - Connect to "Prepare Item Lines".  
    - Add Odoo node "Sale Order Line1" for order lines.  
    - Add Odoo nodes "Get Sales Order line Id" and "Get Sales Order Details" for retrieval.  
    - Add Code node "Prepare output3" to format final data.  
    - Add LangChain toolWorkflow "Sale Quotation1" and sub-workflow "Create Sales Order SubWorkflow".  
    - Connect sub-workflow calls properly.

14. **Add Output & Telegram Response:**  
    - Add LangChain Agent "Quote Creator" with Google Gemini Chat Model2.  
    - Connect to Telegram node "Telegram3" to send quotation messages.

15. **Add Necessary Flow Control Nodes:**  
    - Multiple Switch and Code nodes to handle branching logic.  
    - Implement error and edge case handling.

16. **Configure Credentials:**  
    - Telegram Bot credentials for trigger and message nodes.  
    - Odoo API credentials with appropriate access rights.  
    - Google Gemini AI credentials for language models.  
    - LangChain agent credentials as required.

17. **Test workflow end-to-end** ensuring:  
    - Telegram messages trigger AI processing.  
    - AI correctly interprets intent and fetches or creates customer data.  
    - Product pricing and variants are retrieved correctly.  
    - Sales orders and quotations are created in Odoo.  
    - Responses are sent back to Telegram users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow facilitates real-time sales quotation via Telegram integrating Odoo and Google Gemini AI with LangChain agents.                              | Workflow purpose                                    |
| LangChain integration enhances AI orchestration and memory management for better conversational flow.                                                 | AI & LangChain nodes                                |
| Telegram nodes rely on secure Telegram Bot credentials and webhook configuration.                                                                       | Telegram integration                                |
| Odoo nodes require properly configured Odoo API credentials with permission to read/write customers, products, and sales orders.                      | Odoo integration                                    |
| For more on LangChain AI nodes in n8n, see: https://n8n.io/integrations/n8n-nodes-langchain                                                            | LangChain documentation                             |
| Google Gemini AI nodes require Google Cloud credentials and appropriate API quotas for chat completions.                                               | Google Gemini AI                                   |
| The workflow uses robust retry and wait strategies in AI agent nodes to handle transient errors and rate limits.                                       | Error handling strategies                           |
| Sticky Notes in the workflow contain minimal content but can be used to add contextual info during workflow editing.                                    | Workflow editor notes                               |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.