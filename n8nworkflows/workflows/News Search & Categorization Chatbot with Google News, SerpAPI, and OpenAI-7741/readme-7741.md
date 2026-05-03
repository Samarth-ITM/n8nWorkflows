News Search & Categorization Chatbot with Google News, SerpAPI, and OpenAI

https://n8nworkflows.xyz/workflows/news-search---categorization-chatbot-with-google-news--serpapi--and-openai-7741


# News Search & Categorization Chatbot with Google News, SerpAPI, and OpenAI

### 1. Workflow Overview

This workflow implements a **News Search & Categorization Chatbot** that integrates Google News search results via SerpApi with AI-powered summarization and categorization using OpenAI models. It is designed to receive user chat input queries, search for relevant news articles, extract and aggregate key information from the results, and then employ an AI agent to group news titles into thematic categories and return a concise, categorized summary with representative news links.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Query Trigger:** Receives user chat input via a chat trigger node.
- **1.2 News Search via SerpApi:** Uses the Google News search API to retrieve relevant news articles based on the user query.
- **1.3 Data Transformation & Aggregation:** Splits out individual news results and aggregates relevant fields (titles and links) for AI processing.
- **1.4 AI Categorization & Response Generation:** Uses an OpenAI-powered agent node (LangChain integration) to categorize news titles into groups and generate a summarized response.
- **1.5 AI Memory Management:** Maintains conversational context using a memory buffer node keyed by session ID.

This modular design enables extensibility, such as adding filters, alternative news sources, or output channels.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Query Trigger

- **Overview:**  
  Captures user input via a chat interface webhook to initiate the news search and summarization process.

- **Nodes Involved:**  
  - `Sample Chatbot`

- **Node Details:**

  - **Sample Chatbot**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chatTrigger` — Entry point for receiving user chat inputs.  
    - *Configuration:* Uses a webhook with ID `d38a0072-420b-4de8-86b6-03a8f9a6e254` to listen for incoming chat messages. No additional options configured.  
    - *Expressions/Variables:* `$json.chatInput` is expected as the incoming user message field.  
    - *Input/Output:* No input connections; outputs to `Google_news search`.  
    - *Edge Cases:* Webhook unavailability, invalid or missing chat input fields.  
    - *Sub-workflow:* None.

#### 2.2 News Search via SerpApi

- **Overview:**  
  Queries Google News for articles matching the user's chat input using the SerpApi node.

- **Nodes Involved:**  
  - `Google_news search`

- **Node Details:**

  - **Google_news search**  
    - *Type & Role:* `n8n-nodes-serpapi.serpApi` — Calls SerpApi's Google News search endpoint.  
    - *Configuration:*  
      - Query parameter `q` set dynamically to user input: `={{ $json.chatInput }}`.  
      - Operation set to `google_news`.  
      - Additional field `gl` set to `"us"` to localize results to the United States.  
    - *Credentials:* Uses SerpApi account credential (`rp4rU4y52sWmbV5y`).  
    - *Input/Output:* Input from `Sample Chatbot`; output to `Split Out Links`.  
    - *Edge Cases:* API key invalid or expired, SerpApi service downtime, empty or malformed query, rate limits.  
    - *Sub-workflow:* None.

#### 2.3 Data Transformation & Aggregation

- **Overview:**  
  Processes the raw news results array by splitting it into individual news items and then aggregates key fields (title and link) into a simplified unified list for AI consumption.

- **Nodes Involved:**  
  - `Split Out Links`  
  - `Combine into one field`

- **Node Details:**

  - **Split Out Links**  
    - *Type & Role:* `n8n-nodes-base.splitOut` — Splits array field into separate items.  
    - *Configuration:* Splits on `news_results` field from SerpApi output.  
    - *Input/Output:* Input from `Google_news search`; output to `Combine into one field`.  
    - *Edge Cases:* Missing or empty `news_results` field, null values.  
    - *Sub-workflow:* None.

  - **Combine into one field**  
    - *Type & Role:* `n8n-nodes-base.aggregate` — Merges multiple fields across split items into two aggregated lists (titles and links).  
    - *Configuration:* Aggregates `title` and `link` fields from each news result, merging into lists.  
    - *Input/Output:* Input from `Split Out Links`; output to `Chat with Google News`.  
    - *Edge Cases:* Null or missing title/link in some items, empty aggregation results.  
    - *Sub-workflow:* None.

#### 2.4 AI Categorization & Response Generation

- **Overview:**  
  Uses a LangChain AI agent to analyze the aggregated news titles and links, categorize them into 5 groups based on the user's original query, and return one representative title and link per category.

- **Nodes Involved:**  
  - `Chat with Google News`  
  - `OpenAI Chat Model6`

- **Node Details:**

  - **Chat with Google News**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` — AI agent node orchestrating the categorization task.  
    - *Configuration:*  
      - Text prompt constructed dynamically:  
        ```
        news titles: {{ $json.title }} 
        Links: {{ $json.link }} 
        question: {{ $('Sample Chatbot').item.json.chatInput }}
        ```  
      - System message instructs the model to group news titles into 5 categories and return one URL and title per category.  
      - Prompt type set to `define` (custom prompt).  
    - *Input/Output:* Inputs aggregated titles/links and original query; output is the categorized summary response.  
    - *Edge Cases:* Model timeout, malformed prompt, incomplete input data.  
    - *Sub-workflow:* None.

  - **OpenAI Chat Model6**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Language model node providing OpenAI GPT-5 Nano chat completions.  
    - *Configuration:*  
      - Model: `"gpt-5-nano"` (latest version) with caching enabled.  
      - No additional options set.  
    - *Credentials:* Uses OpenAI API key credential (`4l6TDfLZVFS24g3X`).  
    - *Input/Output:* Receives prompts from `Chat with Google News` as AI language model, outputs completions back to it.  
    - *Edge Cases:* API key invalid, billing issues, request timeouts, rate limits.  
    - *Sub-workflow:* None.

#### 2.5 AI Memory Management

- **Overview:**  
  Maintains a short-term conversation memory buffer keyed by session ID to provide context continuity across chat interactions.

- **Nodes Involved:**  
  - `Simple Memory2`

- **Node Details:**

  - **Simple Memory2**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Keeps a sliding window of conversation history.  
    - *Configuration:*  
      - Session key set dynamically from `Sample Chatbot` session ID: `={{ $('Sample Chatbot').item.json.sessionId }}`.  
      - Session ID type set to `customKey`.  
    - *Input/Output:* Connected as `ai_memory` input to `Chat with Google News`.  
    - *Edge Cases:* Missing or inconsistent session IDs, memory overflow if window size not configured.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name           | Node Type                                   | Functional Role                       | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                                                                                                                                                                                                                                                                                  |
|---------------------|---------------------------------------------|-------------------------------------|-----------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sample Chatbot       | @n8n/n8n-nodes-langchain.chatTrigger        | Receives user chat input via webhook| None                  | Google_news search        |                                                                                                                                                                                                                                                                                                                                                              |
| Google_news search   | n8n-nodes-serpapi.serpApi                    | Performs Google News search via SerpApi API | Sample Chatbot         | Split Out Links           |                                                                                                                                                                                                                                                                                                                                                              |
| Split Out Links      | n8n-nodes-base.splitOut                      | Splits array of news results into individual items | Google_news search     | Combine into one field    |                                                                                                                                                                                                                                                                                                                                                              |
| Combine into one field | n8n-nodes-base.aggregate                    | Aggregates news titles and links into lists | Split Out Links        | Chat with Google News     |                                                                                                                                                                                                                                                                                                                                                              |
| Chat with Google News| @n8n/n8n-nodes-langchain.agent               | Categorizes news titles and generates summary | Combine into one field, Simple Memory2, OpenAI Chat Model6 | None (final output)       |                                                                                                                                                                                                                                                                                                                                                              |
| OpenAI Chat Model6   | @n8n/n8n-nodes-langchain.lmChatOpenAi        | Provides GPT model completions for AI agent | Chat with Google News  | Chat with Google News     |                                                                                                                                                                                                                                                                                                                                                              |
| Simple Memory2       | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation memory buffer | None                  | Chat with Google News     |                                                                                                                                                                                                                                                                                                                                                              |
| Sticky Note59        | n8n-nodes-base.stickyNote                    | Informational note on workflow purpose | None                  | None                      | # 📰 Chat with Google News  \n\nThis workflow lets you search Google News via **SerpApi**, then summarize and categorize results with **OpenAI**.  \n                                                                                                                                                                                                       |
| Sticky Note16        | n8n-nodes-base.stickyNote                    | Setup instructions for SerpApi & OpenAI | None                  | None                      | \n## ⚙️ Setup Instructions  \n\n### 1️⃣ Set Up SerpApi Connection  \n1. Create a free account at [SerpApi](https://serpapi.com/)  \n2. Copy your **API Key** from the dashboard  \n3. In **n8n** → **Credentials → New → SerpApi**  \n   - Paste your API Key → **Save**  \n4. In the workflow, select your SerpApi credential in the **Google News Search** node.  \n\n---\n\n### 2️⃣ Set Up OpenAI Connection  \n1. Go to [OpenAI Platform](https://platform.openai.com/api-keys)  \n2. Navigate to [OpenAI Billing](https://platform.openai.com/settings/organization/billing/overview)  \n3. Add funds to your billing account  \n4. Copy your API key into the **OpenAI credentials** in n8n  \n\n\n\n## 📬 Contact  \nNeed help customizing (e.g., tracking specific industries, filtering sources, or auto-sending results to Slack/Email)?  \n\n📧 **robert@ynteractive.com**  \n🔗 **[Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/)**  \n🌐 **[ynteractive.com](https://ynteractive.com)**\n |
| Sticky Note27        | n8n-nodes-base.stickyNote                    | OpenAI credential setup reminder   | None                  | None                      | ### 2️⃣ Set Up OpenAI Connection\n1. Go to [OpenAI Platform](https://platform.openai.com/api-keys)  \n2. Navigate to [OpenAI Billing](https://platform.openai.com/settings/organization/billing/overview)  \n3. Add funds to your billing account  \n4. Copy your API key into the **OpenAI credentials** in n8n  |
| Sticky Note28        | n8n-nodes-base.stickyNote                    | SerpApi credential setup reminder   | None                  | None                      | ### 1️⃣ Set Up SerpApi Connection  \n1. Create a free account at [SerpApi](https://serpapi.com/)  \n2. Copy your **API Key** from the dashboard  \n3. In **n8n** → **Credentials → New → SerpApi**  \n   - Paste your API Key → **Save**  \n4. In the workflow, select your SerpApi credential in the **Google News Search** node.  \n\n---\n |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the Chat Trigger Node:**  
   - Node Name: `Sample Chatbot`  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook ID (auto-generated or custom) to receive chat input.  
   - No extra parameters needed. This node triggers the workflow on incoming chat messages.

3. **Add SerpApi Node for Google News Search:**  
   - Node Name: `Google_news search`  
   - Type: `n8n-nodes-serpapi.serpApi`  
   - Set operation to `google_news`.  
   - Set query parameter `q` to expression: `={{ $json.chatInput }}` (dynamic from chat input).  
   - Add `gl` (Geo Location) parameter with value `"us"` (to localize news results to the U.S.).  
   - Connect `Sample Chatbot` output to this node’s input.  
   - Credentials: Create or select SerpApi credential with your SerpApi API key.

4. **Add Split Out Node:**  
   - Node Name: `Split Out Links`  
   - Type: `n8n-nodes-base.splitOut`  
   - Configure to split on field: `news_results` (which contains the array of news articles from SerpApi).  
   - Connect output of `Google_news search` to input of this node.

5. **Add Aggregate Node to Combine Titles and Links:**  
   - Node Name: `Combine into one field`  
   - Type: `n8n-nodes-base.aggregate`  
   - Configure to aggregate fields:  
     - `title`  
     - `link`  
   - Set option `mergeLists` to TRUE so output contains two arrays (titles and links).  
   - Connect output of `Split Out Links` to input of this node.

6. **Add LangChain Memory Buffer Node:**  
   - Node Name: `Simple Memory2`  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set session key to expression: `={{ $('Sample Chatbot').item.json.sessionId }}` to maintain session context per user.  
   - Set session ID type to `customKey`.

7. **Add LangChain AI Chat Model Node:**  
   - Node Name: `OpenAI Chat Model6`  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `"gpt-5-nano"` (or latest available model).  
   - No special options required.  
   - Credentials: Create or select OpenAI credentials with valid API key and billing.

8. **Add LangChain Agent Node for Categorization:**  
   - Node Name: `Chat with Google News`  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set prompt type to `define`.  
   - Configure prompt text with expression:  
     ```
     news titles: {{ $json.title }} Links: {{ $json.link }} question: {{ $('Sample Chatbot').item.json.chatInput }}
     ```  
   - Set system message:  
     ```
     Take in the list of titles and the original search term. Group into 5 different categories, and send back 1 url's and title for each category they might be interested in.
     ```  
   - Connect inputs:  
     - Main input from `Combine into one field`.  
     - AI memory input from `Simple Memory2`.  
     - AI language model input from `OpenAI Chat Model6`.

9. **Connect nodes as follows:**  
   - `Sample Chatbot` → `Google_news search` → `Split Out Links` → `Combine into one field` → `Chat with Google News`  
   - `Simple Memory2` connected as AI memory to `Chat with Google News`  
   - `OpenAI Chat Model6` connected as AI language model to `Chat with Google News`

10. **Test the workflow by triggering the webhook with a chat input, e.g., “Latest technology news”.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow allows customization for tracking specific industries, filtering news sources, or auto-sending results to Slack and Email. Contact robert@ynteractive.com for assistance.                                                                                                                                                                                                        | Contact email and professional support                                                         |
| Setup instructions for SerpApi and OpenAI credentials including links to account creation and billing pages.                                                                                                                                                                                                                                                                                  | [SerpApi](https://serpapi.com/), [OpenAI API Keys](https://platform.openai.com/api-keys), [OpenAI Billing](https://platform.openai.com/settings/organization/billing/overview) |
| Workflow uses LangChain integration nodes for chat triggers, memory buffers, and agent orchestration with OpenAI GPT models.                                                                                                                                                                                                                                                                  | n8n LangChain nodes documentation                                                               |
| The SerpApi Google News search is localized to US by default (parameter `gl` = "us"). This can be modified for other regions as needed.                                                                                                                                                                                                                                                     | SerpApi Google News documentation                                                               |
| The AI categorization groups news into exactly 5 categories, returning one representative news item per category. Prompt and system message can be customized for different grouping or summarization styles.                                                                                                                                                                                | Prompt design best practices                                                                      |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected material. All processed data is legal and publicly available.