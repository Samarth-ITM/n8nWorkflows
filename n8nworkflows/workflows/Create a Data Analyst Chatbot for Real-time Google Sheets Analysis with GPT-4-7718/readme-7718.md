Create a Data Analyst Chatbot for Real-time Google Sheets Analysis with GPT-4

https://n8nworkflows.xyz/workflows/create-a-data-analyst-chatbot-for-real-time-google-sheets-analysis-with-gpt-4-7718


# Create a Data Analyst Chatbot for Real-time Google Sheets Analysis with GPT-4

### 1. Workflow Overview

This workflow creates a smart AI-powered Data Analyst Chatbot that performs real-time analysis of data stored in Google Sheets using GPT-4 (actually configured here with GPT-5). It is designed for business intelligence scenarios where users interact via chat to query live data about products, customers, and orders. The workflow fetches fresh data on every chat message, aggregates multiple data sources, maintains conversational context, and uses an AI agent to provide insightful, context-aware responses.

The workflow is logically divided into the following blocks:

- **1.1 Chat Input Reception:** Triggered when a chat message is received, capturing user input and session ID.
- **1.2 Data Retrieval:** Parallel fetching of live data from multiple Google Sheets (Products, Customers, Orders).
- **1.3 Data Aggregation:** Aggregates raw data from each sheet into structured JSON objects.
- **1.4 Data Merging:** Combines aggregated data streams into a single comprehensive data object.
- **1.5 AI Processing:** Uses OpenAI’s GPT-5 model, a memory buffer for session context, and a Data Analyst AI Agent to process input and analyze data.
- **1.6 Response Delivery:** The AI Agent generates the final analytical response for the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Input Reception

- **Overview:**  
  Listens for incoming chat messages from users, capturing the chat text and session ID for context-aware processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: Chat Trigger (Langchain)  
    - Role: Entry point; receives chat messages via webhook with session tracking  
    - Configuration: No special options; webhookId set for unique webhook endpoint  
    - Inputs: External chat interface via webhook  
    - Outputs: Triggers downstream nodes with JSON containing `chatInput` and `sessionId`  
    - Edge cases: Webhook misconfiguration; missing sessionId; malformed chat messages  
    - Version: 1.1

#### 2.2 Data Retrieval

- **Overview:**  
  Fetches live data in parallel from three Google Sheets tabs: Products, Customers, and Orders.

- **Nodes Involved:**  
  - Google Sheets - Get Products Data  
  - Google Sheets - Get Customers Data  
  - Google Sheets - Get Orders Data

- **Node Details:**  
  - **Google Sheets - Get Products Data**  
    - Type: Google Sheets  
    - Role: Retrieves all rows from the "Products" sheet  
    - Configuration: Document ID set by Google Sheets URL; sheet name set to "Products"  
    - Credentials: OAuth2 for Google Sheets with appropriate access  
    - Outputs: Raw rows from "Products" sheet  
    - Edge cases: OAuth token expiration; sheet name mismatch; empty or malformed data  
    - Version: 4.5

  - **Google Sheets - Get Customers Data**  
    - Same as above but targets "Customers" sheet.

  - **Google Sheets - Get Orders Data**  
    - Same as above but targets "Orders" sheet.

#### 2.3 Data Aggregation

- **Overview:**  
  Aggregates all rows retrieved from each Google Sheets node into structured JSON arrays under designated field names.

- **Nodes Involved:**  
  - Aggregate Data 1 (for Products)  
  - Aggregate Data 2 (for Customers)  
  - Aggregate Data 3 (for Orders)

- **Node Details:**  
  - Each Aggregate node:  
    - Type: Aggregate  
    - Role: Converts multiple input items into a single array under a field (e.g., "products")  
    - Configuration: Aggregate all item data; destinationFieldName set accordingly  
    - Inputs: Output from corresponding Google Sheets node  
    - Outputs: Single JSON object with aggregated data field  
    - Edge cases: Empty input arrays; partial data leading to incomplete aggregation  
    - Version: 1

#### 2.4 Data Merging

- **Overview:**  
  Combines the aggregated data from all three sources into one JSON object to pass to the AI Agent.

- **Nodes Involved:**  
  - Merge

- **Node Details:**  
  - Merge  
    - Type: Merge  
    - Role: Combines multiple inputs into a single cohesive data object  
    - Configuration: Mode set to "combine" by position; expects 3 inputs  
    - Inputs: Aggregated products, customers, and orders data streams  
    - Outputs: Single combined JSON object containing all data arrays  
    - Edge cases: Missing inputs due to upstream failure; mismatch in input counts  
    - Version: 3.1

#### 2.5 AI Processing

- **Overview:**  
  Processes user input and aggregated data using an AI agent with memory and a language model to generate intelligent data analysis responses.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Simple Memory  
  - Data Analyst AI Agent

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides GPT-5 based language model capabilities to AI Agent  
    - Configuration: Model set to "gpt-5"; no additional options  
    - Credentials: OpenAI API key configured  
    - Inputs: Connected as ai_languageModel input to AI Agent  
    - Edge cases: API rate limits; network issues; invalid API key  
    - Version: 1.2

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains conversational context per session using sessionKey  
    - Configuration: sessionKey dynamically set to sessionId from chat trigger  
    - Inputs: Connected as ai_memory input to AI Agent  
    - Outputs: Provides context memory for conversation  
    - Edge cases: Missing sessionId; memory overflow or truncation in long conversations  
    - Version: 1.3

  - **Data Analyst AI Agent**  
    - Type: Langchain Agent  
    - Role: Core AI logic combining user input, memory, and data to generate responses  
    - Configuration:  
      - Text input is user chat input from trigger  
      - System message prompt includes detailed instructions and embedded JSON data for Products, Customers, Orders  
      - Prompt type is "define" to use custom prompt  
    - Inputs: Receives ai_languageModel, ai_memory, and merged data input streams  
    - Outputs: Analytical chatbot response  
    - Edge cases: Expression failures in prompt templating; large data causing token limits; ambiguous user queries  
    - Version: 1.9

#### 2.6 Response Delivery

- **Overview:**  
  The AI Agent node’s output is the final analytical response sent back to the chat interface (via the Langchain chat trigger integration).

- **Nodes Involved:**  
  - Data Analyst AI Agent (output)

- **Node Details:**  
  - Output from AI Agent is delivered to the chat interface automatically by the Langchain integration.  
  - Edge cases: Downstream chat interface errors; latency causing poor user experience.

---

### 3. Summary Table

| Node Name                   | Node Type                              | Functional Role                        | Input Node(s)                | Output Node(s)           | Sticky Note                                                                                                  |
|-----------------------------|--------------------------------------|-------------------------------------|-----------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received   | @n8n/n8n-nodes-langchain.chatTrigger| Entry point; receives chat messages | External webhook             | Google Sheets nodes       | Workflow Configurations: Update Google Sheets document ID, sheet names, AI prompt, and webhook ID.           |
| Google Sheets - Get Products Data | n8n-nodes-base.googleSheets        | Fetches "Products" sheet data       | When chat message received   | Aggregate Data 1          | Google Sheet - Get Data: Update document URL, sheet name, adjust nodes based on use case.                     |
| Google Sheets - Get Customers Data| n8n-nodes-base.googleSheets        | Fetches "Customers" sheet data      | When chat message received   | Aggregate Data 2          | Google Sheet - Get Data: Update document URL, sheet name, adjust nodes based on use case.                     |
| Google Sheets - Get Orders Data   | n8n-nodes-base.googleSheets        | Fetches "Orders" sheet data         | When chat message received   | Aggregate Data 3          | Google Sheet - Get Data: Update document URL, sheet name, adjust nodes based on use case.                     |
| Aggregate Data 1             | n8n-nodes-base.aggregate             | Aggregates Products data             | Google Sheets - Get Products Data | Merge                  |                                                                                                              |
| Aggregate Data 2             | n8n-nodes-base.aggregate             | Aggregates Customers data            | Google Sheets - Get Customers Data | Merge                  |                                                                                                              |
| Aggregate Data 3             | n8n-nodes-base.aggregate             | Aggregates Orders data               | Google Sheets - Get Orders Data | Merge                    |                                                                                                              |
| Merge                       | n8n-nodes-base.merge                 | Combines aggregated data streams    | Aggregate Data 1,2,3          | Data Analyst AI Agent     |                                                                                                              |
| OpenAI Chat Model            | @n8n/n8n-nodes-langchain.lmChatOpenAi| Provides GPT-5 language model       | None                        | Data Analyst AI Agent (ai_languageModel) |                                                                                                              |
| Simple Memory                | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context      | None                        | Data Analyst AI Agent (ai_memory) |                                                                                                              |
| Data Analyst AI Agent        | @n8n/n8n-nodes-langchain.agent       | Core AI logic for data analysis     | Merge, OpenAI Chat Model, Simple Memory | (final output)         | Data Analyst AI Agent: Update system message prompt based on data sources and use case.                       |
| Sticky Note                 | n8n-nodes-base.stickyNote             | Informational notes                  | None                        | None                     | Multiple sticky notes provide context about configuration, workflow overview, and credits (see below).       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhookId uniquely for your implementation  
   - No additional parameters needed  
   - Purpose: Receive chat messages with session IDs

2. **Create Google Sheets Nodes (3 total)**  
   - Node 1: Google Sheets - Get Products Data  
     - Type: `n8n-nodes-base.googleSheets`  
     - Set Document ID to your Google Sheets URL  
     - Set Sheet Name to "Products"  
     - Authenticate with Google Sheets OAuth2 credentials  
   - Node 2: Google Sheets - Get Customers Data  
     - Same as above, sheet name "Customers"  
   - Node 3: Google Sheets - Get Orders Data  
     - Same as above, sheet name "Orders"

3. **Create Aggregate Nodes (3 total)**  
   - Aggregate Data 1:  
     - Type: `n8n-nodes-base.aggregate`  
     - Set aggregate mode to "aggregateAllItemData"  
     - Destination field: "products"  
     - Connect input from Google Sheets - Get Products Data  
   - Aggregate Data 2:  
     - Destination field: "customers"  
     - Connect input from Google Sheets - Get Customers Data  
   - Aggregate Data 3:  
     - Destination field: "orders"  
     - Connect input from Google Sheets - Get Orders Data

4. **Create Merge Node**  
   - Type: `n8n-nodes-base.merge`  
   - Mode: "combine"  
   - Combine by: "combineByPosition"  
   - Number of inputs: 3  
   - Connect inputs from Aggregate Data 1, 2, and 3 nodes

5. **Create OpenAI Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Set model to "gpt-5" (or GPT-4 if preferred)  
   - Provide OpenAI API credentials

6. **Create Simple Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set sessionKey to `{{$json["sessionId"]}}` dynamically from the chat trigger node  
   - Purpose: Maintain session conversation context

7. **Create Data Analyst AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set `text` parameter to user input expression: `User Input:\n{{$json["chatInput"]}}`  
   - Set system message prompt to a multi-line string embedding the aggregated JSON arrays:  
     ```
     You are a helpful AI Data Analyst.  
     Use the provided JSON data to analyze and answer questions.  

     Products:  
     {{$json["products"].toJsonString()}}  

     Customers:  
     {{$json["customers"].toJsonString()}}  

     Orders:  
     {{$json["orders"].toJsonString()}}  

     Always use the data above when performing analysis.
     ```  
   - Set promptType to "define"  
   - Connect inputs:  
     - `main` from Merge node (merged aggregated data)  
     - `ai_languageModel` from OpenAI Chat Model node  
     - `ai_memory` from Simple Memory node

8. **Connect Nodes in Sequence**  
   - `When chat message received` → Google Sheets nodes (parallel)  
   - Each Google Sheets node → corresponding Aggregate node  
   - Aggregate nodes → Merge node  
   - Merge node → Data Analyst AI Agent (main input)  
   - OpenAI Chat Model node → Data Analyst AI Agent (ai_languageModel input)  
   - Simple Memory node → Data Analyst AI Agent (ai_memory input)

9. **Credential Setup**  
   - Google Sheets nodes require OAuth2 credentials with read access to the spreadsheet  
   - OpenAI Chat Model requires valid OpenAI API key with GPT-5 or GPT-4 access

10. **Final Testing**  
    - Send chat messages via the configured webhook  
    - Confirm live data is fetched from Google Sheets and combined  
    - Ensure AI Agent returns relevant analytical answers based on data and chat input  
    - Validate session memory maintains context across multiple messages

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Workflow Configurations: Update Google Sheets document ID and sheet names to point to your own data source. Adjust AI Agent system message and chat trigger webhook ID accordingly.                                                     | Sticky Note1 in workflow                                                                                           |
| Google Sheets - Get Data: Update the document URL and sheet name as needed. Add or remove Google Sheets nodes depending on your data sources.                                                                                         | Sticky Note                                                                                                        |
| This template builds an intelligent chatbot that fetches live data from Google Sheets on each chat message and uses GPT-5 for data analysis with contextual memory.                                                                    | Sticky Note2                                                                                                       |
| Workflow Process Overview: Stepwise explanation of data retrieval, aggregation, merging, AI analysis, and response delivery.                                                                                                            | Sticky Note3                                                                                                       |
| Author contact and credits: Billy - n8n workflow and AI automation expert. Contact via email or visit his n8n creator page for assistance or project collaborations.                                                                     | Sticky Note4, https://n8n.io/creators/billy and https://www.billychristi.com/n8n                                 |
| Data Analyst AI Agent: Customize the system message prompt to fit your specific data schema and analytical needs for best results.                                                                                                       | Sticky Note5                                                                                                       |
| Example Google Sheets template available to copy: https://docs.google.com/spreadsheets/d/1-QTFO3TbGFjtYOMUfZb0aY66J_8G-R0Rb0JHLWrEZ90/edit?gid=0#gid=0                                                                                   | Sticky Note1                                                                                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.