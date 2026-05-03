Find high-quality remote jobs with OpenAI, Decodo, and Google Sheets

https://n8nworkflows.xyz/workflows/find-high-quality-remote-jobs-with-openai--decodo--and-google-sheets-12304


# Find high-quality remote jobs with OpenAI, Decodo, and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Find High-Quality Remote Jobs with AI, Decodo, and Google Sheets  
**Purpose:** Automatically fetch remote job listings (RemoteOK), merge them with a candidate profile stored in Google Sheets, score each job’s fit using an AI agent (salary/skills/industry weighted), store qualified matches in Google Sheets, and email a Top 5 shortlist.

**Primary use cases**
- Daily remote job monitoring for a specific candidate profile.
- Automated shortlist generation with explainable scoring.
- Centralized tracking of matches in Google Sheets.

### 1.1 Candidate Profile Input
Reads the candidate profile from Google Sheets (e.g., skills, salary expectations, preferred industries, no-go constraints).

### 1.2 Data Collection, Parsing & Pairing
Fetches RemoteOK HTML via Decodo, extracts JSON-LD “JobPosting” objects, then merges the candidate profile with each job into candidate-job pairs.

### 1.3 AI Scoring & Normalization
Uses an n8n LangChain Agent powered by an OpenAI chat model to score fit. Then parses/normalizes AI output into consistent fields.

### 1.4 Persistence & Notification
Saves normalized results to Google Sheets and filters by fit score threshold to build and send a Top 5 HTML email via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Candidate Profile Input

**Overview:** Triggers daily and loads the candidate’s structured profile from Google Sheets to be used downstream in job matching.  
**Nodes Involved:**  
- Daily trigger (job scan)  
- Load candidate profile (Google Sheets)

#### Node: Daily trigger (job scan)
- **Type / role:** Schedule Trigger; entry point to run the workflow periodically.
- **Configuration (interpreted):** Runs on an interval rule (JSON shows `interval: [{}]`, which typically means “daily” in the UI template but should be verified in n8n).
- **Outputs:** Sends a single item to “Load candidate profile (Google Sheets)”.
- **Edge cases / failures:**
  - Misconfigured schedule rule may cause unexpected frequency or no executions.

#### Node: Load candidate profile (Google Sheets)
- **Type / role:** Google Sheets node; reads candidate profile rows.
- **Configuration choices:**
  - **Document:** `YOUR_GOOGLE_SHEET_ID` (placeholder).
  - **Sheet name:** `Sheet1` (cached name “Candidate_profile” shown in template).
  - Operation is not explicitly shown, but in this pattern it is typically **Read / Get All** rows.
- **Key fields expected (implied by AI prompt):**
  - `skills` (list/string)
  - `salary_expectation`
  - `preferred_industries`
  - `no_go` (negative constraints)
  - Any additional profile fields used by the agent.
- **Connections:**
  - **Main output →** `Fetch RemoteOK HTML (Decodo)`
  - **Main output →** `Merge profile + job list` (input 0)
- **Edge cases / failures:**
  - OAuth failure/expired token.
  - Sheet/Document ID wrong, permissions missing.
  - Profile missing expected columns → AI scoring quality degrades (salary/skills/industry become “unknown”).
  - Multiple profile rows: downstream “Create candidate-job pairs” assumes the first item is the candidate and remaining items are jobs; if multiple candidate rows exist, pairing will be wrong.

---

### Block 1.2 — Data Collection, Parsing & Pairing

**Overview:** Fetches RemoteOK listings via Decodo, extracts JSON-LD JobPosting blocks, merges them with the candidate profile, and generates one item per (candidate, job) pair for AI scoring.  
**Nodes Involved:**  
- Fetch RemoteOK HTML (Decodo)  
- Extract JobPosting JSON  
- Merge profile + job list  
- Create candidate-job pairs

#### Node: Fetch RemoteOK HTML (Decodo)
- **Type / role:** Decodo node (web fetch/scrape proxy) to retrieve HTML content.
- **Configuration choices:**
  - URL: `https://remoteok.com/remote-technical-jobs?location=Worldwide&min_salary=120000`
  - Uses Decodo credentials.
- **Outputs:** HTML content in a structure like `results[0].content` (as referenced by the next Code node).
- **Edge cases / failures:**
  - RemoteOK blocking/WAF changes; Decodo reduces risk but not guaranteed.
  - Decodo credential/auth issues.
  - HTML structure changes → downstream extractor may yield 0 jobs.

#### Node: Extract JobPosting JSON
- **Type / role:** Code node; parses HTML and outputs one item per JSON-LD JobPosting.
- **Configuration choices (as implemented):**
  - Looks for `<script type="application/ld+json"> ... </script>` blocks via `indexOf` scanning (no regex for HTML parsing).
  - `JSON.parse` each block, keeps only those with `@type === 'JobPosting'`.
  - Extracts/enriches fields:
    - `title`, `company` (from `hiringOrganization.name`), `datePosted`, `employmentType`, `jobLocationType`, `url`, `description`
    - Salary: `minSalary`, `maxSalary`, `currency` from `baseSalary`
    - Locations list from `jobLocation[].address` (country/region/locality)
- **Connections:**
  - **Main output →** `Merge profile + job list` (input 1)
- **Edge cases / failures:**
  - If RemoteOK changes script tags or JSON-LD format, parsing can fail silently (node returns fewer/no items).
  - Some JobPosting entries may not include salary or URL; node sets nulls accordingly.
  - Very large HTML could increase execution time.

#### Node: Merge profile + job list
- **Type / role:** Merge node; combines candidate profile stream with extracted job items.
- **Configuration choices:**
  - Parameters are empty in JSON; default behavior in n8n Merge can vary by mode (e.g., Append, Merge by position, etc.).
  - Given downstream code expects `items[0]` to be the candidate and `items[1..]` jobs, this Merge must effectively **append** candidate item(s) before job items in a single list.
- **Inputs:**
  - Input 0: Candidate profile items
  - Input 1: Job items from extractor
- **Outputs:**
  - Single combined stream into “Create candidate-job pairs”
- **Edge cases / failures:**
  - Wrong merge mode/order leads to candidate not at index 0, breaking pairing logic.
  - If candidate sheet returns multiple items, only first is used as candidate; others treated as jobs.

#### Node: Create candidate-job pairs
- **Type / role:** Code node; transforms combined list into per-job objects shaped for AI input.
- **Configuration choices (as implemented):**
  - `candidate = items[0].json`
  - For each job item from index 1 onward, outputs:
    - `{ candidate: <candidate>, job: <job> }`
- **Connections:**
  - **Main output →** `Score fit with AI (salary + skills + industry)`
- **Edge cases / failures:**
  - If there are zero jobs, outputs empty list → downstream nodes do nothing.
  - If merge ordering is wrong, candidate/job mapping becomes incorrect.

---

### Block 1.3 — AI Scoring & Normalization

**Overview:** Uses an AI agent (OpenAI model) to compute fit metrics and recommendations, then robustly parses and normalizes the AI output into consistent job fields for filtering, storage, and email.  
**Nodes Involved:**  
- OpenAI Chat Model  
- Score fit with AI (salary + skills + industry)  
- Flatten AI scores into job fields  
- Parse AI JSON output (safe)  
- Normalize job data for output

#### Node: OpenAI Chat Model
- **Type / role:** LangChain OpenAI Chat Model provider node.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Connected as `ai_languageModel` input to the Agent node.
- **Credentials:** OpenAI API credential.
- **Edge cases / failures:**
  - Invalid API key / quota exceeded.
  - Model availability changes.
  - Latency/timeouts on large batches.

#### Node: Score fit with AI (salary + skills + industry)
- **Type / role:** LangChain Agent node; generates structured scoring JSON per candidate-job pair.
- **Configuration choices:**
  - Prompt includes `{{ JSON.stringify($json) }}` where `$json` contains `{candidate, job}`.
  - Requires output to be **ONLY** a specific JSON structure with:
    - `fit_score`, `salary_score`, `skill_match_percentage`, `industry_match`, `industry_match_score`,
    - arrays: `missing_skills`, `matching_skills`,
    - `salary_alignment`, `location_match`, `job_complexity_score`, `seniority_match`,
    - `alignment_explanation`, `final_recommendation`
  - System message defines weighting: **40% salary, 40% skills, 20% industry**; seniority informational only.
- **Connections:**
  - Receives model from “OpenAI Chat Model” via agent language model connection.
  - **Main output →** `Flatten AI scores into job fields`
- **Edge cases / failures:**
  - Model may output non-JSON or wrap JSON in code fences → downstream parsing nodes attempt to fix.
  - Missing salary fields: instructed to set salary_alignment “unknown”; scoring still requires numeric outputs—agent may behave inconsistently.
  - High volume of jobs could trigger rate limits; consider batching/throttling.

#### Node: Flatten AI scores into job fields
- **Type / role:** Code node; merges the AI output into the original paired item.
- **Configuration choices (as implemented):**
  - Pulls original items from node `Create candidate-job pairs` via `$items('Create candidate-job pairs')`.
  - Parses `items[i].json.output` (AI raw output string) as JSON.
  - Merges keys from the original job-pair object and parsed analysis into a single object.
- **Connections:**
  - **Main output →** `Parse AI JSON output (safe)`
- **Edge cases / failures:**
  - Index alignment assumption: AI results order must match `$items('Create candidate-job pairs')` order 1:1.
  - If parsing fails, analysis becomes `{}` and scores missing → downstream fit filter may treat as 0 or blank.

#### Node: Parse AI JSON output (safe)
- **Type / role:** Code node; robust parser and attaches parsed analysis under `analysis`.
- **Configuration choices (as implemented):**
  - Strips triple backticks (including ```json) using regex with `\x60`.
  - Attempts JSON parsing; if fails, tries unescaping; if still fails, returns:
    - `{ _parse_error: 'Could not parse model output', _raw_preview: ... }`
  - Merges the existing item fields with `analysis: <parsedJson>`
- **Connections:**
  - **Main output →** `Normalize job data for output`
- **Edge cases / failures:**
  - If upstream already flattened fields, this node may be redundant; but it adds resilience and preserves raw parse error context.
  - If AI output is extremely malformed, analysis will be parse error object.

#### Node: Normalize job data for output
- **Type / role:** Code node; produces a consistent schema for downstream storage/filtering/email.
- **Configuration choices (as implemented):**
  - Attempts to locate job data as `item.job` else uses `item`.
  - Also attempts to parse `item.output` again (belt-and-suspenders).
  - Uses a `pick()` helper to prioritize values across multiple potential locations:
    - direct flattened fields, parsed fields, or defaults.
  - Output fields:
    - `title`, `company`, `url`, `date_posted`, `currency`,
    - `min_salary`, `max_salary`, `type`, `locations`,
    - `skill_match_percentage`, `job_complexity_score`, `fit_score`,
    - `final_recommendation`, `summary`
- **Connections:**
  - **Main output →** `Save matches to Google Sheets`
  - **Main output →** `Check whether the fit score is greater than X`
- **Edge cases / failures:**
  - If fit_score becomes empty string or non-numeric, IF node numeric comparison may fail strict validation.
  - Locations may be an array or string; normalized to comma-separated string.

---

### Block 1.4 — Persistence & Notification

**Overview:** Stores all normalized job results in Google Sheets, filters by fit score threshold, formats a Top 5 HTML email, and sends it via Gmail.  
**Nodes Involved:**  
- Save matches to Google Sheets  
- Check whether the fit score is greater than X  
- Build Top 5 email (HTML)  
- Send Top 5 email (Gmail)

#### Node: Save matches to Google Sheets
- **Type / role:** Google Sheets node; appends rows to an output sheet for tracking.
- **Configuration choices:**
  - Operation: **Append**
  - Document: `YOUR_GOOGLE_SHEET_ID` (placeholder)
  - Sheet: `Sheet1` (cached name “output”)
  - Defines mapping for columns like url, name, type, summary, currency, fit_score, salaries, date_posted, description, complexity, skill match.
- **Important integration issue (as configured):**
  - The mappings reference fields such as `job_url`, `job_title`, `job_employment_type`, `job_currency`, `job_max_salary`, etc.
  - However, **the normalized output node provides** `url`, `title`, `type`, `currency`, `max_salary`, `min_salary`, etc.  
  - Unless earlier nodes actually output `job_*` fields (they do not, in the provided normalization), this node will append mostly blank cells.
- **Connections:** Receives from “Normalize job data for output”.
- **Edge cases / failures:**
  - OAuth/permission errors.
  - Column mismatch if the sheet headers don’t exist or differ.
  - Data type conversion disabled; numeric values may be stored as strings.

#### Node: Check whether the fit score is greater than X
- **Type / role:** IF node; filters jobs for email shortlist.
- **Configuration choices:**
  - Condition: `{{$json.fit_score}} > 40` (number comparison)
  - Strict type validation enabled by default options.
- **Connections:**
  - **True output →** `Build Top 5 email (HTML)`
  - False output is unused (jobs below threshold are ignored for email).
- **Edge cases / failures:**
  - If `fit_score` is missing/blank/non-numeric → strict numeric comparison may behave unexpectedly or evaluate to false.

#### Node: Build Top 5 email (HTML)
- **Type / role:** Code node; builds a styled HTML email body using first 5 incoming items.
- **Configuration choices (as implemented):**
  - Escapes HTML special characters.
  - `jobs = items.slice(0, 5)` (assumes incoming items are already filtered and ideally sorted—no sorting is implemented in workflow).
  - Renders job title, company, locations, salary, fit score, short summary, and “skill match” percentage bar.
  - Includes footer timestamp `new Date().toISOString()`.
- **Connections:**
  - **Main output →** `Send Top 5 email (Gmail)`
- **Edge cases / failures:**
  - If more than 5 items and you want “Top 5 by fit_score”, you must add a sort step; current behavior is “first 5 passing items.”
  - If URL is missing, “Open job” link may be empty.

#### Node: Send Top 5 email (Gmail)
- **Type / role:** Gmail node; sends the generated HTML.
- **Configuration choices:**
  - To: `user@example.com` (placeholder)
  - Subject: `Your Top 5 Job Matches`
  - Message body: `={{ $json.html }}`
- **Credentials:** Gmail OAuth2
- **Edge cases / failures:**
  - OAuth token expiry / insufficient Gmail scopes.
  - Gmail sending limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow description and setup notes | — | — | ## What this workflow does… (includes link: https://visit.decodo.com/raqXGD) |
| Sticky Note1 | Sticky Note | Block label: Candidate profile input | — | — | ## 1. Candidate profile input Reads the candidate profile from Google Sheets. |
| Sticky Note2 | Sticky Note | Block label: Data collection & AI scoring | — | — | ##  2. Data collection & AI scoring … |
| Sticky Note3 | Sticky Note | Block label: Results & notifications | — | — | ## 3.Results & notifications … |
| OpenAI Chat Model | LangChain OpenAI Chat Model | Provides LLM to the agent | — | Score fit with AI (salary + skills + industry) | ##  2. Data collection & AI scoring … |
| Sticky Note4 | Sticky Note | Output screenshot (visual) | — | — | ## Output ![txt](https://ik.imagekit.io/agbb7sr41/ouput_2_job_matching.png) |
| Sticky Note5 | Sticky Note | Input screenshot (visual) | — | — | ## Input ![txt](https://ik.imagekit.io/agbb7sr41/job_input.png) |
| Sticky Note6 | Sticky Note | Output screenshot (visual) | — | — | ## Output ![txt](https://ik.imagekit.io/agbb7sr41/job_matching_output.png) |
| Daily trigger (job scan) | Schedule Trigger | Runs workflow on schedule | — | Load candidate profile (Google Sheets) | ## What this workflow does… |
| Load candidate profile (Google Sheets) | Google Sheets | Reads candidate profile | Daily trigger (job scan) | Fetch RemoteOK HTML (Decodo); Merge profile + job list | ## 1. Candidate profile input… |
| Fetch RemoteOK HTML (Decodo) | Decodo | Downloads RemoteOK HTML | Load candidate profile (Google Sheets) | Extract JobPosting JSON | ##  2. Data collection & AI scoring … |
| Extract JobPosting JSON | Code | Parses JSON-LD JobPosting objects | Fetch RemoteOK HTML (Decodo) | Merge profile + job list | ##  2. Data collection & AI scoring … |
| Merge profile + job list | Merge | Combines candidate + jobs stream | Load candidate profile (Google Sheets); Extract JobPosting JSON | Create candidate-job pairs | ##  2. Data collection & AI scoring … |
| Create candidate-job pairs | Code | Creates {candidate, job} per job | Merge profile + job list | Score fit with AI (salary + skills + industry) | ##  2. Data collection & AI scoring … |
| Score fit with AI (salary + skills + industry) | LangChain Agent | AI scoring and recommendation | Create candidate-job pairs (+ model from OpenAI Chat Model) | Flatten AI scores into job fields | ##  2. Data collection & AI scoring … |
| Flatten AI scores into job fields | Code | Merges AI output into job object | Score fit with AI (salary + skills + industry) | Parse AI JSON output (safe) | ##  2. Data collection & AI scoring … |
| Parse AI JSON output (safe) | Code | Robust parsing + attach analysis | Flatten AI scores into job fields | Normalize job data for output | ## 3.Results & notifications… |
| Normalize job data for output | Code | Produces consistent output schema | Parse AI JSON output (safe) | Save matches to Google Sheets; Check whether the fit score is greater than X | ## 3.Results & notifications… |
| Save matches to Google Sheets | Google Sheets | Append matches to tracking sheet | Normalize job data for output | — | ## 3.Results & notifications… |
| Check whether the fit score is greater than X | IF | Filter for shortlist (fit_score > 40) | Normalize job data for output | Build Top 5 email (HTML) | ## 3.Results & notifications… |
| Build Top 5 email (HTML) | Code | Build HTML summary for top 5 items | Check whether the fit score is greater than X (true path) | Send Top 5 email (Gmail) | ## 3.Results & notifications… |
| Send Top 5 email (Gmail) | Gmail | Sends the HTML email | Build Top 5 email (HTML) | — | ## 3.Results & notifications… |
| Sticky Note7 | Sticky Note | Empty (layout only) | — | — |  |
| Sticky Note8 | Sticky Note | Empty (layout only) | — | — |  |
| Sticky Note9 | Sticky Note | Empty (layout only) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: *Find High-Quality Remote Jobs with AI, Decodo, and Google Sheets*.

2. **Add Trigger**
   1. Add **Schedule Trigger** node named **“Daily trigger (job scan)”**.
   2. Configure it to run **daily** (choose your desired time).

3. **Create Candidate Profile sheet**
   1. In Google Sheets, create a spreadsheet (or reuse one) with a tab for the profile (e.g., `Sheet1`).
   2. Add columns that your profile will provide, at minimum:
      - `skills`
      - `salary_expectation`
      - `preferred_industries`
      - `no_go`
   3. Ensure **only one row** represents the candidate profile (to match current pairing logic).

4. **Add Google Sheets node to read profile**
   1. Add **Google Sheets** node named **“Load candidate profile (Google Sheets)”**.
   2. Set **Credentials:** Google Sheets OAuth2.
   3. Select **Document** (your spreadsheet) and **Sheet** (profile sheet).
   4. Choose a read operation (commonly **Get All**).
   5. Connect: **Daily trigger → Load candidate profile**.

5. **Add Decodo fetch**
   1. Add Decodo node named **“Fetch RemoteOK HTML (Decodo)”**.
   2. Set **Credentials:** Decodo API credential.
   3. URL: `https://remoteok.com/remote-technical-jobs?location=Worldwide&min_salary=120000`
   4. Connect: **Load candidate profile → Fetch RemoteOK HTML (Decodo)**.

6. **Add HTML → JSON-LD extractor**
   1. Add **Code** node named **“Extract JobPosting JSON”**.
   2. Paste logic equivalent to: scan HTML for JSON-LD JobPosting scripts, parse JSON, output items with job fields (title, company, salaries, url, description, etc.).
   3. Connect: **Fetch RemoteOK HTML → Extract JobPosting JSON**.

7. **Merge candidate profile + job list**
   1. Add **Merge** node named **“Merge profile + job list”**.
   2. Configure merge mode so that output is a **single list** where the **first item is the candidate**, followed by **job items** (Append-style behavior).
   3. Connect:
      - **Load candidate profile → Merge** (Input 0)
      - **Extract JobPosting JSON → Merge** (Input 1)

8. **Create candidate-job pairs**
   1. Add **Code** node named **“Create candidate-job pairs”**.
   2. Configure to:
      - Take `items[0]` as candidate
      - For each subsequent item, output `{ candidate, job }`
   3. Connect: **Merge profile + job list → Create candidate-job pairs**.

9. **Add OpenAI model provider**
   1. Add **OpenAI Chat Model** node named **“OpenAI Chat Model”**.
   2. Set credentials: **OpenAI API**.
   3. Select model: **gpt-4o-mini** (or another supported model).

10. **Add AI Agent scoring**
    1. Add **AI Agent** node named **“Score fit with AI (salary + skills + industry)”**.
    2. Connect **OpenAI Chat Model → Agent** using the **AI language model** connection type.
    3. In the Agent prompt, instruct it to return **only valid JSON** with fields:
       - fit_score, salary_score, skill_match_percentage, industry_match, industry_match_score, etc.
       - Use weights 40/40/20 (salary/skills/industry).
    4. Connect: **Create candidate-job pairs → Score fit with AI**.

11. **Flatten and parse AI output**
    1. Add **Code** node **“Flatten AI scores into job fields”** to merge AI output JSON into the original job/candidate structure.
    2. Add **Code** node **“Parse AI JSON output (safe)”** to handle code fences, string-wrapped JSON, and parse failures.
    3. Connect:
       - **Score fit with AI → Flatten AI scores**
       - **Flatten AI scores → Parse AI JSON output**

12. **Normalize output schema**
    1. Add **Code** node named **“Normalize job data for output”** producing consistent fields:
       - `title, company, url, date_posted, currency, min_salary, max_salary, type, locations, skill_match_percentage, job_complexity_score, fit_score, final_recommendation, summary`
    2. Connect: **Parse AI JSON output → Normalize job data for output**.

13. **Store in Google Sheets (tracking output)**
    1. Create an “output” sheet with headers matching what you want to store.
    2. Add **Google Sheets** node named **“Save matches to Google Sheets”** with **Append** operation.
    3. Map columns.
       - Recommended: map to the normalized field names (`url`, `title`, `type`, `currency`, `min_salary`, `max_salary`, `fit_score`, `skill_match_percentage`, `job_complexity_score`, `summary`, `date_posted`).
       - If you keep the provided `job_*` mappings, you must also change the normalization to output those `job_*` keys.
    4. Connect: **Normalize job data for output → Save matches to Google Sheets**.

14. **Filter by fit score**
    1. Add **IF** node named **“Check whether the fit score is greater than X”**.
    2. Condition: `fit_score` **greater than** `40` (adjust threshold as desired).
    3. Connect: **Normalize job data for output → IF**.

15. **Build Top 5 email**
    1. Add **Code** node named **“Build Top 5 email (HTML)”**.
    2. Implement:
       - Take incoming items (already filtered)
       - Keep first 5 (or sort first, then take 5)
       - Build HTML with title/company/locations/salary/fit/summary and links
    3. Connect: **IF (true) → Build Top 5 email (HTML)**.

16. **Send email via Gmail**
    1. Add **Gmail** node named **“Send Top 5 email (Gmail)”**.
    2. Set **Credentials:** Gmail OAuth2.
    3. To: your email address.
    4. Subject: `Your Top 5 Job Matches`
    5. Body: map from `{{$json.html}}` and ensure it sends as HTML (depending on node options/version).
    6. Connect: **Build Top 5 email → Send Top 5 email**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Decodo – Web Scraper for n8n | https://visit.decodo.com/raqXGD |
| Workflow output screenshot | https://ik.imagekit.io/agbb7sr41/ouput_2_job_matching.png |
| Workflow input screenshot | https://ik.imagekit.io/agbb7sr41/job_input.png |
| Job matching output screenshot | https://ik.imagekit.io/agbb7sr41/job_matching_output.png |

