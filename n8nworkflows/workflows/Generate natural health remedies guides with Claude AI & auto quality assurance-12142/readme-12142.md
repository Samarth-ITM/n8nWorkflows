Generate natural health remedies guides with Claude AI & auto quality assurance

https://n8nworkflows.xyz/workflows/generate-natural-health-remedies-guides-with-claude-ai---auto-quality-assurance-12142


# Generate natural health remedies guides with Claude AI & auto quality assurance

## disclaimer
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---

## 1. Workflow Overview

**Purpose:** This workflow generates a **personalized natural health remedies guide** based on user (patient) form input, using multiple AI agent steps and an **automatic quality assurance gate**. If the quality evaluation fails, it routes the content to an optimizer loop; if it passes, it generates a final Google Doc and logs execution metrics.

**Typical use cases:**
- Generating structured wellness/remedies guides for a reported condition and user context
- Enforcing “AI output quality” before publishing
- Tracking token usage/metrics for cost and performance monitoring

### 1.1 Input Reception & Normalization
Receives form input and converts it into a structured payload for downstream AI agents.

### 1.2 Disease/Condition Enrichment (AI)
Uses an AI agent to generate disease/condition information that will inform the guide.

### 1.3 Quality Evaluation (AI) + Decision Gate
Evaluates generated health content and decides whether it is acceptable or needs optimization.

### 1.4 Optimization Loop (AI)
If not approved, an optimizer agent attempts to improve the content, then re-enters the evaluation chain.

### 1.5 Final Document Output + Metrics Logging
If approved, performs final calculations/aggregation, writes to Google Docs, and logs metrics to a data table.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception & Normalization
**Overview:** Captures user input from an n8n form trigger and transforms it into a structured, AI-ready format.

**Nodes involved:**
- **Form Input Configuration**
- **Process Patient Input**

#### Node: Form Input Configuration
- **Type / role:** `Form Trigger` (n8n form-based entry point). Starts the workflow when a user submits a form.
- **Configuration (interpreted):** The JSON shows empty `parameters`, meaning the form fields are not visible here. In practice, this node defines the fields (e.g., symptoms, condition, age, constraints, allergies).
- **Inputs/Outputs:**
  - **Output →** Process Patient Input
- **Edge cases / failures:**
  - Missing/blank fields leading to downstream expression errors or poor AI outputs
  - Unexpected field names if the form was edited after building code/expressions downstream
- **Version notes:** `typeVersion 2.3` (form node behavior/field schema can differ slightly across versions)

#### Node: Process Patient Input
- **Type / role:** `Code` node. Normalizes and likely validates the form submission into consistent keys.
- **Configuration (interpreted):** Parameters are empty in exported JSON, so the actual code content is not present. Expected behavior:
  - Read form fields from `$json`
  - Create a structured object such as `{ patientProfile, condition, constraints, desiredFormat }`
- **Inputs/Outputs:**
  - **Input ←** Form Input Configuration
  - **Output →** Disease Info Agent
- **Edge cases / failures:**
  - If code expects fields that the form doesn’t provide, it will throw (undefined access)
  - If multiple items are output unintentionally, AI nodes may receive unexpected batches
- **Version notes:** `typeVersion 2` code node

---

### Block 2.2 — Disease/Condition Enrichment (AI)
**Overview:** Produces condition-specific background and structured medical-style information to ground the guide.

**Nodes involved:**
- **Disease Info Agent**
- **Capture Disease Info Tokens**

#### Node: Disease Info Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (LangChain OpenAI node used as an “agent” step). Despite the workflow title mentioning “Claude”, the node type indicates an OpenAI/LangChain integration.
- **Configuration (interpreted):** Parameters are not shown; typically includes:
  - Model selection, prompt/system instructions, temperature
  - Input variables mapped from `Process Patient Input` output
- **Inputs/Outputs:**
  - **Input ←** Process Patient Input
  - **Output →** Capture Disease Info Tokens
- **Edge cases / failures:**
  - Credential/auth errors to the provider
  - Model errors/timeouts, rate limits
  - Hallucinated medical claims (requires careful prompting; also mitigated by evaluator block)
- **Version notes:** `typeVersion 2.1`

#### Node: Capture Disease Info Tokens
- **Type / role:** `Code` node to extract token usage/metadata from the AI response (common pattern: read usage fields and store them in the item).
- **Configuration (interpreted):** Code not present; likely:
  - Reads AI response usage fields (prompt/completion tokens, total tokens)
  - Writes them to `$json.metrics.diseaseInfoTokens` or similar
- **Inputs/Outputs:**
  - **Input ←** Disease Info Agent
  - **Output →** Set Health Info
- **Edge cases / failures:**
  - If AI node output schema differs (no `usage` field), code may fail
  - Multi-item handling if AI returns multiple generations
- **Version notes:** `typeVersion 2`

---

### Block 2.3 — Shared State Assembly (“Set Health Info”)
**Overview:** Consolidates and stages health/disease info into a consistent structure for evaluation and later document generation; also serves as the re-entry point after optimization.

**Nodes involved:**
- **Set Health Info**

#### Node: Set Health Info
- **Type / role:** `Set` node for shaping the JSON payload (selecting/renaming fields; preparing “current draft”).
- **Configuration (interpreted):** Parameters empty in export; typical use:
  - Set fields like `draftGuide`, `diseaseInfo`, `patientContext`, `metricsSoFar`
  - Choose “Keep Only Set” or “Keep All” (unknown here)
- **Inputs/Outputs:**
  - **Input ←** Capture Disease Info Tokens **and** Capture Optimizer Tokens (loopback)
  - **Output →** Health Evaluator Agent
- **Edge cases / failures:**
  - Overwriting fields unintentionally (e.g., losing previous metrics)
  - If configured to “Keep Only Set”, downstream nodes may miss required keys
- **Version notes:** `typeVersion 3.4`

---

### Block 2.4 — Quality Evaluation (AI) + Decision
**Overview:** Evaluates whether the current health guide draft meets quality/safety criteria and routes either to finalization or to optimization.

**Nodes involved:**
- **Health Evaluator Agent**
- **Capture Evaluator Tokens**
- **Check Quality Approval**

#### Node: Health Evaluator Agent
- **Type / role:** LangChain OpenAI agent node acting as a “quality evaluator”.
- **Configuration (interpreted):**
  - Prompt likely checks for: safety disclaimers, non-diagnostic framing, structure completeness, contraindications, clarity, and consistency with patient constraints
  - Output should include an approval signal (boolean/string/score) consumed by the IF node
- **Inputs/Outputs:**
  - **Input ←** Set Health Info
  - **Output →** Capture Evaluator Tokens
- **Edge cases / failures:**
  - Evaluator output not machine-readable (e.g., missing `approved=true`), breaking IF logic
  - Model variance (temperature too high) causing inconsistent gating
- **Version notes:** `typeVersion 2.1`

#### Node: Capture Evaluator Tokens
- **Type / role:** `Code` node capturing evaluator token usage and/or extracting approval fields into predictable keys.
- **Configuration (interpreted):**
  - Likely parses evaluator response to set fields like `quality.approved`, `quality.score`, `quality.feedback`
  - Records token usage into metrics
- **Inputs/Outputs:**
  - **Input ←** Health Evaluator Agent
  - **Output →** Check Quality Approval
- **Edge cases / failures:**
  - Schema mismatch for evaluator output
  - Parsing errors if approval is embedded in text rather than JSON
- **Version notes:** `typeVersion 2`

#### Node: Check Quality Approval
- **Type / role:** `IF` node (routing gate).
- **Configuration (interpreted):** Parameters are empty in export; normally this node contains a condition such as:
  - `{{$json.quality.approved}} is true` OR `{{$json.quality.score}} >= threshold`
- **Routing (important):**
  - **Output 0 (true path) →** Update Final Calculation Node
  - **Output 1 (false path) →** Health Optimizer Agent
- **Edge cases / failures:**
  - If condition is not configured, it may default or always false/true depending on UI state; exported JSON showing `{}` is a strong sign configuration is missing in the export or was not set.
  - If the evaluator doesn’t produce the expected field, the IF condition will error or route incorrectly.
- **Version notes:** `typeVersion 2.3`

---

### Block 2.5 — Optimization Loop (AI)
**Overview:** When quality fails, an optimizer agent attempts to improve the draft using evaluator feedback, then loops back into the shared state assembly for re-evaluation.

**Nodes involved:**
- **Health Optimizer Agent**
- **Capture Optimizer Tokens**
- (Loop continues via **Set Health Info → Health Evaluator Agent → …**)

#### Node: Health Optimizer Agent
- **Type / role:** LangChain OpenAI agent that rewrites/improves the guide.
- **Configuration (interpreted):**
  - Prompt likely takes `draftGuide` + `quality.feedback` and produces a revised draft
  - Could enforce structure: sections, bullet points, contraindications, “consult a professional” disclaimer
- **Inputs/Outputs:**
  - **Input ←** Check Quality Approval (false branch)
  - **Output →** Capture Optimizer Tokens
- **Edge cases / failures:**
  - Infinite loop risk if no loop counter/stop condition exists (none is visible in JSON)
  - Optimizer may “over-correct” or remove useful personalization if prompt is too strict
- **Version notes:** `typeVersion 2.1`

#### Node: Capture Optimizer Tokens
- **Type / role:** `Code` node capturing optimizer token usage and normalizing optimizer output back into fields expected by `Set Health Info`.
- **Inputs/Outputs:**
  - **Input ←** Health Optimizer Agent
  - **Output →** Set Health Info (loopback)
- **Edge cases / failures:**
  - Same schema/token-metadata concerns as other capture nodes
- **Version notes:** `typeVersion 2`

---

### Block 2.6 — Finalization, Document Generation, and Metrics Logging
**Overview:** Once approved, finalizes calculations/metrics, generates the Google Doc guide, and stores run data.

**Nodes involved:**
- **Update Final Calculation Node**
- **Natural Health Guide**
- **Log Execution Metrics**

#### Node: Update Final Calculation Node
- **Type / role:** `Code` node to compute final metrics and assemble the final document payload.
- **Configuration (interpreted):**
  - Likely aggregates token usage from all capture nodes
  - Finalizes `finalGuideText`, `title`, and document metadata
- **Inputs/Outputs:**
  - **Input ←** Check Quality Approval (true branch)
  - **Output →** Natural Health Guide
- **Edge cases / failures:**
  - Missing metrics fields if earlier branches didn’t set them
  - Number parsing/NaN issues if tokens are strings
- **Version notes:** `typeVersion 2`

#### Node: Natural Health Guide
- **Type / role:** `Google Docs` node to create/update a document containing the final guide.
- **Configuration (interpreted):**
  - Needs Google credentials
  - Typically configured to “Create Document” and insert formatted text
- **Inputs/Outputs:**
  - **Input ←** Update Final Calculation Node
  - **Output →** Log Execution Metrics
- **Edge cases / failures:**
  - Google OAuth credential expired/insufficient permissions
  - Document creation quota, API errors
  - Large content hitting Google Docs API limits (chunking may be needed)
- **Version notes:** `typeVersion 2`

#### Node: Log Execution Metrics
- **Type / role:** `Data Table` node to store metrics (tokens, timestamps, approval status, doc link, etc.).
- **Configuration (interpreted):**
  - Requires an n8n Data Table configured for inserts/updates
- **Inputs/Outputs:**
  - **Input ←** Natural Health Guide
  - **Output:** none (end)
- **Edge cases / failures:**
  - Table not created / schema mismatch
  - Write permissions issues
- **Version notes:** `typeVersion 1`

---

### Block 2.7 — Sticky Notes (Comments/Documentation)
**Overview:** The workflow includes multiple sticky notes, but all have **empty content** in the provided JSON, so they do not add documentation context.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form Input Configuration | Form Trigger | Entry point (collect patient input) | — | Process Patient Input |  |
| Process Patient Input | Code | Normalize/structure incoming form data | Form Input Configuration | Disease Info Agent |  |
| Disease Info Agent | LangChain OpenAI | Generate condition/disease background info | Process Patient Input | Capture Disease Info Tokens |  |
| Capture Disease Info Tokens | Code | Extract token usage + normalize disease info output | Disease Info Agent | Set Health Info |  |
| Set Health Info | Set | Consolidate shared payload for evaluation/guide | Capture Disease Info Tokens; Capture Optimizer Tokens | Health Evaluator Agent |  |
| Health Evaluator Agent | LangChain OpenAI | QA evaluation and approval decision content | Set Health Info | Capture Evaluator Tokens |  |
| Capture Evaluator Tokens | Code | Extract evaluator tokens + approval fields | Health Evaluator Agent | Check Quality Approval |  |
| Check Quality Approval | IF | Route approved vs needs optimization | Capture Evaluator Tokens | Update Final Calculation Node (true); Health Optimizer Agent (false) |  |
| Health Optimizer Agent | LangChain OpenAI | Improve the draft using feedback | Check Quality Approval (false path) | Capture Optimizer Tokens |  |
| Capture Optimizer Tokens | Code | Extract optimizer tokens + normalize revised draft | Health Optimizer Agent | Set Health Info |  |
| Update Final Calculation Node | Code | Aggregate final metrics + finalize payload | Check Quality Approval (true path) | Natural Health Guide |  |
| Natural Health Guide | Google Docs | Create/update the final guide document | Update Final Calculation Node | Log Execution Metrics |  |
| Log Execution Metrics | Data Table | Persist run metrics | Natural Health Guide | — |  |
| Sticky Note | Sticky Note | Canvas comment (empty) | — | — |  |
| Sticky Note1 | Sticky Note | Canvas comment (empty) | — | — |  |
| Sticky Note2 | Sticky Note | Canvas comment (empty) | — | — |  |
| Sticky Note3 | Sticky Note | Canvas comment (empty) | — | — |  |
| Sticky Note4 | Sticky Note | Canvas comment (empty) | — | — |  |
| Sticky Note5 | Sticky Note | Canvas comment (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Form Trigger** named **“Form Input Configuration”**.
   2. Define fields you need (recommended): condition/disease, symptoms, age, sex (optional), allergies, medications, dietary preferences, pregnancy status, constraints, goal, output language.
   3. Save the form and ensure submission produces a predictable JSON structure.

2) **Normalize Input**
   1. Add node: **Code** named **“Process Patient Input”**.
   2. Implement logic to:
      - Validate required fields (condition/symptoms)
      - Create normalized keys (e.g., `patient`, `condition`, `constraints`)
   3. Connect: **Form Input Configuration → Process Patient Input**.

3) **Disease Info AI Step**
   1. Add node: **LangChain OpenAI** named **“Disease Info Agent”**.
   2. Configure credentials for your AI provider (OpenAI-compatible via LangChain node).
   3. Set prompt to output structured disease info relevant to natural remedies (include safety constraints).
   4. Connect: **Process Patient Input → Disease Info Agent**.

4) **Capture Disease Tokens**
   1. Add node: **Code** named **“Capture Disease Info Tokens”**.
   2. Extract token usage fields from the AI node output (as available) and store in `metrics`.
   3. Connect: **Disease Info Agent → Capture Disease Info Tokens**.

5) **Assemble Shared Payload**
   1. Add node: **Set** named **“Set Health Info”**.
   2. Map fields into a stable structure (example fields):
      - `diseaseInfo` (from Disease Info Agent)
      - `draftGuide` (initial draft or placeholder)
      - `metrics` (accumulated)
   3. Connect: **Capture Disease Info Tokens → Set Health Info**.

6) **Evaluator AI Step**
   1. Add node: **LangChain OpenAI** named **“Health Evaluator Agent”**.
   2. Prompt it to return a **machine-readable** result, ideally JSON like:
      - `approved` (boolean)
      - `score` (0–100)
      - `issues` (array)
      - `rewriteInstructions` (text)
   3. Connect: **Set Health Info → Health Evaluator Agent**.

7) **Capture Evaluator Tokens + Parse Approval**
   1. Add node: **Code** named **“Capture Evaluator Tokens”**.
   2. Parse evaluator output into stable fields like `$json.quality.approved`, `$json.quality.score`, `$json.quality.feedback`.
   3. Append evaluator token usage into `$json.metrics`.
   4. Connect: **Health Evaluator Agent → Capture Evaluator Tokens**.

8) **Quality Gate**
   1. Add node: **IF** named **“Check Quality Approval”**.
   2. Configure condition (example): `Boolean → {{$json.quality.approved}} is true`.
   3. Connect: **Capture Evaluator Tokens → Check Quality Approval**.

9) **Optimizer AI Step (Fail Branch)**
   1. Add node: **LangChain OpenAI** named **“Health Optimizer Agent”**.
   2. Prompt it to rewrite `draftGuide` using `quality.feedback` and enforce required sections and safety.
   3. Connect **IF false output → Health Optimizer Agent**.

10) **Capture Optimizer Tokens + Loop Back**
   1. Add node: **Code** named **“Capture Optimizer Tokens”**.
   2. Store optimizer tokens and set `$json.draftGuide` to the improved text.
   3. Connect: **Health Optimizer Agent → Capture Optimizer Tokens → Set Health Info** (this forms the loop).

   **Important:** Add a loop counter (recommended) in `Set` or `Code` to avoid infinite re-tries (e.g., stop after 2–3 passes and force approval/manual review).

11) **Finalize (Pass Branch)**
   1. Add node: **Code** named **“Update Final Calculation Node”**.
   2. Aggregate totals (token sums) and finalize document content/title.
   3. Connect **IF true output → Update Final Calculation Node**.

12) **Create Google Doc**
   1. Add node: **Google Docs** named **“Natural Health Guide”**.
   2. Configure Google OAuth2 credentials with permission to create/edit docs.
   3. Operation: typically **Create** document; set title and body from final payload.
   4. Connect: **Update Final Calculation Node → Natural Health Guide**.

13) **Log Metrics**
   1. Add node: **Data Table** named **“Log Execution Metrics”**.
   2. Configure the target table and columns (e.g., runId, timestamp, approved, score, totalTokens, docUrl).
   3. Connect: **Natural Health Guide → Log Execution Metrics**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow title shown in JSON is “Generate personalized health guides with AI-powered quality assurance”. User-provided title is “Generate natural health remedies guides with Claude AI & auto quality assurance”. The AI nodes are LangChain OpenAI-type, not explicitly Claude. | Ensure the AI node/provider matches your intended model (OpenAI vs Anthropic/Claude). |
| Sticky notes exist but have empty content in the provided workflow export. | No additional embedded documentation was available. |
| Strongly recommended: add a max-iterations safeguard to the optimizer loop. | Prevent infinite cycles if evaluator keeps rejecting. |

---