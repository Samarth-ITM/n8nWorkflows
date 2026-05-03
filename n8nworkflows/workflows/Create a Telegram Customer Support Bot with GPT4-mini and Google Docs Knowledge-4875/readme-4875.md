Create a Telegram Customer Support Bot with GPT4-mini and Google Docs Knowledge

https://n8nworkflows.xyz/workflows/create-a-telegram-customer-support-bot-with-gpt4-mini-and-google-docs-knowledge-4875


# Create a Telegram Customer Support Bot with GPT4-mini and Google Docs Knowledge

### 1. Workflow Overview

This workflow creates an intelligent Telegram Customer Support Bot integrated with GPT-4o-mini (a lightweight GPT-4 variant) and Google Docs as a live knowledge base. It is designed for use cases such as e-commerce support, SaaS onboarding, internal helpdesk bots, and conversational customer interfaces without buttons or forms.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages via a Telegram Trigger node.
- **1.2 AI Processing:** Uses an AI agent powered by Langchain and OpenAI’s GPT-4o-mini to interpret messages.
- **1.3 Knowledge Retrieval:** Fetches relevant documentation content from a specified Google Docs document to ground AI responses.
- **1.4 Memory Handling:** Maintains recent conversation context via a memory buffer to enable coherent multi-turn dialogue.
- **1.5 Output Delivery:** Sends AI-generated responses back to the Telegram user.
- **1.6 Documentation and Notes:** Provides detailed sticky notes explaining the workflow’s purpose, components, and use cases for maintainability and clarity.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Captures incoming user messages from Telegram to trigger the workflow.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: `telegramTrigger` (Webhook trigger node)  
    - Configuration: Listens for "message" updates from Telegram, capturing any text sent by users.  
    - Expressions/Variables: Outputs `message` object including `chat.id` and `text` for downstream nodes.  
    - Connections: Output connected to "Customer Support AI Agent" node.  
    - Edge Cases: Potential webhook misconfigurations, Telegram API downtime, or invalid webhook setup could cause message capture failure.  
    - Sticky Note: Two sticky notes labeled "Telegram Trigger" visually associated for clarity.

#### 2.2 AI Processing

- **Overview:**  
Processes the incoming Telegram message as a prompt using a custom AI Agent that references external knowledge and memory.

- **Nodes Involved:**  
  - Customer Support AI Agent  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**  
  - **Customer Support AI Agent**  
    - Type: Langchain Agent node  
    - Role: Processes user message text with context from Google Docs and memory, uses defined system prompt to guide behavior.  
    - Configuration:  
      - Input text: `{{$json.message.text}}` (text from Telegram message)  
      - System Message: Defines agent as a friendly, casual customer support agent that cross-references Google Docs knowledge and avoids hallucination when no info is found.  
      - Prompt Type: Custom defined prompt.  
    - Connections: Inputs from Telegram Trigger (main), OpenAI Chat Model (ai_languageModel), Google Docs (ai_tool), Simple Memory (ai_memory). Output connected to Telegram node.  
    - Edge Cases: Failures in AI model response, misinterpretation of system prompt, or missing knowledge base content could degrade response quality.  
  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI chat model  
    - Role: Provides GPT-4o-mini language model for AI Agent.  
    - Configuration: Selected GPT-4o-mini model for efficient inference. No special options enabled.  
    - Connections: Output connected as ai_languageModel input to Customer Support AI Agent.  
    - Credentials: Requires OpenAI API credentials.  
    - Edge Cases: API key issues, rate limits, or model unavailability.  
  - **Simple Memory**  
    - Type: Langchain Memory Buffer (Window)  
    - Role: Maintains conversation history using Telegram chat ID as session key for context continuity.  
    - Configuration: Session key expression `={{ $json.message.chat.id }}` to isolate chat sessions.  
    - Connections: Output connected as ai_memory input to Customer Support AI Agent.  
    - Edge Cases: Memory overflow or session key mismatches could cause context loss.

#### 2.3 Knowledge Retrieval

- **Overview:**  
Retrieves the latest content from a Google Docs document which serves as the knowledge base for the AI agent.

- **Nodes Involved:**  
  - Google Docs

- **Node Details:**  
  - **Google Docs**  
    - Type: Google Docs Tool (v2)  
    - Role: Fetches the entire document content from a provided Google Docs URL.  
    - Configuration: Operation set to "get"; document URL specifically points to the source knowledge document.  
    - Connections: Output connected as ai_tool input to Customer Support AI Agent.  
    - Credentials: Requires Google Docs OAuth2 credentials with read access.  
    - Edge Cases: Document permission issues, invalid or expired URL, or API quota exhaustion.  
    - Notes: This integration enables real-time referencing of knowledge for AI response grounding.

#### 2.4 Memory Handling

- **Overview:**  
Manages the conversation history for each user to provide context-aware responses.

- **Nodes Involved:**  
  - Simple Memory (already described in AI Processing)

- **Node Details:**  
  - See above under AI Processing.

#### 2.5 Output Delivery

- **Overview:**  
Sends the AI-generated response back to the Telegram user chat.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**  
  - **Telegram**  
    - Type: Telegram node (send message)  
    - Role: Posts the AI response text back to the originating Telegram chat.  
    - Configuration:  
      - Text: `={{ $json.output }}` (output from AI Agent)  
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}` to reply to the correct chat.  
      - Additional Field: `appendAttribution` set to false to avoid adding extra attribution text.  
    - Connections: Input from Customer Support AI Agent main output.  
    - Credentials: Requires Telegram Bot API credentials.  
    - Edge Cases: Invalid chat ID, Telegram API errors, or message size limits.

#### 2.6 Documentation and Notes

- **Overview:**  
Provides detailed sticky notes explaining the workflow components, use cases, and external references.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note6

- **Node Details:**  
  - **Sticky Note6 (Main explainer)**  
    - Content: Detailed explanation of the workflow purpose, how it works, tools used, and use cases.  
    - Position: Prominently placed for easy reference.  
  - **Sticky Note, Sticky Note1, Sticky Note2**  
    - Content: The label "Telegram Trigger" repeated for clarity and visual grouping near the trigger node.  
  - No functional impact; purely for documentation and user guidance.

---

### 3. Summary Table

| Node Name             | Node Type                              | Functional Role                     | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                              |
|-----------------------|--------------------------------------|-----------------------------------|-----------------------|-----------------------|--------------------------------------------------------------------------------------------------------|
| Telegram Trigger      | telegramTrigger                      | Captures incoming Telegram messages| None                  | Customer Support AI Agent | Telegram Trigger                                                                                         |
| Customer Support AI Agent | Langchain Agent                   | Processes message using AI agent with memory and knowledge base | Telegram Trigger, OpenAI Chat Model, Google Docs, Simple Memory | Telegram              |                                                                                                        |
| OpenAI Chat Model     | Langchain OpenAI Chat Model          | Provides GPT-4o-mini language model| None                  | Customer Support AI Agent |                                                                                                        |
| Simple Memory         | Langchain Memory Buffer Window       | Maintains chat history context    | None                  | Customer Support AI Agent |                                                                                                        |
| Google Docs           | Google Docs Tool                     | Retrieves knowledge base content  | None                  | Customer Support AI Agent |                                                                                                        |
| Telegram              | Telegram send message node           | Sends AI response back to user    | Customer Support AI Agent | None                  |                                                                                                        |
| Sticky Note           | Sticky Note                         | Documentation (Telegram Trigger label) | None                  | None                  | Telegram Trigger                                                                                         |
| Sticky Note1          | Sticky Note                         | Documentation (Telegram Trigger label) | None                  | None                  | Telegram Trigger                                                                                         |
| Sticky Note2          | Sticky Note                         | Documentation (Telegram Trigger label) | None                  | None                  | Telegram Trigger                                                                                         |
| Sticky Note6          | Sticky Note                         | Detailed workflow explanation     | None                  | None                  | 🤖 AI Customer Support Agent with Google Docs Knowledge (Telegram + OpenAI) \n\nThis no-code workflow turns your Telegram bot into an intelligent, always-on AI support agent that references your business documentation in Google Docs to respond to customer queries—instantly and accurately.\n\nWatch full step-by-step video tutorial of the build here:\nhttps://www.youtube.com/@Automatewithmarc\n\n🔧 How it works:\nTelegram Trigger – Captures incoming messages from users on your Telegram bot\n\nLangchain AI Agent (OpenAI GPT) – Interprets the message and uses RAG (retrieval-augmented generation) techniques to craft an answer\n\nGoogle Docs Tool – Connects to and retrieves context from your specified Google Doc (e.g. FAQ, SOPs, policies)\n\nMemory Buffer – Keeps track of recent chat history for more human-like conversations\n\nTelegram Reply Node – Sends the AI-generated response back to the user\n\n💡 Use Cases:\nE-commerce customer service\n\nSaaS product onboarding\n\nInternal helpdesk bot for teams\n\nWhatsApp-style support for digital businesses\n\n🧠 What makes this powerful:\nSupports complex questions by referencing a live Google Doc knowledge base\n\nWorks in plain conversational language (no buttons or forms needed)\n\nRuns 24/7 with zero code\n\nEasily extendable to Slack, WhatsApp, or email support\n\n🛠️ Tools used:\nTelegram Node (trigger + send)\n\nLangchain Agent with OpenAI GPT\n\nGoogle Docs Tool\n\nMemory Buffer\n\nSticky Notes for easy understanding |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Node Type: Telegram Trigger  
   - Configuration: Set "Updates" to listen for "message" events.  
   - Credentials: Connect your Telegram Bot API credentials.  
   - Position: Start node for receiving user messages.

2. **Create OpenAI Chat Model Node**  
   - Node Type: Langchain OpenAI Chat Model  
   - Configuration: Select model "gpt-4o-mini" for lightweight GPT-4 inference.  
   - Credentials: Connect your OpenAI API credentials.

3. **Create Simple Memory Node**  
   - Node Type: Langchain Memory Buffer Window  
   - Configuration:  
     - Set Session Key to expression `={{ $json.message.chat.id }}` to isolate per-chat memory.  
     - Session ID type: Custom Key.

4. **Create Google Docs Node**  
   - Node Type: Google Docs Tool (v2)  
   - Configuration:  
     - Operation: "get"  
     - Document URL: Paste the Google Docs URL containing your knowledge base (e.g. FAQs, SOPs).  
   - Credentials: Connect Google Docs OAuth2 credentials with permission to read the document.

5. **Create Customer Support AI Agent Node**  
   - Node Type: Langchain Agent  
   - Configuration:  
     - Text input: Use expression `={{ $json.message.text }}` to get incoming Telegram message text.  
     - Prompt Type: "define" (custom system prompt)  
     - Options > System Message:  
       ```
       You are a helpful Customer Support Agent, cross-reference the Google Doc for any customer queries for relevant information. If relevant information is not found, do not make up facts. Plainly let the user know that the information isn't available.

       #Tonality
       Friendly, casual and helpful. if user or customer says hi, inform them that you are a Friendly Customer Support Agent
       ```
   - Connect inputs:  
     - `main` input from Telegram Trigger  
     - `ai_languageModel` input from OpenAI Chat Model  
     - `ai_memory` input from Simple Memory  
     - `ai_tool` input from Google Docs

6. **Create Telegram Node (Send Message)**  
   - Node Type: Telegram  
   - Configuration:  
     - Text: `={{ $json.output }}` (output from AI Agent)  
     - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}` (respond to correct user chat)  
     - Additional Fields: Disable `appendAttribution` (set to false)  
   - Credentials: Use Telegram Bot API credentials.

7. **Connect Nodes in Order:**  
   - Telegram Trigger → Customer Support AI Agent (main input)  
   - OpenAI Chat Model → Customer Support AI Agent (ai_languageModel input)  
   - Simple Memory → Customer Support AI Agent (ai_memory input)  
   - Google Docs → Customer Support AI Agent (ai_tool input)  
   - Customer Support AI Agent → Telegram (main output)

8. **Add Sticky Notes (Optional but Recommended):**  
   - Add notes near Telegram Trigger labeled "Telegram Trigger" for identification.  
   - Add a large explanatory sticky note summarizing the workflow, use cases, and providing the YouTube tutorial link:  
     ```
     🤖 AI Customer Support Agent with Google Docs Knowledge (Telegram + OpenAI)

     This no-code workflow turns your Telegram bot into an intelligent, always-on AI support agent that references your business documentation in Google Docs to respond to customer queries—instantly and accurately.

     Watch full step-by-step video tutorial of the build here: https://www.youtube.com/@Automatewithmarc

     [Further descriptive content as in Sticky Note6]
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Full video tutorial of this workflow available at Automate with Marc YouTube channel                                               | https://www.youtube.com/@Automatewithmarc                       |
| Workflow supports retrieval-augmented generation (RAG) by integrating Google Docs live knowledge base with Langchain AI agent    | Enhances accuracy and reduces hallucinations                    |
| Designed for 24/7 operation with zero code; easily extendable to Slack, WhatsApp, or email support                                 | Scalability and multi-channel support                           |
| Use cases include e-commerce customer service, SaaS onboarding, internal helpdesk, and conversational business support bots      | Application context                                             |

---

**Disclaimer:**  
The provided text and workflow are exclusively derived from an automated n8n workflow using only legal, public data, compliant with all content policies. No illegal, offensive, or protected content is included.