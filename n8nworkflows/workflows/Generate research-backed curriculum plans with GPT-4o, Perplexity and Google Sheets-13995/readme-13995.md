Generate research-backed curriculum plans with GPT-4o, Perplexity and Google Sheets

https://n8nworkflows.xyz/workflows/generate-research-backed-curriculum-plans-with-gpt-4o--perplexity-and-google-sheets-13995


# Generate research-backed curriculum plans with GPT-4o, Perplexity and Google Sheets

# 1. Workflow Overview

This workflow generates a research-backed curriculum plan using a multi-agent AI architecture in n8n. It starts from a manual trigger, sends the curriculum request to a supervisor agent powered by GPT-4o, and lets that supervisor delegate work to three specialist agents:

- a **Research Agent** for standards, pedagogy, and subject-matter research,
- a **Content Creation Agent** for lesson and sequence design,
- an **Assessment Agent** for evaluation frameworks and rubrics.

The supervisor synthesizes the specialist outputs into a unified curriculum plan, then the workflow formats the result and stores it in an n8n Data Table. In parallel, the supervisor also has access to a Google Sheets tool for sheet-based storage/retrieval during planning.

## 1.1 Input Reception and Orchestration

The workflow begins with a manual trigger and passes the input to the Curriculum Supervisor Agent. The supervisor uses GPT-4o and a memory buffer to coordinate the overall plan generation.

## 1.2 Research Block

The supervisor can invoke a dedicated Research Agent. That agent uses GPT-4o plus external research tools: SerpAPI-based Google Search, Perplexity, and Wikipedia.

## 1.3 Content Design Block

The supervisor can invoke a Content Creation Agent. This agent uses GPT-4o and helper tools such as a calculator and code execution to structure lessons, timing, and curriculum components.

## 1.4 Assessment Design Block

The supervisor can invoke an Assessment Agent. This agent uses GPT-4o plus calculator and code tools to build rubrics, grading logic, and assessment frameworks aligned with learning objectives.

## 1.5 Formatting and Storage

Once the supervisor produces a final plan, a Set node reformats the output into structured fields, then an n8n Data Table node stores the curriculum record. Separately, the supervisor also has access to a Google Sheets tool, intended for storing or retrieving planning data from Google Sheets as part of the agentic process.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Supervisor Orchestration

### Overview
This block is the entry point and control center of the workflow. It accepts the curriculum request, initializes the main AI orchestration layer, preserves short-term planning context, and lets the supervisor decide when to call specialist agents or the Google Sheets tool.

### Nodes Involved
- Start Curriculum Planning
- Curriculum Supervisor Agent
- Supervisor Model
- Planning Memory
- Google Sheets Tool

### Node Details

#### Start Curriculum Planning
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for interactive execution.
- **Configuration choices:** No parameters are set. It simply starts the workflow when manually run.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Curriculum Supervisor Agent**.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None at runtime beyond normal manual execution constraints.
- **Sub-workflow reference:** None.

#### Curriculum Supervisor Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; top-level orchestration agent that interprets the curriculum request and coordinates specialist tools/agents.
- **Configuration choices:**
  - Main prompt text is:
    - `{{$json.curriculum_request || 'Create a comprehensive curriculum plan for a high school biology course covering cell biology, genetics, and evolution.'}}`
  - This means it uses an incoming `curriculum_request` field if present; otherwise it falls back to a default biology-course request.
  - System message defines the supervisor role and explicitly instructs it to coordinate:
    1. Research Agent
    2. Content Creation Agent
    3. Assessment Agent
  - It is instructed to synthesize all outputs into a cohesive curriculum plan.
- **Key expressions or variables used:**
  - `$json.curriculum_request`
  - fallback static curriculum request
- **Input and output connections:**
  - Main input from **Start Curriculum Planning**
  - AI language model input from **Supervisor Model**
  - AI memory input from **Planning Memory**
  - AI tool inputs from:
    - **Research Agent**
    - **Content Creation Agent**
    - **Assessment Agent**
    - **Google Sheets Tool**
  - Main output to **Prepare Curriculum Data**
- **Version-specific requirements:** Type version 3.1; requires compatible n8n LangChain/AI node support.
- **Edge cases or potential failure types:**
  - Missing or malformed input may cause fallback behavior rather than failure.
  - LLM output may be inconsistent if the supervisor does not call tools as intended.
  - Tool invocation can fail if downstream credentials or parameters are invalid.
  - Long or ambiguous requests may lead to token pressure or poorly scoped delegation.
- **Sub-workflow reference:** None. This is not a sub-workflow node; it is the top-level coordinating agent.

#### Supervisor Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; OpenAI chat model backing the supervisor agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
  - Lower creativity than content design, favoring stable coordination and synthesis.
- **Key expressions or variables used:** None.
- **Input and output connections:** Provides AI language model connection to **Curriculum Supervisor Agent**.
- **Version-specific requirements:** Type version 1.3; requires valid OpenAI credentials and support for the selected model.
- **Edge cases or potential failure types:**
  - OpenAI authentication errors
  - model availability issues
  - rate limits
  - token/context errors
  - billing/quota problems
- **Sub-workflow reference:** None.

#### Planning Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; short-term conversational memory for the supervisor.
- **Configuration choices:**
  - Context window length: `10`
  - Keeps recent exchanges available so the supervisor can preserve planning context across agent/tool interactions.
- **Key expressions or variables used:** None.
- **Input and output connections:** Provides AI memory connection to **Curriculum Supervisor Agent**.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or potential failure types:**
  - Context may be insufficient for very long planning sessions.
  - Earlier decisions may be dropped once memory exceeds the configured buffer.
- **Sub-workflow reference:** None.

#### Google Sheets Tool
- **Type and technical role:** `n8n-nodes-base.googleSheetsTool`; AI-callable Google Sheets tool exposed to the supervisor.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - `sheetName` is not set
  - `documentId` is not set
  - Manual tool description: “Store and retrieve curriculum data, lesson plans, and assessment frameworks in Google Sheets”
- **Key expressions or variables used:** None in the current configuration.
- **Input and output connections:** Connected as an AI tool to **Curriculum Supervisor Agent**.
- **Version-specific requirements:** Type version 4.7; requires Google Sheets OAuth2 credentials and a valid sheet/document selection.
- **Edge cases or potential failure types:**
  - As configured, this node is incomplete because `sheetName` and `documentId` are blank.
  - OAuth2 authentication failure
  - missing spreadsheet permissions
  - append/update mismatch if expected key fields are absent
  - schema mismatch if the sheet columns do not align with tool-generated data
- **Sub-workflow reference:** None.

---

## 2.2 Research Block

### Overview
This block provides the factual and pedagogical foundation for curriculum planning. The supervisor delegates specific research tasks to the Research Agent, which uses GPT-4o together with search and knowledge tools to gather current information and supporting references.

### Nodes Involved
- Research Agent
- Research Model
- Google Search Tool
- Perplexity Research Tool
- Wikipedia Tool

### Node Details

#### Research Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; specialist AI agent exposed as a tool to the supervisor.
- **Configuration choices:**
  - Task prompt:
    - `{{$fromAI('research_task', 'The specific research task or question to investigate')}}`
  - This allows the supervisor to dynamically provide a structured task argument named `research_task`.
  - System message defines the role as educational research specialist covering:
    - standards
    - curriculum frameworks
    - pedagogical approaches
    - subject matter content
  - Tool description positions it as the source for foundational knowledge and research-backed inputs.
- **Key expressions or variables used:**
  - `$fromAI('research_task', ...)`
- **Input and output connections:**
  - AI language model from **Research Model**
  - AI tools from:
    - **Google Search Tool**
    - **Perplexity Research Tool**
    - **Wikipedia Tool**
  - Output as AI tool to **Curriculum Supervisor Agent**
- **Version-specific requirements:** Type version 3.
- **Edge cases or potential failure types:**
  - If the supervisor provides a vague `research_task`, the results may be broad or unfocused.
  - External tools can fail independently, causing incomplete research coverage.
  - Search results may vary over time.
  - Citation quality depends heavily on Perplexity/tool responses and prompt behavior.
- **Sub-workflow reference:** None.

#### Research Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; OpenAI model used by the Research Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
  - Low temperature favors factual consistency and less speculative output.
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model connection to **Research Agent**.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or potential failure types:**
  - OpenAI auth/quota/rate-limit issues
  - token or context overflow
  - model unavailability
- **Sub-workflow reference:** None.

#### Google Search Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolSerpApi`; web search tool for the Research Agent.
- **Configuration choices:** Default options only.
- **Key expressions or variables used:** None shown.
- **Input and output connections:** Exposed as an AI tool to **Research Agent**.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Requires valid SerpAPI / search API configuration in n8n environment or node credentials setup depending on deployment.
  - Search API quota exhaustion
  - transient network failures
  - inconsistent search ranking results
- **Sub-workflow reference:** None.

#### Perplexity Research Tool
- **Type and technical role:** `n8n-nodes-base.perplexityTool`; AI-powered research tool with citations.
- **Configuration choices:**
  - Manual description enabled
  - Tool description: “Conduct AI-powered research with citations on educational topics, learning theories, and subject matter content”
  - Default messages structure is present but not customized.
- **Key expressions or variables used:** None.
- **Input and output connections:** Exposed as an AI tool to **Research Agent**.
- **Version-specific requirements:** Type version 1; requires valid Perplexity API credentials.
- **Edge cases or potential failure types:**
  - Perplexity authentication failure
  - quota/rate limits
  - citation quality may vary
  - occasional tool response format variability
- **Sub-workflow reference:** None.

#### Wikipedia Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolWikipedia`; encyclopedia lookup tool.
- **Configuration choices:** Default configuration.
- **Key expressions or variables used:** None.
- **Input and output connections:** Exposed as an AI tool to **Research Agent**.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Topic coverage may be incomplete or generalized.
  - Content may be too broad for standards-alignment tasks.
  - Ambiguous topic names can yield irrelevant pages.
- **Sub-workflow reference:** None.

---

## 2.3 Content Design Block

### Overview
This block creates the actual instructional structure of the curriculum. The supervisor sends content-generation tasks to a dedicated Content Creation Agent, which uses GPT-4o and utility tools to build lesson plans, sequences, timing estimates, and organized learning experiences.

### Nodes Involved
- Content Creation Agent
- Content Model
- Calculator Tool
- Code Tool

### Node Details

#### Content Creation Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; specialist instructional-design agent exposed to the supervisor.
- **Configuration choices:**
  - Task prompt:
    - `{{$fromAI('content_task', 'The specific content creation task or lesson plan to develop')}}`
  - System message defines the role as designing:
    - lesson plans
    - learning activities
    - instructional sequences
    - educational materials
  - It is guided to create engaging, age-appropriate content aligned with learning objectives.
- **Key expressions or variables used:**
  - `$fromAI('content_task', ...)`
- **Input and output connections:**
  - AI language model from **Content Model**
  - AI tools from:
    - **Calculator Tool**
    - **Code Tool**
  - Output as AI tool to **Curriculum Supervisor Agent**
- **Version-specific requirements:** Type version 3.
- **Edge cases or potential failure types:**
  - If the supervisor does not provide sufficiently scoped content instructions, the output may be too generic.
  - Tool use may be unnecessary for some tasks, but the agent may still attempt to invoke tools.
  - Draft quality depends on how well research findings are synthesized by the supervisor before delegation.
- **Sub-workflow reference:** None.

#### Content Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM used by the Content Creation Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.5`
  - Higher than the research/assessment models to support more creative content generation.
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model connection to **Content Creation Agent**.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or potential failure types:**
  - OpenAI rate limits/auth failures
  - hallucinated educational details if research context is weak
  - inconsistent formatting if prompts are underspecified
- **Sub-workflow reference:** None.

#### Calculator Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCalculator`; arithmetic tool for AI agents.
- **Configuration choices:** Default.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Exposed as AI tool to **Content Creation Agent**
  - Also exposed as AI tool to **Assessment Agent**
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Limited to calculation use cases; cannot perform complex curriculum logic on its own.
  - If the agent uses it incorrectly for nonnumeric reasoning, results may be unhelpful.
- **Sub-workflow reference:** None.

#### Code Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`; executable code tool for structured transformations or logic.
- **Configuration choices:**
  - Description: “Execute custom code for complex curriculum structure generation, data transformations, and specialized calculations”
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Exposed as AI tool to **Content Creation Agent**
  - Exposed as AI tool to **Assessment Agent**
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or potential failure types:**
  - Runtime errors in generated code
  - malformed inputs/outputs
  - unsupported code assumptions
  - security restrictions depending on n8n environment
- **Sub-workflow reference:** None.

---

## 2.4 Assessment Design Block

### Overview
This block generates assessments and evaluation structures that align with the curriculum plan. The supervisor delegates assessment-specific tasks to a specialist agent that uses GPT-4o plus utility tools for scoring logic, rubric structures, and quantitative schemes.

### Nodes Involved
- Assessment Agent
- Assessment Model
- Calculator Tool
- Code Tool

### Node Details

#### Assessment Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; assessment specialist exposed to the supervisor.
- **Configuration choices:**
  - Task prompt:
    - `{{$fromAI('assessment_task', 'The specific assessment design task or evaluation framework to create')}}`
  - System message defines its role as creating:
    - assessment frameworks
    - rubrics
    - evaluation criteria
    - formative and summative assessments
    - grading strategies
  - The prompt emphasizes fairness, validity, and alignment with learning outcomes.
- **Key expressions or variables used:**
  - `$fromAI('assessment_task', ...)`
- **Input and output connections:**
  - AI language model from **Assessment Model**
  - AI tools from:
    - **Calculator Tool**
    - **Code Tool**
  - Output as AI tool to **Curriculum Supervisor Agent**
- **Version-specific requirements:** Type version 3.
- **Edge cases or potential failure types:**
  - Poorly specified assessment scope can lead to generic rubrics.
  - The agent may overuse calculator/code tools when narrative rubric design would suffice.
  - Alignment quality depends on upstream supervisor coordination.
- **Sub-workflow reference:** None.

#### Assessment Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; OpenAI model backing the Assessment Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
  - Keeps output structured and consistent for rubric/evaluation logic.
- **Key expressions or variables used:** None.
- **Input and output connections:** AI language model connection to **Assessment Agent**.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or potential failure types:**
  - OpenAI credential failure
  - API rate limiting
  - quota exhaustion
  - context length limitations
- **Sub-workflow reference:** None.

#### Calculator Tool
- **Type and technical role:** Shared arithmetic AI tool.
- **Configuration choices:** Default.
- **Key expressions or variables used:** None.
- **Input and output connections:** Shared with **Content Creation Agent** and **Assessment Agent**.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** Same as in the content block.
- **Sub-workflow reference:** None.

#### Code Tool
- **Type and technical role:** Shared executable-code AI tool.
- **Configuration choices:** Same description as above.
- **Key expressions or variables used:** None.
- **Input and output connections:** Shared with **Content Creation Agent** and **Assessment Agent**.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or potential failure types:** Same as in the content block.
- **Sub-workflow reference:** None.

---

## 2.5 Output Preparation and Storage

### Overview
This block converts the supervisor’s final output into a structured record and stores it in n8n’s Data Table. It is the final persistence layer of the workflow as currently wired.

### Nodes Involved
- Prepare Curriculum Data
- Store Curriculum Plan

### Node Details

#### Prepare Curriculum Data
- **Type and technical role:** `n8n-nodes-base.set`; creates the final structured payload before storage.
- **Configuration choices:**
  - Assigns five fields:
    - `curriculum_title` = `{{$json.curriculum_request || 'Curriculum Plan'}}`
    - `plan_content` = `{{$json.output}}`
    - `created_date` = `{{$now.toISO()}}`
    - `agent_used` = `Multi-Agent System`
    - `status` = `Generated`
  - Auto-generated metadata makes the stored record easier to track.
- **Key expressions or variables used:**
  - `$json.curriculum_request`
  - `$json.output`
  - `$now.toISO()`
- **Input and output connections:**
  - Main input from **Curriculum Supervisor Agent**
  - Main output to **Store Curriculum Plan**
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**
  - If the supervisor output does not expose content in `$json.output`, `plan_content` may be empty.
  - If no request field exists upstream, title falls back to `Curriculum Plan`.
  - Large outputs may exceed storage/display expectations downstream.
- **Sub-workflow reference:** None.

#### Store Curriculum Plan
- **Type and technical role:** `n8n-nodes-base.dataTable`; persists the prepared curriculum record in an n8n Data Table.
- **Configuration choices:**
  - Data table ID is set to a placeholder:
    - `<__PLACEHOLDER_VALUE__curriculum_plans table__>`
  - Column mapping mode: `autoMapInputData`
- **Key expressions or variables used:** None.
- **Input and output connections:** Main input from **Prepare Curriculum Data**; no downstream node.
- **Version-specific requirements:** Type version 1.1; requires a real Data Table to exist and be selected.
- **Edge cases or potential failure types:**
  - As configured, this node is incomplete because the Data Table ID is still a placeholder.
  - Storage will fail until a real table is selected.
  - Auto-mapping may fail or behave unexpectedly if the table schema does not match the incoming fields.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Curriculum Planning | n8n-nodes-base.manualTrigger | Manual workflow entry point |  | Curriculum Supervisor Agent | ## Curriculum Supervisor Agent<br>**What:** Orchestrates sub-agents using shared planning memory.<br>**Why:** Ensures coherent task delegation and context retention. |
| Curriculum Supervisor Agent | @n8n/n8n-nodes-langchain.agent | Main orchestrator that delegates to specialist agents and synthesizes final plan | Start Curriculum Planning; Supervisor Model; Planning Memory; Research Agent; Content Creation Agent; Assessment Agent; Google Sheets Tool | Prepare Curriculum Data | ## Curriculum Supervisor Agent<br>**What:** Orchestrates sub-agents using shared planning memory.<br>**Why:** Ensures coherent task delegation and context retention. |
| Supervisor Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for supervisor reasoning and synthesis |  | Curriculum Supervisor Agent | ## Research Agent<br>**What:** Queries Google Search, Perplexity, and Wikipedia.<br>**Why:** Grounds curriculum content in current, verified information. |
| Planning Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Short-term memory for supervisor context retention |  | Curriculum Supervisor Agent | ## Research Agent<br>**What:** Queries Google Search, Perplexity, and Wikipedia.<br>**Why:** Grounds curriculum content in current, verified information. |
| Research Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist research agent for standards, pedagogy, and subject content | Research Model; Google Search Tool; Perplexity Research Tool; Wikipedia Tool | Curriculum Supervisor Agent | ## Research Agent<br>**What:** Queries Google Search, Perplexity, and Wikipedia.<br>**Why:** Grounds curriculum content in current, verified information. |
| Research Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model used by research agent |  | Research Agent | ## Research Agent<br>**What:** Queries Google Search, Perplexity, and Wikipedia.<br>**Why:** Grounds curriculum content in current, verified information. |
| Google Search Tool | @n8n/n8n-nodes-langchain.toolSerpApi | Search tool for web-based research |  | Research Agent | ## Research Agent<br>**What:** Queries Google Search, Perplexity, and Wikipedia.<br>**Why:** Grounds curriculum content in current, verified information. |
| Perplexity Research Tool | n8n-nodes-base.perplexityTool | Citation-backed AI research tool |  | Research Agent | ## Research Agent<br>**What:** Queries Google Search, Perplexity, and Wikipedia.<br>**Why:** Grounds curriculum content in current, verified information. |
| Wikipedia Tool | @n8n/n8n-nodes-langchain.toolWikipedia | Encyclopedia lookup for background knowledge |  | Research Agent | ## Research Agent<br>**What:** Queries Google Search, Perplexity, and Wikipedia.<br>**Why:** Grounds curriculum content in current, verified information. |
| Content Creation Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for lesson plans and instructional design | Content Model; Calculator Tool; Code Tool | Curriculum Supervisor Agent | ## Content Creation Agent<br>**What:** Drafts curriculum using GPT-based content model.<br>**Why:** Produces structured, pedagogically sound learning materials. |
| Content Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for content generation |  | Content Creation Agent | ## Content Creation Agent<br>**What:** Drafts curriculum using GPT-based content model.<br>**Why:** Produces structured, pedagogically sound learning materials. |
| Assessment Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for rubrics, evaluation, and assessment design | Assessment Model; Calculator Tool; Code Tool | Curriculum Supervisor Agent | ## Assessment Agent<br>**What:** Applies calculator and code tools for evaluation design.<br>**Why:** Ensures assessments align with learning objectives. |
| Assessment Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for assessment logic and rubric generation |  | Assessment Agent | ## Assessment Agent<br>**What:** Applies calculator and code tools for evaluation design.<br>**Why:** Ensures assessments align with learning objectives. |
| Calculator Tool | @n8n/n8n-nodes-langchain.toolCalculator | Shared arithmetic helper for content timing and scoring logic |  | Content Creation Agent; Assessment Agent | ## Content Creation Agent<br>**What:** Drafts curriculum using GPT-based content model.<br>**Why:** Produces structured, pedagogically sound learning materials.<br>## Assessment Agent<br>**What:** Applies calculator and code tools for evaluation design.<br>**Why:** Ensures assessments align with learning objectives. |
| Code Tool | @n8n/n8n-nodes-langchain.toolCode | Shared code execution helper for structure generation and calculations |  | Content Creation Agent; Assessment Agent | ## Assessment Agent<br>**What:** Applies calculator and code tools for evaluation design.<br>**Why:** Ensures assessments align with learning objectives. |
| Google Sheets Tool | n8n-nodes-base.googleSheetsTool | AI-callable sheet storage/retrieval tool for supervisor |  | Curriculum Supervisor Agent | ## Assessment Agent<br>**What:** Applies calculator and code tools for evaluation design.<br>**Why:** Ensures assessments align with learning objectives. |
| Prepare Curriculum Data | n8n-nodes-base.set | Formats supervisor output into structured storage fields | Curriculum Supervisor Agent | Store Curriculum Plan | ## Prepare & Store<br>**What:** Formats and writes output to Google Sheets.<br>**Why:** Centralises curriculum data for easy access and review. |
| Store Curriculum Plan | n8n-nodes-base.dataTable | Stores final curriculum plan in n8n Data Table | Prepare Curriculum Data |  | ## Prepare & Store<br>**What:** Formats and writes output to Google Sheets.<br>**Why:** Centralises curriculum data for easy access and review. |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation note |  |  | ## How It Works<br>This workflow automates end-to-end curriculum planning using a multi-agent AI architecture in n8n. Designed for educators, instructional designers, and academic institutions, it eliminates the manual effort of researching, structuring, and assessing curriculum content. A Curriculum Supervisor Agent orchestrates three specialised sub-agents: a Research Agent (using Google Search, Perplexity, and Wikipedia), a Content Creation Agent (using GPT for drafting), and an Assessment Agent (using a calculator and code tools). Planning memory persists context across agent interactions. Once all agents complete their tasks, the Prepare Curriculum Data node formats the output, which is then stored in Google Sheets. This pipeline ensures coherent, research-backed, assessment-ready curriculum plans are generated and stored automatically with minimal human intervention. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace setup instructions |  |  | ## Setup Steps<br>1. Connect OpenAI credentials to Supervisor, Research, Content, and Assessment model nodes.<br>2. Add Google Custom Search API key to the Google Search Tool node.<br>3. Configure Perplexity API key in the Perplexity Research Tool node.<br>4. Authenticate Google Sheets with OAuth2; set target sheet in Store Curriculum Plan node.<br>5. Verify Planning Memory node is linked to the Supervisor Agent for context persistence. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace prerequisites and use-case note |  |  | ## Prerequisites<br>- Google Custom Search API key<br>- Perplexity API key<br>- Google Sheets OAuth2 credentials<br>## Use Cases<br>- University course redesign aligned to industry trends<br>## Customisation<br>- Add email output via Gmail node post-storage<br>## Benefits<br>- Cuts curriculum planning time by 70%+ |
| Sticky Note3 | n8n-nodes-base.stickyNote | Workspace note for content block |  |  | ## Content Creation Agent<br>**What:** Drafts curriculum using GPT-based content model.<br>**Why:** Produces structured, pedagogically sound learning materials. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Workspace note for research block |  |  | ## Research Agent<br>**What:** Queries Google Search, Perplexity, and Wikipedia.<br>**Why:** Grounds curriculum content in current, verified information. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Workspace note for supervisor block |  |  | ## Curriculum Supervisor Agent<br>**What:** Orchestrates sub-agents using shared planning memory.<br>**Why:** Ensures coherent task delegation and context retention. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Workspace note for assessment block |  |  | ## Assessment Agent<br>**What:** Applies calculator and code tools for evaluation design.<br>**Why:** Ensures assessments align with learning objectives. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Workspace note for storage block |  |  | ## Prepare & Store<br>**What:** Formats and writes output to Google Sheets.<br>**Why:** Centralises curriculum data for easy access and review. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **AI multi-agent curriculum planning with research and assessment**.
   - Keep execution order at the default unless you specifically need legacy behavior.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Rename it to **Start Curriculum Planning**.
   - This will be the main entry point.

3. **Add the main supervisor agent**
   - Node type: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Rename it to **Curriculum Supervisor Agent**.
   - In the main text/input field, set:
     - `{{$json.curriculum_request || 'Create a comprehensive curriculum plan for a high school biology course covering cell biology, genetics, and evolution.'}}`
   - In the system message, define the supervisor role:
     - it coordinates Research, Content Creation, and Assessment agents
     - it breaks curriculum requests into tasks
     - it synthesizes their outputs into one coherent curriculum plan

4. **Connect the trigger to the supervisor**
   - Main connection:
     - **Start Curriculum Planning** → **Curriculum Supervisor Agent**

5. **Add the supervisor model**
   - Node type: **OpenAI Chat Model**
   - Rename it to **Supervisor Model**.
   - Select model: **gpt-4o**
   - Set temperature to **0.3**
   - Connect it to the supervisor using the **AI language model** connection.

6. **Configure OpenAI credentials for the supervisor model**
   - Create or select an **OpenAI API credential**.
   - Ensure the account has access to `gpt-4o`.

7. **Add memory for the supervisor**
   - Node type: **Memory Buffer Window**
   - Rename it to **Planning Memory**.
   - Set **Context Window Length** to **10**.
   - Connect it to the supervisor using the **AI memory** connection.

8. **Add the research specialist agent**
   - Node type: **AI Agent Tool** (`agentTool`)
   - Rename it to **Research Agent**.
   - In the task text field, set:
     - `{{$fromAI('research_task', 'The specific research task or question to investigate')}}`
   - Add a system message describing it as a specialist in:
     - educational standards
     - curriculum frameworks
     - pedagogical approaches
     - subject content
   - Add a tool description explaining that the agent is used for foundational educational research and best practices.

9. **Add the research model**
   - Node type: **OpenAI Chat Model**
   - Rename it to **Research Model**.
   - Model: **gpt-4o**
   - Temperature: **0.2**
   - Connect it to **Research Agent** with an **AI language model** connection.
   - Reuse the same OpenAI credential or create another if needed.

10. **Add the Google Search research tool**
    - Node type: **SerpAPI Tool**
    - Rename it to **Google Search Tool**.
    - Leave default options unless you need search tuning.
    - Connect it to **Research Agent** using the **AI tool** connection.
    - Configure the required API credential or service settings for SerpAPI / Google Custom Search depending on your deployment and node version.

11. **Add the Perplexity research tool**
    - Node type: **Perplexity Tool**
    - Rename it to **Perplexity Research Tool**.
    - Enable manual description and set:
      - “Conduct AI-powered research with citations on educational topics, learning theories, and subject matter content”
    - Leave request/message defaults unless you need a custom research style.
    - Connect it to **Research Agent** with an **AI tool** connection.
    - Configure **Perplexity API credentials**.

12. **Add the Wikipedia tool**
    - Node type: **Wikipedia Tool**
    - Rename it to **Wikipedia Tool**.
    - Use default settings.
    - Connect it to **Research Agent** via **AI tool**.

13. **Expose the research agent to the supervisor**
    - Connect **Research Agent** to **Curriculum Supervisor Agent** using an **AI tool** connection.
    - This makes the research specialist callable by the supervisor.

14. **Add the content specialist agent**
    - Node type: **AI Agent Tool**
    - Rename it to **Content Creation Agent**.
    - Set task text to:
      - `{{$fromAI('content_task', 'The specific content creation task or lesson plan to develop')}}`
    - Add a system message describing it as an instructional design specialist for:
      - lesson plans
      - activities
      - sequencing
      - age-appropriate learning materials
    - Add a tool description explaining its use for pedagogically sound curriculum drafting.

15. **Add the content model**
    - Node type: **OpenAI Chat Model**
    - Rename it to **Content Model**.
    - Model: **gpt-4o**
    - Temperature: **0.5**
    - Connect to **Content Creation Agent** as **AI language model**.

16. **Add the calculator tool**
    - Node type: **Calculator Tool**
    - Rename it to **Calculator Tool**.
    - Keep default settings.
    - Connect it to **Content Creation Agent** as **AI tool**.

17. **Add the code tool**
    - Node type: **Code Tool**
    - Rename it to **Code Tool**.
    - Set its description to:
      - “Execute custom code for complex curriculum structure generation, data transformations, and specialized calculations”
    - Connect it to **Content Creation Agent** as **AI tool**.

18. **Expose the content agent to the supervisor**
    - Connect **Content Creation Agent** to **Curriculum Supervisor Agent** as **AI tool**.

19. **Add the assessment specialist agent**
    - Node type: **AI Agent Tool**
    - Rename it to **Assessment Agent**.
    - Set task text to:
      - `{{$fromAI('assessment_task', 'The specific assessment design task or evaluation framework to create')}}`
    - Add a system message describing it as a specialist in:
      - rubrics
      - formative/summative assessments
      - grading criteria
      - evaluation strategies aligned to outcomes
    - Add a tool description explaining that it designs comprehensive assessment frameworks.

20. **Add the assessment model**
    - Node type: **OpenAI Chat Model**
    - Rename it to **Assessment Model**.
    - Model: **gpt-4o**
    - Temperature: **0.2**
    - Connect to **Assessment Agent** as **AI language model**.

21. **Share the calculator tool with the assessment agent**
    - Connect **Calculator Tool** to **Assessment Agent** as **AI tool**.

22. **Share the code tool with the assessment agent**
    - Connect **Code Tool** to **Assessment Agent** as **AI tool**.

23. **Expose the assessment agent to the supervisor**
    - Connect **Assessment Agent** to **Curriculum Supervisor Agent** as **AI tool**.

24. **Add the Google Sheets AI tool**
    - Node type: **Google Sheets Tool**
    - Rename it to **Google Sheets Tool**.
    - Operation: **Append or Update**
    - Set a manual tool description such as:
      - “Store and retrieve curriculum data, lesson plans, and assessment frameworks in Google Sheets”
    - Select a target **Google spreadsheet document ID**
    - Select a **sheet name**
    - Connect it to **Curriculum Supervisor Agent** as **AI tool**
    - Configure **Google Sheets OAuth2 credentials**

25. **Important note about Google Sheets configuration**
    - In the provided workflow JSON, `documentId` and `sheetName` are blank.
    - To make the workflow usable, you must choose a real spreadsheet and worksheet.
    - If using append/update, confirm the sheet structure supports the fields the AI may write.

26. **Add a Set node for final output formatting**
    - Node type: **Set**
    - Rename it to **Prepare Curriculum Data**
    - Add these fields:
      1. `curriculum_title` as string:
         - `{{$json.curriculum_request || 'Curriculum Plan'}}`
      2. `plan_content` as string:
         - `{{$json.output}}`
      3. `created_date` as string:
         - `{{$now.toISO()}}`
      4. `agent_used` as string:
         - `Multi-Agent System`
      5. `status` as string:
         - `Generated`

27. **Connect supervisor output to the Set node**
    - Main connection:
      - **Curriculum Supervisor Agent** → **Prepare Curriculum Data**

28. **Add the final storage node**
    - Node type: **Data Table**
    - Rename it to **Store Curriculum Plan**
    - Set mapping mode to **Auto-map input data**
    - Select or create an n8n Data Table intended to store curriculum plans
    - Choose the table in the **Data Table ID** field

29. **Connect the Set node to the Data Table**
    - Main connection:
      - **Prepare Curriculum Data** → **Store Curriculum Plan**

30. **Create the Data Table schema**
    - Ensure your n8n Data Table contains columns compatible with:
      - `curriculum_title`
      - `plan_content`
      - `created_date`
      - `agent_used`
      - `status`
    - Auto-mapping works best when names match exactly.

31. **Test with default input**
    - Run the workflow manually without input data.
    - The supervisor should use the fallback biology curriculum request.

32. **Test with custom input**
    - Supply an incoming JSON object with:
      - `curriculum_request`
    - Example:
      - “Create a semester-long curriculum for an introductory data science course for undergraduates.”
    - Confirm that the final Data Table row contains the generated plan.

33. **Validate tool paths**
    - Confirm the supervisor can call:
      - Research Agent
      - Content Creation Agent
      - Assessment Agent
      - Google Sheets Tool
    - Confirm each specialist has its own model and tool connections.

34. **Review incomplete areas from the original workflow**
    - The original JSON suggests Google Sheets is part of the process, but the actually wired final storage is **Data Table**, not Google Sheets.
    - The sticky note says output is stored in Google Sheets, but the end-of-flow persistence node is **Store Curriculum Plan** using **Data Table**.
    - If you want real Google Sheets final storage, add a regular Google Sheets node after **Prepare Curriculum Data**, or replace the Data Table node.

35. **Optional improvement**
    - Add an input node before the supervisor, such as a Form Trigger, Webhook, or Set node, if you want to pass `curriculum_request` more explicitly rather than relying on fallback text.
    - Add error handling branches for API failures.
    - Add output delivery via Gmail, Slack, or Notion.

### Credential Setup Summary
- **OpenAI**
  - Required for:
    - Supervisor Model
    - Research Model
    - Content Model
    - Assessment Model
- **Perplexity API**
  - Required for:
    - Perplexity Research Tool
- **SerpAPI / Search API**
  - Required for:
    - Google Search Tool
- **Google Sheets OAuth2**
  - Required for:
    - Google Sheets Tool
- **n8n Data Table**
  - Required for:
    - Store Curriculum Plan

### Sub-workflow Setup
This workflow contains **no Execute Workflow node** and **no called sub-workflow**. The specialist agents are internal AI tool agents, not separate n8n workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates end-to-end curriculum planning using a multi-agent AI architecture in n8n. Designed for educators, instructional designers, and academic institutions, it reduces manual effort across research, curriculum design, and assessment generation. | Workspace note |
| Setup steps noted in the workflow: connect OpenAI credentials to all model nodes; add search API credentials to Google Search Tool; configure Perplexity API; authenticate Google Sheets with OAuth2; verify Planning Memory is linked to the Supervisor Agent. | Workspace note |
| Prerequisites listed in the workflow: Google Custom Search API key, Perplexity API key, Google Sheets OAuth2 credentials. | Workspace note |
| Example use case listed in the workflow: university course redesign aligned to industry trends. | Workspace note |
| Suggested customization listed in the workflow: add email output via Gmail after storage. | Workspace note |
| Claimed benefit listed in the workflow: cuts curriculum planning time by 70%+. | Workspace note |
| Important inconsistency: the notes say the final output is stored in Google Sheets, but the actual final connected storage node is an n8n Data Table. The Google Sheets component is currently only available as an AI tool to the supervisor, and its spreadsheet fields are not configured. | Implementation note |
| Important incomplete configuration: `Store Curriculum Plan` uses a placeholder Data Table ID and will not run successfully until a real table is selected. | Implementation note |
| Important incomplete configuration: `Google Sheets Tool` has blank `documentId` and `sheetName`, so it cannot be reliably used until configured. | Implementation note |