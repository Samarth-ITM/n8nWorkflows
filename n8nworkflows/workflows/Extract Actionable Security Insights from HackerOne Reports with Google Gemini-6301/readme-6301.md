Extract Actionable Security Insights from HackerOne Reports with Google Gemini

https://n8nworkflows.xyz/workflows/extract-actionable-security-insights-from-hackerone-reports-with-google-gemini-6301


# Extract Actionable Security Insights from HackerOne Reports with Google Gemini

---

### 1. Workflow Overview

This workflow is designed to transform HackerOne security report URLs into concise, actionable technical insights tailored for advanced bug bounty hunters. It leverages a conversational chat interface to receive report URLs, fetches the report data from HackerOne’s public JSON API, and uses Google Gemini’s large language model (LLM) to analyze and extract unique, high-impact security findings.

The workflow is logically divided into these main blocks:

- **1.1 Chat Interface Input Reception:** A webhook-enabled chat node captures user input containing HackerOne report URLs.
- **1.2 AI Agent Coordination:** The LangChain agent node orchestrates the processing flow, directing the fetch of report data and summarizing insights.
- **1.3 HackerOne Report Fetching:** An HTTP request node retrieves the raw JSON report from the HackerOne API.
- **1.4 AI Language Model Processing:** The Google Gemini Chat Model node executes the natural language understanding and summarization based on the retrieved report content.
- **1.5 Output Delivery:** The agent node outputs a structured technical summary optimized for bug bounty usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Interface Input Reception

- **Overview:**  
  This block receives user input via a public webhook chat interface designed with custom pentester-themed CSS. Users send HackerOne report URLs (ending in `.json`), initiating the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type & Role:* LangChain Chat Trigger node; entry point capturing chat messages.  
    - *Configuration:*  
      - Public webhook enabled, allowing external use.  
      - Custom CSS applied for a pentester-friendly UI theme and branding ("Made with ❤️ by ethicxl").  
      - Initial greeting message: “Hey! Send your report’s link down below.”  
    - *Key expressions:* Uses `$json.chatInput` as input text for downstream processing.  
    - *Connections:* Outputs main data to “H1 report summarizer” node.  
    - *Edge Cases:*  
      - Invalid URLs or non-JSON links may cause downstream failures.  
      - Public access requires secure webhook management to avoid abuse.  
    - *Sub-workflow:* None.

---

#### 2.2 AI Agent Coordination

- **Overview:**  
  Acts as the central controller that receives chat input, instructs the HTTP fetch node to retrieve the report JSON, and processes the output with the language model for summarization.

- **Nodes Involved:**  
  - H1 report summarizer

- **Node Details:**

  - **H1 report summarizer**  
    - *Type & Role:* LangChain Agent node, orchestrates tool calls and AI summarization.  
    - *Configuration:*  
      - Receives input text from chat.  
      - System prompt defines a precise role: extract unique, high-impact technical insights only, avoiding general summaries.  
      - Requires exactly one call to the tool “GET H1 report” with the full URL.  
      - Output format:  
        1. Summary (BLUF)  
        2. Techniques (name, context, technique, impact)  
        3. Optional Pro Tip  
    - *Key expressions:* Uses `={{ $json.chatInput }}` to pass user input to the agent prompt.  
    - *Connections:*  
      - Calls “GET H1 report” HTTP tool automatically with URL.  
      - Receives report data and passes it for AI processing via the Google Gemini model.  
    - *Edge Cases:*  
      - If the HTTP request fails or returns invalid data, summarization will fail or produce incorrect output.  
      - Agent relies on correct prompt and model availability.  
    - *Sub-workflow:* None.

---

#### 2.3 HackerOne Report Fetching

- **Overview:**  
  Fetches the requested HackerOne report JSON from the public API endpoint based on the URL supplied by the AI agent.

- **Nodes Involved:**  
  - GET H1 report

- **Node Details:**

  - **GET H1 report**  
    - *Type & Role:* HTTP Request Tool node, performs GET requests.  
    - *Configuration:*  
      - URL dynamically injected by AI agent tool call (expression placeholder).  
      - Example URLs: `https://hackerone.com/reports/312543.json`  
      - No authentication or credentials required (public API).  
    - *Key expressions:* The URL is dynamically replaced by the AI agent call parameter.  
    - *Connections:* Outputs JSON report data back to “H1 report summarizer” agent node.  
    - *Edge Cases:*  
      - Network errors, 404 Not Found if report ID invalid.  
      - Rate limiting or API changes could disrupt data retrieval.  
    - *Sub-workflow:* None.

---

#### 2.4 AI Language Model Processing

- **Overview:**  
  Uses Google Gemini’s advanced language model to analyze the report contents and generate the final summarized insights.

- **Nodes Involved:**  
  - Google Gemini Chat Model

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type & Role:* LangChain Google Gemini LLM node, performs natural language understanding and text generation.  
    - *Configuration:*  
      - Uses model `models/gemini-2.5-pro`.  
      - Authenticated with Google PaLM API credentials.  
      - Receives instructions and report JSON from the agent node.  
    - *Key expressions:* None directly; linked as AI language model for the agent.  
    - *Connections:* Output feeds into the “H1 report summarizer” agent for final formatting.  
    - *Edge Cases:*  
      - API quota limits, authentication errors.  
      - Model versioning or updates might affect output consistency.  
    - *Sub-workflow:* None.

---

#### 2.5 Output Delivery

- **Overview:**  
  The summarized technical insights generated by the AI agent are returned to the chat interface for user consumption.

- **Nodes Involved:**  
  - H1 report summarizer (final output)

- **Node Details:**

  - Output is returned via the chat webhook automatically after processing by the agent.  
  - *Edge Cases:*  
    - If previous steps fail, output may be empty or error messages.  
  - No additional nodes handle output formatting or delivery.

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                      | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                                                |
|-------------------------|-----------------------------------|------------------------------------|----------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger            | Receives user chat input (H1 URLs) |                            | H1 report summarizer      | ### 📨 CHAT INTERFACE Setup Required: - Deploy webhook publicly - Send H1 URLs ending in .json - Custom CSS for pentester theme Format: https://hackerone.com/reports/ID |
| H1 report summarizer     | LangChain Agent                   | Orchestrates fetch & summarization | When chat message received | GET H1 report             | ### 🧠 Main Agent Requires: Google Gemini API key Purpose: Orchestrates analysis, calls HTTP tool automatically, formats output for hunters |
| GET H1 report            | HTTP Request Tool                 | Fetches HackerOne JSON report      | H1 report summarizer       | H1 report summarizer      | ### 📡 HTTP FETCHER Target: HackerOne JSON API Method: GET request Security: No hardcoded credentials Auto-called by AI agent with URL from chat |
| Google Gemini Chat Model | LangChain Google Gemini LLM       | Performs AI summarization           | H1 report summarizer       | H1 report summarizer      | ### 🔧 GEMINI LLM Config: Use gemini-2.5-pro Auth: Google PaLM API credentials Note: Can substitute with other models if needed             |
| Sticky Note              | Sticky Note                      | Documentation/Comments              |                            |                          | ## 🎯 WORKFLOW PURPOSE Converts HackerOne report URLs into actionable security insights for bug bounty hunters. INPUT: H1 report URL OUTPUT: Structured technical analysis with payloads & techniques |
| Sticky Note1             | Sticky Note                      | Documentation/Comments              |                            |                          | ### 📨 CHAT INTERFACE Setup Required: - Deploy webhook publicly - Send H1 URLs ending in .json - Custom CSS for pentester theme Format: https://hackerone.com/reports/ID |
| Sticky Note2             | Sticky Note                      | Documentation/Comments              |                            |                          | ### 🧠 Main Agent Requires: Google Gemini API key Purpose: Orchestrates analysis, calls HTTP tool automatically, formats output for hunters |
| Sticky Note3             | Sticky Note                      | Documentation/Comments              |                            |                          | ### 🔧 GEMINI LLM Config: Use gemini-2.5-pro Auth: Google PaLM API credentials Note: Can substitute with other models if needed             |
| Sticky Note4             | Sticky Note                      | Documentation/Comments              |                            |                          | ### 📡 HTTP FETCHER Target: HackerOne JSON API Method: GET request Security: No hardcoded credentials Auto-called by AI agent with URL from chat |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: LangChain Chat Trigger  
   - Enable public webhook (to receive external messages)  
   - Customize the chat UI with provided CSS for pentester theme (copy CSS from original node parameters)  
   - Set initial message: “Hey! Send your report’s link down below.”  
   - Output: main connection to “H1 report summarizer” node.

2. **Create "H1 report summarizer" node**  
   - Type: LangChain Agent  
   - Input text parameter: `={{ $json.chatInput }}` (takes chat input text)  
   - System prompt:  
     - Define role as expert AI assistant for bug bounty hunters.  
     - Must call tool “GET H1 report” exactly once with the full URL received.  
     - Focus on unique, high-impact technical insights (payloads, techniques, root cause, etc.)  
     - Output format: Summary, Techniques, Pro Tip (optional)  
   - Set "Tools" to include the HTTP fetch tool “GET H1 report” node.  
   - Connect main input from “When chat message received” node.  
   - Connect AI language model input from “Google Gemini Chat Model” node.

3. **Create "GET H1 report" node**  
   - Type: HTTP Request Tool  
   - Method: GET  
   - URL: dynamic, injected by AI agent tool call — use expression placeholder `={{ $input.all()[0].json.url }}` or as set by the agent tool call.  
   - No authentication needed.  
   - Connect output back to “H1 report summarizer” node as tool output.

4. **Create "Google Gemini Chat Model" node**  
   - Type: LangChain Google Gemini LLM  
   - Model name: `models/gemini-2.5-pro`  
   - Credentials: Link to Google PaLM API credentials (set up in n8n credentials)  
   - Connect output to “H1 report summarizer” as AI language model input.

5. **Configure Credentials**  
   - Set up Google PaLM API credentials with access to Gemini 2.5 Pro model.  
   - No credentials required for HTTP request node targeting public HackerOne API.

6. **Connect Workflow**  
   - "When chat message received" → "H1 report summarizer" (main)  
   - "H1 report summarizer" → "GET H1 report" (tool call)  
   - "GET H1 report" → "H1 report summarizer" (tool output)  
   - "Google Gemini Chat Model" → "H1 report summarizer" (AI language model input)  
   - Final output from “H1 report summarizer” returns to chat interface automatically.

7. **Test Workflow**  
   - Deploy webhook publicly.  
   - Send valid HackerOne report URL ending with `.json`, e.g., `https://hackerone.com/reports/312543.json`.  
   - Verify that the AI agent fetches the report, analyzes it, and returns a structured technical summary.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                      |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| Workflow purpose: Converts HackerOne report URLs into actionable security insights for bug hunters. | Sticky Note on workflow purpose node.                                              |
| Chat interface requires public webhook deployment and expects URLs ending with `.json`.             | Sticky Note1 detailing chat interface setup and URL format requirement.             |
| Main AI agent requires Google Gemini API key for orchestration and summarization.                   | Sticky Note2 describing main agent requirements and purpose.                        |
| Google Gemini model used: `models/gemini-2.5-pro` with Google PaLM API authentication.              | Sticky Note3 describing Gemini LLM configuration and auth.                          |
| HTTP fetcher targets HackerOne’s public JSON API with no credentials; called automatically by agent.| Sticky Note4 describing HTTP fetcher node usage and security notes.                 |
| Chat UI branding and styling credit: "Made with ❤️ by ethicxl".                                     | Embedded in chat CSS of the webhook node.                                          |

---

**Disclaimer:**  
The text provided is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---