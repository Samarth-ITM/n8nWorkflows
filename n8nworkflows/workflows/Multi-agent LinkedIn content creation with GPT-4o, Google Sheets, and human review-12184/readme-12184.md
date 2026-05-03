Multi-agent LinkedIn content creation with GPT-4o, Google Sheets, and human review

https://n8nworkflows.xyz/workflows/multi-agent-linkedin-content-creation-with-gpt-4o--google-sheets--and-human-review-12184


# Multi-agent LinkedIn content creation with GPT-4o, Google Sheets, and human review

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** A weekly, multi-agent content engine that generates a production-grade LinkedIn post about n8n. It avoids repeating past topics by reading a Google Sheets history, separates “engineering blueprint” from “creative writing,” enforces strict formatting/branding rules, then sends the final draft to Gmail for **human approval** before logging it back to Google Sheets.

**Target use cases**
- Weekly thought-leadership posts with consistent technical depth and tone
- Avoiding topic repetition by using a “memory” sheet
- Human-in-the-loop quality gate before publishing
- Operational robustness via timeout and error notifications

### Logical blocks
1.1 **Scheduling & input memory retrieval** (Weekly trigger → read Sheet → aggregate rows)  
1.2 **Topic ideation + deduplication (Agent + structured output)**  
1.3 **Engineering blueprint (Agent + structured output)**  
1.4 **Draft writing + formatting/compliance QC (Agents)**  
1.5 **Human approval (Gmail send-and-wait) + logging**  
1.6 **System error handling (Error trigger → Gmail alert)**

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & input memory retrieval

**Overview:** Triggers weekly, fetches existing post history from Google Sheets, and aggregates it into a single dataset so the topic-selection agent can reason over the entire history at once.

**Nodes involved**
- Weekly Post Trigger
- Retrieve Post History & Ideas
- Aggregate

#### Node: Weekly Post Trigger
- **Type / role:** Schedule Trigger; initiates the workflow automatically.
- **Configuration (interpreted):** Runs **every week** on **Monday** (`triggerAtDay: 1`) at **10:15**.
- **Connections:** Outputs to **Retrieve Post History & Ideas**.
- **Edge cases / failures:** None typical, but timezone behavior depends on n8n instance settings (server timezone vs. user settings).

#### Node: Retrieve Post History & Ideas
- **Type / role:** Google Sheets; reads history rows used as “contextual memory.”
- **Configuration:** Uses OAuth2 credential; documentId and sheetName both set to placeholders `REPLACE_WITH_YOUR_SHEET_ID` (must be replaced). Operation is not explicitly shown in parameters (defaults to a read operation in this node family).
- **Output:** One item per row with fields like `Topic`, `Status`, `Difficulty` (and potentially others).
- **Connections:** Outputs to **Aggregate**.
- **Edge cases / failures:**
  - Invalid Sheet ID / permissions → 403/404.
  - Column names mismatch (e.g., `Topic` missing) will later break prompt assumptions.

#### Node: Aggregate
- **Type / role:** Aggregate node; consolidates all input items.
- **Configuration:** `aggregateAllItemData` (collect all items).
- **Connections:** Outputs to **Deduplication & Topic Selector**.
- **Edge cases / failures:** If the sheet returns zero rows, downstream logic may produce empty history; the agent will still run but may propose topics without constraints.

---

### 2.2 Topic ideation + deduplication (Agent + structured output)

**Overview:** An LLM agent proposes new weekly topics **not overlapping** with history, assigns difficulty, provides value rationale, and selects one best topic. Output is expected as structured JSON.

**Nodes involved**
- Brain: Topic Analyst (GPT-4o-mini)
- Structured Output Parser
- Deduplication & Topic Selector

#### Node: Brain: Topic Analyst (GPT-4o-mini)
- **Type / role:** OpenAI Chat Model (LangChain); language model powering the topic-selection agent.
- **Configuration:** Model `gpt-4o-mini`, temperature `0.4`.
- **Connections:** Connected via **ai_languageModel** to **Deduplication & Topic Selector**.
- **Edge cases / failures:** OpenAI auth/rate limit, model availability changes, timeouts.

#### Node: Structured Output Parser
- **Type / role:** LangChain Structured Output Parser; enforces JSON schema for agent output.
- **Schema intent:**  
  - `candidates[]`: list of `{ topic, difficulty, why }`  
  - `selected_topic`: `{ topic, difficulty, reason }`
- **Connections:** Connected via **ai_outputParser** to **Deduplication & Topic Selector** (the agent uses this parser).
- **Edge cases / failures:** If the model returns non-JSON or schema-incompatible JSON, the agent may error or produce partial output.

#### Node: Deduplication & Topic Selector
- **Type / role:** LangChain Agent; selects non-repeating topic from history.
- **Configuration choices:**
  - **Prompt builds history JSON** from aggregated items:
    - Uses expression to map sheet rows into `{topic, status, difficulty}` and `JSON.stringify(...)`.
  - System rules: avoid repeats, be concrete, sentence case, audience beginner–intermediate.
  - `hasOutputParser: true` → uses **Structured Output Parser**.
- **Key expressions / variables:**
  - `{{ JSON.stringify($input.all().map(item => ({ topic: item.json.Topic, status: item.json.Status, difficulty: item.json.Difficulty }))) }}`
- **Connections:**
  - Input: **Aggregate**
  - Output: **Structure & Logic Designer**
  - AI model: **Brain: Topic Analyst (GPT-4o-mini)**
  - Output parser: **Structured Output Parser**
- **Edge cases / failures:**
  - If Google Sheets columns differ (`Topic`/`Status`/`Difficulty` renamed), expression returns `undefined` values, weakening deduplication.
  - “Overlap” is semantic; LLM may still pick similar topics unless history is very explicit.

---

### 2.3 Engineering blueprint (Agent + structured output)

**Overview:** Converts the selected topic into a “teaching blueprint” that emphasizes architecture, mental models, resiliency, and advanced node usage. Output is structured to feed the copywriter reliably.

**Nodes involved**
- Brain: Content Architect (GPT-4o)
- Structured Output Parser1
- Structure & Logic Designer

#### Node: Brain: Content Architect (GPT-4o)
- **Type / role:** OpenAI Chat Model (LangChain) powering the blueprint agent.
- **Configuration:** Model `gpt-4o`, temperature `0.3`.
- **Connections:** **ai_languageModel** → **Structure & Logic Designer**
- **Edge cases:** Same OpenAI operational issues; schema drift if model output doesn’t match parser.

#### Node: Structured Output Parser1
- **Type / role:** Structured Output Parser for the blueprint schema.
- **Schema intent:**  
  `problem`, `when_it_matters`, `core_idea`, `practical_approach[]`, `common_mistake`, `result`
- **Connections:** **ai_outputParser** → **Structure & Logic Designer**
- **Important mismatch to note:** Downstream nodes reference fields named `architectural_approach` and `silent_killer`, but this parser schema uses `practical_approach` and `common_mistake`. Unless the agent actually outputs both sets of keys, this can cause undefined fields later.
- **Edge cases:** Parser failures if keys differ; downstream expressions resolving to empty strings.

#### Node: Structure & Logic Designer
- **Type / role:** LangChain Agent; generates the technical blueprint (“the why”).
- **Configuration choices:**
  - Input prompt uses selected topic from prior agent output:
    - `{{ $json.output.selected_topic.topic }}`
    - `{{ $json.output.selected_topic.difficulty }}`
    - `{{ $json.output.selected_topic.reason }}`
  - Strong system message enforcing:
    - “why-first” blueprinting
    - advanced nodes/features mention
    - resiliency tip
    - expert tone and sentence case
  - `hasOutputParser: true` → uses **Structured Output Parser1**
- **Connections:**
  - Input: **Deduplication & Topic Selector**
  - Output: **Draft Copywriter**
  - AI model: **Brain: Content Architect (GPT-4o)**
  - Output parser: **Structured Output Parser1**
- **Edge cases / failures:**
  - If `selected_topic` is missing, prompt renders empty topic.
  - Parser schema mismatch (see above) can propagate broken fields to copywriting.

---

### 2.4 Draft writing + formatting/compliance QC (Agents)

**Overview:** Writes the full LinkedIn post from the blueprint, then applies strict style/compliance rules (no em dashes, spaced hyphens, remove “follow” CTAs, exact hashtags, remove formatting).

**Nodes involved**
- Brain: Creative Writer (GPT-4o)
- Draft Copywriter
- Brain: Formatting Police (GPT-4o)
- Style & Compliance Editor

#### Node: Brain: Creative Writer (GPT-4o)
- **Type / role:** OpenAI Chat Model powering the drafting agent.
- **Configuration:** Model `gpt-4o`, temperature `0.5` for more variation.
- **Connections:** **ai_languageModel** → **Draft Copywriter**
- **Edge cases:** model variability may introduce disallowed characters (em dashes), which are later cleaned.

#### Node: Draft Copywriter
- **Type / role:** LangChain Agent; produces initial LinkedIn copy.
- **Configuration choices:**
  - Prompt expects fields:
    - `problem`, `when_it_matters`, `core_idea`
    - `architectural_approach` (joined if array)
    - `silent_killer`
    - `result`
  - System message enforces:
    - no preamble
    - strong hook
    - short paragraphs, bullet points (•)
    - max 2–3 emojis total
    - explicitly name n8n nodes in approach
- **Key expressions / variables:**
  - `{{ Array.isArray($json.output.architectural_approach) ? $json.output.architectural_approach.join("\n- ") : $json.output.architectural_approach }}`
- **Connections:**
  - Input: **Structure & Logic Designer**
  - Output: **Style & Compliance Editor**
  - AI model: **Brain: Creative Writer (GPT-4o)**
- **Critical edge case:** The upstream parser schema uses `practical_approach` / `common_mistake`, but this node references `architectural_approach` / `silent_killer`. If not aligned, the drafted post will have missing sections or “undefined”.
- **Other failures:** Long outputs may exceed Gmail or sheet cell practical limits; consider truncation rules.

#### Node: Brain: Formatting Police (GPT-4o)
- **Type / role:** OpenAI Chat Model powering the editing agent.
- **Configuration:** Model `gpt-4o`, default temperature (not set).
- **Connections:** **ai_languageModel** → **Style & Compliance Editor**
- **Edge cases:** If temperature defaults high, can over-edit; you may want a low temperature for deterministic compliance.

#### Node: Style & Compliance Editor
- **Type / role:** LangChain Agent; final editorial pass enforcing brand rules.
- **Configuration choices:**
  - Removes em dashes, enforces spaced hyphens `word - word`
  - Removes “follow/like/subscribe” CTAs
  - Adds exact hashtags: `#n8n #automation #lowcode`
  - Removes all markdown styles (bold/italic)
  - Replaces “Function Node” → “Code Node”
  - Outputs only final post text (no wrapper)
- **Connections:**
  - Input: **Draft Copywriter**
  - Output: **Human Approval (Gmail)**
  - AI model: **Brain: Formatting Police (GPT-4o)**
- **Edge cases / failures:**
  - If draft is empty (due to schema mismatch), editor produces a thin/invalid post.
  - Hashtag rule is strict; ensure the editor always appends exactly those three.

---

### 2.5 Human approval (Gmail send-and-wait) + logging

**Overview:** Sends the finalized post to a human via Gmail using an approval flow. If approved, logs it in Google Sheets. If not approved within 48 hours, execution ends gracefully.

**Nodes involved**
- Human Approval (Gmail)
- Log Published Content
- Wait Timed Out (48h)

#### Node: Human Approval (Gmail)
- **Type / role:** Gmail node with **sendAndWait** approval.
- **Configuration choices:**
  - To: `user@example.com` (must be changed)
  - Subject includes selected topic:
    - `Ready for LinkedIn: {{ $('Deduplication & Topic Selector').item.json.output.selected_topic.topic }}`
  - Message includes HTML-like `<strong>` and the edited content: `{{ $json.output }}`
  - Approval: `approvalType: double` (requires double confirmation)
  - Timeout: `resumeAmount: 48` hours limit wait time
  - `onError: continueErrorOutput` allows workflow to continue on errors (but still may produce partial behavior)
- **Connections:**
  - Input: **Style & Compliance Editor**
  - Output 0 → **Log Published Content** (approval path)
  - Output 1 → **Wait Timed Out (48h)** (timeout path)
- **Edge cases / failures:**
  - Gmail OAuth token expiry, insufficient scopes.
  - If execution resumes after timeout, logging should not occur (current wiring routes timeout to no-op).

#### Node: Log Published Content
- **Type / role:** Google Sheets append; logs the approved draft to history.
- **Configuration choices:**
  - Append row with:
    - `Date`: `{{ $now.toFormat("dd-MM-yyyy") }}`
    - `Topic`: from Deduplication node selected topic
    - `Status`: literal `Drafted`
    - `Content`: from Style & Compliance Editor output
    - `Difficulty`: from Deduplication node selected difficulty
  - Sheet/document IDs are placeholders to replace.
  - Column schema includes `LinkedIn URL` but marked removed; not written.
- **Connections:** Input from **Human Approval (Gmail)** approval branch.
- **Edge cases / failures:**
  - Sheet permissions / wrong sheet.
  - Large content cells can exceed practical limits; may truncate or fail depending on API.
  - “Status = Drafted” indicates logging happens after approval but before actual posting (intentional per sticky note).

#### Node: Wait Timed Out (48h)
- **Type / role:** NoOp; marks the timeout branch end-state.
- **Notes:** “Execution timed out after 48h.”
- **Connections:** Input from **Human Approval (Gmail)** timeout branch.
- **Edge cases:** None (it’s intentionally inert).

---

### 2.6 System error handling (Error trigger → Gmail alert)

**Overview:** Catches workflow execution errors globally and emails the operator with node name and error message.

**Nodes involved**
- Error Trigger
- Send a message

#### Node: Error Trigger
- **Type / role:** Error Trigger; starts when the workflow errors.
- **Configuration:** Default.
- **Connections:** Outputs to **Send a message**.
- **Edge cases:** Only triggers on workflow errors; if nodes use “continue on fail” patterns, some failures may not bubble up.

#### Node: Send a message
- **Type / role:** Gmail send email; sends failure notification.
- **Configuration choices:**
  - To: `user@example.com` (must be changed)
  - Subject: `⚠️ Workflow Error: LinkedIn Content Engine`
  - Body includes:
    - failing node name: `{{ $node["Error Trigger"].json["node"]["name"] }}`
    - error message: `{{ $node["Error Trigger"].json["execution"]["error"]["message"] }}`
- **Connections:** Input from **Error Trigger** only.
- **Edge cases:** If Gmail credential fails, error notification won’t send (consider adding a secondary channel like Slack).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Post Trigger | n8n-nodes-base.scheduleTrigger | Weekly entry point | — | Retrieve Post History & Ideas | ## PHASE 1: BRAINSTORMING |
| Retrieve Post History & Ideas | n8n-nodes-base.googleSheets | Load topic/history memory from Sheets | Weekly Post Trigger | Aggregate | ## PHASE 1: BRAINSTORMING |
| Aggregate | n8n-nodes-base.aggregate | Aggregate all rows for LLM context | Retrieve Post History & Ideas | Deduplication & Topic Selector | ## PHASE 1: BRAINSTORMING |
| Brain: Topic Analyst (GPT-4o-mini) | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for topic selection | — (AI attachment) | Deduplication & Topic Selector (ai_languageModel) | ## PHASE 1: BRAINSTORMING |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON structure for topic selection | — (AI attachment) | Deduplication & Topic Selector (ai_outputParser) | ## PHASE 1: BRAINSTORMING |
| Deduplication & Topic Selector | @n8n/n8n-nodes-langchain.agent | Propose and pick non-repeating weekly topic | Aggregate | Structure & Logic Designer | ## PHASE 1: BRAINSTORMING |
| Brain: Content Architect (GPT-4o) | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for engineering blueprint | — (AI attachment) | Structure & Logic Designer (ai_languageModel) | ## PHASE 2: ENGINEERING |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON structure for blueprint | — (AI attachment) | Structure & Logic Designer (ai_outputParser) | ## PHASE 2: ENGINEERING |
| Structure & Logic Designer | @n8n/n8n-nodes-langchain.agent | Build teaching blueprint (“the why”) | Deduplication & Topic Selector | Draft Copywriter | ## PHASE 2: ENGINEERING |
| Brain: Creative Writer (GPT-4o) | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for initial draft | — (AI attachment) | Draft Copywriter (ai_languageModel) | ## PHASE 3: CREATIVE & QC |
| Draft Copywriter | @n8n/n8n-nodes-langchain.agent | Write LinkedIn post draft | Structure & Logic Designer | Style & Compliance Editor | ## PHASE 3: CREATIVE & QC |
| Brain: Formatting Police (GPT-4o) | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for compliance editing | — (AI attachment) | Style & Compliance Editor (ai_languageModel) | ## PHASE 3: CREATIVE & QC |
| Style & Compliance Editor | @n8n/n8n-nodes-langchain.agent | Enforce strict style/brand rules | Draft Copywriter | Human Approval (Gmail) | ## PHASE 3: CREATIVE & QC |
| Human Approval (Gmail) | n8n-nodes-base.gmail | Human-in-the-loop approval (send and wait) | Style & Compliance Editor | Log Published Content; Wait Timed Out (48h) | ## PHASE 4: DELIVERY |
| Log Published Content | n8n-nodes-base.googleSheets | Append approved draft to Sheets history | Human Approval (Gmail) | — | ## PHASE 4: DELIVERY |
| Wait Timed Out (48h) | n8n-nodes-base.noOp | End branch when approval times out | Human Approval (Gmail) | — | ## PHASE 4: DELIVERY |
| Error Trigger | n8n-nodes-base.errorTrigger | Global error entry point | — | Send a message | ## SYSTEM: Error Handling |
| Send a message | n8n-nodes-base.gmail | Email alert on workflow failure | Error Trigger | — | ## SYSTEM: Error Handling |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block (meta) | — | — | ## [PRO] n8n Content Architect: Multi-Agent Engine … (content preserved in Notes section) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Phase label | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Phase label | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Phase label | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Phase label | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | System label | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Start instructions | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Google Sheet (history/memory)**
   1. Create a sheet (tab) containing at least these columns: **Topic, Content, Date, Status, Difficulty**.
   2. Copy the Spreadsheet ID (from the URL).

2) **Create credentials**
   1. **Google Sheets OAuth2** credential (connect the Google account that owns/has access to the sheet).
   2. **Gmail OAuth2** credential (account used to send approval + error emails).
   3. **OpenAI API** credential (for GPT-4o and GPT-4o-mini).

3) **Add nodes (Phase 1: Brainstorming)**
   1. Add **Schedule Trigger** named **Weekly Post Trigger**
      - Set weekly schedule: Monday at 10:15 (adjust timezone as needed).
   2. Add **Google Sheets** node named **Retrieve Post History & Ideas**
      - Credentials: Google Sheets OAuth2
      - Document ID: your Spreadsheet ID
      - Sheet name: your tab name
      - Operation: read/get rows (default “read” behavior).
   3. Add **Aggregate** node named **Aggregate**
      - Mode: aggregate all item data.
   4. Add **OpenAI Chat Model** node named **Brain: Topic Analyst (GPT-4o-mini)**
      - Model: `gpt-4o-mini`
      - Temperature: 0.4
   5. Add **Structured Output Parser** node named **Structured Output Parser**
      - Schema example:
        - `candidates[]` with `topic`, `difficulty`, `why`
        - `selected_topic` with `topic`, `difficulty`, `reason`
   6. Add **AI Agent** node named **Deduplication & Topic Selector**
      - System message: n8n expert educator; do not repeat; sentence case; concrete topics.
      - User prompt: stringify history from inputs (map Topic/Status/Difficulty).
      - Attach:
        - Language model: **Brain: Topic Analyst (GPT-4o-mini)**
        - Output parser: **Structured Output Parser**

4) **Add nodes (Phase 2: Engineering)**
   1. Add **OpenAI Chat Model** node named **Brain: Content Architect (GPT-4o)**
      - Model: `gpt-4o`
      - Temperature: 0.3
   2. Add **Structured Output Parser** node named **Structured Output Parser1**
      - Schema example (recommended to match downstream): either
        - Option A (align to current Draft node): `problem, when_it_matters, core_idea, architectural_approach[], silent_killer, result`
        - Option B (update Draft node to match parser): `problem, when_it_matters, core_idea, practical_approach[], common_mistake, result`
   3. Add **AI Agent** node named **Structure & Logic Designer**
      - Prompt uses `selected_topic.topic`, `selected_topic.difficulty`, `selected_topic.reason`
      - System message: blueprint with problem/context/core concept/approach/silent killer/result; include advanced nodes + resiliency tip; sentence case.
      - Attach:
        - Language model: **Brain: Content Architect (GPT-4o)**
        - Output parser: **Structured Output Parser1**

5) **Add nodes (Phase 3: Creative & QC)**
   1. Add **OpenAI Chat Model** node named **Brain: Creative Writer (GPT-4o)**
      - Model: `gpt-4o`
      - Temperature: 0.5
   2. Add **AI Agent** node named **Draft Copywriter**
      - System message: strict LinkedIn writing rules (hook, whitespace, bullets, node naming, low emoji).
      - Prompt: render blueprint fields into the required sections and end with a light open question CTA.
      - Attach language model: **Brain: Creative Writer (GPT-4o)**
   3. Add **OpenAI Chat Model** node named **Brain: Formatting Police (GPT-4o)**
      - Model: `gpt-4o` (set low temperature if you want consistency).
   4. Add **AI Agent** node named **Style & Compliance Editor**
      - System message: no em dashes, spaced hyphens, remove follow CTAs, exact hashtags, remove styles, replace Function Node→Code Node, output only final text.
      - Attach language model: **Brain: Formatting Police (GPT-4o)**

6) **Add nodes (Phase 4: Delivery)**
   1. Add **Gmail** node named **Human Approval (Gmail)**
      - Operation: **Send and wait**
      - To: your email
      - Subject: `Ready for LinkedIn: {{ $('Deduplication & Topic Selector').item.json.output.selected_topic.topic }}`
      - Message: include final post body from previous node
      - Approval type: **double**
      - Max wait/timeout: **48 hours**
      - Consider enabling continue on fail if desired.
   2. Add **Google Sheets** node named **Log Published Content**
      - Operation: **Append**
      - Document ID + Sheet name: same sheet (or a History tab)
      - Map columns:
        - Date = `{{$now.toFormat("dd-MM-yyyy")}}`
        - Topic = selected topic
        - Status = `Drafted`
        - Content = final edited post
        - Difficulty = selected difficulty
   3. Add **NoOp** node named **Wait Timed Out (48h)** for the timeout branch end.

7) **Add system error handling**
   1. Add **Error Trigger** node.
   2. Add **Gmail** node named **Send a message**
      - To: your email
      - Subject: `⚠️ Workflow Error: LinkedIn Content Engine`
      - Message includes failing node + error message from Error Trigger JSON.

8) **Connect nodes (main path)**
   1. Weekly Post Trigger → Retrieve Post History & Ideas → Aggregate → Deduplication & Topic Selector → Structure & Logic Designer → Draft Copywriter → Style & Compliance Editor → Human Approval (Gmail)
   2. Human Approval (approved output) → Log Published Content
   3. Human Approval (timeout output) → Wait Timed Out (48h)
   4. Error Trigger → Send a message
   5. Attach AI model + output parsers to the three agent nodes exactly as above.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **[PRO] n8n Content Architect: Multi-Agent Engine**. Goal: Generate high-authority, “Sentence Case” LinkedIn content with topic variety and technical depth. Logic: contextual memory via Google Sheets; architectural blueprinting separate from creative writing; branding compliance (spaced-hyphens, strip AI fluff); human-in-the-loop via Gmail for review/image pairing/publishing. Resiliency: Gmail approval auto-timeout after 48h; global Error Trigger for API/model failures; only log topics after approval to preserve memory integrity. Setup: Google Sheets columns Topic/Status/Difficulty (and others), OpenAI GPT-4o & GPT-4o-mini, update destination email in Gmail node. | Sticky note content from the workflow canvas |
| START HERE: Create Google Sheet with Topic, Content, Date, Status, Difficulty; update Document ID in both Sheets nodes; update recipient email in Gmail approval node (2x). | Sticky note content from the workflow canvas |
| Important implementation warning: blueprint parser keys (`practical_approach`, `common_mistake`) currently don’t match drafting prompt keys (`architectural_approach`, `silent_killer`). Align them to avoid undefined sections. | Derived from node configuration consistency check |