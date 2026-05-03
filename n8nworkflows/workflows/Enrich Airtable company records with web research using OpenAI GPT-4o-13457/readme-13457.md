Enrich Airtable company records with web research using OpenAI GPT-4o

https://n8nworkflows.xyz/workflows/enrich-airtable-company-records-with-web-research-using-openai-gpt-4o-13457


# Enrich Airtable company records with web research using OpenAI GPT-4o

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Enrich Airtable company records with web research using OpenAI GPT-4o

This workflow monitors an Airtable “Companies” table for newly created records. When a new company is added, it uses OpenAI (GPT‑4o with built-in web search) to research the company, extracting the official website and generating a short (1–2 sentence) description. It then writes the enrichment back into the same Airtable record.

### 1.1 Input Reception (Airtable trigger)
Detects new records based on a “Created time” field.

### 1.2 AI Web Research + Structured Output
Calls GPT‑4o with web search enabled, requesting a strict JSON response containing `website` and `summary`.

### 1.3 Persistence (Update Airtable record)
Updates the originating Airtable record with the AI-produced website and summary.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (New Airtable record)

**Overview:** Polls Airtable for new company records and emits each new record as an item to downstream nodes.

**Nodes Involved:**
- **New Company Trigger** (Airtable Trigger)

#### Node: New Company Trigger
- **Type / Role:** `n8n-nodes-base.airtableTrigger` — polling trigger for new/changed Airtable records.
- **Configuration (interpreted):**
  - **Base/Table selection:** Uses Airtable base/table/view URLs (provided via URL mode).
  - **Polling:** Runs **every minute**.
  - **Trigger field:** `Created` (must be a Created time field or a field that changes when created).
  - **Auth method:** Airtable Personal Access Token (“airtableTokenApi”).
- **Key data produced:**
  - Emits items containing record metadata including `id` and `fields` (notably `fields.Name`).
- **Connections:**
  - **Output →** Research Company with AI
- **Version-specific notes:**
  - **typeVersion: 1** — standard Airtable Trigger behavior; ensure your n8n instance supports Airtable PAT auth for this node version.
- **Edge cases / failure modes:**
  - **Misconfigured triggerField:** If `Created` doesn’t exist or is not populated, new records may not be detected.
  - **Permissions/scopes:** Insufficient token scopes can prevent reading records or base schema.
  - **Polling latency:** Up to ~1 minute delay; duplicates can occur if the trigger logic re-detects records (rare, but possible depending on Airtable view/field behavior).

---

### Block 2 — AI Web Research + Structured Output

**Overview:** Uses GPT‑4o with web search to research the company name and return a machine-readable JSON object containing the company website and a short summary.

**Nodes Involved:**
- **Research Company with AI** (OpenAI / LangChain node)

#### Node: Research Company with AI
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call with optional built-in tools (web search).
- **Configuration (interpreted):**
  - **Model:** `gpt-4o`
  - **Built-in tool enabled:** **webSearch** with `searchContextSize: "medium"` (LLM can consult web sources).
  - **Prompt content (dynamic):**
    - Uses Airtable company name: `{{ $json.fields.Name }}`
    - Instruction: “Look at up to 2-3 sources.”
    - Output requirement: JSON with keys `website` and `summary`
  - **Structured output enforcement:**
    - Uses **JSON Schema** output formatting:
      - Object with required string fields: `website`, `summary`
      - `additionalProperties: false` (disallows extra keys)
- **Key expressions / variables:**
  - `{{ $json.fields.Name }}` — reads the incoming Airtable record’s company name.
- **Connections:**
  - **Input ←** New Company Trigger
  - **Output →** Update Company Record
- **Version-specific notes:**
  - **typeVersion: 2.1** — relies on the newer OpenAI/LangChain node features including schema-based output formatting and built-in web search.
- **Edge cases / failure modes:**
  - **Missing company name:** If `fields.Name` is empty/null, results may be irrelevant or the model may respond with low confidence.
  - **Schema compliance failures:** The model may occasionally fail to match schema (less likely with schema mode, but still possible). This can break downstream expressions expecting `website/summary`.
  - **Web search availability/limits:** Tool availability may depend on n8n/OpenAI capabilities, quotas, or account settings.
  - **Ambiguous company names:** Might retrieve the wrong entity; consider adding location/domain hints in Airtable to reduce ambiguity.

---

### Block 3 — Persistence (Update Airtable record)

**Overview:** Writes enrichment results (website + summary) back into the same Airtable “Companies” record that triggered the workflow.

**Nodes Involved:**
- **Update Company Record** (Airtable)

#### Node: Update Company Record
- **Type / Role:** `n8n-nodes-base.airtable` — updates Airtable records.
- **Configuration (interpreted):**
  - **Operation:** `update`
  - **Base:** `Contacts` (base id `appXXXXXXXXXXXXXX`)
  - **Table:** `Companies` (table id `tblXXXXXXXXXXXXXX`)
  - **Matching / Record targeting:**
    - Matches by column `id`
    - Sets `id` to the triggering Airtable record id:
      - `id = {{ $('New Company Trigger').item.json.id }}`
  - **Fields updated:**
    - **Website**:
      - `{{ $json.output[0].content[0].text.website }}`
    - **Company Details**:
      - `{{ $json.output[0].content[0].text.summary }}`
  - **Mapping mode:** “define below” (explicit field mapping)
- **Key expressions / variables:**
  - `{{ $('New Company Trigger').item.json.id }}` — ensures the update targets the correct original record even if the current node input is from the AI node.
  - `{{ $json.output[0].content[0].text.website }}` and `.summary` — extracts structured fields from the AI node output.
- **Connections:**
  - **Input ←** Research Company with AI
  - **Output →** (none)
- **Version-specific notes:**
  - **typeVersion: 2.1** — updated Airtable node; field mapping behavior may differ from older versions.
- **Edge cases / failure modes:**
  - **Expression path mismatch:** If the OpenAI node output structure differs (or schema fails), `$json.output[0]...` may be undefined and the node will error.
  - **Field name mismatch:** Airtable fields must exist exactly as “Website” and “Company Details” (case-sensitive in mapping). If renamed, update will fail or silently not write as expected.
  - **Permissions/scopes:** Requires `data.records:write` and usually `schema.bases:read` for dynamic field mapping.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Company Trigger | n8n-nodes-base.airtableTrigger | Detect newly created company records in Airtable (polling) | — | Research Company with AI | ## Workflow Overview\n\nThis workflow automatically researches companies and enriches your Airtable database whenever a new company is added. The AI agent searches the web to find the company's website and generates a concise 1-2 sentence summary of their business activities, then updates the Airtable record with these findings.\n\n### First Setup\n\n**Airtable Configuration:**\n- Ensure your Companies table has a **Created time** field (this triggers the workflow)\n- Create an Airtable Personal Access Token at https://airtable.com/create/tokens\n- Grant these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`\n- Add the credential in n8n under the Airtable Personal Access Token API option\n\n**OpenAI Setup:**\n- Register for an OpenAI account and obtain an API key\n- Add your OpenAI credentials in n8n\n\n**Workflow Configuration:**\n- Update the Airtable Trigger node with your own base and table URLs\n- Update the Airtable Update node with your base and table information\n- Ensure your table has fields for **Website** and **Company Details** (or modify the field mappings to match your schema)\n\nOnce published, the workflow runs automatically whenever a new company is added to your table. |
| Research Company with AI | @n8n/n8n-nodes-langchain.openAi | Web research + structured JSON extraction (website, summary) | New Company Trigger | Update Company Record | ## Workflow Overview\n\nThis workflow automatically researches companies and enriches your Airtable database whenever a new company is added. The AI agent searches the web to find the company's website and generates a concise 1-2 sentence summary of their business activities, then updates the Airtable record with these findings.\n\n### First Setup\n\n**Airtable Configuration:**\n- Ensure your Companies table has a **Created time** field (this triggers the workflow)\n- Create an Airtable Personal Access Token at https://airtable.com/create/tokens\n- Grant these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`\n- Add the credential in n8n under the Airtable Personal Access Token API option\n\n**OpenAI Setup:**\n- Register for an OpenAI account and obtain an API key\n- Add your OpenAI credentials in n8n\n\n**Workflow Configuration:**\n- Update the Airtable Trigger node with your own base and table URLs\n- Update the Airtable Update node with your base and table information\n- Ensure your table has fields for **Website** and **Company Details** (or modify the field mappings to match your schema)\n\nOnce published, the workflow runs automatically whenever a new company is added to your table. |
| Update Company Record | n8n-nodes-base.airtable | Update the triggering Airtable record with enrichment results | Research Company with AI | — | ## Workflow Overview\n\nThis workflow automatically researches companies and enriches your Airtable database whenever a new company is added. The AI agent searches the web to find the company's website and generates a concise 1-2 sentence summary of their business activities, then updates the Airtable record with these findings.\n\n### First Setup\n\n**Airtable Configuration:**\n- Ensure your Companies table has a **Created time** field (this triggers the workflow)\n- Create an Airtable Personal Access Token at https://airtable.com/create/tokens\n- Grant these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`\n- Add the credential in n8n under the Airtable Personal Access Token API option\n\n**OpenAI Setup:**\n- Register for an OpenAI account and obtain an API key\n- Add your OpenAI credentials in n8n\n\n**Workflow Configuration:**\n- Update the Airtable Trigger node with your own base and table URLs\n- Update the Airtable Update node with your base and table information\n- Ensure your table has fields for **Website** and **Company Details** (or modify the field mappings to match your schema)\n\nOnce published, the workflow runs automatically whenever a new company is added to your table. |
| Workflow Description | n8n-nodes-base.stickyNote | Documentation / setup guidance displayed on canvas | — | — | ## Workflow Overview\n\nThis workflow automatically researches companies and enriches your Airtable database whenever a new company is added. The AI agent searches the web to find the company's website and generates a concise 1-2 sentence summary of their business activities, then updates the Airtable record with these findings.\n\n### First Setup\n\n**Airtable Configuration:**\n- Ensure your Companies table has a **Created time** field (this triggers the workflow)\n- Create an Airtable Personal Access Token at https://airtable.com/create/tokens\n- Grant these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`\n- Add the credential in n8n under the Airtable Personal Access Token API option\n\n**OpenAI Setup:**\n- Register for an OpenAI account and obtain an API key\n- Add your OpenAI credentials in n8n\n\n**Workflow Configuration:**\n- Update the Airtable Trigger node with your own base and table URLs\n- Update the Airtable Update node with your base and table information\n- Ensure your table has fields for **Website** and **Company Details** (or modify the field mappings to match your schema)\n\nOnce published, the workflow runs automatically whenever a new company is added to your table. |
| Video Walkthrough | n8n-nodes-base.stickyNote | Embedded link/thumbnail for video reference | — | — | # Video Walkthrough\n[https://youtu.be/lQh1fuIrBN8](https://youtu.be/lQh1fuIrBN8) |

---

## 4. Reproducing the Workflow from Scratch

1. **Prepare Airtable**
   1. Create (or open) a base and a table named **Companies** (name can differ, but then adjust node config).
   2. Ensure fields exist:
      - **Name** (single line text) — company name to research
      - **Website** (single line text / URL) — will be filled by the workflow
      - **Company Details** (long text) — will be filled by the workflow
      - **Created** (Created time field) — used as the trigger field

2. **Create Airtable Personal Access Token (PAT)**
   1. Generate a token at: https://airtable.com/create/tokens
   2. Grant scopes:
      - `data.records:read`
      - `data.records:write`
      - `schema.bases:read`
   3. In n8n, add credentials: **Airtable Personal Access Token API** and paste the token.

3. **Create OpenAI credentials**
   1. Obtain an OpenAI API key from your OpenAI account.
   2. In n8n, add credentials for **OpenAI** (node credential: `openAiApi`).

4. **Build the workflow nodes**
   1. Add node **Airtable Trigger** and name it **New Company Trigger**.
      - Set **Authentication** to *Airtable Personal Access Token API* and select your Airtable PAT credential.
      - Select the **Base** and **Table** (or paste URLs in URL mode).
      - Set **Poll Times** to **Every Minute**.
      - Set **Trigger Field** to **Created**.
   2. Add node **OpenAI (LangChain)** and name it **Research Company with AI**.
      - Choose **Model**: `gpt-4o`.
      - Enable **Built-in Tools → Web Search** and set context size to **medium** (if available in your n8n version).
      - In **Responses / Prompt**, use:
        - `Research the following company: {{ $json.fields.Name }}`
        - Ask for website + 1–2 sentence summary, “Look at up to 2-3 sources.”
        - Require JSON keys `website` and `summary`.
      - In **Options → Text format**, set to **json_schema** and define schema requiring:
        - `website` (string)
        - `summary` (string)
        - `additionalProperties: false`
   3. Add node **Airtable** and name it **Update Company Record**.
      - **Operation:** Update
      - **Authentication:** Airtable PAT credential
      - Pick the same **Base** and **Table** used for Companies.
      - In field mapping (“Define below”):
        - Map **id** to: `{{ $('New Company Trigger').item.json.id }}`
        - Map **Website** to: `{{ $json.output[0].content[0].text.website }}`
        - Map **Company Details** to: `{{ $json.output[0].content[0].text.summary }}`
      - Ensure matching is performed by **id** (record identifier).

5. **Connect the nodes**
   1. Connect **New Company Trigger → Research Company with AI**
   2. Connect **Research Company with AI → Update Company Record**

6. **Test**
   1. Manually execute (where possible) or create a new record in Airtable with **Name** populated.
   2. Confirm that **Website** and **Company Details** are written back.
   3. If the update fails, inspect:
      - Whether the OpenAI node output path matches the expressions used in the Airtable update node.
      - Whether fields exist and are writable in Airtable.

7. **Activate**
   - Activate the workflow so it polls every minute and processes new records automatically.

**Sub-workflows:** None (no “Execute Workflow” / sub-workflow invocation nodes present).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create an Airtable Personal Access Token and grant scopes `data.records:read`, `data.records:write`, `schema.bases:read` | https://airtable.com/create/tokens |
| Video Walkthrough | https://youtu.be/lQh1fuIrBN8 |