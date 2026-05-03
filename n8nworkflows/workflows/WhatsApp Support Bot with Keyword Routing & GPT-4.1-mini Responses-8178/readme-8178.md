WhatsApp Support Bot with Keyword Routing & GPT-4.1-mini Responses

https://n8nworkflows.xyz/workflows/whatsapp-support-bot-with-keyword-routing---gpt-4-1-mini-responses-8178


# WhatsApp Support Bot with Keyword Routing & GPT-4.1-mini Responses

---
### 1. Workflow Overview

This workflow implements a **WhatsApp Support Bot** that processes incoming WhatsApp messages, routes queries based on keywords, and leverages an AI language model ("gpt-4.1-mini") for dynamic responses beyond predefined FAQs. It targets customer support use cases where users seek information about pricing, support, or other company/product details through WhatsApp chat.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures incoming WhatsApp messages via a webhook trigger.
- **1.2 Greeting & Keyword Routing:** Checks if the message is a greeting or contains predefined keywords ("price", "support") to send canned responses.
- **1.3 AI Processing:** For messages not matched by keywords, invokes an AI agent powered by GPT-4.1-mini to generate contextual replies.
- **1.4 Response Delivery:** Sends the appropriate reply back to the user via WhatsApp.

Sticky notes scattered across the workflow clarify the role of nodes and groups of nodes, emphasizing the sequential evaluation of conditions and response types.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block serves as the entry point, capturing incoming WhatsApp messages in real-time via a webhook.

- **Nodes Involved:**  
  - WhatsApp Trigger1  
  - Sticky Note (explaining entry point)

- **Node Details:**

  - **WhatsApp Trigger1**  
    - Type: WhatsApp Trigger  
    - Role: Listens for incoming WhatsApp messages (specifically message updates).  
    - Configuration:  
      - Webhook enabled for message updates.  
      - Uses OAuth credentials ("WhatsApp OAuth account 2") for secure API access.  
    - Inputs: External WhatsApp messages via webhook.  
    - Outputs: JSON containing message text, sender info, contacts array.  
    - Edge Cases: Webhook failures, API auth errors, message format inconsistencies.  
    - Sticky Note: "WhatsApp Trigger: Entry point that captures incoming messages."

---

#### 1.2 Greeting & Keyword Routing

- **Overview:**  
  Evaluates the incoming message content to identify greetings or keywords ("price", "support") for predefined responses. This block uses conditional checks (IF nodes) to branch the flow accordingly.

- **Nodes Involved:**  
  - Check Greeting (IF node)  
  - If (keyword check for "price")  
  - If1 (keyword check for "support")  
  - Send Greeting Response (for greetings)  
  - Send Greeting Response1 (for pricing info)  
  - Send Greeting Response2 (for support info)  
  - Sticky Notes (covering IF conditions and responses)

- **Node Details:**

  - **Check Greeting**  
    - Type: IF  
    - Role: Checks if message contains greetings "hi" or "hello" (case-insensitive).  
    - Configuration:  
      - Uses string "contains" operator on lowercase message body.  
      - Conditions combined with "OR".  
    - Inputs: Message text from WhatsApp Trigger1.  
    - Outputs:  
      - True branch: Sends greeting response.  
      - False branch: Passes to next keyword check (If node).  
    - Edge Cases: Variations in greeting spelling or language not detected.

  - **Send Greeting Response**  
    - Type: WhatsApp node (send message)  
    - Role: Sends a friendly welcome message listing main support categories.  
    - Configuration:  
      - Fixed text body explaining bot capabilities (pricing, support, other info).  
      - Recipient phone number dynamically set from trigger contact wa_id.  
      - Uses WhatsApp API credentials ("WhatsApp account 2").  
    - Inputs: True output from Check Greeting.  
    - Outputs: None (end of flow for greetings).  
    - Edge Cases: WhatsApp API errors, invalid phone numbers.

  - **If**  
    - Type: IF  
    - Role: Checks if message contains keyword "price" (case-sensitive, strict).  
    - Configuration:  
      - String "contains" operator on lowercase message body.  
      - Single condition.  
    - Inputs: From Check Greeting false branch.  
    - Outputs:  
      - True branch: Sends pricing details via Send Greeting Response1.  
      - False branch: Passes to If1 for next keyword check.  
    - Edge Cases: Keyword variations like "pricing" or misspellings not detected.

  - **Send Greeting Response1**  
    - Type: WhatsApp node (send message)  
    - Role: Sends pricing information with invitation to get full price list.  
    - Configuration:  
      - Fixed text body outlining basic and premium plans.  
      - Dynamic recipient from trigger contact wa_id.  
      - Uses WhatsApp API credentials.  
    - Inputs: True output from If node.  
    - Outputs: None.  
    - Edge Cases: WhatsApp API or network errors.

  - **If1**  
    - Type: IF  
    - Role: Checks if message exactly equals "support" (case-sensitive).  
    - Configuration:  
      - String "equals" operator on lowercase message body.  
      - Single condition.  
    - Inputs: False output from If node.  
    - Outputs:  
      - True branch: Sends support contact info via Send Greeting Response2.  
      - False branch: Passes message to AI Agent node.  
    - Edge Cases: Partial matches or synonyms like "help" not handled.

  - **Send Greeting Response2**  
    - Type: WhatsApp node (send message)  
    - Role: Sends support contact details including email and phone number.  
    - Configuration:  
      - Fixed text body with contact info and availability.  
      - Dynamic recipient from trigger contact wa_id.  
      - Uses WhatsApp API credentials.  
    - Inputs: True output from If1.  
    - Outputs: None.  
    - Edge Cases: API delivery failures.

  - **Sticky Notes**  
    - Cover the IF nodes and response nodes, explaining:  
      - "IF Conditions: Sequential checks for predefined responses"  
      - "Response Types: Different greeting responses for various scenarios"

---

#### 1.3 AI Processing

- **Overview:**  
  Handles messages not matched by predefined keywords by querying an AI agent (GPT-4.1-mini) to generate dynamic and context-aware replies.

- **Nodes Involved:**  
  - AI Agent (Langchain agent node)  
  - OpenAI Chat Model (GPT-4.1-mini)  
  - Sticky Note (explaining AI Agent)

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Processes the original message with a defined prompt to generate a response using the linked language model.  
    - Configuration:  
      - Takes message text from JSON path `$json.messages[0].text`.  
      - Uses "define" prompt type (custom prompt definition).  
      - Connects to OpenAI Chat Model as language model.  
    - Inputs: From If1 false output (messages without matched keywords).  
    - Outputs: AI-generated text in property `output`.  
    - Edge Cases: Model API rate limits, incomplete prompt context, network failures.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node  
    - Role: Provides GPT-4.1-mini model as the language model for the AI agent.  
    - Configuration:  
      - Model set to "gpt-4.1-mini" (compact variant of GPT-4).  
      - No extra options configured.  
      - Uses OpenAI API credentials ("OpenAi account 2").  
    - Inputs: From AI Agent (ai_languageModel input).  
    - Outputs: Language model completions back to AI Agent.  
    - Edge Cases: API key expiration, quota exhaustion, version compatibility.

  - **Sticky Note**  
    - Content: "AI Agent: LLM processing for complex/dynamic questions"

---

#### 1.4 Response Delivery

- **Overview:**  
  Sends the AI-generated response text back to the WhatsApp user.

- **Nodes Involved:**  
  - Send message1 (WhatsApp send node)  
  - Sticky Note (explaining final send action)

- **Node Details:**

  - **Send message1**  
    - Type: WhatsApp node (send message)  
    - Role: Delivers the AI-generated response text to the user.  
    - Configuration:  
      - Text body set dynamically from AI Agent output property `$json.output`.  
      - Recipient set from original WhatsApp Trigger1 contact wa_id.  
      - Uses WhatsApp API credentials ("WhatsApp account 2").  
    - Inputs: From AI Agent.  
    - Outputs: None (end of flow).  
    - Edge Cases: WhatsApp API delivery errors, invalid or outdated wa_id.

  - **Sticky Note**  
    - Content: "Final Send Action: Delivers AI-generated responses"

---

### 3. Summary Table

| Node Name              | Node Type                         | Functional Role                             | Input Node(s)         | Output Node(s)          | Sticky Note                                                |
|------------------------|----------------------------------|---------------------------------------------|-----------------------|-------------------------|------------------------------------------------------------|
| WhatsApp Trigger1      | WhatsApp Trigger                 | Entry point for incoming WhatsApp messages | (none)                | Check Greeting          | WhatsApp Trigger: Entry point that captures incoming messages |
| Check Greeting         | IF                              | Checks if message is greeting               | WhatsApp Trigger1     | Send Greeting Response, If | IF Conditions: Sequential checks for predefined responses  |
| Send Greeting Response | WhatsApp Send                   | Sends greeting message with bot options    | Check Greeting (true) | (none)                  | Response Types: Different greeting responses for various scenarios |
| If                     | IF                              | Checks if message contains "price" keyword | Check Greeting (false)| Send Greeting Response1, If1 | IF Conditions: Sequential checks for predefined responses  |
| Send Greeting Response1| WhatsApp Send                   | Sends pricing information                    | If (true)             | (none)                  | Response Types: Different greeting responses for various scenarios |
| If1                    | IF                              | Checks if message equals "support"          | If (false)            | Send Greeting Response2, AI Agent | IF Conditions: Sequential checks for predefined responses  |
| Send Greeting Response2| WhatsApp Send                   | Sends support contact info                   | If1 (true)            | (none)                  | Response Types: Different greeting responses for various scenarios |
| AI Agent               | Langchain Agent Node            | Processes unmatched messages with AI        | If1 (false)           | Send message1           | AI Agent: LLM processing for complex/dynamic questions     |
| OpenAI Chat Model      | Langchain OpenAI Chat Model     | Provides GPT-4.1-mini model for AI Agent    | AI Agent (ai_languageModel) | AI Agent          |                                                            |
| Send message1          | WhatsApp Send                   | Sends AI-generated response                  | AI Agent              | (none)                  | Final Send Action: Delivers AI-generated responses          |
| Sticky Note            | Sticky Note                    | Notes on WhatsApp Trigger                    | (none)                | (none)                  | WhatsApp Trigger: Entry point that captures incoming messages |
| Sticky Note1           | Sticky Note                    | Notes on IF conditions                        | (none)                | (none)                  | IF Conditions: Sequential checks for predefined responses  |
| Sticky Note2           | Sticky Note                    | Notes on response types                       | (none)                | (none)                  | Response Types: Different greeting responses for various scenarios |
| Sticky Note3           | Sticky Note                    | Notes on response types                       | (none)                | (none)                  | Response Types: Different greeting responses for various scenarios |
| Sticky Note4           | Sticky Note                    | Notes on response types                       | (none)                | (none)                  | Response Types: Different greeting responses for various scenarios |
| Sticky Note5           | Sticky Note                    | Notes on IF conditions                        | (none)                | (none)                  | IF Conditions: Sequential checks for predefined responses  |
| Sticky Note6           | Sticky Note                    | Notes on IF conditions                        | (none)                | (none)                  | IF Conditions: Sequential checks for predefined responses  |
| Sticky Note7           | Sticky Note                    | Notes on AI Agent                             | (none)                | (none)                  | AI Agent: LLM processing for complex/dynamic questions     |
| Sticky Note8           | Sticky Note                    | Notes on final send action                    | (none)                | (none)                  | Final Send Action: Delivers AI-generated responses          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure webhook to listen for "messages" updates.  
   - Set OAuth2 credentials for WhatsApp API access ("WhatsApp OAuth account 2").  
   - Position: Start of workflow.

2. **Add IF Node "Check Greeting"**  
   - Type: IF  
   - Configure with OR condition: message text contains "hi" or "hello" (case-insensitive).  
   - Input expression: `$json.messages[0].text.body.toLowerCase()`  
   - Connect WhatsApp Trigger output to this node input.

3. **Add WhatsApp Send Node "Send Greeting Response"**  
   - Type: WhatsApp  
   - Text:  
     ```
     Hello! 👋 Welcome to our support bot.  
     I can help you with the following:  
     📦 *Pricing* - Get details of our plans and offers  
     🛠 *Support* - Resolve technical issues and get contact details  
     ℹ️ *Other* - Company information and product details  
     What would you like to know?
     ```  
   - Recipient phone number: `={{ $('WhatsApp Trigger1').item.json.contacts[0].wa_id }}`  
   - PhoneNumberId: Use your WhatsApp phone number ID (e.g., "817103004812479").  
   - Credentials: WhatsApp API credentials ("WhatsApp account 2").  
   - Connect "Check Greeting" true output to this node.

4. **Add IF Node "If" for Pricing Keyword**  
   - Type: IF  
   - Condition: message text contains "price" (case sensitive as per config, but recommended to use lowercase).  
   - Expression: `$json.messages[0].text.body.toLowerCase()`  
   - Connect "Check Greeting" false output to this node.

5. **Add WhatsApp Send Node "Send Greeting Response1"**  
   - Type: WhatsApp  
   - Text:  
     ```
     📦 *Pricing Details*  
     Our basic package starts at $50/month.  
     Premium plans are also available with additional features.  
     Would you like me to share the full pricing list?
     ```  
   - Recipient phone number: same as above.  
   - Credentials: WhatsApp API credentials.  
   - Connect "If" true output to this node.

6. **Add IF Node "If1" for Support Keyword**  
   - Type: IF  
   - Condition: message text equals "support" (case-sensitive).  
   - Expression: `$json.messages[0].text.body.toLowerCase()`  
   - Connect "If" false output to this node.

7. **Add WhatsApp Send Node "Send Greeting Response2"**  
   - Type: WhatsApp  
   - Text:  
     ```
     🛠 *Support Assistance*  
     For technical help or queries, you can reach us:  
     - 📧 Email: support@company.com  
     - 📞 Phone: +1 234 567 890  
     We’re here 24/7 to help you out!
     ```  
   - Recipient phone number: same as above.  
   - Credentials: WhatsApp API credentials.  
   - Connect "If1" true output to this node.

8. **Add Langchain Agent Node "AI Agent"**  
   - Type: Langchain Agent  
   - Text input: `={{ $json.messages[0].text }}`  
   - Prompt type: "define" (custom prompt)  
   - Connect "If1" false output (messages not matched by keywords) to this node.

9. **Add Langchain OpenAI Chat Model Node "OpenAI Chat Model"**  
   - Type: Langchain OpenAI Chat Model  
   - Model: Select "gpt-4.1-mini" from model list.  
   - Credentials: OpenAI API credentials ("OpenAi account 2").  
   - Connect this node's output to AI Agent's ai_languageModel input.

10. **Add WhatsApp Send Node "Send message1"**  
    - Type: WhatsApp  
    - Text: `={{ $json.output }}` (output from AI Agent)  
    - Recipient phone number: same as above.  
    - Credentials: WhatsApp API credentials.  
    - Connect AI Agent output to this node.

11. **Add Sticky Notes (Optional for clarity)**  
    - Add notes explaining: entry point (WhatsApp Trigger), IF conditions, response types, AI agent role, and final send action.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| WhatsApp API credentials require proper OAuth2 setup and phone number ID configuration.                 | WhatsApp Business API documentation                  |
| OpenAI API usage requires API key with access to GPT-4.1-mini model and sufficient quota.                | https://platform.openai.com/docs/models/gpt-4-mini  |
| The workflow performs case-sensitive checks; consider expanding keyword recognition for better UX.     | Best practices for natural language processing       |
| AI responses depend on prompt design; customizing prompts in the Langchain Agent node can improve results. | Langchain documentation: https://js.langchain.com/  |
| For production, add error handling nodes (e.g., catch errors, retries) to handle API failures gracefully. | n8n error workflow design guide                       |

---

This documentation fully describes the WhatsApp Support Bot workflow's structure, logic, and configuration to enable reproduction, modification, and troubleshooting by developers and automated agents.