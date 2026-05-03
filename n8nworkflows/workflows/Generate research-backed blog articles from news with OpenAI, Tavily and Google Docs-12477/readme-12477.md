Generate research-backed blog articles from news with OpenAI, Tavily and Google Docs

https://n8nworkflows.xyz/workflows/generate-research-backed-blog-articles-from-news-with-openai--tavily-and-google-docs-12477


# Generate research-backed blog articles from news with OpenAI, Tavily and Google Docs

## 1. Workflow Overview

**Purpose:** Generate a research-backed blog article starting from “news” input, using Tavily for extraction/research and OpenAI (LangChain nodes) for outlining, section drafting, editing, and metadata generation, then publish the result into **Google Docs**.

**Typical use cases:**
- Turning a news link/topic into a full blog post with citations or research support
- Automating content pipelines: outline → sections → merge → edit → title/slug/description → Google Doc

### 1.1 Entry & News Extraction
Manual trigger starts the workflow; Tavily extracts relevant information from the provided news source(s), then code prepares/normalizes items for AI.

### 1.2 Planning (Outline / Table of Contents)
An AI Agent produces a Table of Contents using an OpenAI chat model and a Tavily “tool workflow” for research.

### 1.3 Section Generation (Per-section Drafting)
The outline is converted into discrete “sections”, split into individual items, and an AI Agent generates content for each section (with Tavily tool available).

### 1.4 Merge, Aggregate & Editing
Generated sections are merged back, aggregated into a single article body, then a “Content Editor” agent refines the full draft.

### 1.5 Metadata (Title / Slug / Description)
OpenAI nodes generate the final article title, a URL slug, and an article description.

### 1.6 Google Docs Output + Loop Control
A Google Doc is created and updated, then the workflow loops via SplitInBatches until completion.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Entry & Source Extraction
**Overview:** Starts execution manually, extracts source content via Tavily, then prepares it for the downstream AI agent that sets direction/topic context.

**Nodes involved:**
- **When clicking ‘Execute workflow’** (Manual Trigger)
- **Extract** (Tavily)
- **Code1** (Code)
- **AI Agent** (LangChain Agent)
- **OpenAI Chat Model1** (OpenAI Chat LLM)

#### Node: When clicking ‘Execute workflow’
- **Type/role:** `manualTrigger` — manual entry point.
- **Config (interpreted):** No parameters shown; user runs workflow from n8n UI.
- **Outputs:** Sends one empty старт item to **Extract**.
- **Failure modes:** None (except workflow not executed).

#### Node: Extract
- **Type/role:** `@tavily/n8n-nodes-tavily.tavily` — Tavily extraction/search node (named “Extract”).
- **Config (interpreted):** Parameters not provided in JSON export; typically requires Tavily API key and an input query/URL.
- **Input:** From Manual Trigger.
- **Output:** Extracted text/structured data used by **Code1**.
- **Failure modes / edge cases:**
  - Missing/invalid Tavily credentials
  - URL unreachable / rate limits / timeouts
  - Empty extraction results → downstream prompt quality degrades

#### Node: Code1
- **Type/role:** `code` — transforms Tavily output into a normalized structure for AI.
- **Config (interpreted):** No visible code in JSON; expected to:
  - pick key fields (headline, summary, entities, source URL)
  - format a “brief” or “topic packet” for the agent
- **Input:** Extract output.
- **Output:** Feeds **AI Agent**.
- **Failure modes:**
  - JS errors (undefined fields if Tavily response shape differs)
  - Producing empty/invalid JSON structure for agent prompts

#### Node: AI Agent
- **Type/role:** `@n8n/n8n-nodes-langchain.agent` — agent to interpret extracted news and set initial framing (topic, angle, audience, constraints).
- **Config (interpreted):**
  - Uses **OpenAI Chat Model1** via `ai_languageModel` connection.
  - No tools connected here (only LLM).
- **Input:** From Code1.
- **Output:** Sent to **Code** node for further shaping and batching.
- **Failure modes:**
  - OpenAI auth/model errors
  - Prompt/response mismatch with Code expectations

#### Node: OpenAI Chat Model1
- **Type/role:** `lmChatOpenAi` — provides chat completion model for AI Agent.
- **Config (interpreted):** Credentials + model selection required in n8n; unspecified here.
- **Connections:** `ai_languageModel → AI Agent`
- **Failure modes:** Invalid API key, model not available, rate limits.

---

### Block 2.2 — Item Preparation & Batch Loop Setup
**Overview:** Converts the agent output into a list/batches used to drive the outline and later loop behavior.

**Nodes involved:**
- **Code**
- **Loop Over Items** (SplitInBatches)

#### Node: Code
- **Type/role:** `code` — shapes the AI Agent output into items suitable for batching.
- **Input:** From **AI Agent**.
- **Output:** To **Loop Over Items**.
- **Failure modes:** Same as Code1; common issue is expecting an array but receiving a string/object.

#### Node: Loop Over Items
- **Type/role:** `splitInBatches` — controls iteration/batch execution.
- **Config (interpreted):** Batch size not shown; default commonly 1.
- **Connections:**
  - **Input 0:** from Code
  - **Output (index 1):** to **Table of Contents** (this is the “next batch” path in n8n SplitInBatches)
  - Receives a loop-back from **Update a document** into its input, indicating iterative processing.
- **Edge cases / failure modes:**
  - If batch size misconfigured: too many parallel LLM calls or too slow
  - If loop-back logic never reaches completion: infinite loop risk
  - If incoming items empty: Table of Contents never runs

---

### Block 2.3 — Outline Generation (Table of Contents)
**Overview:** Builds an article outline/TOC using an agent that can call a Tavily tool for research.

**Nodes involved:**
- **Table of Contents** (Agent)
- **OpenAI Chat Model** (LLM)
- **tavily tool** (ToolWorkflow)

#### Node: Table of Contents
- **Type/role:** `@n8n/n8n-nodes-langchain.agent` — generates structured TOC.
- **Config (interpreted):**
  - Uses **OpenAI Chat Model** as `ai_languageModel`.
  - Has **tavily tool** attached as an agent tool (`ai_tool`).
- **Input:** From **Loop Over Items** (batch output index 1).
- **Output:** To **Create the Sections**.
- **Edge cases:**
  - Agent returns TOC in unexpected format (bullets vs JSON) → section creation may break
  - Tool calling disabled/misconfigured → weaker research grounding

#### Node: OpenAI Chat Model
- **Type/role:** `lmChatOpenAi` — LLM for TOC agent.
- **Connections:** `ai_languageModel → Table of Contents`
- **Failure modes:** Auth/rate limits/model mismatch.

#### Node: tavily tool
- **Type/role:** `toolWorkflow` — a LangChain tool that calls another workflow (sub-workflow) to perform Tavily search/research.
- **Config (interpreted):** Parameters not shown; typically needs:
  - Referenced workflow ID/name
  - Input schema (query) and output schema (search results)
- **Connections:** `ai_tool → Table of Contents`
- **Failure modes / edge cases:**
  - Referenced sub-workflow missing/unpublished
  - Tool returns huge text → token overflow in agent context
  - Tool schema mismatch causing agent tool-call errors
- **Sub-workflow reference:** **Yes** (ToolWorkflow implies invoking another workflow). The referenced workflow details are not included in the JSON.

---

### Block 2.4 — Section List Creation & Split
**Overview:** Converts the TOC into a list of sections, then splits them into single items for per-section drafting.

**Nodes involved:**
- **Create the Sections** (OpenAI)
- **Split Out1** (SplitOut)

#### Node: Create the Sections
- **Type/role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI generation step (non-agent) to transform TOC into section prompts/objects.
- **Input:** From Table of Contents.
- **Output:** To Split Out1.
- **Config (interpreted):**
  - Likely prompts the model to return an array of sections (title, key points, sources needed).
- **Failure modes:**
  - Output not parseable as array → Split Out fails
  - Token limits if TOC too large

#### Node: Split Out1
- **Type/role:** `splitOut` — splits an array field into separate items.
- **Input:** From Create the Sections.
- **Outputs:**
  - **Main index 0 → Generate the Content**
  - **Main index 1 → Merge** (used as the second input stream of Merge)
- **Edge cases:**
  - If no array present, node outputs nothing
  - If array elements are not uniform, downstream prompts vary in quality

---

### Block 2.5 — Per-Section Content Drafting + Merge
**Overview:** For each section item, an agent drafts the content with the ability to call Tavily research, then merges drafted content with the parallel stream.

**Nodes involved:**
- **Generate the Content** (Agent)
- **OpenAI Chat Model2** (LLM)
- **tavily tool1** (ToolWorkflow)
- **Merge** (Merge)

#### Node: Generate the Content
- **Type/role:** LangChain Agent — generates section text.
- **Config (interpreted):**
  - Uses **OpenAI Chat Model2**
  - Uses **tavily tool1** for research/tool calling
- **Input:** From Split Out1 main output 0.
- **Output:** To Merge input 0.
- **Failure modes:**
  - Tool calling errors/schema mismatch
  - Inconsistent formatting across sections (headings, citations)
  - Long sections → token limits

#### Node: OpenAI Chat Model2
- **Type/role:** `lmChatOpenAi` — drafting model.
- **Connections:** `ai_languageModel → Generate the Content`

#### Node: tavily tool1
- **Type/role:** `toolWorkflow` — sub-workflow tool for research.
- **Connections:** `ai_tool → Generate the Content`
- **Sub-workflow reference:** Yes (not included in JSON).

#### Node: Merge
- **Type/role:** `merge` — combines two inputs.
- **Config (interpreted):** TypeVersion 3; operation not visible. Given wiring:
  - Input 0: generated section content
  - Input 1: passthrough items from Split Out1
  This often indicates “merge by position” or “combine” to keep metadata + generated text together.
- **Output:** To Aggregate1.
- **Failure modes:**
  - Mismatched item counts/order → wrong content associated with wrong section
  - If one branch is empty → merge outputs empty or partial depending on mode

---

### Block 2.6 — Aggregate to Full Draft + Edit
**Overview:** Aggregates all sections into a single article body and performs editing/polishing via an agent.

**Nodes involved:**
- **Aggregate1**
- **Content Editor** (Agent)
- **OpenAI** (LLM for editor)

#### Node: Aggregate1
- **Type/role:** `aggregate` — combines many items into one (typically concatenation).
- **Input:** From Merge.
- **Output:** To Content Editor.
- **Config (interpreted):** Not shown; commonly aggregates a field like `content` into one large string with separators.
- **Failure modes:**
  - Wrong field aggregated → empty article
  - Ordering issues if items not stable

#### Node: Content Editor
- **Type/role:** LangChain Agent — refines the complete draft (consistency, tone, transitions, factual cautions).
- **Input:** From Aggregate1.
- **LLM:** Uses **OpenAI** node as `ai_languageModel`.
- **Output:** To Title Maker.
- **Failure modes:**
  - Over-editing that removes citations/attribution
  - Token overflow if aggregated text too large

#### Node: OpenAI
- **Type/role:** `lmChatOpenAi` — editing model.
- **Connections:** `ai_languageModel → Content Editor`

---

### Block 2.7 — Metadata Generation (Title, Slug, Description)
**Overview:** Produces publishing metadata after final text is edited.

**Nodes involved:**
- **Title Maker** (OpenAI)
- **Making Slug** (OpenAI)
- **Article Description** (OpenAI)

#### Node: Title Maker
- **Type/role:** OpenAI generation — produces an article title.
- **Input:** From Content Editor.
- **Output:** To Making Slug.
- **Failure modes:** Title too long, not aligned with SEO, or missing.

#### Node: Making Slug
- **Type/role:** OpenAI generation — converts title into URL-friendly slug.
- **Input:** From Title Maker.
- **Output:** To Article Description.
- **Edge cases:** Non-ASCII characters, duplicate slugs, whitespace/punctuation handling.

#### Node: Article Description
- **Type/role:** OpenAI generation — creates meta description / summary.
- **Input:** From Making Slug.
- **Output:** To Create a document1.
- **Failure modes:** Length constraints (e.g., 155–160 chars) not enforced unless prompted.

---

### Block 2.8 — Google Docs Creation, Update, and Loop-Back
**Overview:** Creates a Google Doc, writes/updates the generated article, then loops back into batching control.

**Nodes involved:**
- **Create a document1** (Google Docs)
- **Update a document** (Google Docs)

#### Node: Create a document1
- **Type/role:** `googleDocs` — creates a new Google document.
- **Input:** From Article Description (which implies final payload contains title/body/slug/description).
- **Output:** To Update a document.
- **Config (interpreted):**
  - Requires Google OAuth2 credentials.
  - Likely sets document title to the generated title and initializes content.
- **Failure modes:**
  - OAuth not authorized / insufficient scopes
  - Drive permission errors
  - Wrong folder ID

#### Node: Update a document
- **Type/role:** `googleDocs` — appends/inserts formatted content into the created doc.
- **Input:** From Create a document1.
- **Output:** Loops back to **Loop Over Items** input.
- **Edge cases:**
  - Document ID not passed correctly from creation step
  - Large content → API limits/timeouts
  - Formatting/Markdown not rendered as expected unless explicitly handled

---

### Block 2.9 — Sticky Notes (No content)
**Overview:** The workflow includes several sticky notes, but all are empty in the provided JSON.

**Nodes involved:** Sticky Note, Sticky Note1, Sticky Note4, Sticky Note5, Sticky Note6  
- **Impact:** None on execution.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note1 | Sticky Note | Comment (empty) |  |  |  |
| Sticky Note | Sticky Note | Comment (empty) |  |  |  |
| Sticky Note5 | Sticky Note | Comment (empty) |  |  |  |
| When clicking ‘Execute workflow’ | Manual Trigger | Workflow entry point |  | Extract |  |
| Extract | Tavily | Extract/retrieve source content | When clicking ‘Execute workflow’ | Code1 |  |
| Code1 | Code | Normalize Tavily output for agent | Extract | AI Agent |  |
| AI Agent | LangChain Agent | Interpret extracted news & set direction | Code1 | Code |  |
| OpenAI Chat Model1 | OpenAI Chat Model | LLM backend for AI Agent |  | AI Agent (ai_languageModel) |  |
| Code | Code | Prepare items for batching | AI Agent | Loop Over Items |  |
| Loop Over Items | Split In Batches | Iteration control / batching | Code; Update a document | Table of Contents (batch output) |  |
| Table of Contents | LangChain Agent | Generate outline/TOC (tool-enabled) | Loop Over Items | Create the Sections |  |
| OpenAI Chat Model | OpenAI Chat Model | LLM backend for TOC agent |  | Table of Contents (ai_languageModel) |  |
| tavily tool | Tool Workflow | Research tool callable by TOC agent |  | Table of Contents (ai_tool) |  |
| Create the Sections | OpenAI | Convert TOC into structured sections list | Table of Contents | Split Out1 |  |
| Split Out1 | Split Out | Split sections array into items | Create the Sections | Generate the Content; Merge |  |
| Generate the Content | LangChain Agent | Draft each section (tool-enabled) | Split Out1 | Merge |  |
| OpenAI Chat Model2 | OpenAI Chat Model | LLM backend for section drafting |  | Generate the Content (ai_languageModel) |  |
| tavily tool1 | Tool Workflow | Research tool callable by section agent |  | Generate the Content (ai_tool) |  |
| Merge | Merge | Combine section metadata + generated text | Generate the Content; Split Out1 | Aggregate1 |  |
| Aggregate1 | Aggregate | Combine all sections into full draft | Merge | Content Editor |  |
| Content Editor | LangChain Agent | Edit/polish full article | Aggregate1 | Title Maker |  |
| OpenAI | OpenAI Chat Model | LLM backend for editor agent |  | Content Editor (ai_languageModel) |  |
| Sticky Note6 | Sticky Note | Comment (empty) |  |  |  |
| Title Maker | OpenAI | Generate final article title | Content Editor | Making Slug |  |
| Making Slug | OpenAI | Generate URL slug | Title Maker | Article Description |  |
| Article Description | OpenAI | Generate meta description/summary | Making Slug | Create a document1 |  |
| Sticky Note4 | Sticky Note | Comment (empty) |  |  |  |
| Create a document1 | Google Docs | Create Google Doc | Article Description | Update a document |  |
| Update a document | Google Docs | Write/update document content | Create a document1 | Loop Over Items |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add **Manual Trigger** node named **“When clicking ‘Execute workflow’”**.

2. **Add Tavily extraction**
   1) Add **Tavily** node (package `@tavily/n8n-nodes-tavily`) named **“Extract”**.  
   2) Configure Tavily **API credentials**.  
   3) Set it to **extract/search** from your intended input (commonly a URL, query, or both).  
   4) Connect: **Manual Trigger → Extract**.

3. **Normalize extracted data**
   1) Add **Code** node named **“Code1”**.  
   2) Write JS to map Tavily’s output into fields your prompts will use (example targets): `topic`, `sourceUrl`, `keyFacts`, `rawText`.  
   3) Connect: **Extract → Code1**.

4. **Initial framing agent**
   1) Add **OpenAI Chat Model** node named **“OpenAI Chat Model1”** and set:
      - OpenAI credentials (API key)
      - Model (e.g., GPT-4-class / GPT-4.1-mini depending on your account)
   2) Add **AI Agent** node named **“AI Agent”**.
   3) In **AI Agent**, select **OpenAI Chat Model1** as its **Language Model** (connect via `ai_languageModel`).
   4) Prompt the agent to: define angle, audience, key claims, constraints, and what to research.
   5) Connect: **Code1 → AI Agent**.

5. **Prepare items for batching**
   1) Add **Code** node named **“Code”** that outputs an array of items to iterate over (often 1 item if you generate one article at a time, or multiple if multiple topics/URLs).
   2) Connect: **AI Agent → Code**.

6. **Add Split In Batches loop controller**
   1) Add **Split In Batches** node named **“Loop Over Items”**.
   2) Set **Batch Size** (commonly `1`).
   3) Connect: **Code → Loop Over Items**.
   4) Use the node’s **“Next Batch”** output to continue the workflow (as in the provided wiring).

7. **TOC agent with Tavily tool**
   1) Add **OpenAI Chat Model** node named **“OpenAI Chat Model”** (credentials + model).
   2) Add **Tool Workflow** node named **“tavily tool”**:
      - Configure it to call a **separate n8n workflow** that performs Tavily search and returns results in a concise schema (e.g., `sources[]` with `title,url,snippet`).
   3) Add **LangChain Agent** node named **“Table of Contents”**:
      - Connect **OpenAI Chat Model → Table of Contents** via `ai_languageModel`
      - Connect **tavily tool → Table of Contents** via `ai_tool`
      - Prompt: produce a structured TOC (preferably JSON with section titles + goals + required sources).
   4) Connect: **Loop Over Items (Next Batch output) → Table of Contents**.

8. **Convert TOC to sections list**
   1) Add **OpenAI** node named **“Create the Sections”**:
      - Prompt it to transform the TOC into an **array** of section objects (e.g., `{heading, intent, bulletPoints, researchQueries}`).
   2) Connect: **Table of Contents → Create the Sections**.

9. **Split into per-section items**
   1) Add **Split Out** node named **“Split Out1”**.
   2) Configure it to split the sections array field produced by “Create the Sections”.
   3) Connect: **Create the Sections → Split Out1**.

10. **Draft each section with agent + tool**
   1) Add **OpenAI Chat Model** node named **“OpenAI Chat Model2”** (credentials + model).
   2) Add **Tool Workflow** node named **“tavily tool1”** (can reuse same sub-workflow as “tavily tool”).
   3) Add **LangChain Agent** node named **“Generate the Content”**:
      - Connect **OpenAI Chat Model2 → Generate the Content** via `ai_languageModel`
      - Connect **tavily tool1 → Generate the Content** via `ai_tool`
      - Prompt: write the section with citations/links based on tool results.
   4) Connect: **Split Out1 → Generate the Content**.

11. **Merge drafting output with section metadata**
   1) Add **Merge** node named **“Merge”**.
   2) Configure merge mode to match your intent:
      - Common: **Merge by position** so generated content aligns with the originating section item.
   3) Connect: **Generate the Content → Merge (Input 1)**.
   4) Connect: **Split Out1 (secondary output stream) → Merge (Input 2)**  
      - In your JSON, Split Out1 feeds Merge on output index 1.

12. **Aggregate into a single article**
   1) Add **Aggregate** node named **“Aggregate1”**.
   2) Configure to aggregate section texts into one field (e.g., join with `\n\n` and preserve ordering).
   3) Connect: **Merge → Aggregate1**.

13. **Edit the full article**
   1) Add **OpenAI Chat Model** node named **“OpenAI”** (credentials + model).
   2) Add **LangChain Agent** node named **“Content Editor”**:
      - Connect **OpenAI → Content Editor** via `ai_languageModel`
      - Prompt: improve clarity, structure, consistency; preserve sources/claims; add intro/conclusion if needed.
   3) Connect: **Aggregate1 → Content Editor**.

14. **Generate title, slug, description**
   1) Add **OpenAI** node named **“Title Maker”**; prompt for SEO-friendly title.
   2) Add **OpenAI** node named **“Making Slug”**; prompt to output a URL-safe slug (lowercase, hyphens).
   3) Add **OpenAI** node named **“Article Description”**; prompt for meta description with length constraint.
   4) Connect: **Content Editor → Title Maker → Making Slug → Article Description**.

15. **Google Docs output**
   1) Add **Google Docs** node named **“Create a document1”**:
      - Configure **Google OAuth2** credentials with Docs/Drive scopes.
      - Set document title from the generated title field.
   2) Add **Google Docs** node named **“Update a document”**:
      - Use document ID from the create step.
      - Insert/append the article body, and optionally include slug/description at top.
   3) Connect: **Article Description → Create a document1 → Update a document**.

16. **Close the loop**
   1) Connect: **Update a document → Loop Over Items** (to trigger next batch).
   2) Ensure SplitInBatches is configured so it stops when no items remain.

17. **Sticky Notes (optional)**
   - Add sticky notes as desired; in the provided workflow they are present but empty.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided by the user with the workflow context |

