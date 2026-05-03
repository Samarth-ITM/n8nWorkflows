Claude 3.7 Sonnet AI Chatbot Agent with Anthropic Web Search and Think Functions

https://n8nworkflows.xyz/workflows/claude-3-7-sonnet-ai-chatbot-agent-with-anthropic-web-search-and-think-functions-4036


# Claude 3.7 Sonnet AI Chatbot Agent with Anthropic Web Search and Think Functions

### 1. Workflow Overview

This workflow implements a conversational AI chatbot agent leveraging the **Claude 3.7 Sonnet** language model from Anthropic, enhanced with advanced capabilities for real-time web search and internal reasoning via the **Think** function. It is designed to provide accurate, context-aware, and ethically guided responses to user queries, including up-to-date factual information fetched from the internet.

**Target Use Cases:**  
- Interactive chatbot on websites requiring dynamic, factual answers.  
- Customer support or information desks needing continuous conversational context and memory.  
- Scenarios demanding safe, ethical AI interactions with access to live web data.

**Logical Blocks:**

- **1.1 Input Reception:** Captures incoming user messages via a webhook trigger.  
- **1.2 AI Agent Core:** Processes user queries, decides on internal reasoning or external web search, and manages conversation logic according to embedded system prompts.  
- **1.3 Language Model Execution:** Uses Claude 3.7 Sonnet to generate conversational responses based on inputs and tools results.  
- **1.4 Memory Buffer:** Maintains conversational context to enable continuity across messages.  
- **1.5 External Tools:**  
  - **Web Search Tool:** Executes real-time web queries to fetch current information.  
  - **Think Tool:** Performs internal reasoning or thought logging without external data fetching.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives chat messages from users via a webhook, initiating the workflow.

- **Nodes Involved:**  
  - *When chat message received*

- **Node Details:**  
  - **When chat message received**  
    - *Type:* LangChain Chat Trigger node  
    - *Role:* Entry point capturing incoming chat messages through a uniquely configured webhook URL.  
    - *Configuration:* Default options; webhook ID assigned for external integration.  
    - *Input/Output:* No input; outputs the received user message to the AI Agent node.  
    - *Edge Cases:* Potential webhook connectivity issues, invalid payload formats, or delayed triggers. Requires secure webhook URL management to avoid unauthorized calls.  
    - *Version:* 1.1

---

#### 2.2 AI Agent Core

- **Overview:**  
Central decision-making node that interprets queries, manages tool invocation (web search, think), and applies system prompt rules for ethical and functional behavior.

- **Nodes Involved:**  
  - *AI Agent*

- **Node Details:**  
  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Orchestrates the chatbot logic, analyzing queries to determine whether to use website context, invoke web search, or think internally.  
    - *Configuration:*  
      - System message defines strict guidelines on chatbot behavior, ethical constraints, and how to use tools: web_search and Think.  
      - Specifies usage limits (max 3 web searches per query).  
      - Enforces output format to exclude internal thoughts or tool usage details.  
    - *Key Expressions:* The system message is embedded as a complex multiline string with instructions and placeholders (e.g., `{web_search_question}`) for tool invocation.  
    - *Input:* Receives user messages from the chat trigger.  
    - *Output:* Sends commands to the Anthropic Chat Model, invokes the web_search and Think tools, and updates the Simple Memory buffer.  
    - *Edge Cases:*  
      - Misinterpretation of query leading to unnecessary or missing web searches.  
      - Tool invocation failures or API rate limits.  
      - Expression errors in system message placeholders.  
    - *Version:* 1.9

---

#### 2.3 Language Model Execution

- **Overview:**  
Executes the Anthropic Claude 3.7 Sonnet language model to generate chatbot responses based on user input, system prompt, and tool outputs.

- **Nodes Involved:**  
  - *Anthropic Chat Model*

- **Node Details:**  
  - **Anthropic Chat Model**  
    - *Type:* LangChain Anthropic Chat Model node  
    - *Role:* Generates natural language responses using Claude 3.7 Sonnet model.  
    - *Configuration:*  
      - Model: `claude-3-7-sonnet-20250219` selected from the list.  
      - `maxTokensToSample` set to 1024 to allow sufficiently detailed responses.  
    - *Credentials:* Uses Anthropic API credentials for authentication.  
    - *Input:* Receives processed prompts and tool outputs from AI Agent.  
    - *Output:* Outputs final text responses to AI Agent.  
    - *Edge Cases:*  
      - API authentication errors.  
      - Response latency or timeouts.  
      - Token limits exceeded or truncated responses.  
    - *Version:* 1.3

---

#### 2.4 Memory Buffer

- **Overview:**  
Maintains conversation history to provide context retention and continuity across multiple user interactions.

- **Nodes Involved:**  
  - *Simple Memory*

- **Node Details:**  
  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window node  
    - *Role:* Stores and retrieves recent conversational exchanges to simulate memory in the AI agent.  
    - *Configuration:* Default parameters implying a sliding window buffer storing recent messages.  
    - *Input:* Receives data from AI Agent.  
    - *Output:* Provides memory context back to AI Agent for prompt generation.  
    - *Edge Cases:*  
      - Memory overflow if too many interactions occur without clearing.  
      - Possible stale or irrelevant context if buffer is too large or small.  
    - *Version:* 1.3

---

#### 2.5 External Tools

- **Overview:**  
Provide additional capabilities to the AI Agent: real-time web search for current facts and an internal “Think” tool for reasoning or caching thoughts.

- **Nodes Involved:**  
  - *web_search*  
  - *Think*

- **Node Details:**  

  - **web_search**  
    - *Type:* HTTP Request Tool node  
    - *Role:* Performs POST requests to Anthropic’s API endpoint `/v1/messages` for web search queries.  
    - *Configuration:*  
      - URL: `https://api.anthropic.com/v1/messages`  
      - Method: POST  
      - JSON body dynamically composed with the user’s search query `{web_search_question}` embedded.  
      - Headers include `anthropic-version: 2023-06-01` and `content-type: application/json` (note a minor typo in header value `"application/jso"` likely meant `"application/json"`).  
      - Authentication via Anthropic API credentials and HTTP Header Auth node.  
      - Tool description specifies it is for web search use only.  
    - *Input:* Receives search query from AI Agent.  
    - *Output:* Returns search result data to AI Agent.  
    - *Edge Cases:*  
      - API quota limits or auth failures.  
      - Network timeouts.  
      - Malformed query or unexpected API response format.  
    - *Version:* 4.2

  - **Think**  
    - *Type:* LangChain Tool Think node  
    - *Role:* Enables internal reasoning, thought logging, or caching without querying external sources.  
    - *Configuration:* Default; no parameters specified.  
    - *Input:* Receives “think” commands from AI Agent.  
    - *Output:* Returns appended thought logs to AI Agent.  
    - *Edge Cases:* Minimal, mostly logical issues if reasoning steps are ill-formed.  
    - *Version:* 1

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                           | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                               |
|----------------------------|-------------------------------|-----------------------------------------|-----------------------------|-------------------------|----------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger         | Receives incoming chat messages         | None                        | AI Agent                |                                                                                                          |
| AI Agent                   | LangChain Agent                | Central logic, query analysis, tool orchestration | When chat message received, Anthropic Chat Model, Simple Memory, web_search, Think | Anthropic Chat Model, Simple Memory, web_search, Think |                                                                                                          |
| Anthropic Chat Model       | LangChain Anthropic Chat Model | Generates AI text responses              | AI Agent                    | AI Agent                |                                                                                                          |
| Simple Memory              | LangChain Memory Buffer Window | Maintains conversation context          | AI Agent                    | AI Agent                |                                                                                                          |
| web_search                 | HTTP Request Tool              | Performs real-time web search            | AI Agent                    | AI Agent                | Use this tool to search on the web                                                                       |
| Think                      | LangChain Tool Think           | Performs internal reasoning/thought logging | AI Agent                    | AI Agent                |                                                                                                          |
| Sticky Note                | Sticky Note                   | Documentation note                       | None                        | None                    | ## Claude 3.7 Sonnet AI Agent with web search and Think functions<br>This workflow builds a conversational AI chatbot agent using **Claude 3.7 Sonnet** model with the new . It enhances standard LLM capabilities with Anthropic’s features: **Web Search** and **Think**. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node:**  
   - Type: LangChain Chat Trigger  
   - Configure webhook with a unique URL to receive chat messages.  
   - Leave default options.  

2. **Create "AI Agent" node:**  
   - Type: LangChain Agent  
   - Set the system message to:  
     ```
     You are an AI-powered chatbot assistant for a website. Your primary function is to assist users by answering their queries and providing relevant information. You have access to a web_search tool that allows you to browse the internet for up-to-date information. Here's how you should operate:

     1. Website Information:
     Familiarize yourself with this information about the website you're assisting. Use this as context for user interactions.

     2. Web Search Tool:
     You have access to a web_search tool that can browse the internet. To use it, write the variable {web_search_question}. The tool will return relevant search results.

     3. Handling User Queries:
     When a user asks a question, follow these steps:
     a) Analyze the query to determine if it's related to the website or requires external information.
     b) If the query is about the website, use the provided website information to answer.
     c) If external information is needed, use the web_search tool to find relevant data.

     4. Using web_search:
     - Use web_search for factual, current information that isn't provided in the website info.
     - Formulate clear, concise search queries.
     - If the first search doesn't yield useful results, refine your query and search again.
     - Limit searches to a maximum of three per user query to maintain efficiency.

     5. Using Think:
     - Using Think tool to think about something. It will not obtain new information or change the database, but just append the thought to the log. Use it when complex reasoning or some cache memory is needed.

     6. Formulating Responses:
     - Begin with information from the website if relevant.
     - Incorporate web search results to provide up-to-date, accurate information.
     - Summarize findings concisely and coherently.
     - If you're unsure or can't find reliable information, be honest about limitations.

     7. Ethical Considerations:
     - Respect user privacy. Don't ask for or store personal information.
     - Provide factual information. Avoid speculation or unverified claims.
     - If asked about controversial topics, strive for a balanced, neutral response.
     - Don't engage in or encourage illegal activities.

     8. Output Format:
     Do not include your thought process, web searches, or any other tags in the final output.
     ```  
   - Connect input from "When chat message received."  
   - Leave other options default.  

3. **Create "Anthropic Chat Model" node:**  
   - Type: LangChain Anthropic Chat Model  
   - Select model: `claude-3-7-sonnet-20250219` (Claude 3.7 Sonnet)  
   - Set `maxTokensToSample` to 1024.  
   - Assign Anthropic API credentials with appropriate permissions.  
   - Connect input from AI Agent node (ai_languageModel output).  
   - Connect output to AI Agent node (ai_languageModel input).  

4. **Create "Simple Memory" node:**  
   - Type: LangChain Memory Buffer Window  
   - Use default parameters (sliding window memory).  
   - Connect input from AI Agent node (ai_memory output).  
   - Connect output to AI Agent node (ai_memory input).  

5. **Create "web_search" node:**  
   - Type: HTTP Request Tool  
   - Method: POST  
   - URL: `https://api.anthropic.com/v1/messages`  
   - Headers:  
     - `anthropic-version`: `2023-06-01`  
     - `content-type`: `application/json`  
   - Body (JSON):  
     ```json
     {
       "model": "claude-3-7-sonnet-latest",
       "max_tokens": 1024,
       "messages": [
         {
           "role": "user",
           "content": "{web_search_question}"
         }
       ],
       "tools": [
         {
           "type": "web_search_20250305",
           "name": "web_search",
           "max_uses": 5
         }
       ]
     }
     ```  
   - Authentication: Use Anthropic API credentials and corresponding HTTP Header Auth node.  
   - Connect input from AI Agent node (ai_tool output).  
   - Connect output to AI Agent node (ai_tool input).  

6. **Create "Think" node:**  
   - Type: LangChain Tool Think  
   - No additional parameters needed.  
   - Connect input from AI Agent node (ai_tool output).  
   - Connect output to AI Agent node (ai_tool input).  

7. **Connect all nodes:**  
   - "When chat message received" → "AI Agent" (main)  
   - "AI Agent" → "Anthropic Chat Model" (ai_languageModel)  
   - "Anthropic Chat Model" → "AI Agent" (ai_languageModel)  
   - "AI Agent" → "Simple Memory" (ai_memory)  
   - "Simple Memory" → "AI Agent" (ai_memory)  
   - "AI Agent" → "web_search" (ai_tool)  
   - "web_search" → "AI Agent" (ai_tool)  
   - "AI Agent" → "Think" (ai_tool)  
   - "Think" → "AI Agent" (ai_tool)  

8. **Credentials Setup:**  
   - Add and configure Anthropic API credentials to allow access to both chat and web search endpoints.  
   - Configure HTTP Header Auth credentials for the web_search node to add required headers.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow builds a conversational AI chatbot agent using Claude 3.7 Sonnet model enhanced with Anthropic’s Web Search and Think tools.                 | Workflow description and overview                                                                   |
| The system prompt enforces ethical rules: privacy preservation, factual accuracy, neutrality, and forbids illegal content generation.                   | System message in AI Agent node configuration                                                       |
| For support or customization contact: info@n3w.it or visit https://www.linkedin.com/in/davideboizza/                                                   | Contact information                                                                                  |
| Web search tool uses Anthropic API version 2023-06-01. Ensure API keys have access to this endpoint and version.                                        | web_search node HTTP request headers                                                                |
| Minor header typo in `content-type` header as "application/jso" should be corrected to "application/json" in production setups.                        | Potential header misconfiguration in web_search node                                                |
| Memory buffer uses sliding window strategy; consider adjusting size for long conversations to avoid stale context or excessive memory usage.            | Simple Memory node default behavior                                                                 |

---

**Disclaimer:**  
The text provided is extracted solely from an automated n8n workflow utilizing legal and publicly available data and complies with all applicable content policies. It contains no illegal, offensive, or protected content.