Create a Knowledge Base Chatbot with OpenAI and Notion for Website Embedding

https://n8nworkflows.xyz/workflows/create-a-knowledge-base-chatbot-with-openai-and-notion-for-website-embedding-6500


# Create a Knowledge Base Chatbot with OpenAI and Notion for Website Embedding

### 1. Workflow Overview

This workflow implements a **Knowledge Base Chatbot** designed for embedding on websites by integrating **OpenAI's language models** with **Notion** as a knowledge base. Its primary goal is to receive user chat inputs, process them using an AI agent enhanced with memory and retrieval capabilities, and respond intelligently based on information stored in a Notion database.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Handles incoming chat messages from website users via a webhook.
- **1.2 AI Processing:** Uses a Smart AI Agent to process the conversation, powered by OpenAI’s chat model, enriched with chat history memory, and connected to Notion as a knowledge base tool.
- **1.3 Knowledge Base Integration:** Interfaces with Notion to read/write data, enabling the AI to retrieve or update relevant knowledge.
- **1.4 Memory Management:** Maintains context of the conversation through a windowed chat memory buffer.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives chat messages from users via a webhook trigger, serving as the entry point for the chatbot conversation.

- **Nodes Involved:**  
  - Start Chat Conversation

- **Node Details:**

  - **Start Chat Conversation**  
    - Type: LangChain Chat Trigger (Webhook-based trigger for chat)  
    - Configuration: Listens for incoming chat messages on a webhook URL (webhookId: cf1de04f-3e38-426c-89f0-3bdb110a5dcf)  
    - Key Expressions/Variables: Receives user chat inputs as webhook payload  
    - Input connections: None (trigger node)  
    - Output connections: Feeds into the Smart AI Agent node  
    - Version: 1.1  
    - Edge Cases / Failures: Webhook downtime, invalid payload format, latency  
    - Sub-workflow: None

#### 2.2 AI Processing

- **Overview:**  
  Processes the chat input using a Smart AI Agent node that orchestrates interaction between the language model, memory, and the Notion tool to generate responses.

- **Nodes Involved:**  
  - Smart AI Agent  
  - OpenAI Chat Model  
  - Remember Chat History  
  - Set & Get Notion Database

- **Node Details:**

  - **Smart AI Agent**  
    - Type: LangChain Agent  
    - Role: Core AI orchestrator that integrates language model, memory, and external tools to generate responses  
    - Configuration: Receives input from the chat trigger and directs requests to the OpenAI Chat Model, memory, and Notion tool  
    - Key Expressions: Dynamically uses ai_languageModel, ai_memory, ai_tool input/output bindings  
    - Input connections: Receives from Start Chat Conversation, OpenAI Chat Model, Remember Chat History, Set & Get Notion Database (via ai_languageModel, ai_memory, ai_tool respectively)  
    - Output connections: None (terminal node for this data chain)  
    - Version: 1.7  
    - Edge Cases: Authentication failures with AI or Notion, API rate limits, timeout, unexpected responses from AI or Notion, memory overflow issues  
    - Sub-workflow: None

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI LM Chat Node  
    - Role: Provides the language generation capabilities based on OpenAI's GPT-like models  
    - Configuration: Connected as the language model component for the Smart AI Agent  
    - Key Expressions: ai_languageModel output to Smart AI Agent  
    - Input connections: None (feeds into Smart AI Agent)  
    - Output connections: To Smart AI Agent (ai_languageModel)  
    - Version: 1.2  
    - Edge Cases: API quota limits, network issues, invalid API key, response timeouts  
    - Credentials: Requires OpenAI API credentials

  - **Remember Chat History**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains recent conversation context to provide memory for the AI agent  
    - Configuration: Stores a sliding window of chat messages  
    - Key Expressions: Outputs ai_memory to Smart AI Agent  
    - Input connections: None (feeds into Smart AI Agent)  
    - Output connections: To Smart AI Agent (ai_memory)  
    - Version: 1.3  
    - Edge Cases: Memory size constraints, data consistency, context loss over long conversations

  - **Set & Get Notion Database**  
    - Type: Notion Tool Node  
    - Role: Interfaces with Notion to read/write knowledge base data that informs AI responses  
    - Configuration: Configured to access a specific Notion database  
    - Key Expressions: Outputs ai_tool to Smart AI Agent  
    - Input connections: None (feeds into Smart AI Agent)  
    - Output connections: To Smart AI Agent (ai_tool)  
    - Version: 2.2  
    - Edge Cases: Notion API rate limits, permission errors, incorrect database schema, network failures  
    - Credentials: Requires Notion integration credentials

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                  | Input Node(s)          | Output Node(s)          | Sticky Note |
|-------------------------|----------------------------------|---------------------------------|-----------------------|------------------------|-------------|
| Start Chat Conversation | LangChain Chat Trigger            | Receive user chat input          | None                  | Smart AI Agent          |             |
| Smart AI Agent          | LangChain Agent                  | Process chat with AI + tools     | Start Chat Conversation, OpenAI Chat Model, Remember Chat History, Set & Get Notion Database | None                   |             |
| OpenAI Chat Model       | LangChain OpenAI LM Chat          | Generate AI language responses   | None                  | Smart AI Agent          |             |
| Remember Chat History   | LangChain Memory Buffer Window    | Maintain conversation context    | None                  | Smart AI Agent          |             |
| Set & Get Notion Database | Notion Tool Node                | Interface with Notion knowledge base | None                  | Smart AI Agent          |             |
| Sticky Note1            | Sticky Note                      | (Empty content)                  | None                  | None                   |             |
| Sticky Note2            | Sticky Note                      | (Empty content)                  | None                  | None                   |             |
| Sticky Note3            | Sticky Note                      | (Empty content)                  | None                  | None                   |             |
| Sticky Note             | Sticky Note                      | (Empty content)                  | None                  | None                   |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **LangChain Chat Trigger** node named `Start Chat Conversation`  
   - Configure webhook with auto-generated ID or custom path  
   - No credentials needed  
   - Purpose: Receive chat inputs

2. **Create AI Agent Node**  
   - Add **LangChain Agent** node named `Smart AI Agent`  
   - No special parameters at creation but will connect inputs below  
   - Version 1.7 or later recommended

3. **Create OpenAI Model Node**  
   - Add **LangChain OpenAI LM Chat** node named `OpenAI Chat Model`  
   - Configure with OpenAI API credentials  
   - Select model (e.g., GPT-4 or GPT-3.5) and adjust parameters as needed (temperature, max tokens)  
   - Version 1.2 or later

4. **Create Memory Node**  
   - Add **LangChain Memory Buffer Window** node named `Remember Chat History`  
   - Configure window size (number of messages to remember) as desired  
   - Version 1.3 or later

5. **Create Notion Tool Node**  
   - Add **Notion Tool** node named `Set & Get Notion Database`  
   - Configure with Notion credentials and select the target database for knowledge base interaction  
   - Version 2.2 or later

6. **Link Nodes**  
   - Connect `Start Chat Conversation` → `Smart AI Agent` (main connection)  
   - Connect `OpenAI Chat Model` → `Smart AI Agent` via `ai_languageModel` output/input  
   - Connect `Remember Chat History` → `Smart AI Agent` via `ai_memory` output/input  
   - Connect `Set & Get Notion Database` → `Smart AI Agent` via `ai_tool` output/input

7. **Test Webhook**  
   - Activate workflow  
   - Test sending chat input to webhook URL  
   - Verify AI agent responds with context-aware answers informed by Notion data

8. **Optional: Add Sticky Notes**  
   - Add sticky notes for documentation or instructions as needed (empty in this workflow)

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                           |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow integrates OpenAI with Notion as a knowledge base to enable AI-powered chatbots.  | n8n, OpenAI, Notion official documentation                |
| The workflow uses LangChain nodes for seamless AI orchestration and memory management.           | https://docs.langchain.com/                                |
| Ensure API keys for OpenAI and Notion are kept secure and have appropriate permissions.          | OpenAI and Notion developer portals                        |
| Webhook URL should be embedded in the website frontend to enable chat input capture.              | n8n webhook documentation                                  |

---

**Disclaimer:** The provided text is exclusively generated from an n8n automated workflow. It complies strictly with content policies and contains no illegal or protected information. All data manipulated is legal and public.