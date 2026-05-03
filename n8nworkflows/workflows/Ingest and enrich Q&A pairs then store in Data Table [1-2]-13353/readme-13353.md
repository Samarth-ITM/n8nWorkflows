Ingest and enrich Q&A pairs then store in Data Table [1/2]

https://n8nworkflows.xyz/workflows/ingest-and-enrich-q-a-pairs-then-store-in-data-table--1-2--13353


# Ingest and enrich Q&A pairs then store in Data Table [1/2]

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Collect Question/Answer submissions via an n8n Form, enrich each submission by (a) flagging whether it is “trusted” based on the submitter’s email domain, and (b) generating topical tags via an OpenAI chat model, then **store the enriched record in an n8n Data Table**.

**Typical use cases:** internal knowledge capture, Q&A intake for documentation/FAQ, community/partner submissions where “trusted” sources should be identified.

### 1.1 Input Reception (Form)
Captures Name, Email, Question, Answer via an n8n form trigger and starts the workflow.

### 1.2 Trust Flagging (Email Domain Check)
Checks whether the submitted email contains `n8n.io` and sets `isTrusted` accordingly.

### 1.3 AI Enrichment (Tag Generation)
Uses an OpenAI chat model through LangChain nodes to produce 4–8 comma-separated tags based on the Q&A.

### 1.4 Persistence (Data Table Insert)
Writes Name, Email, Question, Answer, Tags, and isTrusted into the `q&a` Data Table.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception (Form)

**Overview:** Receives Q&A submissions from an n8n Form and outputs a JSON item containing the submitted fields.

**Nodes involved:**
- **On form submission**

#### Node: On form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — entry point that starts the workflow on form submit.
- **Configuration (interpreted):**
  - Form title: **“QA Submission”**
  - Fields:
    - `Name` (text)
    - `Email` (text)
    - `Question` (text)
    - `Answer` (textarea)
- **Key fields produced:** `Name`, `Email`, `Question`, `Answer` plus form metadata (e.g., `submittedAt`, `formMode` during tests).
- **Connections:**
  - **Output →** `is n8n.io email?`
- **Version-specific:** typeVersion **2.5** (Form Trigger node behavior/fields depend on n8n version; ensure Form Trigger is available and supports the UI form builder).
- **Edge cases / failures:**
  - Form not published / wrong webhook URL used.
  - Missing expected fields if the form is edited later (downstream expressions will fail if `Email`, `Question`, or `Answer` are absent).

---

### Block 1.2 — Trust Flagging (Email Domain Check)

**Overview:** Determines whether the submitter’s email is from `n8n.io` and adds a boolean `isTrusted` field to the item.

**Nodes involved:**
- **is n8n.io email?**
- **isTrusted:true**
- **isTrusted:false**
- **ref**

#### Node: is n8n.io email?
- **Type / role:** `n8n-nodes-base.if` — conditional branching.
- **Configuration (interpreted):**
  - Condition: **string contains**
    - Left: `{{$json.Email}}`
    - Right: `n8n.io`
  - Case sensitive: **true**
  - Strict type validation: **strict**
- **Connections:**
  - **True →** `isTrusted:true`
  - **False →** `isTrusted:false`
- **Version-specific:** typeVersion **2.3** (condition editor v3 indicated in params).
- **Edge cases / failures:**
  - **Case sensitivity:** emails like `User@N8N.IO` will not match.
  - **Contains vs domain check:** `someone@notn8n.io.example` would match unintentionally; likewise `n8n.io` anywhere in the string matches.
  - If `Email` is missing/null, strict validation may cause condition evaluation issues.

#### Node: isTrusted:true
- **Type / role:** `n8n-nodes-base.set` — enrich item with a new field.
- **Configuration (interpreted):**
  - Adds `isTrusted` = `true`
  - **Include other fields:** enabled (keeps the original Name/Email/Q/A)
- **Connections:**
  - **Output →** `ref`
- **Version-specific:** typeVersion **3.4**
- **Edge cases / failures:**
  - If later nodes rely on `isTrusted` existence, ensure both branches converge (they do via `ref`).

#### Node: isTrusted:false
- **Type / role:** `n8n-nodes-base.set`
- **Configuration (interpreted):**
  - Adds `isTrusted` = `false`
  - **Include other fields:** enabled
- **Connections:**
  - **Output →** `ref`
- **Version-specific:** typeVersion **3.4**
- **Edge cases / failures:** same considerations as `isTrusted:true`.

#### Node: ref
- **Type / role:** `n8n-nodes-base.noOp` — “join/reference anchor” used to converge branches without changing data.
- **Configuration (interpreted):** No operation; used so downstream nodes can reliably reference a single node (`ref`) regardless of branch.
- **Connections:**
  - **Output →** `Add tags`
- **Version-specific:** typeVersion **1**
- **Edge cases / failures:**
  - Generally safe; main risk is structural—if you bypass it, expressions like `$('ref').item.json...` downstream must be updated.

---

### Block 1.3 — AI Enrichment (Tag Generation)

**Overview:** Feeds Question and Answer into an OpenAI chat model with a system-style instruction to generate 4–8 comma-separated tags.

**Nodes involved:**
- **OpenAI Chat Model**
- **Add tags**

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides an LLM “languageModel” connection for LangChain nodes.
- **Configuration (interpreted):**
  - Model: **`gpt-5-mini`**
  - Uses OpenAI API credential: **“n8n free OpenAI API credits”**
  - No extra options enabled in the node.
- **Connections:**
  - **AI (languageModel) output →** `Add tags` (as its LLM provider)
- **Version-specific:** typeVersion **1.3** (requires the LangChain nodes package in your n8n instance).
- **Edge cases / failures:**
  - Credential missing/invalid, quota exceeded, or model not available in the account/region.
  - Rate limiting/timeouts on OpenAI side.
  - If the model name changes/deprecates, node will fail until updated.

#### Node: Add tags
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — runs an LLM chain/prompt to generate tags.
- **Configuration (interpreted):**
  - Prompt text (user content):
    - `Analyze the following question and answer and output relevant tags`
    - Injects:
      - `Question: {{ $json.Question }}`
      - `Answer: {{ $json.Answer }}`
  - Instruction message (system-style guidance):
    - “You are a content tagging expert… Always provide 4–8 tags as a comma separated list… Keep tags short…”
    - Includes an example output.
  - Prompt type: **define**
- **Key outputs:**
  - Produces a field containing the model’s completion; in this workflow it is referenced as **`$json.text`** downstream (so the node outputs `text` with the tags string).
- **Connections:**
  - **Main output →** `Insert row`
  - Receives **AI languageModel input** from `OpenAI Chat Model`
- **Version-specific:** typeVersion **1.9**
- **Edge cases / failures:**
  - Model may return tags not strictly comma-separated or outside 4–8 range (LLMs can drift).
  - If `Question`/`Answer` are empty, tags may be generic or blank.
  - If the node output schema changes (e.g., not `text`), the Data Table mapping will break.

---

### Block 1.4 — Persistence (Data Table Insert)

**Overview:** Inserts a new record into the `q&a` Data Table combining original submission fields with generated tags and trust flag.

**Nodes involved:**
- **Insert row**

#### Node: Insert row
- **Type / role:** `n8n-nodes-base.dataTable` — inserts a row into an n8n Data Table.
- **Configuration (interpreted):**
  - Operation: **Insert**
  - Target Data Table: **`q&a`**
  - Column mapping mode: explicit “define below”
  - Columns written:
    - `Name` = `{{ $('ref').item.json.Name }}`
    - `Email` = `{{ $('ref').item.json.Email }}`
    - `Question` = `{{ $('ref').item.json.Question }}`
    - `Answer` = `{{ $('ref').item.json.Answer }}`
    - `Tags` = `{{ $json.text }}` (from **Add tags** output)
    - `isTrusted` = `{{ $('ref').item.json.isTrusted }}`
  - Type conversion: disabled (`attemptToConvertTypes: false`)
- **Connections:** terminal node (no outputs used).
- **Version-specific:** typeVersion **1.1** (Data Tables are an n8n feature; availability depends on plan/version).
- **Edge cases / failures:**
  - Data Table missing/renamed or insufficient permissions.
  - Schema mismatch (e.g., column type changes; `isTrusted` must remain boolean).
  - If `$('ref')...` cannot be resolved (node renamed/deleted), expressions fail.
  - If `Add tags` returns a structured object rather than `text`, Tags mapping will be wrong.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Collect Q&A submissions | — | is n8n.io email? | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| is n8n.io email? | n8n-nodes-base.if | Branch by email domain trust check | On form submission | isTrusted:true; isTrusted:false | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| isTrusted:true | n8n-nodes-base.set | Set `isTrusted=true` | is n8n.io email? (true branch) | ref | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| isTrusted:false | n8n-nodes-base.set | Set `isTrusted=false` | is n8n.io email? (false branch) | ref | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| ref | n8n-nodes-base.noOp | Merge branches; stable reference for expressions | isTrusted:true; isTrusted:false | Add tags | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for tag generation | — | Add tags (ai_languageModel) | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| Add tags | @n8n/n8n-nodes-langchain.chainLlm | Generate comma-separated tags from Q&A | ref (main); OpenAI Chat Model (ai_languageModel) | Insert row | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| Insert row | n8n-nodes-base.dataTable | Persist enriched submission to Data Table | Add tags | — | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## QA Ingest Template; ### Flow 1/2; This workflow lets users submit Question and Answer pairs, automatically enriches them by flagging trusted submissions based on email validation and using an LLM to generate relevant tags, then stores everything in an n8n Data Table.; By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“QA Ingest [quickstart 2026]”** (or your preferred name).

2. **Add trigger: Form Trigger**
   - Add node: **Form Trigger** (`On form submission`)
   - Configure:
     - Form Title: **QA Submission**
     - Fields:
       1) Name (text)
       2) Email (text)
       3) Question (text)
       4) Answer (textarea)
   - Save and (if needed) publish/activate the form so it can accept submissions.

3. **Add an IF node for email trust**
   - Add node: **IF** (`is n8n.io email?`)
   - Condition:
     - String → **contains**
     - Value 1 (left): `{{$json.Email}}`
     - Value 2 (right): `n8n.io`
     - Case sensitive: **on** (to match the workflow exactly)
   - Connect: `On form submission` → `is n8n.io email?`

4. **Add Set nodes to write `isTrusted`**
   - Add node: **Set** named `isTrusted:true`
     - Add field: `isTrusted` (Boolean) = **true**
     - Enable **Include Other Fields**
   - Add node: **Set** named `isTrusted:false`
     - Add field: `isTrusted` (Boolean) = **false**
     - Enable **Include Other Fields**
   - Connect:
     - IF **true** output → `isTrusted:true`
     - IF **false** output → `isTrusted:false`

5. **Add a NoOp merge anchor**
   - Add node: **No Operation (NoOp)** named `ref`
   - Connect:
     - `isTrusted:true` → `ref`
     - `isTrusted:false` → `ref`
   - (This provides a stable node name to reference for original fields downstream.)

6. **Add the OpenAI Chat Model (LangChain)**
   - Add node: **OpenAI Chat Model** (LangChain) named `OpenAI Chat Model`
   - Select model: **gpt-5-mini**
   - Credentials:
     - Configure an **OpenAI API** credential (API key) and select it in the node.
   - No connection from main stream is required; it will connect via the AI port.

7. **Add the LLM Chain node to generate tags**
   - Add node: **LLM Chain** (LangChain) named `Add tags`
   - Configure prompt (user text) similar to:
     - `Analyze the following question and answer and output relevant tags`
     - `Question: {{ $json.Question }}`
     - `Answer: {{ $json.Answer }}`
   - Add an instruction message (system guidance) requiring:
     - 4–8 tags
     - comma-separated list
     - short focused tags
   - Connect:
     - Main: `ref` → `Add tags`
     - AI Model connection: `OpenAI Chat Model` → `Add tags` (to its **ai_languageModel** input)

8. **Create / select the Data Table**
   - In n8n Data Tables, create a table named **`q&a`** (or use an existing one) with columns:
     - `Name` (string)
     - `Email` (string)
     - `Question` (string)
     - `Answer` (string)
     - `Tags` (string)
     - `isTrusted` (boolean)

9. **Add Data Table node to insert rows**
   - Add node: **Data Table** named `Insert row`
   - Operation: **Insert row**
   - Table: select **`q&a`**
   - Map columns explicitly:
     - `Name` = `{{ $('ref').item.json.Name }}`
     - `Email` = `{{ $('ref').item.json.Email }}`
     - `Question` = `{{ $('ref').item.json.Question }}`
     - `Answer` = `{{ $('ref').item.json.Answer }}`
     - `isTrusted` = `{{ $('ref').item.json.isTrusted }}`
     - `Tags` = `{{ $json.text }}`
   - Connect: `Add tags` → `Insert row`

10. **Activate and test**
   - Submit a form entry.
   - Verify:
     - `isTrusted` is set correctly based on email containing `n8n.io`
     - `Tags` is a comma-separated string
     - A new row appears in the Data Table.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| QA Ingest Template — Flow 1/2. Captures Q&A, flags trusted submissions by email validation, generates tags with an LLM, stores in n8n Data Table. | Sticky note covering the workflow |
| By Max Tkacz \| The Original Flowgrammer | https://www.linkedin.com/in/maxtkacz/ |