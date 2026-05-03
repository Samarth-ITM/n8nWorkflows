Evaluate organizational data maturity and email HTML reports with Postgres

https://n8nworkflows.xyz/workflows/evaluate-organizational-data-maturity-and-email-html-reports-with-postgres-12963


# Evaluate organizational data maturity and email HTML reports with Postgres

## 1. Workflow Overview

**Title:** Evaluate organizational data maturity and email HTML reports with Postgres  
**Purpose:** This workflow receives an incoming payload (via webhook) containing an organization name and maturity questionnaire scores, computes per-dimension averages plus an overall score, generates an HTML “bar chart” + formatted maturity narratives, stores results in Postgres, and emails an HTML report.

**Typical use cases**
- Data maturity / AI maturity self-assessment forms
- Automated scoring, benchmarking (“rank” lookups), and executive-ready email reporting
- Persisting form submissions and computed scores for later analytics

### 1.1 Input reception & normalization
Receives the webhook payload and normalizes/collects key fields into a single string field (`CollectedData`) used by downstream computations.

### 1.2 Score computation & HTML chart generation
Parses the collected CSV-like string, computes averages across 6 dimensions, computes an overall average, and renders an HTML table-based bar chart.

### 1.3 Persistence (Postgres) + synchronization
Saves (a) raw form details and (b) computed results to Postgres, then uses **Wait** nodes and a **Merge** to ensure both writes complete before continuing.

### 1.4 Rank retrieval / benchmarking (Postgres)
Runs several Postgres queries (one per dimension + overall) to retrieve “rank”/benchmark values.

### 1.5 Email content rendering + file creation + send
Creates styled maturity assessments (levels, gradients, descriptions), prepares the HTML body, optionally converts HTML to a file attachment, merges streams, and sends the email.

---

## 2. Block-by-Block Analysis

### Block 1 — Input reception & initial routing
**Overview:** Accepts an inbound HTTP request and splits the flow: one path prepares data for scoring; the other stores raw submission details.  
**Nodes involved:** `Webhook1`, `DataCollector`, `Save Form Details`

#### Node: Webhook1
- **Type / role:** `Webhook` (entry point). Receives HTTP request and starts the workflow.
- **Configuration (interpreted):**
  - Uses default parameters (not shown), so method/path/response mode must be set in the n8n UI or left default.
- **Inputs / outputs:**
  - **Output →** `DataCollector` and `Save Form Details` (two parallel branches).
- **Edge cases / failures:**
  - Missing expected body fields used later (e.g., the string that becomes `CollectedData`) will cause failures downstream in Code nodes.
  - If webhook response mode is “On Received” vs “Last Node”, it may affect caller experience (timeouts if long processing).

#### Node: DataCollector
- **Type / role:** `Set` node used to shape incoming webhook data.
- **Configuration (interpreted):**
  - Expected to create a field named **`CollectedData`** (required by the next Code node).
  - The JSON does not include field mappings, so you must configure this manually to build a CSV line like:
    - `Organization,score1,score2,...`
- **Input / output connections:**
  - **Input ←** `Webhook1`
  - **Output →** `Calculate & Generate Charts`
- **Edge cases:**
  - If the generated string is not valid CSV or does not match the expected score count, parsing/averaging will fail.

#### Node: Save Form Details
- **Type / role:** `Postgres` node storing raw form submission details.
- **Configuration (interpreted):**
  - Operation/query not present in JSON; must be configured (INSERT recommended).
- **Input / output connections:**
  - **Input ←** `Webhook1`
  - **Output →** `Wait Save Form Details`
- **Edge cases / failures:**
  - Credential/auth errors; schema/table missing; constraint violations; data type mismatch.
  - Large payload fields may exceed column size if not planned.

---

### Block 2 — Score computation & HTML chart creation
**Overview:** Parses the collected CSV string, computes dimension averages + overall score, and generates an HTML bar chart snippet.  
**Nodes involved:** `Calculate & Generate Charts`, `Save Results`

#### Node: Calculate & Generate Charts
- **Type / role:** `Code` node (Python-like code shown in notes) to compute averages and create HTML.
- **Key logic (as implemented):**
  - Reads: `raw_string = items[0]["json"]["CollectedData"]`
  - Parses CSV first row into `input_data`
  - Expects:
    - `input_data[0]` = organization name
    - `input_data[1:]` = integer scores
  - Dimension mapping (score counts per dimension):
    - Data Strategy & Governance: **4**
    - Data Quality & Integrity: **3**
    - Data-Driven Decision Intelligence: **4**
    - Data Management & Operations: **4**
    - Data Ethics & Privacy: **4**
    - AI Maturity Assessment: **6**
  - Computes average per dimension and overall average (mean of dimension averages).
  - Builds an **HTML table** with colored bars:
    - `bar_width = avg * 20` (so avg=5 → 100%)
    - Colors: green (>=4), amber (>=3), red (<3)
- **Outputs:**
  - `organization`
  - `html_chart`
  - `dimension_avgs` (object/dict)
  - `overall_avg`
- **Input / output connections:**
  - **Input ←** `DataCollector`
  - **Output →** `Save Results`
- **Version-specific notes:**
  - n8n “Code” node normally runs **JavaScript**. The node notes contain **Python syntax** (`import csv`, `StringIO`, etc.). This will fail unless:
    - you are using an n8n environment/node variant that supports Python, or
    - you convert this code to JavaScript.
- **Edge cases / failures:**
  - `CollectedData` missing → KeyError.
  - Non-integer score strings → `int(x)` conversion fails.
  - Wrong number of scores → dimension slicing still “works” but produces incorrect averages; could also yield empty slices (division by count still uses fixed count, masking bad input).
  - Scores outside 0–5 are not validated here (bars can exceed 100% if >5).

#### Node: Save Results
- **Type / role:** `Postgres` node to store computed results (dimension averages, overall, organization, chart HTML, etc.).
- **Configuration (interpreted):**
  - Operation/query not present in JSON; typically an INSERT into a results table.
- **Input / output connections:**
  - **Input ←** `Calculate & Generate Charts`
  - **Output →** `Wait Save Results`
- **Edge cases / failures:**
  - Storing nested JSON (`dimension_avgs`) may require JSONB column or serialization.
  - Storing raw HTML may require TEXT column.

---

### Block 3 — Synchronization after DB writes
**Overview:** Uses Wait nodes and a Merge to ensure both “Save Form Details” and “Save Results” complete before rank queries run.  
**Nodes involved:** `Wait Save Form Details`, `Wait Save Results`, `Collect Step`

#### Node: Wait Save Form Details
- **Type / role:** `Wait` node used as a synchronization step.
- **Configuration (interpreted):**
  - Uses default parameters (not shown). It has a webhookId (internal) but no explicit wait condition visible.
- **Input / output connections:**
  - **Input ←** `Save Form Details`
  - **Output →** `Collect Step` (index 1)
- **Edge cases:**
  - If configured as “Wait for webhook” or “resume”, it may require external resume calls; with default/empty config it may not behave as intended.

#### Node: Wait Save Results
- **Type / role:** `Wait` node used as a synchronization step.
- **Input / output connections:**
  - **Input ←** `Save Results`
  - **Output →** `Collect Step` (index 0)

#### Node: Collect Step
- **Type / role:** `Merge` node; combines the two branches after DB persistence.
- **Configuration (interpreted):**
  - Merge mode not shown; typically “Wait for both inputs” to synchronize.
- **Input / output connections:**
  - **Inputs ←** `Wait Save Results` (Input 1), `Wait Save Form Details` (Input 2)
  - **Outputs →** fans out to all rank query nodes
- **Edge cases:**
  - If merge mode is not “wait for both”, the rank queries could execute before one DB write finishes.

---

### Block 4 — Rank retrieval / benchmarking
**Overview:** After persistence, the workflow queries Postgres to retrieve ranks/benchmarks for each dimension and overall, then merges those into a single multi-item stream for assessment formatting.  
**Nodes involved:** `Rank_Data_Strategy_Governance`, `Rank_Data_Quality_Integrity`, `Rank_Data_DrivenDecision_Intelligence`, `Rank_Data_Management_Operations`, `Rank_Data_Ethics_Privacy`, `Rank_AI_Maturity_Assessment`, `Rank_Overall_Avg`, `Collect Ranks`

#### Rank_* nodes (7 nodes total)
- **Type / role:** `Postgres` nodes, each presumably executes a query returning a value used for email formatting.
- **Configuration (interpreted):**
  - Not shown in JSON. Likely `SELECT` statements computing percentile/rank relative to stored results.
- **Input / output connections:**
  - **Input ←** `Collect Step`
  - Each outputs into `Collect Ranks` at a dedicated input index:
    - Strategy → index 0
    - Quality → index 1
    - Driven Decision → index 2
    - Management Ops → index 3
    - Ethics → index 4
    - AI → index 5
    - Overall → index 6
- **Edge cases / failures:**
  - Queries returning no rows → downstream Code node indexing may fail if expected fields absent.
  - Numeric conversions if rank columns are text.
  - Performance: multiple sequential queries; consider a single query returning all ranks.

#### Node: Collect Ranks
- **Type / role:** `Merge` node that gathers results from the 7 rank queries into one stream.
- **Configuration (interpreted):**
  - Must be set to a mode that produces a multi-item output consumable by the next Code node (commonly “Append”).
- **Input / output connections:**
  - **Inputs ←** all `Rank_*` nodes
  - **Output →** `Generate Email styling and Formats`
- **Edge cases:**
  - If merge mode outputs a single combined item instead of 7 items, the next Code node (which uses `items[0]..items[5]`) will break.

---

### Block 5 — Email assessment rendering, HTML body, attachment, and sending
**Overview:** Builds maturity level narratives and styling, prepares the email HTML body, optionally converts HTML to a file, merges the streams, and sends the final email.  
**Nodes involved:** `Generate Email styling and Formats`, `HTML Body`, `Create HTML FIle`, `Merge`, `Send Email seek chart wtith overall`

#### Node: Generate Email styling and Formats
- **Type / role:** `Code` node generating maturity “cards” (level, description, gradients) and an overall assessment.
- **Key expressions/variables:**
  - Reads dimension scores from separate incoming items:
    - `items[0]["json"]["Data_Strategy_Governance"]`
    - `items[1]["json"]["Data_Quality_Integrity"]`
    - `items[2]["json"]["Data_DrivenDecision_Intelligence"]` (note the inconsistent naming in code: `Data_DrivenDecision_Intelligence`)
    - `items[3]["json"]["Data_Management_Operations"]`
    - `items[4]["json"]["Data_Ethics_Privacy"]`
    - `items[5]["json"]["AI_Maturity_Assessment"]`
  - Validates each score is number 0–5; raises ValueError otherwise.
  - Assigns maturity level thresholds:
    - >=4: Excellent
    - >=3: Good
    - else: Needs_Improvement
  - Returns:
    - `overall` (assessment dict)
    - `dimensions` (list of 6 assessment dicts)
- **Input / output connections:**
  - **Input ←** `Collect Ranks`
  - **Outputs →** `HTML Body` and `Merge` (index 0)
- **Version-specific notes:**
  - Same concern as earlier: the notes show **Python**, but n8n Code is typically **JavaScript**.
- **Edge cases / failures:**
  - If `Collect Ranks` output is not exactly 6+ items in the expected order, indexing breaks.
  - Field-name mismatches:
    - Code expects `Data_DrivenDecision_Intelligence` in `items[2].json`
    - But the dimension key used in descriptions is `Data_Driven_Decision_Intelligence` (with an extra underscore). This is handled internally (description dict keys), but input keys must match what rank query returns.
  - If any rank query returns null/empty string → validation fails.

#### Node: HTML Body
- **Type / role:** `Set` node to assemble final HTML email body (likely combining chart + assessments + organization).
- **Configuration:**
  - Not shown in JSON; must be manually set using expressions referencing:
    - The generated `overall`/`dimensions` object from the previous Code node
    - The `html_chart` and `organization` produced earlier (but note: this branch currently doesn’t directly carry those fields unless you merge them in)
- **Input / output connections:**
  - **Input ←** `Generate Email styling and Formats`
  - **Output →** `Create HTML FIle`
- **Edge cases:**
  - Missing chart content if not merged in from earlier computation branch (a common issue in n8n when parallel branches don’t rejoin with required data).

#### Node: Create HTML FIle
- **Type / role:** `Convert to File` node; turns HTML text into a binary file (attachment).
- **Configuration (interpreted):**
  - Must define:
    - Source field containing HTML (from `HTML Body`)
    - File name and MIME type (typically `text/html`)
- **Input / output connections:**
  - **Input ←** `HTML Body`
  - **Output →** `Merge` (index 1)
- **Edge cases:**
  - If HTML field name is wrong/empty, file conversion yields empty attachment.

#### Node: Merge
- **Type / role:** `Merge` node combining:
  - Assessment JSON (from `Generate Email styling and Formats`) and
  - HTML file (from `Create HTML FIle`)
- **Input / output connections:**
  - **Input 0 ←** `Generate Email styling and Formats`
  - **Input 1 ←** `Create HTML FIle`
  - **Output →** `Send Email seek chart wtith overall`
- **Edge cases:**
  - If merge mode is incorrect, you may lose either the JSON body fields or the binary attachment field.

#### Node: Send Email seek chart wtith overall
- **Type / role:** `Email Send` node to deliver the report.
- **Configuration (interpreted):**
  - Not shown in JSON; you must configure:
    - SMTP credentials
    - To/From/Subject
    - HTML body field
    - Attachments (binary property from `Create HTML FIle`)
- **Input / output connections:**
  - **Input ←** `Merge`
- **Edge cases / failures:**
  - SMTP auth failures, TLS issues, blocked ports.
  - If using a provider like Gmail/M365, ensure app-password/OAuth and “From” alignment.
  - Large HTML attachment could exceed limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook1 | Webhook | Entry point receiving form submission | — | DataCollector; Save Form Details |  |
| DataCollector | Set | Build/normalize `CollectedData` for scoring | Webhook1 | Calculate & Generate Charts |  |
| Calculate & Generate Charts | Code | Parse scores, compute averages, generate HTML bar chart | DataCollector | Save Results |  |
| Save Results | Postgres | Persist computed results | Calculate & Generate Charts | Wait Save Results |  |
| Wait Save Results | Wait | Synchronize after saving results | Save Results | Collect Step |  |
| Save Form Details | Postgres | Persist raw form submission | Webhook1 | Wait Save Form Details |  |
| Wait Save Form Details | Wait | Synchronize after saving form details | Save Form Details | Collect Step |  |
| Collect Step | Merge | Join the two DB-write branches before rank queries | Wait Save Results; Wait Save Form Details | Rank_Data_Strategy_Governance; Rank_Data_Quality_Integrity; Rank_Data_DrivenDecision_Intelligence; Rank_Data_Management_Operations; Rank_Data_Ethics_Privacy; Rank_AI_Maturity_Assessment; Rank_Overall_Avg |  |
| Rank_Data_Strategy_Governance | Postgres | Retrieve rank/benchmark for Strategy & Governance | Collect Step | Collect Ranks |  |
| Rank_Data_Quality_Integrity | Postgres | Retrieve rank/benchmark for Quality & Integrity | Collect Step | Collect Ranks |  |
| Rank_Data_DrivenDecision_Intelligence | Postgres | Retrieve rank/benchmark for Decision Intelligence | Collect Step | Collect Ranks |  |
| Rank_Data_Management_Operations | Postgres | Retrieve rank/benchmark for Management & Ops | Collect Step | Collect Ranks |  |
| Rank_Data_Ethics_Privacy | Postgres | Retrieve rank/benchmark for Ethics & Privacy | Collect Step | Collect Ranks |  |
| Rank_AI_Maturity_Assessment | Postgres | Retrieve rank/benchmark for AI Maturity | Collect Step | Collect Ranks |  |
| Rank_Overall_Avg | Postgres | Retrieve overall rank/benchmark | Collect Step | Collect Ranks |  |
| Collect Ranks | Merge | Combine rank query outputs into one stream | All Rank_* nodes | Generate Email styling and Formats |  |
| Generate Email styling and Formats | Code | Build maturity levels, gradients, descriptions + overall | Collect Ranks | HTML Body; Merge |  |
| HTML Body | Set | Assemble final email HTML content | Generate Email styling and Formats | Create HTML FIle |  |
| Create HTML FIle | Convert to File | Create HTML attachment from email body | HTML Body | Merge |  |
| Merge | Merge | Combine attachment + email data | Generate Email styling and Formats; Create HTML FIle | Send Email seek chart wtith overall |  |
| Send Email seek chart wtith overall | Email Send | Send final email with HTML body/attachment | Merge | — |  |
| Sticky Note | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note1 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note2 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note3 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note4 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note5 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note6 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note7 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note8 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note9 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note10 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note11 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note12 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note13 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note14 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note15 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note16 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note17 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note18 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note19 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note20 | Sticky Note | Comment container (empty) | — | — |  |

> All sticky notes in the provided JSON have **empty content**, so no comment text applies.

---

## 4. Reproducing the Workflow from Scratch

1) **Create `Webhook1` (Webhook node)**
- Set an HTTP path (e.g., `/data-maturity`) and method (POST recommended).
- Decide response mode:
  - If you want fast acknowledgement, respond “On Received”.
  - If you want to return computed data, respond “Last Node” (risk of timeout if email/DB slow).

2) **Create `DataCollector` (Set node)**
- Add a new string field: **`CollectedData`**
- Build a CSV line in the exact order expected by the chart code:
  - `organization, s1, s2, ... s25`
- Example expression pattern (adapt to your webhook payload structure):
  - `{{$json.organization}},{{$json.q1}},{{$json.q2}},...`
- Connect: `Webhook1 → DataCollector`

3) **Create `Save Form Details` (Postgres node)**
- Configure Postgres credentials.
- Use an **INSERT** query into a `form_submissions` table (example columns: received_at, raw_payload, organization, email).
- Connect: `Webhook1 → Save Form Details`

4) **Create `Calculate & Generate Charts` (Code node)**
- Important: rewrite the provided Python into **JavaScript** unless your n8n supports Python execution.
- Ensure it reads `items[0].json.CollectedData`, parses CSV, computes averages and outputs:
  - `organization`, `html_chart`, `dimension_avgs`, `overall_avg`
- Connect: `DataCollector → Calculate & Generate Charts`

5) **Create `Save Results` (Postgres node)**
- Configure Postgres credentials.
- Insert computed results into a `maturity_results` table (recommend JSONB for `dimension_avgs`).
- Connect: `Calculate & Generate Charts → Save Results`

6) **Add synchronization nodes**
- Create `Wait Save Results` (Wait) and connect: `Save Results → Wait Save Results`
- Create `Wait Save Form Details` (Wait) and connect: `Save Form Details → Wait Save Form Details`
- Create `Collect Step` (Merge)
  - Set merge mode to **wait for both inputs** (or equivalent).
  - Connect:
    - `Wait Save Results → Collect Step (Input 1)`
    - `Wait Save Form Details → Collect Step (Input 2)`

7) **Create the 7 rank query nodes (Postgres)**
- Create each Postgres node:
  - `Rank_Data_Strategy_Governance`
  - `Rank_Data_Quality_Integrity`
  - `Rank_Data_DrivenDecision_Intelligence`
  - `Rank_Data_Management_Operations`
  - `Rank_Data_Ethics_Privacy`
  - `Rank_AI_Maturity_Assessment`
  - `Rank_Overall_Avg`
- Each should run a SELECT returning a **single numeric score** in a predictable field name that matches the next Code node’s expectations (see below).
- Connect: `Collect Step → each Rank_*`

8) **Create `Collect Ranks` (Merge)**
- Set mode to **Append** (so outputs are multiple items).
- Connect each `Rank_* → Collect Ranks` (the exact input index mapping is not required, but the next Code node assumes item ordering—so keep it consistent).

9) **Create `Generate Email styling and Formats` (Code node)**
- Rewrite to JavaScript if needed.
- Ensure the incoming items contain these fields exactly (or update the code to match your actual query output):
  - `Data_Strategy_Governance`
  - `Data_Quality_Integrity`
  - `Data_DrivenDecision_Intelligence`
  - `Data_Management_Operations`
  - `Data_Ethics_Privacy`
  - `AI_Maturity_Assessment`
- Output a single item with:
  - `overall` and `dimensions` (array)
- Connect: `Collect Ranks → Generate Email styling and Formats`

10) **Create `HTML Body` (Set node)**
- Build the final HTML email body.
- Include:
  - Organization name
  - The `html_chart` (from the chart code) — this is currently from a different branch; to include it, you must either:
    - carry it through DB + re-query it here, or
    - add an additional Merge earlier to bring chart data into the email branch.
- Connect: `Generate Email styling and Formats → HTML Body`

11) **Create `Create HTML FIle` (Convert to File node)**
- Convert the HTML body field (e.g., `email_html`) to a file:
  - File name: `report.html`
  - MIME type: `text/html`
- Connect: `HTML Body → Create HTML FIle`

12) **Create final `Merge` (Merge node)**
- Merge assessment JSON + binary file so the Email node can use both.
- Connect:
  - `Generate Email styling and Formats → Merge (Input 0)`
  - `Create HTML FIle → Merge (Input 1)`

13) **Create `Send Email seek chart wtith overall` (Email Send node)**
- Configure SMTP credentials (or provider integration).
- Set:
  - **To:** from webhook payload or a fixed recipient
  - **Subject:** include org name and overall level/score
  - **HTML Body:** the field from `HTML Body` (or merged output)
  - **Attachment:** binary property produced by Convert to File
- Connect: `Merge → Send Email seek chart wtith overall`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The Code node notes contain Python code; n8n Code node is typically JavaScript. Plan to port the scripts to JS unless your environment explicitly supports Python execution. | Applies to `Calculate & Generate Charts` and `Generate Email styling and Formats`. |
| Sticky notes exist but contain no text in the provided workflow export. | No additional documentation embedded in the canvas. |

If you want, paste an example of the webhook payload you send (fields + sample values) and your intended Postgres table schemas; I can provide exact Set-node expressions and Postgres SQL for all INSERT/SELECT steps and ensure the merge ordering matches the Code nodes.