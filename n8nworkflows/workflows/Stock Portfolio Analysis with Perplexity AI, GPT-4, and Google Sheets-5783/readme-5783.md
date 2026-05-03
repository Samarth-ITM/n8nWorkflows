Stock Portfolio Analysis with Perplexity AI, GPT-4, and Google Sheets

https://n8nworkflows.xyz/workflows/stock-portfolio-analysis-with-perplexity-ai--gpt-4--and-google-sheets-5783


# Stock Portfolio Analysis with Perplexity AI, GPT-4, and Google Sheets

---

### 1. Workflow Overview

This n8n workflow, titled **"Dynamic Portfolio Advisor Stock Market News"**, is designed to deliver a personalized daily stock market intelligence briefing tailored to an investor’s portfolio holdings stored in Google Sheets. It integrates real-time market data and news fetched via Perplexity AI, leverages GPT-4 for summarization and analysis, and outputs actionable insights through Telegram messages.

**Target Use Cases:**
- Active investors seeking daily, portfolio-aware market updates.
- Financial advisors automating client briefings.
- Traders looking for AI-generated trade ideas based on recent market news.
- Finance content creators or analysts streamlining market research workflows.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a fixed time.
- **1.2 Data Retrieval:** Fetches current portfolio holdings from Google Sheets.
- **1.3 AI Processing & Memory:** Uses a LangChain agent combining GPT-4 and Perplexity AI for market news analysis, referencing portfolio data and session memory.
- **1.4 Market News Fetching:** Calls Perplexity AI to retrieve up-to-date stock market news.
- **1.5 Output Delivery:** Sends the compiled briefing to a Telegram chat.
- **1.6 Support Nodes:** Includes sticky notes for documentation and workflow clarity.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically every day at 10:00 AM.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* n8n-nodes-base.scheduleTrigger  
    - *Role:* Time-based trigger to start the workflow daily.  
    - *Configuration:* Set to trigger every day at 10:00 AM (hour 10).  
    - *Input/Output:* No input; output triggers the AI processing block.  
    - *Edge Cases:* Potential failure if n8n server time zone differs from expected or if the scheduler service is down.  
    - *Sticky Note:* “Scheduled Trigger” documentation node nearby.

---

#### 1.2 Data Retrieval

- **Overview:**  
  Reads the current stock portfolio holdings dynamically from Google Sheets to inform AI analysis.

- **Nodes Involved:**  
  - Portfolio Holdings (Google Sheets Tool)

- **Node Details:**

  - **Portfolio Holdings**  
    - *Type:* n8n-nodes-base.googleSheetsTool  
    - *Role:* Reads portfolio data from a Google Sheets document.  
    - *Configuration:* Uses Google OAuth2 credentials for authentication. No hardcoded values; dynamic read.  
    - *Input/Output:* Receives request from the AI agent node; outputs portfolio data for analysis.  
    - *Edge Cases:* Authentication failure, sheet permissions issues, empty or malformed data.  
    - *Credential:* Google Sheets OAuth2 API required.

---

#### 1.3 AI Processing & Memory

- **Overview:**  
  This core block runs an AI agent that combines GPT-4 chat capabilities, session memory, and external web search (Perplexity) tools to generate a daily market briefing contextualized by the user’s portfolio.

- **Nodes Involved:**  
  - Stock Market News & Analytics Agent (LangChain Agent)  
  - OpenAI Chat Model (GPT-4)  
  - Simple Memory (Buffer Window Memory)

- **Node Details:**

  - **Stock Market News & Analytics Agent**  
    - *Type:* @n8n/n8n-nodes-langchain.agent  
    - *Role:* Coordinates AI analysis using GPT-4, Perplexity search, and portfolio data.  
    - *Configuration:*  
      - Prompt instructs the agent to fetch a summary of the latest stock market news referencing today’s date dynamically.  
      - Uses Perplexity tool for web searches.  
      - Uses Google Sheets tool for portfolio holdings reference.  
      - System message guides the agent to generate a briefing including headlines, investment opportunities, risks, and trade suggestions.  
    - *Input/Output:* Receives trigger and memory; outputs composed briefing text.  
    - *Edge Cases:* API rate limits, data inconsistencies, prompt errors, memory session loss.  
    - *Sub-workflow:* Integrates Perplexity and Google Sheets as tools inside the agent.

  - **OpenAI Chat Model**  
    - *Type:* @n8n/n8n-nodes-langchain.lmChatOpenAi  
    - *Role:* Provides GPT-4 chat model capabilities to the agent.  
    - *Configuration:* Set to GPT-4 (model version "gpt-4.1").  
    - *Credential:* Requires valid OpenAI API key.  
    - *Input/Output:* Input from agent node; outputs language model completions.  
    - *Edge Cases:* Authentication errors, API outages, usage limits.

  - **Simple Memory**  
    - *Type:* @n8n/n8n-nodes-langchain.memoryBufferWindow  
    - *Role:* Maintains session context for the agent to provide coherent multi-turn conversations.  
    - *Configuration:* Uses workflow ID as session key with custom key session ID type.  
    - *Input/Output:* Connected as AI memory input to the agent.  
    - *Edge Cases:* Memory overflow, session resets, potential stale data.

---

#### 1.4 Market News Fetching

- **Overview:**  
  This node calls the Perplexity AI tool to perform real-time web research on the latest stock market news, feeding results back to the AI agent.

- **Nodes Involved:**  
  - Perplexity

- **Node Details:**

  - **Perplexity**  
    - *Type:* n8n-nodes-base.perplexityTool  
    - *Role:* Fetches recent stock market news using Perplexity AI’s “sonar-pro” model.  
    - *Configuration:* Sends messages dynamically generated from AI agent's context; can simplify output if requested.  
    - *Credential:* Requires Perplexity API key.  
    - *Input/Output:* Input from agent node (ai_tool channel); outputs news data back to the agent.  
    - *Edge Cases:* API limits, network failures, response delays, malformed requests.

---

#### 1.5 Output Delivery

- **Overview:**  
  Sends the finalized daily briefing text to the user’s Telegram chat.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - *Type:* n8n-nodes-base.telegram  
    - *Role:* Sends messages to a specified Telegram chat.  
    - *Configuration:*  
      - Chat ID must be set by user ("Insert Chat ID" placeholder).  
      - Message text is the AI agent’s output field `output`.  
    - *Credential:* Requires Telegram API credentials.  
    - *Input/Output:* Input from AI agent’s main output; no outputs.  
    - *Edge Cases:* Invalid chat ID, Telegram API downtime, message size limits.

---

#### 1.6 Support Nodes (Sticky Notes)

- **Overview:**  
  Sticky notes provide in-workflow documentation and explanations for blocks and overall workflow context.

- **Nodes Involved:**  
  - Sticky Note (Scheduled Trigger)  
  - Sticky Note1 (Stock Market Fetch & Portfolio Analysis)  
  - Sticky Note2 (Telegram Output)  
  - Sticky Note3 (General workflow description and instructions)

- **Node Details:**

  - Provide high-level explanations, usage instructions, and links to external resources such as Marc's YouTube channel for building similar workflows.  
  - Sticky Note3 includes a detailed description of the workflow purpose, components, and use cases.

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                       | Input Node(s)                  | Output Node(s)                 | Sticky Note                         |
|----------------------------------|-------------------------------------|------------------------------------|-------------------------------|-------------------------------|-----------------------------------|
| Schedule Trigger                 | n8n-nodes-base.scheduleTrigger       | Initiates workflow daily at 10 AM  | None                          | Stock Market News & Analytics Agent | Scheduled Trigger                 |
| Stock Market News & Analytics Agent | @n8n/n8n-nodes-langchain.agent      | Core AI processing and summarization | Schedule Trigger, OpenAI Chat Model, Simple Memory, Portfolio Holdings, Perplexity | Telegram                      | Stock Market Fetch & Portfolio Analysis |
| OpenAI Chat Model               | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4 language model for AI agent  | Stock Market News & Analytics Agent | Stock Market News & Analytics Agent | Stock Market Fetch & Portfolio Analysis |
| Simple Memory                  | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains AI session memory       | Stock Market News & Analytics Agent | Stock Market News & Analytics Agent | Stock Market Fetch & Portfolio Analysis |
| Portfolio Holdings             | n8n-nodes-base.googleSheetsTool      | Fetches portfolio data from Google Sheets | Stock Market News & Analytics Agent | Stock Market News & Analytics Agent | Stock Market Fetch & Portfolio Analysis |
| Perplexity                    | n8n-nodes-base.perplexityTool          | Fetches recent stock market news    | Stock Market News & Analytics Agent | Stock Market News & Analytics Agent | Stock Market Fetch & Portfolio Analysis |
| Telegram                      | n8n-nodes-base.telegram                | Sends briefing message via Telegram | Stock Market News & Analytics Agent | None                          | Telegram Output                   |
| Sticky Note                   | n8n-nodes-base.stickyNote              | Documentation                      | None                          | None                          | Scheduled Trigger                 |
| Sticky Note1                  | n8n-nodes-base.stickyNote              | Documentation                      | None                          | None                          | Stock Market Fetch & Portfolio Analysis |
| Sticky Note2                  | n8n-nodes-base.stickyNote              | Documentation                      | None                          | None                          | Telegram Output                   |
| Sticky Note3                  | n8n-nodes-base.stickyNote              | Workflow overview & detailed description | None                          | None                          |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Dynamic Portfolio Advisor Stock Market News").

2. **Add a Schedule Trigger node:**
   - Type: `Schedule Trigger`
   - Parameters: Set interval to trigger daily at 10:00 AM (hour: 10).
   - No credentials required.

3. **Add Google Sheets node for portfolio holdings:**
   - Type: `Google Sheets Tool`
   - Set credentials: Use OAuth2 credentials for Google Sheets API.
   - Configure to read your portfolio data from a specific Google Sheet (e.g., sheet name, range).
   - No input connections yet.

4. **Add OpenAI Chat Model node:**
   - Type: `LangChain OpenAI Chat Model`
   - Credentials: Add OpenAI API key for GPT-4 access.
   - Model: Select GPT-4 (e.g., "gpt-4.1").
   - No input connections yet.

5. **Add Simple Memory node:**
   - Type: `LangChain Memory Buffer Window`
   - Parameters:  
     - Session Key: use expression `{{$workflow.id}}` to identify session.  
     - Session ID Type: Set to "customKey".
   - No input connections yet.

6. **Add Perplexity node:**
   - Type: `Perplexity Tool`
   - Credentials: Add Perplexity API key.
   - Configure model to "sonar-pro".
   - Set messages input to accept dynamic content from AI agent.
   - No input connections yet.

7. **Add LangChain Agent node (Stock Market News & Analytics Agent):**
   - Type: `LangChain Agent`
   - Parameters:
     - Prompt: Use a dynamic prompt instructing the agent to fetch daily stock market news referencing today’s date (`{{ $json['Readable date'] }}`).
     - System Message: Include detailed instructions for summarizing headlines, opportunities, risks, and trade suggestions referencing portfolio holdings.
     - Tools: Add Perplexity Tool and Google Sheets Tool (portfolio holdings) as AI tools.
   - Connect AI language model input to OpenAI Chat Model node.
   - Connect AI memory input to Simple Memory node.
   - Connect AI tool inputs to Perplexity node and Google Sheets node.
   - Connect main input from Schedule Trigger node.

8. **Connect all above nodes:**
   - Schedule Trigger → LangChain Agent (main input)
   - OpenAI Chat Model → LangChain Agent (ai_languageModel input)
   - Simple Memory → LangChain Agent (ai_memory input)
   - Portfolio Holdings → LangChain Agent (ai_tool input)
   - Perplexity → LangChain Agent (ai_tool input)

9. **Add Telegram node:**
   - Type: `Telegram`
   - Credentials: Add Telegram API credentials.
   - Parameters:  
     - Chat ID: enter your Telegram chat ID (replace placeholder "Insert Chat ID").  
     - Text: set to expression `{{ $json.output }}` to send the AI agent’s output.
   - Connect main input from LangChain Agent node’s main output.

10. **Add Sticky Notes (Optional for documentation):**
    - Add sticky notes near logical blocks with descriptions for clarity.

11. **Test the workflow:**
    - Run manually or wait for scheduled trigger.
    - Validate Google Sheets data is fetched correctly.
    - Confirm AI agent receives data and outputs appropriate briefing.
    - Confirm Telegram message delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 📊 Dynamic Portfolio Advisor – Daily Stock Market Intelligence with Google Sheets. This advanced AI-powered workflow combines real-time data from Perplexity AI, GPT-4, and Google Sheets to generate a personalized daily market briefing delivered via Telegram.     | Described in Sticky Note3 within the workflow                                                  |
| For detailed workflow creation and related advanced automation techniques, visit Marc’s YouTube channel: [https://www.youtube.com/@Automatewithmarc](https://www.youtube.com/@Automatewithmarc)                                                                   | External resource for learning workflow building                                                |
| The workflow is designed to be easily adaptable; the Telegram output node can be replaced with Slack, Email, Notion, or any other supported output tool in n8n.                                                                                                     | Flexibility note for integration                                                                 |
| Reminder: Ensure all API credentials (OpenAI, Google Sheets, Perplexity, Telegram) are set up correctly with appropriate permissions and quota to avoid runtime errors.                                                                                             | Best practice for credential management                                                         |
| The workflow uses dynamic date references (`{{ $json['Readable date'] }}`) for fetching and contextualizing daily news — ensure the date field is correctly set or generated upstream if modifying the workflow.                                                     | Important for maintaining accurate daily analysis                                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow built with n8n, a tool for integration and automation. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---