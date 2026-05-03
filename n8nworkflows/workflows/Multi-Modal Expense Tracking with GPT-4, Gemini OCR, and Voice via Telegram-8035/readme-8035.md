Multi-Modal Expense Tracking with GPT-4, Gemini OCR, and Voice via Telegram

https://n8nworkflows.xyz/workflows/multi-modal-expense-tracking-with-gpt-4--gemini-ocr--and-voice-via-telegram-8035


# Multi-Modal Expense Tracking with GPT-4, Gemini OCR, and Voice via Telegram

### 1. Workflow Overview

This workflow is a sophisticated multi-modal personal finance assistant built on n8n, designed to track expenses and provide financial insights via Telegram. It supports multiple input formats—text messages, voice notes, and receipt images—and leverages AI models (GPT-4 and Google Gemini) alongside external tools and APIs for currency conversion, OCR, and transcription.

**Target Use Cases:**  
- Logging daily expenses through various input types  
- Automatically categorizing and converting expenses into USD  
- Providing spending summaries, balance inquiries, and detailed expense analyses  
- Handling multi-item expense messages by parsing and splitting them  
- Extracting expense data from receipt images and voice notes  
- Maintaining an up-to-date financial record in Google Sheets  

**Logical Blocks:**  
- **1.1 Input Reception and Routing:** Captures Telegram messages and routes based on content type (text, voice, image).  
- **1.2 Multi-Modal Data Extraction:** Converts voice to text and extracts receipt data from images using AI tools.  
- **1.3 Intent Classification:** Determines if the user's message is an expense entry or a query/other.  
- **1.4 Expense Parsing:** For expense entries, splits and formats multiple expenses into structured lines.  
- **1.5 Core AI Agent Processing:** The central GPT-4-powered assistant applies business logic, calls tools (Google Sheets, currency conversion, calculator), categorizes expenses, and constructs responses.  
- **1.6 Tool Invocation via MCP Pattern:** The Agent uses an MCP client-server pattern to asynchronously invoke various utility tools (currency exchange, calculator, Google Sheets tools).  
- **1.7 Final Response Delivery:** Sends formatted responses back to the user via Telegram.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Routing

- **Overview:** Detects incoming Telegram messages (text, voice, images) and routes them accordingly for processing.  
- **Nodes Involved:**  
  - Telegram Trigger  
  - Route Input by Message Type  

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Listens for new Telegram messages (text, voice, image) from the user.  
    - Configuration: Watches for "message" updates only.  
    - Input Connections: External (Telegram user messages).  
    - Output Connections: Main output connected to "Route Input by Message Type".  
    - Edge Cases: Telegram API rate limits, invalid webhook setup, unsupported message types.  

  - **Route Input by Message Type**  
    - Type: Switch node  
    - Role: Routes messages based on content type: text, audio (voice note), or image (receipt).  
    - Configuration: Checks message JSON for presence of "text", "audio", or "document.mime_type" containing "image".  
    - Inputs: Telegram Trigger  
    - Outputs:  
      - Text → Rename Fields  
      - Voice → Get Voice File from Telegram  
      - Image → Download Receipt Image  
    - Edge Cases: Messages without supported media types, malformed message JSON.  

#### 1.2 Multi-Modal Data Extraction

- **Overview:** Converts voice notes to text and extracts expense details from receipt images.  
- **Nodes Involved:**  
  - Get Voice File from Telegram  
  - Convert Voice to Text (ElevenLabs)  
  - Download Receipt Image  
  - Extract Receipt Data  
  - Rename Fields (to unify message format)  

- **Node Details:**

  - **Get Voice File from Telegram**  
    - Type: Telegram node  
    - Role: Downloads voice note file from Telegram servers for transcription.  
    - Configuration: Uses file ID from incoming voice message.  
    - Inputs: Route Input by Message Type (voice branch)  
    - Outputs: Convert Voice to Text (ElevenLabs)  
    - Edge Cases: File download failures, expired file IDs.  

  - **Convert Voice to Text (ElevenLabs)**  
    - Type: HTTP Request  
    - Role: Sends voice file to ElevenLabs API to transcribe speech to text.  
    - Configuration: POST multipart/form-data with "scribe_v1" model; uses API key via HTTP header authentication.  
    - Inputs: Get Voice File from Telegram  
    - Outputs: Rename Fields  
    - Edge Cases: API key invalid, transcription errors, network timeouts.  

  - **Download Receipt Image**  
    - Type: Telegram node  
    - Role: Downloads receipt image file from Telegram servers for OCR analysis.  
    - Configuration: Uses thumbnail file ID from document message.  
    - Inputs: Route Input by Message Type (image branch)  
    - Outputs: Extract Receipt Data  
    - Edge Cases: Missing thumbnails, file download issues.  

  - **Extract Receipt Data**  
    - Type: Google Gemini (AI OCR) node  
    - Role: Analyzes receipt image to extract total expense amount and description.  
    - Configuration: Uses Gemini-2.5-flash model; strict output format enforced for amount and description.  
    - Inputs: Download Receipt Image  
    - Outputs: Rename Fields  
    - Edge Cases: OCR inaccuracies, ambiguous receipts, model limit reach.  

  - **Rename Fields**  
    - Type: Set node  
    - Role: Normalizes and composes a unified text prompt field combining recognized text from all input types.  
    - Configuration: Assigns a "Prompt" field combining Telegram text, transcribed text, and OCR text parts.  
    - Inputs: Extract Receipt Data, Convert Voice to Text, Route Input by Message Type (text branch)  
    - Outputs: Classify Message Intent  
    - Edge Cases: Missing or empty input fields, expression evaluation errors.  

#### 1.3 Intent Classification

- **Overview:** Classifies user input as either an expense entry or a query/other type.  
- **Nodes Involved:**  
  - Classify Message Intent (Chain LLM)  
  - Intent Classification Model (GPT-4)  
  - Route to Expense or Query Processing (If node)  

- **Node Details:**

  - **Intent Classification Model**  
    - Type: GPT-4 (Azure OpenAI) LLM node  
    - Role: Processes unified prompt text to classify intent as "expenses" or "other".  
    - Configuration: Uses GPT-4.1 model with default options.  
    - Inputs: Rename Fields  
    - Outputs: Classify Message Intent  
    - Edge Cases: Model API errors, ambiguous messages, latency.  

  - **Classify Message Intent**  
    - Type: Chain LLM (Langchain)  
    - Role: Wraps the classification prompt logic; parses output from Intent Classification Model.  
    - Configuration: Custom prompt defining classification rules, expects single word output.  
    - Inputs: Intent Classification Model  
    - Outputs: Route to Expense or Query Processing  
    - Edge Cases: Parsing failures, unexpected output format.  

  - **Route to Expense or Query Processing**  
    - Type: If node  
    - Role: Routes workflow based on classification result ("other" → Main Financial Assistant; "expenses" → Expense Parsing).  
    - Inputs: Classify Message Intent  
    - Outputs:  
      - True ("other"): Main Financial Assistant  
      - False ("expenses"): Parse and Split Multiple Expenses  
    - Edge Cases: Unexpected classification values.  

#### 1.4 Expense Parsing

- **Overview:** Parses complex expense entries into individual, normalized expense lines for detailed processing.  
- **Nodes Involved:**  
  - Parse and Split Multiple Expenses (Chain LLM)  
  - Expense Parsing Model (GPT-4)  

- **Node Details:**

  - **Expense Parsing Model**  
    - Type: GPT-4 (Azure OpenAI) LLM node  
    - Role: Analyzes raw text prompt to split and format multiple expenses into individual lines.  
    - Configuration: Uses GPT-4.1 with a detailed prompt specifying formatting rules and normalization steps.  
    - Inputs: Rename Fields  
    - Outputs: Parse and Split Multiple Expenses  
    - Edge Cases: Parsing ambiguity, multi-language input, large input size limits.  

  - **Parse and Split Multiple Expenses**  
    - Type: Chain LLM (Langchain)  
    - Role: Processes output of Expense Parsing Model, ensuring format compliance and line-by-line expense entries.  
    - Configuration: Custom chain prompt enforcing strict output format for downstream processing.  
    - Inputs: Expense Parsing Model  
    - Outputs: Agent  
    - Edge Cases: Output validation failures, format deviations.  

#### 1.5 Core AI Agent Processing

- **Overview:** The central GPT-4-powered AI agent that processes expense or query requests, calls various tools, and composes user responses.  
- **Nodes Involved:**  
  - Agent (Langchain Agent)  
  - Main Financial Assistant (GPT-4)  
  - MCP Client  
  - Google Sheets Tools (Add_Expenses_Tool, Read_Rows_Tool, Total_Spent_Tool, Balance_Tool, Expenses_Tool)  
  - Exchange_Rate (HTTP Request)  
  - Calculator  

- **Node Details:**

  - **Main Financial Assistant**  
    - Type: GPT-4 (Azure OpenAI) LLM node  
    - Role: Serves as a wrapper to invoke the Agent with user messages or parsed expenses.  
    - Configuration: GPT-4.1 model; receives prompt from expense parsing or query path.  
    - Inputs: Route to Expense or Query Processing ("other" path) or Parse and Split Multiple Expenses ("expenses" path)  
    - Outputs: Agent  
    - Edge Cases: Model timeouts, incomplete inputs.  

  - **Agent**  
    - Type: Langchain Agent node  
    - Role: Executes the detailed system prompt logic for expense processing, categorization, financial reporting, currency conversion, and response generation.  
    - Configuration:  
      - System message includes detailed instructions, category references, response formats, currency handling, and error handling.  
      - Maximum 50 iterations per conversation to allow multi-step reasoning.  
      - Uses tools via MCP Client to interact with Google Sheets, exchange rate API, calculator, etc.  
    - Inputs: Main Financial Assistant, Parse and Split Multiple Expenses  
    - Outputs: Send Message  
    - Edge Cases: Tool invocation failures, miscategorization, API rate limits, currency conversion errors.  

  - **MCP Client**  
    - Type: MCP Client Tool  
    - Role: Sends AI tool requests (e.g., calculator, Google Sheets queries) to MCP Server Trigger node asynchronously.  
    - Configuration: Configured with SSE endpoint for communication.  
    - Inputs: Agent (ai_tool connection)  
    - Outputs: Agent (ai_tool response)  
    - Edge Cases: SSE connection drops, request timeouts.  

  - **MCP Server Trigger**  
    - Type: MCP Server Trigger  
    - Role: Central hub listening for tool requests from MCP Client, dispatches requests to appropriate tools, and returns results.  
    - Configuration: Webhook listening on custom path.  
    - Inputs: Calculator, Add_Expenses_Tool, Read_Rows_Tool, Total_Spent_Tool, Balance_Tool, Expenses_Tool, Exchange_Rate (all ai_tool connections)  
    - Outputs: MCP Client (ai_tool)  
    - Edge Cases: Tool failures, webhook errors.  

  - **Google Sheets Tools:**  
    - **Add_Expenses_Tool:** Appends new expense rows to "Expenses" sheet with date, category, description, amount.  
    - **Read_Rows_Tool:** Reads existing expense rows to find insertion points.  
    - **Total_Spent_Tool:** Reads pre-calculated daily/weekly/monthly totals from "Balance&Total_Spent" sheet.  
    - **Balance_Tool:** Reads current balance data.  
    - **Expenses_Tool:** Queries expense data filtered by category or date range.  
    - All configured with OAuth2 credentials for Google Sheets API.  
    - Edge Cases: Sheet access errors, data format inconsistencies, API quotas.  

  - **Exchange_Rate**  
    - Type: HTTP Request  
    - Role: Queries external ExchangeRate-API to get currency conversion rate to USD.  
    - Configuration: URL dynamically built with currency code from AI prompt; API key required.  
    - Edge Cases: API key invalid, rate limits, network errors.  

  - **Calculator**  
    - Type: Langchain Calculator Tool  
    - Role: Performs arithmetic calculations requested by the Agent, e.g., currency conversion.  
    - Edge Cases: Calculation errors, invalid expressions.  

#### 1.7 Final Response Delivery

- **Overview:** Sends the AI Agent's crafted response back to the Telegram user.  
- **Nodes Involved:**  
  - Send Message (Telegram)  

- **Node Details:**

  - **Send Message**  
    - Type: Telegram  
    - Role: Sends a text message response to the user's chat ID.  
    - Configuration: Takes output from Agent node; disables attribution text.  
    - Inputs: Agent  
    - Edge Cases: Telegram API failures, chat ID errors.  

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                              | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                   |
|-----------------------------|---------------------------------|----------------------------------------------|-------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger                | Entry point: listens for Telegram messages   | External                      | Route Input by Message Type  |                                                                                               |
| Route Input by Message Type  | Switch                         | Routes by message type: text, voice, image   | Telegram Trigger              | Rename Fields, Get Voice File from Telegram, Download Receipt Image |                                                                                               |
| Get Voice File from Telegram | Telegram                       | Downloads voice note file                      | Route Input by Message Type   | Convert Voice to Text (ElevenLabs) |                                                                                               |
| Convert Voice to Text (ElevenLabs) | HTTP Request               | Transcribes voice note to text                 | Get Voice File from Telegram  | Rename Fields                |                                                                                               |
| Download Receipt Image       | Telegram                       | Downloads receipt image                        | Route Input by Message Type   | Extract Receipt Data         |                                                                                               |
| Extract Receipt Data         | Google Gemini (AI OCR)         | Extracts amount and description from receipt | Download Receipt Image        | Rename Fields                |                                                                                               |
| Rename Fields               | Set                            | Normalizes and merges text inputs             | Route Input by Message Type (text), Convert Voice to Text, Extract Receipt Data | Classify Message Intent      |                                                                                               |
| Intent Classification Model  | GPT-4 (Azure OpenAI)           | Classifies message intent                      | Rename Fields                 | Classify Message Intent      |                                                                                               |
| Classify Message Intent      | Chain LLM (Langchain)          | Parses classification result                   | Intent Classification Model   | Route to Expense or Query Processing | Sticky Note5: Intent Classification explanation                                               |
| Route to Expense or Query Processing | If                       | Routes workflow to expense parsing or query handling | Classify Message Intent       | Main Financial Assistant, Parse and Split Multiple Expenses |                                                                                               |
| Parse and Split Multiple Expenses | Chain LLM (Langchain)      | Splits multiple expenses into lines            | Expense Parsing Model         | Agent                       | Sticky Note6: Expense Parsing explanation                                                    |
| Expense Parsing Model        | GPT-4 (Azure OpenAI)           | Parses raw expense text                         | Rename Fields                 | Parse and Split Multiple Expenses |                                                                                               |
| Main Financial Assistant     | GPT-4 (Azure OpenAI)           | Feeds final formatted request to Agent         | Route to Expense or Query Processing ("other"), Parse and Split Multiple Expenses ("expenses") | Agent                       | Sticky Note7: Main Financial Assistant explanation                                           |
| Agent                       | Langchain Agent                | Core AI agent processing with tools            | Main Financial Assistant, Parse and Split Multiple Expenses | Send Message                | Sticky Note: Detailed system prompt explaining agent logic and multi-tool usage              |
| MCP Client                  | MCP Client Tool                | Sends tool requests to MCP Server               | Agent (ai_tool)              | Agent (ai_tool)              | Sticky Note3: MCP Server & Client explanation                                               |
| MCP Server Trigger          | MCP Server Trigger             | Dispatches requests to tools and returns results | Calculator, Add_Expenses_Tool, Read_Rows_Tool, Total_Spent_Tool, Balance_Tool, Expenses_Tool, Exchange_Rate | MCP Client (ai_tool)         | Sticky Note3: MCP Server & Client explanation                                               |
| Calculator                 | Langchain Calculator Tool       | Performs arithmetic calculations                 | MCP Server Trigger (ai_tool) | MCP Server Trigger (ai_tool) | Sticky Note8: Google Sheets and Utility Tools explanation                                  |
| Add_Expenses_Tool           | Google Sheets Tool             | Appends new expense rows                         | MCP Server Trigger (ai_tool) | MCP Server Trigger (ai_tool) | Sticky Note8: Google Sheets and Utility Tools explanation                                  |
| Read_Rows_Tool             | Google Sheets Tool             | Reads existing expense rows                      | MCP Server Trigger (ai_tool) | MCP Server Trigger (ai_tool) | Sticky Note8: Google Sheets and Utility Tools explanation                                  |
| Total_Spent_Tool            | Google Sheets Tool             | Fetches total spending summaries                 | MCP Server Trigger (ai_tool) | MCP Server Trigger (ai_tool) | Sticky Note8: Google Sheets and Utility Tools explanation                                  |
| Balance_Tool                | Google Sheets Tool             | Reads current balance                            | MCP Server Trigger (ai_tool) | MCP Server Trigger (ai_tool) | Sticky Note8: Google Sheets and Utility Tools explanation                                  |
| Expenses_Tool               | Google Sheets Tool             | Queries and filters expense data                  | MCP Server Trigger (ai_tool) | MCP Server Trigger (ai_tool) | Sticky Note8: Google Sheets and Utility Tools explanation                                  |
| Exchange_Rate               | HTTP Request                  | Retrieves currency conversion rate to USD       | MCP Server Trigger (ai_tool) | MCP Server Trigger (ai_tool) | Sticky Note8: Google Sheets and Utility Tools explanation                                  |
| Send Message               | Telegram                       | Sends response back to Telegram user            | Agent                        | External                    |                                                                                               |
| Sticky Note                 | Sticky Note                   | Various notes providing documentation and context | -                            | -                           | Sticky notes contain detailed workflow explanations, usage instructions, and requirements. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot Token.  
   - Set to listen for "message" updates only.

2. **Add Switch Node ("Route Input by Message Type"):**  
   - Add conditions to detect:  
     - Text messages (JSON contains "text")  
     - Voice messages (JSON contains "audio")  
     - Receipt images (document.mime_type contains "image")  
   - Connect Telegram Trigger output to this node.

3. **Voice Branch:**  
   - Add "Get Voice File from Telegram" node.  
     - Use expression to get voice file ID from incoming message.  
   - Add "Convert Voice to Text (ElevenLabs)" HTTP Request node:  
     - POST to ElevenLabs speech-to-text API with multipart form data.  
     - Use your ElevenLabs API key in HTTP header authentication.  
   - Connect Get Voice File → Convert Voice to Text.

4. **Receipt Image Branch:**  
   - Add "Download Receipt Image" Telegram node.  
     - Use the thumbnail file ID from the document message.  
   - Add "Extract Receipt Data" Google Gemini node:  
     - Use Gemini-2.5-flash model.  
     - Configure prompt to extract total amount and description, with strict output format.  
   - Connect Download Receipt Image → Extract Receipt Data.

5. **Text Branch:**  
   - Connect text output from switch directly.

6. **Normalize Input Text:**  
   - Add "Rename Fields" Set node.  
   - Compose a unified "Prompt" field combining:  
     - Telegram text message text  
     - Transcribed voice text  
     - OCR extracted receipt text  
   - Connect all three branches (text, voice transcription, receipt extraction) to Rename Fields.

7. **Intent Classification:**  
   - Add GPT-4 node ("Intent Classification Model") with a prompt to classify input as "expenses" or "other".  
   - Add Chain LLM node ("Classify Message Intent") to parse model output.  
   - Connect Rename Fields → Intent Classification Model → Classify Message Intent.

8. **Routing by Intent:**  
   - Add If node ("Route to Expense or Query Processing"):  
     - Condition: If classification result = "other" → True branch; else False branch.  
   - True branch connects to "Main Financial Assistant" (query handling).  
   - False branch connects to "Parse and Split Multiple Expenses" (expense parsing).

9. **Expense Parsing:**  
   - Add GPT-4 node ("Expense Parsing Model") with a prompt to split and format multiple expenses.  
   - Add Chain LLM node ("Parse and Split Multiple Expenses") to parse output in strict line format.  
   - Connect Rename Fields → Expense Parsing Model → Parse and Split Multiple Expenses.

10. **Core AI Agent:**  
    - Add GPT-4 Chain LLM ("Main Financial Assistant") node to feed formatted requests to Agent.  
    - Add Langchain Agent node ("Agent") with detailed system prompt including:  
      - Expense processing logic, categorization, currency conversion  
      - Financial reporting and alerting  
      - Access to tools for Google Sheets, calculator, exchange rates  
      - Max iterations ~50 for reasoning  
    - Connect:  
      - Main Financial Assistant → Agent  
      - Parse and Split Multiple Expenses → Agent

11. **MCP Pattern Setup:**  
    - Add MCP Client node connected to Agent (ai_tool).  
    - Add MCP Server Trigger node listening on a webhook path.  
    - Connect all tool nodes (Calculator, Google Sheets tools, Exchange_Rate) to MCP Server Trigger (ai_tool).  
    - Connect MCP Server Trigger output back to MCP Client input.

12. **Tools Configuration:**  
    - Add Google Sheets nodes:  
      - Add_Expenses_Tool (append mode to "Expenses" sheet)  
      - Read_Rows_Tool (read mode)  
      - Total_Spent_Tool (read specific ranges in "Balance&Total_Spent")  
      - Balance_Tool (read balance cell)  
      - Expenses_Tool (query/filter expenses)  
    - Add HTTP Request node for Exchange_Rate API with dynamic URL including currency code.  
    - Add Langchain Calculator tool node.

13. **Final Output:**  
    - Add Telegram node ("Send Message") to send Agent's output text back to the user.  
    - Connect Agent → Send Message.

14. **Credential Setup:**  
    - Configure:  
      - Telegram Bot credentials in Telegram nodes.  
      - Google Sheets OAuth2 credentials for all Google Sheets nodes.  
      - Azure OpenAI credentials for GPT-4 nodes.  
      - ElevenLabs API key in HTTP Request node.  
      - ExchangeRate-API key in HTTP Request node for exchange rates.

15. **Google Sheets Preparation:**  
    - Create a Google Sheet with two tabs:  
      - **Expenses:** Columns: Expense Date, Expense Description, Expense Amount (USD), Expense Category (exact names).  
      - **Balance&Total_Spent:** Contains starting balance in a fixed cell (e.g., B3) and formulas for current balance, daily/weekly/monthly totals.

16. **Testing & Validation:**  
    - Test text expense messages, voice notes, and receipt images.  
    - Verify correct parsing, currency conversion, category assignment, and data appending.  
    - Confirm summary queries return accurate reports and alerts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| ## 🤖 Your AI Financial Assistant is Here! This n8n template transforms a Telegram bot into a powerful, multi-modal personal finance assistant. It can track your expenses from text, voice notes, and even receipt images, providing intelligent summaries and alerts to help you stay on top of your finances. Use it to effortlessly log daily purchases, get quick spending summaries, check your budget, and analyze your financial habits, all from the convenience of a Telegram chat. | Workflow purpose and user instructions as detailed in Sticky Note2                                               |
| ### The MCP Server & Client: How the AI Uses Tools This workflow uses a powerful MCP pattern to allow the main AI Agent to use tools that aren't directly connected to it. The MCP Server Trigger acts as a central listening hub, dispatching tool requests, and the MCP Client acts as the messenger for the Agent. This design keeps the workflow modular and clean.                                                                                                                                                                                                                                      | Sticky Note3 explaining MCP architecture                                                                           |
| #### Google Sheets & Utility Tools The workflow uses several Google Sheets tools to log, read, and analyze expenses, plus utility tools like the Calculator and Exchange Rate HTTP request. Each tool adds a specific skill to the AI Agent's capabilities, enabling complex financial interactions.                                                                                                                                                                                                                                                                                                                           | Sticky Note8 describing tools                                                                                      |
| #### Intent Classification Purpose: To quickly figure out what the user wants. Task: Classify message as either "expenses" or "other". This determines the workflow path.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note5                                                                                                       |
| #### Expense Parsing Purpose: To clean up and structure complex expense entries. Task: Break down multi-expense messages into clean, separate lines for accurate processing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note6                                                                                                       |
| #### Main Financial Assistant Purpose: The core decision-making brain. Task: Uses reasoning to decide tool usage and crafts detailed responses for users.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note7                                                                                                       |
| Requirements: Telegram Bot Token, Google Sheets API credentials, Azure OpenAI credentials (GPT-4), ElevenLabs API key, ExchangeRate-API key. Google Sheets must have precisely named tabs and columns as specified for smooth operation.                                                                                                                                                                                                                                                                                                                                                                                        | Included in Sticky Note2                                                                                            |

---

**Disclaimer:** The provided workflow is an automated n8n integration respecting all content policies. It handles only legal and public data, ensuring no illegal or offensive content is processed.