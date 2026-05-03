News Research and Sentiment Analysis AI Agent with Gemini and SearXNG

https://n8nworkflows.xyz/workflows/news-research-and-sentiment-analysis-ai-agent-with-gemini-and-searxng-5286


# News Research and Sentiment Analysis AI Agent with Gemini and SearXNG

### 1. Workflow Overview

This workflow, titled **"Stock Sentiment Analysis"**, is an AI-powered automation designed to research and analyze the latest financial news related to specific companies or stock tickers. It targets financial analysts, investors, or automated trading systems that require up-to-date market sentiment insights derived from recent news.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input in the form of chat messages containing company names or stock tickers.
- **1.2 Research Agent Processing:** Uses AI agents and web search tools to gather and summarize recent news articles relevant to the input.
- **1.3 Sentiment Analysis Agent:** Analyzes the summarized news and produces an investor-oriented sentiment report.
- **1.4 Auxiliary Tools:** Includes support nodes such as date retrieval and language model selection to assist the agents.
- **1.5 Setup and Documentation:** Provides instructions and configuration notes via a sticky note node.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by listening for incoming chat messages. It serves as the entry point where user queries specifying a company or stock ticker are received.

- **Nodes Involved:**  
  - *When chat message received*

- **Node Details:**

  | Node Name                 | Details                                                                                     |
  |---------------------------|---------------------------------------------------------------------------------------------|
  | When chat message received| **Type:** Chat Trigger (LangChain) <br> **Role:** Listens for chat input to trigger workflow.<br>**Configuration:** Default parameters, no additional options.<br>**Inputs:** External chat interface webhook.<br>**Outputs:** User chat text forwarded to "Research Agent".<br>**Edge Cases:** Webhook connectivity issues, malformed chat input, no message received.<br>**Version:** 1.1 |

---

#### 1.2 Research Agent Processing

- **Overview:**  
  This block executes the core research task by retrieving the current date, performing web searches, and summarizing recent news articles related to the user query. It uses an AI agent configured with a financial research assistant prompt.

- **Nodes Involved:**  
  - *get_current_date*  
  - *web_search*  
  - *Research Agent*

- **Node Details:**

  | Node Name        | Details                                                                                           |
  |------------------|-------------------------------------------------------------------------------------------------|
  | get_current_date | **Type:** DateTime Tool <br> **Role:** Provides the current date to define the research time window.<br>**Configuration:** Manual description "use this tool to find out the current date and time."<br>**Inputs:** None.<br>**Outputs:** Current date/time data.<br>**Edge Cases:** System clock errors, timezone inconsistencies.<br>**Version:** 2 |
  | web_search       | **Type:** SearXNG Tool (LangChain) <br> **Role:** Performs web searches using a self-hosted SearXNG instance.<br>**Configuration:** Language set to English; uses SearXNG API credentials.<br>**Inputs:** Search queries provided by "Research Agent".<br>**Outputs:** Search result summaries.<br>**Edge Cases:** API authentication failure, network timeouts, empty search results.<br>**Version:** 1 |
  | Research Agent   | **Type:** LangChain Agent <br> **Role:** Coordinates research by invoking date and web search tools, processes results.<br>**Configuration:** System message defines it as a financial research assistant tasked with finding and summarizing news from the past 7 days. It instructs usage of date retrieval and two separate web searches (company name and ticker).<br>**Inputs:** Receives chat input from "When chat message received", and tools "get_current_date" and "web_search".<br>**Outputs:** Consolidated news summaries forwarded to Sentiment Analysis Agent.<br>**Edge Cases:** Expression evaluation errors, incomplete search data, ambiguous input, tool invocation failures.<br>**Version:** 2 |

---

#### 1.3 Sentiment Analysis Agent

- **Overview:**  
  This block analyzes the news summaries produced by the Research Agent to determine overall investor sentiment and justify the evaluation with specific references to the news content.

- **Nodes Involved:**  
  - *Sentiment Analysis Agent*  
  - *Gemini 2.5 Flash* (Language Model used by both Research and Sentiment Analysis Agents)

- **Node Details:**

  | Node Name               | Details                                                                                                                                                                                                                                                                                                                                                     |
  |-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Sentiment Analysis Agent| **Type:** LangChain Agent <br> **Role:** Analyzes news summary text to classify sentiment (Positive, Negative, Neutral) from an investor perspective.<br>**Configuration:** System message instructs to read all news, determine sentiment, and justify with references. Input text dynamically constructed with user query and latest news summaries.<br>**Inputs:** News summary from Research Agent, user question text.<br>**Outputs:** Sentiment classification and detailed justification.<br>**Edge Cases:** NLP model misinterpretation, insufficient news data, incomplete justification.<br>**Version:** 2 |
  | Gemini 2.5 Flash        | **Type:** Google Gemini Language Model Node <br> **Role:** Provides AI-generated language understanding and generation capabilities for both agents.<br>**Configuration:** Uses model "models/gemini-2.5-flash".<br>**Inputs:** Connected as language model to both Research Agent and Sentiment Analysis Agent.<br>**Outputs:** AI-generated text responses.<br>**Credentials:** Google PaLM API credentials.<br>**Edge Cases:** API rate limits, authentication errors, model unavailability.<br>**Version:** 1 |

---

#### 1.4 Auxiliary Tools

- **Overview:**  
  Support nodes provide necessary data or configuration, such as the current date and language model selection, enabling the main AI agents to function effectively.

- **Nodes Involved:**  
  - *get_current_date* (already covered in 1.2)  
  - *Gemini 2.5 Flash* (already covered in 1.3)

- **Node Details:**  
  (Details incorporated above in respective blocks.)

---

#### 1.5 Setup and Documentation

- **Overview:**  
  Enables users to understand configuration steps and usage instructions directly within the n8n canvas via a sticky note.

- **Nodes Involved:**  
  - *Sticky Note*

- **Node Details:**

  | Node Name   | Details                                                                                                                     |
  |-------------|-----------------------------------------------------------------------------------------------------------------------------|
  | Sticky Note | **Type:** Sticky Note <br> **Role:** Provides setup instructions and configuration guidance for the workflow.<br>**Content Highlights:**<br>1. Selecting the language model (default: Google Gemini).<br>2. Setting up credentials for LLM and SearXNG.<br>3. Customizing Research and Sentiment Analysis agents' prompts.<br>4. Testing the workflow via built-in chat interface.<br>**Position:** Off to the side for visibility.<br>**Edge Cases:** None (informational only).<br>**Version:** 1 |

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                 | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                          |
|-------------------------|----------------------------------|--------------------------------|------------------------------|------------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger           | Entry point for user input      | (external webhook)            | Research Agent               | See setup instructions in Sticky Note node                                                        |
| Research Agent          | LangChain Agent                   | Conducts research and summarization | When chat message received; get_current_date; web_search | Sentiment Analysis Agent       | See setup instructions in Sticky Note node                                                        |
| get_current_date         | DateTime Tool                    | Provides current date/time      | None                         | Research Agent               |                                                                                                |
| web_search               | SearXNG Tool                    | Performs web search             | Research Agent (tool invocation) | Research Agent               | Requires SearXNG API credentials setup as per Sticky Note                                          |
| Sentiment Analysis Agent | LangChain Agent                   | Analyzes sentiment of news     | Research Agent               | (end of workflow)            | See setup instructions in Sticky Note node                                                        |
| Gemini 2.5 Flash         | Google Gemini Language Model     | Provides AI language model support | (connected as AI model to Research Agent and Sentiment Analysis Agent) | Research Agent, Sentiment Analysis Agent | Requires Google PaLM API credentials setup as per Sticky Note                                      |
| Sticky Note              | Sticky Note                      | Provides setup and usage instructions | None                         | None                         | Contains detailed workflow setup and configuration steps                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a **"LangChain Chat Trigger"** node named **"When chat message received"**.  
   - Leave parameters default. This node listens for user chat input to trigger the workflow.

2. **Add the DateTime Tool Node**  
   - Add a **"DateTime Tool"** node named **"get_current_date"**.  
   - Set description type to manual, and add description: "use this tool to find out the current date and time."  
   - No input connections required.

3. **Configure the Web Search Node**  
   - Add a **"LangChain SearXNG Tool"** node named **"web_search"**.  
   - Set language option to English.  
   - Under credentials, select or create a **SearXNG API credential** connected to your self-hosted SearXNG instance.  
   - This node will perform web searches called by the Research Agent.

4. **Create the Research Agent Node**  
   - Add a **"LangChain Agent"** node named **"Research Agent"**.  
   - Set the system message prompt as follows (customize to your needs):  
     ```
     You are a financial research assistant. Your task is to find and summarize the latest news for a given company name or stock ticker.

     Instructions:

     First, use the get_current_date tool to determine the date range for the last 7 days.
     Perform two separate web searches using the web_search tool: one with the company's full name and one with its stock ticker. Use the date range in your search queries to filter for news from the last week only.

     After reviewing the search results, provide a summary of each news article. The summary should cover the key events, announcements, and market sentiment discussed in the articles.
     ```
   - Connect inputs:  
     - Main input from **"When chat message received"** (user query).  
     - Tool inputs from **"get_current_date"** and **"web_search"** nodes.  
   - Connect its AI language model input to the **Gemini 2.5 Flash** node (next step).

5. **Add the Google Gemini Language Model Node**  
   - Add a **"LangChain Google Gemini"** node named **"Gemini 2.5 Flash"**.  
   - Set model to "models/gemini-2.5-flash".  
   - Configure credentials with your **Google PaLM API account**.  
   - Connect its output as the AI language model for both **Research Agent** and **Sentiment Analysis Agent** nodes.

6. **Create the Sentiment Analysis Agent Node**  
   - Add a **"LangChain Agent"** node named **"Sentiment Analysis Agent"**.  
   - Set the system message prompt:  
     ```
     You are a Financial Sentiment Analyst. You will be provided with a news summary. Your task is to analyze this summary and categorize its sentiment from an investor's perspective.

     Instructions:
     1. Read all the news.
     2. Determine the overall sentiment (Positive, Negative, or Neutral) based on the potential impact on the company's financial performance, stock price, or market reputation.
     3. Write a justification that references specific points from the news summaries.

     Output Format:
     Sentiment: {sentiment_category}
     Justification: {Your detailed justification}
     ```
   - Set the input text to use expressions:  
     ```
     =User asked about: {{ $('When chat message received').item.json.chatInput }}

     Latest news:
     {{ $json.output }}
     ```
   - Connect its main input from the **Research Agent** output.  
   - Connect its AI language model input from the **Gemini 2.5 Flash** node.

7. **Add a Sticky Note for Setup Instructions**  
   - Add a **Sticky Note** node, position it clearly on the canvas.  
   - Paste the setup steps and configuration notes as content for user reference.

8. **Connect Nodes According to Workflow Logic**  
   - "When chat message received" main output → "Research Agent" main input  
   - "get_current_date" tool output → "Research Agent" tool input  
   - "web_search" tool output → "Research Agent" tool input  
   - "Research Agent" main output → "Sentiment Analysis Agent" main input  
   - "Gemini 2.5 Flash" AI language model output → connect to "Research Agent" and "Sentiment Analysis Agent" AI language model inputs

9. **Validate Credentials**  
   - Ensure Google PaLM API credentials are valid and authorized for "Gemini 2.5 Flash".  
   - Ensure SearXNG API credentials are valid and point to an accessible SearXNG instance.

10. **Test the Workflow**  
    - Use n8n's built-in chat interface to send a message with a company name or stock ticker.  
    - Verify the workflow triggers, performs searches, summarizes news, and outputs sentiment analysis.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is pre-configured with Google Gemini as the default language model but can be adapted to other LLMs by adjusting the corresponding nodes.         | See Sticky Note node in workflow for detailed setup instructions.                                   |
| The SearXNG node requires a self-hosted instance or accessible API endpoint, ensuring web searches are private and customizable.                               | https://searxng.org/                                                                                 |
| Using multiple agents (Research and Sentiment Analysis) allows modular, scalable AI workflows with clear separation of concerns.                              | n8n official docs on LangChain & AI agents                                                          |
| Ensure API rate limits and quotas are monitored to prevent workflow disruptions, especially for Google PaLM and SearXNG.                                      | Google PaLM API docs: https://developers.generativeai.google/api/guides/overview                    |
| The system messages in agent nodes should be customized carefully to reflect precise business logic and expected output formats for better accuracy.           |                                                                                                    |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.