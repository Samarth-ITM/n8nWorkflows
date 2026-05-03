Visualize your SQL Agent queries with OpenAI and Quickchart.io

https://n8nworkflows.xyz/workflows/visualize-your-sql-agent-queries-with-openai-and-quickchart-io-2559


# Visualize your SQL Agent queries with OpenAI and Quickchart.io

### 1. Workflow Overview

This workflow provides enhanced data analysis and visualization capabilities by integrating a native SQL Agent with OpenAI’s Structured Output and Quickchart.io. It enables business users to interact via chat, query a SQL database, and receive answers supplemented by dynamically generated charts when appropriate.

The workflow is logically split into the following blocks:

- **1.1 Input Reception and Extraction:** Capture user chat messages and extract the core question excluding chart-related content.
- **1.2 SQL Agent Querying and Response Generation:** Use the extracted question to query the connected SQL database and produce a human-readable textual answer.
- **1.3 Chart Necessity Classification:** Determine if the SQL Agent’s response would benefit from visual representation.
- **1.4 Chart Generation Sub-Workflow:** If a chart is warranted, invoke a sub-workflow that requests a Chart.js definition from OpenAI and constructs a chart URL using Quickchart.io.
- **1.5 Final Response Assembly and Output:** Compose and output the final message, either plain text or text combined with the chart image.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Extraction

- **Overview:**  
This block listens for incoming chat messages and extracts the user’s core question, filtering out any chart-related instructions to avoid confusion for the SQL Agent.

- **Nodes Involved:**  
  - When chat message received  
  - Information Extractor - User question  

- **Node Details:**

  - **When chat message received**  
    - *Type:* LangChain chat trigger node  
    - *Role:* Entry point webhook node receiving public user chat input  
    - *Config:* Public webhook enabled; listens for chat messages  
    - *Inputs:* External chat request  
    - *Outputs:* Passes entire chat input JSON downstream  
    - *Potential Failures:* Webhook downtime, malformed or empty input messages  

  - **Information Extractor - User question**  
    - *Type:* LangChain Information Extractor  
    - *Role:* Extracts the core user question, omitting any chart-related text  
    - *Config:* Uses attribute named `user_question`; input text is the raw chat input from the previous node  
    - *Expressions:* `={{ $json.chatInput }}` as input text, outputs `user_question` string attribute  
    - *Inputs:* From chat message node  
    - *Outputs:* Extracted user question for SQL Agent  
    - *Edge Cases:* If user input is ambiguous or heavily focused on charts, extraction may fail or produce incomplete question  

---

#### 2.2 SQL Agent Querying and Response Generation

- **Overview:**  
This block leverages an AI SQL Agent to convert the extracted user question into a valid SQL query, execute it on the connected database, and return a natural language answer.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Window Buffer Memory  
  - AI Agent  

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI chat model node  
    - *Role:* Provides GPT-4o model for AI interactions with low temperature for deterministic results  
    - *Config:* Model `"gpt-4o"`, temperature `0.1`  
    - *Credentials:* OpenAI API key configured  
    - *Inputs:* Receives AI prompts from AI Agent or memory node  
    - *Outputs:* AI-generated text responses  
    - *Edge Cases:* API rate limits, network timeouts, or model unavailability  

  - **Window Buffer Memory**  
    - *Type:* LangChain memory buffer (window)  
    - *Role:* Maintains session context using session key from the chat message for conversational continuity  
    - *Config:* Session key tied to chat session ID (`$('When chat message received').item.json.sessionId`)  
    - *Inputs:* Receives chat history and AI outputs  
    - *Outputs:* Provides memory context to AI Agent  
    - *Edge Cases:* Memory overflow, session key mismatch causing context loss  

  - **AI Agent**  
    - *Type:* LangChain SQL Agent  
    - *Role:* Main AI agent that takes user question, formulates SQL query, executes it on PostgreSQL DB, and returns answer  
    - *Config:*  
      - Uses SQL dialect specific to PostgreSQL  
      - Prefix prompt instructs to generate syntactically correct queries, limit results, avoid DML statements, and only use relevant columns  
      - Includes multi-step reasoning instructions and answer formatting requirements  
    - *Credentials:* PostgreSQL database connection (e.g., Supabase DB with coffee sales dataset)  
    - *Inputs:* Receives extracted `user_question` and memory context  
    - *Outputs:* Produces a textual answer and raw SQL output for further processing  
    - *Edge Cases:* SQL syntax errors, query execution failures, empty result sets, unauthorized DB access  

---

#### 2.3 Chart Necessity Classification

- **Overview:**  
Evaluates whether the SQL Agent’s textual response would benefit from a chart visualization based on the user's original request and the query output.

- **Nodes Involved:**  
  - OpenAI Chat Model Classifier  
  - Text Classifier - Chart required?  

- **Node Details:**

  - **OpenAI Chat Model Classifier**  
    - *Type:* LangChain OpenAI chat model node  
    - *Role:* Provides OpenAI model with temperature 0.2 to classify text into categories  
    - *Credentials:* OpenAI API key  
    - *Inputs:* Receives text for classification (user request + data output)  
    - *Outputs:* Classification label used to branch workflow  
    - *Potential Failures:* Misclassification, API errors  

  - **Text Classifier - Chart required?**  
    - *Type:* LangChain text classifier  
    - *Role:* Categorizes input as either `chart_required` or `chart_not_required`  
    - *Config:*  
      - Input text includes user request and SQL data output  
      - Categories clearly defined with descriptions  
    - *Inputs:* SQL Agent response and user request  
    - *Outputs:* Two outputs:  
      - If `chart_required`, proceeds to chart sub-workflow  
      - If `chart_not_required`, proceeds to output plain text response  
    - *Edge Cases:* Ambiguous or borderline cases where chart usefulness is subjective  

---

#### 2.4 Chart Generation Sub-Workflow

- **Overview:**  
When a chart is needed, this sub-workflow dynamically requests a Chart.js JSON definition from OpenAI using Structured Output, then builds a Quickchart.io URL for rendering.

- **Nodes Involved:**  
  - Execute "Generate a chart" tool (Sub-workflow trigger)  
  - OpenAI - Generate Chart definition with Structured Output (HTTP Request)  
  - Set response  

- **Node Details:**

  - **Execute "Generate a chart" tool**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Invokes a sub-workflow (the same workflow ID here) with `waitForSubWorkflow` enabled to ensure synchronous execution  
    - *Inputs:* Receives user question and SQL Agent output from main workflow  
    - *Outputs:* Returns chart JSON definition after OpenAI call and URL construction  
    - *Edge Cases:* Sub-workflow errors, long execution timeouts  

  - **OpenAI - Generate Chart definition with Structured Output**  
    - *Type:* HTTP Request (OpenAI chat completions API)  
    - *Role:* Sends user question and data to OpenAI GPT-4o-2024-08-06 model with a system prompt instructing to generate a valid Chart.js JSON definition  
    - *Config:*  
      - POST to `https://api.openai.com/v1/chat/completions`  
      - `response_format` uses a strict JSON schema for Chart.js configuration (includes chart type, data labels, datasets, options with proper typing)  
      - Headers include `Content-Type: application/json` and OpenAI API key  
    - *Inputs:* JSON with user question and SQL output data  
    - *Outputs:* JSON string containing Chart.js configuration  
    - *Edge Cases:* Invalid JSON output from AI, schema validation errors, API errors  

  - **Set response**  
    - *Type:* Set node  
    - *Role:* Constructs a Quickchart.io URL embedding the Chart.js JSON definition (URL-encoded) and setting chart width to 200  
    - *Config:* Sets `output` to `"https://quickchart.io/chart?width=200&c=" + encodeURIComponent(chartJSON)`  
    - *Inputs:* Chart JSON from previous HTTP Request node  
    - *Outputs:* Final chart image URL sent back to main workflow  
    - *Edge Cases:* URL encoding issues, oversized charts, Quickchart display limitations  

---

#### 2.5 Final Response Assembly and Output

- **Overview:**  
Depending on the classification, either outputs the SQL Agent’s textual response alone or appends the generated chart image URL to the response, then delivers it to the user.

- **Nodes Involved:**  
  - User question + Agent initial response  
  - Execute Workflow  
  - Set Text output  
  - Set Text + Chart output  

- **Node Details:**

  - **User question + Agent initial response**  
    - *Type:* Set node  
    - *Role:* Combines the original user question and SQL Agent’s output into a single JSON object for downstream use  
    - *Config:* Assigns two fields: `user_question` from chat input, and `output` from SQL Agent’s response  
    - *Inputs:* From Text Classifier node  
    - *Outputs:* Prepared input for sub-workflow or final output nodes  
    - *Edge Cases:* Data consistency errors if upstream nodes fail  

  - **Execute Workflow**  
    - *Type:* Execute Workflow node  
    - *Role:* Invokes the same workflow as a sub-workflow to generate the chart synchronously  
    - *Config:* Wait for sub-workflow completion is enabled; workflow ID is self-reference  
    - *Inputs:* Receives combined user question and SQL Agent output  
    - *Outputs:* Returns chart URL and data for final message  
    - *Edge Cases:* Recursive call risks if not properly managed, sub-workflow failures  

  - **Set Text output**  
    - *Type:* Set node  
    - *Role:* Passes the SQL Agent’s textual output as-is when no chart is required  
    - *Config:* Sets `output` field to the SQL Agent’s output string  
    - *Inputs:* From Text Classifier node branch `chart_not_required`  
    - *Outputs:* Final plain text output  
    - *Edge Cases:* Empty or null outputs  

  - **Set Text + Chart output**  
    - *Type:* Set node  
    - *Role:* Appends the Quickchart.io image markdown to the SQL Agent’s textual output for chart visualization  
    - *Config:* Sets `output` to SQL Agent’s output plus markdown image syntax referencing the chart URL  
    - *Inputs:* From Execute Workflow node (chart URL)  
    - *Outputs:* Final combined text + chart output  
    - *Edge Cases:* Markdown rendering issues in some clients, broken image URLs  

---

### 3. Summary Table

| Node Name                                | Node Type                              | Functional Role                                | Input Node(s)                       | Output Node(s)                          | Sticky Note                                                                                          |
|-----------------------------------------|--------------------------------------|-----------------------------------------------|-----------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received               | LangChain chat trigger                | Receive user chat input                        | External webhook                  | Information Extractor - User question  |                                                                                                    |
| Information Extractor - User question    | LangChain Information Extractor       | Extract core user question                      | When chat message received        | AI Agent                              | See Sticky Note2: Information Extractor details                                                     |
| OpenAI Chat Model                        | LangChain OpenAI chat model           | Provide AI language model for SQL Agent       | AI Agent (ai_languageModel)       | AI Agent                              |                                                                                                    |
| Window Buffer Memory                    | LangChain memory buffer window        | Maintain conversational context                | When chat message received        | AI Agent                              |                                                                                                    |
| AI Agent                               | LangChain SQL Agent                   | Generate SQL queries and produce answer       | Information Extractor - User question, Window Buffer Memory | Text Classifier - Chart required? | See Sticky Note2: SQL Agent details                                                                |
| OpenAI Chat Model Classifier            | LangChain OpenAI chat model           | Classify if chart visualization is needed     | Text Classifier - Chart required? | Text Classifier - Chart required?      |                                                                                                    |
| Text Classifier - Chart required?       | LangChain text classifier             | Decide if chart visualization is required     | AI Agent                         | User question + Agent initial response, Set Text output | See Sticky Note3: Chart decision logic                                                             |
| User question + Agent initial response  | Set                                  | Combine user question and AI Agent output      | Text Classifier - Chart required? | Execute Workflow                      |                                                                                                    |
| Execute Workflow                        | Execute Workflow (sub-workflow)       | Invoke chart generation sub-workflow           | User question + Agent initial response | Set Text + Chart output              |                                                                                                    |
| Set Text output                        | Set                                  | Output plain text response                      | Text Classifier - Chart required? | (Workflow output)                     |                                                                                                    |
| Set Text + Chart output                 | Set                                  | Output text plus appended chart image          | Execute Workflow                  | (Workflow output)                     |                                                                                                    |
| Execute "Generate a chart" tool          | Execute Workflow Trigger              | Trigger sub-workflow for chart generation      | (Internal)                      | OpenAI - Generate Chart definition with Structured Output | See Sticky Note: Chart generation details and link to original template                            |
| OpenAI - Generate Chart definition with Structured Output | HTTP Request                         | Request Chart.js JSON definition from OpenAI  | Execute "Generate a chart" tool   | Set response                        | See Sticky Note: Detailed explanation of HTTP Request node and Chart.js schema                    |
| Set response                           | Set                                  | Build Quickchart.io URL with embedded chart    | OpenAI - Generate Chart definition with Structured Output | Execute Workflow                    |                                                                                                    |
| Sticky Note1                          | Sticky Note                         | Overview and workflow explanation               | None                            | None                                | Extensive overview and usage instructions                                                          |
| Sticky Note2                          | Sticky Note                         | Information Extractor and SQL Agent explanation | None                            | None                                | See block 2.1 and 2.2 details                                                                      |
| Sticky Note3                          | Sticky Note                         | Chart decision explanation                       | None                            | None                                | See block 2.3 details                                                                              |
| Sticky Note                           | Sticky Note                         | Chart generation sub-workflow explanation       | None                            | None                                | See block 2.4 details and links                                                                    |
| Sticky Note4                          | Sticky Note                         | Visual demo screenshot                           | None                            | None                                | Visual demo image                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a `LangChain Chat Trigger` node named **When chat message received**.  
   - Set it as a public webhook to receive user chat messages.  

2. **Add Information Extraction:**  
   - Add a `LangChain Information Extractor` node named **Information Extractor - User question**.  
   - Configure it to extract an attribute named `user_question` from the incoming chat input, filtering out chart-related text.  
   - Connect the trigger node output to this node.  

3. **Configure SQL Agent:**  
   - Add `LangChain OpenAI Chat Model` node named **OpenAI Chat Model**.  
   - Set model to `"gpt-4o"` with temperature `0.1`.  
   - Add credentials for OpenAI API.  
   - Add `Window Buffer Memory` node named **Window Buffer Memory** with session key linked to the chat session ID from the trigger node.  
   - Add `LangChain Agent` node named **AI Agent** configured as an SQL Agent:  
     - Set the agent prompt to extract the question, create syntactically correct SQL queries with a limit clause, avoid DML, and format answers as instructed in the prefix prompt.  
     - Connect the output of Information Extractor and Window Buffer Memory to AI Agent.  
     - Configure credentials with PostgreSQL (or your DB) connection.  

4. **Add Chart Necessity Classifier:**  
   - Add `LangChain OpenAI Chat Model` node named **OpenAI Chat Model Classifier** with temperature `0.2` and OpenAI credentials.  
   - Add `LangChain Text Classifier` node named **Text Classifier - Chart required?** with two categories: `chart_required` and `chart_not_required`.  
   - Input text should combine user request and SQL Agent output.  
   - Connect AI Agent output to classifier chain: AI Agent → OpenAI Chat Model Classifier → Text Classifier.  

5. **Prepare User Question and Output Set:**  
   - Add a `Set` node named **User question + Agent initial response**.  
   - Assign `user_question` to the chat input and `output` to AI Agent’s output.  
   - Connect from Text Classifier (chart_required output) to this node.  

6. **Setup Sub-Workflow Execution for Chart Generation:**  
   - Add `Execute Workflow` node named **Execute Workflow**.  
   - Configure it to call the current workflow ID with `waitForSubWorkflow` enabled for synchronous operation.  
   - Connect **User question + Agent initial response** to this node.  

7. **Set Text Outputs:**  
   - Add `Set` node **Set Text output** for the `chart_not_required` branch.  
   - Assign output to AI Agent’s output string directly.  
   - Connect from Text Classifier (chart_not_required output) to this node.  

8. **Set Text + Chart Output:**  
   - Add `Set` node **Set Text + Chart output**.  
   - Assign output to combine AI Agent’s output text plus markdown image link referencing the chart URL from the sub-workflow.  
   - Connect from **Execute Workflow** node to this node.  

9. **Create Chart Generation Sub-Workflow:**  
   - Add `Execute Workflow Trigger` node **Execute "Generate a chart" tool** to trigger the chart generation portion.  
   - Connect from **Execute Workflow** node or use as entry in the sub-workflow.  

10. **Add HTTP Request Node for Chart Definition:**  
    - Add `HTTP Request` node named **OpenAI - Generate Chart definition with Structured Output**.  
    - Configure:  
      - POST to `https://api.openai.com/v1/chat/completions`  
      - JSON body includes system prompt instructing chart generation, user content with question and SQL data, and strict JSON schema for Chart.js config.  
      - Headers: `Content-Type: application/json`, OpenAI API key credential.  
    - Connect from **Execute "Generate a chart" tool** node.  

11. **Add Set Node for Chart URL:**  
    - Add `Set` node named **Set response**.  
    - Configure to create `output` string as: `"https://quickchart.io/chart?width=200&c=" + encodeURIComponent(chartJSON)` where `chartJSON` is the OpenAI response content.  
    - Connect from HTTP Request node.  

12. **Finalize Connections:**  
    - Ensure all nodes are connected as per the logical flow above.  
    - Test with sample queries to verify text-only and text+chart outputs.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses OpenAI’s Structured Output feature to enforce strict JSON schema compliance when generating Chart.js definitions. | OpenAI documentation on Structured Output: https://platform.openai.com/docs/guides/structured-output                             |
| Quickchart.io URL parameters and usage details are essential for rendering charts properly.                                           | Quickchart.io documentation: https://quickchart.io/documentation/usage/parameters/                                                |
| Dataset example used: Kaggle Coffee Sales uploaded to Supabase PostgreSQL.                                                           | Kaggle dataset: https://www.kaggle.com/datasets/ihelon/coffee-sales/versions/15?resource=download                                 |
| SQLite alternative workflow template referenced for users preferring SQLite binaries.                                                | SQLite template workflow: https://n8n.io/workflows/2292-talk-to-your-sqlite-database-with-a-langchain-ai-agent/                  |
| Known limitations: Some chart types such as radar charts may display improperly due to size constraints in Quickchart.io.            | Mentioned in Sticky Note1 and overview section                                                                                    |
| Original workflow inspiration and template available at:                                                                             | https://n8n.io/workflows/2400-ai-agent-with-charts-capabilities-using-openai-structured-output-and-quickchart/                   |
| Demo GIF of SQL Agent interaction illustrating the workflow in action.                                                               | ![Demo SQL Agent](https://media.licdn.com/dms/image/v2/D4E22AQERT4FEXEUncw/feedshare-shrink_800/feedshare-shrink_800/0/1731433289953?e=1741824000&v=beta&t=e6xUqjcsSq5U_NELeD-nn1mFROGYZLazkYC0eELTv5Y) |

---

This completes a thorough, structured reference of the "Visualize your SQL Agent queries with OpenAI and Quickchart.io" workflow, facilitating understanding, reproduction, and modification by users and automation agents alike.