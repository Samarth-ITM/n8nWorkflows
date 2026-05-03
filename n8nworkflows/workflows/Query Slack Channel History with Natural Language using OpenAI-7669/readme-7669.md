Query Slack Channel History with Natural Language using OpenAI

https://n8nworkflows.xyz/workflows/query-slack-channel-history-with-natural-language-using-openai-7669


# Query Slack Channel History with Natural Language using OpenAI

---

### 1. Workflow Overview

This workflow enables natural language querying of a Slack channel’s message history by integrating Slack’s API with OpenAI’s language models via n8n. It allows users to ask questions in everyday language (e.g., “What were the decisions?”, “Who’s blocked?”, “Summarize yesterday?”), and the workflow fetches relevant Slack messages and responds based strictly on actual channel history, avoiding any hallucination or fabricated answers.

**Target Use Cases:**

- Teams wanting quick summaries or insights from Slack conversations.
- Managers or members querying action items, decisions, or mentions historically.
- Automating Slack conversational analysis with AI assistance.

**Logical Blocks:**

- **1.1 Input Reception:** Receives natural language questions via a chat trigger.
- **1.2 Slack Data Retrieval:** Fetches relevant Slack channel history through Slack API.
- **1.3 AI Processing:** Uses LangChain’s agent with OpenAI to interpret the query and answer based solely on Slack history.
- **1.4 Documentation and Setup Guidance:** Sticky notes provide setup instructions, example questions, and contact info.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:** This block listens for incoming chat messages (natural language queries) that trigger the workflow.

**Nodes Involved:**

- Chat with Slack

**Node Details:**

- **Chat with Slack**  
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chatTrigger` — Receives chat input as trigger for the workflow.  
  - *Configuration:* Uses a webhook (ID: `d38a0072-420b-4de8-86b6-03a8f9a6e254`) to receive user questions. No additional options configured.  
  - *Expressions/Variables:* None explicitly configured; acts as entry point.  
  - *Connections:* Output connected to “Slack Channel Chatbot” node (main input).  
  - *Version Requirements:* Version 1.3 of the node.  
  - *Potential Failures:* Webhook errors, invalid payloads, timeout on input.  
  - *Sub-Workflow:* None.

---

#### 1.2 Slack Data Retrieval

**Overview:** Retrieves the full message history from a specific Slack channel to provide factual data for answering queries.

**Nodes Involved:**

- Slack History

**Node Details:**

- **Slack History**  
  - *Type & Role:* `n8n-nodes-base.slackTool` — Fetches channel history from Slack API.  
  - *Configuration:*  
    - Resource: `channel`  
    - Channel ID: `C04FXLG2YRJ` (named “general”)  
    - Operation: `history`  
    - Return all messages (`returnAll: true`)  
    - No filters applied, implying full history fetch.  
  - *Expressions/Variables:* Channel ID is selected via list mode (cached).  
  - *Connections:* Output connected to “Slack Channel Chatbot” node (ai_tool input).  
  - *Credentials:* Uses Slack OAuth2 API credential (`Slack account 11`).  
  - *Version Requirements:* v2.3 of the slackTool node.  
  - *Potential Failures:* Authentication failure, API rate limits, network timeouts, invalid channel ID.  
  - *Sub-Workflow:* None.

---

#### 1.3 AI Processing

**Overview:** Processes the natural language question and Slack history data using a LangChain agent powered by an OpenAI chat model, producing the final answer.

**Nodes Involved:**

- Slack Channel Chatbot
- OpenAI Chat Model

**Node Details:**

- **Slack Channel Chatbot**  
  - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` — Acts as the AI agent orchestrating the query resolution. Receives Slack history and user question as inputs.  
  - *Configuration:*  
    - System Message: Instructs the agent to be a helpful assistant that **only** uses Slack channel history data to answer questions, forbidding any invention.  
  - *Inputs:*  
    - AI language model input from “OpenAI Chat Model”  
    - AI tool input from “Slack History” (the fetched Slack messages)  
    - Main input from “Chat with Slack” (user question)  
  - *Outputs:* Returns AI-generated response to the chat trigger.  
  - *Version Requirements:* v2.2 of the LangChain agent node.  
  - *Potential Failures:* Language model API errors, logic errors in instruction, misinterpretation of Slack data, missing data edge cases.  
  - *Sub-Workflow:* None.

- **OpenAI Chat Model**  
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides OpenAI’s GPT-4o-mini model as the language model backend.  
  - *Configuration:*  
    - Model: `gpt-4o-mini` (an OpenAI GPT-4 variant)  
    - Default options without fine-tuning or temperature settings shown.  
  - *Input:* Connected to “Slack Channel Chatbot” (ai_languageModel input)  
  - *Credentials:* Uses OpenAI API credential (`OpenAi account 4`)  
  - *Version Requirements:* v1.2 of the node  
  - *Potential Failures:* API key invalid, billing issues, rate limits, model unavailability.  
  - *Sub-Workflow:* None.

---

#### 1.4 Documentation and Setup Guidance

**Overview:** Provides users with onboarding instructions, example queries, and contact information via sticky notes.

**Nodes Involved:**

- Sticky Note53
- Sticky Note4
- Sticky Note52
- Sticky Note50
- Sticky Note54

**Node Details:**

- **Sticky Note53**  
  - Lists example natural language queries users can ask, e.g., summaries, action items, mentions, blockers, shared files.  
  - No inputs or outputs.

- **Sticky Note4**  
  - Detailed setup instructions for OpenAI API key and Slack API app creation, including OAuth scopes and token setup.  
  - Includes links to OpenAI billing, Slack app creation, and n8n credential configuration.  
  - No inputs or outputs.

- **Sticky Note52**  
  - Focused on Slack API connection instructions: scopes needed, installation, token retrieval, and n8n credential configuration.  
  - No inputs or outputs.

- **Sticky Note50**  
  - High-level overview describing the workflow’s purpose: chat with Slack channel history using AI, emphasizing truthful answers using only channel data.  
  - No inputs or outputs.

- **Sticky Note54**  
  - Summarizes OpenAI connection setup steps: API key retrieval, billing, and credential setup in n8n.  
  - No inputs or outputs.

---

### 3. Summary Table

| Node Name           | Node Type                                     | Functional Role                        | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                     |
|---------------------|-----------------------------------------------|-------------------------------------|---------------------|-----------------------|------------------------------------------------------------------------------------------------|
| Chat with Slack     | @n8n/n8n-nodes-langchain.chatTrigger          | Entry point, receive user queries    | -                   | Slack Channel Chatbot  |                                                                                                |
| Slack History       | n8n-nodes-base.slackTool                       | Fetch Slack channel message history  | -                   | Slack Channel Chatbot  |                                                                                                |
| Slack Channel Chatbot | @n8n/n8n-nodes-langchain.agent                | AI agent processing and answering    | Chat with Slack, Slack History, OpenAI Chat Model | -             |                                                                                                |
| OpenAI Chat Model    | @n8n/n8n-nodes-langchain.lmChatOpenAi         | OpenAI language model backend        | -                   | Slack Channel Chatbot  |                                                                                                |
| Sticky Note53        | n8n-nodes-base.stickyNote                      | Example questions for users          | -                   | -                     | “- “Give me a 5-bullet summary of the last 24 hours.”  - “What action items were assigned, and to whom?”  - “List open questions that haven’t been answered yet.”  - “Who was mentioned most this week?”  - “Summarize decisions from the last sprint planning.”  - “Show messages with the word ‘blocker’ from the past 2 days.”  - “What files/links were shared today?”” |
| Sticky Note4         | n8n-nodes-base.stickyNote                      | Setup instructions for OpenAI & Slack | -                   | -                     | “## ⚙️ Setup Instructions 1️⃣ Set Up OpenAI Connection … [OpenAI Platform](https://platform.openai.com/api-keys) … 2️⃣ Connect Slack API … [Slack API](https://api.slack.com/apps) … Contact info for Robert Breen at ynteractive.com” |
| Sticky Note52        | n8n-nodes-base.stickyNote                      | Slack API connection setup steps     | -                   | -                     | “### 2️⃣ Connect Slack API 1. Create an app → <https://api.slack.com/apps> … 5. In the **Slack History** node, select your Slack credential and the **Channel ID** to read” |
| Sticky Note50        | n8n-nodes-base.stickyNote                      | Workflow overview and purpose        | -                   | -                     | “# 💬 Slack Channel Chatbot (n8n + OpenAI) Chat with a Slack channel using AI. … The assistant **only** answers from the channel’s actual messages—no guessing.” |
| Sticky Note54        | n8n-nodes-base.stickyNote                      | OpenAI connection setup summary      | -                   | -                     | “### 1️⃣ Set Up OpenAI Connection 1. Go to [OpenAI Platform](https://platform.openai.com/api-keys) … 4. Copy your API key into the **OpenAI credentials** in n8n” |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node:**  
   - Add node: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: “Chat with Slack”  
   - Configure webhook (auto-generated) to receive incoming chat messages. No additional parameters needed.  

2. **Create Slack History Node:**  
   - Add node: `n8n-nodes-base.slackTool`  
   - Name: “Slack History”  
   - Set resource to `channel`  
   - Set operation to `history`  
   - Set `channelId` to target channel (e.g., `C04FXLG2YRJ` for “general”) — select via list mode if available.  
   - Enable `returnAll` to fetch full history.  
   - Configure Slack OAuth2 API credentials:  
     - Create Slack App with needed scopes: `channels:history`, `groups:history`, `im:history`, `mpim:history`, `channels:read`, `groups:read`, `users:read`. Add `chat:write` if bot replies are needed.  
     - Install app, copy Bot User OAuth Token.  
     - In n8n, create new Slack OAuth2 credential and paste token.  
   - Attach Slack credentials to node.  

3. **Create OpenAI Chat Model Node:**  
   - Add node: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: “OpenAI Chat Model”  
   - Select model: `gpt-4o-mini` (or similar GPT-4 variant)  
   - Configure OpenAI API credentials:  
     - Obtain API key from OpenAI platform (ensure billing and funds are active).  
     - In n8n, create OpenAI credentials and paste API key.  
   - Attach OpenAI credentials to node.  

4. **Create Slack Channel Chatbot Node:**  
   - Add node: `@n8n/n8n-nodes-langchain.agent`  
   - Name: “Slack Channel Chatbot”  
   - Configure system message:  
     ```
     You are a helpful assistant. For all questions, get the slack channel history from the slack history tool. Do not make anything up. Use only data in the slack tool to answer questions.
     ```  
   - Connect inputs:  
     - Main input from “Chat with Slack” (user query)  
     - `ai_tool` input from “Slack History” (Slack messages)  
     - `ai_languageModel` input from “OpenAI Chat Model” (language model)  

5. **Connect Nodes:**  
   - “Chat with Slack” (main output) → “Slack Channel Chatbot” (main input)  
   - “Slack History” (output named `ai_tool`) → “Slack Channel Chatbot” (input named `ai_tool`)  
   - “OpenAI Chat Model” (output named `ai_languageModel`) → “Slack Channel Chatbot” (input named `ai_languageModel`)  

6. **Add Sticky Notes for Documentation:**  
   - Add sticky notes containing setup instructions, example questions, and overview as per original content for user assistance.

7. **Activate Workflow:**  
   - Save and activate the workflow.  
   - Test by sending webhook requests containing natural language questions.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Detailed setup steps for OpenAI API key generation and billing activation.                                                                                                                                                                                                                                                                                                                                     | [OpenAI Platform](https://platform.openai.com/api-keys), [OpenAI Billing](https://platform.openai.com/settings/organization/billing/overview) |
| Slack API app creation instructions, including necessary OAuth scopes for reading channel history and posting messages.                                                                                                                                                                                                                                                                                         | [Slack API Apps](https://api.slack.com/apps)                                                                 |
| Example natural language questions demonstrating the variety of queries supported by the chatbot.                                                                                                                                                                                                                                                                                                              | Sticky note content in workflow                                                                              |
| Contact information for workflow customization assistance: Robert Breen, ynteractive.com, LinkedIn profile available.                                                                                                                                                                                                                                                                                          | Email: robert@ynteractive.com; LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/; https://ynteractive.com |
| Workflow purpose summary emphasizing AI answers strictly based on Slack channel messages to avoid hallucination.                                                                                                                                                                                                                                                                                               | Workflow overview sticky note                                                                                |

---

**Disclaimer:** The provided text is extracted solely from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected material. All manipulated data is public and lawful.