Handle WhatsApp Customer Inquiries with AI and Intent Routing

https://n8nworkflows.xyz/workflows/handle-whatsapp-customer-inquiries-with-ai-and-intent-routing-10240


# Handle WhatsApp Customer Inquiries with AI and Intent Routing

---

### 1. Workflow Overview

This workflow is a comprehensive WhatsApp Customer Support Bot designed to automate handling customer inquiries via WhatsApp Business API. It targets businesses across various industries such as fashion, electronics, food, furniture, and beauty. The workflow’s goal is to receive customer messages, classify the user intent, and intelligently route queries to the appropriate response mechanism—either pre-built static replies or a sophisticated AI agent for complex interactions. It supports context-aware conversations with a memory buffer, enabling personalized and professional engagement.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception**: Receiving and parsing incoming WhatsApp messages.
- **1.2 Intent Classification**: Analyzing user messages to determine the customer’s intent using keyword and pattern matching.
- **1.3 Intelligent Routing**: Routing messages based on detected intent to appropriate response generators.
- **1.4 Pre-built Response Generation**: Providing fast, cost-effective answers using static or catalog-based responses.
- **1.5 AI Agent Processing**: Handling complex queries via an AI model with conversational memory and rich company context.
- **1.6 Message Sending**: Formatting and sending the final response back to the customer on WhatsApp.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow on new WhatsApp messages, extracts and structures relevant data for downstream processing.

**Nodes Involved:**  
- WhatsApp Trigger - Receive Messages  
- Parse WhatsApp Message Data

**Node Details:**

- **WhatsApp Trigger - Receive Messages**  
  - Type: WhatsApp Trigger  
  - Role: Listens for incoming WhatsApp messages using WhatsApp Business API credentials (OAuth).  
  - Configuration: Subscribed to "messages" updates only.  
  - Inputs: External webhook from WhatsApp Business API.  
  - Outputs: Raw message JSON forwarded downstream.  
  - Requirements: Valid WhatsApp OAuth credentials, webhook endpoint configured and publicly accessible.  
  - Potential Failures: Authentication errors, webhook misconfiguration, message format changes.

- **Parse WhatsApp Message Data**  
  - Type: Code Node  
  - Role: Extracts key fields such as Chat ID, User ID, User Name, Message Text, Message ID, and Timestamp.  
  - Configuration: JavaScript code lowers message text for intent matching while preserving original text for AI use.  
  - Inputs: Raw WhatsApp message JSON from trigger node.  
  - Outputs: Structured session data JSON for intent classification.  
  - Edge Cases: Non-text messages currently unsupported; media handling would require expansion. Missing user name fields default to “Customer”.

---

#### 2.2 Intent Classification

**Overview:**  
Determines the user's intent from their message via pattern matching and keyword detection, enabling targeted routing.

**Nodes Involved:**  
- Classify User Intent

**Node Details:**

- **Classify User Intent**  
  - Type: Code Node  
  - Role: Implements hierarchical intent classification using regex patterns for contact info, greetings, thanks, product categories, order tracking, price queries, shipping, returns, and support requests.  
  - Configuration: Customizable category patterns to fit different business types; defaults to "general_inquiry" if no match.  
  - Inputs: Structured message data from Parse WhatsApp Message Data node.  
  - Outputs: JSON object extended with detected `intent` and `entities` such as product category.  
  - Edge Cases: Ambiguous or multi-intent messages are classified by priority order; complex NLP is not used here to ensure speed. Misclassification possible with overlapping keywords.  
  - Customization: Businesses must adjust category patterns and keywords for accurate classification.

---

#### 2.3 Intelligent Routing

**Overview:**  
Routes messages to different response handlers based on the classified intent, supporting hybrid handling of queries.

**Nodes Involved:**  
- Route by Intent

**Node Details:**

- **Route by Intent**  
  - Type: Switch Node  
  - Role: Routes flow into one of three outputs: product inquiries, contact info requests, or a default “extra” route for other intents.  
  - Configuration: Matches on `intent` field values like "product_inquiry" and "contact_info".  
  - Inputs: Intent-classified JSON.  
  - Outputs:  
    - Output 0: product_inquiry → Generate Product Response  
    - Output 1: contact_info → Generate Contact Info Response  
    - Fallback output: Generate Default Response (for all other intents)  
  - Edge Cases: Unknown or new intents fall back to default response; if classification misses, user receives generic info.

---

#### 2.4 Pre-built Response Generation

**Overview:**  
Generates fast, cost-effective replies using hardcoded templates and product catalogs based on intent.

**Nodes Involved:**  
- Generate Product Response  
- Generate Contact Info Response  
- Generate Default Response

**Node Details:**

- **Generate Product Response**  
  - Type: Code Node  
  - Role: Returns product catalog responses tailored to detected product category with markdown formatting, prices, availability, and links.  
  - Configuration: Multiple example categories (electronics, fashion, furniture, skincare, groceries) with sample products and URLs; customizable per business.  
  - Inputs: JSON with `entities.category` and user context.  
  - Outputs: Response text with chatId and formatting instructions.  
  - Edge Cases: If no category matches, provides a general category menu with call-to-action.  
  - Customization: Businesses must update product lists, prices, URLs, and contact information.

- **Generate Contact Info Response**  
  - Type: Code Node  
  - Role: Provides static company contact details, address, phone numbers, emails, website, and store hours.  
  - Configuration: Markdown formatted, personalized with user name.  
  - Inputs: JSON with user context.  
  - Outputs: Response text with chatId and formatting instructions.  
  - Customization: Must replace placeholders with actual company info.

- **Generate Default Response**  
  - Type: Code Node  
  - Role: Fallback menu offering general navigation instructions, categories, order info, pricing, and contact details.  
  - Configuration: Customizable response listing business features and prompts for common questions.  
  - Inputs: JSON with user context.  
  - Outputs: Response text with chatId and formatting instructions.  
  - Edge Cases: Triggered for miscellaneous or unrecognized intents.

---

#### 2.5 AI Agent Processing

**Overview:**  
Handles complex or unclassified customer queries by invoking Google Gemini AI with rich system prompts and conversational memory.

**Nodes Involved:**  
- Build AI System Prompt  
- AI Agent - Handle Complex Queries  
- Google Gemini Chat Model  
- Conversation Memory  
- Format AI Response  
- Google Docs - Product Catalog (Optional)

**Node Details:**

- **Build AI System Prompt**  
  - Type: Code Node  
  - Role: Constructs a detailed system prompt embedding company knowledge, product categories, policies, and customer context (name, intent, original query).  
  - Configuration: Placeholder text for business info and instructions for AI response style with emojis, markdown, and call-to-action.  
  - Inputs: JSON from pre-built response nodes or routing fallback.  
  - Outputs: JSON including systemPrompt and userPrompt for AI agent.  
  - Customization: Critical to update with company-specific data for accurate AI answers.

- **AI Agent - Handle Complex Queries**  
  - Type: LangChain Agent Node  
  - Role: Processes userPrompt with systemPrompt using LangChain framework, managing logic to generate AI responses.  
  - Configuration: Uses Google Gemini Chat Model as language model backend, with conversation memory buffer keyed by userId.  
  - Inputs: userPrompt and systemPrompt from Build AI System Prompt node.  
  - Outputs: Raw AI text output.  
  - Requirements: Google Gemini API credentials, LangChain integration enabled.  
  - Edge Cases: API timeouts, quota exhaustion, context length limits.

- **Google Gemini Chat Model**  
  - Type: LangChain Language Model Node  
  - Role: Connects to Google Gemini (PaLM) API for chat completions.  
  - Configuration: Uses API key credentials.  
  - Inputs: Prompts from AI Agent node.  
  - Outputs: AI-generated text.  
  - Requirements: Valid Google Gemini API key.

- **Conversation Memory**  
  - Type: LangChain Memory Buffer Window Node  
  - Role: Maintains session conversation history per userId to enable context-aware responses.  
  - Configuration: Uses userId as session key; buffers previous interactions for AI.  
  - Inputs/Outputs: Linked as AI memory to AI Agent node.  
  - Edge Cases: Memory size limits; stale or missing session data.

- **Format AI Response**  
  - Type: Code Node  
  - Role: Prepares AI response for WhatsApp sending, wrapping AI output with chatId and markdown parse mode.  
  - Inputs: AI Agent output JSON.  
  - Outputs: Formatted message JSON for sending node.

- **Google Docs - Product Catalog (Optional)**  
  - Type: Google Docs Tool Node  
  - Role: (Optional) Fetches product catalog from Google Docs for AI agent use.  
  - Configuration: Document URL must be configured and accessible.  
  - Inputs: Triggered as AI tool input.  
  - Outputs: Document content for AI context.  
  - Requirements: Google Docs OAuth credentials.  
  - Edge Cases: Access permission issues, document formatting changes.

---

#### 2.6 Message Sending

**Overview:**  
Sends the generated response back to the customer on WhatsApp with proper formatting.

**Nodes Involved:**  
- Send WhatsApp Response

**Node Details:**

- **Send WhatsApp Response**  
  - Type: WhatsApp Node (Send)  
  - Role: Sends text responses over WhatsApp API using configured account credentials.  
  - Configuration: Uses chatId and message content with markdown parsing enabled.  
  - Inputs: Formatted response JSON from pre-built or AI response nodes.  
  - Outputs: Confirmation of message dispatch.  
  - Requirements: Valid WhatsApp API credentials; message size limits considered.  
  - Edge Cases: API errors, network issues, invalid chatId.

---

### 3. Summary Table

| Node Name                          | Node Type                      | Functional Role                          | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                                     |
|-----------------------------------|--------------------------------|----------------------------------------|-------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger - Receive Messages | WhatsApp Trigger               | Receive WhatsApp messages               | -                             | Parse WhatsApp Message Data         | Step 1: Receive WhatsApp Message - Captures chat/user/message details; supports text messages only             |
| Parse WhatsApp Message Data         | Code                          | Extract and structure message data     | WhatsApp Trigger               | Classify User Intent                | Step 1: Receive WhatsApp Message                                                                                 |
| Classify User Intent                | Code                          | Classify user intent via pattern match | Parse WhatsApp Message Data    | Route by Intent                    | Step 2: Intent Classification - Recognizes product, contact, greeting, thanks, price, support, general intents  |
| Route by Intent                    | Switch                        | Route flow based on classified intent  | Classify User Intent           | Generate Product Response, Generate Contact Info Response, Generate Default Response | Step 3: Intelligent Routing - Routes to product, contact info, or default handlers                              |
| Generate Product Response          | Code                          | Static product catalog response         | Route by Intent (product_inquiry) | Build AI System Prompt              | Step 3: Intelligent Routing                                                                                        |
| Generate Contact Info Response     | Code                          | Static contact info response             | Route by Intent (contact_info) | Build AI System Prompt              | Step 3: Intelligent Routing                                                                                        |
| Generate Default Response          | Code                          | Default fallback response                | Route by Intent (extra)         | Build AI System Prompt              | Step 3: Intelligent Routing                                                                                        |
| Build AI System Prompt             | Code                          | Build AI prompt with company context    | Generate Product Response, Generate Contact Info Response, Generate Default Response | AI Agent - Handle Complex Queries | Step 4: AI Agent - Builds system prompt embedding company info and user context                                  |
| AI Agent - Handle Complex Queries  | LangChain Agent               | Process complex queries with AI         | Build AI System Prompt          | Format AI Response                 | Step 4: AI Agent                                                                                                  |
| Google Gemini Chat Model           | LangChain LM Chat Google Gemini | Google Gemini AI model                   | AI Agent - Handle Complex Queries | AI Agent - Handle Complex Queries  | Step 4: AI Agent                                                                                                  |
| Conversation Memory               | LangChain Memory Buffer Window | Maintain conversation context           | AI Agent - Handle Complex Queries (ai_memory) | AI Agent - Handle Complex Queries (ai_memory) | Step 4: AI Agent                                                                                                  |
| Format AI Response                 | Code                          | Format AI output for WhatsApp            | AI Agent - Handle Complex Queries | Send WhatsApp Response             | Step 5: Send WhatsApp Response - Formats AI replies with markdown and emojis                                    |
| Google Docs - Product Catalog (Optional) | Google Docs Tool              | Optional product catalog retrieval       | -                             | AI Agent - Handle Complex Queries (ai_tool) | -                                                                                                              |
| Send WhatsApp Response             | WhatsApp                      | Send final message back to user          | Format AI Response             | -                                 | Step 5: Send WhatsApp Response - Sends formatted replies via WhatsApp API                                       |
| Sticky Note - Main Explanation     | Sticky Note                   | Overview and instructions                | -                             | -                                 | Contains full workflow explanation and setup instructions                                                      |
| Sticky Note - Step 1               | Sticky Note                   | Explains message reception block         | -                             | -                                 | Explains Step 1: Receiving WhatsApp message                                                                    |
| Sticky Note - Step 2               | Sticky Note                   | Explains intent classification           | -                             | -                                 | Explains Step 2: Intent Classification                                                                          |
| Sticky Note - Step 3               | Sticky Note                   | Explains routing logic                    | -                             | -                                 | Explains Step 3: Intelligent Routing                                                                             |
| Sticky Note - Step 4               | Sticky Note                   | Explains AI agent handling                | -                             | -                                 | Explains Step 4: AI Agent                                                                                         |
| Sticky Note - Step 5               | Sticky Note                   | Explains sending response                  | -                             | -                                 | Explains Step 5: Sending WhatsApp response                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure webhook for messages updates only  
   - Connect WhatsApp OAuth credentials  
   - Position at start of workflow

2. **Add Code Node "Parse WhatsApp Message Data"**  
   - Input: WhatsApp Trigger output  
   - JavaScript extracts chatId, userId, userName (fallback to "Customer"), message text (lowercased and original), messageId, and timestamp  
   - Output structured session data

3. **Add Code Node "Classify User Intent"**  
   - Input: Parsed message data  
   - Implement regex-based hierarchical intent detection: contact info, greetings, thanks, product categories (customizable), order tracking, price inquiries, shipping, returns, support  
   - Outputs JSON with `intent` and `entities`

4. **Add Switch Node "Route by Intent"**  
   - Input: Intent-classified data  
   - Configure rules for:  
     - `product_inquiry` → Output 0  
     - `contact_info` → Output 1  
     - Fallback output for others  
   - Connect outputs accordingly

5. **Add Code Node "Generate Product Response"**  
   - Input: Switch output 0  
   - Write response templates for product categories with markdown formatting, prices, availability, links  
   - Customize product list, URLs, promotions, and contact info  
   - Output chatId, response, parseMode

6. **Add Code Node "Generate Contact Info Response"**  
   - Input: Switch output 1  
   - Static markdown response with company address, phones, emails, store hours, personalized with user name  
   - Output chatId, response, parseMode

7. **Add Code Node "Generate Default Response"**  
   - Input: Switch fallback output  
   - Provides general menu and prompts with customizable categories and contact info  
   - Output chatId, response, parseMode

8. **Add Code Node "Build AI System Prompt"**  
   - Input: Outputs of all three pre-built response nodes  
   - Construct detailed system prompt embedding company info, product categories, policies, pricing, shipping, refund, and customer context including user name and intent  
   - Output JSON with `systemPrompt` and `userPrompt` (original user message)

9. **Add LangChain Node "AI Agent - Handle Complex Queries"**  
   - Input: Build AI System Prompt output  
   - Configure to use Google Gemini Chat Model as language model and Conversation Memory node for context  
   - Set prompt type to "define" with systemMessage from systemPrompt and userPrompt as input text

10. **Add LangChain Node "Google Gemini Chat Model"**  
    - Configure with Google Gemini API credentials  
    - Connect as language model to AI Agent node

11. **Add LangChain Memory Node "Conversation Memory"**  
    - Set session key to userId from JSON  
    - Connect as memory buffer to AI Agent node (ai_memory input/output)

12. **Optionally Add "Google Docs - Product Catalog" Node**  
    - Configure with Google Docs OAuth credentials  
    - Provide document URL with product catalog for AI context  
    - Connect as ai_tool input to AI Agent node

13. **Add Code Node "Format AI Response"**  
    - Input: AI Agent output  
    - Format response text for WhatsApp: include chatId, response, and parseMode 'Markdown'  
    - Output formatted message JSON

14. **Add WhatsApp Node "Send WhatsApp Response"**  
    - Input: Format AI Response output  
    - Configure with WhatsApp API credentials  
    - Sends the message back to customer

15. **Connect the workflow** as follows:  
    - WhatsApp Trigger → Parse WhatsApp Message Data → Classify User Intent → Route by Intent → (Product Response / Contact Info Response / Default Response) → Build AI System Prompt → AI Agent → Format AI Response → Send WhatsApp Response

16. **Add Sticky Notes** for documentation and user guidance (optional but recommended).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| 💬 WhatsApp Customer Support Bot: Universal template for any business with hybrid AI and pre-built response approach. Saves up to 80% on AI costs by handling simple queries fast.                                                                                                                                                                                            | Sticky Note - Main Explanation                                                                           |
| 📱 Step 1: Receives WhatsApp messages automatically with user and message metadata. Supports text messages only (media can be added).                                                                                                                                                                                                                                     | Sticky Note - Step 1                                                                                      |
| 🧠 Step 2: Intent Classification using regex pattern matching for categories like product inquiry, contact info, greetings, thanks, price inquiry, support, etc.                                                                                                                                                                                                           | Sticky Note - Step 2                                                                                      |
| 🔀 Step 3: Routes incoming queries to pre-built responses or AI agent based on intent for balanced speed and cost.                                                                                                                                                                                                                                                        | Sticky Note - Step 3                                                                                      |
| 🤖 Step 4: AI Agent powered by Google Gemini processes complex queries with conversation memory and company knowledge embedded.                                                                                                                                                                                                                                          | Sticky Note - Step 4                                                                                      |
| 📤 Step 5: Sends professional, markdown formatted WhatsApp responses with emojis and links for engagement.                                                                                                                                                                                                                                                                 | Sticky Note - Step 5                                                                                      |
| Google Gemini API key and WhatsApp Business API credentials are mandatory for AI processing and message sending respectively.                                                                                                                                                                                                                                            | Credential requirements                                                                                   |
| Customize product categories, company info, and response templates in code nodes to tailor the bot to your business.                                                                                                                                                                                                                                                      | Instructions in code nodes for customization                                                             |
| LangChain integration allows use of conversation memory and modular AI agent design for advanced workflows in n8n.                                                                                                                                                                                                                                                       | LangChain nodes usage                                                                                      |

---

**Disclaimer:** The text provided is exclusively extracted from an automated n8n workflow respecting all applicable content policies and legal frameworks. The workflow manipulates only legal and public data.

---