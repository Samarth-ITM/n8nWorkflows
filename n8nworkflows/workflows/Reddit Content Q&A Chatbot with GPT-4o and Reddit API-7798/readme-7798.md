Reddit Content Q&A Chatbot with GPT-4o and Reddit API

https://n8nworkflows.xyz/workflows/reddit-content-q-a-chatbot-with-gpt-4o-and-reddit-api-7798


# Reddit Content Q&A Chatbot with GPT-4o and Reddit API

### 1. Workflow Overview

This workflow implements a **Reddit Content Q&A Chatbot** that leverages the **Reddit API** and **OpenAI GPT-4o** language model to provide intelligent answers to user questions based on real Reddit posts. The chatbot listens for incoming chat messages, queries recent posts from a specified subreddit (default `n8n`), and uses GPT-4o with a LangChain agent configured to incorporate Reddit data to generate relevant responses.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users to trigger the workflow.
- **1.2 Data Retrieval from Reddit:** Fetches recent posts from a specified subreddit using Reddit API credentials.
- **1.3 AI Processing:** Uses a LangChain agent configured with GPT-4o and the Reddit data tool to generate contextual answers.
- **1.4 Setup & Documentation:** Provides sticky notes with setup instructions and workflow purpose for user guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages to initiate the Q&A interaction.

- **Nodes Involved:**  
  - `When chat message received`

- **Node Details:**

  - **Node Name:** When chat message received  
    - **Type:** `@n8n/n8n-nodes-langchain.chatTrigger`  
    - **Technical Role:** Webhook-based trigger node designed to listen for chat messages from an integrated chat interface.  
    - **Configuration:** Default options, no filters or transformations applied.  
    - **Expressions/Variables:** None specified; triggers on every incoming message.  
    - **Input:** None (trigger).  
    - **Output:** Forwards the message to the `Reddit Chatbot` node.  
    - **Edge Cases:** Possible webhook connectivity issues, message format inconsistencies, or delayed message delivery.  
    - **Sub-workflow:** None.

---

#### 1.2 Data Retrieval from Reddit

- **Overview:**  
  Retrieves up to 10 recent posts from the `n8n` subreddit using OAuth2 authenticated Reddit API calls.

- **Nodes Involved:**  
  - `Get many posts in Reddit`

- **Node Details:**

  - **Node Name:** Get many posts in Reddit  
    - **Type:** `n8n-nodes-base.redditTool`  
    - **Technical Role:** API integration node to fetch Reddit posts.  
    - **Configuration:**  
      - Operation: `getAll` (fetch multiple posts)  
      - Subreddit: `n8n` (hardcoded)  
      - Limit: 10 posts per request  
      - No additional filters applied.  
    - **Credentials:** Uses `Reddit OAuth2 API` credential configured with client ID, secret, and OAuth login flow.  
    - **Input:** Not directly triggered by input data but triggered by the `Reddit Chatbot` agent as a tool.  
    - **Output:** Provides an array of Reddit posts to the LangChain agent.  
    - **Edge Cases:** OAuth token expiration, API rate limits, subreddit restrictions, network timeouts, empty or malformed response from Reddit API.  
    - **Sub-workflow:** None.

---

#### 1.3 AI Processing

- **Overview:**  
  Manages the user query by invoking a LangChain agent that uses GPT-4o as the language model and the Reddit API tool for data retrieval. The agent constructs an answer based on the fetched Reddit posts.

- **Nodes Involved:**  
  - `Reddit Chatbot`  
  - `OpenAI Chat Model`

- **Node Details:**

  - **Node Name:** OpenAI Chat Model  
    - **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - **Technical Role:** Provides GPT-4o language model inference for natural language generation.  
    - **Configuration:**  
      - Model: `gpt-4o`  
      - Default parameters, no special options enabled.  
    - **Credentials:** Uses an OpenAI API key credential stored in n8n.  
    - **Input:** Receives prompts and context from the `Reddit Chatbot` agent node.  
    - **Output:** Returns generated chat completions to the `Reddit Chatbot` node.  
    - **Edge Cases:** API key invalid or missing, exceeded quota or billing issues, network errors, model unavailability or errors.  
    - **Sub-workflow:** None.

  - **Node Name:** Reddit Chatbot  
    - **Type:** `@n8n/n8n-nodes-langchain.agent`  
    - **Technical Role:** LangChain agent node that orchestrates AI interaction combining language model and external tools.  
    - **Configuration:**  
      - System message instructs: "use the reddit tool to answer user questions."  
      - Configured to use `OpenAI Chat Model` as the language model and `Get many posts in Reddit` as the tool.  
    - **Input:** Receives user chat message from `When chat message received` node.  
    - **Output:** Sends final generated answer back to the chat interface or downstream processes.  
    - **Edge Cases:** Misinterpretation of user queries, tool integration failures, incomplete or empty Reddit data, timeout or API errors, malformed system prompt.  
    - **Sub-workflow:** None.

---

#### 1.4 Setup & Documentation

- **Overview:**  
  Contains sticky notes providing detailed instructions for setting up OpenAI and Reddit API credentials, workflow purpose, and contact information.

- **Nodes Involved:**  
  - `Sticky Note1`  
  - `Sticky Note25`  
  - `Sticky Note26`  
  - `Sticky Note56`

- **Node Details:**

  - **Sticky Note1:**  
    - Content: Overview describing the workflow purpose, enabling chat with Reddit using AI and Reddit API.  
    - Role: Informational, no direct functional impact.

  - **Sticky Note25:**  
    - Content: Step-by-step instructions to set up OpenAI API and Reddit OAuth2 API credentials, including links to relevant external sites.  
    - Role: User onboarding and setup guidance.

  - **Sticky Note26:**  
    - Content: Detailed Reddit API credential setup instructions, including creating a Reddit app, filling redirect URI, and configuring credentials in n8n.  
    - Role: Credential setup guidance.

  - **Sticky Note56:**  
    - Content: OpenAI connection setup instructions, including billing and API key acquisition.  
    - Role: Credential setup guidance.

- **Edge Cases:** Sticky notes do not affect workflow execution but are crucial for users to correctly configure external services.

---

### 3. Summary Table

| Node Name               | Node Type                                     | Functional Role                     | Input Node(s)                    | Output Node(s)                 | Sticky Note                                                                                                                                            |
|-------------------------|-----------------------------------------------|-----------------------------------|---------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger          | Captures incoming chat messages   | None                            | Reddit Chatbot                |                                                                                                                                                        |
| OpenAI Chat Model         | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Provides GPT-4o language model    | Reddit Chatbot (ai_languageModel) | Reddit Chatbot (ai_languageModel) |                                                                                                                                                        |
| Get many posts in Reddit  | n8n-nodes-base.redditTool                       | Fetches Reddit posts              | Reddit Chatbot (ai_tool)         | Reddit Chatbot (ai_tool)       |                                                                                                                                                        |
| Reddit Chatbot            | @n8n/n8n-nodes-langchain.agent                  | Orchestrates AI + Reddit tool     | When chat message received       | OpenAI Chat Model, Get many posts in Reddit |                                                                                                                                                        |
| Sticky Note1              | n8n-nodes-base.stickyNote                        | Workflow purpose & overview       | None                            | None                          | # 📋 Chat with Reddit (AI + Reddit API)\n\nThis workflow lets you **chat with Reddit** using OpenAI and the Reddit API. The chatbot pulls posts from a subreddit and uses GPT to answer your questions. |
| Sticky Note25             | n8n-nodes-base.stickyNote                        | Setup instructions for APIs       | None                            | None                          | ## ⚙️ Setup Instructions\n\n### 2️⃣ Set Up OpenAI Connection\n1. Go to [OpenAI Platform](https://platform.openai.com/api-keys) ... (see detailed content in 2.4) |
| Sticky Note26             | n8n-nodes-base.stickyNote                        | Reddit API setup instructions     | None                            | None                          | ### 2️⃣ Set Up Reddit API\n1. Go to [Reddit Apps](https://www.reddit.com/prefs/apps) ... (see detailed content in 2.4)                                |
| Sticky Note56             | n8n-nodes-base.stickyNote                        | OpenAI API setup instructions     | None                            | None                          | ### 2️⃣ Set Up OpenAI Connection\n1. Go to [OpenAI Platform](https://platform.openai.com/api-keys) ... (see detailed content in 2.4)                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add node: `When chat message received`  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configuration: Default options, no filters  
   - Purpose: Trigger the workflow when a chat message is received.

2. **Create the Reddit Data Fetch Node**  
   - Add node: `Get many posts in Reddit`  
   - Type: `n8n-nodes-base.redditTool`  
   - Parameters:  
     - Operation: `getAll`  
     - Subreddit: `n8n` (or any desired subreddit)  
     - Limit: 10  
   - Credentials: Configure a new `Reddit OAuth2 API` credential:  
     - Register a Reddit script app at https://www.reddit.com/prefs/apps  
     - Set Redirect URI (e.g., `http://localhost:8080` or your n8n URL)  
     - Enter Client ID and Secret in n8n credentials  
     - Authenticate via OAuth2 login  
   - Attach this credential to the node.

3. **Create the OpenAI Chat Model Node**  
   - Add node: `OpenAI Chat Model`  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Parameters:  
     - Model: `gpt-4o`  
   - Credentials: Configure OpenAI API key in n8n credentials (see https://platform.openai.com/api-keys)  
   - Attach the credential to this node.

4. **Create the LangChain Agent Node**  
   - Add node: `Reddit Chatbot`  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - System Message: `"use the reddit tool to answer user questions."`  
     - Configure two integrations:  
       - Language Model: Connect the `OpenAI Chat Model` node here  
       - Tool: Connect the `Get many posts in Reddit` node here  
   - Input Connection: Connect from `When chat message received` node main output.  
   - Output Connections: Connect the `OpenAI Chat Model` node as language model integration and `Get many posts in Reddit` as tool integration.

5. **Connect Nodes**  
   - Connect `When chat message received` → `Reddit Chatbot`  
   - Connect `Reddit Chatbot` → `OpenAI Chat Model` (ai_languageModel)  
   - Connect `Reddit Chatbot` → `Get many posts in Reddit` (ai_tool)

6. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add notes describing workflow purpose, setup instructions for OpenAI and Reddit API credentials, and contact info.  
   - Use the content from the sticky notes section for guidance.

7. **Test the Workflow**  
   - Trigger a chat message to verify the full flow: message received → Reddit posts fetched → GPT-4o generates response → response returned.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow lets you chat with Reddit using AI and Reddit API. Pulls posts from subreddit and uses GPT to answer questions.                                                                                                                  | Workflow purpose (Sticky Note1)                                                                                         |
| OpenAI API key setup: Visit [OpenAI Platform](https://platform.openai.com/api-keys), add billing funds, copy API key into n8n credentials.                                                                                                | Setup instructions (Sticky Note25, Sticky Note56)                                                                       |
| Reddit API setup: Create Reddit app at [Reddit Apps](https://www.reddit.com/prefs/apps), choose script type, set Redirect URI, create OAuth2 credentials in n8n, and authenticate.                                                        | Setup instructions (Sticky Note25, Sticky Note26)                                                                       |
| Contact for customization help: robert@ynteractive.com, LinkedIn [Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/), Website [ynteractive.com](https://ynteractive.com)                                                    | Support and customization contact                                                                                       |

---

**Disclaimer:** The provided text is generated from an automated n8n workflow. It complies with content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.