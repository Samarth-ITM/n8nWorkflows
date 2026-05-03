Evaluate RAG Response Accuracy with OpenAI: Document Groundedness Metric

https://n8nworkflows.xyz/workflows/evaluate-rag-response-accuracy-with-openai--document-groundedness-metric-4426


# Evaluate RAG Response Accuracy with OpenAI: Document Groundedness Metric

### 1. Workflow Overview

This workflow evaluates the accuracy of Retrieval-Augmented Generation (RAG) responses by measuring the **document groundedness** metric using OpenAI’s language models. It is designed to assess if AI-generated answers are strictly based on retrieved documents from a vector store, rather than hallucinating or adding ungrounded information.

**Target Use Cases:**  
- AI evaluation teams validating RAG system outputs for fidelity to source documents  
- Developers building AI agents that rely on vector stores for information retrieval  
- Automated quality control pipelines for generative AI responses  

**Logical Blocks:**  
- **1.1 Data Preparation and Vector Store Setup:** Download and ingest source documents into an in-memory vector store with embeddings.  
- **1.2 Chat Input Reception and Agent Query:** Triggered by chat input or manual execution, runs an AI Agent that queries the vector store.  
- **1.3 Evaluation Trigger and Input Remapping:** Periodically fetches test data rows from Google Sheets for evaluation and prepares inputs.  
- **1.4 Document Retrieval and AI Response Evaluation:** Extracts documents used by the agent, prompts an LLM to score groundedness, parses structured output.  
- **1.5 Output and Metrics Logging:** Records evaluation scores and reasons back to Google Sheets and sets metrics for further analysis.  
- **1.6 Supporting Nodes:** Includes no-op nodes, sticky notes with instructions and references, and error handling points implicitly.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Data Preparation and Vector Store Setup

**Overview:**  
This block fetches the Bitcoin whitepaper PDF, loads and splits it into text chunks, creates embeddings with OpenAI, and inserts them into an in-memory vector store used as a knowledge base.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get Datasheet  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI  
- Simple Vector Store  

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually for testing or setup  
  - Config: No parameters  
  - Input: None  
  - Output: Triggers "Get Datasheet"  
  - Edge Cases: Manual trigger only; no automatic error expected  

- **Get Datasheet**  
  - Type: HTTP Request  
  - Role: Downloads Bitcoin whitepaper PDF from bitcoin.org  
  - Config: URL set to https://bitcoin.org/bitcoin.pdf; no special options  
  - Input: Manual Trigger output  
  - Output: Binary data (PDF)  
  - Edge Cases: Network failures, HTTP timeouts, invalid URL  

- **Recursive Character Text Splitter**  
  - Type: Text Splitter (recursive character)  
  - Role: Splits loaded PDF content into manageable text chunks for embeddings  
  - Config: Defaults; no custom parameters  
  - Input: Output of Default Data Loader (documents)  
  - Output: Text chunks ready for embedding  
  - Edge Cases: Splitter may fail on malformed or empty documents  

- **Default Data Loader**  
  - Type: Document Data Loader  
  - Role: Converts binary PDF data into document format for vector store ingestion  
  - Config: Sets metadata “origin” to “evaluations”  
  - Input: Output of Recursive Character Text Splitter or HTTP Request? (actual connection shows from Recursive Character Text Splitter)  
  - Output: Document objects with metadata  
  - Edge Cases: Incompatible file format, parsing errors  

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings Node  
  - Role: Generates vector embeddings for text chunks using OpenAI embeddings API  
  - Config: Default embedding options; linked to OpenAI credential  
  - Input: Documents from Default Data Loader  
  - Output: Embeddings for insertion into vector store  
  - Edge Cases: API rate limits, authentication failures  

- **Simple Vector Store**  
  - Type: LangChain In-Memory Vector Store  
  - Role: Stores embeddings, enables retrieval and insertion  
  - Config: Mode “insert”, memory key “evaluations_document_groundness”, clearStore=true (resets on each run)  
  - Input: Embeddings from Embeddings OpenAI  
  - Output: Vector store ready for querying  
  - Edge Cases: Memory overflow if very large data; transient storage (not persistent across workflow restarts)  

---

#### Block 1.2: Chat Input Reception and Agent Query

**Overview:**  
Handles incoming chat messages, remaps input, and runs an AI Agent that uses the vector store to answer queries grounded in the Bitcoin whitepaper.

**Nodes Involved:**  
- When chat message received  
- Remap Input  
- AI Agent  
- Simple Vector Store1  
- Embeddings OpenAI1  
- OpenAI Chat Model  

**Node Details:**  

- **When chat message received**  
  - Type: LangChain Chat Trigger (Webhook)  
  - Role: Entry point for live chat queries  
  - Config: Webhook ID configured; no options  
  - Input: HTTP chat messages  
  - Output: Triggers “Remap Input”  
  - Edge Cases: Webhook connectivity issues, malformed requests  

- **Remap Input**  
  - Type: Set Node  
  - Role: Maps incoming JSON input to a standardized variable `chatInput`  
  - Config: Sets `chatInput` = incoming `input` field  
  - Input: Chat trigger output  
  - Output: Prepares input for AI Agent  
  - Edge Cases: Missing or malformed input JSON  

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates querying vector store and running OpenAI chat model to produce grounded answer  
  - Config: System message instructs agent to be strictly grounded in retrieved knowledgebase documents; intermediate steps returned for evaluation  
  - Input: `chatInput` from Remap Input; vector store tool  
  - Output: Agent’s response and retrieval steps  
  - Edge Cases: Model timeouts, hallucination risk, vector store retrieval errors  

- **Simple Vector Store1**  
  - Type: LangChain Vector Store (Tool mode)  
  - Role: Provides vector store as a tool named “bitcoin_whitepaper” for the agent to query  
  - Config: Mode “retrieve-as-tool”, memory key matches main vector store, tool description included  
  - Input: Embeddings from Embeddings OpenAI1  
  - Output: Vector store tool for agent  
  - Edge Cases: Tool invocation errors, empty retrieval results  

- **Embeddings OpenAI1**  
  - Type: Embeddings Node  
  - Role: Supports vector store retrieval by generating embeddings for agent queries  
  - Config: Default OpenAI embeddings settings  
  - Input: Text queries from agent  
  - Output: Embeddings for Simple Vector Store1  
  - Edge Cases: API quota limits, auth errors  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Generates the agent’s natural language responses  
  - Config: Model set to “gpt-4o-mini” (a smaller GPT-4 variant), default options  
  - Input: Agent prompt including retrieved documents  
  - Output: Text response with intermediate steps  
  - Edge Cases: API failures, rate limiting, unexpected model output  

---

#### Block 1.3: Evaluation Trigger and Input Remapping

**Overview:**  
The workflow periodically triggers evaluations by fetching rows from a Google Sheet, remaps inputs, and initiates the AI agent query/evaluation process.

**Nodes Involved:**  
- When fetching a dataset row  
- Remap Input  

**Node Details:**  

- **When fetching a dataset row**  
  - Type: Evaluation Trigger (Google Sheets integration)  
  - Role: Automatically triggers workflow execution per each row in a specific Google Sheet and sheet tab  
  - Config: Google Sheet ID and tab name set for “RAG Document Groundedness” test data  
  - Input: None (time or event based)  
  - Output: Row data passed downstream  
  - Edge Cases: Authentication errors, sheet access permissions, empty rows  

- **Remap Input**  
  - (Same node as in 1.2; reused here)  
  - Purpose: Standardizes input field for AI agent consumption  

---

#### Block 1.4: Document Retrieval and AI Response Evaluation

**Overview:**  
Extracts documents used by the agent, constructs a prompt combining retrieved documents and AI-generated response, and uses an OpenAI LLM chain to evaluate the groundedness of the AI answer.

**Nodes Involved:**  
- Evaluation  
- Get Documents  
- Document Grounding  
- Structured Output Parser  

**Node Details:**  

- **Evaluation**  
  - Type: Evaluation Node  
  - Role: Checks if currently evaluating or not, controls flow to next steps  
  - Config: Operation “checkIfEvaluating”  
  - Input: AI Agent output  
  - Output: Routes to “Get Documents” or “No Operation” depending on evaluation status  
  - Edge Cases: State mismatch, concurrency issues  

- **Get Documents**  
  - Type: Set Node  
  - Role: Extracts AI agent output and the specific documents retrieved by the tool “bitcoin_whitepaper” from intermediate steps  
  - Config:  
    - Sets `output` to AI agent final output  
    - Sets `documents` to array of documents retrieved in intermediate steps  
  - Input: Evaluation node output  
  - Output: JSON with `output` and `documents` fields  
  - Edge Cases: Missing intermediate steps, malformed data  

- **Document Grounding**  
  - Type: LangChain Chain LLM Node  
  - Role: Uses an OpenAI model to evaluate groundedness by comparing AI response and retrieved documents according to a defined rubric and instructions  
  - Config:  
    - Prompt includes concatenated document texts and AI response  
    - Instructions define groundedness metric and rating rubric (1 or 0)  
    - Output parser enabled for structured JSON output (`rating` and `reason`)  
  - Input: Output of “Get Documents” node  
  - Output: Structured evaluation results  
  - Edge Cases: Model hallucination, timeout, parsing errors  

- **Structured Output Parser**  
  - Type: Output Parser Node  
  - Role: Parses the JSON structured output from Document Grounding node into usable fields  
  - Config: Example JSON schema provided with `rating` and `reason`  
  - Input: Document Grounding output  
  - Output: Parsed JSON object  
  - Edge Cases: Parsing failures if OpenAI output deviates from schema  

---

#### Block 1.5: Output and Metrics Logging

**Overview:**  
Takes the evaluation results and writes them back to the Google Sheet, while also setting metrics for analytic purposes.

**Nodes Involved:**  
- Set Outputs  
- Set Metrics  

**Node Details:**  

- **Set Outputs**  
  - Type: Evaluation Node  
  - Role: Writes evaluation results back to the Google Sheet row  
  - Config:  
    - Maps evaluation output fields to sheet columns: `output` (agent response), `documents` (concatenated document texts), `score` (rating), `score_reason` (rationale)  
    - Google Sheet ID and tab specified  
  - Input: Parsed evaluation results  
  - Output: Triggers “Set Metrics”  
  - Edge Cases: Write permission errors, sheet access issues  

- **Set Metrics**  
  - Type: Evaluation Node  
  - Role: Sets numeric metrics (`score`) for downstream analytics or dashboarding  
  - Config: Associates metric “score” with the rating value from evaluation output  
  - Input: Set Outputs node  
  - Output: Ends workflow or triggers further processes if configured  
  - Edge Cases: Metric registration errors  

---

#### Block 1.6: Supporting Nodes (No Operation and Sticky Notes)

**Overview:**  
Includes nodes for workflow control and documentation.

**Nodes Involved:**  
- No Operation, do nothing  
- Sticky Note (multiple)  

**Node Details:**  

- **No Operation, do nothing**  
  - Type: NoOp Node  
  - Role: Placeholder to consume outputs where no further action is needed  
  - Config: None  
  - Input: From Evaluation node (when not evaluating)  
  - Output: None  
  - Edge Cases: None  

- **Sticky Notes**  
  - Type: Sticky Note Node  
  - Role: Provide instructions, references, and important links visually in the workflow editor  
  - Content includes:  
    - Vector Store setup instructions and link [Simple Vector Store](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreinmemory/)  
    - Evaluations Trigger documentation link [Evaluations Trigger](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger)  
    - Evaluations Node details [Evaluations Node](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluation)  
    - Overall workflow explanation and external references for groundedness metric [Vertex AI documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/models/metrics-templates#pointwise_groundedness)  
    - Community support links (Discord, Forum)  

---

### 3. Summary Table

| Node Name                    | Node Type                                  | Functional Role                              | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                               |
|------------------------------|--------------------------------------------|----------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                            | Workflow manual start                        | None                        | Get Datasheet               | ## Try It Out! ... Demonstrates RAG document groundedness evaluation metric ... [Vertex AI link]           |
| Get Datasheet                | HTTP Request                               | Download Bitcoin whitepaper PDF               | When clicking ‘Execute workflow’ | Simple Vector Store          | ## 1. Ready your RAG Vector Store ... [Simple Vector Store link]                                           |
| Recursive Character Text Splitter | Text Splitter (recursive)                  | Split PDF text into chunks for embedding     | Default Data Loader          | Default Data Loader          |                                                                                                           |
| Default Data Loader          | Document Data Loader                        | Convert binary PDF to documents with metadata | Recursive Character Text Splitter | Embeddings OpenAI            |                                                                                                           |
| Embeddings OpenAI            | OpenAI Embeddings Node                      | Generate vector embeddings                    | Default Data Loader          | Simple Vector Store          |                                                                                                           |
| Simple Vector Store          | In-memory Vector Store                      | Store embeddings for retrieval                | Embeddings OpenAI            | None                        | ## 1. Ready your RAG Vector Store ... [Simple Vector Store link]                                           |
| When chat message received   | LangChain Chat Trigger (Webhook)            | Receive chat input trigger                     | None                        | Remap Input                 | ## 2. Setup Your AI Workflow to Use Evaluations ... [Evaluations Trigger link]                             |
| Remap Input                 | Set Node                                   | Normalize input JSON field                     | When chat message received, When fetching a dataset row | AI Agent                   | ## 2. Setup Your AI Workflow to Use Evaluations ... [Evaluations Trigger link]                             |
| AI Agent                    | LangChain Agent                            | Query vector store and generate AI response  | Remap Input, Simple Vector Store1, OpenAI Chat Model | Evaluation                  | ## 3. Document Groundedness: Is the AI response based on retrieved documents? ... [Evaluations Node link] |
| Simple Vector Store1        | Vector Store Tool for Agent                 | Provide vector store as a tool for querying   | Embeddings OpenAI1          | AI Agent                    | ## 1. Ready your RAG Vector Store ... [Simple Vector Store link]                                           |
| Embeddings OpenAI1          | OpenAI Embeddings Node                      | Generate embeddings for agent queries         | None                        | Simple Vector Store1        |                                                                                                           |
| OpenAI Chat Model           | LangChain Chat Model                        | Generate natural language response            | AI Agent                    | AI Agent                    |                                                                                                           |
| Evaluation                  | Evaluation Node                            | Control evaluation process                     | AI Agent                    | Get Documents, No Operation | ## 3. Document Groundedness: Is the AI response based on retrieved documents? ... [Evaluations Node link] |
| Get Documents               | Set Node                                   | Extract AI response and retrieved documents   | Evaluation                  | Document Grounding          |                                                                                                           |
| Document Grounding          | LangChain Chain LLM                        | Evaluate groundedness of AI response          | Get Documents, Structured Output Parser | Set Outputs                |                                                                                                           |
| Structured Output Parser    | Output Parser Node                         | Parse structured evaluation output             | Document Grounding          | Document Grounding          |                                                                                                           |
| Set Outputs                 | Evaluation Node                            | Write evaluation results to Google Sheet      | Document Grounding          | Set Metrics                 |                                                                                                           |
| Set Metrics                 | Evaluation Node                            | Set numeric evaluation metrics                 | Set Outputs                 | None                       |                                                                                                           |
| When fetching a dataset row | Evaluation Trigger (Google Sheets)          | Trigger evaluation per row in Google Sheet    | None                        | Remap Input                 | ## 2. Setup Your AI Workflow to Use Evaluations ... [Evaluations Trigger link]                             |
| No Operation, do nothing    | NoOp Node                                  | Placeholder for no further action              | Evaluation                  | None                       |                                                                                                           |
| Sticky Note                 | Sticky Note                                | Provide workflow documentation and instructions | None                        | None                       | See content in Node Details                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: “When clicking ‘Execute workflow’”  
   - Purpose: Start workflow manually for setup/testing  

2. **Add HTTP Request Node:**  
   - Name: “Get Datasheet”  
   - URL: `https://bitcoin.org/bitcoin.pdf`  
   - Purpose: Download Bitcoin whitepaper PDF  
   - Connect: Manual Trigger → Get Datasheet  

3. **Add Recursive Character Text Splitter Node:**  
   - Name: “Recursive Character Text Splitter”  
   - Default settings  
   - Connect: Get Datasheet → Recursive Character Text Splitter  

4. **Add Default Data Loader Node:**  
   - Name: “Default Data Loader”  
   - Set metadata key `origin` = “evaluations”  
   - Data type: binary  
   - Connect: Recursive Character Text Splitter → Default Data Loader  

5. **Add OpenAI Embeddings Node:**  
   - Name: “Embeddings OpenAI”  
   - Credentials: Configure OpenAI API credentials  
   - Default options  
   - Connect: Default Data Loader → Embeddings OpenAI  

6. **Add Simple Vector Store Node:**  
   - Name: “Simple Vector Store”  
   - Mode: insert  
   - Memory Key: `evaluations_document_groundness`  
   - Clear Store: true (reset on run)  
   - Connect: Embeddings OpenAI → Simple Vector Store  

7. **Add LangChain Chat Trigger Node:**  
   - Name: “When chat message received”  
   - Webhook ID: configure or generate  
   - Connect: None (entry point)  

8. **Add Set Node:**  
   - Name: “Remap Input”  
   - Set field `chatInput` = `{{$json.input}}`  
   - Connect: Chat Trigger → Remap Input  

9. **Add OpenAI Embeddings Node (for agent retrieval):**  
   - Name: “Embeddings OpenAI1”  
   - Same OpenAI credentials as before  
   - Connect: None (used by vector store tool)  

10. **Add Simple Vector Store Node (Tool Mode):**  
    - Name: “Simple Vector Store1”  
    - Mode: retrieve-as-tool  
    - Memory Key: same as first vector store  
    - Tool Name: `bitcoin_whitepaper`  
    - Tool Description: “Call this tool to query over the bitcoin whitepaper to answer technical questions about bitcoin.”  
    - Connect: Embeddings OpenAI1 → Simple Vector Store1  

11. **Add OpenAI Chat Model Node:**  
    - Name: “OpenAI Chat Model”  
    - Model: `gpt-4o-mini`  
    - Credentials: OpenAI API  
    - Connect: Simple Vector Store1 and Remap Input → AI Agent  

12. **Add AI Agent Node:**  
    - Name: “AI Agent”  
    - System Message: instruct grounding in vector store docs only, return intermediate steps  
    - Connect: Remap Input, Simple Vector Store1, OpenAI Chat Model → AI Agent  

13. **Add Evaluation Trigger Node (Google Sheets):**  
    - Name: “When fetching a dataset row”  
    - Configure Google Sheets credentials  
    - Sheet ID and tab: as per your test sheet  
    - Connect: None (entry point)  

14. **Connect Google Sheets Trigger to Remap Input:**  
    - When fetching a dataset row → Remap Input (same node as above)  

15. **Add Evaluation Node:**  
    - Name: “Evaluation”  
    - Operation: checkIfEvaluating  
    - Connect: AI Agent → Evaluation  

16. **Add No Operation Node:**  
    - Name: “No Operation, do nothing”  
    - Connect: Evaluation (negative branch) → No Operation  

17. **Add Set Node to Extract Documents:**  
    - Name: “Get Documents”  
    - Assignments:  
      - `output` = AI Agent output  
      - `documents` = extract documents from intermediate steps where tool = `bitcoin_whitepaper`  
    - Connect: Evaluation (positive branch) → Get Documents  

18. **Add LangChain Chain LLM Node:**  
    - Name: “Document Grounding”  
    - Prompt: Combine documents and AI output; instruct to evaluate groundedness with rubric and detailed steps  
    - Enable structured output parser with JSON schema example: `{ "rating": 1, "reason": "..." }`  
    - Model: `gpt-4.1-mini` or suitable OpenAI Chat model  
    - Connect: Get Documents → Document Grounding  

19. **Add Structured Output Parser Node:**  
    - Name: “Structured Output Parser”  
    - JSON schema example as above  
    - Connect: Document Grounding → Structured Output Parser (integrated internally)  

20. **Add Evaluation Node to Set Outputs:**  
    - Name: “Set Outputs”  
    - Configure outputs to write back to Google Sheet:  
      - `output` (agent response)  
      - `documents` (concatenated)  
      - `score` (rating)  
      - `score_reason` (rationale)  
    - Sheet ID and tab same as evaluation trigger  
    - Credentials: Google Sheets OAuth2  
    - Connect: Document Grounding → Set Outputs  

21. **Add Evaluation Node to Set Metrics:**  
    - Name: “Set Metrics”  
    - Metric: number `score` = evaluation rating  
    - Connect: Set Outputs → Set Metrics  

22. **Add Sticky Notes:**  
    - Add sticky notes with instructions and links at appropriate positions for documentation  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                                                                                                                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Ready your RAG Vector Store**: Use the Simple Vector Store to store document embeddings for retrieval.                                                                                                                                                                                       | [Simple Vector Store Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreinmemory/)                                                                                            |
| **Setup Your AI Workflow to Use Evaluations**: The Evaluations Trigger allows safe, manual evaluation runs without affecting production.                                                                                                                                                      | [Evaluations Trigger Documentation](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger)                                              |
| **Document Groundedness Metric**: Measures how well AI responses are based on retrieved documents, not hallucinated info.                                                                                                                                                                      | [Evaluations Node Documentation](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluation)                                                        |
| **Groundedness Scoring Approach**: Adapted from Google Vertex AI metric template for pointwise groundedness.                                                                                                                                                                                  | [Vertex AI Groundedness Metric](https://cloud.google.com/vertex-ai/generative-ai/docs/models/metrics-templates#pointwise_groundedness)                                                                                                     |
| **Community Support**: Join n8n Discord or Forum for help and discussion.                                                                                                                                                                                                                        | [Discord](https://discord.com/invite/XPKeKXeB7d) / [Forum](https://community.n8n.io/)                                                                                                                                                         |
| **Sample Data Sheet**: Public Google Sheet with example evaluation data for testing.                                                                                                                                                                                                             | [Google Sheet Sample](https://docs.google.com/spreadsheets/d/1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y/edit?usp=sharing)                                                                                                              |

---

**Disclaimer:** The provided text derives exclusively from an n8n automated workflow. It complies strictly with current content policies and contains no illegal or protected content. All data processed is legal and publicly accessible.