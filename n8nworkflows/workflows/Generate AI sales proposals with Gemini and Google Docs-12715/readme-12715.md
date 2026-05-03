Generate AI sales proposals with Gemini and Google Docs

https://n8nworkflows.xyz/workflows/generate-ai-sales-proposals-with-gemini-and-google-docs-12715


# Generate AI sales proposals with Gemini and Google Docs

## 1. Workflow Overview

**Title:** Generate sales proposals with Gemini and Google Docs  
**Purpose:** Collect sales proposal inputs via an n8n Form, calculate basic ROI metrics, generate proposal narrative sections using **Gemini (Google Generative AI)**, then **copy a Google Docs template** and replace placeholders with the generated and computed content.

**Typical use cases**
- Sales teams generating consistent, on-brand proposals from minimal discovery notes
- Rapid proposal drafting with financial justification (ROI, savings, break-even)
- Standardized document output using a reusable Google Docs template

### 1.1 Input Reception (Form)
Collects client + deal details (industry, pain points, pricing, dates, team info).

### 1.2 Data Preparation (ROI + formatting)
Computes ROI/savings/break-even and produces “human formatted” currency/percent strings for insertion into the proposal.

### 1.3 AI Generation (Gemini agent)
Uses the form + ROI narrative context to generate proposal sections as a single JSON object.

### 1.4 Output Parsing & Cleanup
Parses the AI JSON from the agent output and formats bullet-point sections for Google Docs insertion.

### 1.5 Document Creation (Google Docs)
Copies a template doc in Google Drive and replaces placeholders with the generated/computed content.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Starts the workflow when a sales rep submits the form. Produces one item containing all submitted fields.

**Nodes involved**
- Sticky Note (documentation)
- Receive proposal details (Form Trigger)

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Explains overall logic and setup steps; includes the required Google Docs placeholders.
- **Connections:** None (documentation-only).
- **Failure modes:** None.
- **Sticky content (important):**
  - Placeholders expected in the template:  
    `{{client_name}}`, `{{executive_summary}}`, `{{key_challenges}}`, `{{solution_strategy}}`, `{{team_bios}}`, `{{formatted_roi}}`, `{{formatted_net_savings}}`, `{{formatted_break_even}}`, `{{next_steps}}`, `{{date}}`
  - Setup guidance: connect Google Drive, Google Docs, Gemini; configure Copy node; customize AI system message.

#### Node: Receive proposal details
- **Type / role:** `n8n-nodes-base.formTrigger` — workflow entry point (interactive form submission).
- **Key configuration:**
  - **Form title:** “Proposal Generation Form”
  - **Fields collected:**
    - `user_company_name` (text)
    - `team_members_input` (textarea)
    - `client_name` (required)
    - `client_industry` (required)
    - `key_pain_points` (textarea, required)
    - `current_annual_spend` (number, required)
    - `offer_type` (dropdown: SEO Retainer, Web Dev; required)
    - `proposed_solution_cost` (number, required)
    - `project_start_date` (date, required)
- **Outputs:** One item with `.json` containing the above fields.
- **Downstream connections:** → **Calculate ROI metrics**
- **Edge cases / failures:**
  - Missing required fields (blocked by form validation).
  - “number” fields may still arrive as strings depending on UI/client; downstream code cleans this.

---

### Block 2 — Data Preparation (ROI Metrics)
**Overview:** Cleans numeric inputs and computes net savings, ROI %, and break-even time. Also produces formatted strings for proposal insertion and a narrative sentence for the AI.

**Nodes involved**
- Sticky Note1 (documentation)
- Calculate ROI metrics (Code)

#### Node: Sticky Note1
- **Type / role:** Sticky Note (documentation)
- **Meaning:** “Data preparation” description (Form → calculations).

#### Node: Calculate ROI metrics
- **Type / role:** `n8n-nodes-base.code` — JavaScript computation of financial metrics.
- **Key configuration choices (interpreted):**
  - Reads first incoming item: `const formData = items[0].json;`
  - Cleans numbers using `cleanNumber()` which strips non-numeric characters.
  - Uses a configurable constant:  
    `efficiencyAssumption = 1.0`  
    (currently assumes 100% of current spend is “savings driver”; adjust to your real model).
- **Core calculations:**
  - `grossSavings = currentSpend * efficiencyAssumption`
  - `netSavings = grossSavings - solutionCost`
  - `roiPercent = (netSavings / solutionCost) * 100` (only if `solutionCost > 0`)
  - `breakEvenMonths = solutionCost / (grossSavings / 12)` (only if `solutionCost > 0`)
- **Outputs (important fields):**
  - `formatted_current_spend`, `formatted_solution_cost`, `formatted_net_savings` (USD, 0 decimals)
  - `formatted_roi` (rounded percent string)
  - `formatted_break_even` (one decimal + “ Months”)
  - `ai_financial_context` (sentence embedding ROI + break-even)
- **Input node:** Receive proposal details
- **Output node:** Generate proposal content
- **Edge cases / failures:**
  - `solutionCost = 0` → ROI and break-even remain 0 (ROI shown as `0%`, break-even `0.0 Months`).
  - If `currentSpend = 0` with non-zero solution cost → break-even calculation becomes division by zero risk; current code guards only on solutionCost, not on grossSavings. If `grossSavings` is 0, `breakEvenMonths` becomes `Infinity`. Consider guarding with `if (solutionCost > 0 && grossSavings > 0)`.
  - Negative net savings yields negative ROI; formatting still works but may be undesirable in sales output.

---

### Block 3 — AI Generation (Gemini)
**Overview:** Sends the client details + computed financial context to a LangChain Agent configured with Gemini 1.5 Flash. The agent must output a JSON object with specific keys.

**Nodes involved**
- Sticky Note2 (documentation)
- Generate proposal content (LangChain Agent)
- Gemini 1.5 Flash (Gemini chat model)

#### Node: Sticky Note2
- **Type / role:** Sticky Note (documentation)
- **Meaning:** “AI generation” description and parsing note.

#### Node: Generate proposal content
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM prompt + response.
- **Key configuration choices:**
  - **Prompt (“text”) is dynamically composed** using n8n expressions:
    - Client name, pain points from **Receive proposal details**
    - Formatted ROI metrics + `ai_financial_context` from **Calculate ROI metrics**
  - **System message** enforces:
    - Tone and financial tie-back requirements
    - Team section rules (narrative text block with dash list; do not invent company name)
    - **Strict JSON output** with keys:
      - `short_outcome_phrase`
      - `executive_summary`
      - `key_challenges_bullet_points`
      - `solution_strategy`
      - `team_bio_section`
      - `call_to_action`
  - **PromptType:** “define” (custom prompt).
- **Model connection:** Uses **Gemini 1.5 Flash** as `ai_languageModel` input.
- **Input node:** Calculate ROI metrics
- **Output node:** Parse AI response
- **Edge cases / failures:**
  - Model may output extra text, markdown fences, or invalid JSON → handled downstream by Parse AI response.
  - If expressions reference missing fields (e.g., user_company_name empty), prompt still runs but output quality may degrade.
  - If Gemini credentials/API are misconfigured → node fails (auth / quota / permissions).

#### Node: Gemini 1.5 Flash
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — LLM provider node.
- **Credentials:** `googlePalmApi` (“Gemini n8n”)
- **Connections:** Provides the language model to Generate proposal content (AI language model connection, not main flow).
- **Edge cases / failures:**
  - API key invalid/revoked, project not enabled, quota exceeded, region restrictions.
  - Model availability changes (Gemini version deprecations).

---

### Block 4 — Parse & Prepare Text for Docs
**Overview:** Extracts JSON from the agent output and prepares bullet formatting for insertion into Google Docs.

**Nodes involved**
- Parse AI response (Code)
- Format bullet points (Code)

#### Node: Parse AI response
- **Type / role:** `n8n-nodes-base.code` — JSON extraction and parsing.
- **Key configuration choices:**
  - Reads raw model output from:  
    `$('Generate proposal content').first().json.output`
  - Extracts the first `{ ... }` block via regex: `/\{[\s\S]*\}/`
  - `JSON.parse()` to convert to object.
  - On failure returns:
    - `error: "Failed to parse JSON"`
    - `message`
    - `raw_ai_response`
- **Input node:** Generate proposal content
- **Output node:** Copy proposal template
- **Edge cases / failures:**
  - If AI output contains multiple JSON objects or braces in text, regex may capture too much.
  - If AI returns JSON with trailing commas or comments → `JSON.parse` fails.
  - If parsing fails, downstream nodes may still run but will receive an “error object” instead of expected keys (likely causing placeholder replacement issues).

#### Node: Format bullet points
- **Type / role:** `n8n-nodes-base.code` — normalizes bullet lists.
- **Key configuration choices:**
  - Reads parsed data from: `$('Parse AI response').first().json`
  - `formatAsBullets(input)`:
    - If `input` is an array → ensures each item has no leading `- • *`, then joins into:
      ```
      • item1
      • item2
      ```
    - If `input` is a string → returns it unchanged.
  - Produces:
    - `ready_challenges`
    - `ready_bios`
- **Input node:** Copy proposal template
- **Output node:** Populate proposal document
- **Important note:** In the current workflow, `ready_bios` is produced but **not used** in the Google Docs replacement (the doc uses `team_bio_section` directly).
- **Edge cases / failures:**
  - If Parse AI response returns an error object, `aiData.key_challenges_bullet_points` is undefined → function returns empty strings; proposal may be blank.

---

### Block 5 — Document Creation (Copy + Placeholder Replacement)
**Overview:** Copies a master Google Doc template, then replaces placeholders with computed and AI-generated content.

**Nodes involved**
- Sticky Note3 (documentation)
- Copy proposal template (Google Drive)
- Populate proposal document (Google Docs)

#### Node: Sticky Note3
- **Type / role:** Sticky Note (documentation)
- **Meaning:** “Document creation” description (copy template + placeholder replacement).

#### Node: Copy proposal template
- **Type / role:** `n8n-nodes-base.googleDrive` — duplicates a template document.
- **Operation:** Copy
- **Key configuration choices:**
  - **Template File ID:** A specific Google Doc (“Proposal Template Master”)
  - **New file name:** `={{ client_name }} Proposal`
- **Credentials:** Google Drive OAuth2 (“Daniel Google Drive account”)
- **Input node:** Parse AI response
- **Output node:** Format bullet points
- **Edge cases / failures:**
  - Drive permission issues (template not shared with credential user).
  - Copying a non-Doc file or invalid file ID.
  - Google API rate limits.

#### Node: Populate proposal document
- **Type / role:** `n8n-nodes-base.googleDocs` — updates the copied doc by replacing placeholders.
- **Operation:** Update (replaceAll actions)
- **Document target:** `documentURL` set to the **ID** of the copied file:  
  `={{ $('Copy proposal template').item.json.id }}`
- **Placeholder replacements (interpreted):**
  - `{{client_name}}` ← form `client_name`
  - `{{user_company_name}}` ← form `user_company_name`
  - `{{dynamic_title}}` ← `client_name + user_company_name + short_outcome_phrase`
  - `{{formatted_roi}}` ← calculated `formatted_roi`
  - `{{executive_summary}}` ← AI `executive_summary`
  - `{{key_challenges}}` ← **formatted** `$json.ready_challenges` (from Format bullet points)
  - `{{solution_strategy}}` ← AI `solution_strategy`
  - `{{team_bios}}` ← AI `team_bio_section` (note: not using `$json.ready_bios`)
  - `{{next_steps}}` ← AI `call_to_action`
  - `{{formatted_net_savings}}` ← calculated
  - `{{formatted_solution_cost}}` ← calculated
  - `{{formatted_break_even}}` ← calculated
  - `{{date}}` ← `{{ $now.toFormat('MMMM dd, yyyy') }}`
- **Credentials:** Google Docs OAuth2 (“Daniel Google Docs account”)
- **Input node:** Format bullet points
- **Terminal node:** Yes (no outgoing connections)
- **Edge cases / failures:**
  - If placeholders don’t exist in the template, replaceAll does nothing (silent mismatch risk).
  - If the copied file ID is wrong/empty, the update fails.
  - Large text replacements can hit Google Docs API constraints (rare, but possible for long AI output).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Workflow explanation + setup requirements | — | — | **How it works** + **Setup steps** including required placeholders: `{{client_name}}`, `{{executive_summary}}`, `{{key_challenges}}`, `{{solution_strategy}}`, `{{team_bios}}`, `{{formatted_roi}}`, `{{formatted_net_savings}}`, `{{formatted_break_even}}`, `{{next_steps}}`, `{{date}}` |
| Sticky Note1 | n8n-nodes-base.stickyNote | Notes for ROI/data prep | — | — | **Data preparation** Form collects client info. Code node calculates ROI, savings, and break-even from financials. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Notes for AI generation | — | — | **AI generation** Gemini writes proposal sections based on pain points and financials. Parser extracts JSON fields. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Notes for doc creation | — | — | **Document creation** Copies template, formats bullet points, and replaces all placeholders with generated content. |
| Receive proposal details | n8n-nodes-base.formTrigger | Entry point: collect proposal inputs | — | Calculate ROI metrics | **Data preparation** Form collects client info. Code node calculates ROI, savings, and break-even from financials. |
| Calculate ROI metrics | n8n-nodes-base.code | Compute ROI/savings/break-even + formatted strings | Receive proposal details | Generate proposal content | **Data preparation** Form collects client info. Code node calculates ROI, savings, and break-even from financials. |
| Generate proposal content | @n8n/n8n-nodes-langchain.agent | LLM agent prompt + response generation (JSON) | Calculate ROI metrics | Parse AI response | **AI generation** Gemini writes proposal sections based on pain points and financials. Parser extracts JSON fields. |
| Gemini 1.5 Flash | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Language model provider for agent | — (AI model connection) | Generate proposal content (ai_languageModel) | **AI generation** Gemini writes proposal sections based on pain points and financials. Parser extracts JSON fields. |
| Parse AI response | n8n-nodes-base.code | Extract JSON from agent output; error object on failure | Generate proposal content | Copy proposal template | **AI generation** Gemini writes proposal sections based on pain points and financials. Parser extracts JSON fields. |
| Copy proposal template | n8n-nodes-base.googleDrive | Copy Google Docs template in Drive | Parse AI response | Format bullet points | **Document creation** Copies template, formats bullet points, and replaces all placeholders with generated content. |
| Format bullet points | n8n-nodes-base.code | Normalize bullet formatting for insertion | Copy proposal template | Populate proposal document | **Document creation** Copies template, formats bullet points, and replaces all placeholders with generated content. |
| Populate proposal document | n8n-nodes-base.googleDocs | Replace placeholders in copied doc | Format bullet points | — | **Document creation** Copies template, formats bullet points, and replaces all placeholders with generated content. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Name: `Receive proposal details`
   - Set **Form title**: `Proposal Generation Form`
   - Add fields:
     - `user_company_name` (text)
     - `team_members_input` (textarea)
     - `client_name` (text, required)
     - `client_industry` (text, required)
     - `key_pain_points` (textarea, required)
     - `current_annual_spend` (number, required)
     - `offer_type` (dropdown, required) with options:
       - `SEO Retainer`
       - `Web Dev`
     - `proposed_solution_cost` (number, required)
     - `project_start_date` (date, required)

3. **Add a Code node** for ROI
   - Node type: **Code**
   - Name: `Calculate ROI metrics`
   - Paste logic implementing:
     - numeric cleaning
     - compute net savings, ROI %, break-even months
     - formatted currency/percent strings
     - `ai_financial_context` sentence
   - Connect: `Receive proposal details` → `Calculate ROI metrics`

4. **Add a Gemini model node**
   - Node type: **Google Gemini Chat Model** (Gemini)
   - Name: `Gemini 1.5 Flash`
   - Credentials:
     - Create/choose **Google Palm / Gemini API** credential (API key)
     - Ensure the Google GenAI API is enabled and quota is available

5. **Add a LangChain Agent node**
   - Node type: **AI Agent (LangChain)**
   - Name: `Generate proposal content`
   - Set **Prompt type** to custom/define prompt.
   - In the prompt text, insert expressions for:
     - client name, pain points (from form)
     - ROI strings + `ai_financial_context` (from ROI code node)
   - In the **System message**, require:
     - professional proposal tone
     - strict JSON output with the required keys
     - team bio rules (narrative with dash list, do not invent company name)
   - Connect main flow: `Calculate ROI metrics` → `Generate proposal content`
   - Connect AI model: link `Gemini 1.5 Flash` to the Agent’s **Language Model** input.

6. **Add a Code node** to parse AI JSON
   - Node type: **Code**
   - Name: `Parse AI response`
   - Implement:
     - read `Generate proposal content` output text
     - regex extract `{...}`
     - `JSON.parse`
     - return error object with raw output if parsing fails
   - Connect: `Generate proposal content` → `Parse AI response`

7. **Add a Google Drive node** to copy the template
   - Node type: **Google Drive**
   - Name: `Copy proposal template`
   - Operation: **Copy**
   - Select your **template Google Doc** file ID
   - New name expression: `{{client_name}} Proposal`
   - Credentials:
     - Google Drive OAuth2 credential with access to the template doc
   - Connect: `Parse AI response` → `Copy proposal template`

8. **Add a Code node** to format bullets
   - Node type: **Code**
   - Name: `Format bullet points`
   - Format `key_challenges_bullet_points` into a `•` list when input is an array
   - Output at least: `ready_challenges` (and optionally `ready_bios`)
   - Connect: `Copy proposal template` → `Format bullet points`

9. **Add a Google Docs node** to replace placeholders
   - Node type: **Google Docs**
   - Name: `Populate proposal document`
   - Operation: **Update**
   - Document: set to the copied doc ID from the Drive node output
   - Add **Replace All** actions for each placeholder, including:
     - `{{client_name}}`, `{{user_company_name}}`, `{{dynamic_title}}`
     - `{{executive_summary}}`, `{{key_challenges}}`, `{{solution_strategy}}`, `{{team_bios}}`, `{{next_steps}}`
     - `{{formatted_roi}}`, `{{formatted_net_savings}}`, `{{formatted_solution_cost}}`, `{{formatted_break_even}}`
     - `{{date}}` using `$now.toFormat('MMMM dd, yyyy')`
   - Credentials:
     - Google Docs OAuth2 credential with permission to edit the copied document
   - Connect: `Format bullet points` → `Populate proposal document`

10. **Create the Google Docs template**
   - In Google Docs, create “Proposal Template Master”
   - Add placeholders exactly matching what the node replaces (case-sensitive):
     - `{{client_name}}`, `{{user_company_name}}`, `{{dynamic_title}}`, `{{executive_summary}}`, `{{key_challenges}}`, `{{solution_strategy}}`, `{{team_bios}}`, `{{next_steps}}`, `{{formatted_roi}}`, `{{formatted_net_savings}}`, `{{formatted_solution_cost}}`, `{{formatted_break_even}}`, `{{date}}`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Template must contain placeholders like `{{client_name}}`, `{{executive_summary}}`, `{{key_challenges}}`, `{{solution_strategy}}`, `{{team_bios}}`, `{{formatted_roi}}`, `{{formatted_net_savings}}`, `{{formatted_break_even}}`, `{{next_steps}}`, `{{date}}` | From Sticky Note “Setup steps” |
| Connect credentials for Google Drive, Google Docs, and Gemini API; configure the copy node to point to your template | From Sticky Note “Setup steps” |
| Customize the AI system message in “Generate proposal content” to match company tone | From Sticky Note “Setup steps” |
| The workflow parses JSON from AI output using a broad `{...}` regex; if the model returns extra braces/text, parsing can fail | Implementation detail in “Parse AI response” |
| Efficiency assumption is hard-coded to `1.0` in ROI calculations; adjust to match your ROI model | Implementation detail in “Calculate ROI metrics” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.