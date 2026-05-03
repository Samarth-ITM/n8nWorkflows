Stock Technical Analysis with Google Gemini

https://n8nworkflows.xyz/workflows/stock-technical-analysis-with-google-gemini-2735


# Stock Technical Analysis with Google Gemini

### 1. Workflow Overview

The **"Sell: Stock Vision"** workflow is designed to provide AI-powered technical analysis for equity stocks and cryptocurrencies. It integrates financial chart retrieval, AI-driven pattern recognition, and news sentiment analysis to generate actionable trading insights. The workflow is ideal for traders and analysts seeking automated, data-driven market evaluations.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input (stock or crypto symbol) via a chat trigger.
- **1.2 Data Preparation:** Processes and formats input data for API requests.
- **1.3 Chart Retrieval:** Fetches detailed financial charts from TradingView via Chart-Img.com API.
- **1.4 AI Analysis:** Uses Google Gemini Chat Model and Langchain AI Agent with buffer memory to analyze charts and generate insights.
- **1.5 News & Sentiment Analysis:** Integrates SerpAPI to gather relevant news and sentiment data to complement technical analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving chat messages containing stock or cryptocurrency symbols. It triggers the entire analysis process.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type & Role:* Chat trigger node; entry point for user input via chat interface.  
    - *Configuration:* Listens for incoming chat messages; configured with a webhook ID for external integration.  
    - *Expressions/Variables:* Captures message content as input symbol.  
    - *Input/Output:* No input; outputs message data to "Edit Fields".  
    - *Version:* 1.1  
    - *Edge Cases:* Missing or malformed input messages; webhook connectivity issues.  
    - *Sub-workflow:* None.

#### 2.2 Data Preparation

- **Overview:**  
  This block formats and prepares the input symbol for API requests, ensuring compatibility with downstream nodes.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**

  - **Edit Fields**  
    - *Type & Role:* Set node; modifies or sets fields in the incoming data.  
    - *Configuration:* Likely extracts and formats the stock/crypto symbol from the chat message for API usage.  
    - *Expressions/Variables:* Uses expressions to map input message to API parameters.  
    - *Input/Output:* Input from "When chat message received"; output to "TradingView Chart".  
    - *Version:* 3.4  
    - *Edge Cases:* Incorrect symbol formatting; empty or null values.  
    - *Sub-workflow:* None.

#### 2.3 Chart Retrieval

- **Overview:**  
  Retrieves detailed financial charts for the specified symbol using the Chart-Img.com API, enabling visual data for AI analysis.

- **Nodes Involved:**  
  - TradingView Chart

- **Node Details:**

  - **TradingView Chart**  
    - *Type & Role:* HTTP Request node; fetches chart images/data from external API.  
    - *Configuration:* Configured to call Chart-Img.com API with the formatted symbol and parameters for chart type, timeframe, and indicators.  
    - *Expressions/Variables:* Uses input symbol from "Edit Fields" to build API request URL and parameters.  
    - *Input/Output:* Input from "Edit Fields"; output to "AI Agent".  
    - *Version:* 4.2  
    - *Edge Cases:* API key invalid or missing; network timeouts; invalid symbol causing API errors; rate limiting.  
    - *Sub-workflow:* None.

#### 2.4 AI Analysis

- **Overview:**  
  This block performs the core AI-driven technical analysis using Google's Gemini Chat Model and Langchain AI Agent, enhanced with buffer memory for context retention.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Window Buffer Memory

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model node; provides advanced AI chat capabilities using Google's Gemini model.  
    - *Configuration:* Set up with credentials for Google Gemini; used as the language model backend for the AI Agent.  
    - *Expressions/Variables:* Receives prompts including chart data and instructions for analysis.  
    - *Input/Output:* Input from "Window Buffer Memory"; output to "AI Agent".  
    - *Version:* 1  
    - *Edge Cases:* Authentication failures; API quota limits; response delays.  
    - *Sub-workflow:* None.

  - **Window Buffer Memory**  
    - *Type & Role:* Memory node; maintains conversation context for the AI Agent to enable continuous and coherent analysis.  
    - *Configuration:* Configured to store a windowed buffer of recent interactions.  
    - *Expressions/Variables:* Stores and retrieves conversation history.  
    - *Input/Output:* Input from "AI Agent"; output to "Google Gemini Chat Model".  
    - *Version:* 1.3  
    - *Edge Cases:* Memory overflow; loss of context if buffer size is too small.  
    - *Sub-workflow:* None.

  - **AI Agent**  
    - *Type & Role:* Langchain AI Agent node; orchestrates AI model calls and tool usage to generate analysis.  
    - *Configuration:* Configured to use "Google Gemini Chat Model" as language model, "Window Buffer Memory" for memory, and "SerpAPI" as an AI tool.  
    - *Expressions/Variables:* Receives chart data and news input; outputs analysis and recommendations.  
    - *Input/Output:* Inputs from "TradingView Chart" (main), "Window Buffer Memory" (ai_memory), "Google Gemini Chat Model" (ai_languageModel), and "SerpAPI" (ai_tool); output is final AI analysis.  
    - *Version:* 1.7  
    - *Edge Cases:* Model response errors; tool integration failures; memory sync issues.  
    - *Sub-workflow:* None.

#### 2.5 News & Sentiment Analysis

- **Overview:**  
  Enhances technical analysis by fetching relevant news and sentiment data for the symbol using SerpAPI.

- **Nodes Involved:**  
  - SerpAPI

- **Node Details:**

  - **SerpAPI**  
    - *Type & Role:* AI tool node; queries SerpAPI to retrieve news articles and sentiment information.  
    - *Configuration:* Configured with SerpAPI credentials and query parameters based on the input symbol.  
    - *Expressions/Variables:* Uses symbol from AI Agent context to fetch relevant news.  
    - *Input/Output:* Input from "AI Agent" as an AI tool; output back to "AI Agent".  
    - *Version:* 1  
    - *Edge Cases:* API key issues; rate limits; no news found for symbol.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                  | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                      |
|-------------------------|-----------------------------------|--------------------------------|-----------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger            | Input Reception (chat trigger) | None                        | Edit Fields                |                                                                                                 |
| Edit Fields             | Set                               | Data Preparation (format input) | When chat message received  | TradingView Chart          |                                                                                                 |
| TradingView Chart       | HTTP Request                      | Chart Retrieval (fetch charts)  | Edit Fields                 | AI Agent                  |                                                                                                 |
| AI Agent                | Langchain AI Agent                | AI Analysis (core AI logic)     | TradingView Chart, Window Buffer Memory, Google Gemini Chat Model, SerpAPI | None (final output)         |                                                                                                 |
| Google Gemini Chat Model | Langchain LM Chat Google Gemini  | AI Analysis (language model)    | Window Buffer Memory        | AI Agent                  |                                                                                                 |
| Window Buffer Memory    | Langchain Memory Buffer Window   | AI Analysis (context memory)    | AI Agent                   | Google Gemini Chat Model   |                                                                                                 |
| SerpAPI                 | Langchain Tool SerpAPI            | News & Sentiment Analysis       | AI Agent (ai_tool)          | AI Agent                  |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: Langchain Chat Trigger  
   - Configure webhook to receive chat messages containing stock or crypto symbols.  
   - No input connections. Output to "Edit Fields".

2. **Create "Edit Fields" node**  
   - Type: Set  
   - Configure to extract and format the symbol from the chat message payload.  
   - Connect input from "When chat message received". Output to "TradingView Chart".

3. **Create "TradingView Chart" node**  
   - Type: HTTP Request  
   - Configure to call Chart-Img.com API:  
     - Set method to GET or POST as per API spec.  
     - Use expressions to insert the formatted symbol into the request URL or body.  
     - Add necessary headers including API key for Chart-Img.com.  
   - Connect input from "Edit Fields". Output to "AI Agent".

4. **Create "Google Gemini Chat Model" node**  
   - Type: Langchain LM Chat Google Gemini  
   - Configure with Google Gemini API credentials.  
   - No direct input connection; will be connected via memory node.

5. **Create "Window Buffer Memory" node**  
   - Type: Langchain Memory Buffer Window  
   - Configure buffer size to retain recent conversation context (default or adjusted as needed).  
   - Connect input from "AI Agent" (ai_memory). Output to "Google Gemini Chat Model".

6. **Create "SerpAPI" node**  
   - Type: Langchain Tool SerpAPI  
   - Configure with SerpAPI API key.  
   - Set query parameters to search news related to the input symbol.  
   - Connect input from "AI Agent" (ai_tool). Output back to "AI Agent".

7. **Create "AI Agent" node**  
   - Type: Langchain AI Agent  
   - Configure to use:  
     - Language Model: connect to "Google Gemini Chat Model" (ai_languageModel).  
     - Memory: connect to "Window Buffer Memory" (ai_memory).  
     - Tools: connect to "SerpAPI" (ai_tool).  
   - Connect main input from "TradingView Chart".  
   - Output is the final AI-powered analysis result.

8. **Connect all nodes as per above connections** ensuring data flows from input reception through data preparation, chart retrieval, AI analysis, and news integration.

9. **Set credentials** for:  
   - Chart-Img.com API (used in "TradingView Chart" HTTP Request node).  
   - Google Gemini API (used in "Google Gemini Chat Model").  
   - SerpAPI (used in "SerpAPI" node).

10. **Test the workflow** by triggering a chat message with a valid stock or crypto symbol and verify the AI-generated analysis output.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|------------------------------------------------------------------------------------------------|----------------------------------------|
| Video walkthrough of the workflow setup and usage                                              | https://youtu.be/9fR4qNMT5LM            |
| Workflow designed for traders and analysts seeking AI-powered technical analysis               | Workflow description                    |
| Requires API keys for Chart-Img.com, SerpAPI, and Google Gemini                                | Setup instructions                      |
| Uses Langchain AI Agent framework with Google Gemini as the language model and buffer memory   | AI architecture details                 |

---

This documentation provides a complete understanding of the "Sell: Stock Vision" workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.