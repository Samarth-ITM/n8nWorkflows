Smart Lead Outreach with Explorium MCP Using Google Sheets and Telegram

https://n8nworkflows.xyz/workflows/smart-lead-outreach-with-explorium-mcp-using-google-sheets-and-telegram-5133


# Smart Lead Outreach with Explorium MCP Using Google Sheets and Telegram

### 1. Workflow Overview

This workflow, titled **"Smart Lead Outreach with Explorium MCP Using Google Sheets and Telegram"**, enables automated lead engagement by combining AI-driven data enrichment, intelligent processing, and communication via Telegram. It leverages Explorium MCP for data enrichment, LangChain AI agents with OpenAI GPT-4 and Google Gemini models for conversational intelligence, and Google Sheets for data storage. The workflow is designed for sales or marketing teams aiming to automate personalized outreach and lead qualification.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Receiving incoming Telegram messages that trigger the workflow.
- **1.2 AI Processing and Enrichment:** Using LangChain AI agents with multiple tools and language models to analyze and enrich lead data, including Explorium MCP and Tavily Search HTTP request.
- **1.3 Data Processing and Routing:** Post-AI processing for data filtering, lead type classification (person or company), and conditional routing.
- **1.4 Lead Data Storage:** Saving processed person-type lead data to Google Sheets.
- **1.5 Response Handling:** Sending Telegram responses back to the user.
- **1.6 Auxiliary Logic:** Includes memory storage for context, intermediate code processing, and handling different AI models (OpenAI GPT-4 and Google Gemini).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block listens for incoming Telegram messages that initiate the workflow.
- **Nodes Involved:** 
  - Telegram Trigger
- **Node Details:**
  - **Telegram Trigger**
    - Type: Telegram Trigger node (Webhook listener)
    - Configuration: Default Telegram webhook with no additional parameters specified.
    - Input: External Telegram message events.
    - Output: Triggers the workflow with the message data payload.
    - Edge Cases: Telegram connectivity issues, webhook misconfiguration, invalid message formats.
    - Version: 1.2

#### 1.2 AI Processing and Enrichment

- **Overview:** Processes the Telegram message through an enhanced AI agent that integrates multiple AI tools and language models to enrich and analyze lead data.
- **Nodes Involved:** 
  - Enhanced AI Agent
  - OpenAI GPT-4
  - Postgres Memory
  - Explorium MCP Client
  - Enhanced Tavily Search
  - Smart Data Processor (Code node)
- **Node Details:**
  - **Enhanced AI Agent**
    - Type: LangChain Agent node
    - Role: Central AI orchestrator combining language models, memory, and AI tools.
    - Configuration: No explicit parameters, but linked to OpenAI GPT-4 (languageModel), Postgres Memory (memory), Explorium MCP, and Tavily Search (ai_tools).
    - Inputs: Telegram Trigger node output.
    - Outputs: Feeds processed data to Smart Data Processor.
    - Edge Cases: AI API quota limits, tool unavailability, input data issues.
    - Version: 2
  - **OpenAI GPT-4**
    - Type: LangChain OpenAI GPT-4 chat model node
    - Role: Provides advanced language understanding and generation capabilities.
    - Configuration: Default GPT-4 chat parameters.
    - Connected as language model to Enhanced AI Agent.
    - Edge Cases: OpenAI API key/auth failure, rate limits.
    - Version: 1.2
  - **Postgres Memory**
    - Type: LangChain Postgres memory node
    - Role: Maintains conversational context/history.
    - Configuration: Connected as memory to Enhanced AI Agent.
    - Edge Cases: Database connectivity issues.
    - Version: 1.3
  - **Explorium MCP Client**
    - Type: LangChain Explorium MCP tool node
    - Role: Provides data enrichment via Explorium MCP API.
    - Configuration: Connected as AI tool to Enhanced AI Agent.
    - Edge Cases: API authentication, rate limits, query failures.
    - Version: 1
  - **Enhanced Tavily Search**
    - Type: HTTP Request Tool node
    - Role: Additional data enrichment or search via Tavily API.
    - Configuration: HTTP request setup (details unspecified).
    - Connected as AI tool to Enhanced AI Agent.
    - Edge Cases: HTTP errors, timeouts.
    - Version: 4.2
  - **Smart Data Processor**
    - Type: Code node (JavaScript)
    - Role: Processes AI agent output, formats or filters data.
    - Configuration: Custom code logic (not detailed in JSON).
    - Inputs: Enhanced AI Agent output.
    - Outputs: Leads to Telegram Response and Lead Data Filter nodes.
    - Edge Cases: Code exceptions, data format mismatches.
    - Version: 2

#### 1.3 Data Processing and Routing

- **Overview:** Filters the processed lead data to classify leads and route accordingly.
- **Nodes Involved:** 
  - Lead Data Filter (If node)
  - Person/Company Split (If node)
  - AI Agent
  - Code (processing)
- **Node Details:**
  - **Lead Data Filter**
    - Type: If node
    - Role: Filters leads based on criteria (e.g., presence of data or lead qualification).
    - Configuration: Conditional logic (details unspecified).
    - Input: Smart Data Processor output.
    - Output: Routes to Person/Company Split node.
    - Edge Cases: Incorrect conditional logic, missing data.
    - Version: 2
  - **Person/Company Split**
    - Type: If node
    - Role: Determines if lead is a person or company.
    - Configuration: Branches into two outputs both leading to AI Agent.
    - Input: Lead Data Filter output.
    - Output: Two branches connected to AI Agent.
    - Edge Cases: Ambiguous data causing incorrect classification.
    - Version: 2
  - **AI Agent**
    - Type: LangChain agent node
    - Role: Further AI processing based on lead type.
    - Configuration: Uses Google Gemini Chat Model as language model.
    - Input: Person/Company Split outputs.
    - Output: Feeds into Code node.
    - Edge Cases: AI API errors or misclassification.
    - Version: 2
  - **Code**
    - Type: Code node
    - Role: Final processing before data storage.
    - Configuration: Custom JavaScript code.
    - Input: AI Agent output.
    - Output: Leads to Save Person Data node.
    - Edge Cases: Code execution errors, data format issues.
    - Version: 2

#### 1.4 Lead Data Storage

- **Overview:** Saves the processed person lead data into Google Sheets for record-keeping and further use.
- **Nodes Involved:** 
  - Save Person Data (Google Sheets node)
- **Node Details:**
  - **Save Person Data**
    - Type: Google Sheets node
    - Role: Appends or updates lead data in a designated Google Sheet.
    - Configuration: Sheet ID, range, and authentication credentials (not detailed).
    - Input: Code node output.
    - Output: None (end node).
    - Edge Cases: Google API auth failure, sheet access errors, quota limits.
    - Version: 4.6

#### 1.5 Response Handling

- **Overview:** Sends a response back to the Telegram user based on processed data.
- **Nodes Involved:** 
  - Telegram Response
- **Node Details:**
  - **Telegram Response**
    - Type: Telegram node
    - Role: Sends text or media messages back to the user.
    - Configuration: Uses webhook ID and message details from Smart Data Processor.
    - Input: Smart Data Processor output.
    - Output: Workflow ends after sending response.
    - Edge Cases: Telegram API errors, message formatting issues.
    - Version: 1.2

#### 1.6 Auxiliary Logic and Notes

- **Overview:** Miscellaneous nodes for workflow annotations and additional AI processing.
- **Nodes Involved:**
  - Google Gemini Chat Model
  - Multiple Sticky Note nodes
- **Node Details:**
  - **Google Gemini Chat Model**
    - Type: LangChain Google Gemini chat model node
    - Role: Provides an alternative AI language model for the AI Agent node.
    - Configuration: Default settings.
    - Connected as languageModel to AI Agent.
    - Edge Cases: API availability and authentication.
    - Version: 1
  - **Sticky Notes**
    - Type: Sticky Note nodes
    - Role: Workflow annotations with no effect on execution.
    - Content: Empty in JSON provided.
    - Version: 1

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                         | Input Node(s)              | Output Node(s)                | Sticky Note |
|---------------------|-----------------------------------|---------------------------------------|----------------------------|------------------------------|-------------|
| Telegram Trigger     | Telegram Trigger                  | Input reception from Telegram messages| -                          | Enhanced AI Agent             |             |
| Enhanced AI Agent    | LangChain Agent                  | AI processing and enrichment          | Telegram Trigger, OpenAI GPT-4, Postgres Memory, Explorium MCP Client, Enhanced Tavily Search | Smart Data Processor         |             |
| OpenAI GPT-4         | LangChain LM Chat OpenAI          | Language model for AI agent            | -                          | Enhanced AI Agent (ai_languageModel) |             |
| Postgres Memory      | LangChain Memory Postgres Chat    | Conversational memory for AI agent    | -                          | Enhanced AI Agent (ai_memory)  |             |
| Explorium MCP Client | LangChain MCP Client Tool          | Data enrichment via Explorium MCP API | -                          | Enhanced AI Agent (ai_tool)    |             |
| Enhanced Tavily Search| HTTP Request Tool                 | Data enrichment via HTTP API           | -                          | Enhanced AI Agent (ai_tool)    |             |
| Smart Data Processor | Code Node                        | Processes AI output for routing        | Enhanced AI Agent           | Telegram Response, Lead Data Filter |             |
| Telegram Response    | Telegram Node                   | Sends response back to Telegram user   | Smart Data Processor        | -                            |             |
| Lead Data Filter     | If Node                         | Filters lead data                      | Smart Data Processor        | Person/Company Split          |             |
| Person/Company Split | If Node                         | Classifies lead as person or company  | Lead Data Filter            | AI Agent                     |             |
| AI Agent             | LangChain Agent                 | Further AI processing                   | Person/Company Split        | Code                         |             |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini | Language model for AI Agent             | -                          | AI Agent (ai_languageModel)   |             |
| Code                 | Code Node                      | Final data processing before storage   | AI Agent                   | Save Person Data             |             |
| Save Person Data      | Google Sheets Node             | Saves person lead data                  | Code                       | -                            |             |
| Sticky Note          | Sticky Note Node               | Workflow annotation                     | -                          | -                            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Configure webhook for Telegram bot with appropriate webhook URL.
   - No additional parameters required.
   - Position: Starting node.

2. **Create OpenAI GPT-4 Node**
   - Type: LangChain LM Chat OpenAI
   - Configure with OpenAI API credentials.
   - Use default GPT-4 chat parameters.

3. **Create Postgres Memory Node**
   - Type: LangChain Memory Postgres Chat
   - Configure with Postgres database credentials for conversational memory.

4. **Create Explorium MCP Client Node**
   - Type: LangChain MCP Client Tool
   - Configure with Explorium MCP API credentials.

5. **Create Enhanced Tavily Search Node**
   - Type: HTTP Request Tool
   - Configure HTTP request details for Tavily Search API (URL, auth).
   - Use GET or POST as required.

6. **Create Enhanced AI Agent Node**
   - Type: LangChain Agent
   - Connect input from Telegram Trigger.
   - Assign languageModel input to OpenAI GPT-4 node.
   - Assign ai_memory input to Postgres Memory node.
   - Assign ai_tool inputs to Explorium MCP Client and Enhanced Tavily Search nodes.
   - No additional parameters needed.

7. **Create Smart Data Processor Node**
   - Type: Code node
   - Connect input from Enhanced AI Agent.
   - Implement custom JavaScript to parse and prepare data for further processing and response.
   - Output two branches: one to Telegram Response, another to Lead Data Filter.

8. **Create Telegram Response Node**
   - Type: Telegram node
   - Connect input from Smart Data Processor.
   - Configure with Telegram bot credentials.
   - Set message content based on processed output.

9. **Create Lead Data Filter Node (If Node)**
   - Connect input from Smart Data Processor.
   - Configure conditional logic to filter valid leads or based on other criteria.

10. **Create Person/Company Split Node (If Node)**
    - Connect input from Lead Data Filter.
    - Configure logic to branch based on lead type (person or company).

11. **Create Google Gemini Chat Model Node**
    - Type: LangChain LM Chat Google Gemini
    - Configure with Google Gemini API credentials.

12. **Create AI Agent Node**
    - Type: LangChain Agent
    - Connect inputs from both outputs of Person/Company Split.
    - Assign languageModel input to Google Gemini Chat Model node.

13. **Create Code Node**
    - Connect input from AI Agent.
    - Implement JavaScript code for final data formatting/preparation.

14. **Create Save Person Data Node**
    - Type: Google Sheets node
    - Connect input from Code node.
    - Configure Google Sheets credentials and specify target spreadsheet and sheet.
    - Set action to append or update rows with person lead data.

15. **Optional: Add Sticky Note nodes for annotations**

16. **Connect nodes as per the described logic flow**

17. **Test the workflow end-to-end with Telegram messages**

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                            |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------|
| This workflow integrates Explorium MCP for predictive data enrichment in lead outreach.        | Explorium MCP documentation: https://explorium.ai/docs/    |
| Uses LangChain agents to combine multiple AI tools and memory for enhanced conversational AI. | LangChain docs: https://python.langchain.com/en/latest/    |
| Telegram integration requires bot setup with webhook configuration.                            | Telegram Bot API: https://core.telegram.org/bots/api       |
| Google Sheets node requires OAuth2 credentials for access and update of sheets.                | Google Sheets API docs: https://developers.google.com/sheets/api |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. It strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.