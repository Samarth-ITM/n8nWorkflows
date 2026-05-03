Conversational Razorpay Analytics with Google Gemini & Telegram Bot

https://n8nworkflows.xyz/workflows/conversational-razorpay-analytics-with-google-gemini---telegram-bot-10815


# Conversational Razorpay Analytics with Google Gemini & Telegram Bot

---

## 1. Workflow Overview

**Workflow Title:** Conversational Razorpay Analytics with Google Gemini & Telegram Bot

**Purpose:**  
This workflow creates an intelligent conversational Telegram bot named **PayInsighter** that provides real-time analytics and insights on Razorpay payments, orders, and refunds. It leverages Google Gemini AI to classify user intents, parse queries for relevant parameters, fetch data from Razorpay APIs, analyze it, and respond concisely. It also handles general chat interactions unrelated to analytics.

**Target Use Cases:**  
- Instantly retrieve Razorpay payment, order, or refund data summaries via Telegram chat  
- Automate customer support and team queries on Razorpay data  
- Provide succinct, AI-generated responses avoiding data flooding  
- Handle multi-intent queries (e.g., orders + payments) seamlessly  
- Support general bot-related inquiries and greetings  

**Logical Blocks:**  

- **1.1 Input Reception & Intent Classification:**  
  Captures Telegram messages and classifies user intent into payments, orders, refunds, or general chat using Google Gemini AI.

- **1.2 Parallel Routing:**  
  Routes the classified intents into four parallel branches handling payments, orders, refunds, and general chat independently. Supports multi-intent queries by activating multiple branches simultaneously.

- **1.3 Razorpay Data Fetch & AI Processing:**  
  Each branch parses user query parameters via AI agents, fetches corresponding Razorpay data (payments, orders, refunds), and uses AI agents to analyze and generate user-friendly summaries.

- **1.4 Response Consolidation & Delivery:**  
  Merges all branch outputs preserving multi-intent responses and sends the consolidated answer back to the user on Telegram.

- **1.5 AI Model Infrastructure:**  
  The entire AI logic is powered by Google Gemini chat models, connected to all AI agent nodes for classification, parameter extraction, data analysis, and response generation.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Intent Classification

**Overview:**  
Receives user queries from Telegram and classifies the intent into one or multiple categories using a Google Gemini-powered AI agent.

**Nodes Involved:**  
- Telegram Trigger  
- Intent Classifier (Langchain Agent)  
- Google Gemini Chat Model (AI Provider)  

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages (updates of type "message") to start the workflow.  
  - Configuration: Webhook ID set; captures message text and metadata (user ID, chat ID).  
  - Input: Telegram user message  
  - Output: JSON with message text, user info  
  - Edge cases: Telegram service downtime, malformed messages  

- **Intent Classifier**  
  - Type: Langchain Agent (AI)  
  - Role: Classifies user message intent into one or multiple categories ("payments", "orders", "refunds", "general_chat") using a text prompt.  
  - Configuration: System message instructs single-word or underscore-separated multi-intent classification; no formatting or special chars allowed.  
  - Key Expression: Uses `{{$json.message.text}}` from Telegram  
  - Input: Telegram message text JSON  
  - Output: Intent string (e.g., "orders_payments")  
  - Error handling: Continues on errors, fallback needed if AI fails or returns invalid output  
  - Version: Langchain Agent v3  
  - Failure modes: AI model API quota, malformed input, ambiguous messages  

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini Chat Model  
  - Role: Provides AI language model backend for intent classification and subsequent AI agents.  
  - Configuration: Default options, model "gemini-2.5-flash" or similar.  
  - Connected to Intent Classifier and other AI agents.  
  - Edge cases: API key or quota issues, latency  

---

### 2.2 Parallel Routing of Intents

**Overview:**  
Uses four IF nodes to route the workflow based on detected intents. Supports multiple branches firing simultaneously for multi-intent queries.

**Nodes Involved:**  
- Payment Branch (IF)  
- Refund Branch (IF)  
- Order Branch (IF)  
- General Chat Branch1 (IF)  

**Node Details:**  

- **Payment Branch (IF)**  
  - Type: If (version 2.2)  
  - Role: Checks if intent string contains "payments" to route payment-related flow.  
  - Condition: `{{ $json.output }} contains "payments"`  
  - Input: Intent string from Intent Classifier  
  - Output: Triggers Payments branch  
  - Edge cases: Empty or malformed intent string  

- **Refund Branch (IF)**  
  - Type: If (version 2.2)  
  - Role: Checks if intent string contains "refunds" to route refund-related flow.  
  - Condition: `{{ $json.output }} contains "refunds"`  
  - Input & output like above  

- **Order Branch (IF)**  
  - Type: If (version 2.2)  
  - Role: Checks if intent string contains "orders" to route orders-related flow.  
  - Condition: `{{ $json.output }} contains "orders"`  

- **General Chat Branch1 (IF)**  
  - Type: If (version 2.2)  
  - Role: Checks if intent string contains "general_chat" for general non-analytics queries.  
  - Condition: `{{ $json.output }} contains "general_chat"`  

---

### 2.3 Payments Branch

**Overview:**  
Processes user queries about payments, extracting date parameters via AI, fetching payments from Razorpay, and producing an AI-generated concise answer.

**Nodes Involved:**  
- Payments Processor (Langchain Agent)  
- Fetch all payments (Razorpay node)  
- AI Agent3 (Langchain Agent)  
- Google Gemini Chat Model1 (AI Model)  

**Node Details:**  

- **Payments Processor**  
  - Type: Langchain Agent  
  - Role: Parses user query to generate JSON with "from", "to", "skip", "count" parameters in date-time format (e.g., "2025-11-07 00:00:00").  
  - System Message: Provides instructions on max count (100), date format, and JSON output without markdown.  
  - Input: Telegram message text  
  - Output: JSON string with parameters  
  - Connected to: Google Gemini Chat Model1  
  - Edge cases: Unclear date ranges, invalid user input, AI parsing errors  

- **Fetch all payments**  
  - Type: Razorpay API node (payment resource)  
  - Role: Calls Razorpay API to fetch payment records with parameters from Payments Processor output.  
  - Parameters: Uses expressions to parse JSON output for "from", "to", "skip", "count".  
  - Input: JSON with query params  
  - Output: Payment records JSON  
  - Edge cases: API errors, authentication failure, rate limits, empty data  

- **AI Agent3 (Response Formatter)**  
  - Type: Langchain Agent  
  - Role: Analyzes payment data and user query; generates concise user-friendly answer avoiding data flooding.  
  - System Message: Instructions to be concise and answer as PayInsighter bot.  
  - Input: Payment data JSON and user query text  
  - Output: Final answer text  
  - Connected to: Google Gemini Chat Model1  

---

### 2.4 Orders Branch

**Overview:**  
Handles order-related queries by extracting parameters via AI, fetching order data from Razorpay, and producing a concise AI response.

**Nodes Involved:**  
- Orders (Langchain Agent)  
- Fetch all orders (Razorpay node)  
- Order Processing Agent (Langchain Agent)  
- Google Gemini Chat Model (AI Model)  

**Node Details:**  

- **Orders**  
  - Type: Langchain Agent  
  - Role: Parses user query to produce JSON with "from", "to", "skip", "count" parameters for orders.  
  - System Message: Specifies max count 100, date format, no markdown in JSON.  
  - Input: Telegram message text  
  - Output: JSON parameters string  
  - Connected to: Google Gemini Chat Model  

- **Fetch all orders**  
  - Type: Razorpay API node (order resource)  
  - Role: Calls Razorpay API to fetch orders with parsed parameters.  
  - Parameters: Extracted via JSON.parse from AI output  
  - Output: Order data JSON  
  - Edge cases: API failures, empty orders, auth issues  

- **Order Processing Agent**  
  - Type: Langchain Agent  
  - Role: Analyzes order data and user query; generates concise answer as PayInsighter bot.  
  - System Message: Conciseness emphasized, no data flooding.  
  - Output: Formatted answer text  
  - Connected to: Google Gemini Chat Model  

---

### 2.5 Refunds Branch

**Overview:**  
Processes refund-related queries by parsing epoch timestamp parameters via AI, fetching refund data from Razorpay, and generating concise AI responses.

**Nodes Involved:**  
- Refunds (Langchain Agent)  
- Fetch all refunds (Razorpay node)  
- Refund Processing Agent (Langchain Agent)  
- Google Gemini Chat Model2 (AI Model)  

**Node Details:**  

- **Refunds**  
  - Type: Langchain Agent  
  - Role: Parses user query to produce JSON with parameters "from", "to" (in epoch timestamps), "skip", "count".  
  - System Message: Specifies epoch time format, max count 100, no markdown.  
  - Input: Telegram message text  
  - Output: JSON parameters string  
  - Connected to: Google Gemini Chat Model2  

- **Fetch all refunds**  
  - Type: Razorpay API node (refund resource)  
  - Role: Calls Razorpay Refunds API with parameters from AI output.  
  - Parameters: Extracted via JSON.parse  
  - Edge cases: API errors, empty refunds, auth failures  

- **Refund Processing Agent**  
  - Type: Langchain Agent  
  - Role: Analyzes refund data and user query; returns concise answer maintaining PayInsighter persona.  
  - System Message: Conciseness and persona emphasized.  
  - Connected to: Google Gemini Chat Model2  

---

### 2.6 General Chat Branch

**Overview:**  
Handles non-analytics general chat queries like greetings or help requests by generating concise AI responses without API calls.

**Nodes Involved:**  
- General Chat Processing Agent (Langchain Agent)  
- Google Gemini Chat Model3 (AI Model)  

**Node Details:**  

- **General Chat Processing Agent**  
  - Type: Langchain Agent  
  - Role: Answers general queries concisely as PayInsighter bot, no Razorpay data involved.  
  - System Message: Conciseness and persona emphasized.  
  - Input: Telegram message text  
  - Connected to: Google Gemini Chat Model3  

---

### 2.7 Response Consolidation & Delivery

**Overview:**  
Merges outputs from all active branches preserving multi-intent answers and sends the final response back to the user via Telegram.

**Nodes Involved:**  
- Merge Responses (Merge node)  
- Send a text message (Telegram node)  

**Node Details:**  

- **Merge Responses**  
  - Type: Merge (version 3.2)  
  - Role: Combines outputs from Payments, Orders, Refunds, and General Chat processing agents.  
  - Parameters: Number of inputs set to 4, mode set to "append" to preserve all responses.  
  - Input: AI agent outputs from each branch  
  - Output: Combined response string  

- **Send a text message**  
  - Type: Telegram node  
  - Role: Sends the consolidated answer text back to the Telegram user.  
  - Parameters: Text set from merged output, chat ID from Telegram Trigger message metadata  
  - Edge cases: Telegram API failures, message size limits  

---

### 2.8 AI Model Provider Overview

**Overview:**  
Google Gemini Chat Models power all AI agents in the workflow, including intent classification, parameter parsing, data analysis, and response generation.

**Nodes Involved:**  
- Google Gemini Chat Model (orders branch)  
- Google Gemini Chat Model1 (payments branch)  
- Google Gemini Chat Model2 (refunds branch)  
- Google Gemini Chat Model3 (general chat branch)  

**Node Details:**  

- Each is a Langchain Google Gemini chat model node configured with default options.  
- Connected to respective Langchain agent nodes for AI tasks.  
- Common failure points: API key issues, model unavailability, rate limits.  

---

## 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                                    | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                     |
|----------------------------|-----------------------------------|---------------------------------------------------|----------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------|
| Telegram Trigger           | Telegram Trigger                  | Entry point; listens for Telegram messages         | -                          | Intent Classifier          | Entry Point: Telegram trigger captures user queries and metadata                                                |
| Intent Classifier           | Langchain Agent                  | Classifies user intent into categories             | Telegram Trigger           | Payment Branch, Refund Branch, Order Branch, General Chat Branch1 | Intent Classification: classifies into payments, orders, refunds, general_chat                                  |
| Payment Branch             | If Node (v2.2)                   | Routes if intent includes "payments"                | Intent Classifier          | Payments Processor         | Routing Logic: parallel IF nodes route to branches based on intent                                              |
| Refund Branch              | If Node (v2.2)                   | Routes if intent includes "refunds"                 | Intent Classifier          | Refunds                   | Routing Logic                                                                                                   |
| Order Branch               | If Node (v2.2)                   | Routes if intent includes "orders"                  | Intent Classifier          | Orders                    | Routing Logic                                                                                                   |
| General Chat Branch1       | If Node (v2.2)                   | Routes if intent includes "general_chat"            | Intent Classifier          | General Chat Processing Agent | Routing Logic                                                                                                   |
| Payments Processor         | Langchain Agent                  | Parses payment query parameters                      | Payment Branch             | Fetch all payments         | Payments Branch: AI agent parses user query for payment parameters                                             |
| Fetch all payments         | Razorpay API Node                | Fetches payments from Razorpay API                   | Payments Processor         | AI Agent3                 | Payments Branch: fetch Razorpay payments                                                                       |
| AI Agent3                 | Langchain Agent                  | Analyzes payments data and generates answer         | Fetch all payments         | Merge Responses            | Payments Branch: formats payment data into concise answer                                                      |
| Orders                    | Langchain Agent                  | Parses order query parameters                         | Order Branch               | Fetch all orders           | Orders Branch: AI agent parses user query for orders parameters                                                |
| Fetch all orders           | Razorpay API Node                | Fetches orders from Razorpay API                      | Orders                    | Order Processing Agent     | Orders Branch: fetch Razorpay orders                                                                           |
| Order Processing Agent     | Langchain Agent                  | Analyzes orders data and generates answer            | Fetch all orders           | Merge Responses            | Orders Branch: formats order data into concise answer                                                          |
| Refunds                   | Langchain Agent                  | Parses refund query parameters                        | Refund Branch              | Fetch all refunds          | Refunds Branch: AI agent parses user query for refund parameters                                               |
| Fetch all refunds          | Razorpay API Node                | Fetches refunds from Razorpay API                     | Refunds                   | Refund Processing Agent    | Refunds Branch: fetch Razorpay refunds                                                                          |
| Refund Processing Agent    | Langchain Agent                  | Analyzes refund data and generates answer             | Fetch all refunds          | Merge Responses            | Refunds Branch: formats refund data into concise answer                                                        |
| General Chat Processing Agent | Langchain Agent                | Handles general chat queries, generates answers      | General Chat Branch1       | Merge Responses            | General Chat Branch: handles greetings, help, general questions                                               |
| Merge Responses           | Merge (v3.2)                    | Consolidates all branch responses                      | AI Agent3, Order Processing Agent, Refund Processing Agent, General Chat Processing Agent | Send a text message       | Response Consolidation: merges multi-intent responses                                                          |
| Send a text message       | Telegram node                   | Sends final response to Telegram user                 | Merge Responses            | -                         | Response Consolidation: sends consolidated answer to user                                                      |
| Google Gemini Chat Model  | LM Chat Google Gemini            | AI model for orders branch agents                      | -                          | Orders, Order Processing Agent | AI Model Provider: powers all AI agents                                                                        |
| Google Gemini Chat Model1 | LM Chat Google Gemini            | AI model for payments branch agents                    | -                          | Payments Processor, AI Agent3 | AI Model Provider                                                                                                |
| Google Gemini Chat Model2 | LM Chat Google Gemini            | AI model for refunds branch agents                     | -                          | Refunds, Refund Processing Agent | AI Model Provider                                                                                                |
| Google Gemini Chat Model3 | LM Chat Google Gemini            | AI model for general chat branch                       | -                          | General Chat Processing Agent | AI Model Provider                                                                                                |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook with Telegram Bot API credentials  
   - Set to listen for update type "message"  
   - Output: user message text and metadata  

2. **Add Google Gemini Chat Model Node (Intent Classification)**  
   - Type: Langchain LM Chat Google Gemini  
   - Use model "gemini-2.5-flash" or similar  
   - Default options  

3. **Add Intent Classifier Node (Langchain Agent)**  
   - Input: `{{$json.message.text}}` from Telegram Trigger  
   - Configure system message:  
     ```
     Give a single word answer
     classify the user input message in the following categories
     do not add any formatting or special characters like "\n"
     If the users message has multiple intents then give output as intent1_intent2 separated by underscore (_)
     Eg. orders_payments
     1. orders
     2. payments
     3. refunds
     4. general_chat
     ```  
   - Connect AI model: Google Gemini Chat Model node  
   - Output: intent string  

4. **Add Four IF Nodes for Routing**  
   - Payment Branch: check if intent contains "payments"  
   - Refund Branch: check if intent contains "refunds"  
   - Order Branch: check if intent contains "orders"  
   - General Chat Branch: check if intent contains "general_chat"  

5. **Payments Branch Setup:**  
   a. Add Google Gemini Chat Model node for payments (Google Gemini Chat Model1)  
   b. Add Payments Processor (Langchain Agent)  
      - Input: Telegram message text  
      - System message: instruct extraction of JSON with from/to/skip/count with date format (e.g., "2025-11-07 00:00:00")  
   c. Add Razorpay "Fetch all payments" node  
      - Resource: payment  
      - Operation: fetchAllPayments  
      - Parameters: extract from/to/skip/count from Payments Processor output JSON  
      - Use Razorpay credentials  
   d. Add AI Agent3 (Langchain Agent)  
      - Input: payment data and user query text  
      - System message: concise answer, no data flooding, persona PayInsighter  
      - Connect Google Gemini Chat Model1  

6. **Orders Branch Setup:**  
   a. Add Google Gemini Chat Model node for orders (Google Gemini Chat Model)  
   b. Add Orders (Langchain Agent)  
      - Parse user query to JSON with from/to/skip/count parameters (date format)  
   c. Add Razorpay "Fetch all orders" node  
      - Resource: order  
      - Operation: fetchAllOrders  
      - Use parameters from Orders node output  
   d. Add Order Processing Agent (Langchain Agent)  
      - Input: order data and user query text  
      - System message: concise answer, PayInsighter persona  
      - Connect Google Gemini Chat Model  

7. **Refund Branch Setup:**  
   a. Add Google Gemini Chat Model node for refunds (Google Gemini Chat Model2)  
   b. Add Refunds (Langchain Agent)  
      - Parse user query to JSON with from/to/skip/count parameters in epoch timestamps  
   c. Add Razorpay "Fetch all refunds" node  
      - Resource: refund  
      - Use parameters from Refunds node output  
   d. Add Refund Processing Agent (Langchain Agent)  
      - Input: refund data and user query text  
      - System message: concise answer, PayInsighter persona  
      - Connect Google Gemini Chat Model2  

8. **General Chat Branch Setup:**  
   a. Add Google Gemini Chat Model node for general chat (Google Gemini Chat Model3)  
   b. Add General Chat Processing Agent (Langchain Agent)  
      - Input: Telegram message text  
      - System message: concise answer, PayInsighter persona  
      - Connect Google Gemini Chat Model3  

9. **Add Merge Node**  
   - Type: Merge (v3.2)  
   - Number of inputs: 4 (from AI Agent3, Order Processing Agent, Refund Processing Agent, General Chat Processing Agent)  
   - Mode: Append (preserves all responses)  

10. **Add Telegram Send Message Node**  
    - Input text: merged output from Merge node  
    - Chat ID: from Telegram Trigger message metadata (`{{$node["Telegram Trigger"].item.json.message.from.id}}`)  
    - Configure Telegram credentials  

11. **Connect Nodes**  
    - Telegram Trigger → Intent Classifier  
    - Intent Classifier → 4 IF nodes (Payment Branch, Refund Branch, Order Branch, General Chat Branch)  
    - Each IF node → respective AI processor branch as detailed above  
    - AI agents outputs → Merge node  
    - Merge node → Telegram send message node  

12. **Credential Setup:**  
    - Configure Telegram bot credentials for Trigger and Send message nodes  
    - Configure Razorpay API credentials for Razorpay nodes  
    - Configure Google Gemini API credentials for Langchain Google Gemini nodes  

13. **Test and Validate:**  
    - Test with sample Telegram messages covering all intents and multi-intent queries  
    - Validate AI outputs, API calls, and responses  
    - Handle errors gracefully (e.g., fallback messages on AI or API failure)  

---

## 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| The bot is named **PayInsighter**, designed to maintain a consistent persona in all AI-generated responses.                       | Branding                                                                                                                            |
| Uses Google Gemini models (e.g., "gemini-2.5-flash") for all AI tasks including intent classification and response generation.    | AI Model Provider                                                                                                                   |
| Multi-intent support allows users to combine queries like "orders_payments" for simultaneous analytics.                            | Workflow feature                                                                                                                    |
| Maximum count parameter for API calls is capped at 100 to avoid overloading API or response.                                        | Razorpay API constraint                                                                                                            |
| Razorpay API parameters are parsed strictly from AI JSON outputs to enable dynamic queries.                                         | Integration detail                                                                                                                  |
| Telegram message size limits and API rate limits should be monitored to avoid delivery failures.                                   | Operational edge case                                                                                                              |
| This workflow requires valid API credentials for Razorpay, Telegram, and Google Gemini configured in n8n credentials manager.     | Credential requirement                                                                                                             |
| Sticky notes in the workflow provide detailed explanations of each logical block and node roles for clarity.                       | Workflow documentation embedded in n8n                                                                                             |
| Ensure timezone consistency in date parameters when parsing user queries for date ranges.                                          | Potential source of errors                                                                                                         |
| Google Gemini API quota or downtime may affect responsiveness; implement error handling or fallback messages.                     | Operational risk                                                                                                                   |

---

**Disclaimer:** The provided documentation is based exclusively on an automated n8n workflow and adheres to all current content policies. It contains no illegal or protected content. All data handled is public and lawful.

---