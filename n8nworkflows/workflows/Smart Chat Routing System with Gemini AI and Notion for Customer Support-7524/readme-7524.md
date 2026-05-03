Smart Chat Routing System with Gemini AI and Notion for Customer Support

https://n8nworkflows.xyz/workflows/smart-chat-routing-system-with-gemini-ai-and-notion-for-customer-support-7524


# Smart Chat Routing System with Gemini AI and Notion for Customer Support

### 1. Workflow Overview

This workflow implements a **Smart Chat Routing System** for customer support using Google Gemini AI models integrated with Notion as a knowledge base. It is designed to handle incoming chat messages, classify them into one of three categories—Customer Service, Question, or Booking—and route them accordingly to specialized AI agents. Each agent leverages Google Gemini language models and the Notion database to provide personalized responses, information retrieval, or consultation booking assistance.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receiving incoming chat messages via webhook.
- **1.2 Lead Classification:** Using an AI routing agent to classify the message into a lead category.
- **1.3 Message Routing:** Directing messages to specific AI agents based on classification.
- **1.4 Customer Service Processing:** Handling customer service inquiries with context memory and Notion CRM integration.
- **1.5 General Question Processing:** Responding to product/service questions, querying Notion Automations database.
- **1.6 Booking Processing:** Managing booking requests with a dedicated agent sending consultation invitation messages.
- **1.7 Data Retrieval from Notion:** Accessing customer and automation databases to enrich AI responses.
- **1.8 Memory Management:** Using window buffer memory nodes to maintain conversational context.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives incoming chat messages via a public webhook to trigger the workflow.
- **Nodes Involved:**  
  - `When chat message received`
- **Node Details:**  
  - **Type:** Chat Trigger (Langchain)  
  - **Role:** Entry point receiving chat input  
  - **Configuration:** Public webhook enabled, no additional options  
  - **Expressions/Variables:** Outputs `chatInput` and `sessionId` from incoming message  
  - **Connections:** Feeds into `Routing Agent` node  
  - **Failure Modes:** Webhook unavailability, malformed input, rate limits

---

#### 2.2 Lead Classification

- **Overview:** Classifies the incoming chat message into one of three lead categories for routing.
- **Nodes Involved:**  
  - `Routing Agent`  
  - `Google Gemini Chat Model` (used by Routing Agent)  
  - `Structured Output Parser`  
  - `Simple Memory`
- **Node Details:**  
  - **Routing Agent**  
    - Type: Langchain Agent  
    - Role: AI classification of message intent  
    - Config: Uses a system prompt describing lead categories (Customer Service, Question, Booking) and outputs JSON with `lead_category` field  
    - Inputs: Raw chat message from `When chat message received`  
    - Outputs: JSON classification to `Switch` node  
    - Uses `Google Gemini Chat Model` for NLP  
    - Uses `Structured Output Parser` to enforce JSON output format  
    - Uses `Simple Memory` (window buffer) to maintain context/history to improve classification accuracy  
    - Failure Modes: Misclassification, model latency, parsing errors  
  - **Google Gemini Chat Model**  
    - Type: Language Model (Google Gemini)  
    - Role: Provides NLP understanding for classification  
    - Config: Default options, version 1  
  - **Structured Output Parser**  
    - Type: Output Parser (Langchain)  
    - Role: Parses model output enforcing JSON schema with `lead_category`  
    - Config: JSON schema example expects `lead_category` string  
  - **Simple Memory**  
    - Type: Window Buffer Memory  
    - Role: Keeps recent conversation context for routing agent to improve classification  
    - Config: Context window length 15 messages

---

#### 2.3 Message Routing

- **Overview:** Routes classified chat messages to the appropriate AI agent for handling based on lead category.
- **Nodes Involved:**  
  - `Switch`
- **Node Details:**  
  - Type: Switch node  
  - Role: Routes message based on `lead_category` value from `Routing Agent` output  
  - Configuration:  
    - Routes "Customer Service" → `Customer Service Agent`  
    - Routes "Question" → `Question Agent`  
    - Routes "Booking" → `Booking Agent`  
  - Inputs: Classification JSON from `Routing Agent`  
  - Outputs: Three possible branches connecting to respective agents  
  - Failure Modes: Missing or malformed classification output

---

#### 2.4 Customer Service Processing

- **Overview:** Handles customer service requests by interacting with Notion customer data and product info, providing personalized support.
- **Nodes Involved:**  
  - `Customer Service Agent`  
  - `Window Buffer Memory2`  
  - `Google Gemini Chat Model2`  
  - `Get Customers` (Notion)  
  - `Get Automations` (Notion)
- **Node Details:**  
  - **Customer Service Agent**  
    - Type: Langchain Agent  
    - Role: Acts as a customer support AI agent with rich system prompt guiding behavior and interaction flow  
    - Configuration:  
      - System prompt instructs agent to check Notion CRM for user data, diagnose issues, recommend solutions from knowledge base  
      - Receives chat input from original message  
    - Inputs: Chat input and data from Notion nodes (`Get Customers`, `Get Automations`)  
    - Memory: Uses `Window Buffer Memory2` for conversation context  
    - Language Model: Uses `Google Gemini Chat Model2`  
    - Failure Modes: Notion API failures, model timeouts, memory inconsistencies  
  - **Get Customers**  
    - Type: Notion Database (databasePage.getAll)  
    - Role: Fetches customer profiles from Notion CRM for agent reference  
    - Config: Returns all entries from customer database  
  - **Get Automations**  
    - Type: Notion Database (databasePage.getAll)  
    - Role: Fetches automation product details to recommend solutions  
    - Config: Returns filtered automation entries based on AI input variables  
  - **Window Buffer Memory2**  
    - Maintains dialogue context for customer service agent

---

#### 2.5 General Question Processing

- **Overview:** Responds to general inquiries about products and services by querying Notion knowledge base and using AI summarization.
- **Nodes Involved:**  
  - `Question Agent`  
  - `Window Buffer Memory1`  
  - `Google Gemini Chat Model1`  
  - `Get Automations1` (Notion)  
  - `Get More Info` (Notion)
- **Node Details:**  
  - **Question Agent**  
    - Type: Langchain Agent  
    - Role: Provides informative answers about products/services by querying Notion Automations database  
    - Configuration: System prompt details steps to query Notion, summarize info, and compose structured JSON output  
    - Inputs: Original chat input  
    - Memory: Uses `Window Buffer Memory1` to maintain conversational context  
    - Language Model: Uses `Google Gemini Chat Model1`  
  - **Get Automations1**  
    - Fetches automation product pages from Notion for matching user queries  
  - **Get More Info**  
    - Fetches detailed nested blocks from selected Notion pages to enrich responses  
  - Failure Modes: Notion API limits, no matched results, model hallucination (mitigated by strict output contract)

---

#### 2.6 Booking Processing

- **Overview:** Manages booking requests by sending a predefined consultation invitation message via an AI agent.
- **Nodes Involved:**  
  - `Booking Agent`  
  - `Google Gemini Chat Model5`
- **Node Details:**  
  - **Booking Agent**  
    - Type: Langchain Agent  
    - Role: Sends a fixed message inviting user to fill out a consultation form  
    - Configuration: System prompt contains a fixed message with a URL to online intake form  
    - Inputs: Triggered from `Switch` node when category is "Booking"  
    - Language Model: Uses `Google Gemini Chat Model5` but acts mostly as message sender (minimal AI generation)  
  - Failure Modes: Message delivery failure, link unavailability

---

#### 2.7 Data Retrieval from Notion

- **Overview:** Provides access to Notion databases—customer profiles and automation knowledge—to enable AI agents to ground their responses.
- **Nodes Involved:**  
  - `Notion` (initial fetch for Customer Service routing)  
  - `Get Customers`  
  - `Get Automations`  
  - `Get Automations1`  
  - `Get More Info`
- **Node Details:**  
  - All nodes use Notion credentialed API access  
  - Return all entries or filtered entries based on AI-provided parameters  
  - `Get More Info` supports nested block retrieval for detailed content  
  - Failure Modes: API authentication errors, rate limits, data schema changes in Notion

---

#### 2.8 Memory Management

- **Overview:** Maintains conversational context for AI agents to improve continuity and context-awareness.
- **Nodes Involved:**  
  - `Simple Memory` (for routing agent)  
  - `Window Buffer Memory1` (for Question Agent)  
  - `Window Buffer Memory2` (for Customer Service Agent)
- **Node Details:**  
  - Type: Langchain Memory Buffer Window  
  - Role: Keeps recent chat messages or interactions in memory with configurable window size  
  - Parameters: Default context window length 15 for `Simple Memory`; default for others  
  - Inputs and outputs connected to respective agents  
  - Failure Modes: Memory overflow, session ID mismatches

---

### 3. Summary Table

| Node Name                  | Node Type                                   | Functional Role                              | Input Node(s)                  | Output Node(s)               | Sticky Note                                                        |
|----------------------------|---------------------------------------------|----------------------------------------------|-------------------------------|------------------------------|-------------------------------------------------------------------|
| When chat message received | Chat Trigger (Langchain)                     | Entry point receiving chat messages           | —                             | Routing Agent                 | Note 1: Entry point webhook for incoming chat                     |
| Routing Agent              | Langchain Agent                              | Classifies chat into lead categories          | When chat message received, Notion, Simple Memory, Google Gemini Chat Model, Structured Output Parser | Switch                       | Note 2: AI lead classification node                              |
| Google Gemini Chat Model   | Language Model (Google Gemini)               | Provides NLP for Routing Agent                 | —                             | Routing Agent                |                                                                   |
| Structured Output Parser   | Output Parser (Langchain)                    | Parses classification output to JSON          | Google Gemini Chat Model       | Routing Agent                |                                                                   |
| Simple Memory             | Memory Buffer Window                         | Maintains chat context for Routing Agent      | When chat message received     | Routing Agent                |                                                                   |
| Switch                    | Switch Node                                 | Routes chat based on lead category             | Routing Agent                 | Customer Service Agent, Question Agent, Booking Agent | Note 3: Routes based on lead_category                            |
| Customer Service Agent    | Langchain Agent                              | Handles customer service chats                  | Switch, Get Customers, Get Automations, Window Buffer Memory2, Google Gemini Chat Model2 | —                          | Note 4: Customer service AI agent with Notion integration        |
| Window Buffer Memory2     | Memory Buffer Window                         | Maintains context for Customer Service Agent   | —                             | Customer Service Agent       |                                                                   |
| Google Gemini Chat Model2 | Language Model (Google Gemini)               | Language model for Customer Service Agent      | —                             | Customer Service Agent       |                                                                   |
| Get Customers             | Notion Database (databasePage.getAll)       | Fetches customer profiles                       | —                             | Customer Service Agent       |                                                                   |
| Get Automations           | Notion Database (databasePage.getAll)       | Fetches automations for Customer Service Agent | —                             | Customer Service Agent       |                                                                   |
| Question Agent            | Langchain Agent                              | Answers product/service questions               | Switch, Get Automations1, Get More Info, Window Buffer Memory1, Google Gemini Chat Model1 | —                          | Note 5: Question AI agent with knowledge base queries            |
| Window Buffer Memory1     | Memory Buffer Window                         | Maintains context for Question Agent           | —                             | Question Agent              |                                                                   |
| Google Gemini Chat Model1 | Language Model (Google Gemini)               | Language model for Question Agent               | —                             | Question Agent              |                                                                   |
| Get Automations1          | Notion Database (databasePage.getAll)       | Fetches automation pages for Question Agent    | —                             | Question Agent              |                                                                   |
| Get More Info             | Notion Block (block.getAll)                  | Fetches nested detail blocks for Question Agent | —                             | Question Agent              |                                                                   |
| Booking Agent             | Langchain Agent                              | Sends booking consultation invitation          | Switch, Google Gemini Chat Model5 | —                          |                                                                   |
| Google Gemini Chat Model5 | Language Model (Google Gemini)               | Language model for Booking Agent                | —                             | Booking Agent              |                                                                   |
| Notion                   | Notion Database                             | Initial Notion access for routing               | —                             | Routing Agent                |                                                                   |
| Sticky Note (Workflow Summary) | Sticky Note                                | Describes high-level workflow overview          | —                             | —                            |                                                                   |
| Setup Note 1              | Sticky Note                                  | Entry point explanation                         | —                             | —                            | Note 1: Entry point node description                             |
| Setup Note 2              | Sticky Note                                  | Routing agent classification explanation        | —                             | —                            | Note 2: Lead classification explanation                         |
| Setup Note 3              | Sticky Note                                  | Switch node routing explanation                  | —                             | —                            | Note 3: Switch routing explanation                              |
| Setup Note 4              | Sticky Note                                  | Customer Service AI agent explanation            | —                             | —                            | Note 4: Customer Service agent details                          |
| Setup Note 5              | Sticky Note                                  | Question AI agent explanation                     | —                             | —                            | Note 5: Question agent details                                  |
| Sticky Note               | Sticky Note                                  | Consultation booking label                        | —                             | —                            | "CONSULTATION BOOKING FLOW"                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure as public webhook, no extra options.  
   - Position: Start of workflow.  

2. **Add Google Gemini Chat Model Node (Routing):**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Use default options, version 1.  

3. **Add Structured Output Parser Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Set JSON schema example to expect `{ "lead_category": "Customer " }`.  

4. **Add Simple Memory Node for Routing Context:**  
   - Node Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set context window length to 15 messages.  

5. **Create Routing Agent Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set system prompt describing lead categories (Customer Service, Question, Booking).  
   - Connect input from Chat Trigger node.  
   - Link to Google Gemini Chat Model, Structured Output Parser, and Simple Memory nodes.  
   - Set output to feed into a Switch node.  

6. **Create Switch Node:**  
   - Node Type: `Switch`  
   - Configure rules to check the `lead_category` output from Routing Agent:  
     - If `Customer Service` → route to Customer Service Agent  
     - If `Question` → route to Question Agent  
     - If `Booking` → route to Booking Agent  

7. **Set up Customer Service AI Agent:**  
   - Create Google Gemini Chat Model2 node (default options).  
   - Create Window Buffer Memory2 node for context.  
   - Create Notion node to fetch customer database (`Get Customers`), configured with correct database ID and credentials.  
   - Create Notion node to fetch automation database (`Get Automations`), configured similarly.  
   - Create Customer Service Agent node:  
     - Type: Langchain Agent  
     - Input: Chat input and Notion data (`Get Customers`, `Get Automations`)  
     - System prompt: Detailed instructions for customer support including checking Notion CRM and recommending solutions.  
     - Connect to Google Gemini Chat Model2 and Window Buffer Memory2 nodes.  

8. **Set up Question AI Agent:**  
   - Create Google Gemini Chat Model1 node.  
   - Create Window Buffer Memory1 node.  
   - Create Notion node to fetch automations database (`Get Automations1`).  
   - Create Notion node to fetch detailed blocks (`Get More Info`) with input parameter `blockId` from AI variable.  
   - Create Question Agent node:  
     - Type: Langchain Agent  
     - System prompt instructs querying Notion automations and summarizing.  
     - Connect inputs from Chat Trigger, Notion nodes, Google Gemini Chat Model1, and Window Buffer Memory1.  

9. **Set up Booking Agent:**  
   - Create Google Gemini Chat Model5 node.  
   - Create Booking Agent node:  
     - Type: Langchain Agent  
     - System prompt contains fixed message inviting user to consultation form with URL.  
     - Connect input from Switch node when booking category is selected.  

10. **Connect all nodes according to the routing logic:**  
    - Chat Trigger → Routing Agent  
    - Routing Agent → Switch  
    - Switch → Customer Service Agent / Question Agent / Booking Agent (based on classification)  

11. **Credential Configurations:**  
    - Configure Notion credentials for all Notion nodes with access to relevant databases.  
    - Configure Google Gemini credentials or API keys for all Gemini model nodes.  

12. **Set default parameter values and constraints:**  
    - Context window lengths as specified.  
    - JSON schema strictly enforced for classification output.  
    - Enable retry on fail for Routing Agent (max 5 tries).  

13. **Add Sticky Notes for documentation (optional):**  
    - Add notes describing each major block as in the original workflow for clarity.  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow uses Google Gemini AI models integrated with Notion CRM and product databases for customer support automation. | Workflow purpose and architecture overview       |
| The system prompt for Customer Service Agent includes detailed instructions to verify users via Notion CRM and recommend solutions based on product database. | Customer Service Agent system prompt details      |
| The Question Agent strictly outputs valid JSON for web chat widget compatibility, avoiding hallucinations by grounding in Notion data. | Question Agent output contract details            |
| Booking Agent sends a static invitation message with a link to a consultation booking form: https://kkbranding.com/intake | Booking Agent invitation message and URL         |
| The routing logic classifies incoming messages into three categories, enabling efficient lead handling and personalized AI responses. | Routing logic and lead classification overview    |

---

**Disclaimer:** The text provided is exclusively generated from an automated workflow built with n8n, strictly adhering to content policies without illegal or offensive elements. All data processed is legal and public.