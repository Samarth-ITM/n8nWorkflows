Scrape & Summarize Industry News with Bright Data & OpenAI

https://n8nworkflows.xyz/workflows/scrape---summarize-industry-news-with-bright-data---openai-5977


# Scrape & Summarize Industry News with Bright Data & OpenAI

---
### 1. Workflow Overview

This workflow automates the process of scraping, analyzing, and sharing industry news focused on geopolitical developments, specifically around the Israel-Iran conflict and Hezbollah’s strategic shifts. The target use case is to monitor Reuters news articles related to Hezbollah and extract structured insights for a Trends Team, facilitating timely and automated geopolitical intelligence delivery.

The workflow is logically divided into three main blocks:

- **1.1 Start & Input:** Manual initiation and input of the Reuters news article URL.
- **1.2 AI Processing & Data Extraction:** An AI Agent combined with OpenAI and Bright Data services scrapes the article, extracts relevant data, and summarizes key trends.
- **1.3 Reporting & Sharing:** Automatically formats the insights and sends a detailed email to the Trends Team.

---

### 2. Block-by-Block Analysis

#### 1.1 Start & Input

**Overview:**  
This block handles the manual trigger to start the workflow and sets the input with the specific Reuters news URL about the Israel-Iran conflict, particularly Hezbollah.

**Nodes Involved:**  
- 🚦 Start Workflow (Manual Trigger)  
- 🔗 Enter Reuters News URL

**Node Details:**  

- **🚦 Start Workflow (Manual Trigger)**
  - **Type & Role:** Manual Trigger node; initiates the workflow when manually executed.
  - **Configuration:** No parameters; triggered by user interaction.
  - **Expressions/Variables:** None.
  - **Connections:** Outputs to the "Enter Reuters News URL" node.
  - **Edge Cases:** Workflow only runs when triggered; no automatic scheduling.
  
- **🔗 Enter Reuters News URL**
  - **Type & Role:** Set node; stores the URL of the Reuters article to be scraped.
  - **Configuration:** A string variable `reuterURL` set to a fixed URL about Hezbollah’s strategic considerations.
  - **Expressions/Variables:** Static URL value provided; could be replaced dynamically if needed.
  - **Connections:** Feeds the URL variable into the AI Agent for scraping.
  - **Edge Cases:** If the URL is invalid or unreachable, subsequent scraping will fail.

---

#### 1.2 AI Processing & Data Extraction

**Overview:**  
This core block uses an AI Agent to scrape the news article from Reuters, extract structured information about the article (title, authors, dates, content), and summarize geopolitical trends. It leverages OpenAI’s GPT-4.1-mini model and Bright Data’s MCP Client tool for robust data scraping and analysis.

**Nodes Involved:**  
- 🤖 Agent: Scrape Reuters News  
- OpenAI Chat Model  
- 🌐 MCP Client Tool  
- Auto-fixing Output Parser  
- OpenAI Chat Model1  
- 📦 Format Article as Structured Output

**Node Details:**  

- **🤖 Agent: Scrape Reuters News**
  - **Type & Role:** AI Agent node from LangChain integration; coordinates scraping and summarization.
  - **Configuration:**  
    - Prompt: Requests scraping of latest news about the Iran-Israel war from the provided Reuters URL.  
    - Output: Structured data including article titles, authors, dates, content, plus trend summaries.
  - **Expressions:** Uses `{{ $json.reuterURL }}` to dynamically incorporate the URL.  
  - **Connections:** Receives input from URL node; outputs structured data to email node.
  - **Edge Cases:**  
    - Errors if URL is unreachable or format changes.  
    - AI misinterpretation possible if article structure varies.  
    - Timeout or API quota limits from OpenAI or MCP Client.  
  - **Sub-workflow:** Integrates multiple AI sub-nodes internally.

- **OpenAI Chat Model**
  - **Type & Role:** OpenAI GPT-4.1-mini chat model; provides language model capabilities for the AI Agent.
  - **Configuration:** Model set to "gpt-4.1-mini"; no additional options.
  - **Connections:** Serves as the language model for the AI Agent node.
  - **Edge Cases:** API rate limits, authentication errors, or unexpected model output.

- **🌐 MCP Client Tool**
  - **Type & Role:** Bright Data MCP Client; securely scrapes web data from Reuters.
  - **Configuration:** Executes the `web_data_reuter_news` tool with parameters generated dynamically by AI.
  - **Connections:** Called by the AI Agent node as a scraping tool.
  - **Edge Cases:**  
    - Network failures, IP blocking, or scraping limits.  
    - Authentication failures with MCP Client API.

- **Auto-fixing Output Parser**  
  - **Type & Role:** LangChain Output Parser with auto-fixing; ensures AI output conforms to expected JSON schema.
  - **Configuration:** Default auto-fixing enabled.
  - **Connections:** Connects between structured output node and AI Agent for validation.
  - **Edge Cases:** Malformed AI output requiring correction; potential parsing errors.

- **OpenAI Chat Model1**  
  - **Type & Role:** Secondary OpenAI GPT-4.1-mini chat model; assists in refining output parsing.
  - **Configuration:** Same as the primary OpenAI Chat Model.
  - **Connections:** Tied to the Auto-fixing Output Parser for language model support.
  - **Edge Cases:** Same as primary OpenAI node.

- **📦 Format Article as Structured Output**
  - **Type & Role:** Structured Output Parser; formats the scraped and analyzed article into a JSON schema.
  - **Configuration:** Predefined JSON schema example includes platform, article metadata (title, authors, date), content summaries, and trend summaries.
  - **Connections:** Outputs structured data to the Auto-fixing Output Parser.
  - **Edge Cases:** Schema mismatch if AI output deviates; missing fields if scraping incomplete.

---

#### 1.3 Reporting & Sharing

**Overview:**  
This block composes and sends an email containing the article summary and key geopolitical insights to the Trends Team via Gmail.

**Nodes Involved:**  
- ✉️ Send Insights to Trends Team (Gmail)

**Node Details:**  

- **✉️ Send Insights to Trends Team (Gmail)**
  - **Type & Role:** Gmail node to send email messages.
  - **Configuration:**  
    - Recipient: shahkar.genai@gmail.com  
    - Subject: “Industry News: Hezbollah's Strategic Shift Amid Regional Tensions”  
    - Message Body: Formatted HTML content summarizing article title, authors, date, content points, and trend insights.  
    - Options: Attribution disabled.
  - **Credentials:** Uses OAuth2 Gmail account credential.
  - **Connections:** Receives structured summary from the AI Agent node.
  - **Edge Cases:**  
    - Email sending failures due to auth errors, quota limits, or connectivity.  
    - Formatting issues in email client if HTML malformed.

---

### 3. Summary Table

| Node Name                         | Node Type                                | Functional Role                               | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                  |
|----------------------------------|-----------------------------------------|-----------------------------------------------|----------------------------------|---------------------------------|-------------------------------------------------------------|
| 🚦 Start Workflow (Manual Trigger) | n8n-nodes-base.manualTrigger            | Manual trigger to start workflow              | —                                | 🔗 Enter Reuters News URL       | Simple input, manual control over workflow execution.       |
| 🔗 Enter Reuters News URL          | n8n-nodes-base.set                      | Sets the Reuters article URL to scrape        | 🚦 Start Workflow                | 🤖 Agent: Scrape Reuters News   | Paste the URL related to Israel-Iran conflict article.      |
| 🤖 Agent: Scrape Reuters News      | @n8n/n8n-nodes-langchain.agent         | AI Agent orchestrating scraping and analysis  | 🔗 Enter Reuters News URL        | ✉️ Send Insights to Trends Team | Uses AI to scrape and summarize Reuters news article.       |
| OpenAI Chat Model                  | @n8n/n8n-nodes-langchain.lmChatOpenAi  | Provides GPT-4.1-mini language model support  | — (internal to AI Agent)          | 🤖 Agent: Scrape Reuters News   | Processes instructions for scraping and analysis.           |
| 🌐 MCP Client Tool                 | n8n-nodes-mcp.mcpClientTool             | Scrapes article content using Bright Data MCP | — (internal to AI Agent)          | 🤖 Agent: Scrape Reuters News   | Scrapes detailed content securely and reliably.             |
| Auto-fixing Output Parser          | @n8n/n8n-nodes-langchain.outputParserAutofixing | Validates and auto-corrects AI output         | 📦 Format Article as Structured Output | 🤖 Agent: Scrape Reuters News   | Ensures valid structured JSON output from AI.               |
| OpenAI Chat Model1                 | @n8n/n8n-nodes-langchain.lmChatOpenAi  | Secondary GPT-4.1-mini model for output parsing | — (internal to Output Parser)     | Auto-fixing Output Parser       | Supports output parsing and fixing.                         |
| 📦 Format Article as Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Formats article and trend data to JSON schema | — (internal to AI Agent)          | Auto-fixing Output Parser       | Defines expected data schema for article and trends.        |
| ✉️ Send Insights to Trends Team (Gmail) | n8n-nodes-base.gmail                    | Sends formatted email with insights to team   | 🤖 Agent: Scrape Reuters News   | —                               | Automatically delivers actionable insights to Trends Team.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger Node**  
   - Add a **Manual Trigger** node named "🚦 Start Workflow (Manual Trigger)".  
   - No parameters required.

2. **Add Set Node for Input URL**  
   - Add a **Set** node named "🔗 Enter Reuters News URL".  
   - Create a string field `reuterURL` with value:  
     `https://www.reuters.com/world/middle-east/under-pressure-hezbollah-weighs-scaling-back-its-arsenal-2025-07-04/`  
   - Connect output from Manual Trigger to this node.

3. **Add AI Agent Node**  
   - Add a **LangChain AI Agent** node named "🤖 Agent: Scrape Reuters News".  
   - Set prompt to:  
     ```
     Scrape the latest news articles from the following news site about Iran and Israel war: 
     {{ $json.reuterURL }}
     Please return the article titles, authors, dates, and article content.
     and also summarize the trends
     ```  
   - Set prompt type to "define" and enable output parsing.  
   - Connect input from the "🔗 Enter Reuters News URL" node.

4. **Add OpenAI Chat Model Node**  
   - Add a **LangChain OpenAI Chat Model** node named "OpenAI Chat Model".  
   - Select model: `gpt-4.1-mini`.  
   - Connect it as the AI language model source for the AI Agent node.

5. **Add MCP Client Tool Node**  
   - Add an **MCP Client Tool** node named "🌐 MCP Client Tool".  
   - Set tool name to `web_data_reuter_news`.  
   - Set operation to `executeTool`.  
   - For tool parameters, leave dynamic (auto from AI).  
   - Connect as AI tool provider to the AI Agent node.  
   - Configure MCP Client API credentials.

6. **Add Structured Output Parser Node**  
   - Add a **LangChain Structured Output Parser** node named "📦 Format Article as Structured Output".  
   - Paste the provided JSON schema example (structured with platform, article metadata, content summary, trend summary).  
   - Connect output parser to feed into an Auto-fixing Output Parser node.

7. **Add Auto-fixing Output Parser Node**  
   - Add **LangChain Output Parser Autofixing** node named "Auto-fixing Output Parser".  
   - Connect input from the Structured Output Parser.  
   - Connect output back to AI Agent node as output parser.

8. **Add Secondary OpenAI Chat Model Node**  
   - Add another **LangChain OpenAI Chat Model** node named "OpenAI Chat Model1".  
   - Use the same model `gpt-4.1-mini`.  
   - Connect to support Auto-fixing Output Parser node.

9. **Add Gmail Send Node**  
   - Add a **Gmail** node named "✉️ Send Insights to Trends Team (Gmail)".  
   - Set recipient email: `shahkar.genai@gmail.com`.  
   - Compose email subject: "Industry News: Hezbollah's Strategic Shift Amid Regional Tensions".  
   - Compose message body as HTML, including article title, authors, date, content summary, and trend summary (use expressions referencing scraped data).  
   - Use Gmail OAuth2 credentials configured previously.  
   - Connect input from the AI Agent node’s main output.

10. **Link Workflow Connections**  
    - Connect nodes in this order:  
      Manual Trigger → Enter Reuters News URL → AI Agent  
      AI Agent uses OpenAI Chat Model and MCP Client Tool as internal tools.  
      AI Agent outputs to Gmail send node.

11. **Test & Validate**  
    - Execute the workflow manually.  
    - Verify the article is scraped, summarized, and email sent correctly.  
    - Monitor for errors related to API limits, URL access, or parsing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                       |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| I’ll receive a tiny commission if you join Bright Data through this link—thanks for fueling more free content!                   | https://get.brightdata.com/1tndi4600b25                              |
| Workflow assistance available – contact Yaron for support and explore more tips and videos on YouTube and LinkedIn.              | YouTube: https://www.youtube.com/@YaronBeen/videos                   |
|                                                                                                                                    | LinkedIn: https://www.linkedin.com/in/yaronbeen/                     |
| This workflow automates geopolitical news monitoring with AI and Bright Data’s scraping capabilities, delivering insights fast. | General overview from workflow sticky notes included in section 1-3.|

---

**Disclaimer:** The provided text and workflow are generated exclusively from an automated n8n workflow. The process respects all content policies, contains no illegal or offensive content, and handles only legal and public data.