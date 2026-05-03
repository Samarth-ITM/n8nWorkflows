WhatsApp Recipe Suggestions from Pantry Items with Gemini AI & FatSecret API

https://n8nworkflows.xyz/workflows/whatsapp-recipe-suggestions-from-pantry-items-with-gemini-ai---fatsecret-api-7919


# WhatsApp Recipe Suggestions from Pantry Items with Gemini AI & FatSecret API

### 1. Workflow Overview

This workflow enables a WhatsApp chatbot that suggests recipes based on a user's pantry items and preferences, leveraging Google’s Gemini AI for natural language understanding and the FatSecret API for recipe data retrieval. The main use case is a conversational cooking assistant where users text ingredients or dietary needs via WhatsApp and receive tailored recipe suggestions.

The workflow logic is organized into these blocks:

- **1.1 Input Reception:** Captures incoming WhatsApp messages.
- **1.2 AI Processing & Memory:** Uses Google Gemini AI and a memory buffer to interpret user messages, maintain context, and convert free text into structured recipe search parameters.
- **1.3 Recipe Search Tool Invocation:** Calls the FatSecret Recipes Search API with AI-generated parameters.
- **1.4 Response Composition & Delivery:** Formats AI responses and recipe results, then sends them back to the user on WhatsApp.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

**Overview:**  
This block captures incoming WhatsApp messages in real time via a webhook, triggering the workflow to process user queries.

**Nodes Involved:**  
- WhatsApp Trigger

**Node Details:**

- **WhatsApp Trigger**  
  - **Type & Role:** Captures WhatsApp message events via webhook.  
  - **Configuration:** Listens specifically for "messages" updates; uses WhatsApp Trigger API credentials linked to the chatbot.  
  - **Expressions/Variables:** Extracts sender phone number and message text from incoming JSON at `messages[0].from` and `messages[0].text.body`.  
  - **Connections:** Outputs to "The Chef Agent" node for AI processing.  
  - **Edge Cases:** Webhook connectivity issues, invalid or unsupported message formats, or API authentication failures.  
  - **Version:** 1

---

#### Block 1.2: AI Processing & Memory

**Overview:**  
Processes the user’s message using Google Gemini AI to extract structured recipe search parameters, while maintaining conversational context using a memory buffer.

**Nodes Involved:**  
- The Chef Agent  
- Google Gemini Chat Model1  
- Simple Memory

**Node Details:**

- **The Chef Agent**  
  - **Type & Role:** LangChain agent node acting as the core AI processor and orchestrator.  
  - **Configuration:**  
    - Input text is the user's WhatsApp message body.  
    - System prompt defines roles and instructions for PantryChef, including how to parse user input into FatSecret API parameters (via `$fromAI` keys).  
    - Handles multiple intents: ingredient extraction, dietary restrictions, calorie limits, cuisine preferences, paging, etc.  
    - Outputs either AI-generated parameters to call the FatSecret API or a user prompt if clarification is needed.  
  - **Expressions/Variables:** Uses `$json.messages[0].text.body` for input text; emits `$fromAI` keys for parameters like `search_expression`, `cal_max`, `page_number`.  
  - **Input Connections:** From WhatsApp Trigger; AI language model and HTTP tool integrations via LangChain AI plugin connections.  
  - **Output Connections:** Main output to "Send message" (WhatsApp response); AI tool output to "Fatsecret_Recipes" HTTP request; AI memory output to "Simple Memory".  
  - **Edge Cases:** Misinterpretation of user input, failure to extract parameters, API rate limits, or timeout in AI response.  
  - **Version:** 2.2

- **Google Gemini Chat Model1**  
  - **Type & Role:** Google Gemini AI language model node used by the agent for natural language understanding and generation.  
  - **Configuration:** Uses Google PaLM API credentials. No additional options configured explicitly.  
  - **Connections:** Provides language model integration to "The Chef Agent".  
  - **Edge Cases:** API quota exhaustion, authentication failure, or latency causing delays.  
  - **Version:** 1

- **Simple Memory**  
  - **Type & Role:** LangChain memory node storing recent conversational context to maintain state across messages.  
  - **Configuration:** Uses a windowed memory buffer keyed by sender phone number (`sessionKey` from WhatsApp sender); keeps last 5 messages for context.  
  - **Connections:** Memory input/output linked with "The Chef Agent" for conversational continuity.  
  - **Edge Cases:** Memory overflow if many users simultaneously; session key extraction failures if incoming message format changes.  
  - **Version:** 1.3

---

#### Block 1.3: Recipe Search Tool Invocation

**Overview:**  
Executes the API call to FatSecret’s recipe search endpoint using parameters extracted by the AI agent, to retrieve matching recipes.

**Nodes Involved:**  
- Fatsecret_Recipes

**Node Details:**

- **Fatsecret_Recipes**  
  - **Type & Role:** HTTP Request node calling FatSecret’s v3 recipe search API.  
  - **Configuration:**  
    - Method: GET with query parameters.  
    - Query parameters include `search_expression`, `format=json`, `max_results=5`, and optional calorie max (`calories.to`).  
    - OAuth2 credentials configured for FatSecret API authentication.  
  - **Expressions/Variables:** Values for `search_expression` and `calories.to` are dynamically injected from AI agent outputs via `$fromAI` overrides.  
  - **Connections:** Invoked as AI tool call from "The Chef Agent".  
  - **Edge Cases:** OAuth token expiration or failure, network timeouts, invalid parameter values causing API errors, rate limiting by FatSecret.  
  - **Version:** 4.2

---

#### Block 1.4: Response Composition & Delivery

**Overview:**  
Formats the AI-generated recipe suggestions or clarification prompts and sends them back to the user on WhatsApp.

**Nodes Involved:**  
- Send message

**Node Details:**

- **Send message**  
  - **Type & Role:** WhatsApp node sending outbound messages to users.  
  - **Configuration:**  
    - Sends text taken from `$json.output` generated by "The Chef Agent".  
    - Uses WhatsApp API credentials for message delivery.  
    - Recipient phone number dynamically set to the sender's number extracted from the incoming message.  
  - **Connections:** Connected from "The Chef Agent" main output node.  
  - **Edge Cases:** Failures in WhatsApp API connectivity, invalid recipient number, message formatting issues, or rate limiting by WhatsApp.  
  - **Version:** 1

---

### 3. Summary Table

| Node Name            | Node Type                                   | Functional Role                       | Input Node(s)       | Output Node(s)       | Sticky Note                        |
|----------------------|---------------------------------------------|------------------------------------|---------------------|----------------------|----------------------------------|
| WhatsApp Trigger     | WhatsApp Trigger                            | Receive user messages               |                     | The Chef Agent       |                                  |
| The Chef Agent       | LangChain Agent                             | AI processing, parameter extraction | WhatsApp Trigger    | Send message, Fatsecret_Recipes, Simple Memory |                                  |
| Google Gemini Chat Model1 | LangChain Google Gemini AI Model          | Language model for AI processing    |                     | The Chef Agent       |                                  |
| Simple Memory        | LangChain Memory Buffer (Window)            | Maintain context memory             | The Chef Agent (ai_memory) | The Chef Agent (ai_memory) |                                  |
| Fatsecret_Recipes    | HTTP Request Tool                           | Call FatSecret Recipe Search API   | The Chef Agent (ai_tool) | The Chef Agent (ai_tool) |                                  |
| Send message         | WhatsApp Node                              | Send response messages to users    | The Chef Agent       |                      |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node:**  
   - Type: WhatsApp Trigger  
   - Configure webhook with your WhatsApp Business API credentials.  
   - Enable listening for "messages" updates.  
   - No special parameters.  
   - Connect output to The Chef Agent node.

2. **Create The Chef Agent Node:**  
   - Type: LangChain Agent  
   - Set input text to `{{$json.messages[0].text.body}}` (extract WhatsApp user message).  
   - Paste the provided system prompt defining PantryChef’s role, input parsing rules, output contract, and examples.  
   - Under "AI Language Model," link to "Google Gemini Chat Model1".  
   - Under "AI Tool," link to "Fatsecret_Recipes" HTTP request node.  
   - Under "AI Memory," link to "Simple Memory" node.  
   - Version 2.2 or latest recommended.  
   - Connect main output to "Send message".

3. **Create Google Gemini Chat Model1 Node:**  
   - Type: LangChain Google Gemini AI Model  
   - Provide Google PaLM API credentials with required scopes and billing enabled.  
   - No additional options required.  
   - Connect output to "The Chef Agent" AI language model input.

4. **Create Simple Memory Node:**  
   - Type: LangChain Memory Buffer Window  
   - Set `sessionKey` to `={{ $('WhatsApp Trigger').item.json.messages[0].from }}` to key memory by sender.  
   - Set `contextWindowLength` to 5 messages.  
   - Connect memory input and output to "The Chef Agent" AI memory interface.

5. **Create Fatsecret_Recipes HTTP Request Node:**  
   - Type: HTTP Request  
   - Set HTTP method to GET.  
   - URL: `https://platform.fatsecret.com/rest/recipes/search/v3`  
   - Add Query Parameters:  
     - `search_expression` from AI output `$fromAI('parameters0_Value')`  
     - `format` = `json` (fixed)  
     - `max_results` = `5` (fixed)  
     - `calories.to` from AI output `$fromAI('parameters3_Value')` (optional)  
   - Authentication: OAuth2 using FatSecret API credentials.  
   - Connect as AI tool input to "The Chef Agent".

6. **Create Send message Node:**  
   - Type: WhatsApp Node  
   - Configure WhatsApp API credentials for sending messages.  
   - Set `textBody` to `={{ $json.output }}` (AI-generated reply).  
   - Set `recipientPhoneNumber` to `={{ $('WhatsApp Trigger').item.json.messages[0].from }}`.  
   - Connect main input from "The Chef Agent".

7. **Workflow Connections:**  
   - WhatsApp Trigger → The Chef Agent  
   - Google Gemini Chat Model1 → The Chef Agent (AI language model)  
   - Simple Memory → The Chef Agent (AI memory)  
   - Fatsecret_Recipes → The Chef Agent (AI tool)  
   - The Chef Agent → Send message (main output)

8. **Credentials Setup:**  
   - Configure WhatsApp Trigger and Send message nodes with WhatsApp Business API OAuth2 credentials.  
   - Configure Google Gemini Chat Model1 with Google PaLM API credentials.  
   - Configure Fatsecret_Recipes with OAuth2 credentials for FatSecret API.  
   - Ensure all tokens are valid and permissions enabled.

9. **Test Workflow:**  
   - Send test WhatsApp messages with ingredient lists or recipe requests to trigger the flow.  
   - Verify AI extracts parameters, calls FatSecret API, and replies appropriately.  
   - Adjust system prompt or query parameters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The system prompt in "The Chef Agent" node is central to parsing user messages into precise API calls; tweaking it enables refinement of AI behavior and response style.                                                                       | N/A                                                 |
| Google Gemini (PaLM) API requires setup in Google Cloud Console with appropriate billing and API access enabled for n8n integration.                                                                                                         | https://cloud.google.com/palm                         |
| FatSecret API v3 requires OAuth2 credentials and proper scopes for recipe search; refer to FatSecret developer docs for client registration and token management.                                                                            | https://platform.fatsecret.com/                      |
| WhatsApp Business API credentials must be configured with valid webhook URLs and phone numbers; this workflow assumes a single phone number ID configured for sending messages.                                                                | https://developers.facebook.com/docs/whatsapp       |
| For best conversational experience, keep memory window short (here 5 messages) to balance context and performance.                                                                                                                           | N/A                                                 |
| The workflow is designed for English language input; adapting for other languages requires modifying the system prompt and potentially AI model settings.                                                                                   | N/A                                                 |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.