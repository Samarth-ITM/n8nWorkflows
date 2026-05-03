Screen CVs with OpenAI and PostgreSQL using chained prompts

https://n8nworkflows.xyz/workflows/screen-cvs-with-openai-and-postgresql-using-chained-prompts-13876


# Screen CVs with OpenAI and PostgreSQL using chained prompts

# 1. Workflow Overview

This workflow automates CV screening for an existing job opening using OpenAI and PostgreSQL. It accepts a job ID and a list of candidate IDs, loads the corresponding records from the database, optionally generates a structured job evaluation template, evaluates each candidate through three chained AI prompts, stores the per-candidate analysis, updates processing state, and finally generates a pool-level executive summary once no pending candidates remain.

The workflow is designed for cases where:
- Jobs and candidates are already stored in PostgreSQL
- Candidate CV text has already been extracted into `candidates.cv_text`
- Screening results should be written directly to the database
- No separate backend orchestration layer is desired

## 1.1 Input Reception and Data Loading

The workflow starts from a webhook endpoint. It receives a `job_id` and `candidate_ids`, then queries PostgreSQL to fetch:
- the target job,
- its current `gabarito` (job template),
- and the selected candidate records.

## 1.2 Conditional Job Template Generation

If the job record does not yet contain a `gabarito`, the workflow asks OpenAI to transform the job description into a structured evaluation template. That template is then saved back into the `jobs` table.

## 1.3 Candidate Preparation and Iteration

The workflow normalizes the job and candidate data into one item per candidate, then iterates through the candidates one by one using `Split In Batches`.

## 1.4 Chained Candidate Analysis

For each candidate, three OpenAI prompts are executed in strict sequence:
1. score the CV against the job template,
2. identify strengths and gaps using the score as context,
3. generate personalized interview questions using the prior analysis.

The workflow then parses the outputs into a structured analysis payload.

## 1.5 Persistence and Candidate State Management

The candidate analysis is inserted or updated in `analyses`, the candidate’s status is changed to `processed`, and the database is checked to see whether any `pending` candidates remain for the job.

## 1.6 Final Summary Generation

When pending candidates reach zero, the workflow marks the job as `done`, loads the full analyzed pool, asks OpenAI for an executive summary, and upserts the result into `job_summaries`.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Data Loading

### Overview
This block receives the screening request and loads all required database records for the specified job and selected candidates. It establishes the execution context used by all downstream nodes.

### Nodes Involved
- Receive CVs
- Fetch Job and Candidates

### Node Details

#### Receive CVs
- **Type and technical role:** `n8n-nodes-base.webhook`; entry-point node receiving HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `cv-analyze`
  - Effective endpoint: `/webhook/cv-analyze` in production mode
- **Key expressions or variables used:** None inside the node; downstream nodes use `{{$json.body.job_id}}` and `{{$json.body.candidate_ids}}`.
- **Input and output connections:**
  - Input: none
  - Output: `Fetch Job and Candidates`
- **Version-specific requirements:** Type version `2.1`; standard Webhook node behavior in recent n8n versions.
- **Edge cases or potential failure types:**
  - Invalid HTTP body shape
  - Missing `job_id`
  - Missing or empty `candidate_ids`
  - Non-array `candidate_ids`
  - Webhook not active in production if workflow is inactive
- **Sub-workflow reference:** None

#### Fetch Job and Candidates
- **Type and technical role:** `n8n-nodes-base.postgres`; SQL data retrieval from PostgreSQL.
- **Configuration choices:**
  - Operation: Execute query
  - Query joins `jobs` and `candidates`
  - Filters by:
    - `j.id = {{$json.body.job_id}}`
    - `c.id = ANY(ARRAY[{{ $json.body.candidate_ids.join(',') }}]::int[])`
  - Aggregates candidate IDs and candidate objects into arrays/JSON
  - Returns one grouped row per job
- **Key expressions or variables used:**
  - `{{ $json.body.job_id }}`
  - `{{ $json.body.candidate_ids.join(',') }}`
- **Input and output connections:**
  - Input: `Receive CVs`
  - Output: `Job Template exists?`
- **Version-specific requirements:** Type version `2.5`; query execution support required.
- **Edge cases or potential failure types:**
  - PostgreSQL credential/authentication failure
  - Empty `candidate_ids` causing malformed SQL
  - Non-numeric IDs causing SQL casting issues
  - No matching job/candidate records leading to zero output items
  - Candidate IDs from another job are silently excluded by the join condition
  - Manual string interpolation in SQL can be brittle
- **Sub-workflow reference:** None

---

## 2.2 Conditional Job Template Generation

### Overview
This block determines whether the job already has a structured template (`gabarito`). If not, it uses OpenAI to extract one from the job description and persists it to PostgreSQL.

### Nodes Involved
- Job Template exists?
- Prompt 0 — Extract Job Template
- Save Job Template

### Node Details

#### Job Template exists?
- **Type and technical role:** `n8n-nodes-base.if`; conditional branching.
- **Configuration choices:**
  - Tests whether `{{$json.gabarito}}` is not empty
  - True branch: skip extraction and proceed directly
  - False branch: generate template
- **Key expressions or variables used:**
  - `={{ $json.gabarito }}`
- **Input and output connections:**
  - Input: `Fetch Job and Candidates`
  - True output: `Prepare Candidates`
  - False output: `Prompt 0 — Extract Job Template`
- **Version-specific requirements:** Type version `2.2`; conditions v2 format.
- **Edge cases or potential failure types:**
  - If `gabarito` exists but is malformed JSON, the block still treats it as valid
  - Empty object vs null behavior depends on node evaluation semantics
- **Sub-workflow reference:** None

#### Prompt 0 — Extract Job Template
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; OpenAI chat/completion step to create a structured job template.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - System prompt enforces HR-specialist behavior and strict JSON-only output
  - User prompt sends the job description and expected JSON schema:
    - `mandatory_requirements`
    - `differential_requirements`
    - `behavioral_competencies`
    - `seniority_level`
    - `area`
    - `weights`
  - Instructs that weights must sum to `1.0`
- **Key expressions or variables used:**
  - `{{ $('Fetch Job and Candidates').item.json.description }}`
- **Input and output connections:**
  - Input: `Job Template exists?` false branch
  - Output: `Save Job Template`
- **Version-specific requirements:** Type version `2.1`; requires configured OpenAI credentials and compatible LangChain/OpenAI node availability in the n8n instance.
- **Edge cases or potential failure types:**
  - OpenAI auth or quota errors
  - Model unavailability
  - Non-JSON or malformed JSON output despite prompt constraints
  - Weights not summing to 1.0
  - Hallucinated structure or missing fields
- **Sub-workflow reference:** None

#### Save Job Template
- **Type and technical role:** `n8n-nodes-base.postgres`; updates the job record with the extracted template.
- **Configuration choices:**
  - Operation: Execute query
  - SQL updates `jobs.gabarito` using the raw AI text cast to `jsonb`
  - Returns the stored `gabarito`
- **Key expressions or variables used:**
  - `{{ $json.output[0].content[0].text }}`
  - `{{ $('Fetch Job and Candidates').item.json.job_id }}`
- **Input and output connections:**
  - Input: `Prompt 0 — Extract Job Template`
  - Output: `Prepare Candidates`
- **Version-specific requirements:** Type version `2.5`.
- **Edge cases or potential failure types:**
  - Invalid JSON causes PostgreSQL cast failure
  - SQL string quoting risks if the AI output contains unexpected characters
  - Database connectivity issues
- **Sub-workflow reference:** None

---

## 2.3 Candidate Preparation and Iteration

### Overview
This block consolidates the final job template source and emits one workflow item per candidate. It then enters a loop that processes candidates sequentially.

### Nodes Involved
- Prepare Candidates
- Loop Candidates

### Node Details

#### Prepare Candidates
- **Type and technical role:** `n8n-nodes-base.code`; transforms aggregated query output into candidate-level items.
- **Configuration choices:**
  - Reads the first item from `Fetch Job and Candidates`
  - Uses existing `gabarito` if present
  - Otherwise reads Prompt 0 output and parses it with `JSON.parse`
  - Normalizes `candidates` whether it arrives as stringified JSON or native JSON
  - Returns one item per candidate with:
    - `job_id`
    - `job_title`
    - `gabarito`
    - `candidate_id`
    - `candidate_name`
    - `cv_text`
- **Key expressions or variables used:**
  - `$('Fetch Job and Candidates').first().json`
  - `$('Prompt 0 — Extract Job Template').first().json.output[0].content[0].text`
  - `JSON.parse(raw)`
- **Input and output connections:**
  - Input: from either `Job Template exists?` true branch or `Save Job Template`
  - Output: `Loop Candidates`
- **Version-specific requirements:** Type version `2`; JavaScript Code node.
- **Edge cases or potential failure types:**
  - No item from `Fetch Job and Candidates`
  - Prompt 0 output not valid JSON
  - `jobData.candidates` undefined
  - Candidate `cv_text` missing or null
- **Sub-workflow reference:** None

#### Loop Candidates
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; loop controller for sequential item processing.
- **Configuration choices:**
  - Uses default batch behavior; effectively processes one candidate per cycle
  - First output is unused in this workflow
  - Second output goes to the candidate-processing branch
  - Loop continuation occurs by connecting the "not all processed" path back into this node
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `Prepare Candidates`, and loopback from `All Candidates Processed?` false branch
  - Output 1: unused
  - Output 2: `Prompt 1 — Score`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Misunderstanding of `Split In Batches` output semantics can break looping if recreated incorrectly
  - Empty input list means no candidate analysis branch executes
- **Sub-workflow reference:** None

---

## 2.4 Chained Candidate Analysis

### Overview
This is the core AI screening pipeline. Each candidate goes through three dependent prompts, where each prompt uses prior outputs as context to deepen the analysis.

### Nodes Involved
- Prompt 1 — Score
- Prompt 2 — Gaps
- Prompt 3 — Interview Questions
- Build Analysis Payload

### Node Details

#### Prompt 1 — Score
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; AI scoring of the candidate.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - System prompt defines strict scoring discipline and JSON-only output
  - User prompt includes:
    - full job template
    - CV text
    - calibration anchors for score bands
    - instructions to compute final score and per-criteria scores
  - Output schema:
    - `score`
    - `adherence_level`
    - `justification`
    - `criteria_scores`
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json.gabarito) }}`
  - `{{ $json.cv_text }}`
- **Input and output connections:**
  - Input: `Loop Candidates`
  - Output: `Prompt 2 — Gaps`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Invalid or oversized CV text
  - Non-JSON output
  - Missing expected keys
  - Token/context limits for very long CVs
  - Model inconsistencies in score calibration
- **Sub-workflow reference:** None

#### Prompt 2 — Gaps
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; extracts strengths and gap analysis using the scoring result as context.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Prompt asks for:
    - exact evidence-backed strengths
    - mandatory critical gaps
    - secondary gaps
  - Consumes both the candidate CV and Prompt 1 JSON text
- **Key expressions or variables used:**
  - `{{ JSON.stringify($('Loop Candidates').item.json.gabarito) }}`
  - `{{ $('Loop Candidates').item.json.cv_text }}`
  - `{{ $('Prompt 1 — Score').item.json.output[0].content[0].text }}`
- **Input and output connections:**
  - Input: `Prompt 1 — Score`
  - Output: `Prompt 3 — Interview Questions`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Referenced upstream item mismatch if node execution order is altered
  - Invalid JSON from Prompt 1 affecting prompt quality
  - Non-JSON output from this node
- **Sub-workflow reference:** None

#### Prompt 3 — Interview Questions
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; generates personalized interview questions from prior analysis.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Prompt includes:
    - job template
    - candidate name
    - score output
    - full gaps analysis
  - Requires 5–7 questions
  - Enforces varied question types and distinct objectives
- **Key expressions or variables used:**
  - `{{ JSON.stringify($('Loop Candidates').item.json.gabarito) }}`
  - `{{ $('Loop Candidates').item.json.candidate_name }}`
  - `{{ $('Prompt 1 — Score').item.json.output[0].content[0].text }}`
  - `{{ $('Prompt 2 — Gaps').item.json.output[0].content[0].text }}`
- **Input and output connections:**
  - Input: `Prompt 2 — Gaps`
  - Output: `Build Analysis Payload`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Non-JSON output
  - Wrong number of questions
  - Missing fields inside returned interview question objects
- **Sub-workflow reference:** None

#### Build Analysis Payload
- **Type and technical role:** `n8n-nodes-base.code`; parses AI outputs and converts them into database-ready fields.
- **Configuration choices:**
  - Reads current candidate context from `Loop Candidates`
  - Parses three AI outputs with `JSON.parse`
  - Maps English AI fields into Portuguese-oriented database column names:
    - `adherence_level` → `nivel_aderencia`
    - `justification` → `justificativa_score`
    - `criteria_scores` → `score_criterios`
    - `strengths` → `pontos_fortes`
    - `critical_gaps` → `gaps_criticos`
    - `secondary_gaps` → `gaps_secundarios`
    - `interview_questions` → `perguntas_entrevista`
- **Key expressions or variables used:**
  - `$('Loop Candidates').item.json`
  - `JSON.parse($('Prompt 1 — Score').item.json.output[0].content[0].text)`
  - `JSON.parse($('Prompt 2 — Gaps').item.json.output[0].content[0].text)`
  - `JSON.parse($('Prompt 3 — Interview Questions').item.json.output[0].content[0].text)`
- **Input and output connections:**
  - Input: `Prompt 3 — Interview Questions`
  - Output: `Save Analysis`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Any invalid JSON from AI nodes causes runtime failure
  - Missing fields produce undefined payload values
  - Type mismatches, especially if score is not numeric
- **Sub-workflow reference:** None

---

## 2.5 Persistence and Candidate State Management

### Overview
This block writes the candidate analysis, marks the candidate as processed, and checks whether there are any pending candidates left for the same job.

### Nodes Involved
- Save Analysis
- Update Candidate Status
- Check Pending Candidates
- All Candidates Processed?

### Node Details

#### Save Analysis
- **Type and technical role:** `n8n-nodes-base.postgres`; upserts per-candidate analysis results.
- **Configuration choices:**
  - Operation: Execute query
  - Inserts into `analyses`
  - Uses `ON CONFLICT (candidate_id) DO UPDATE`
  - Serializes arrays/objects to `jsonb`
  - Escapes single quotes in text justification
- **Key expressions or variables used:**
  - `{{ $json.candidate_id }}`
  - `{{ $json.job_id }}`
  - `{{ $json.score }}`
  - `{{ $json.nivel_aderencia }}`
  - `{{ $json.justificativa_score.replace(/'/g, "''") }}`
  - `{{ JSON.stringify($json.score_criterios) }}`
  - `{{ JSON.stringify($json.pontos_fortes) }}`
  - `{{ JSON.stringify($json.gaps_criticos) }}`
  - `{{ JSON.stringify($json.gaps_secundarios) }}`
  - `{{ JSON.stringify($json.perguntas_entrevista) }}`
- **Input and output connections:**
  - Input: `Build Analysis Payload`
  - Output: `Update Candidate Status`
- **Version-specific requirements:** Type version `2.5`.
- **Edge cases or potential failure types:**
  - SQL string construction can fail on unusual characters not fully escaped
  - JSON serialization errors if payload contains unsupported values
  - Constraint issues if foreign keys no longer exist
- **Sub-workflow reference:** None

#### Update Candidate Status
- **Type and technical role:** `n8n-nodes-base.postgres`; updates current candidate processing status.
- **Configuration choices:**
  - Operation: Execute query
  - Sets `status = 'processed'` for the analyzed candidate
- **Key expressions or variables used:**
  - `{{ $('Build Analysis Payload').item.json.candidate_id }}`
- **Input and output connections:**
  - Input: `Save Analysis`
  - Output: `Check Pending Candidates`
- **Version-specific requirements:** Type version `2.5`.
- **Edge cases or potential failure types:**
  - Candidate row missing
  - Database connectivity issues
- **Sub-workflow reference:** None

#### Check Pending Candidates
- **Type and technical role:** `n8n-nodes-base.postgres`; counts remaining pending candidates for the job.
- **Configuration choices:**
  - Operation: Execute query
  - `SELECT COUNT(*) AS pending FROM candidates WHERE job_id = ... AND status = 'pending'`
- **Key expressions or variables used:**
  - `{{ $('Build Analysis Payload').item.json.job_id }}`
- **Input and output connections:**
  - Input: `Update Candidate Status`
  - Output: `All Candidates Processed?`
- **Version-specific requirements:** Type version `2.5`.
- **Edge cases or potential failure types:**
  - PostgreSQL often returns counts as strings depending on driver behavior; the IF node assumes numeric comparison
  - Status drift if some selected candidates were not marked `pending` before execution
  - The count considers all candidates for the job, not only the subset passed in the webhook
- **Sub-workflow reference:** None

#### All Candidates Processed?
- **Type and technical role:** `n8n-nodes-base.if`; routes either to final summary or back into the loop.
- **Configuration choices:**
  - Condition: `{{$json.pending}} == 0`
  - True branch: finish the job and summarize
  - False branch: continue loop
- **Key expressions or variables used:**
  - `={{ $json.pending }}`
- **Input and output connections:**
  - Input: `Check Pending Candidates`
  - True output: `Update Job Status`
  - False output: `Loop Candidates`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If `pending` is a string `"0"` and strict numeric validation is enforced, comparison behavior should be verified in the target n8n version
  - Summary may never run if unrelated candidates for the same job remain pending
- **Sub-workflow reference:** None

---

## 2.6 Final Summary Generation

### Overview
This block runs once all candidates for the job are no longer pending. It marks the job as done, loads all analysis rows for the job, generates an executive summary with OpenAI, and saves it.

### Nodes Involved
- Update Job Status
- Fetch Full Pool
- Fetch Job for Summary
- Prompt 4 — Executive Summary
- Build Summary Payload
- Save Summary

### Node Details

#### Update Job Status
- **Type and technical role:** `n8n-nodes-base.postgres`; updates the job lifecycle state.
- **Configuration choices:**
  - Sets `jobs.status = 'done'`
- **Key expressions or variables used:**
  - `{{ $('Build Analysis Payload').item.json.job_id }}`
- **Input and output connections:**
  - Input: `All Candidates Processed?` true branch
  - Output: `Fetch Full Pool`
- **Version-specific requirements:** Type version `2.5`.
- **Edge cases or potential failure types:**
  - Job row missing
  - Database connectivity issues
- **Sub-workflow reference:** None

#### Fetch Full Pool
- **Type and technical role:** `n8n-nodes-base.postgres`; retrieves all analyzed candidates for summary generation.
- **Configuration choices:**
  - Joins `candidates` and `analyses`
  - Retrieves:
    - candidate name
    - score
    - adherence level
    - score justification
    - strengths
    - critical gaps
  - Orders descending by score
- **Key expressions or variables used:**
  - `{{ $('Build Analysis Payload').item.json.job_id }}`
- **Input and output connections:**
  - Input: `Update Job Status`
  - Output: `Fetch Job for Summary`
- **Version-specific requirements:** Type version `2.5`.
- **Edge cases or potential failure types:**
  - Missing analysis rows
  - Large candidate pools increasing prompt size later
- **Sub-workflow reference:** None

#### Fetch Job for Summary
- **Type and technical role:** `n8n-nodes-base.postgres`; fetches job metadata for the summary prompt.
- **Configuration choices:**
  - Retrieves `title` and `gabarito` for the current job
- **Key expressions or variables used:**
  - `{{ $('Build Analysis Payload').item.json.job_id }}`
- **Input and output connections:**
  - Input: `Fetch Full Pool`
  - Output: `Prompt 4 — Executive Summary`
- **Version-specific requirements:** Type version `2.5`.
- **Edge cases or potential failure types:**
  - Job row missing
  - Null `gabarito` if earlier stages failed
- **Sub-workflow reference:** None

#### Prompt 4 — Executive Summary
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; pool-level synthesis for HR decision support.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - System prompt positions the model as an executive HR consultant
  - User prompt includes:
    - job title
    - job template
    - complete analyzed candidate pool ordered by score
  - Output schema:
    - `total_analyzed`
    - `recommended`
    - `destaque`
    - `gap_comum`
    - `resumo`
- **Key expressions or variables used:**
  - `{{ $('Fetch Job for Summary').item.json.title }}`
  - `{{ JSON.stringify($('Fetch Job for Summary').item.json.gabarito) }}`
  - `{{ JSON.stringify($('Fetch Full Pool').all().map(i => i.json)) }}`
- **Input and output connections:**
  - Input: `Fetch Job for Summary`
  - Output: `Build Summary Payload`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Prompt length may exceed token limits for large candidate pools
  - Non-JSON output
  - Recommended list may not strictly follow `score >= 70` instruction
- **Sub-workflow reference:** None

#### Build Summary Payload
- **Type and technical role:** `n8n-nodes-base.code`; parses summary JSON and maps it to database field names.
- **Configuration choices:**
  - Parses Prompt 4 result with `JSON.parse`
  - Uses current `job_id`
  - Maps `total_analyzed` to `total_analisados`
- **Key expressions or variables used:**
  - `$('Prompt 4 — Executive Summary').item.json.output[0].content[0].text`
  - `$('Build Analysis Payload').item.json.job_id`
- **Input and output connections:**
  - Input: `Prompt 4 — Executive Summary`
  - Output: `Save Summary`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Invalid JSON from Prompt 4
  - Missing required summary fields
- **Sub-workflow reference:** None

#### Save Summary
- **Type and technical role:** `n8n-nodes-base.postgres`; upserts executive summary into `job_summaries`.
- **Configuration choices:**
  - Operation: Execute query
  - Inserts summary data
  - `ON CONFLICT (job_id) DO UPDATE`
  - Serializes recommendations to `jsonb`
  - Escapes quotes in text fields
- **Key expressions or variables used:**
  - `{{ $json.job_id }}`
  - `{{ $json.total_analisados }}`
  - `{{ JSON.stringify($json.recomendados) }}`
  - `{{ $json.destaque.replace(/'/g, "''") }}`
  - `{{ $json.gap_comum.replace(/'/g, "''") }}`
  - `{{ $json.resumo.replace(/'/g, "''") }}`
- **Input and output connections:**
  - Input: `Build Summary Payload`
  - Output: none
- **Version-specific requirements:** Type version `2.5`.
- **Edge cases or potential failure types:**
  - SQL escaping issues
  - Database auth/connectivity problems
  - Unique constraint or foreign key issues if data changed concurrently
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | n8n-nodes-base.stickyNote | Documentation note describing workflow purpose, setup, and customization |  |  | ## 🤖 AI CV Screening with Chained Prompts<br>Automatically screen resumes using 4 sequential AI prompts, each building on the previous one's output. Results are saved directly to PostgreSQL — no external backend required.<br><br>### How it works<br>1. **Webhook** receives a job ID + list of candidate IDs. Candidates and job must already exist in the database.<br>2. **Prompt 0** extracts a structured job template (requirements, differentials, behavioral competencies and weights) from the job description — runs only if `gabarito` is null in the jobs table.<br>3. **Prompts 1–3** run for each candidate in a loop:<br>   - **Prompt 1** scores the candidate (0–100) against the job template with calibration anchors to avoid score inflation, plus per-criteria scores<br>   - **Prompt 2** uses the score as context to identify concrete strengths (with CV evidence) and critical vs secondary gaps<br>   - **Prompt 3** uses the gaps as context to generate personalized interview questions for that specific candidate<br>4. Results are saved directly to the `analyses` table and candidate status updated to `processed`<br>5. **Prompt 4** runs automatically when all candidates are processed — generates an executive summary saved to `job_summaries`<br><br>### Setup<br>- Add your **OpenAI API credentials** to all AI nodes<br>- Add your **PostgreSQL credentials** to all Postgres nodes<br>- Create the required tables using the SQL schema in the sticky note below<br>- Trigger via `POST /webhook/cv-analyze` with `{ "job_id": 1, "candidate_ids": [1, 2, 3] }`<br><br>### Customization<br>- Swap `gpt-4.1-mini` for a more powerful model for higher accuracy<br>- Adjust scoring anchors in Prompt 1 to match your hiring standards<br>- Add more criteria to the job template in Prompt 0 |
| Database Schema | n8n-nodes-base.stickyNote | Documentation note containing required SQL schema |  |  | ## 🗄️ Required Database Schema<br>```sql<br>CREATE TABLE jobs (<br>  id SERIAL PRIMARY KEY,<br>  title VARCHAR(255) NOT NULL,<br>  description TEXT NOT NULL,<br>  gabarito JSONB,<br>  status VARCHAR(20) DEFAULT 'draft',<br>  created_at TIMESTAMP DEFAULT NOW()<br>);<br><br>CREATE TABLE candidates (<br>  id SERIAL PRIMARY KEY,<br>  job_id INTEGER REFERENCES jobs(id) ON DELETE CASCADE,<br>  name VARCHAR(255),<br>  filename VARCHAR(255),<br>  cv_text TEXT,<br>  status VARCHAR(20) DEFAULT 'pending',<br>  created_at TIMESTAMP DEFAULT NOW()<br>);<br><br>CREATE TABLE analyses (<br>  id SERIAL PRIMARY KEY,<br>  candidate_id INTEGER REFERENCES candidates(id) ON DELETE CASCADE,<br>  job_id INTEGER REFERENCES jobs(id) ON DELETE CASCADE,<br>  score INTEGER,<br>  nivel_aderencia VARCHAR(20),<br>  justificativa_score TEXT,<br>  pontos_fortes JSONB,<br>  gaps_criticos JSONB,<br>  gaps_secundarios JSONB,<br>  perguntas_entrevista JSONB,<br>  score_criterios JSONB,<br>  created_at TIMESTAMP DEFAULT NOW(),<br>  CONSTRAINT analyses_candidate_unique UNIQUE (candidate_id)<br>);<br><br>CREATE TABLE job_summaries (<br>  id SERIAL PRIMARY KEY,<br>  job_id INTEGER REFERENCES jobs(id) ON DELETE CASCADE,<br>  total_analisados INTEGER,<br>  recomendados JSONB,<br>  destaque TEXT,<br>  gap_comum TEXT,<br>  resumo TEXT,<br>  created_at TIMESTAMP DEFAULT NOW(),<br>  CONSTRAINT job_summaries_job_id_key UNIQUE (job_id)<br>);<br>``` |
| Input Format | n8n-nodes-base.stickyNote | Documentation note describing webhook payload |  |  | ## 📥 Webhook Input<br>Send a POST request:<br>```json<br>{<br>  "job_id": 1,<br>  "candidate_ids": [1, 2, 3]<br>}<br>```<br>Candidates and job must already exist in the database with `cv_text` populated. |
| Prompt 0 Section | n8n-nodes-base.stickyNote | Documentation note for conditional job template extraction |  |  | ## 🧠 Prompt 0 — Job Template Extraction<br>Runs only when `gabarito` is null in the jobs table.<br>Extracts structured requirements and sets weights automatically. |
| Candidate Loop Section | n8n-nodes-base.stickyNote | Documentation note for sequential per-candidate processing |  |  | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| Summary Section | n8n-nodes-base.stickyNote | Documentation note for final executive summary phase |  |  | ## 📊 Executive Summary — Prompt 4<br>Runs once when all candidates are processed (pending = 0).<br>Saves pool-level recommendation to job_summaries table. |
| Receive CVs | n8n-nodes-base.webhook | Receives POST requests to start screening |  | Fetch Job and Candidates | ## 📥 Webhook Input<br>Send a POST request:<br>```json<br>{<br>  "job_id": 1,<br>  "candidate_ids": [1, 2, 3]<br>}<br>```<br>Candidates and job must already exist in the database with `cv_text` populated. |
| Fetch Job and Candidates | n8n-nodes-base.postgres | Loads job, existing template, and selected candidates from PostgreSQL | Receive CVs | Job Template exists? |  |
| Job Template exists? | n8n-nodes-base.if | Branches based on whether `jobs.gabarito` already exists | Fetch Job and Candidates | Prepare Candidates; Prompt 0 — Extract Job Template |  |
| Prompt 0 — Extract Job Template | @n8n/n8n-nodes-langchain.openAi | Uses AI to extract a structured job template from the job description | Job Template exists? | Save Job Template | ## 🧠 Prompt 0 — Job Template Extraction<br>Runs only when `gabarito` is null in the jobs table.<br>Extracts structured requirements and sets weights automatically. |
| Save Job Template | n8n-nodes-base.postgres | Stores generated job template in `jobs.gabarito` | Prompt 0 — Extract Job Template | Prepare Candidates | ## 🧠 Prompt 0 — Job Template Extraction<br>Runs only when `gabarito` is null in the jobs table.<br>Extracts structured requirements and sets weights automatically. |
| Prepare Candidates | n8n-nodes-base.code | Builds one workflow item per candidate with job context | Job Template exists?; Save Job Template | Loop Candidates |  |
| Loop Candidates | n8n-nodes-base.splitInBatches | Iterates candidates sequentially | Prepare Candidates; All Candidates Processed? | Prompt 1 — Score | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| Prompt 1 — Score | @n8n/n8n-nodes-langchain.openAi | Scores candidate against job template | Loop Candidates | Prompt 2 — Gaps | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| Prompt 2 — Gaps | @n8n/n8n-nodes-langchain.openAi | Identifies evidence-backed strengths and gaps | Prompt 1 — Score | Prompt 3 — Interview Questions | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| Prompt 3 — Interview Questions | @n8n/n8n-nodes-langchain.openAi | Generates personalized interview questions | Prompt 2 — Gaps | Build Analysis Payload | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| Build Analysis Payload | n8n-nodes-base.code | Parses AI outputs and maps them to DB fields | Prompt 3 — Interview Questions | Save Analysis | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| Save Analysis | n8n-nodes-base.postgres | Upserts candidate analysis into `analyses` | Build Analysis Payload | Update Candidate Status | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| Update Candidate Status | n8n-nodes-base.postgres | Marks candidate as processed | Save Analysis | Check Pending Candidates | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| Check Pending Candidates | n8n-nodes-base.postgres | Counts remaining pending candidates for the job | Update Candidate Status | All Candidates Processed? | ## 🔄 Candidate Loop<br>Processes one candidate at a time.<br>Prompts 1 → 2 → 3 run sequentially,<br>each using the previous output as context.<br>Results saved to Postgres after each candidate. |
| All Candidates Processed? | n8n-nodes-base.if | Decides whether to continue the loop or generate summary | Check Pending Candidates | Update Job Status; Loop Candidates |  |
| Update Job Status | n8n-nodes-base.postgres | Marks job as done when no pending candidates remain | All Candidates Processed? | Fetch Full Pool | ## 📊 Executive Summary — Prompt 4<br>Runs once when all candidates are processed (pending = 0).<br>Saves pool-level recommendation to job_summaries table. |
| Fetch Full Pool | n8n-nodes-base.postgres | Retrieves all analyses for the job for summary generation | Update Job Status | Fetch Job for Summary | ## 📊 Executive Summary — Prompt 4<br>Runs once when all candidates are processed (pending = 0).<br>Saves pool-level recommendation to job_summaries table. |
| Fetch Job for Summary | n8n-nodes-base.postgres | Loads job title and template for final summary prompt | Fetch Full Pool | Prompt 4 — Executive Summary | ## 📊 Executive Summary — Prompt 4<br>Runs once when all candidates are processed (pending = 0).<br>Saves pool-level recommendation to job_summaries table. |
| Prompt 4 — Executive Summary | @n8n/n8n-nodes-langchain.openAi | Produces executive summary of the candidate pool | Fetch Job for Summary | Build Summary Payload | ## 📊 Executive Summary — Prompt 4<br>Runs once when all candidates are processed (pending = 0).<br>Saves pool-level recommendation to job_summaries table. |
| Build Summary Payload | n8n-nodes-base.code | Parses summary JSON and maps fields to DB schema | Prompt 4 — Executive Summary | Save Summary | ## 📊 Executive Summary — Prompt 4<br>Runs once when all candidates are processed (pending = 0).<br>Saves pool-level recommendation to job_summaries table. |
| Save Summary | n8n-nodes-base.postgres | Upserts executive summary into `job_summaries` | Build Summary Payload |  | ## 📊 Executive Summary — Prompt 4<br>Runs once when all candidates are processed (pending = 0).<br>Saves pool-level recommendation to job_summaries table. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the PostgreSQL schema first.**
   - In your PostgreSQL database, create these tables:
     - `jobs`
     - `candidates`
     - `analyses`
     - `job_summaries`
   - Use the schema reflected in the workflow:
     - `jobs`: includes `title`, `description`, `gabarito`, `status`
     - `candidates`: includes `job_id`, `name`, `filename`, `cv_text`, `status`
     - `analyses`: one unique row per candidate via `UNIQUE (candidate_id)`
     - `job_summaries`: one unique row per job via `UNIQUE (job_id)`

2. **Prepare seed data expectations.**
   - Insert at least one job into `jobs`.
   - Insert candidate records into `candidates` tied to that job.
   - Ensure `cv_text` is already populated.
   - Set candidate statuses to `pending`.
   - Optionally leave `jobs.gabarito` as `NULL` so Prompt 0 runs automatically.

3. **Create a new workflow in n8n.**
   - Name it something like: `AI CV Screening with Chained Prompts`.

4. **Add the Webhook node named `Receive CVs`.**
   - Type: `Webhook`
   - Method: `POST`
   - Path: `cv-analyze`
   - Leave options at default unless your environment requires custom response handling.

5. **Add a PostgreSQL credential.**
   - Create one PostgreSQL credential in n8n pointing to your target database.
   - Reuse this credential for every Postgres node.

6. **Add an OpenAI credential.**
   - Create one OpenAI API credential in n8n.
   - Reuse this credential for every OpenAI node.

7. **Add the Postgres node `Fetch Job and Candidates`.**
   - Type: `Postgres`
   - Operation: `Execute Query`
   - Connect `Receive CVs` → `Fetch Job and Candidates`
   - Set the query to:
     - select job fields: `id`, `title`, `description`, `gabarito`
     - aggregate selected candidate IDs with `array_agg`
     - aggregate candidate objects with `json_agg(json_build_object(...))`
     - join `jobs` to `candidates`
     - filter by webhook body `job_id`
     - filter by `candidate_ids` using `ANY(ARRAY[...]::int[])`
     - group by job fields
   - Use these dynamic inputs:
     - `{{$json.body.job_id}}`
     - `{{$json.body.candidate_ids.join(',')}}`

8. **Add the IF node `Job Template exists?`.**
   - Type: `If`
   - Connect `Fetch Job and Candidates` → `Job Template exists?`
   - Condition:
     - left value: `{{$json.gabarito}}`
     - operator: `is not empty`
   - True branch means template already exists.
   - False branch means template must be generated.

9. **Add the OpenAI node `Prompt 0 — Extract Job Template`.**
   - Type: OpenAI / LangChain OpenAI chat node
   - Connect the **false** output of `Job Template exists?` to this node
   - Model: `gpt-4.1-mini`
   - Add a system message instructing the model to act as an HR specialist and return only valid JSON.
   - Add a user message containing:
     - the job description from `$('Fetch Job and Candidates').item.json.description`
     - the expected JSON structure:
       - `mandatory_requirements`
       - `differential_requirements`
       - `behavioral_competencies`
       - `seniority_level`
       - `area`
       - `weights`
   - Explicitly state that weights must sum to `1.0`.

10. **Add the Postgres node `Save Job Template`.**
    - Type: `Postgres`
    - Connect `Prompt 0 — Extract Job Template` → `Save Job Template`
    - Operation: `Execute Query`
    - Query logic:
      - update `jobs`
      - set `gabarito` to the raw AI text cast as `jsonb`
      - filter by the current job ID from `Fetch Job and Candidates`
      - optionally `RETURNING gabarito`
    - Use:
      - AI output text from `{{$json.output[0].content[0].text}}`
      - job ID from `{{ $('Fetch Job and Candidates').item.json.job_id }}`

11. **Add the Code node `Prepare Candidates`.**
    - Type: `Code`
    - Connect:
      - true output of `Job Template exists?` → `Prepare Candidates`
      - `Save Job Template` → `Prepare Candidates`
    - Configure JavaScript to:
      - read job data from `Fetch Job and Candidates`
      - use existing `gabarito` if available
      - otherwise parse Prompt 0 output with `JSON.parse`
      - normalize `candidates` whether returned as string or object
      - return one item per candidate with:
        - `job_id`
        - `job_title`
        - `gabarito`
        - `candidate_id`
        - `candidate_name`
        - `cv_text`

12. **Add the node `Loop Candidates`.**
    - Type: `Split In Batches`
    - Connect `Prepare Candidates` → `Loop Candidates`
    - Keep default settings if you want the same behavior as the source workflow.
    - Important: reproduce the loop wiring exactly later, because this workflow uses the loopback pattern.

13. **Add the OpenAI node `Prompt 1 — Score`.**
    - Connect from the active candidate-processing output of `Loop Candidates`
    - Model: `gpt-4.1-mini`
    - System message:
      - define the AI as a senior recruiter
      - require strict JSON output
      - discourage score inflation
    - User message must include:
      - `{{ JSON.stringify($json.gabarito) }}`
      - `{{ $json.cv_text }}`
      - score anchor ranges:
        - 90–100
        - 70–89
        - 50–69
        - 30–49
        - 0–29
      - instructions to compute:
        - overall score
        - adherence level
        - concise evidence-based justification
        - per-criteria scores
    - Output JSON should include:
      - `score`
      - `adherence_level`
      - `justification`
      - `criteria_scores`

14. **Add the OpenAI node `Prompt 2 — Gaps`.**
    - Connect `Prompt 1 — Score` → `Prompt 2 — Gaps`
    - Model: `gpt-4.1-mini`
    - System message:
      - define evidence-based resume analysis
      - require JSON only
    - User message should include:
      - job template from `$('Loop Candidates').item.json.gabarito`
      - CV text from `$('Loop Candidates').item.json.cv_text`
      - score result from `$('Prompt 1 — Score').item.json.output[0].content[0].text`
    - Require JSON output with:
      - `strengths`
      - `critical_gaps`
      - `secondary_gaps`

15. **Add the OpenAI node `Prompt 3 — Interview Questions`.**
    - Connect `Prompt 2 — Gaps` → `Prompt 3 — Interview Questions`
    - Model: `gpt-4.1-mini`
    - System message:
      - define the AI as an interview specialist
      - require JSON only
    - User message should include:
      - job template
      - candidate name
      - Prompt 1 score result
      - Prompt 2 full analysis
    - Instruct the model to:
      - generate 5 to 7 questions
      - include `question`, `objective`, `type`
      - vary `technical`, `behavioral`, and `situational`

16. **Add the Code node `Build Analysis Payload`.**
    - Connect `Prompt 3 — Interview Questions` → `Build Analysis Payload`
    - Use JavaScript to:
      - parse Prompt 1, Prompt 2, and Prompt 3 outputs with `JSON.parse`
      - read candidate context from `Loop Candidates`
      - build a single JSON object with database-ready fields:
        - `candidate_id`
        - `job_id`
        - `score`
        - `nivel_aderencia`
        - `justificativa_score`
        - `score_criterios`
        - `pontos_fortes`
        - `gaps_criticos`
        - `gaps_secundarios`
        - `perguntas_entrevista`

17. **Add the Postgres node `Save Analysis`.**
    - Connect `Build Analysis Payload` → `Save Analysis`
    - Operation: `Execute Query`
    - Insert into `analyses`:
      - `candidate_id`
      - `job_id`
      - `score`
      - `nivel_aderencia`
      - `justificativa_score`
      - `score_criterios`
      - `pontos_fortes`
      - `gaps_criticos`
      - `gaps_secundarios`
      - `perguntas_entrevista`
    - Use `ON CONFLICT (candidate_id) DO UPDATE`
    - Convert arrays and objects to JSON strings and cast them to `jsonb`
    - Escape single quotes in text fields such as justification

18. **Add the Postgres node `Update Candidate Status`.**
    - Connect `Save Analysis` → `Update Candidate Status`
    - Operation: `Execute Query`
    - Query:
      - update `candidates`
      - set `status = 'processed'`
      - where `id` equals the current candidate ID from `Build Analysis Payload`

19. **Add the Postgres node `Check Pending Candidates`.**
    - Connect `Update Candidate Status` → `Check Pending Candidates`
    - Operation: `Execute Query`
    - Query:
      - count rows in `candidates`
      - where `job_id` is the current job
      - and `status = 'pending'`
    - Return the count as `pending`

20. **Add the IF node `All Candidates Processed?`.**
    - Connect `Check Pending Candidates` → `All Candidates Processed?`
    - Condition:
      - left value: `{{$json.pending}}`
      - operator: equals
      - right value: `0`
    - True branch means summary phase.
    - False branch means continue candidate loop.

21. **Wire the loopback.**
    - Connect the **false** output of `All Candidates Processed?` back to `Loop Candidates`.
    - This is essential to continue processing the next candidate.

22. **Add the Postgres node `Update Job Status`.**
    - Connect the **true** output of `All Candidates Processed?` → `Update Job Status`
    - Operation: `Execute Query`
    - Query:
      - update `jobs`
      - set `status = 'done'`
      - where `id` equals the current job ID

23. **Add the Postgres node `Fetch Full Pool`.**
    - Connect `Update Job Status` → `Fetch Full Pool`
    - Operation: `Execute Query`
    - Query should:
      - join `candidates` and `analyses`
      - fetch candidate name, score, adherence level, justification, strengths, critical gaps
      - filter by current job ID
      - order by score descending

24. **Add the Postgres node `Fetch Job for Summary`.**
    - Connect `Fetch Full Pool` → `Fetch Job for Summary`
    - Operation: `Execute Query`
    - Query:
      - select `title, gabarito`
      - from `jobs`
      - where `id` equals current job ID

25. **Add the OpenAI node `Prompt 4 — Executive Summary`.**
    - Connect `Fetch Job for Summary` → `Prompt 4 — Executive Summary`
    - Model: `gpt-4.1-mini`
    - System message:
      - define executive HR consultant behavior
      - require JSON only
    - User message should include:
      - job title
      - job template
      - full ordered candidate pool from `$('Fetch Full Pool').all().map(i => i.json)`
    - Require output:
      - `total_analyzed`
      - `recommended`
      - `destaque`
      - `gap_comum`
      - `resumo`
    - Instruct the model to recommend candidates with score `>= 70`.

26. **Add the Code node `Build Summary Payload`.**
    - Connect `Prompt 4 — Executive Summary` → `Build Summary Payload`
    - Use JavaScript to:
      - parse the prompt output JSON
      - map fields to:
        - `job_id`
        - `total_analisados`
        - `recomendados`
        - `destaque`
        - `gap_comum`
        - `resumo`

27. **Add the Postgres node `Save Summary`.**
    - Connect `Build Summary Payload` → `Save Summary`
    - Operation: `Execute Query`
    - Insert into `job_summaries`:
      - `job_id`
      - `total_analisados`
      - `recomendados`
      - `destaque`
      - `gap_comum`
      - `resumo`
    - Use `ON CONFLICT (job_id) DO UPDATE`
    - Serialize `recomendados` as `jsonb`
    - Escape single quotes in text fields

28. **Optionally add the sticky notes for maintainability.**
    - Add notes for:
      - overview
      - webhook input
      - Prompt 0 section
      - candidate loop section
      - summary section
      - database schema

29. **Set credentials on all nodes.**
    - PostgreSQL nodes:
      - `Fetch Job and Candidates`
      - `Save Job Template`
      - `Save Analysis`
      - `Update Candidate Status`
      - `Check Pending Candidates`
      - `Update Job Status`
      - `Fetch Full Pool`
      - `Fetch Job for Summary`
      - `Save Summary`
    - OpenAI nodes:
      - `Prompt 0 — Extract Job Template`
      - `Prompt 1 — Score`
      - `Prompt 2 — Gaps`
      - `Prompt 3 — Interview Questions`
      - `Prompt 4 — Executive Summary`

30. **Test with sample input.**
    - Send a POST request to the webhook:
      ```json
      {
        "job_id": 1,
        "candidate_ids": [1, 2, 3]
      }
      ```
    - Confirm:
      - `jobs.gabarito` is created if previously null
      - `analyses` receives one row per candidate
      - candidate statuses become `processed`
      - once no pending candidates remain for that job, `jobs.status` becomes `done`
      - `job_summaries` receives a summary row

31. **Validate the main operational assumptions.**
    - All selected candidates must belong to the specified job.
    - `cv_text` should already contain usable resume text.
    - Candidate statuses should start as `pending` if you want Prompt 4 to run automatically.
    - The summary logic checks all pending candidates for the job, not just the subset provided in the webhook.

32. **Recommended hardening if you adapt the workflow.**
    - Add input validation before SQL execution.
    - Replace interpolated SQL with parameterized queries where possible.
    - Add JSON validation after each AI node before database writes.
    - Add fallback handling for malformed model output.
    - Consider chunking or truncating long CVs and large candidate pools to avoid token-limit failures.

## Entry Points and Sub-workflows

- **Entry point:** one webhook entry point only: `Receive CVs`
- **Sub-workflows invoked:** none
- **Execute Workflow / Call Workflow nodes:** none

## External Dependencies

- **OpenAI API**
  - Used by Prompt 0, 1, 2, 3, and 4
  - Requires valid OpenAI credentials in n8n
- **PostgreSQL**
  - Used for all persistence and source-of-truth data loading
  - Requires database tables described above

## Notable Integration and Logic Caveats

- The workflow assumes AI nodes return strict valid JSON, but it does not implement defensive parsing beyond `JSON.parse`.
- SQL is built with string interpolation in multiple places; this works but is less robust than parameterized queries.
- Final summary generation depends on `pending = 0` across the entire job, not only the candidate subset submitted in the webhook.
- The job template existence check validates non-empty presence, not JSON correctness.
- For very large CVs or large candidate pools, prompt token limits may become a practical issue.