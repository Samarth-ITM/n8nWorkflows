Analyze and modernize university curricula with GPT-4o and employment data

https://n8nworkflows.xyz/workflows/analyze-and-modernize-university-curricula-with-gpt-4o-and-employment-data-13899


# Analyze and modernize university curricula with GPT-4o and employment data

# 1. Workflow Overview

This workflow analyzes a university curriculum and produces a structured modernization plan by combining three categories of input:

- graduate employment data
- enrollment pattern data
- curriculum/syllabus documents in PDF form

It uses a multi-agent AI architecture built in n8n with OpenAI models and a semantic knowledge base. The workflow loads and combines institutional data, converts curriculum content into embeddings for semantic search, delegates analysis to two specialist AI agents, and then synthesizes the results into a final modernization plan that is stored in a data table.

## 1.1 Input Reception and Source Loading

The workflow starts manually and fans out into three parallel branches:

- graduate employment data retrieval
- enrollment pattern retrieval
- syllabus PDF extraction

These three streams are then merged into a single combined input.

## 1.2 Knowledge Base Construction

The merged data is used to build an in-memory vector store. Curriculum content is loaded as documents, split into chunks, embedded with OpenAI embeddings, and inserted into the vector store so later agents can query it semantically.

## 1.3 Specialized Curriculum Analysis Tools

A semantic retrieval tool is exposed from the vector store for semantic search. A code-based cognitive load tool is also made available for Bloom’s taxonomy-based complexity scoring. In parallel, an employment data query tool exposes graduate employment records for agent use.

## 1.4 Multi-Agent Analysis

Two specialist agent tools are configured:

- **Learning Outcome Alignment Agent**  
  Maps learning outcomes to Bloom’s taxonomy, detects redundancy, and assesses cognitive load.
- **Industry Demand Forecast Agent**  
  Analyzes employment data and skill demand trends to identify curriculum gaps and redesign opportunities.

Both agents use GPT-4o and structured output parsers.

## 1.5 Supervisory Synthesis and Storage

A top-level supervisor agent coordinates both specialist agents and produces a unified modernization strategy. The result is parsed into a strict schema, transformed into storage-ready fields, and saved to a target data table.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Workflow Trigger and Parallel Data Intake

### Overview
This block starts the workflow and loads the three primary data sources needed for analysis. It establishes the raw inputs that will later be merged and embedded.

### Nodes Involved
- Start Curriculum Analysis
- Load Graduate Employment Data
- Load Enrollment Patterns
- Extract Course Syllabi

### Node Details

#### Start Curriculum Analysis
- **Type and role:** `n8n-nodes-base.manualTrigger`; manual execution entry point.
- **Configuration choices:** No parameters are set; the node simply starts the workflow when triggered manually.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input. Outputs to:
  - Load Graduate Employment Data
  - Load Enrollment Patterns
  - Extract Course Syllabi
- **Version-specific requirements:** Type version 1.
- **Edge cases or failure types:** None beyond manual execution context.
- **Sub-workflow reference:** None.

#### Load Graduate Employment Data
- **Type and role:** `n8n-nodes-base.dataTable`; retrieves graduate employment data from an n8n Data Table.
- **Configuration choices:** Uses `operation: get` and a placeholder table ID for the graduate employment table.
- **Key expressions or variables used:** Data table ID placeholder: `graduate_employment_table`.
- **Input and output connections:** Input from Start Curriculum Analysis. Output to Combine Data Sources.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or failure types:**
  - invalid or missing data table ID
  - permission/access issues
  - empty table result
- **Sub-workflow reference:** None.

#### Load Enrollment Patterns
- **Type and role:** `n8n-nodes-base.dataTable`; retrieves enrollment trend data from an n8n Data Table.
- **Configuration choices:** Uses `operation: get` with placeholder ID for the enrollment table.
- **Key expressions or variables used:** Data table ID placeholder: `enrollment_patterns_table`.
- **Input and output connections:** Input from Start Curriculum Analysis. Output to Combine Data Sources.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or failure types:**
  - invalid table ID
  - empty or inconsistent enrollment records
  - access permission issues
- **Sub-workflow reference:** None.

#### Extract Course Syllabi
- **Type and role:** `n8n-nodes-base.extractFromFile`; extracts text from PDF syllabus files.
- **Configuration choices:** `operation: pdf`; no extra extraction options configured.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from Start Curriculum Analysis. Output to Combine Data Sources.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or failure types:**
  - missing binary PDF input
  - malformed or scanned PDFs with poor OCR suitability
  - empty extraction result
- **Sub-workflow reference:** None.

---

## 2.2 Block: Data Merge and Curriculum Embedding Pipeline

### Overview
This block combines all incoming source data and builds a searchable curriculum knowledge base. It uses document loading, chunking, and embedding generation to populate an in-memory vector store.

### Nodes Involved
- Combine Data Sources
- Curriculum Knowledge Base
- Generate Embeddings
- Load Curriculum Documents
- Split Curriculum Text

### Node Details

#### Combine Data Sources
- **Type and role:** `n8n-nodes-base.merge`; consolidates three upstream data streams.
- **Configuration choices:** Configured with `numberInputs: 3`, meaning it expects all three branches.
- **Key expressions or variables used:** None.
- **Input and output connections:** Inputs from:
  - Load Graduate Employment Data
  - Load Enrollment Patterns
  - Extract Course Syllabi  
  Output to Curriculum Knowledge Base.
- **Version-specific requirements:** Type version 3.2.
- **Edge cases or failure types:**
  - one or more upstream branches return no items
  - mismatched item counts depending on merge semantics
  - unexpected merged structure affecting downstream nodes
- **Sub-workflow reference:** None.

#### Curriculum Knowledge Base
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStoreInMemory`; in-memory vector store used in insert mode to build a semantic index.
- **Configuration choices:**
  - `mode: insert`
  - memory key set to `vector_store_key`
- **Key expressions or variables used:** Memory key `vector_store_key`.
- **Input and output connections:**
  - Main input from Combine Data Sources
  - AI embedding input from Generate Embeddings
  - AI document input from Load Curriculum Documents
  - Main output to Curriculum Modernization Supervisor
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failure types:**
  - document loader not supplying valid documents
  - embeddings not connected or failing
  - memory store lifecycle limitations because this is in-memory only
- **Sub-workflow reference:** None.

#### Generate Embeddings
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; generates embeddings for document chunks.
- **Configuration choices:** Default embedding options; uses OpenAI credentials.
- **Key expressions or variables used:** None.
- **Input and output connections:** AI embedding output to Curriculum Knowledge Base.
- **Version-specific requirements:** Type version 1.2; requires valid OpenAI API credentials.
- **Edge cases or failure types:**
  - invalid OpenAI credentials
  - API quota/rate limit issues
  - model or endpoint availability problems
- **Sub-workflow reference:** None.

#### Load Curriculum Documents
- **Type and role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`; converts incoming content into LangChain-style documents.
- **Configuration choices:**
  - default options
  - `textSplittingMode: custom`, meaning it expects a separate text splitter connection
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - AI text splitter input from Split Curriculum Text
  - AI document output to Curriculum Knowledge Base
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or failure types:**
  - incoming merged data not in a shape suitable for document creation
  - no text produced from syllabus extraction
  - metadata or content field mismatch
- **Sub-workflow reference:** None.

#### Split Curriculum Text
- **Type and role:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`; chunks long curriculum text into retrievable segments.
- **Configuration choices:** Default options only; no custom chunk size or overlap is specified.
- **Key expressions or variables used:** None.
- **Input and output connections:** AI text splitter output to Load Curriculum Documents.
- **Version-specific requirements:** Type version 1.
- **Edge cases or failure types:**
  - chunking defaults may be suboptimal for long syllabi
  - poor chunk boundaries can reduce retrieval accuracy
- **Sub-workflow reference:** None.

---

## 2.3 Block: Retrieval and Analytical Tooling for Agents

### Overview
This block exposes reusable tools to the AI agents. It includes semantic search over the curriculum corpus, an employment data query interface, and a JavaScript-based cognitive load calculator.

### Nodes Involved
- Semantic Retrieval Tool
- Query Embeddings
- Employment Data Query Tool
- Cognitive Load Calculator

### Node Details

#### Semantic Retrieval Tool
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStoreInMemory`; the same vector-store family, but configured in retrieval-as-tool mode.
- **Configuration choices:**
  - `mode: retrieve-as-tool`
  - memory key `curriculum_semantic_search`
  - custom tool description explaining semantic search purpose
- **Key expressions or variables used:** Memory key `curriculum_semantic_search`.
- **Input and output connections:**
  - AI embedding input from Query Embeddings
  - AI tool output to:
    - Learning Outcome Alignment Agent
    - Industry Demand Forecast Agent
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failure types:**
  - memory key mismatch with inserted data
  - empty vector store
  - retrieval returning irrelevant results if chunking/embedding quality is poor
- **Sub-workflow reference:** None.

#### Query Embeddings
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; generates embeddings for retrieval queries.
- **Configuration choices:** Default options; OpenAI credentials attached.
- **Key expressions or variables used:** None.
- **Input and output connections:** AI embedding output to Semantic Retrieval Tool.
- **Version-specific requirements:** Type version 1.2; requires OpenAI credentials.
- **Edge cases or failure types:**
  - OpenAI rate limits or authentication failures
  - retrieval tool unusable if this node fails
- **Sub-workflow reference:** None.

#### Employment Data Query Tool
- **Type and role:** `n8n-nodes-base.dataTableTool`; exposes graduate employment data as an AI-usable tool.
- **Configuration choices:**
  - `operation: get`
  - same graduate employment table placeholder ID used earlier
- **Key expressions or variables used:** Data table placeholder `graduate_employment_table`.
- **Input and output connections:** AI tool output to Industry Demand Forecast Agent.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or failure types:**
  - inaccessible data table
  - schema inconsistency in employment data
  - excessively broad tool queries leading to low-value responses
- **Sub-workflow reference:** None.

#### Cognitive Load Calculator
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; custom JavaScript AI tool for Bloom-level weighted scoring.
- **Configuration choices:**
  - accepts AI-provided `course_data`
  - maps Bloom levels to weights:
    - remember: 1
    - understand: 2
    - apply: 3
    - analyze: 4
    - evaluate: 5
    - create: 6
  - computes:
    - `total_cognitive_load`
    - `average_load`
    - `outcome_count`
    - `load_category`
    - `is_balanced`
- **Key expressions or variables used:**
  - `$fromAI('course_data', 'Course data with learning outcomes and Bloom levels', 'json')`
- **Input and output connections:** AI tool output to Learning Outcome Alignment Agent.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failure types:**
  - malformed AI-provided JSON
  - missing `learning_outcomes`
  - unsupported or misspelled Bloom levels default to weight 2
  - zero outcomes produces average 0 and may classify as low
- **Sub-workflow reference:** None.

---

## 2.4 Block: Learning Outcome Alignment Specialist

### Overview
This block defines a specialist agent focused on pedagogical analysis. It uses semantic search and the cognitive load calculator to map course outcomes, identify overlap, and detect curriculum balance issues.

### Nodes Involved
- Learning Outcome Alignment Agent
- Alignment Agent Model
- Alignment Output Parser

### Node Details

#### Learning Outcome Alignment Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; a callable agent tool used by the supervisor.
- **Configuration choices:**
  - task text is provided dynamically via:
    - `{{ $fromAI('analysis_task', 'The curriculum analysis task to perform') }}`
  - system message instructs the agent to:
    - map learning outcomes to Bloom’s taxonomy
    - identify gaps
    - detect redundancy via semantic similarity
    - calculate cognitive load
    - produce structured output
  - has output parser enabled
  - tool description explains its function to the supervisor
- **Key expressions or variables used:**
  - `$fromAI('analysis_task', ...)`
- **Input and output connections:**
  - AI language model input from Alignment Agent Model
  - AI tool inputs from:
    - Semantic Retrieval Tool
    - Cognitive Load Calculator
  - AI output parser input from Alignment Output Parser
  - AI tool output to Curriculum Modernization Supervisor
- **Version-specific requirements:** Type version 3.
- **Edge cases or failure types:**
  - model may produce data not matching parser schema
  - retrieval tool may return weak evidence if KB is poor
  - too little syllabus detail can reduce confidence in Bloom mapping
- **Sub-workflow reference:** None.

#### Alignment Agent Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4o chat model used by the alignment agent.
- **Configuration choices:**
  - model: `gpt-4o`
  - temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model output to Learning Outcome Alignment Agent.
- **Version-specific requirements:** Type version 1.3; valid OpenAI credentials required.
- **Edge cases or failure types:**
  - authentication failure
  - token limits for large contexts
  - model behavior changes across provider versions
- **Sub-workflow reference:** None.

#### Alignment Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces JSON schema for the alignment agent output.
- **Configuration choices:** Manual schema requiring:
  - `bloom_taxonomy_mapping`
  - `redundancy_analysis`
  - `cognitive_load_assessment`
  Optional:
  - `program_outcome_gaps`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI output parser output to Learning Outcome Alignment Agent.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failure types:**
  - schema validation failure if model omits required keys
  - wrong data types such as strings instead of arrays/numbers
- **Sub-workflow reference:** None.

---

## 2.5 Block: Industry Demand Forecast Specialist

### Overview
This block defines a specialist agent that studies graduate employment outcomes and labor-market relevance. It translates employment evidence into skill gap findings and redesign proposals.

### Nodes Involved
- Industry Demand Forecast Agent
- Forecast Agent Model
- Forecast Output Parser
- Employment Data Query Tool

### Node Details

#### Industry Demand Forecast Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; callable specialist agent used by the supervisor.
- **Configuration choices:**
  - task text supplied dynamically via:
    - `{{ $fromAI('forecast_task', 'The industry demand forecasting task to perform') }}`
  - system message directs it to:
    - analyze graduate outcomes
    - identify emerging skills
    - correlate enrollment and employment success
    - forecast future needs
    - create accreditation-ready justification
  - structured output enabled
- **Key expressions or variables used:**
  - `$fromAI('forecast_task', ...)`
- **Input and output connections:**
  - AI language model input from Forecast Agent Model
  - AI tool inputs from:
    - Employment Data Query Tool
    - Semantic Retrieval Tool
  - AI output parser input from Forecast Output Parser
  - AI tool output to Curriculum Modernization Supervisor
- **Version-specific requirements:** Type version 3.
- **Edge cases or failure types:**
  - missing or weak employment data weakens forecasting quality
  - parser schema mismatch
  - agent may overgeneralize if tool results are sparse
- **Sub-workflow reference:** None.

#### Forecast Agent Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4o model for the forecast agent.
- **Configuration choices:**
  - model: `gpt-4o`
  - temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model output to Industry Demand Forecast Agent.
- **Version-specific requirements:** Type version 1.3 with OpenAI credentials.
- **Edge cases or failure types:**
  - auth and quota issues
  - context constraints
- **Sub-workflow reference:** None.

#### Forecast Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates the forecast agent’s structured output.
- **Configuration choices:** Manual schema requiring:
  - `employment_trends`
  - `skill_gap_analysis`
  - `curriculum_redesign_proposals`
  Optional:
  - `accreditation_documentation`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI output parser output to Industry Demand Forecast Agent.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failure types:**
  - missing required arrays
  - invalid nested object shapes
- **Sub-workflow reference:** None.

#### Employment Data Query Tool
- Already described in Block 2.3; functionally belongs here as a key supporting tool for this agent.

---

## 2.6 Block: Supervisory Orchestration and Final Synthesis

### Overview
This block acts as the top-level coordinator. It invokes both specialist agents as tools, synthesizes their outputs, and generates the final modernization strategy in a strict schema.

### Nodes Involved
- Curriculum Modernization Supervisor
- Supervisor Model
- Supervisor Output Parser

### Node Details

#### Curriculum Modernization Supervisor
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; main orchestration agent.
- **Configuration choices:**
  - prompt type: `define`
  - fixed task prompt instructing the agent to analyze curriculum data and coordinate both specialist agents
  - system message explicitly defines workflow order:
    1. call learning outcome alignment agent
    2. call industry demand forecast agent
    3. synthesize into a unified modernization plan
  - output parser enabled
- **Key expressions or variables used:**
  - inline expression returning the full instruction text
- **Input and output connections:**
  - Main input from Curriculum Knowledge Base
  - AI language model input from Supervisor Model
  - AI tool inputs from:
    - Learning Outcome Alignment Agent
    - Industry Demand Forecast Agent
  - AI output parser input from Supervisor Output Parser
  - Main output to Prepare Results for Storage
- **Version-specific requirements:** Type version 3.1.
- **Edge cases or failure types:**
  - tool-calling may fail if specialist agents return invalid outputs
  - if the vector store was not populated correctly, analysis quality degrades
  - final parser validation may fail on malformed synthesis
- **Sub-workflow reference:** None.

#### Supervisor Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4o model used by the supervisory agent.
- **Configuration choices:**
  - model: `gpt-4o`
  - temperature: `0.3`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model output to Curriculum Modernization Supervisor.
- **Version-specific requirements:** Type version 1.3 with OpenAI credentials.
- **Edge cases or failure types:**
  - API quota/authentication issues
  - possible variability due to slightly higher temperature than specialist models
- **Sub-workflow reference:** None.

#### Supervisor Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates the final supervisory synthesis.
- **Configuration choices:** Manual schema requiring:
  - `executive_summary`
  - `learning_outcome_analysis`
  - `industry_demand_analysis`
  - `integrated_recommendations`
  Optional but expected in practice:
  - `implementation_roadmap`
  - `accreditation_documentation`
- **Key expressions or variables used:** None.
- **Input and output connections:** AI output parser output to Curriculum Modernization Supervisor.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or failure types:**
  - invalid nested arrays/objects
  - omitted required fields
  - stringified text where object structure is expected
- **Sub-workflow reference:** None.

---

## 2.7 Block: Result Preparation and Persistence

### Overview
This block converts the structured final AI output into fields suitable for storage and inserts the result into a destination data table.

### Nodes Involved
- Prepare Results for Storage
- Store Analysis Results

### Node Details

#### Prepare Results for Storage
- **Type and role:** `n8n-nodes-base.set`; reshapes final supervisor output for persistence.
- **Configuration choices:** Creates these fields:
  - `analysis_timestamp` = current ISO timestamp
  - `analysis_type` = `curriculum_modernization`
  - `executive_summary` = JSON string of `$json.output.executive_summary`
  - `learning_outcome_analysis` = JSON string of `$json.output.learning_outcome_analysis`
  - `industry_demand_analysis` = JSON string of `$json.output.industry_demand_analysis`
  - `integrated_recommendations` = JSON string of `$json.output.integrated_recommendations`
  - `implementation_roadmap` = JSON string of `$json.output.implementation_roadmap`
  - `accreditation_documentation` = JSON string of `$json.output.accreditation_documentation`
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
  - multiple `{{ JSON.stringify(...) }}`
  - source path `$json.output.*`
- **Input and output connections:** Input from Curriculum Modernization Supervisor. Output to Store Analysis Results.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or failure types:**
  - if supervisor output does not expose `output` as expected, expressions fail
  - `undefined` fields may stringify unexpectedly
  - field types are declared as object/array but values are actually stringified JSON
- **Sub-workflow reference:** None.

#### Store Analysis Results
- **Type and role:** `n8n-nodes-base.dataTable`; writes final analysis rows to a target data table.
- **Configuration choices:**
  - auto-maps input data
  - destination table placeholder `curriculum_analysis_results`
- **Key expressions or variables used:** Destination table ID placeholder.
- **Input and output connections:** Input from Prepare Results for Storage. No downstream output.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or failure types:**
  - write permissions or missing data table
  - schema mismatch between prepared fields and target table columns
  - auto-mapping may not behave as intended if column names differ
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Curriculum Analysis | manualTrigger | Manual entry point for the workflow |  | Load Graduate Employment Data; Load Enrollment Patterns; Extract Course Syllabi | ## How It Works<br>This workflow automates higher education curriculum analysis and modernisation using a multi-agent AI system. Designed for academic administrators, curriculum designers, and institutional planners, it eliminates manual effort in aligning course content with graduate employment outcomes and industry demand signals. The pipeline starts by concurrently loading graduate employment data, enrolment patterns, and extracting course syllabi from PDFs. These are merged and fed into a Curriculum Knowledge Base using semantic embeddings and text splitting. A Curriculum Modernisation Agent orchestrates two sub-agents: a Learning Outcome Alignment Agent (using semantic retrieval and cognitive load analysis) and an Industry Demand Forecast Agent (querying live employment data). Outputs are parsed and stored as structured analysis results, enabling institutions to make evidence-based curriculum decisions at scale. |
| Load Graduate Employment Data | dataTable | Loads graduate employment records from a data table | Start Curriculum Analysis | Combine Data Sources | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Load Enrollment Patterns | dataTable | Loads enrollment pattern records from a data table | Start Curriculum Analysis | Combine Data Sources | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Extract Course Syllabi | extractFromFile | Extracts text from PDF syllabi | Start Curriculum Analysis | Combine Data Sources | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Combine Data Sources | merge | Merges the three data inputs | Load Graduate Employment Data; Load Enrollment Patterns; Extract Course Syllabi | Curriculum Knowledge Base | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Load Curriculum Documents | documentDefaultDataLoader | Converts merged curriculum data into documents for vector storage | Split Curriculum Text | Curriculum Knowledge Base | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Split Curriculum Text | textSplitterRecursiveCharacterTextSplitter | Splits curriculum text into chunks |  | Load Curriculum Documents | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Generate Embeddings | embeddingsOpenAi | Creates embeddings for curriculum chunks |  | Curriculum Knowledge Base | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Curriculum Knowledge Base | vectorStoreInMemory | Stores embedded curriculum documents for later semantic use | Combine Data Sources; Load Curriculum Documents; Generate Embeddings | Curriculum Modernization Supervisor | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Semantic Retrieval Tool | vectorStoreInMemory | Exposes semantic curriculum search as an AI tool | Query Embeddings | Learning Outcome Alignment Agent; Industry Demand Forecast Agent | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage.<br><br>## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Query Embeddings | embeddingsOpenAi | Embeds semantic search queries |  | Semantic Retrieval Tool | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage.<br><br>## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Employment Data Query Tool | dataTableTool | Exposes graduate employment data as an AI tool |  | Industry Demand Forecast Agent | ## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Cognitive Load Calculator | toolCode | Calculates Bloom-based cognitive load metrics |  | Learning Outcome Alignment Agent | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage. |
| Learning Outcome Alignment Agent | agentTool | Specialist agent for taxonomy alignment, redundancy, and load analysis | Alignment Agent Model; Semantic Retrieval Tool; Cognitive Load Calculator | Curriculum Modernization Supervisor | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage. |
| Alignment Agent Model | lmChatOpenAi | GPT-4o model backing the alignment agent |  | Learning Outcome Alignment Agent | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage. |
| Alignment Output Parser | outputParserStructured | Enforces structured output for the alignment agent |  | Learning Outcome Alignment Agent | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage. |
| Industry Demand Forecast Agent | agentTool | Specialist agent for labor-market and skill-demand forecasting | Forecast Agent Model; Semantic Retrieval Tool; Employment Data Query Tool | Curriculum Modernization Supervisor | ## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Forecast Agent Model | lmChatOpenAi | GPT-4o model backing the forecast agent |  | Industry Demand Forecast Agent | ## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Forecast Output Parser | outputParserStructured | Enforces structured output for the forecast agent |  | Industry Demand Forecast Agent | ## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Curriculum Modernization Supervisor | agent | Main orchestrator that coordinates both specialist agents and synthesizes the final plan | Curriculum Knowledge Base; Supervisor Model; Learning Outcome Alignment Agent; Industry Demand Forecast Agent | Prepare Results for Storage | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage.<br><br>## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Supervisor Model | lmChatOpenAi | GPT-4o model for the top-level supervisor |  | Curriculum Modernization Supervisor | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage. |
| Supervisor Output Parser | outputParserStructured | Enforces structured output for the supervisor |  | Curriculum Modernization Supervisor | ## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Prepare Results for Storage | set | Converts final output into storage-ready fields | Curriculum Modernization Supervisor | Store Analysis Results | ## Store Results<br>**What** — Prepares and stores final modernisation analysis.<br>**Why** — Produces structured outputs ready for institutional review and action. |
| Store Analysis Results | dataTable | Persists the final structured analysis in a data table | Prepare Results for Storage |  | ## Store Results<br>**What** — Prepares and stores final modernisation analysis.<br>**Why** — Produces structured outputs ready for institutional review and action. |
| Sticky Note | stickyNote | Documentation note about prerequisites, use cases, customization, and benefits |  |  | ## Prerequisites<br>- Vector store (e.g., Pinecone, Qdrant, or Supabase)<br>- Graduate employment & enrolment data (CSV or DB)<br>- Course syllabi in PDF format<br>## Use Cases<br>- Annual curriculum review aligned to graduate employment trends<br>## Customisation<br>- Swap embedding models for domain-specific academic corpora<br>## Benefits<br>- Automates labour-intensive curriculum mapping processes |
| Sticky Note1 | stickyNote | Documentation note listing setup actions |  |  | ## Setup Steps<br>1. Add OpenAI or compatible LLM API credentials to all Chat Model and Embedding nodes.<br>2. Connect graduate employment and enrolment data sources.<br>3. Set up vector store credentials for the `Curriculum Knowledge Base` node.<br>4. Configure `Employment Data Query Tool` with your labour market data source or API.<br>5. Update `Store Analysis Results` with your target storage destination. |
| Sticky Note2 | stickyNote | Documentation note describing the workflow behavior |  |  | ## How It Works<br>This workflow automates higher education curriculum analysis and modernisation using a multi-agent AI system. Designed for academic administrators, curriculum designers, and institutional planners, it eliminates manual effort in aligning course content with graduate employment outcomes and industry demand signals. The pipeline starts by concurrently loading graduate employment data, enrolment patterns, and extracting course syllabi from PDFs. These are merged and fed into a Curriculum Knowledge Base using semantic embeddings and text splitting. A Curriculum Modernisation Agent orchestrates two sub-agents: a Learning Outcome Alignment Agent (using semantic retrieval and cognitive load analysis) and an Industry Demand Forecast Agent (querying live employment data). Outputs are parsed and stored as structured analysis results, enabling institutions to make evidence-based curriculum decisions at scale. |
| Sticky Note3 | stickyNote | Documentation note about the forecast agent area |  |  | ## Forecast Agent<br>**What** — Queries employment data to forecast industry skill demand.<br>**Why** — Grounds curriculum recommendations in real labour market signals. |
| Sticky Note4 | stickyNote | Documentation note about the alignment agent area |  |  | ## Alignment Agent<br>**What** — Analyses learning outcomes against curriculum content using semantic retrieval.<br>**Why** — Identifies gaps between stated outcomes and actual course coverage. |
| Sticky Note5 | stickyNote | Documentation note about merge and embedding |  |  | ## Combine & Embed<br>**What** — Merges data sources, splits curriculum text, and generates semantic embeddings.<br>**Why** — Creates a searchable knowledge base for accurate alignment analysis. |
| Sticky Note6 | stickyNote | Documentation note about final storage |  |  | ## Store Results<br>**What** — Prepares and stores final modernisation analysis.<br>**Why** — Produces structured outputs ready for institutional review and action. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Smart curriculum modernisation with learning outcome and industry demand`.
   - Keep execution order at default unless you specifically need sequential behavior.

2. **Add the trigger node**
   - Create a **Manual Trigger** node named `Start Curriculum Analysis`.

3. **Add the graduate employment source**
   - Create a **Data Table** node named `Load Graduate Employment Data`.
   - Set operation to **Get**.
   - Select or create the graduate employment data table.
   - Connect `Start Curriculum Analysis` → `Load Graduate Employment Data`.

4. **Add the enrollment source**
   - Create a **Data Table** node named `Load Enrollment Patterns`.
   - Set operation to **Get**.
   - Select or create the enrollment patterns data table.
   - Connect `Start Curriculum Analysis` → `Load Enrollment Patterns`.

5. **Add the syllabus extraction node**
   - Create an **Extract From File** node named `Extract Course Syllabi`.
   - Set operation to **PDF**.
   - Ensure your workflow execution will provide binary PDF input to this node.
   - Connect `Start Curriculum Analysis` → `Extract Course Syllabi`.

6. **Add the merge node**
   - Create a **Merge** node named `Combine Data Sources`.
   - Set number of inputs to **3**.
   - Connect:
     - `Load Graduate Employment Data` → input 1
     - `Load Enrollment Patterns` → input 2
     - `Extract Course Syllabi` → input 3

7. **Add the embedding model for document insertion**
   - Create an **OpenAI Embeddings** node named `Generate Embeddings`.
   - Add your OpenAI credential.
   - Keep default options unless you want a specific embedding model.

8. **Add the text splitter**
   - Create a **Recursive Character Text Splitter** node named `Split Curriculum Text`.
   - Leave defaults initially.
   - Optionally tune chunk size and overlap later for long syllabi.

9. **Add the document loader**
   - Create a **Default Data Loader** node named `Load Curriculum Documents`.
   - Set text splitting mode to **Custom**.
   - Connect the text splitter AI port:
     - `Split Curriculum Text` → `Load Curriculum Documents` (`ai_textSplitter`)

10. **Add the vector store for insertion**
    - Create a **Vector Store In-Memory** node named `Curriculum Knowledge Base`.
    - Set mode to **Insert**.
    - Set memory key to `vector_store_key`.
    - Connect:
      - `Combine Data Sources` → `Curriculum Knowledge Base` (main)
      - `Generate Embeddings` → `Curriculum Knowledge Base` (`ai_embedding`)
      - `Load Curriculum Documents` → `Curriculum Knowledge Base` (`ai_document`)

11. **Add the retrieval embedding node**
    - Create another **OpenAI Embeddings** node named `Query Embeddings`.
    - Use the same OpenAI credential.

12. **Add the semantic retrieval tool**
    - Create another **Vector Store In-Memory** node named `Semantic Retrieval Tool`.
    - Set mode to **Retrieve as Tool**.
    - Set memory key to `curriculum_semantic_search`.
    - Add a tool description similar to:  
      `Search curriculum documents, syllabi, and learning outcomes using semantic similarity to find related content, redundancies, and alignment patterns`
    - Connect `Query Embeddings` → `Semantic Retrieval Tool` (`ai_embedding`).

13. **Important implementation note**
    - The JSON uses `vector_store_key` for insertion and `curriculum_semantic_search` for retrieval.
    - In practice, if you want the retrieval tool to query the same in-memory store, use the **same memory key** for both insert and retrieve nodes.
    - Otherwise, the retrieval node may point to a different empty store.
    - This is the main modernization/fix recommendation for the workflow.

14. **Add the employment data tool**
    - Create a **Data Table Tool** node named `Employment Data Query Tool`.
    - Set operation to **Get**.
    - Point it to the same graduate employment data table used earlier.

15. **Add the cognitive load code tool**
    - Create a **Tool Code** node named `Cognitive Load Calculator`.
    - Paste the JavaScript logic that:
      - receives `course_data` from AI
      - reads `learning_outcomes`
      - maps Bloom levels to numeric weights
      - returns total load, average load, count, category, and balance flag
    - Use the exact input concept:
      - `$fromAI('course_data', 'Course data with learning outcomes and Bloom levels', 'json')`

16. **Add the learning outcome model**
    - Create a **OpenAI Chat Model** node named `Alignment Agent Model`.
    - Select model `gpt-4o`.
    - Set temperature to `0.2`.
    - Add OpenAI credentials.

17. **Add the alignment output parser**
    - Create a **Structured Output Parser** node named `Alignment Output Parser`.
    - Use manual schema mode.
    - Define a schema with:
      - `bloom_taxonomy_mapping` array
      - `redundancy_analysis` array
      - `cognitive_load_assessment` object
      - optional `program_outcome_gaps` array

18. **Add the learning outcome specialist**
    - Create an **Agent Tool** node named `Learning Outcome Alignment Agent`.
    - Set its text prompt to:
      - `{{ $fromAI('analysis_task', 'The curriculum analysis task to perform') }}`
    - Enable structured output.
    - Add the system message describing its role in:
      - Bloom taxonomy mapping
      - redundancy detection
      - cognitive load assessment
      - recommendation generation
    - Add a tool description stating it analyzes syllabi and rubrics for learning outcome alignment.
    - Connect:
      - `Alignment Agent Model` → `Learning Outcome Alignment Agent` (`ai_languageModel`)
      - `Alignment Output Parser` → `Learning Outcome Alignment Agent` (`ai_outputParser`)
      - `Semantic Retrieval Tool` → `Learning Outcome Alignment Agent` (`ai_tool`)
      - `Cognitive Load Calculator` → `Learning Outcome Alignment Agent` (`ai_tool`)

19. **Add the forecast model**
    - Create an **OpenAI Chat Model** node named `Forecast Agent Model`.
    - Select model `gpt-4o`.
    - Set temperature to `0.2`.
    - Add OpenAI credentials.

20. **Add the forecast output parser**
    - Create a **Structured Output Parser** node named `Forecast Output Parser`.
    - Manual schema should include:
      - `employment_trends`
      - `skill_gap_analysis`
      - `curriculum_redesign_proposals`
      - optional `accreditation_documentation`

21. **Add the industry demand specialist**
    - Create an **Agent Tool** node named `Industry Demand Forecast Agent`.
    - Set text prompt to:
      - `{{ $fromAI('forecast_task', 'The industry demand forecasting task to perform') }}`
    - Enable structured output.
    - Add the system message covering:
      - employment outcome analysis
      - trend detection
      - enrollment/employment correlation
      - future skill forecasting
      - curriculum redesign
      - accreditation-ready evidence
    - Add a tool description describing labor market and redesign analysis.
    - Connect:
      - `Forecast Agent Model` → `Industry Demand Forecast Agent` (`ai_languageModel`)
      - `Forecast Output Parser` → `Industry Demand Forecast Agent` (`ai_outputParser`)
      - `Employment Data Query Tool` → `Industry Demand Forecast Agent` (`ai_tool`)
      - `Semantic Retrieval Tool` → `Industry Demand Forecast Agent` (`ai_tool`)

22. **Add the supervisor model**
    - Create an **OpenAI Chat Model** node named `Supervisor Model`.
    - Select model `gpt-4o`.
    - Set temperature to `0.3`.
    - Add OpenAI credentials.

23. **Add the supervisor parser**
    - Create a **Structured Output Parser** node named `Supervisor Output Parser`.
    - Manual schema should include:
      - `executive_summary`
      - `learning_outcome_analysis`
      - `industry_demand_analysis`
      - `integrated_recommendations`
      - optional `implementation_roadmap`
      - optional `accreditation_documentation`

24. **Add the top-level supervisor**
    - Create an **Agent** node named `Curriculum Modernization Supervisor`.
    - Set prompt type to **Define**.
    - Set the main text to a fixed instruction telling the agent to:
      - analyze curriculum data
      - coordinate both specialist agents
      - synthesize a modernization plan
    - Add the system message that explicitly orders:
      1. learning outcome analysis
      2. employment/forecast analysis
      3. integrated synthesis
    - Enable structured output.
    - Connect:
      - `Supervisor Model` → `Curriculum Modernization Supervisor` (`ai_languageModel`)
      - `Supervisor Output Parser` → `Curriculum Modernization Supervisor` (`ai_outputParser`)
      - `Learning Outcome Alignment Agent` → `Curriculum Modernization Supervisor` (`ai_tool`)
      - `Industry Demand Forecast Agent` → `Curriculum Modernization Supervisor` (`ai_tool`)
      - `Curriculum Knowledge Base` → `Curriculum Modernization Supervisor` (main)

25. **Add the result formatting node**
    - Create a **Set** node named `Prepare Results for Storage`.
    - Add these fields:
      - `analysis_timestamp` as string: `{{ $now.toISO() }}`
      - `analysis_type` as string: `curriculum_modernization`
      - `executive_summary` from `{{ JSON.stringify($json.output.executive_summary) }}`
      - `learning_outcome_analysis` from `{{ JSON.stringify($json.output.learning_outcome_analysis) }}`
      - `industry_demand_analysis` from `{{ JSON.stringify($json.output.industry_demand_analysis) }}`
      - `integrated_recommendations` from `{{ JSON.stringify($json.output.integrated_recommendations) }}`
      - `implementation_roadmap` from `{{ JSON.stringify($json.output.implementation_roadmap) }}`
      - `accreditation_documentation` from `{{ JSON.stringify($json.output.accreditation_documentation) }}`
    - Connect `Curriculum Modernization Supervisor` → `Prepare Results for Storage`.

26. **Add the storage node**
    - Create a **Data Table** node named `Store Analysis Results`.
    - Set it to write to your target results table.
    - Use auto-mapping if your column names match the Set node fields.
    - Connect `Prepare Results for Storage` → `Store Analysis Results`.

27. **Configure credentials**
    - OpenAI credential must be attached to:
      - Generate Embeddings
      - Query Embeddings
      - Alignment Agent Model
      - Forecast Agent Model
      - Supervisor Model
    - Data table access must be available for:
      - Load Graduate Employment Data
      - Load Enrollment Patterns
      - Employment Data Query Tool
      - Store Analysis Results

28. **Prepare the input data**
    - Graduate employment table should ideally include:
      - job title
      - sector
      - salary band
      - employment rate
      - graduation year
      - skill tags
    - Enrollment table should ideally include:
      - course/program identifier
      - year/term
      - enrollment counts
      - retention or completion indicators
    - Syllabus PDFs should contain:
      - course codes
      - learning outcomes
      - topics
      - assessments
      - instructional format if possible

29. **Recommended modernization improvements**
    - Use the same vector memory key for insert and retrieve nodes.
    - Consider replacing the in-memory vector store with Pinecone, Qdrant, Supabase, or another persistent vector DB for production.
    - Consider adding preprocessing before the document loader to normalize merged data.
    - Consider adding validation/error branches for empty tables or PDF extraction failures.
    - Consider replacing JSON-string storage with native JSON fields if your storage backend supports them.

30. **Run and verify**
    - Trigger the workflow manually.
    - Confirm:
      - source tables return records
      - PDF extraction returns text
      - vector store receives chunked documents
      - specialist agents produce valid parsed outputs
      - supervisor output matches schema
      - results are stored in the target table

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: Vector store (e.g., Pinecone, Qdrant, or Supabase); Graduate employment & enrolment data (CSV or DB); Course syllabi in PDF format | General workflow prerequisites |
| Use case: Annual curriculum review aligned to graduate employment trends | General usage |
| Customisation: Swap embedding models for domain-specific academic corpora | General design note |
| Benefit: Automates labour-intensive curriculum mapping processes | General design note |
| Setup step: Add OpenAI or compatible LLM API credentials to all Chat Model and Embedding nodes | Configuration guidance |
| Setup step: Connect graduate employment and enrolment data sources | Configuration guidance |
| Setup step: Set up vector store credentials for the `Curriculum Knowledge Base` node | Configuration guidance; note that the current JSON actually uses an in-memory vector store, so persistent vector-store credentials are only needed if you replace it |
| Setup step: Configure `Employment Data Query Tool` with your labour market data source or API | Configuration guidance |
| Setup step: Update `Store Analysis Results` with your target storage destination | Configuration guidance |
| Workflow intent: automates higher education curriculum analysis and modernisation using a multi-agent AI system for academic administrators, curriculum designers, and institutional planners | Project context |

## Additional implementation note
Although the sticky notes mention external vector stores such as Pinecone, Qdrant, or Supabase, the actual workflow JSON uses **in-memory vector store nodes**. That is acceptable for prototyping, but for persistent and scalable usage, replacing both vector-store nodes with a persistent backend would be the most important production upgrade.