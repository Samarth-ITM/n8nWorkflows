Find Valid Vouchers and Promo Codes with SerpAPI, Decodo, and GPT-5 Mini

https://n8nworkflows.xyz/workflows/find-valid-vouchers-and-promo-codes-with-serpapi--decodo--and-gpt-5-mini-8075


# Find Valid Vouchers and Promo Codes with SerpAPI, Decodo, and GPT-5 Mini

---

### 1. Workflow Overview

This workflow, titled **"Find Valid Vouchers and Promo Codes with SerpAPI, Decodo, and GPT-5 Mini"** (also named *Promo Seeker*), is designed to automate the discovery of valid, new, and working promotional codes and vouchers from across the internet. Its primary use case is interactive chat-based querying, where a user sends a message requesting discount codes, and the system uses AI combined with external search tools to find and verify relevant promo codes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Trigger:** Capturing user messages via a chat trigger node.
- **1.2 AI Agent Setup and Memory:** Establishing an AI agent that acts as a voucher-seeking expert, using stored conversational memory for context.
- **1.3 Search and Scraping Tools Integration:** Using external APIs (SerpAPI and Decodo Scrapper) to search the web for promo codes.
- **1.4 AI Language Model Processing:** Utilizing GPT-5 Mini to process, filter, and generate responses based on gathered data.
- **1.5 Supportive Documentation:** Sticky notes providing credential setup instructions and workflow usage details.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

- **Overview:**  
  This block captures incoming user messages via a webhook and initiates the workflow execution.

- **Nodes Involved:**  
  - Message

- **Node Details:**  

  - **Message**  
    - *Type & Role:* Chat trigger node; listens for incoming chat messages to start the workflow.  
    - *Configuration:* Webhook-based trigger with ID `8cca6785-5e60-4cca-92c2-859e362c68c9`. No additional custom parameters.  
    - *Expressions/Variables:* Receives raw user input text to be processed downstream.  
    - *Input/Output:* No input nodes; output connects to the "Promo Seeker Agent."  
    - *Version:* 1.3  
    - *Edge Cases:* Webhook connectivity issues, missing or malformed messages, and unexpected payloads could cause trigger failure.  
    - *Sub-workflow:* None.

#### 2.2 AI Agent Setup and Memory

- **Overview:**  
  This block hosts the core AI agent that orchestrates the voucher search process, maintaining chat memory to provide coherent and contextual responses.

- **Nodes Involved:**  
  - Promo Seeker Agent  
  - Chat Memory

- **Node Details:**  

  - **Promo Seeker Agent**  
    - *Type & Role:* Langchain agent node; coordinates use of tools and language models to act as a voucher and discount seeking expert.  
    - *Configuration:* System message set to: "you are a voucher and discount seeker expert. use provided tools to find a new, valid and working voucher accross internet." This instructs the AI on its role.  
    - *Expressions/Variables:* Uses linked tools (`ai_tool`), language model (`ai_languageModel`), and memory (`ai_memory`).  
    - *Input/Output:* Inputs from "Message" node; outputs conversational responses. Connected to SerpAPI, Decodo Scrapper, GPT 5 Mini, and Chat Memory nodes.  
    - *Version:* 2.2  
    - *Edge Cases:* Agent misinterpretation of instructions, tool invocation failures, or missing memory context could degrade performance. Requires all linked nodes to be operational.  
    - *Sub-workflow:* None.

  - **Chat Memory**  
    - *Type & Role:* Memory buffer node; stores recent conversational context to maintain dialogue coherence.  
    - *Configuration:* Default window buffer with no custom parameters.  
    - *Expressions/Variables:* Stores and retrieves messages dynamically as conversation progresses.  
    - *Input/Output:* Inputs from "Promo Seeker Agent" for memory updates; outputs used by the agent for context.  
    - *Version:* 1.3  
    - *Edge Cases:* Memory overflow or reset could cause loss of context.  

#### 2.3 Search and Scraping Tools Integration

- **Overview:**  
  This block integrates external search and scraping APIs to retrieve the latest promo codes and vouchers from the internet.

- **Nodes Involved:**  
  - SerpAPI  
  - Decodo Scrapper

- **Node Details:**  

  - **SerpAPI**  
    - *Type & Role:* Langchain tool node interfacing with SerpAPI for search engine queries.  
    - *Configuration:* Default options; uses API key credential named "Login w Github."  
    - *Expressions/Variables:* Receives search queries from the AI agent; returns search results.  
    - *Input/Output:* Outputs connected to "Promo Seeker Agent" as a tool.  
    - *Version:* 1  
    - *Edge Cases:* Possible API key authentication failure, rate limiting, or query errors. Network issues may also cause failures.  
    - *Sub-workflow:* None.

  - **Decodo Scrapper**  
    - *Type & Role:* Dedicated scraping tool node for Decodo API to extract discount and promo codes from various sources.  
    - *Configuration:* Default parameters; uses Decodo API credential named "cm@g.com."  
    - *Expressions/Variables:* Receives requests from the AI agent; outputs scraped promo data.  
    - *Input/Output:* Outputs connected to "Promo Seeker Agent" as a tool.  
    - *Version:* 1  
    - *Edge Cases:* API failures, invalid credentials, or rate limiting. Scraping inconsistencies may cause incomplete data.  
    - *Sub-workflow:* None.

#### 2.4 AI Language Model Processing

- **Overview:**  
  This block uses the GPT-5 Mini language model hosted on Azure OpenAI to process gathered data, refine results, and generate user-facing responses.

- **Nodes Involved:**  
  - Gpt 5 Mini

- **Node Details:**  

  - **Gpt 5 Mini**  
    - *Type & Role:* Azure OpenAI language model node; responsible for natural language understanding and generation.  
    - *Configuration:* Model set to "gpt5mini"; uses Azure OpenAI API credentials named "GPT5-mini."  
    - *Expressions/Variables:* Receives prompts and context from the AI agent; outputs text completions.  
    - *Input/Output:* Output linked to "Promo Seeker Agent" as the language model.  
    - *Version:* 1  
    - *Edge Cases:* API key invalidation, quota exhaustion, or model unavailability could lead to failures. Latency issues may affect response time.  
    - *Sub-workflow:* None.

#### 2.5 Supportive Documentation

- **Overview:**  
  This block contains sticky notes with detailed instructions on credential setup and functional overview to assist users in deploying and understanding the workflow.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  

  - **Sticky Note**  
    - *Type & Role:* Documentation node detailing credential acquisition for SerpAPI, Azure OpenAI, and Decodo.  
    - *Configuration:* Large note with step-by-step instructions and important security reminders.  
    - *Input/Output:* No inputs or outputs; purely informational.  
    - *Version:* 1  

  - **Sticky Note1**  
    - *Type & Role:* Documentation node explaining workflow purpose, operational logic, and user instructions.  
    - *Configuration:* Medium note summarizing workflow design and usage.  
    - *Input/Output:* No inputs or outputs; informational.  
    - *Version:* 1  

---

### 3. Summary Table

| Node Name          | Node Type                               | Functional Role                         | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                                                   |
|--------------------|---------------------------------------|---------------------------------------|-----------------------|-----------------------|------------------------------------------------------------------------------------------------------------------------------|
| Message            | @n8n/n8n-nodes-langchain.chatTrigger | Captures user chat input to trigger workflow | None                  | Promo Seeker Agent     | "## How This Workflow Works (Sticky Note)\n\n- This workflow is designed to help you find valid and working promo codes and vouchers from across the internet.\n- When you send a message, the AI Agent uses search tools (SerpAPI, Decodo Scrapper) to look for the latest discounts and vouchers.\n- The AI Agent checks and filters results to show only new and active promos.\n- All results are displayed in the chat for easy access.\n- Just type what kind of promo you want and let the workflow do the rest!" |
| Promo Seeker Agent  | @n8n/n8n-nodes-langchain.agent        | Orchestrates AI interaction and tool usage | Message, SerpAPI, Decodo Scrapper, Gpt 5 Mini, Chat Memory | None                  | Same as Message node sticky note above                                                                                         |
| Chat Memory        | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational context      | Promo Seeker Agent    | Promo Seeker Agent     |                                                                                                                              |
| SerpAPI            | @n8n/n8n-nodes-langchain.toolSerpApi | Performs web searches for promo codes | Promo Seeker Agent    | Promo Seeker Agent     | "## How to Get Credentials \n\n### 1. SerpAPI Node\n\n- Go to [SerpAPI](https://serpapi.com/) and sign up.\n- After logging in, find your **API Key** in your dashboard.\n- In n8n, go to **Credentials** > **Create New** > **SerpAPI**.\n- Paste your API Key and save." |
| Decodo Scrapper    | @decodo/n8n-nodes-decodo.decodoTool   | Scrapes promo codes from Decodo sources | Promo Seeker Agent    | Promo Seeker Agent     | "## How to Get Credentials \n\n### 3. Decodo Scrapper Node\n\n- Go to [Decodo](https://decodo.com/) and sign up.\n- After logging in, find your **API Key** in your account settings.\n- In n8n, go to **Credentials** > **Create New** > **Decodo API**.\n- Paste your API Key and save." |
| Gpt 5 Mini          | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Processes language understanding and generation | Promo Seeker Agent    | Promo Seeker Agent     | "## How to Get Credentials \n\n### 2. Azure OpenAI (Gpt 5 Mini) Node\n\n- Go to [Azure Portal](https://portal.azure.com/) and create an Azure OpenAI resource.\n- In your resource, go to **Keys and Endpoint**.\n- Copy your **API Key** and **Endpoint URL**.\n- In n8n, go to **Credentials** > **Create New** > **Azure OpenAI API**.\n- Enter your API Key and Endpoint, then save." |
| Sticky Note         | n8n-nodes-base.stickyNote             | Credential setup instructions          | None                  | None                  | See above for details                                                                                                         |
| Sticky Note1        | n8n-nodes-base.stickyNote             | Workflow overview and usage instructions | None                  | None                  | See above for details                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a `Chat Trigger` node named **Message**.  
   - Configure it with a webhook to receive chat messages. No special parameters needed.  
   - Save webhook URL for external integration.

2. **Create the Langchain Agent Node**  
   - Add an `Agent` node named **Promo Seeker Agent**.  
   - Set the system message to:  
     `"you are a voucher and discount seeker expert. use provided tools to find a new, valid and working voucher accross internet"`  
   - This agent will coordinate other nodes as AI tools, language models, and memory.

3. **Add Chat Memory Node**  
   - Add a `Memory Buffer Window` node named **Chat Memory** to maintain conversation context.  
   - Use default parameters.

4. **Add SerpAPI Node**  
   - Add a `SerpApi` tool node named **SerpAPI**.  
   - Configure with SerpAPI credentials:  
       - Go to [SerpAPI](https://serpapi.com/), sign up, and obtain API key.  
       - In n8n, create SerpAPI credential and attach here.

5. **Add Decodo Scrapper Node**  
   - Add a `Decodo Tool` node named **Decodo Scrapper**.  
   - Configure with Decodo API credentials:  
       - Sign up at [Decodo](https://decodo.com/), get API key.  
       - Create Decodo API credential in n8n and attach.

6. **Add GPT-5 Mini Language Model Node**  
   - Add an `Azure OpenAI` language model node named **Gpt 5 Mini**.  
   - Configure to use model `"gpt5mini"`.  
   - Setup Azure OpenAI API credentials:  
       - Create Azure OpenAI resource at [Azure Portal](https://portal.azure.com/).  
       - Copy API key and endpoint.  
       - Create Azure OpenAI credentials in n8n and attach.

7. **Connect Nodes Appropriately**  
   - Connect **Message** node output to **Promo Seeker Agent** input.  
   - Connect **SerpAPI** and **Decodo Scrapper** outputs to **Promo Seeker Agent** as AI tools.  
   - Connect **Gpt 5 Mini** output to **Promo Seeker Agent** as AI language model.  
   - Connect **Chat Memory** output to **Promo Seeker Agent** as AI memory.

8. **Set Execution Order and Save**  
   - Confirm execution order and versions are compatible (Agent node version 2.2, others as specified).  
   - Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Keep your API keys secure and never share them publicly.                                                                                                                                                                    | Security best practice                           |
| Credential setup instructions for SerpAPI, Azure OpenAI, and Decodo Scrapper are detailed inside sticky notes in the workflow.                                                                                              | See Sticky Note nodes                            |
| This workflow uses GPT-5 Mini model hosted on Azure OpenAI, which requires Azure subscription and resource setup.                                                                                                            | Azure Portal: https://portal.azure.com/         |
| SerpAPI is a powerful search API that scrapes Google and other engines, ideal for real-time promo code searches.                                                                                                            | SerpAPI: https://serpapi.com/                    |
| Decodo specializes in scraping and aggregating discount codes and deals from various sources, complementing search API results.                                                                                            | Decodo: https://decodo.com/                       |
| The workflow is designed for chat interaction, making it suitable for chatbot integrations or automated customer support for promotions and discounts discovery.                                                             | Use case context                                 |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---