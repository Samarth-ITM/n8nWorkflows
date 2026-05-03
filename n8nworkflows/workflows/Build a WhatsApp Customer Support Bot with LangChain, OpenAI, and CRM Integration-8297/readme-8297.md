Build a WhatsApp Customer Support Bot with LangChain, OpenAI, and CRM Integration

https://n8nworkflows.xyz/workflows/build-a-whatsapp-customer-support-bot-with-langchain--openai--and-crm-integration-8297


# Build a WhatsApp Customer Support Bot with LangChain, OpenAI, and CRM Integration

### 1. Workflow Overview

This workflow is designed to build a comprehensive WhatsApp Customer Support Bot that leverages LangChain, OpenAI language models, and CRM integration to provide automated and context-aware customer service for businesses. The bot can classify incoming WhatsApp messages, route them to specialized AI agents with context-specific prompts, interact with CRM systems and product databases, and send replies back to users on WhatsApp.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming WhatsApp messages via webhook and initiates message classification.
- **1.2 Message Classification:** Uses a LangChain-based AI agent to determine the message category (e.g., branches inquiry, menu questions, orders, complaints).
- **1.3 Routing & Prompt Setting:** Routes classified messages to preset system prompts tailored to each category.
- **1.4 AI Processing:** Executes AI agents with relevant memory buffers, language models, and external HTTP tools for CRM and product data.
- **1.5 CRM & Data Integration:** Provides tools to query CRM, loyalty points, product categories, branches, orders, and save CRM records.
- **1.6 Response Delivery:** Sends the AI-generated response back to the user's WhatsApp via the Evolution API integration.
- **1.7 Greeting & Initialization:** Sends an introductory greeting message when appropriate.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives WhatsApp messages via webhook and forwards them for classification.
- **Nodes Involved:**  
  - Webhook: Receive WhatsApp message  
  - AI Agent Router: Classify msg
- **Node Details:**

  - **Webhook: Receive WhatsApp message**  
    - Type: Webhook node (HTTP endpoint)  
    - Role: Entry point for incoming WhatsApp messages. Listens for incoming HTTP POST requests from WhatsApp integration.  
    - Configuration: Standard webhook with a unique webhook ID. Triggers when a WhatsApp message is received.  
    - Input: External HTTP POST from WhatsApp  
    - Output: Passes message data to AI Agent Router: Classify msg  
    - Edge Cases: Failure if webhook is not publicly accessible or WhatsApp webhook is misconfigured.

  - **AI Agent Router: Classify msg**  
    - Type: LangChain AI Agent node  
    - Role: Classifies incoming messages into categories for routing.  
    - Configuration: Uses OpenAI Chat Model for classification; retries on failure enabled.  
    - Inputs: Message from webhook; AI memory from Simple Memory node; output parser Structured Output Parser  
    - Outputs: Routes to Switch node  
    - Edge Cases: Classification errors if input is ambiguous or model latency; retry mitigates transient failures.

#### 2.2 Message Classification and Routing

- **Overview:** Routes classified messages to appropriate system prompt blocks based on message category.
- **Nodes Involved:**  
  - Structured Output Parser  
  - Switch  
  - Branches System Prompt  
  - Menu System Prompt  
  - Orders System Prompt  
  - Complaints System Prompt  
  - Send WhatsApp Greeting
- **Node Details:**

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses output from classification AI Agent to structured format for routing.  
    - Input: OpenAI Chat Model2  
    - Output: Provides structured classification output to AI Agent Router: Classify msg  
    - Edge Cases: Parsing errors if AI output format changes or is malformed.

  - **Switch**  
    - Type: Switch node  
    - Role: Routes flow based on classification result to different prompt setups or greeting.  
    - Inputs: Classification output from AI Agent Router  
    - Outputs: Six branches to different prompts and greeting node  
    - Edge Cases: Misrouting if classification output unexpected or missing.

  - **Branches System Prompt**  
    - Type: Set node  
    - Role: Loads system prompt for branch inquiries.  
    - Output: Feeds prompt to AI Agent  
    - Edge Cases: Empty or incorrect prompt configuration.

  - **Menu System Prompt**  
    - Type: Set node  
    - Role: Loads system prompt related to menu inquiries.  
    - Output: Feeds prompt to AI Agent  

  - **Orders System Prompt**  
    - Type: Set node  
    - Role: Loads system prompt for order-related queries.  
    - Output: Feeds prompt to AI Agent  

  - **Complaints System Prompt**  
    - Type: Set node  
    - Role: Loads system prompt for complaint handling.  
    - Output: Feeds prompt to AI Agent  

  - **Send WhatsApp Greeting**  
    - Type: Evolution API node  
    - Role: Sends a greeting message to user on WhatsApp.  
    - Inputs: Triggered when Switch routes to greeting branch  
    - Edge Cases: API failures, message sending errors, rate limits.

#### 2.3 AI Processing and Memory Management

- **Overview:** Executes the AI agent with specific memory buffers and language models for generating responses.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - OpenAI Chat Model1  
  - OpenAI Chat Model2  
  - Simple Memory  
  - Simple Memory-2
- **Node Details:**

  - **AI Agent**  
    - Type: LangChain AI Agent node  
    - Role: Main AI engine that generates responses based on prompts, tools, and memory.  
    - Configuration: Connected to multiple OpenAI Chat Models and HTTP tools (CRM, product search).  
    - Inputs: System prompts, memory buffers, language models, and external tool outputs  
    - Outputs: Sends response to "Send Answer to User's WhatsApp" node  
    - Edge Cases: Model timeouts, tool integration failures, API quota issues.

  - **OpenAI Chat Model** / **OpenAI Chat Model1** / **OpenAI Chat Model2**  
    - Type: LangChain OpenAI Chat Model nodes  
    - Role: Provide language model completions for classification and response generation.  
    - Configuration: Different versions or parameters for classification vs. response generation.  
    - Edge Cases: API key errors, rate limits, model update incompatibilities.

  - **Simple Memory** / **Simple Memory-2**  
    - Type: LangChain Memory Buffer Window nodes  
    - Role: Maintain conversational context and history for classification and response agents respectively.  
    - Inputs/Outputs: Link AI agents and routers to maintain state across interactions.  
    - Edge Cases: Memory overflow or truncation; data privacy considerations.

#### 2.4 CRM & Data Integration Tools

- **Overview:** Provides HTTP request tools to query and update CRM data, loyalty points, product categories, branches, and orders.
- **Nodes Involved:**  
  - crm_search_tool  
  - save_crm_record_tool  
  - get_loyalty_points_tool  
  - items_search_tool  
  - branches_search_tool  
  - categories_search_tool
- **Node Details:**

  - **crm_search_tool**  
    - Type: HTTP Request Tool  
    - Role: Searches CRM database for customer records or info.  
    - Configuration: Uses credentials and API endpoints specific to CRM.  
    - Inputs: Called by AI Agent as needed.  
    - Edge Cases: Network errors, auth failures, invalid queries.

  - **save_crm_record_tool**  
    - Type: HTTP Request Tool  
    - Role: Saves or updates records in CRM.  
    - Edge Cases: Data validation errors, API write failures.

  - **get_loyalty_points_tool**  
    - Type: HTTP Request Tool  
    - Role: Retrieves loyalty points for customers from CRM or loyalty system.  
    - Edge Cases: Missing data, unauthorized access.

  - **items_search_tool**  
    - Type: HTTP Request Tool  
    - Role: Searches product items database.  
    - Edge Cases: Empty results, timeout.

  - **branches_search_tool**  
    - Type: HTTP Request Tool  
    - Role: Retrieves information about branch locations.  
    - Edge Cases: Data inconsistencies.

  - **categories_search_tool**  
    - Type: HTTP Request Tool  
    - Role: Retrieves product category data.  
    - Edge Cases: API downtime.

#### 2.5 Response Delivery

- **Overview:** Sends the AI-generated answer back to the user on WhatsApp.
- **Nodes Involved:**  
  - Send Answer to User's WhatsApp
- **Node Details:**

  - **Send Answer to User's WhatsApp**  
    - Type: Evolution API node  
    - Role: Sends the AI-generated response back to the WhatsApp user.  
    - Configuration: Uses WhatsApp messaging API credentials.  
    - Inputs: Receives output from AI Agent.  
    - Edge Cases: API failures, message formatting issues, rate limits.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                               | Input Node(s)                          | Output Node(s)                          | Sticky Note                                     |
|-------------------------------|----------------------------------|-----------------------------------------------|--------------------------------------|----------------------------------------|------------------------------------------------|
| Webhook: Receive WhatsApp message | Webhook                          | Entry point for incoming WhatsApp messages   | -                                    | AI Agent Router: Classify msg           |                                                |
| AI Agent Router: Classify msg  | LangChain AI Agent                | Classifies user messages into categories      | Webhook: Receive WhatsApp message, Simple Memory, Structured Output Parser | Switch                                  |                                                |
| Structured Output Parser       | LangChain Output Parser Structured | Parses classification output for routing      | OpenAI Chat Model2                    | AI Agent Router: Classify msg            |                                                |
| Switch                        | Switch                           | Routes flow to system prompts or greeting     | AI Agent Router: Classify msg         | Branches System Prompt, Menu System Prompt, Orders System Prompt, Complaints System Prompt, Send WhatsApp Greeting |                                                |
| Branches System Prompt         | Set                              | Loads system prompt for branch inquiries      | Switch                               | AI Agent                                |                                                |
| Menu System Prompt             | Set                              | Loads system prompt for menu questions        | Switch                               | AI Agent                                |                                                |
| Orders System Prompt           | Set                              | Loads system prompt for order queries          | Switch                               | AI Agent                                |                                                |
| Complaints System Prompt       | Set                              | Loads system prompt for complaints             | Switch                               | AI Agent                                |                                                |
| Send WhatsApp Greeting         | Evolution API                    | Sends greeting message to WhatsApp user       | Switch                               | -                                      |                                                |
| AI Agent                      | LangChain AI Agent                | Generates AI responses with memory and tools | System Prompts (Set nodes), OpenAI Chat Models, HTTP Tools, Simple Memory, Simple Memory-2 | Send Answer to User's WhatsApp           |                                                |
| OpenAI Chat Model              | LangChain OpenAI Chat Model      | Language model for classification             | -                                    | AI Agent Router: Classify msg            |                                                |
| OpenAI Chat Model1             | LangChain OpenAI Chat Model      | Language model for AI Agent responses          | -                                    | AI Agent                                |                                                |
| OpenAI Chat Model2             | LangChain OpenAI Chat Model      | Language model for output parsing               | -                                    | Structured Output Parser                |                                                |
| Simple Memory                 | LangChain Memory Buffer Window   | Maintains conversation memory for classification | AI Agent Router: Classify msg         | AI Agent Router: Classify msg            |                                                |
| Simple Memory-2               | LangChain Memory Buffer Window   | Maintains conversation memory for AI Agent    | AI Agent                              | AI Agent                                |                                                |
| crm_search_tool               | HTTP Request Tool                | Searches CRM records                            | AI Agent                              | AI Agent                                |                                                |
| save_crm_record_tool          | HTTP Request Tool                | Saves or updates CRM records                    | AI Agent                              | AI Agent                                |                                                |
| get_loyalty_points_tool       | HTTP Request Tool                | Retrieves loyalty points                        | AI Agent                              | AI Agent                                |                                                |
| items_search_tool             | HTTP Request Tool                | Searches product items                          | AI Agent                              | AI Agent                                |                                                |
| branches_search_tool          | HTTP Request Tool                | Retrieves branch info                           | AI Agent                              | AI Agent                                |                                                |
| categories_search_tool        | HTTP Request Tool                | Retrieves product categories                    | AI Agent                              | AI Agent                                |                                                |
| Send Answer to User's WhatsApp | Evolution API                    | Sends AI-generated response to WhatsApp user  | AI Agent                              | -                                      |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Configure for HTTP POST to receive WhatsApp messages.  
   - Set unique webhook ID. This is the entry point.

2. **Create AI Agent Router: Classify msg**  
   - Type: LangChain AI Agent  
   - Configure with OpenAI Chat Model (classification) and set retry on failure.  
   - Connect webhook output to this node.  
   - Link to Simple Memory node for classification context.  
   - Connect output to Switch node.

3. **Create Structured Output Parser**  
   - Type: LangChain Structured Output Parser  
   - Configure to parse classification AI output into structured data.  
   - Connect OpenAI Chat Model2 output to this parser.  
   - Connect parser output to AI Agent Router node’s ai_outputParser input.

4. **Create Switch Node**  
   - Type: Switch  
   - Configure rules to route based on classification output (e.g., branches, menu, orders, complaints, greeting).  
   - Connect AI Agent Router output main to Switch input.

5. **Create Set Nodes for System Prompts**  
   - Create four Set nodes named: Branches System Prompt, Menu System Prompt, Orders System Prompt, Complaints System Prompt.  
   - Each loads the respective system prompt text content used to guide AI agent responses.  
   - Connect Switch node outputs to these Set nodes according to message classification branches.

6. **Create AI Agent Node**  
   - Type: LangChain AI Agent  
   - Configure with multiple OpenAI Chat Models (OpenAI Chat Model1 for response generation).  
   - Add HTTP Request Tool nodes (crm_search_tool, save_crm_record_tool, get_loyalty_points_tool, items_search_tool, branches_search_tool, categories_search_tool) as AI tools to this agent.  
   - Connect each Set node (system prompts) output to AI Agent main input.  
   - Connect AI Agent output to "Send Answer to User's WhatsApp" node.

7. **Create Memory Buffer Nodes**  
   - Create two LangChain Memory Buffer Window nodes named Simple Memory and Simple Memory-2.  
   - Connect Simple Memory to AI Agent Router (classification) as ai_memory input.  
   - Connect Simple Memory-2 to AI Agent (response generation) as ai_memory input.

8. **Create HTTP Request Tool Nodes for CRM and Data**  
   - Create HTTP Request nodes named crm_search_tool, save_crm_record_tool, get_loyalty_points_tool, items_search_tool, branches_search_tool, categories_search_tool.  
   - Configure each node with appropriate API endpoints and credentials to access CRM, product databases, branches, loyalty systems.  
   - Connect all these tools to the AI Agent node as ai_tool inputs.

9. **Create Send WhatsApp Greeting Node**  
   - Type: Evolution API node  
   - Configure with WhatsApp messaging API credentials.  
   - Connect Switch node’s greeting branch output to this node.  
   - Set message content for greeting.

10. **Create Send Answer to User's WhatsApp Node**  
    - Type: Evolution API node  
    - Configure with WhatsApp messaging API credentials.  
    - Connect AI Agent output to this node.  
    - This node sends the final AI-generated reply back to the user.

11. **Connect all nodes accordingly**  
    - Webhook → AI Agent Router: Classify msg → Switch  
    - Switch → System Prompt Set nodes → AI Agent → Send Answer to User's WhatsApp  
    - Switch → Send WhatsApp Greeting (for greeting branch)  
    - Memory nodes linked to classification and AI Agent nodes.  
    - HTTP Tools linked to AI Agent as ai_tools.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                             |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------|
| The workflow uses Evolution API nodes for WhatsApp messaging integration, requiring valid API keys. | Evolution API official docs for setup      |
| LangChain agent nodes provide modular AI tool and memory integration enabling complex workflows.    | LangChain documentation: https://docs.langchain.com/ |
| Multiple OpenAI Chat Models are used separately for classification and response generation to optimize performance. | OpenAI API documentation                    |
| Memory Buffer Window nodes maintain conversation context to improve user experience across messages. | LangChain memory management best practices |
| HTTP Request Tools must be configured with correct CRM API endpoints and credentials for full functionality. | CRM vendor API docs                         |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.