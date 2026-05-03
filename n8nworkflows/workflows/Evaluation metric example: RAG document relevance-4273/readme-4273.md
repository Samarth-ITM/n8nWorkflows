Evaluation metric example: RAG document relevance

https://n8nworkflows.xyz/workflows/evaluation-metric-example--rag-document-relevance-4273


# Evaluation metric example: RAG document relevance

### 1. Workflow Overview

This n8n workflow demonstrates how to evaluate the relevance of documents retrieved from a vector store in response to a question, implementing a **retrieved document relevance** metric. It targets use cases where automated evaluation of document retrieval quality is required, such as validating AI-powered information retrieval systems or testing vector database responses.

The workflow is logically divided into two main blocks:

- **1.1 Setup: Populate Vector Database**  
  This block loads a dataset of documents from a Google Sheets spreadsheet, processes and embeds the documents, and inserts them into an in-memory vector store. This initialization is a prerequisite to performing relevance evaluation.

- **1.2 Main Evaluation Workflow**  
  This block listens for incoming questions (either via dataset rows or chat messages), queries the vector store for relevant documents, uses an AI agent with OpenAI GPT-4o-mini to determine the relevance of the retrieved documents with respect to the question, and sets an evaluation metric score accordingly. Metrics calculation is gated by an "evaluating" check to reduce unnecessary costs.

Supporting sticky notes provide guidance and context throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup: Populate Vector Database

**Overview:**  
This block is intended to be run once to initialize the vector store with the dataset documents. It reads raw document data from a Google Sheets spreadsheet, removes duplicates, splits text, generates embeddings via OpenAI, and stores them in an in-memory vector database.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get dataset  
- Remove Duplicates  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI  
- Simple Vector Store  
- Sticky Note2 (instructional)  
- Sticky Note4 (dataset link)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the setup process manually  
  - Inputs: None  
  - Outputs: Starts the chain to read dataset  
  - Edge cases: None (manual trigger)

- **Get dataset**  
  - Type: Google Sheets  
  - Role: Reads dataset rows from a Google Sheet containing questions and documents  
  - Configuration: Uses OAuth2 credentials for Google Sheets; accesses a specified spreadsheet and sheet by URL  
  - Input: Trigger from manual trigger  
  - Output: Raw rows of dataset  
  - Edge cases: Authentication failures, rate limits, empty dataset

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Role: Removes duplicate documents based on `document_id` field  
  - Input: Dataset rows from Google Sheets  
  - Output: Unique documents only  
  - Edge cases: Missing `document_id` fields, no duplicates found

- **Recursive Character Text Splitter**  
  - Type: Text Splitter (Recursive Character)  
  - Role: Splits document text into smaller chunks for embedding  
  - Input: Unique dataset rows  
  - Output: Text chunks  
  - Edge cases: Very large documents, empty text

- **Default Data Loader**  
  - Type: Document Default Data Loader  
  - Role: Loads text chunks into documents format compatible with LangChain nodes  
  - Input: Text chunks from splitter  
  - Output: Documents for embedding  
  - Edge cases: Invalid JSON data, empty input

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings  
  - Role: Generates vector embeddings for documents  
  - Configuration: Uses OpenAI API credentials; no special options  
  - Input: Documents from loader  
  - Output: Embeddings  
  - Edge cases: API limits, authentication errors, timeouts

- **Simple Vector Store**  
  - Type: Vector Store In-Memory  
  - Role: Inserts embeddings into vector store under key `rag_evaluation_demo`, clearing previous data  
  - Configuration: Mode is "insert", clearing store before insert  
  - Input: Embeddings  
  - Output: Confirmation of stored embeddings  
  - Edge cases: Memory limits, insertion errors

- **Sticky Note2 & Sticky Note4**  
  - Type: Sticky Note  
  - Role: Provide instructions and dataset link for setup  
  - No inputs/outputs

---

#### 2.2 Main Evaluation Workflow

**Overview:**  
This block handles question inputs, queries the vector store for relevant documents, uses an AI agent to assess the relevance of those documents, and sets an evaluation metric score. It supports two input methods: evaluation trigger from dataset rows and chat message webhook.

**Nodes Involved:**  
- When fetching a dataset row  
- Match chat format  
- When chat message received  
- AI Agent  
- Simple Vector Store1 (vector store used as a tool)  
- OpenAI Chat Model  
- Extract documents  
- Calculate doc relevance metric  
- Set metrics  
- Evaluating?  
- Return chat response  
- Sticky Note1, Sticky Note3, Sticky Note5, Sticky Note6

**Node Details:**

- **When fetching a dataset row**  
  - Type: Evaluation Trigger  
  - Role: Starts evaluation on each row from the Google Sheets dataset (same URL as setup)  
  - Credentials: Google Sheets OAuth2  
  - Output: Row data containing a question  
  - Edge cases: Sheet access errors, empty rows

- **Match chat format**  
  - Type: Set node  
  - Role: Converts the dataset row question field into a variable `chatInput` for the AI agent  
  - Input: Dataset row from evaluation trigger  
  - Output: JSON with `chatInput` string  
  - Edge cases: Missing `question` field

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Webhook entry point for chat messages containing questions  
  - Configuration: Webhook ID assigned, no options set  
  - Output: Incoming chat message data  
  - Edge cases: Webhook downtime, invalid payloads

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Central AI processing node that answers questions using vector store knowledge base as external tool  
  - Configuration: System message sets assistant role and restricts knowledge base usage, returns intermediate steps  
  - Inputs: From chat format or chat message trigger, plus vector store tool and OpenAI language model connections  
  - Output: AI response with intermediate steps including vector search results  
  - Edge cases: API errors, timeout, incomplete data, dependency failures

- **Simple Vector Store1**  
  - Type: Vector Store In-Memory (Retrieve as Tool)  
  - Role: Provides vector knowledge base tool for AI Agent to query  
  - Configuration: Uses key `rag_evaluation_demo` (populated in setup)  
  - Input: Embeddings for tool usage; output used by AI Agent  
  - Edge cases: Empty store, retrieval errors

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides the underlying language model used by AI Agent  
  - Configuration: Model set to `gpt-4o-mini`, with OpenAI credentials  
  - Edge cases: API limits, authentication errors

- **Extract documents**  
  - Type: Set node  
  - Role: Extracts the retrieved documents from AI Agent’s intermediate steps (specifically the vector knowledge base tool observation) into a `documents` field  
  - Input: AI Agent output  
  - Output: JSON with `documents` string containing retrieved facts  
  - Edge cases: Missing intermediate steps or tool observations

- **Calculate doc relevance metric**  
  - Type: OpenAI Node (LangChain OpenAI)  
  - Role: Uses a GPT-4o-mini model to evaluate the relevance of the retrieved documents (facts) to the original question and output a structured evaluation JSON with score 0 or 1  
  - Configuration:  
    - System prompt: Detailed instructions to behave as a teacher grading relevance, with criteria and scoring rules  
    - User prompt: Injects question and facts from dataset and extracted documents  
    - Output: JSON with `extended_reasoning`, `reasoning_summary`, and `score`  
  - Edge cases: Model errors, parsing errors, incomplete input

- **Set metrics**  
  - Type: Evaluation node  
  - Role: Sets the metric `similarity` to the score from the relevance evaluation output  
  - Input: Output of relevance evaluation (score)  
  - Edge cases: Missing or invalid score data

- **Evaluating?**  
  - Type: Evaluation node  
  - Role: Checks whether the workflow is running in evaluation mode to conditionally proceed with metric calculation  
  - Output: Branches to either metric calculation or to return response immediately  
  - Edge cases: Incorrect evaluation mode flag

- **Return chat response**  
  - Type: No Operation (NoOp) node  
  - Role: Returns the chat response without calculating metrics if not evaluating, reducing costs  
  - Edge cases: None

- **Sticky Notes (1, 3, 5, 6)**  
  - Provide instructions such as:  
    - "Check whether the documents returned are relevant to the question"  
    - Overview of how the workflow works and links to documentation  
    - Reminder to enable "Return intermediate steps" in AI Agent  
    - Label the main workflow section

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                              | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                           |
|----------------------------|------------------------------------|----------------------------------------------|---------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------|
| Sticky Note1               | Sticky Note                        | Instruction: Check document relevance        | None                            | None                          | Check whether the documents returned a relevant to the question                                     |
| Sticky Note3               | Sticky Note                        | Overview and explanation with documentation  | None                            | None                          | ## How it works This template shows how to calculate a workflow evaluation metric: **retrieved document relevance**... [link included] |
| Sticky Note4               | Sticky Note                        | Dataset info and link                         | None                            | None                          | Read in [this test dataset](https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=630157309#gid=630157309) of questions |
| When fetching a dataset row | Evaluation Trigger                 | Triggers evaluation on dataset rows          | None                            | Match chat format             |                                                                                                     |
| Match chat format          | Set                               | Formats question for AI Agent input           | When fetching a dataset row     | AI Agent                     |                                                                                                     |
| When chat message received | LangChain Chat Trigger            | Webhook entry point for chat questions        | None                            | AI Agent                     |                                                                                                     |
| AI Agent                  | LangChain Agent                    | Central AI node answering questions           | Match chat format, When chat message received, Simple Vector Store1, OpenAI Chat Model | Evaluating?                  | Make sure to enable 'Return intermediate steps' in the agent, to get the list of executed tools     |
| Simple Vector Store1       | Vector Store In-Memory (tool)      | Provides vector knowledge base tool to AI Agent | Embeddings OpenAI1              | AI Agent                     |                                                                                                     |
| OpenAI Chat Model          | LangChain OpenAI Chat Model        | Provides language model for AI Agent          | None                            | AI Agent                     |                                                                                                     |
| Extract documents          | Set                               | Extracts retrieved documents from AI Agent    | Evaluating?                    | Calculate doc relevance metric |                                                                                                     |
| Calculate doc relevance metric | LangChain OpenAI                 | Evaluates relevance of documents vs question  | Extract documents              | Set metrics                  |                                                                                                     |
| Set metrics                | Evaluation                        | Sets similarity metric to relevance score     | Calculate doc relevance metric  | None                         |                                                                                                     |
| Evaluating?                | Evaluation                        | Checks if workflow is in evaluation mode      | AI Agent                      | Extract documents, Return chat response | Only calculate metrics if we're evaluating, to reduce costs                                         |
| Return chat response       | NoOp                             | Returns response without metric calculation   | Evaluating?                    | None                         |                                                                                                     |
| Get dataset                | Google Sheets                    | Loads dataset from Google Sheets               | When clicking ‘Execute workflow’ | Remove Duplicates             |                                                                                                     |
| Remove Duplicates          | Remove Duplicates                | Removes duplicate documents by document_id    | Get dataset                   | Simple Vector Store           |                                                                                                     |
| Recursive Character Text Splitter | Text Splitter               | Splits document text into chunks               | None (used by Default Data Loader) | Default Data Loader           |                                                                                                     |
| Default Data Loader        | Document Default Data Loader     | Loads chunks as LangChain documents            | Recursive Character Text Splitter | Embeddings OpenAI            |                                                                                                     |
| Embeddings OpenAI          | OpenAI Embeddings               | Generates embeddings for documents             | Default Data Loader            | Simple Vector Store           |                                                                                                     |
| Simple Vector Store        | Vector Store In-Memory (insert) | Inserts document embeddings into vector store | Embeddings OpenAI             | None                         |                                                                                                     |
| Embeddings OpenAI1         | OpenAI Embeddings               | Embeddings for vector store tool               | None                          | Simple Vector Store1          |                                                                                                     |
| When clicking ‘Execute workflow’ | Manual Trigger               | Triggers setup to populate vector store        | None                          | Get dataset                  |                                                                                                     |
| Sticky Note2               | Sticky Note                    | Instruction for setup block                      | None                          | None                         | ### Setup: Populate vector DB Run this once before running the main workflow...                     |
| Sticky Note5               | Sticky Note                    | Instruction to enable "Return intermediate steps" | None                          | None                         | Make sure to enable 'Return intermediate steps' in the agent, to get the list of executed tools     |
| Sticky Note6               | Sticky Note                    | Labels the main workflow                         | None                          | None                         | ### Main workflow                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: Manual start for setup block

2. **Create Google Sheets node**  
   - Name: "Get dataset"  
   - Credentials: Google Sheets OAuth2 (configured with appropriate account)  
   - Parameters: Set document ID and sheet name to URL of dataset spreadsheet  
   - Connect from "When clicking ‘Execute workflow’"

3. **Create Remove Duplicates node**  
   - Name: "Remove Duplicates"  
   - Parameters: Compare by field `document_id`  
   - Connect from "Get dataset"

4. **Create Recursive Character Text Splitter node**  
   - Name: "Recursive Character Text Splitter"  
   - Default settings  
   - Connect from "Remove Duplicates"

5. **Create Default Data Loader node**  
   - Name: "Default Data Loader"  
   - Parameters: JSON data = `{{$json.document_text}}` (expression mode)  
   - Connect from "Recursive Character Text Splitter"

6. **Create OpenAI Embeddings node**  
   - Name: "Embeddings OpenAI"  
   - Credentials: OpenAI API key  
   - Default options  
   - Connect from "Default Data Loader"

7. **Create Vector Store In-Memory node**  
   - Name: "Simple Vector Store"  
   - Parameters: Mode = "insert", Memory Key = "rag_evaluation_demo", Clear Store = true  
   - Connect from "Embeddings OpenAI"

8. **Create Evaluation Trigger node**  
   - Name: "When fetching a dataset row"  
   - Credentials: Google Sheets OAuth2 (same as above)  
   - Parameters: Same document and sheet URL as "Get dataset"

9. **Create Set node**  
   - Name: "Match chat format"  
   - Assign `chatInput` = `={{ $json.question }}`  
   - Connect from "When fetching a dataset row"

10. **Create LangChain Chat Trigger node**  
    - Name: "When chat message received"  
    - Configure webhook (webhook ID generated automatically)  
    - No additional options

11. **Create OpenAI Chat Model node**  
    - Name: "OpenAI Chat Model"  
    - Credentials: OpenAI API key  
    - Model: "gpt-4o-mini"

12. **Create OpenAI Embeddings node (for tool usage)**  
    - Name: "Embeddings OpenAI1"  
    - Credentials: OpenAI API key

13. **Create Vector Store In-Memory node (tool mode)**  
    - Name: "Simple Vector Store1"  
    - Parameters: Mode = "retrieve-as-tool", Memory Key = "rag_evaluation_demo", Tool Name = "vector_knowledge_base"  
    - Connect from "Embeddings OpenAI1"

14. **Create LangChain Agent node**  
    - Name: "AI Agent"  
    - Parameters:  
      - System message: "You are a helpful assistant. Answer the user's questions using information from your vector knowledge base only."  
      - Enable "Return intermediate steps"  
    - Inputs: Connect from "Match chat format" and "When chat message received" (both connect to AI Agent main input)  
    - Connect AI Tool input from "Simple Vector Store1"  
    - Connect AI Language Model input from "OpenAI Chat Model"

15. **Create Evaluation node**  
    - Name: "Evaluating?"  
    - Operation: "checkIfEvaluating"  
    - Connect from "AI Agent"

16. **Create Set node**  
    - Name: "Extract documents"  
    - Assignment: Set `documents` = `={{ $json.intermediateSteps.filter(x => x.action.tool == 'vector_knowledge_base')[0].observation }}`  
    - Connect from first output of "Evaluating?"

17. **Create OpenAI node**  
    - Name: "Calculate doc relevance metric"  
    - Model: "gpt-4o-mini"  
    - Credentials: OpenAI API key  
    - Parameters:  
      - System prompt: Detailed teacher-style evaluation instructions as in the workflow  
      - User prompt: Inject question and facts using expressions:  
        - QUESTION: `={{ $('When fetching a dataset row').item.json.question }}`  
        - FACTS: `={{ $json.documents }}`  
      - JSON output enabled  
    - Connect from "Extract documents"

18. **Create Evaluation node**  
    - Name: "Set metrics"  
    - Operation: "setMetrics"  
    - Metrics: Set metric "similarity" to `={{ $json.message.content.score }}`  
    - Connect from "Calculate doc relevance metric"

19. **Create No Operation node**  
    - Name: "Return chat response"  
    - Connect from second output of "Evaluating?"

20. **Link outputs:**  
    - From "Evaluating?" main output 1 → "Extract documents"  
    - From "Evaluating?" main output 2 → "Return chat response"  
    - From "Extract documents" → "Calculate doc relevance metric"  
    - From "Calculate doc relevance metric" → "Set metrics"

21. **Add Sticky Notes nodes** for guidance and documentation:  
    - Sticky Note1 near relevance metric calculation: "Check whether the documents returned are relevant to the question"  
    - Sticky Note2 near setup block: "### Setup: Populate vector DB Run this once before running the main workflow..."  
    - Sticky Note3 near evaluation trigger: Explanation and documentation links  
    - Sticky Note4 near dataset trigger: Link to dataset  
    - Sticky Note5 near AI Agent: "Make sure to enable 'Return intermediate steps' in the agent, to get the list of executed tools"  
    - Sticky Note6 labeling "Main workflow" section

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                               | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow evaluates the quality of retrieved documents by scoring their relevance to the input question, enabling AI system testing and benchmarking.                                                  | Core workflow purpose                                                                                  |
| Workflow evaluation documentation and other metric examples can be found here: https://docs.n8n.io/advanced-ai/evaluations/overview and https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/   | Official n8n docs                                                                                      |
| Dataset used for evaluation can be accessed at: https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=630157309#gid=630157309                                          | Public Google Sheets dataset                                                                           |
| Ensure 'Return intermediate steps' is enabled in the AI Agent node to allow extraction of vector store results for metric calculation.                                                                     | Critical configuration note                                                                            |
| The OpenAI GPT-4o-mini model is used consistently for both the chat completions and relevance evaluation to maintain coherence and cost-effectiveness.                                                      | Model choice rationale                                                                                 |

---

**Disclaimer:** The text provided originates exclusively from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.