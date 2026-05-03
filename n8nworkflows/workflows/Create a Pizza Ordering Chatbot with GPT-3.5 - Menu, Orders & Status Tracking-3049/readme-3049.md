Create a Pizza Ordering Chatbot with GPT-3.5 - Menu, Orders & Status Tracking

https://n8nworkflows.xyz/workflows/create-a-pizza-ordering-chatbot-with-gpt-3-5---menu--orders---status-tracking-3049


# Create a Pizza Ordering Chatbot with GPT-3.5 - Menu, Orders & Status Tracking

### 1. Workflow Overview

This workflow implements a **Pizza Ordering Chatbot** using n8n and OpenAI’s GPT-3.5 model. It automates customer interactions for a pizza store, handling menu inquiries, order placement, and order status tracking through conversational AI.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages from customers.
- **1.2 AI Agent Processing:** Uses an AI agent to interpret customer intents and route requests.
- **1.3 Memory Management:** Maintains conversational context using window buffer memory.
- **1.4 AI Language Model Interaction:** Generates AI responses via OpenAI GPT-3.5.
- **1.5 Tool Integrations:** Executes specific tasks such as fetching menu data, placing orders, retrieving order status, and performing calculations through HTTP requests or calculator tools.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives chat messages from customers via a webhook-based chat trigger node, initiating the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* Chat Trigger (Webhook)  
    - *Role:* Entry point for customer chat messages.  
    - *Configuration:*  
      - Public webhook enabled for external access.  
      - Initial greeting message set to introduce the chatbot “Pizzaro” and provide ordering instructions.  
    - *Inputs:* External HTTP chat messages.  
    - *Outputs:* Passes chat input text to the AI Agent node.  
    - *Edge Cases:*  
      - Webhook downtime or misconfiguration may cause message loss.  
      - Malformed or empty chat messages may require validation downstream.

#### 2.2 AI Agent Processing

- **Overview:**  
  This block interprets the customer’s chat input, determines intent (menu inquiry, order placement, or status tracking), and orchestrates calls to appropriate tools or AI models.

- **Nodes Involved:**  
  - AI Agent

- **Node Details:**  
  - **AI Agent**  
    - *Type:* LangChain Agent Node  
    - *Role:* Central decision-making node that processes chat input and selects tools to fulfill requests.  
    - *Configuration:*  
      - Receives raw chat input text.  
      - System message defines the chatbot persona “Pizzaro” and outlines three main functions: menu info, order placement, and order status.  
      - Prompt type set to “define” to customize agent behavior.  
      - Executes continuously for each incoming message.  
    - *Inputs:* Chat input from “When chat message received” node.  
    - *Outputs:* Routes requests to AI language model, memory, calculator, or HTTP request tools based on intent.  
    - *Edge Cases:*  
      - Ambiguous or complex queries may confuse intent classification.  
      - Unexpected input formats could cause failures.  
      - Requires robust error handling for downstream tool failures.

#### 2.3 Memory Management

- **Overview:**  
  Maintains conversational context to enable coherent multi-turn dialogue.

- **Nodes Involved:**  
  - Window Buffer Memory

- **Node Details:**  
  - **Window Buffer Memory**  
    - *Type:* LangChain Memory Node  
    - *Role:* Stores recent conversation history in a sliding window buffer.  
    - *Configuration:* Default window size (not explicitly set) to keep recent messages.  
    - *Inputs:* Connected as AI memory for the AI Agent node.  
    - *Outputs:* Provides context to the AI Agent for better response generation.  
    - *Edge Cases:*  
      - Memory overflow if window size is too large.  
      - Loss of context if window size too small.

#### 2.4 AI Language Model Interaction

- **Overview:**  
  Generates natural language responses using OpenAI’s GPT-3.5 model based on processed input and context.

- **Nodes Involved:**  
  - Chat OpenAI

- **Node Details:**  
  - **Chat OpenAI**  
    - *Type:* LangChain OpenAI Chat Model Node  
    - *Role:* Produces AI-generated text responses.  
    - *Configuration:*  
      - Uses OpenAI credentials (API key).  
      - Model: gpt-3.5-turbo (implied from description).  
      - Default options (temperature, max tokens) not explicitly set here but recommended as 0.7 temperature and 150 max tokens.  
    - *Inputs:* Receives prompts from AI Agent node.  
    - *Outputs:* Returns AI-generated chat responses.  
    - *Edge Cases:*  
      - API key invalid or quota exceeded causes authentication errors.  
      - Network timeouts or rate limits may cause failures.  
      - Unexpected prompt formatting may degrade response quality.

#### 2.5 Tool Integrations

- **Overview:**  
  Executes specific tasks related to menu retrieval, order processing, order status checking, and calculations via HTTP requests or calculator tools.

- **Nodes Involved:**  
  - Get Products  
  - Order Product  
  - Get Order  
  - Calculator

- **Node Details:**  
  - **Get Products**  
    - *Type:* HTTP Request Tool Node  
    - *Role:* Fetches detailed pizza menu data from an external webhook endpoint.  
    - *Configuration:*  
      - URL: `https://n8n.io/webhook/get-products`  
      - Method: GET (default)  
      - Tool description clarifies purpose.  
    - *Inputs:* Called by AI Agent when menu info requested.  
    - *Outputs:* Returns menu details to AI Agent.  
    - *Edge Cases:*  
      - Endpoint downtime or incorrect URL leads to request failure.  
      - Unexpected response format may cause parsing errors.

  - **Order Product**  
    - *Type:* HTTP Request Tool Node  
    - *Role:* Processes customer orders by sending order details to an external webhook.  
    - *Configuration:*  
      - URL: `https://n8n.io/webhook/order-product`  
      - Method: POST  
      - Sends body parameter “message” containing the chat input text.  
      - Tool description clarifies purpose.  
    - *Inputs:* Called by AI Agent when order placement detected.  
    - *Outputs:* Returns order confirmation or processing status.  
    - *Edge Cases:*  
      - Network or server errors during POST.  
      - Invalid order data or missing fields may cause rejection.

  - **Get Order**  
    - *Type:* HTTP Request Tool Node  
    - *Role:* Retrieves order status information based on order ID or customer query.  
    - *Configuration:*  
      - URL: `https://n8n.io/webhook/get-orders`  
      - Method: GET (default)  
      - Tool description clarifies purpose.  
    - *Inputs:* Called by AI Agent when order status inquiry detected.  
    - *Outputs:* Returns order details including date, pizza type, quantity.  
    - *Edge Cases:*  
      - Order ID not found or invalid leads to empty or error responses.  
      - Endpoint unavailability or timeout.

  - **Calculator**  
    - *Type:* LangChain Calculator Tool Node  
    - *Role:* Performs any required calculations (e.g., price totals, quantities).  
    - *Configuration:* Default, no parameters set explicitly.  
    - *Inputs:* Called by AI Agent as needed.  
    - *Outputs:* Returns calculation results.  
    - *Edge Cases:*  
      - Invalid mathematical expressions may cause errors.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                         | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                          |
|-------------------------|----------------------------------|---------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger (Webhook)            | Entry point for customer chat messages | External HTTP chat messages  | AI Agent                   | Initial greeting message sets chatbot intro and ordering instructions.                             |
| AI Agent                | LangChain Agent Node             | Processes chat input, routes intents  | When chat message received   | Chat OpenAI, Calculator, Get Products, Order Product, Get Order, Window Buffer Memory | Defines chatbot persona “Pizzaro” and main functions: menu, order, status tracking.                |
| Window Buffer Memory    | LangChain Memory Node            | Maintains conversation context        | AI Agent                    | AI Agent                   | Stores recent conversation history for coherent dialogue.                                         |
| Chat OpenAI             | LangChain OpenAI Chat Model Node | Generates AI text responses            | AI Agent                    | AI Agent                   | Uses OpenAI GPT-3.5 with recommended temperature 0.7 and max tokens 150.                            |
| Get Products            | HTTP Request Tool Node           | Fetches pizza menu data                | AI Agent                    | AI Agent                   | Retrieves detailed product menu from external webhook.                                            |
| Order Product           | HTTP Request Tool Node           | Processes pizza orders                 | AI Agent                    | AI Agent                   | Sends order details to external webhook for processing.                                           |
| Get Order               | HTTP Request Tool Node           | Retrieves order status                 | AI Agent                    | AI Agent                   | Gets order status details from external webhook.                                                  |
| Calculator              | LangChain Calculator Tool Node   | Performs calculations                  | AI Agent                    | AI Agent                   | Used for price or quantity calculations as needed.                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add the “When chat message received” node**:  
   - Type: Chat Trigger (Webhook)  
   - Set webhook to public.  
   - Configure initial messages:  
     ```
     Hellooo! 👋 My name is Pizzaro 🍕. I'm here to help with your pizza order. How can I assist you?

     📣 INFO: If you’d like to order a pizza, please include your name + pizza type + quantity. Thank you!
     ```  
   - This node will receive incoming chat messages.

3. **Add the “AI Agent” node**:  
   - Type: LangChain Agent Node  
   - Connect input from “When chat message received” node.  
   - Parameters:  
     - Text input: `={{ $json.chatInput }}`  
     - Prompt type: Define  
     - System message:  
       ```
       Your name is Pizzaro, and you are an assistant for handling customer pizza orders.

       1. If a customer asks about the menu, provide information on the available products.
       2. If a customer is placing an order, confirm the order details, inform them that the order is being processed, and thank them.
       3. If a customer inquires about their order status, provide the order date, pizza type, and quantity.
       ```  
   - Set to execute on every new message.

4. **Add “Window Buffer Memory” node**:  
   - Type: LangChain Memory Buffer Window  
   - Connect as AI memory input to the AI Agent node.  
   - Use default settings for conversation context management.

5. **Add “Chat OpenAI” node**:  
   - Type: LangChain OpenAI Chat Model Node  
   - Connect as AI language model input to AI Agent node.  
   - Credentials: Create new OpenAI API credentials with your API key.  
   - Recommended parameters (if configurable):  
     - Model: `gpt-3.5-turbo`  
     - Temperature: `0.7`  
     - Max tokens: `150`

6. **Add “Get Products” node**:  
   - Type: LangChain HTTP Request Tool Node  
   - Connect as AI tool input to AI Agent node.  
   - Configure URL: `https://n8n.io/webhook/get-products`  
   - Method: GET (default)  
   - Description: Retrieve detailed pizza menu data.

7. **Add “Order Product” node**:  
   - Type: LangChain HTTP Request Tool Node  
   - Connect as AI tool input to AI Agent node.  
   - Configure URL: `https://n8n.io/webhook/order-product`  
   - Method: POST  
   - Enable sending body with parameter:  
     - Name: `message`  
     - Value: `={{ $json.chatInput }}`  
   - Description: Process pizza orders.

8. **Add “Get Order” node**:  
   - Type: LangChain HTTP Request Tool Node  
   - Connect as AI tool input to AI Agent node.  
   - Configure URL: `https://n8n.io/webhook/get-orders`  
   - Method: GET (default)  
   - Description: Retrieve order status information.

9. **Add “Calculator” node**:  
   - Type: LangChain Calculator Tool Node  
   - Connect as AI tool input to AI Agent node.  
   - No additional configuration needed unless specific calculations are required.

10. **Connect all nodes properly:**  
    - “When chat message received” → “AI Agent”  
    - “AI Agent” → “Chat OpenAI” (ai_languageModel)  
    - “AI Agent” → “Window Buffer Memory” (ai_memory)  
    - “AI Agent” → “Get Products” (ai_tool)  
    - “AI Agent” → “Order Product” (ai_tool)  
    - “AI Agent” → “Get Order” (ai_tool)  
    - “AI Agent” → “Calculator” (ai_tool)

11. **Set credentials:**  
    - Create OpenAI API credentials with your API key and assign to “Chat OpenAI” node.

12. **Test the workflow:**  
    - Execute workflow.  
    - Send chat messages like “What pizzas do you have?”, “I want to order a Margherita pizza”, or “What is the status of my order #123?”.  
    - Verify responses and adjust prompts or node parameters as needed.

13. **Deploy and integrate:**  
    - Deploy the workflow on your n8n instance.  
    - Integrate with messaging platforms (Telegram, WhatsApp, website chat) by connecting their webhook or API to the “When chat message received” node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| OpenAI account and API key are required to use GPT-3.5 capabilities.                                       | https://platform.openai.com/signup/                                                             |
| n8n installation guide for setting up your automation environment.                                        | https://docs.n8n.io/getting-started/installation/                                               |
| Workflow enables a conversational pizza ordering experience with menu info, order placement, and tracking. |                                                                                                 |
| Initial chatbot greeting encourages customers to include name, pizza type, and quantity when ordering.    |                                                                                                 |
| Recommended OpenAI parameters: temperature 0.7 for creativity, max tokens 150 to limit response length.   |                                                                                                 |
| External webhook URLs (`https://n8n.io/webhook/...`) are placeholders and should be replaced with actual endpoints. |                                                                                                 |
| For advanced customization, prompts and system messages can be refined to improve chatbot accuracy.      |                                                                                                 |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the pizza ordering chatbot workflow using n8n and OpenAI GPT-3.5.