Create Adaptive RAG Chat Agent with Google Gemini and Qdrant

https://n8nworkflows.xyz/workflows/create-adaptive-rag-chat-agent-with-google-gemini-and-qdrant-5111


# Create Adaptive RAG Chat Agent with Google Gemini and Qdrant

---

### 1. Workflow Overview

This workflow, titled **"Adaptive & Conditional AI Chat Agent - www.quantralabs.com"**, implements a sophisticated Adaptive Retrieval-Augmented Generation (RAG) chat system. It is designed to classify user queries into four distinct categories—Factual, Analytical, Opinion, or Contextual—and dynamically adapt the retrieval and generation strategy accordingly. The workflow integrates Google Gemini large language models (LLMs) and Qdrant vector store to deliver contextually tailored and precise answers.

**Target Use Cases:**

- Intelligent chatbots that provide nuanced answers depending on question type.
- Knowledge base querying with adaptive retrieval strategies.
- Enhanced customer support or information retrieval systems requiring multi-faceted query understanding.
- Situations where user context and query complexity vary widely.

**Logical Blocks:**

- **1.1 Input Reception & Standardization**: Accept user input via chat or external workflow, normalize key inputs.
- **1.2 Query Classification**: Use an LLM agent to categorize the query.
- **1.3 Adaptive Strategy Routing**: Route processing based on classification.
- **1.4 Strategy-Specific Query Adaptation**: Modify the query or generate sub-queries using specialized LLM prompts per category.
- **1.5 Prompt and Output Setup**: Prepare tailored system prompts and outputs for answer generation.
- **1.6 Document Retrieval**: Search Qdrant vector store with adapted queries using Google Gemini embeddings.
- **1.7 Context Concatenation**: Aggregate retrieved document content for context.
- **1.8 Answer Generation**: Generate final response using Google Gemini with context and memory.
- **1.9 Response Delivery**: Return the generated answer to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Standardization

- **Overview:** Receives user inputs either from a chat trigger or another workflow and standardizes essential variables for downstream nodes.
- **Nodes Involved:** Chat, When Executed by Another Workflow, Combined Fields
- **Node Details:**

  - **Chat**
    - Type: Chat Trigger Node
    - Role: Entry point for chat-based user queries.
    - Config: Listens for incoming chat messages with no additional parameters.
    - Input: External user chat input.
    - Output: Provides raw chat input as JSON.
    - Edge Cases: Missing or malformed chat input.
  
  - **When Executed by Another Workflow**
    - Type: Execute Workflow Trigger Node
    - Role: Allows triggering this workflow from another workflow with inputs.
    - Config: Expects inputs `user_query`, `chat_memory_key`, `vector_store_id`.
    - Edge Cases: Missing inputs or incorrect input types.
  
  - **Combined Fields**
    - Type: Set Node
    - Role: Normalizes input fields for consistent use.
    - Config:
      - Sets `user_query` from `user_query` or fallback `chatInput`.
      - Sets `chat_memory_key` from input or `Chat` sessionId.
      - Sets `vector_store_id` from input or a placeholder `<ID HERE>`.
    - Expressions used to fallback values.
    - Input: From Chat or Execute Workflow Trigger.
    - Output: Standardized JSON object for query, memory key, and vector store ID.

#### 1.2 Query Classification

- **Overview:** Classifies the user query into one of four categories using a Google Gemini LLM agent.
- **Nodes Involved:** Query Classification, Switch
- **Node Details:**

  - **Query Classification**
    - Type: LangChain Agent Node (Google Gemini)
    - Role: Classifies query into Factual, Analytical, Opinion, or Contextual.
    - Config:
      - Input text is the `user_query`.
      - System message instructs strict classification with no explanation.
    - Output: Single category string.
    - Edge Cases: Ambiguous queries, LLM timeout, or unexpected outputs.
  
  - **Switch**
    - Type: Switch Node
    - Role: Routes workflow based on classification output.
    - Config: Routes by exact string matches for the four categories.
    - Fallback: Default route is disabled (0).
    - Edge Cases: Classification output outside expected values.

#### 1.3 Adaptive Strategy Routing

- **Overview:** Routes the workflow to one of four strategy blocks for specific query types.
- **Nodes Involved:** Switch, Factual Strategy - Focus on Precision, Analytical Strategy - Comprehensive Coverage, Opinion Strategy - Diverse Perspectives, Contextual Strategy - User Context Integration
- **Node Details:**

  - **Factual Strategy - Focus on Precision**
    - Type: LangChain Agent Node (Google Gemini)
    - Role: Reformulates factual queries to be precise and entity-focused.
    - Config: System prompt emphasizes rewriting for precision.
    - Input: user_query.
    - Output: Enhanced query string.
    - Memory: Uses dedicated chat buffer memory (Chat Buffer Memory Factual).
    - Edge Cases: Over-simplification or loss of original intent.

  - **Analytical Strategy - Comprehensive Coverage**
    - Type: LangChain Agent Node (Google Gemini)
    - Role: Generates exactly 3 sub-questions to cover analytical query breadth.
    - Config: System prompt instructs generating a list of sub-questions.
    - Output: Multi-line string of sub-questions.
    - Memory: Chat Buffer Memory Analytical.
    - Edge Cases: Incomplete or irrelevant sub-questions.

  - **Opinion Strategy - Diverse Perspectives**
    - Type: LangChain Agent Node (Google Gemini)
    - Role: Identifies 3 distinct viewpoints on opinion-based queries.
    - Config: System prompt directs unbiased viewpoint generation.
    - Memory: Chat Buffer Memory Opinion.
    - Edge Cases: Biased or insufficient viewpoints.

  - **Contextual Strategy - User Context Integration**
    - Type: LangChain Agent Node (Google Gemini)
    - Role: Infers implied context relevant to the query.
    - Config: System prompt requests brief inferred context.
    - Memory: Chat Buffer Memory Contextual.
    - Edge Cases: Misinterpretation of context or insufficient context.

- **Memory Buffer Nodes:**
  - Chat Buffer Memory nodes linked to each strategy preserve recent conversational context, using `chat_memory_key` as session identifier and maintaining a 10-message context window.

#### 1.4 Prompt and Output Setup

- **Overview:** Prepares tailored output and system prompt for the final answer generation, based on strategy output.
- **Nodes Involved:** Factual Prompt and Output, Analytical Prompt and Output, Opinion Prompt and Output, Contextual Prompt and Output, Set Prompt and Output
- **Node Details:**

  - **Factual Prompt and Output**
    - Type: Set Node
    - Role: Sets system prompt and output for factual answers.
    - Config: Output is the enhanced query; prompt instructs precise factual answering.

  - **Analytical Prompt and Output**
    - Type: Set Node
    - Role: Sets system prompt and output for analytical answers.
    - Config: Output is sub-questions list; prompt instructs comprehensive analysis.

  - **Opinion Prompt and Output**
    - Type: Set Node
    - Role: Sets system prompt and output for opinion answers.
    - Config: Output is viewpoint list; prompt instructs unbiased discussion.

  - **Contextual Prompt and Output**
    - Type: Set Node
    - Role: Sets system prompt and output for contextual answers.
    - Config: Output is inferred context; prompt instructs contextual relevance.

  - **Set Prompt and Output**
    - Type: Set Node
    - Role: Aggregates output and prompt from the previous nodes to pass downstream.
    - Inputs: All four category prompt/output nodes connect here.
    - Output: JSON with `output` (adapted query) and `prompt` (system instruction).

#### 1.5 Document Retrieval

- **Overview:** Uses the adapted query and system prompt to retrieve relevant documents from Qdrant vector store with Google Gemini embeddings.
- **Nodes Involved:** Embeddings, Retrieve Documents from Vector Store
- **Node Details:**

  - **Embeddings**
    - Type: LangChain Embeddings Node (Google Gemini)
    - Role: Generates vector embeddings for the query for similarity search.
    - Config: Uses `models/text-embedding-004`.
    - Output: Vector embedding data.
    - Connected to the Retrieval node via `ai_embedding` input.

  - **Retrieve Documents from Vector Store**
    - Type: LangChain Vector Store Node (Qdrant)
    - Role: Retrieves top 10 most relevant document chunks from specified Qdrant collection.
    - Config:
      - `mode` set to load documents.
      - `topK` = 10.
      - `prompt` built by concatenating system prompt and output (adapted query).
      - Uses `vector_store_id` for Qdrant collection selection.
    - Edge Cases: Empty retrieval, wrong collection ID, Qdrant connectivity issues.

#### 1.6 Context Concatenation

- **Overview:** Aggregates retrieved document content into a single context block for final answer generation.
- **Nodes Involved:** Concatenate Context
- **Node Details:**

  - **Concatenate Context**
    - Type: Summarize Node
    - Role: Concatenates retrieved `document.pageContent` fields separated by `\n\n---\n\n`.
    - Output: Single concatenated string of document content.
    - Edge Cases: No documents retrieved, very large content causing timeout.

#### 1.7 Answer Generation

- **Overview:** Generates the final answer integrating the user query, retrieval context, strategy prompt, and chat memory.
- **Nodes Involved:** Answer, Gemini Answer, Chat Buffer Memory, Respond to Webhook
- **Node Details:**

  - **Answer**
    - Type: LangChain Agent Node (Google Gemini)
    - Role: Generates response using:
      - System prompt set in Set Prompt and Output.
      - Concatenated context wrapped in `<ctx>` tags.
      - Original user query.
      - Chat history from Chat Buffer Memory.
    - Config: Prompt includes instructions, context, and history.
    - Edge Cases: LLM failure, context insufficient for answer, chat memory desync.

  - **Gemini Answer**
    - Type: LangChain LM Chat Google Gemini Node
    - Role: Executes underlying LLM call for the Answer node.
    - Model: `models/gemini-2.0-flash`.
  
  - **Chat Buffer Memory**
    - Type: LangChain Memory Buffer Node
    - Role: Maintains conversation state for answer generation.
    - Config: Context window length 10, identified by `chat_memory_key`.
  
  - **Respond to Webhook**
    - Type: Respond to Webhook Node
    - Role: Sends the generated answer back to the user.
    - No additional config.
    - Edge Cases: Webhook client disconnect, response formatting errors.

---

### 3. Summary Table

| Node Name                          | Node Type                                       | Functional Role                            | Input Node(s)                   | Output Node(s)                             | Sticky Note                                                                                              |
|-----------------------------------|------------------------------------------------|--------------------------------------------|---------------------------------|--------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Chat                              | Chat Trigger Node                               | Entry point for chat user input             | -                               | Combined Fields                            |                                                                                                        |
| When Executed by Another Workflow | Execute Workflow Trigger                        | Entry point for external workflow trigger   | -                               | Combined Fields                            |                                                                                                        |
| Combined Fields                   | Set Node                                       | Normalize and unify input fields             | Chat, When Executed by Another Workflow | Query Classification                       |                                                                                                        |
| Query Classification              | LangChain Agent (Google Gemini)                 | Classify user query into 4 categories        | Combined Fields                 | Switch                                     | ## User query classification **Classify the query into one of four categories: Factual, Analytical, Opinion, or Contextual.** |
| Switch                           | Switch Node                                    | Route processing based on query category     | Query Classification            | Factual Strategy, Analytical Strategy, Opinion Strategy, Contextual Strategy |                                                                                                        |
| Factual Strategy - Focus on Precision | LangChain Agent (Google Gemini)             | Refine factual queries for precision         | Switch (Factual)                | Factual Prompt and Output                   | ## Factual Strategy **Retrieve precise facts and figures.**                                            |
| Chat Buffer Memory Factual        | Memory Buffer Node (LangChain)                  | Maintain chat memory for factual strategy    | Factual Strategy                | Factual Strategy                           |                                                                                                        |
| Factual Prompt and Output         | Set Node                                       | Set prompt and output for factual answers    | Factual Strategy                | Set Prompt and Output                       |                                                                                                        |
| Analytical Strategy - Comprehensive Coverage | LangChain Agent (Google Gemini)          | Generate sub-questions for analytical queries | Switch (Analytical)             | Analytical Prompt and Output                | ## Analytical Strategy **Provide comprehensive coverage of a topics and exploring different aspects.**  |
| Chat Buffer Memory Analytical     | Memory Buffer Node (LangChain)                  | Maintain chat memory for analytical strategy | Analytical Strategy             | Analytical Strategy                         |                                                                                                        |
| Analytical Prompt and Output      | Set Node                                       | Set prompt and output for analytical answers | Analytical Strategy             | Set Prompt and Output                       |                                                                                                        |
| Opinion Strategy - Diverse Perspectives | LangChain Agent (Google Gemini)           | Identify multiple viewpoints for opinion queries | Switch (Opinion)               | Opinion Prompt and Output                   | ## Opinion Strategy **Gather diverse viewpoints on a subjective issue.**                               |
| Chat Buffer Memory Opinion        | Memory Buffer Node (LangChain)                  | Maintain chat memory for opinion strategy    | Opinion Strategy               | Opinion Strategy                           |                                                                                                        |
| Opinion Prompt and Output         | Set Node                                       | Set prompt and output for opinion answers    | Opinion Strategy               | Set Prompt and Output                       |                                                                                                        |
| Contextual Strategy - User Context Integration | LangChain Agent (Google Gemini)          | Infer implied context for contextual queries | Switch (Contextual)            | Contextual Prompt and Output                | ## Contextual Strategy **Incorporate user-specific context to fine-tune the retrieval.**                |
| Chat Buffer Memory Contextual     | Memory Buffer Node (LangChain)                  | Maintain chat memory for contextual strategy | Contextual Strategy            | Contextual Strategy                         |                                                                                                        |
| Contextual Prompt and Output      | Set Node                                       | Set prompt and output for contextual answers | Contextual Strategy            | Set Prompt and Output                       |                                                                                                        |
| Set Prompt and Output             | Set Node                                       | Aggregate prompt and output for retrieval    | Factual Prompt, Analytical Prompt, Opinion Prompt, Contextual Prompt | Retrieve Documents from Vector Store            | ## Perform adaptive retrieval **Find document considering both query and context.**                    |
| Embeddings                       | LangChain Embeddings Node (Google Gemini)       | Generate vector embeddings for retrieval     | Set Prompt and Output (output) | Retrieve Documents from Vector Store (ai_embedding) |                                                                                                        |
| Retrieve Documents from Vector Store | LangChain Vector Store Node (Qdrant)        | Retrieve relevant documents from vector store | Set Prompt and Output, Embeddings | Concatenate Context                       |                                                                                                        |
| Concatenate Context              | Summarize Node                                 | Concatenate retrieved document contents      | Retrieve Documents from Vector Store | Answer                                     |                                                                                                        |
| Answer                          | LangChain Agent (Google Gemini)                 | Generate final answer integrating context    | Concatenate Context, Chat Buffer Memory, Set Prompt and Output | Respond to Webhook                          | ## Reply to the user integrating retrieval context                                                    |
| Gemini Answer                   | LangChain LM Chat Google Gemini Node            | Execute LLM call for final answer             | Answer                         | Answer                                     |                                                                                                        |
| Chat Buffer Memory              | Memory Buffer Node (LangChain)                   | Maintain chat memory for final answer         | Answer                         | Answer                                     |                                                                                                        |
| Respond to Webhook              | Respond to Webhook Node                          | Send response back to user                     | Answer                         | -                                          |                                                                                                        |
| Sticky Note                    | Sticky Note                                     | Factual Strategy description                   | -                             | -                                          | ## Factual Strategy **Retrieve precise facts and figures.**                                            |
| Sticky Note1                   | Sticky Note                                     | Analytical Strategy description                 | -                             | -                                          | ## Analytical Strategy **Provide comprehensive coverage of a topics and exploring different aspects.**  |
| Sticky Note2                   | Sticky Note                                     | Opinion Strategy description                    | -                             | -                                          | ## Opinion Strategy **Gather diverse viewpoints on a subjective issue.**                               |
| Sticky Note3                   | Sticky Note                                     | Contextual Strategy description                 | -                             | -                                          | ## Contextual Strategy **Incorporate user-specific context to fine-tune the retrieval.**                |
| Sticky Note4                   | Sticky Note                                     | Adaptive retrieval description                   | -                             | -                                          | ## Perform adaptive retrieval **Find document considering both query and context.**                    |
| Sticky Note5                   | Sticky Note                                     | Reply integration description                     | -                             | -                                          | ## Reply to the user integrating retrieval context                                                    |
| Sticky Note6                   | Sticky Note                                     | User query classification description            | -                             | -                                          | ## User query classification **Classify the query into one of four categories: Factual, Analytical, Opinion, or Contextual.** |
| Sticky Note7                   | Sticky Note                                     | Full workflow summary                             | -                             | -                                          | # Adaptive RAG Workflow (Full explanation of workflow and steps)                                       |
| Sticky Note8                   | Sticky Note                                     | Reminder to update vector_store_id in Chat mode | -                             | -                                          | ## ⚠️  If using in Chat mode **Update the `vector_store_id` variable to the corresponding Qdrant ID needed to perform the documents retrieval.** |
| Sticky Note9                   | Sticky Note                                     | Quantra Labs branding and contact info           | -                             | -                                          | ## Quantra Labs \nFollow Us https://www.x.com/quantralabs\nConnect with Us https://www.linkedin.com/company/quantra-labs\nwww.quantralabs.com |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node** (`Chat`)
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Purpose: Receive chat inputs.
   - No specific parameters.
   - Connect output to `Combined Fields`.

2. **Create Execute Workflow Trigger Node** (`When Executed by Another Workflow`)
   - Type: `n8n-nodes-base.executeWorkflowTrigger`
   - Configure inputs: `user_query`, `chat_memory_key`, `vector_store_id`
   - Connect output to `Combined Fields`.

3. **Create Set Node** (`Combined Fields`)
   - Purpose: Normalize inputs.
   - Assignments:
     - `user_query`: `{{$json.user_query || $json.chatInput}}`
     - `chat_memory_key`: `{{$json.chat_memory_key || $('Chat').item.json.sessionId}}`
     - `vector_store_id`: `{{$json.vector_store_id || "<ID HERE>"}}`
   - Connect output to `Query Classification`.

4. **Create LangChain Agent Node** (`Query Classification`)
   - Model: Google Gemini via LangChain Agent node.
   - Input text: `Classify this query: {{ $('Combined Fields').item.json.user_query }}`
   - System prompt: Expert classifier with instructions to return exactly one category: Factual, Analytical, Opinion, Contextual (no explanations).
   - Connect output to `Switch`.

5. **Create Switch Node** (`Switch`)
   - Rules: Route by string equals trimmed output to keys: Factual, Analytical, Opinion, Contextual.
   - Connect each output to respective strategy node.

6. **Create Factual Strategy Node** (`Factual Strategy - Focus on Precision`)
   - LangChain Agent Node with Google Gemini.
   - Input text: `Enhance this factual query: {{ $('Combined Fields').item.json.user_query }}`
   - System prompt: Rewrite query focusing on precision and key entities.
   - Connect to `Factual Prompt and Output`.
   - Connect to `Chat Buffer Memory Factual` for AI memory.

7. **Create Chat Buffer Memory Node** (`Chat Buffer Memory Factual`)
   - LangChain Memory Buffer Window Node.
   - Session key: `{{$('Combined Fields').item.json.chat_memory_key}}`
   - Context window: 10 messages.
   - Connect to `Factual Strategy`.

8. **Create Set Node** (`Factual Prompt and Output`)
   - Assignments:
     - `output`: `{{$json.output}}` (from strategy)
     - `prompt`: System prompt instructing factual answer generation focusing on accuracy and limitations.
   - Connect output to `Set Prompt and Output`.

9. **Repeat steps 6-8 for the other three strategies:**

   - **Analytical Strategy - Comprehensive Coverage**
     - Input text: Generate exactly 3 sub-questions from user query.
     - System prompt: Break down complex questions comprehensively.
     - Memory: Chat Buffer Memory Analytical.
     - Set Prompt with analytical answer instructions.

   - **Opinion Strategy - Diverse Perspectives**
     - Input text: Identify 3 different viewpoints.
     - Memory: Chat Buffer Memory Opinion.
     - Set Prompt with unbiased opinion instructions.

   - **Contextual Strategy - User Context Integration**
     - Input text: Infer implied context from query.
     - Memory: Chat Buffer Memory Contextual.
     - Set Prompt with context-aware answer instructions.

10. **Create Set Node** (`Set Prompt and Output`)
    - Purpose: Aggregate `output` and `prompt` from each strategy's Set node.
    - Connect all four strategy Set nodes as inputs.
    - Connect output to `Embeddings`.

11. **Create LangChain Embeddings Node** (`Embeddings`)
    - Model: Google Gemini embeddings `models/text-embedding-004`.
    - Input: `Set Prompt and Output` output (adapted query).
    - Connect to `Retrieve Documents from Vector Store` ai_embedding input.

12. **Create LangChain Vector Store Node** (`Retrieve Documents from Vector Store`)
    - Vector store: Qdrant.
    - Collection ID: `{{$('Combined Fields').item.json.vector_store_id}}`
    - Mode: Load documents.
    - Top K: 10.
    - Prompt: Combine system prompt and adapted query.
    - Connect output documents to `Concatenate Context`.

13. **Create Summarize Node** (`Concatenate Context`)
    - Field to summarize: `document.pageContent`
    - Aggregation: Concatenate with separator `\n\n---\n\n`.
    - Connect output to `Answer`.

14. **Create LangChain Agent Node** (`Answer`)
    - Model: Google Gemini `models/gemini-2.0-flash`.
    - Prompt: Includes system prompt from `Set Prompt and Output`, concatenated context wrapped in `<ctx>` tags, and original user query.
    - Memory: Connect to `Chat Buffer Memory`.
    - Connect output to `Respond to Webhook`.

15. **Create Chat Buffer Memory Node** (`Chat Buffer Memory`)
    - Session key: `{{$('Combined Fields').item.json.chat_memory_key}}`
    - Context window: 10 messages.
    - Connect memory to `Answer`.

16. **Create Respond to Webhook Node** (`Respond to Webhook`)
    - Purpose: Return final answer to user.
    - Connect input from `Answer`.

17. **Add Sticky Notes** (Optional but recommended for documentation)
    - Add descriptive sticky notes near respective logical blocks for clarity.

18. **Credential Setup**
    - Configure Google Gemini credentials for all LangChain Google Gemini nodes (classification, strategies, embeddings, answers).
    - Configure Qdrant credentials and ensure `vector_store_id` corresponds to an existing Qdrant collection.

19. **Final Testing**
    - Trigger via Chat or external workflow.
    - Validate the classification, strategy adaptation, retrieval, and final answer correctness.
    - Monitor for errors such as timeouts, wrong query routing, or empty retrievals.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow is an implementation of Adaptive Retrieval-Augmented Generation combining query classification with conditional retrieval and generation strategies. | Workflow summary sticky note included inside the workflow.                                                  |
| Update the `vector_store_id` variable to the correct Qdrant collection ID to enable document retrieval in Chat mode.                                            | Sticky Note8 near input block.                                                                              |
| Quantra Labs branding and social channels: Twitter https://www.x.com/quantralabs, LinkedIn https://www.linkedin.com/company/quantra-labs, www.quantralabs.com    | Sticky Note9 at workflow top-left corner.                                                                   |
| This workflow leverages Google Gemini models, requiring appropriate API keys and access.                                                                        | Credential setup required for Google Gemini nodes.                                                          |
| The design assumes that queries fall strictly into one of four categories; ambiguous queries may require manual review or refinement of classification prompts.  | Edge case noted in query classification analysis.                                                           |
| Chat memory buffers use a fixed window of 10 messages to maintain context; adjust window length for longer or shorter memory as needed.                         | Memory nodes configuration.                                                                                   |
| In case of empty retrieval results, the final answer generation node will acknowledge limitations based on prompts instructions.                               | Error handling via prompt design, no explicit error nodes implemented.                                       |
| The workflow is suitable for advanced chatbot applications requiring dynamic multi-strategy information retrieval and generation.                              | Workflow goal and use case.                                                                                   |

---

**Disclaimer:** The provided content is derived exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal or offensive material. All manipulated data is legal and publicly accessible.

---