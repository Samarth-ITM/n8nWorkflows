Evaluate tool usage accuracy in multi-agent AI workflows using Evaluation nodes

https://n8nworkflows.xyz/workflows/evaluate-tool-usage-accuracy-in-multi-agent-ai-workflows-using-evaluation-nodes-5523


# Evaluate tool usage accuracy in multi-agent AI workflows using Evaluation nodes

### 1. Workflow Overview

This workflow is designed to evaluate the accuracy of tool usage within multi-agent AI workflows by leveraging n8n’s Evaluation nodes. It orchestrates a multi-agent AI system that processes chat queries, calls multiple specialized tools (search, calculator, summarizer, web search), and then assesses if the expected tools were correctly invoked according to a predefined dataset. The evaluation results are recorded back into a Google Sheets dataset for analysis.

Logical blocks within the workflow:

- **1.1 Input Reception**: Receives chat messages and dataset rows triggering the evaluation process.
- **1.2 Multi-Agent AI Processing**: Uses a LangChain agent that coordinates calls to various tools (search database, calculator, web search, summarizer) to answer queries.
- **1.3 Evaluation Logic**: Checks actual tool usage against expected tools from the dataset, sets evaluation metrics, and records outputs.
- **1.4 Integration and Data Flow**: Connects with Google Sheets for data input and output, and manages credentials for external APIs and services.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block handles incoming data through two triggers: chat messages (user queries) and dataset rows (expected tool usage and questions). It prepares input data for the multi-agent AI processing.

**Nodes Involved:**  
- When chat message received  
- When fetching a dataset row  
- Match chat format

**Node Details:**

- **When chat message received**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Listens for incoming chat messages via webhook.  
  - *Config:* Default parameters, webhook ID assigned.  
  - *IO:* Output passes chat message JSON to next node.  
  - *Failures:* Webhook connectivity issues, malformed input.  

- **When fetching a dataset row**  
  - *Type:* Evaluation Trigger (Google Sheets)  
  - *Role:* Triggers when a new row is fetched from a Google Sheets document containing tool expectations and questions.  
  - *Config:* Points to a specific Google Sheet and sheet tab by ID; uses OAuth2 credentials.  
  - *IO:* Outputs row data including "tools_to_call" and "question".  
  - *Failures:* Authentication errors, sheet access issues, sheet structure changes.  

- **Match chat format**  
  - *Type:* Set node  
  - *Role:* Normalizes the incoming question into a uniform `chatInput` string for downstream processing.  
  - *Config:* Assigns `chatInput` = `$json.question`.  
  - *IO:* Takes input from "When fetching a dataset row", outputs normalized data for agent.  
  - *Failures:* Expression failures if `question` is missing or malformed.

---

#### 2.2 Multi-Agent AI Processing

**Overview:**  
This core block runs a LangChain agent that coordinates calls to multiple tools to answer queries. The agent uses a system message to enforce rules on tool usage and returns intermediate steps to track tool calls.

**Nodes Involved:**  
- Search Agent  
- Search_db  
- Calculator  
- Web search  
- Summarizer  
- OpenRouter Chat Model  
- Embeddings OpenAI  

**Node Details:**

- **Search Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Central orchestrator that processes the chat input, deciding which tools to call and when.  
  - *Config:*  
    - System prompt instructs tool usage (search_db first, web search only once, calculator for math, summarizer mandatory at end).  
    - Returns intermediate steps for evaluation.  
  - *IO:* Inputs chat message and tool responses; outputs agent’s response and intermediate steps.  
  - *Failures:* Model API errors, tool invocation issues, incorrect intermediate step formatting.  

- **Search_db**  
  - *Type:* LangChain Vector Store Qdrant Tool  
  - *Role:* Retrieves relevant results from vector database as a tool for agent queries.  
  - *Config:* Uses Qdrant collection "search_queries"; retrieve-as-tool mode.  
  - *IO:* Accepts embedding queries, outputs search results to agent.  
  - *Failures:* Qdrant API errors, credential issues, empty results.  

- **Calculator**  
  - *Type:* LangChain Calculator Tool  
  - *Role:* Performs math computations on demand.  
  - *Config:* Default settings, no parameters.  
  - *IO:* Accepts math expressions, returns computed result.  
  - *Failures:* Invalid math expressions, calculation errors.  

- **Web search**  
  - *Type:* HTTP Request Tool  
  - *Role:* Performs web search via Firecrawl API when the search_db tool is insufficient.  
  - *Config:*  
    - POST request to Firecrawl search endpoint.  
    - Authenticated with HTTP Bearer token credential.  
    - Sends query parameter and limit=3.  
  - *IO:* Receives query from agent, returns web search results.  
  - *Failures:* API downtime, auth errors, rate limits.  

- **Summarizer**  
  - *Type:* LangChain Workflow Tool  
  - *Role:* Summarizes gathered information at the end of the agent’s process.  
  - *Config:* Invokes a separate "Summarizer Agent" workflow by ID, passing the query string.  
  - *IO:* Receives text input, returns summarized output.  
  - *Failures:* Sub-workflow unavailable, input/output mismatches.  

- **OpenRouter Chat Model**  
  - *Type:* LangChain OpenRouter Chat Model  
  - *Role:* Language model used by the agent for generating responses.  
  - *Config:* Uses model "openai/o3" via OpenRouter API with appropriate credentials.  
  - *IO:* Feeds language model response into the agent.  
  - *Failures:* API key invalid, rate limit exceeded, model errors.  

- **Embeddings OpenAI**  
  - *Type:* LangChain OpenAI Embeddings  
  - *Role:* Generates embeddings for queries to facilitate vector search.  
  - *Config:* Default OpenAI embedding options with credentials.  
  - *IO:* Produces embeddings for Search_db node.  
  - *Failures:* API errors, credential issues, invalid input.

---

#### 2.3 Evaluation Logic

**Overview:**  
This block compares the tools actually called by the agent against those expected per dataset, calculates evaluation metrics, and writes results back to Google Sheets.

**Nodes Involved:**  
- Evaluating?  
- Check if tool called  
- Set Outputs  
- Evaluation  
- Return chat response  

**Node Details:**

- **Evaluating?**  
  - *Type:* Evaluation Node (checkIfEvaluating)  
  - *Role:* Checks if the current execution is in evaluation mode, retrieving intermediate steps from the agent.  
  - *IO:* Input from Search Agent; output to "Check if tool called" and "Return chat response".  
  - *Failures:* Missing intermediate steps, evaluation context errors.  

- **Check if tool called**  
  - *Type:* Set Node  
  - *Role:* Computes a boolean indicating whether the tools expected in the dataset row were actually called by the agent.  
  - *Config:*  
    - Expression parses `tools_to_call` (CSV string), trims and converts to lowercase.  
    - Checks if every expected tool name is present in the agent’s intermediate steps actions.  
  - *IO:* Input from "Evaluating?"; output to "Set Outputs".  
  - *Failures:* Expression errors if `tools_to_call` or `intermediateSteps` missing or malformed.  

- **Set Outputs**  
  - *Type:* Evaluation Node (setMetrics and setOutputs)  
  - *Role:*  
    - Records the actual tools called as a CSV string in output.  
    - Updates Google Sheets document with evaluation outputs.  
    - Sets metric `tool_called` to 1 or 0 based on boolean from "Check if tool called".  
  - *Config:* Uses same Google Sheet and tab as input with credentials.  
  - *IO:* Input from "Check if tool called"; output to "Evaluation".  
  - *Failures:* Sheet write errors, auth failures, expression errors.  

- **Evaluation**  
  - *Type:* Evaluation Node (setMetrics)  
  - *Role:* Persists evaluation metrics (numeric) for the workflow run.  
  - *Config:* Sets metric `tool_called` as number (1 or 0).  
  - *IO:* Input from "Set Outputs".  
  - *Failures:* Metric recording errors.  

- **Return chat response**  
  - *Type:* NoOp Node  
  - *Role:* Placeholder node to return chat response in evaluation branch.  
  - *IO:* Input from "Evaluating?".  
  - *Failures:* None (pass-through node).

---

#### 2.4 Integration and Data Flow

**Overview:**  
Handles credentials, data exchange with Google Sheets, and connects all nodes to maintain smooth data flow and authentication.

**Nodes Involved:**  
- When fetching a dataset row (Google Sheets OAuth2)  
- Set Outputs (Google Sheets OAuth2)  
- Web search (HTTP Bearer Auth)  
- OpenRouter Chat Model (OpenRouter API key)  
- Embeddings OpenAI (OpenAI API key)  
- Search_db (Qdrant API key)

**Details:**

- Google Sheets OAuth2 credentials are used to fetch and update evaluation data in a specified sheet.  
- Firecrawl API is accessed via HTTP Bearer authentication for web search.  
- OpenRouter and OpenAI credentials provide access to language and embedding models.  
- Qdrant API credentials allow vector search operations.  

Failures in this block typically arise from expired tokens, revoked permissions, or misconfigured credentials.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                                | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                         |
|-------------------------|---------------------------------|-----------------------------------------------|----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger          | Receives incoming chat messages                |                            | Search Agent                |                                                                                                   |
| When fetching a dataset row | Evaluation Trigger (Google Sheets) | Triggers on dataset row with expected tools and questions |                            | Match chat format           |                                                                                                   |
| Match chat format         | Set                             | Normalizes question for agent input             | When fetching a dataset row | Search Agent                |                                                                                                   |
| Search Agent             | LangChain Agent                 | Coordinates tool calls to answer queries       | When chat message received, Match chat format | Evaluating?                 | System message enforces tool usage order and frequency.                                           |
| Search_db                | LangChain Vector Store (Qdrant) | Retrieves relevant info as a tool               | Embeddings OpenAI          | Search Agent                |                                                                                                   |
| Calculator               | LangChain Calculator Tool       | Performs math operations                         |                            | Search Agent                |                                                                                                   |
| Web search               | HTTP Request Tool               | Performs external web search                     |                            | Search Agent                | Authenticated via Firecrawl API, limited to one call per query.                                   |
| Summarizer               | LangChain Workflow Tool         | Summarizes gathered information                  |                            | Search Agent                | Invokes separate “Summarizer Agent” workflow.                                                     |
| OpenRouter Chat Model    | LangChain Chat Model            | Language model for generating responses          |                            | Search Agent                | Uses OpenRouter API with model openai/o3.                                                         |
| Embeddings OpenAI        | LangChain Embeddings OpenAI    | Generates embeddings for vector search           |                            | Search_db                   |                                                                                                   |
| Evaluating?              | Evaluation Node (checkIfEvaluating) | Checks if evaluation mode and retrieves steps   | Search Agent               | Check if tool called, Return chat response |                                                                                                   |
| Check if tool called     | Set                             | Compares expected vs actual tool calls           | Evaluating?                | Set Outputs                 |                                                                                                   |
| Set Outputs              | Evaluation Node (setOutputs)    | Writes actual tools called and metrics to sheet | Check if tool called       | Evaluation                  |                                                                                                   |
| Evaluation               | Evaluation Node (setMetrics)    | Records evaluation metrics                        | Set Outputs                |                             |                                                                                                   |
| Return chat response     | NoOp                            | Returns chat response placeholder                 | Evaluating?                |                             |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add **When chat message received** (LangChain Chat Trigger): configure webhook with default params.  
   - Add **When fetching a dataset row** (Evaluation Trigger): configure to connect to Google Sheets with OAuth2 credentials. Specify document ID and sheet name for evaluation data.

2. **Standardize Input**  
   - Add **Match chat format** (Set node): set variable `chatInput` to `{{$json.question}}`.

3. **Build Multi-Agent AI**  
   - Add **Search Agent** (LangChain Agent):  
     - Configure system message instructing tool usage order and constraints.  
     - Enable returning intermediate steps for evaluation.  
   - Add **OpenRouter Chat Model** (LangChain LM Chat OpenRouter): configure with OpenRouter API credentials, model `openai/o3`. Connect to Search Agent language model input.  
   - Add **Embeddings OpenAI** (LangChain Embeddings OpenAI): configure OpenAI credentials. Connect output to Search_db.  
   - Add **Search_db** (LangChain Vector Store Qdrant): set to “retrieve-as-tool” mode, specify Qdrant collection name, connect embeddings input and Search Agent tool input.  
   - Add **Calculator** (LangChain Calculator Tool): connect tool output to Search Agent.  
   - Add **Web search** (HTTP Request Tool): configure POST to Firecrawl API with HTTP Bearer Auth. Use dynamic query from agent. Connect tool output to Search Agent.  
   - Add **Summarizer** (LangChain Workflow Tool): link to external “Summarizer Agent” workflow by ID, map input query parameter, connect output to Search Agent.

4. **Connect Input to Agent**  
   - Connect output of **When chat message received** and **Match chat format** to **Search Agent**.

5. **Evaluation Logic Setup**  
   - Add **Evaluating?** (Evaluation Node, checkIfEvaluating): input from Search Agent main output.  
   - Add **Check if tool called** (Set node): create boolean `tool_called` by comparing expected tools (`tools_to_call` from dataset row) with actual tools used (agent’s `intermediateSteps`). Use expression to check all expected tools are included.  
   - Add **Set Outputs** (Evaluation Node, setOutputs): write actual tools called back to Google Sheets along with evaluation outputs. Use same sheet and credentials as dataset.  
   - Add **Evaluation** (Evaluation Node, setMetrics): record numeric metric for `tool_called` (1 if true, 0 if false).  
   - Add **Return chat response** (NoOp): connect from "Evaluating?" for chat response path.

6. **Connect Evaluation Nodes**  
   - Connect **Evaluating?** to **Check if tool called** and **Return chat response**.  
   - Connect **Check if tool called** to **Set Outputs**.  
   - Connect **Set Outputs** to **Evaluation**.

7. **Credential Setup**  
   - Configure Google Sheets OAuth2 credentials with access to evaluation spreadsheet.  
   - Setup OpenRouter API credentials for language model.  
   - Setup OpenAI API credentials for embeddings.  
   - Setup Qdrant API credentials for vector store queries.  
   - Setup Firecrawl HTTP Bearer token for web search.

8. **Test Workflow**  
   - Use sample queries and dataset rows to verify tool calls, intermediate step logging, and evaluation metric recording.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The system message in the Search Agent enforces strict tool usage order and limits (web search once only, summarizer mandatory). | Critical for correct multi-agent coordination and evaluation integrity.                              |
| The "Summarizer" node calls a separate workflow named "Summarizer Agent" by ID `fehTbkLtPtlVwDYq` | Sub-workflow for summarization, ensure it is deployed and accessible with matching input/output.     |
| Firecrawl Search API documentation: https://firecrawl.dev/docs/api/v1/search                    | Reference for configuring the Web search node’s API request and authentication.                      |
| Google Sheets OAuth2 setup requires proper permissions for reading and writing to the evaluation spreadsheet | OAuth2 token must have access to the specific Google Sheets document and sheet tab.                   |
| Qdrant vector store must contain the collection "search_queries" with indexed search data         | Properly provisioned Qdrant instance is essential for Search_db tool functionality.                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.