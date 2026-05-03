Build Your First AI Data Analyst Chatbot

https://n8nworkflows.xyz/workflows/build-your-first-ai-data-analyst-chatbot-3050


# Build Your First AI Data Analyst Chatbot

### 1. Workflow Overview

This workflow implements an AI-powered Data Analyst Chatbot designed to interactively analyze transactional data stored in Google Sheets. It enables users to query data, filter by various criteria (date ranges, product names, transaction status), and perform calculations on the retrieved data via an AI agent interface.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures chat messages from users to trigger the AI processing.
- **1.2 AI Processing and Memory:** Uses an AI Agent with a chat model and short-term memory to interpret user queries and decide which data retrieval or calculation tools to invoke.
- **1.3 Data Retrieval Tools:** Multiple tools and sub-workflows that fetch data from Google Sheets based on different filters (by date, product name, status, or all transactions).
- **1.4 Data Transformation:** Processes raw Google Sheets API responses into structured JSON objects for AI consumption.
- **1.5 Data Aggregation and Filtering:** Aggregates multiple data items into a single item and filters data by status when needed.
- **1.6 Calculation Tool:** Performs mathematical calculations on the data as requested by the AI.
- **1.7 Sub-Workflow for Date-Filtered Records:** A callable sub-workflow that retrieves records filtered by date and optionally by status, returning aggregated results to the AI.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming chat messages from users to initiate the AI data analysis process.
- **Nodes Involved:**  
  - When chat message received
- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point for user chat messages, triggering the workflow.  
    - Configuration: Default options, webhook ID set for receiving chat messages.  
    - Inputs: External chat message webhook.  
    - Outputs: Passes message data to AI Agent node.  
    - Edge Cases: Webhook connectivity issues, malformed chat messages.

#### 2.2 AI Processing and Memory

- **Overview:** Processes user input using an AI Agent with a chat model and short-term memory to maintain conversational context and decide which tools to invoke.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Buffer Memory  
- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Core AI logic interpreting user queries and orchestrating tool usage.  
    - Configuration:  
      - Max iterations: 15 (limits AI reasoning steps)  
      - System message includes current timestamp for context.  
    - Inputs: Chat messages from trigger, AI language model, memory, and tools.  
    - Outputs: AI responses or tool invocations.  
    - Edge Cases: AI model timeouts, exceeding max iterations, incorrect tool usage.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides language understanding and generation capabilities to the AI Agent.  
    - Configuration:  
      - Model: GPT-4o (a GPT-4 variant)  
      - Temperature: 0.2 (low randomness for consistent responses)  
      - Credentials: OpenAI API key configured.  
    - Inputs: Prompts from AI Agent.  
    - Outputs: Generated text responses.  
    - Edge Cases: API rate limits, authentication failures, model unavailability.

  - **Buffer Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains short-term memory of last 5 interactions to provide conversational context.  
    - Configuration: Default (window size 5).  
    - Inputs: Conversation history.  
    - Outputs: Contextual memory to AI Agent.  
    - Edge Cases: Memory overflow, loss of context on restart.

#### 2.3 Data Retrieval Tools

- **Overview:** Provides multiple tools for the AI Agent to fetch data from Google Sheets filtered by different criteria.
- **Nodes Involved:**  
  - Get all transactions  
  - Get transactions by product name  
  - Get transactions by status  
  - Records by date (sub-workflow tool)  
- **Node Details:**

  - **Get all transactions**  
    - Type: Google Sheets Tool  
    - Role: Retrieves all transaction records from the specified Google Sheet.  
    - Configuration:  
      - Document URL and sheet GID set to the example spreadsheet.  
      - No filters applied (pulls entire dataset).  
      - Credentials: Google Sheets OAuth2.  
    - Inputs: AI Agent tool invocation.  
    - Outputs: Full transaction dataset.  
    - Edge Cases: Large data volume causing timeouts or API limits.

  - **Get transactions by product name**  
    - Type: Google Sheets Tool  
    - Role: Retrieves transactions filtered by product name.  
    - Configuration:  
      - Filter on "Product" column using AI-provided product_name variable.  
      - Document and sheet set as above.  
      - Credentials: Google Sheets OAuth2.  
    - Inputs: AI Agent tool invocation with product_name parameter.  
    - Outputs: Filtered transactions by product.  
    - Edge Cases: Missing or misspelled product names, empty results.

  - **Get transactions by status**  
    - Type: Google Sheets Tool  
    - Role: Retrieves transactions filtered by status (e.g., Refund, Completed, Error).  
    - Configuration:  
      - Filter on "Status" column using AI-provided transaction_status variable.  
      - Document and sheet as above.  
      - Credentials: Google Sheets OAuth2.  
    - Inputs: AI Agent tool invocation with transaction_status parameter.  
    - Outputs: Filtered transactions by status.  
    - Edge Cases: Invalid status values, empty results.

  - **Records by date**  
    - Type: LangChain Tool Workflow (Sub-workflow)  
    - Role: Retrieves records filtered by start_date, end_date, and optionally status by invoking a sub-workflow.  
    - Configuration:  
      - Points to sub-workflow ID "a2BIIjr2gLBay06M" (contained in same workflow).  
      - Inputs mapped from AI variables: start_date, end_date, status.  
    - Inputs: AI Agent tool invocation with date and status parameters.  
    - Outputs: Filtered and aggregated records.  
    - Edge Cases: Invalid date formats, missing parameters, sub-workflow failures.

#### 2.4 Data Transformation

- **Overview:** Converts the raw Google Sheets API JSONP response into structured JSON objects with proper date formatting for AI consumption.
- **Nodes Involved:**  
  - Google Sheets request  
  - Code  
- **Node Details:**

  - **Google Sheets request**  
    - Type: HTTP Request  
    - Role: Sends a custom HTTP request to Google Sheets API to query data with date filtering.  
    - Configuration:  
      - URL points to Google Sheets visualization API endpoint.  
      - Query parameters include sheet name and a SQL-like query filtering rows by date range.  
      - Authentication: Google Sheets OAuth2.  
    - Inputs: Sub-workflow trigger with start_date and end_date.  
    - Outputs: Raw JSONP response string.  
    - Edge Cases: API authentication errors, malformed queries, rate limits.

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Parses the JSONP response, extracts JSON, validates structure, converts date fields to ISO format, and outputs clean JSON objects.  
    - Configuration:  
      - Uses regex to extract JSON from JSONP.  
      - Parses JSON, checks for expected table structure.  
      - Converts Google Visualization date strings to "YYYY-MM-DD".  
      - Maps rows to objects keyed by column labels.  
    - Inputs: Raw JSONP data from HTTP Request.  
    - Outputs: Array of JSON objects representing rows.  
    - Edge Cases: Parsing failures, unexpected data format, missing fields.

#### 2.5 Data Aggregation and Filtering

- **Overview:** Filters records by status and aggregates multiple items into a single item for AI processing.
- **Nodes Involved:**  
  - Filter by status  
  - Aggregate  
- **Node Details:**

  - **Filter by status**  
    - Type: Filter  
    - Role: Filters incoming items to only those matching the status parameter from the sub-workflow input.  
    - Configuration:  
      - Condition: Checks if item.Status contains the status string from input.  
      - Case sensitive and strict validation enabled.  
    - Inputs: JSON objects from Code node.  
    - Outputs: Filtered items to Aggregate node.  
    - Edge Cases: No matching items, case mismatches.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all filtered items into a single aggregated item to send back to AI.  
    - Configuration: Default aggregation of all item data.  
    - Inputs: Filtered items.  
    - Outputs: Single aggregated item.  
    - Edge Cases: Empty input array, aggregation errors.

#### 2.6 Calculation Tool

- **Overview:** Allows the AI Agent to perform mathematical calculations on data.
- **Nodes Involved:**  
  - Calculator  
- **Node Details:**

  - **Calculator**  
    - Type: LangChain Calculator Tool  
    - Role: Executes mathematical operations as requested by the AI Agent.  
    - Configuration: Default, no parameters.  
    - Inputs: AI Agent tool invocation.  
    - Outputs: Calculation results to AI Agent.  
    - Edge Cases: Invalid mathematical expressions, division by zero.

#### 2.7 Sub-Workflow for Date-Filtered Records

- **Overview:** A sub-workflow triggered by the "Records by date" tool to fetch and process records filtered by date and optionally status.
- **Nodes Involved:**  
  - When Executed by Another Workflow (sub-workflow trigger)  
  - Google Sheets request  
  - Code  
  - Filter by status  
  - Aggregate  
- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for sub-workflow execution when called by main workflow tool.  
    - Configuration: Accepts inputs: start_date, end_date, status.  
    - Inputs: Called by "Records by date" tool.  
    - Outputs: Passes inputs to Google Sheets request.  
    - Edge Cases: Missing inputs, trigger failures.

  - Other nodes as described in 2.4 and 2.5 process the data and return aggregated results.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                                  | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                          |
|----------------------------|--------------------------------------|-------------------------------------------------|-------------------------------|------------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger                | Entry point for chat messages                    | -                             | AI Agent                     |                                                                                                    |
| AI Agent                   | LangChain Agent                      | Core AI processing and tool orchestration       | When chat message received, OpenAI Chat Model, Buffer Memory, Calculator, Data Tools | -                            | The **AI Tools Agent** has access to all tools and uses $fromAI function to decide tool usage. See [docs](https://docs.n8n.io/advanced-ai/examples/using-the-fromai-function/) |
| OpenAI Chat Model          | LangChain OpenAI Chat Model          | Provides language understanding                  | AI Agent                      | AI Agent                     | Use many models here, including free Google Gemini options. Test thoroughly for data analysis.     |
| Buffer Memory              | LangChain Memory Buffer Window       | Maintains short-term conversation memory        | -                             | AI Agent                     | Short term memory remembers last 5 interactions during chat.                                       |
| Calculator                 | LangChain Calculator Tool            | Performs mathematical calculations                | AI Agent                      | AI Agent                     | Calculator allows agent to run math calculations.                                                  |
| Get all transactions       | Google Sheets Tool                   | Retrieves all transactions                        | AI Agent                      | AI Agent                     | Change the URL of the Sheets file in all Sheets tools.                                            |
| Get transactions by product name | Google Sheets Tool             | Retrieves transactions filtered by product name | AI Agent                      | AI Agent                     | Change the URL of the Sheets file in all Sheets tools.                                            |
| Get transactions by status | Google Sheets Tool                   | Retrieves transactions filtered by status        | AI Agent                      | AI Agent                     | Change the URL of the Sheets file in all Sheets tools.                                            |
| Records by date            | LangChain Tool Workflow (Sub-workflow) | Retrieves records filtered by date and status    | AI Agent                      | AI Agent                     | Special tool to call sub-workflow for date-filtered records. See [docs](https://docs.n8n.io/flow-logic/subworkflows/) |
| When Executed by Another Workflow | Execute Workflow Trigger       | Sub-workflow entry point                          | Records by date               | Google Sheets request        | Sub-workflow automatically returns last executed node result to AI.                              |
| Google Sheets request      | HTTP Request                        | Custom request to Google Sheets API with date filter | When Executed by Another Workflow | Code                        | Sends complex query to Google Sheets API to filter by date range.                                 |
| Code                       | Code (JavaScript)                   | Parses and transforms Google Sheets API response | Google Sheets request         | Filter by status             | Parses JSONP response and converts dates to ISO format.                                           |
| Filter by status           | Filter                             | Filters records by status                         | Code                         | Aggregate                    |                                                                                                    |
| Aggregate                  | Aggregate                         | Aggregates multiple items into one               | Filter by status              | When Executed by Another Workflow (returns to AI) | To send all items back to AI, aggregate into one item to avoid AI processing only first item.     |
| Sticky Note                | Sticky Note                       | Various instructional and contextual notes       | -                             | -                            | Multiple sticky notes provide setup instructions, usage tips, and author info. See section 5.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "When chat message received"**  
   - Type: LangChain Chat Trigger  
   - Configure webhook to receive chat messages.

2. **Create AI Agent Node: "AI Agent"**  
   - Type: LangChain Agent  
   - Set max iterations to 15.  
   - System message: "You are a helpful assistant. Current timestamp is {{ $now }}".  
   - Connect input from "When chat message received".  
   - Connect outputs to tools (Calculator, Data Retrieval Tools, OpenAI Chat Model, Buffer Memory).

3. **Create OpenAI Chat Model Node: "OpenAI Chat Model"**  
   - Type: LangChain OpenAI Chat Model  
   - Model: GPT-4o (or preferred model)  
   - Temperature: 0.2  
   - Add OpenAI API credentials.  
   - Connect output to AI Agent as language model.

4. **Create Buffer Memory Node: "Buffer Memory"**  
   - Type: LangChain Memory Buffer Window  
   - Default settings (window size 5).  
   - Connect output to AI Agent as memory.

5. **Create Calculator Node: "Calculator"**  
   - Type: LangChain Calculator Tool  
   - Default settings.  
   - Connect output to AI Agent as tool.

6. **Create Google Sheets Tools:**  
   - For each tool, configure Google Sheets OAuth2 credentials and set the document URL and sheet GID.  
   - **Get all transactions:** No filters, returns all data. Connect output to AI Agent as tool.  
   - **Get transactions by product name:** Filter on "Product" column using AI variable `product_name`. Connect output to AI Agent as tool.  
   - **Get transactions by status:** Filter on "Status" column using AI variable `transaction_status`. Connect output to AI Agent as tool.

7. **Create Sub-Workflow for Date-Filtered Records:**  
   - Add a new workflow section with trigger node: "When Executed by Another Workflow"  
     - Accept inputs: start_date, end_date, status.  
   - Add HTTP Request node: "Google Sheets request"  
     - URL: Google Sheets visualization API endpoint for the spreadsheet.  
     - Query parameters: sheet=Sheet1, tq=SQL-like query filtering dates between start_date and end_date.  
     - Authentication: Google Sheets OAuth2.  
   - Add Code node: "Code"  
     - JavaScript code to parse JSONP response, extract JSON, convert date strings to ISO format, and output JSON objects.  
   - Add Filter node: "Filter by status"  
     - Condition: item.Status contains input status parameter.  
   - Add Aggregate node: "Aggregate"  
     - Aggregate all filtered items into one.  
   - Connect nodes in order: Trigger → Google Sheets request → Code → Filter by status → Aggregate.  
   - Configure sub-workflow to return last node output to caller.

8. **Create LangChain Tool Workflow Node: "Records by date"**  
   - Type: LangChain Tool Workflow  
   - Set workflow ID to the sub-workflow created above.  
   - Map inputs from AI variables: start_date, end_date, status.  
   - Connect output to AI Agent as tool.

9. **Connect all tools (Calculator, Get all transactions, Get transactions by product name, Get transactions by status, Records by date) as tools input to AI Agent.**

10. **Test the workflow:**  
    - Use the chat interface to send queries such as:  
      - "How many refunds in January and what was the amount refunded?"  
      - "What is the most frequent reason for refunds?"  
    - Verify data retrieval, filtering, calculation, and AI responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Copy this Sheets file to your Google Drive to use with the workflow: https://docs.google.com/spreadsheets/d/18A4d7KYrk8-uEMbu7shoQe_UIzmbTLV1FMN43bjA7qc/edit#gid=0 | Setup instruction for data source.                                                             |
| How to connect to Google Sheets? Requires Google OAuth credentials. See documentation: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/ | Credential setup for Google Sheets OAuth2.                                                     |
| The AI Tools Agent uses the `$fromAI` function to dynamically pass parameters from AI to tools. See docs: https://docs.n8n.io/advanced-ai/examples/using-the-fromai-function/ | Explanation of AI parameter passing mechanism.                                                 |
| Sub-workflows allow modular workflow calls within the same workflow. See documentation: https://docs.n8n.io/flow-logic/subworkflows/ | Explanation of sub-workflow usage.                                                             |
| Author: Solomon, freelance consultant specializing in automations and data analysis. More templates: https://n8n.io/creators/solomon/ | Author and additional resources.                                                               |
| For help, create a topic on n8n community forums: https://community.n8n.io/c/questions/                        | Support resource.                                                                               |
| Some example questions to try with the AI: How many refunds in January? What was the amount refunded? What is the most frequent reason for refunds? | Usage examples to test the chatbot.                                                            |

---

This document provides a detailed, structured reference for understanding, reproducing, and extending the "Build Your First AI Data Analyst Chatbot" workflow in n8n. It covers all nodes, their roles, configurations, and interconnections, enabling both human developers and AI agents to work effectively with the workflow.