Automate LinkedIn job search, AI scoring, and resume creation with OpenRouter, OpenAI, Google Sheets & Docs

https://n8nworkflows.xyz/workflows/automate-linkedin-job-search--ai-scoring--and-resume-creation-with-openrouter--openai--google-sheets---docs-14037


# Automate LinkedIn job search, AI scoring, and resume creation with OpenRouter, OpenAI, Google Sheets & Docs

# 1. Workflow Overview

This workflow automates a semi-supervised job application pipeline centered on LinkedIn job discovery, AI-based qualification, and tailored application document generation.

Its main purpose is to:

- scrape recent LinkedIn job offers through Apify,
- filter irrelevant postings,
- evaluate relevance against a custom candidate profile using two AI stages,
- store strong matches in Google Sheets,
- notify the user via Telegram,
- and, when a sheet row is updated to indicate intent to apply, generate a customized resume and cover letter in Google Docs.

The workflow has **two explicit entry points**:

1. **Schedule Trigger** for recurring job discovery and scoring.
2. **Google Sheets Trigger** for resume/cover-letter generation after manual action in the spreadsheet.

## 1.1 Scheduled Job Discovery

A cron-based trigger launches the scraping process several times per day, along with loading a reusable candidate master profile.

## 1.2 Job Filtering and Preparation

Scraped jobs are filtered to remove undesirable recruiter companies and to keep only jobs in the target country. The HTML job description is then converted into Markdown text for AI consumption.

## 1.3 First-Stage AI Qualification

A strict “GateKeeper” LLM performs a fast reject/accept decision using the job description and a lightweight candidate profile logic. Only jobs whose returned text contains `"match": true` continue.

## 1.4 Second-Stage AI Scoring and Persistence

Accepted jobs are scored more precisely by a second LLM against the full master profile. Its JSON output is parsed, merged with the original job data, filtered by a score threshold, written to Google Sheets, and followed by a Telegram notification.

## 1.5 Manual Apply Trigger and Resume Generation

A Google Sheets row update triggers the second half of the workflow. Rows with `Status = apply` are loaded, a full profile is injected, and OpenAI generates a structured JSON response containing a tailored resume and cover letter.

## 1.6 Language-Based Document Creation

The generated language code routes the workflow into an English or German branch. Each branch copies Google Docs templates, replaces placeholders with generated values, and updates the source Google Sheet with document links and final status.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled scraping and profile initialization

### Overview
This block starts the recurring job-search pipeline. It triggers execution on a schedule, loads the candidate’s master profile, and calls the Apify LinkedIn jobs scraper.

### Nodes Involved
- Schedule Trigger
- Set Profile
- Scrape Linkedin w/Apify
- Sticky Note1
- Sticky Note - Overview

### Node Details

#### 1. Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point for periodic execution.
- **Configuration:** Uses a cron expression `0 0 7-23 * * *`, meaning it runs every hour from 07:00 through 23:00, at minute 0 and second 0.
- **Key expressions / variables:** None.
- **Connections:**  
  - Outputs to **Set Profile**
  - Outputs to **Scrape Linkedin w/Apify**
- **Version-specific notes:** Type version `1.2`.
- **Failure / edge cases:**  
  - Timezone assumptions matter; n8n instance timezone will affect actual run time.
  - Overlapping executions may occur if prior runs take longer than one hour.
- **Sub-workflow reference:** None.

#### 2. Set Profile
- **Type / role:** `n8n-nodes-base.set` — stores the reusable master career profile as a string field called `Profile`.
- **Configuration:** Creates a long Markdown-formatted profile template with sections for identity, experience, skills, education, and AI usage instructions.
- **Key expressions / variables:**  
  - Outputs `Profile`
- **Connections:**  
  - Receives input from **Schedule Trigger**
  - Its output is referenced later by **Job Match Scorer**
- **Version-specific notes:** Type version `3.4`.
- **Failure / edge cases:**  
  - If left uncustomized, the AI evaluation becomes low quality or misleading.
  - Since it is not directly chained into the scraping path, downstream nodes reference it by node name using expressions like `$('Set Profile').first().json.Profile`; if this node fails or is removed, those expressions break.
- **Sub-workflow reference:** None.

#### 3. Scrape Linkedin w/Apify
- **Type / role:** `n8n-nodes-base.httpRequest` — calls the Apify actor endpoint synchronously and retrieves dataset items.
- **Configuration:**  
  - Method: `POST`
  - URL: `https://api.apify.com/v2/acts/curious_coder~linkedin-jobs-scraper/run-sync-get-dataset-items`
  - Body is JSON with:
    - `count: 100`
    - `scrapeCompany: true`
    - `splitByLocation: false`
    - `urls`: array of LinkedIn search URLs
  - Authentication uses **generic credential type** with **httpQueryAuth**, suitable for passing an API token in the query string.
- **Key expressions / variables:** Static JSON body, but requires manual replacement of:
  - `YOUR_GEO_ID`
  - `YOUR_KEYWORD_1`
  - `YOUR_KEYWORD_2`
- **Connections:**  
  - Receives input from **Schedule Trigger**
  - Outputs to **Remove Big Recruiters**
- **Version-specific notes:** Type version `4.2`.
- **Failure / edge cases:**  
  - Invalid Apify token or query auth setup.
  - Apify actor downtime or quota exhaustion.
  - Bad LinkedIn URLs produce poor or empty results.
  - Large datasets can slow execution.
  - Synchronous actor execution can timeout if scraping is slow.
- **Sub-workflow reference:** None.

#### 4. Sticky Note1
- **Type / role:** `stickyNote` — documentation only.
- **Content summary:** Explains that this section scrapes LinkedIn via Apify and requires API key plus customized search URLs.
- **Connections:** None.
- **Failure / edge cases:** None.

#### 5. Sticky Note - Overview
- **Type / role:** `stickyNote` — high-level workflow summary.
- **Content summary:** Describes the end-to-end automation flow and warns that setup is required.
- **Connections:** None.
- **Failure / edge cases:** None.

---

## Block 2 — Filtering and job description preparation

### Overview
This block reduces noise before AI scoring. It removes large recruiter companies, keeps only jobs in the desired country, and converts HTML job descriptions to Markdown text for downstream LLM nodes.

### Nodes Involved
- Remove Big Recruiters
- Keep only [Your Country]
- Job Description
- Sticky Note - Filter

### Node Details

#### 1. Remove Big Recruiters
- **Type / role:** `n8n-nodes-base.filter` — excludes jobs from named recruiter companies.
- **Configuration:** Uses multiple `notEquals` conditions against `companyName`.
- **Key expressions / variables:**  
  - `$('Scrape Linkedin w/Apify').item.json.companyName`
  - `$json.companyName`
  - Right-side values are placeholders like `Recruiter Company 1` through `Recruiter Company 6`
- **Connections:**  
  - Receives from **Scrape Linkedin w/Apify**
  - Outputs to **Keep only [Your Country]**
- **Version-specific notes:** Type version `2.2`.
- **Failure / edge cases:**  
  - Exact string matching means variant company names may slip through.
  - Placeholder conditions must be customized.
  - Mixed use of `$('Scrape Linkedin w/Apify')` and `$json` works but is inconsistent and may be harder to maintain.
- **Sub-workflow reference:** None.

#### 2. Keep only [Your Country]
- **Type / role:** `n8n-nodes-base.filter` — keeps only jobs whose location contains the target country string.
- **Configuration:** Single `contains` condition on the job `location`.
- **Key expressions / variables:**  
  - `$('Scrape Linkedin w/Apify').item.json.location`
  - Right value: `YOUR_COUNTRY`
- **Connections:**  
  - Receives from **Remove Big Recruiters**
  - Outputs to **Job Description**
- **Version-specific notes:** Type version `2.3`.
- **Failure / edge cases:**  
  - Must be customized.
  - `contains` is case-sensitive due to strict settings, so formatting differences may exclude valid jobs.
  - Country may be omitted or represented as region/city only.
- **Sub-workflow reference:** None.

#### 3. Job Description
- **Type / role:** `n8n-nodes-base.markdown` — converts HTML into Markdown/plain structured text.
- **Configuration:**  
  - Input HTML comes from the filtered job item’s `descriptionHtml`
  - Output field is `job_description`
- **Key expressions / variables:**  
  - `{{ $('Keep only CH').item.json.descriptionHtml }}`
- **Important note:** The expression references a node named **Keep only CH**, but the actual node in this workflow is named **Keep only [Your Country]**.
- **Connections:**  
  - Receives from **Keep only [Your Country]**
  - Outputs to **GateKeeper**
  - Outputs to **Merge w/ Original Data**
- **Version-specific notes:** Type version `1`.
- **Failure / edge cases:**  
  - **Likely broken expression** unless the node was renamed at some point. This is one of the most important repair points before use.
  - Poorly formatted or malformed HTML can yield noisy Markdown.
- **Sub-workflow reference:** None.

#### 4. Sticky Note - Filter
- **Type / role:** `stickyNote` — documentation only.
- **Content summary:** Explains recruiter/country filtering and the two-stage AI evaluation.
- **Connections:** None.

---

## Block 3 — Gatekeeper AI evaluation

### Overview
This block performs fast first-pass qualification. It uses a cheaper OpenRouter model with a strict recruiter-style prompt and only passes jobs whose raw text output contains `"match": true`.

### Nodes Involved
- GateKeeper
- Cheap Model
- Merge w/ Original Data
- Keep Matches

### Node Details

#### 1. Cheap Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — LLM provider node for the first screening stage.
- **Configuration:** Uses OpenRouter credentials; no explicit model is set in parameters, so it depends on default/account configuration.
- **Key expressions / variables:** None.
- **Connections:**  
  - Feeds AI language model input into **GateKeeper**
- **Version-specific notes:** Type version `1`.
- **Failure / edge cases:**  
  - OpenRouter auth issues.
  - Unclear default model may change behavior over time; setting a fixed model would be safer.
  - Rate limits and model availability can affect throughput.
- **Sub-workflow reference:** None.

#### 2. GateKeeper
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — first-stage classification LLM.
- **Configuration:**  
  - Prompt instructs the model to behave as a cynical recruiter and reject most jobs.
  - Uses a simplified embedded candidate profile section with manual placeholders.
  - Expects JSON-only output:
    - `match`
    - `score`
    - `reason`
  - `onError: continueErrorOutput` is enabled.
- **Key expressions / variables:**  
  - `{{ $json.job_description }}`
- **Connections:**  
  - Receives main input from **Job Description**
  - Receives model from **Cheap Model**
  - Outputs to **Merge w/ Original Data**
- **Version-specific notes:** Type version `1.9`.
- **Failure / edge cases:**  
  - Prompt must be customized manually; otherwise screening logic is generic.
  - The model may return fenced JSON or explanatory text despite instructions.
  - Because downstream routing checks raw text with string matching, formatting deviations can break routing.
  - `continueErrorOutput` means failures may not stop execution, but can produce malformed downstream items.
- **Sub-workflow reference:** None.

#### 3. Merge w/ Original Data
- **Type / role:** `n8n-nodes-base.merge` — recombines the GateKeeper result with the original job record.
- **Configuration:**  
  - Mode: `combine`
  - Combine by: position
- **Key expressions / variables:** None.
- **Connections:**  
  - Input 0 from **Job Description**
  - Input 1 from **GateKeeper**
  - Output to **Keep Matches**
- **Version-specific notes:** Type version `3.2`.
- **Failure / edge cases:**  
  - Combining by position assumes both branches preserve the same item order and count.
  - If GateKeeper skips, errors, or returns mismatched item counts, jobs may be paired with wrong AI results.
- **Sub-workflow reference:** None.

#### 4. Keep Matches
- **Type / role:** `n8n-nodes-base.switch` — routes only jobs that were accepted by GateKeeper.
- **Configuration:**  
  - Checks whether GateKeeper raw text contains `"match": true`
  - `allMatchingOutputs: false`
- **Key expressions / variables:**  
  - `{{ $('GateKeeper').item.json.text }}`
- **Connections:**  
  - Receives from **Merge w/ Original Data**
  - Output goes to **Job Match Scorer**
  - Same output also goes to **Merge w/ Original Data1**
- **Version-specific notes:** Type version `3.4`.
- **Failure / edge cases:**  
  - Brittle string-based detection.
  - If the model returns `true` with whitespace/formatting differences, or invalid JSON, matches may be missed.
  - Better practice would be to parse the JSON first.
- **Sub-workflow reference:** None.

---

## Block 4 — Detailed scoring, parsing, sheet append, and notification

### Overview
This block performs a deeper fit analysis on jobs that passed the GateKeeper. It scores them against the full master profile, parses the JSON, merges results back with source data, keeps only high scores, appends them to Google Sheets, and notifies the user on Telegram.

### Nodes Involved
- Job Match Scorer
- Better Model
- Parse in JSON
- Merge w/ Original Data1
- Keep Matches1
- Add Roles to apply
- Send a text message
- Sticky Note - Score

### Node Details

#### 1. Better Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — stronger LLM provider for detailed scoring.
- **Configuration:** Explicit model: `openai/gpt-4.1`.
- **Key expressions / variables:** None.
- **Connections:**  
  - Feeds **Job Match Scorer**
- **Version-specific notes:** Type version `1`.
- **Failure / edge cases:**  
  - OpenRouter billing/model availability/rate limits.
  - Higher cost than the cheap model.
- **Sub-workflow reference:** None.

#### 2. Job Match Scorer
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — second-stage scoring LLM.
- **Configuration:**  
  - Uses the full master profile from **Set Profile**
  - Compares profile against `job_description`
  - Requires strict raw JSON output:
    - `match_score`
    - `reasoning`
    - `missing_critical_skills`
    - `matching_strong_skills`
- **Key expressions / variables:**  
  - `{{ $('Set Profile').first().json.Profile }}`
  - `{{ $('Job Description').item.json.job_description }}`
- **Connections:**  
  - Receives main input from **Keep Matches**
  - Receives model from **Better Model**
  - Outputs to **Parse in JSON**
- **Version-specific notes:** Type version `1.9`.
- **Failure / edge cases:**  
  - AI may still emit extra text or markdown fences.
  - If **Set Profile** is empty or generic, scores lose meaning.
  - If **Job Description** expression is broken upstream, this node receives bad text.
- **Sub-workflow reference:** None.

#### 3. Parse in JSON
- **Type / role:** `n8n-nodes-base.code` — extracts and parses JSON from the scorer’s raw text.
- **Configuration:**  
  - Finds the first `{` and last `}`
  - Slices the text between them
  - Parses JSON
  - Merges parsed fields into the current item
- **Key expressions / variables:**  
  - Uses JavaScript over `$input.all()`
  - Reads `item.json.text`
- **Connections:**  
  - Receives from **Job Match Scorer**
  - Outputs to **Merge w/ Original Data1**
- **Version-specific notes:** Type version `2`.
- **Failure / edge cases:**  
  - Throws an error if no braces are found.
  - Will fail if the JSON is malformed despite brace extraction.
  - If the model returns nested non-JSON text with braces, parsing can fail.
- **Sub-workflow reference:** None.

#### 4. Merge w/ Original Data1
- **Type / role:** `n8n-nodes-base.merge` — merges accepted original jobs with parsed score details.
- **Configuration:**  
  - Mode: `combine`
  - Combine by position
- **Key expressions / variables:** None.
- **Connections:**  
  - Input 0 from **Keep Matches**
  - Input 1 from **Parse in JSON**
  - Output to **Keep Matches1**
- **Version-specific notes:** Type version `3.2`.
- **Failure / edge cases:**  
  - Same positional merge risk as earlier.
  - Any parsing failure can desynchronize results.
- **Sub-workflow reference:** None.

#### 5. Keep Matches1
- **Type / role:** `n8n-nodes-base.switch` — final threshold filter.
- **Configuration:** Keeps only items where `match_score >= 80`.
- **Key expressions / variables:**  
  - `{{ $json.match_score }}`
- **Connections:**  
  - Receives from **Merge w/ Original Data1**
  - Outputs to **Add Roles to apply**
- **Version-specific notes:** Type version `3.4`.
- **Failure / edge cases:**  
  - If `match_score` is parsed as string rather than number, comparison behavior should be checked.
  - Threshold is hardcoded; may need tuning.
- **Sub-workflow reference:** None.

#### 6. Add Roles to apply
- **Type / role:** `n8n-nodes-base.googleSheets` — appends validated jobs to the spreadsheet.
- **Configuration:**  
  - Operation: `append`
  - Mapping mode: auto-map input data
  - Target sheet: `Validated`
  - Uses append mode
  - Matching column is `id`, though append mode means it primarily inserts rows
- **Key expressions / variables:**  
  - Google Spreadsheet document and sheet IDs are placeholders.
- **Connections:**  
  - Receives from **Keep Matches1**
  - Outputs to **Send a text message**
- **Version-specific notes:** Type version `4.7`.
- **Failure / edge cases:**  
  - Google OAuth permissions.
  - Wrong spreadsheet or sheet IDs.
  - Schema drift if columns differ from expected names.
  - Duplicate rows can accumulate if scraping returns already-seen job IDs and append mode is used.
- **Sub-workflow reference:** None.

#### 7. Send a text message
- **Type / role:** `n8n-nodes-base.telegram` — sends a notification that new jobs were added.
- **Configuration:**  
  - Sends a fixed message with a Google Sheets link
  - `appendAttribution: false`
  - Uses a configured bot token credential
- **Key expressions / variables:**  
  - Static sheet URL placeholder
  - `chatId` must be customized
- **Connections:**  
  - Receives from **Add Roles to apply**
- **Version-specific notes:** Type version `1.2`.
- **Failure / edge cases:**  
  - Invalid Telegram bot token or chat ID.
  - Message may send once per item if multiple rows are appended in one run; depending on n8n execution behavior, this can create notification spam.
- **Sub-workflow reference:** None.

#### 8. Sticky Note - Score
- **Type / role:** `stickyNote` — documentation only.
- **Content summary:** Explains parsing, merging, filtering, Google Sheets append, and Telegram notification.
- **Connections:** None.

---

## Block 5 — Manual apply trigger and profile injection

### Overview
This block starts when the user updates the spreadsheet. It listens for row updates, reloads the candidate master profile, and fetches rows whose status indicates that application documents should be generated.

### Nodes Involved
- Google Sheets Trigger
- Set Profile1
- Get Job Description
- Sticky Note - Resume
- Sticky Note - Setup Guide

### Node Details

#### 1. Google Sheets Trigger
- **Type / role:** `n8n-nodes-base.googleSheetsTrigger` — second workflow entry point.
- **Configuration:**  
  - Event: `rowUpdate`
  - Polling every hour at minute 10
  - Watches the same `Validated` sheet
- **Key expressions / variables:** None.
- **Connections:**  
  - Outputs to **Set Profile1**
- **Version-specific notes:** Type version `1`.
- **Failure / edge cases:**  
  - Google trigger auth issues.
  - Polling triggers are not instant.
  - A row update anywhere may trigger execution; later filtering decides whether the row is relevant.
- **Sub-workflow reference:** None.

#### 2. Set Profile1
- **Type / role:** `n8n-nodes-base.set` — duplicate of the profile node for the second branch.
- **Configuration:** Stores the same `Profile` content used for generation.
- **Key expressions / variables:**  
  - Outputs `Profile`
- **Connections:**  
  - Receives from **Google Sheets Trigger**
  - Outputs to **Get Job Description**
- **Version-specific notes:** Type version `3.4`.
- **Failure / edge cases:**  
  - Must be kept in sync with **Set Profile** manually.
  - If customized in one place but not the other, scoring and resume generation may use different candidate profiles.
- **Sub-workflow reference:** None.

#### 3. Get Job Description
- **Type / role:** `n8n-nodes-base.googleSheets` — fetches rows to process.
- **Configuration:**  
  - Reads from the `Validated` sheet
  - Applies a filter where `Status = apply`
- **Key expressions / variables:** None.
- **Connections:**  
  - Receives from **Set Profile1**
  - Outputs to **Create Resume **
- **Version-specific notes:** Type version `4.7`.
- **Failure / edge cases:**  
  - This node filters the full sheet for all rows with `Status = apply`, not necessarily only the row that changed.
  - Multiple apply rows could all be processed in a single run.
  - If the sheet does not contain a `Status` column, no results.
- **Sub-workflow reference:** None.

#### 4. Sticky Note - Resume
- **Type / role:** `stickyNote` — documentation only.
- **Content summary:** Describes the AI-generated resume/cover-letter phase and language branching.
- **Connections:** None.

#### 5. Sticky Note - Setup Guide
- **Type / role:** `stickyNote` — documentation only.
- **Content summary:** Lists required credentials, sheet setup, templates, customization points.
- **Connections:** None.

---

## Block 6 — Resume and cover letter generation

### Overview
This block uses OpenAI to generate a tailored resume and cover letter in strict JSON format based on the selected job and the stored master profile.

### Nodes Involved
- Create Resume
- Select Language

### Node Details

#### 1. Create Resume 
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI generation node for structured output.
- **Configuration:**  
  - Model: `gpt-5.1`
  - Text format: `json_object`
  - Prompt instructs the model to:
    - use only the supplied profile,
    - avoid hallucinations,
    - detect job language,
    - generate resume + cover letter,
    - output strict JSON with `meta.language`, `resume`, and `cover_letter`.
- **Key expressions / variables:**  
  - `{{ $('Set Profile1').first().json.Profile }}`
  - `{{ $json.job_description }}`
- **Connections:**  
  - Receives from **Get Job Description**
  - Outputs to **Select Language**
- **Version-specific notes:** Type version `2.1`.
- **Failure / edge cases:**  
  - OpenAI auth/quota/model access issues.
  - The node expects structured text output; if model behavior changes, downstream field paths may break.
  - Very long job descriptions plus profile may hit token limits.
- **Sub-workflow reference:** None.

#### 2. Select Language
- **Type / role:** `n8n-nodes-base.switch` — routes document generation by language code.
- **Configuration:**  
  - Output `EN` if language equals `EN`
  - Output `DE` if language equals `DE`
  - Uses renamed outputs
- **Key expressions / variables:**  
  - `{{ $json.output[0].content[0].text.meta.language }}`
- **Connections:**  
  - Receives from **Create Resume **
  - Output 0 to **Make a Copy of EN Resume**
  - Output 1 to **Make a Copy of DE Resume**
- **Version-specific notes:** Type version `3.4`.
- **Failure / edge cases:**  
  - Field path is tightly coupled to current OpenAI node output structure.
  - If the model emits `FR`, lowercase, or another code, no branch will match.
  - Extending to more languages requires adding more switch outputs and downstream template branches.
- **Sub-workflow reference:** None.

---

## Block 7 — English document creation and sheet update

### Overview
This branch handles English-language output. It copies a resume template and a cover-letter template from Google Drive, replaces placeholders in Google Docs, and writes the generated document URLs back to Google Sheets.

### Nodes Involved
- Make a Copy of EN Resume
- Update all Resume Variables
- Make a Copy of EN CL
- Update all CL Variables
- Update Link & Status

### Node Details

#### 1. Make a Copy of EN Resume
- **Type / role:** `n8n-nodes-base.googleDrive` — duplicates the English resume template.
- **Configuration:**  
  - Operation: `copy`
  - Output file name: `Resume_<companyName>_<row_number>`
  - Source template file ID must be set manually.
- **Key expressions / variables:**  
  - `{{ $('Get Job Description').item.json.companyName }}`
  - `{{ $('Get Job Description').item.json.row_number }}`
- **Connections:**  
  - Receives from **Select Language** EN path
  - Outputs to **Update all Resume Variables**
- **Version-specific notes:** Type version `3`.
- **Failure / edge cases:**  
  - Invalid file ID or missing Drive permissions.
  - Company names with unsupported characters can affect filename readability.
- **Sub-workflow reference:** None.

#### 2. Update all Resume Variables
- **Type / role:** `n8n-nodes-base.googleDocs` — fills placeholders in the copied English resume.
- **Configuration:**  
  - Operation: `update`
  - Replaces placeholders like `{{summary}}`, `{{experience_1.title}}`, `{{skills_tech}}`, etc.
  - Uses output from **Create Resume **
- **Key expressions / variables:** Extensive references such as:
  - `$('Create Resume ').item.json.output[0].content[0].text.resume.summary`
  - similar paths for all experience fields and skill fields
  - document URL/input uses copied document ID from **Make a Copy of EN Resume**
- **Connections:**  
  - Receives from **Make a Copy of EN Resume**
  - Outputs to **Make a Copy of EN CL**
- **Version-specific notes:** Type version `2`.
- **Failure / edge cases:**  
  - Placeholder names in the Google Doc must exactly match the configured tokens.
  - Missing JSON fields from AI output will cause empty replacements or failures.
  - The node uses `documentURL` but passes the copied doc `id`; verify compatibility in your n8n version.
- **Sub-workflow reference:** None.

#### 3. Make a Copy of EN CL
- **Type / role:** `n8n-nodes-base.googleDrive` — duplicates the English cover-letter template.
- **Configuration:**  
  - Operation: `copy`
  - Output file name: `CoverLetter_<companyName>_<row_number>`
- **Key expressions / variables:** Same naming approach as the resume branch.
- **Connections:**  
  - Receives from **Update all Resume Variables**
  - Outputs to **Update all CL Variables**
- **Version-specific notes:** Type version `3`.
- **Failure / edge cases:** Same as other Drive copy nodes.
- **Sub-workflow reference:** None.

#### 4. Update all CL Variables
- **Type / role:** `n8n-nodes-base.googleDocs` — fills the English cover letter template.
- **Configuration:** Replaces:
  - `{{salutation}}`
  - `{{body_text}}`
  - `{{date}}`
  - `{{companyName}}`
- **Key expressions / variables:**  
  - Cover letter text from **Create Resume **
  - `{{$now.format('dd-MM-yyyy')}}`
  - Company name from **Get Job Description**
- **Connections:**  
  - Receives from **Make a Copy of EN CL**
  - Outputs to **Update Link & Status**
- **Version-specific notes:** Type version `2`.
- **Failure / edge cases:**  
  - Date format uses n8n’s expression runtime; ensure timezone/date expectations are correct.
  - Template placeholders must match exactly.
- **Sub-workflow reference:** None.

#### 5. Update Link & Status
- **Type / role:** `n8n-nodes-base.googleSheets` — updates the source spreadsheet row with generated links.
- **Configuration:**  
  - Operation: `update`
  - Matching column: `id`
  - Sets:
    - `Resume` to copied resume doc URL
    - `Cover Letter` to copied cover-letter doc URL
    - `Status` to `created`
- **Key expressions / variables:**  
  - `{{ $('Get Job Description').item.json.id }}`
  - resume link from **Update all Resume Variables**
  - cover letter link from **Update all CL Variables**
- **Connections:**  
  - Receives from **Update all CL Variables**
- **Version-specific notes:** Type version `4.7`.
- **Failure / edge cases:**  
  - Matching by `id` assumes uniqueness in the sheet.
  - Wrong schema or missing columns prevents updates.
- **Sub-workflow reference:** None.

---

## Block 8 — German document creation and sheet update

### Overview
This branch mirrors the English branch for German outputs. It creates a German resume (`Lebenslauf`) and cover letter (`Anschreiben`) from dedicated templates and updates the spreadsheet.

### Nodes Involved
- Make a Copy of DE Resume
- Update all Lebenslauf Variables
- Make a Copy of DE Motivation
- Update all Anschreiben Variables
- Update Link & Status for DE

### Node Details

#### 1. Make a Copy of DE Resume
- **Type / role:** `n8n-nodes-base.googleDrive`
- **Configuration:** Copies a German resume template and names it `Lebenslauf_<companyName>_<row_number>`.
- **Key expressions / variables:** Company name and row number from **Get Job Description**.
- **Connections:**  
  - Receives from **Select Language** DE path
  - Outputs to **Update all Lebenslauf Variables**
- **Version-specific notes:** Type version `3`.
- **Failure / edge cases:** Same as English branch.

#### 2. Update all Lebenslauf Variables
- **Type / role:** `n8n-nodes-base.googleDocs`
- **Configuration:** Same replacement structure as English resume, but applied to German template.
- **Key expressions / variables:** Reads the same AI JSON paths from **Create Resume **.
- **Connections:**  
  - Receives from **Make a Copy of DE Resume**
  - Outputs to **Make a Copy of DE Motivation**
- **Version-specific notes:** Type version `2`.
- **Failure / edge cases:**  
  - German template placeholders still use English-like token names; template must match exactly.
- **Sub-workflow reference:** None.

#### 3. Make a Copy of DE Motivation
- **Type / role:** `n8n-nodes-base.googleDrive`
- **Configuration:** Copies the German motivation letter template, naming it `Anschreiben_<companyName>_<row_number>`.
- **Connections:**  
  - Receives from **Update all Lebenslauf Variables**
  - Outputs to **Update all Anschreiben Variables**
- **Failure / edge cases:** Same as other copy nodes.

#### 4. Update all Anschreiben Variables
- **Type / role:** `n8n-nodes-base.googleDocs`
- **Configuration:** Replaces:
  - `{{salutation}}`
  - `{{body_text}}`
  - `{{date}}`
  - `{{companyName}}`
- **Key expressions / variables:** Same sourcing as English CL node.
- **Connections:**  
  - Receives from **Make a Copy of DE Motivation**
  - Outputs to **Update Link & Status for DE**
- **Version-specific notes:** Type version `2`.
- **Failure / edge cases:** Same as English CL branch.

#### 5. Update Link & Status for DE
- **Type / role:** `n8n-nodes-base.googleSheets`
- **Configuration:** Updates the row by `id`, storing German doc links and `Status = created`.
- **Key expressions / variables:**  
  - Resume URL from **Update all Lebenslauf Variables**
  - Cover letter URL from **Update all Anschreiben Variables**
- **Connections:**  
  - Receives from **Update all Anschreiben Variables**
- **Version-specific notes:** Type version `4.7`.
- **Failure / edge cases:** Same as English update node.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point for job scraping |  | Set Profile, Scrape Linkedin w/Apify | ## 🔎 Auto-Apply: Automated Job Search & Application Workflow<br>**What this does:**<br>1. Scrapes LinkedIn jobs on a schedule<br>2. Filters out unwanted recruiters & wrong locations<br>3. AI evaluates each job against YOUR profile (2-stage: GateKeeper + Scorer)<br>4. Matching jobs get added to your Google Sheet<br>5. Sends you a Telegram notification<br>6. When you trigger an application, it generates a tailored Resume + Cover Letter<br>7. Copies your Google Doc templates, fills in variables, and updates status<br><br>**⚠️ SETUP REQUIRED (see notes below)** |
| Set Profile | n8n-nodes-base.set | Stores master profile for job scoring | Schedule Trigger |  | ## 🔎 Auto-Apply: Automated Job Search & Application Workflow<br>**What this does:**<br>1. Scrapes LinkedIn jobs on a schedule<br>2. Filters out unwanted recruiters & wrong locations<br>3. AI evaluates each job against YOUR profile (2-stage: GateKeeper + Scorer)<br>4. Matching jobs get added to your Google Sheet<br>5. Sends you a Telegram notification<br>6. When you trigger an application, it generates a tailored Resume + Cover Letter<br>7. Copies your Google Doc templates, fills in variables, and updates status<br><br>**⚠️ SETUP REQUIRED (see notes below)** |
| Scrape Linkedin w/Apify | n8n-nodes-base.httpRequest | Calls Apify LinkedIn jobs scraper | Schedule Trigger | Remove Big Recruiters | ## 1️⃣ Scrape LinkedIn Jobs<br>Scrapes LinkedIn job listings via Apify.<br>**Setup:** Add your Apify API key and customize LinkedIn search URLs with your keywords and location. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Remove Big Recruiters | n8n-nodes-base.filter | Excludes recruiter company names | Scrape Linkedin w/Apify | Keep only [Your Country] | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| Keep only [Your Country] | n8n-nodes-base.filter | Keeps jobs in target country | Remove Big Recruiters | Job Description | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| Job Description | n8n-nodes-base.markdown | Converts HTML job description to Markdown | Keep only [Your Country] | GateKeeper, Merge w/ Original Data | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| GateKeeper | @n8n/n8n-nodes-langchain.chainLlm | First-pass AI accept/reject filter | Job Description, Cheap Model | Merge w/ Original Data | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| Cheap Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for GateKeeper |  | GateKeeper | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| Merge w/ Original Data | n8n-nodes-base.merge | Recombines GateKeeper output with job data | Job Description, GateKeeper | Keep Matches | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| Keep Matches | n8n-nodes-base.switch | Routes only GateKeeper-approved jobs | Merge w/ Original Data | Job Match Scorer, Merge w/ Original Data1 | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| Job Match Scorer | @n8n/n8n-nodes-langchain.chainLlm | Detailed AI scoring against full profile | Keep Matches, Better Model | Parse in JSON | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| Better Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter model for detailed scoring |  | Job Match Scorer | ## 2️⃣ Filter & Evaluate<br>Removes spam recruiters, filters by country, then runs 2-stage AI evaluation:<br>• **GateKeeper** (fast, cheap model): Quick yes/no filter<br>• **Job Match Scorer** (smarter model): Detailed scoring for matches |
| Parse in JSON | n8n-nodes-base.code | Parses scorer JSON from raw text | Job Match Scorer | Merge w/ Original Data1 | ## 3️⃣ Score & Notify<br>Parses AI response, merges with original job data, filters matches above threshold, adds to Google Sheet, and sends Telegram notification. |
| Merge w/ Original Data1 | n8n-nodes-base.merge | Recombines detailed score with job data | Keep Matches, Parse in JSON | Keep Matches1 | ## 3️⃣ Score & Notify<br>Parses AI response, merges with original job data, filters matches above threshold, adds to Google Sheet, and sends Telegram notification. |
| Keep Matches1 | n8n-nodes-base.switch | Keeps only jobs above score threshold | Merge w/ Original Data1 | Add Roles to apply | ## 3️⃣ Score & Notify<br>Parses AI response, merges with original job data, filters matches above threshold, adds to Google Sheet, and sends Telegram notification. |
| Add Roles to apply | n8n-nodes-base.googleSheets | Appends validated jobs to sheet | Keep Matches1 | Send a text message | ## 3️⃣ Score & Notify<br>Parses AI response, merges with original job data, filters matches above threshold, adds to Google Sheet, and sends Telegram notification. |
| Send a text message | n8n-nodes-base.telegram | Sends Telegram notification | Add Roles to apply |  | ## 3️⃣ Score & Notify<br>Parses AI response, merges with original job data, filters matches above threshold, adds to Google Sheet, and sends Telegram notification. |
| Google Sheets Trigger | n8n-nodes-base.googleSheetsTrigger | Spreadsheet-based entry point for document creation |  | Set Profile1 | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Set Profile1 | n8n-nodes-base.set | Stores master profile for document generation | Google Sheets Trigger | Get Job Description | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Get Job Description | n8n-nodes-base.googleSheets | Loads rows with Status=apply | Set Profile1 | Create Resume  | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Create Resume  | @n8n/n8n-nodes-langchain.openAi | Generates resume and cover letter JSON | Get Job Description | Select Language | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Select Language | n8n-nodes-base.switch | Routes to EN or DE template branch | Create Resume  | Make a Copy of EN Resume, Make a Copy of DE Resume | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Make a Copy of EN Resume | n8n-nodes-base.googleDrive | Copies English resume template | Select Language | Update all Resume Variables | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Update all Resume Variables | n8n-nodes-base.googleDocs | Replaces placeholders in English resume | Make a Copy of EN Resume | Make a Copy of EN CL | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Make a Copy of EN CL | n8n-nodes-base.googleDrive | Copies English cover letter template | Update all Resume Variables | Update all CL Variables | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Update all CL Variables | n8n-nodes-base.googleDocs | Replaces placeholders in English cover letter | Make a Copy of EN CL | Update Link & Status | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Update Link & Status | n8n-nodes-base.googleSheets | Writes English doc links and created status | Update all CL Variables |  | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Make a Copy of DE Resume | n8n-nodes-base.googleDrive | Copies German resume template | Select Language | Update all Lebenslauf Variables | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Update all Lebenslauf Variables | n8n-nodes-base.googleDocs | Replaces placeholders in German resume | Make a Copy of DE Resume | Make a Copy of DE Motivation | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Make a Copy of DE Motivation | n8n-nodes-base.googleDrive | Copies German cover-letter template | Update all Lebenslauf Variables | Update all Anschreiben Variables | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Update all Anschreiben Variables | n8n-nodes-base.googleDocs | Replaces placeholders in German cover letter | Make a Copy of DE Motivation | Update Link & Status for DE | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Update Link & Status for DE | n8n-nodes-base.googleSheets | Writes German doc links and created status | Update all Anschreiben Variables |  | ## 4️⃣ Generate Resume & Apply<br>Triggered from Google Sheets. Uses AI to generate tailored resume + cover letter, copies your template docs, fills in all variables, and updates status.<br>Supports EN and DE (add more languages by extending the Select Language switch). |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual summary |  |  |  |
| Sticky Note - Setup Guide | n8n-nodes-base.stickyNote | Visual setup documentation |  |  |  |
| Sticky Note - Filter | n8n-nodes-base.stickyNote | Visual block documentation |  |  |  |
| Sticky Note - Score | n8n-nodes-base.stickyNote | Visual block documentation |  |  |  |
| Sticky Note - Resume | n8n-nodes-base.stickyNote | Visual block documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

## A. Prepare external services first

1. **Create credentials in n8n**
   1. Apify credential via **HTTP Query Auth**
   2. OpenRouter credential
   3. OpenAI credential
   4. Google Sheets OAuth2 credential
   5. Google Sheets Trigger OAuth2 credential
   6. Google Drive OAuth2 credential
   7. Google Docs OAuth2 credential
   8. Telegram Bot credential

2. **Prepare Google Sheets**
   1. Create a spreadsheet.
   2. Create a sheet named something like `Validated`.
   3. Include at minimum these columns:
      - `id`
      - `title`
      - `companyName`
      - `location`
      - `link`
      - `applyUrl`
      - `job_description`
      - `text`
      - `match_score`
      - `reasoning`
      - `missing_critical_skills`
      - `matching_strong_skills`
      - `Status`
      - `Resume`
      - `Cover Letter`
   4. If you want to preserve more scraper fields, add the full column set used by the Google Sheets nodes.

3. **Prepare Google Docs templates**
   1. English resume template
   2. English cover-letter template
   3. German resume template
   4. German cover-letter template
   5. Add placeholders exactly matching node replacements, such as:
      - `{{summary}}`
      - `{{experience_1.title}}`
      - `{{experience_1.company}}`
      - `{{experience_1.date}}`
      - `{{experience_1.bullets}}`
      - `{{experience_2.title}}`
      - `{{experience_3.title}}`
      - `{{skills_tech}}`
      - `{{skills_industry}}`
      - `{{skills_soft}}`
      - `{{salutation}}`
      - `{{body_text}}`
      - `{{date}}`
      - `{{companyName}}`

4. **Prepare LinkedIn search URLs**
   - Build LinkedIn search URLs with your target keywords and `geoId`.
   - Example filters already present:
     - `f_TPR=r3600` recent jobs
     - `f_WT=1,3`
   - Replace all placeholders.

---

## B. Build the scheduled discovery branch

5. **Add Schedule Trigger**
   - Type: **Schedule Trigger**
   - Set cron to `0 0 7-23 * * *`.

6. **Add Set node named `Set Profile`**
   - Type: **Set**
   - Add one string field: `Profile`
   - Paste your complete master candidate profile.
   - Include all real experiences, skills, education, languages, and narrative.

7. **Add HTTP Request node named `Scrape Linkedin w/Apify`**
   - Method: `POST`
   - URL: `https://api.apify.com/v2/acts/curious_coder~linkedin-jobs-scraper/run-sync-get-dataset-items`
   - Body type: JSON
   - Send body: true
   - Body:
     - `count: 100`
     - `scrapeCompany: true`
     - `splitByLocation: false`
     - `urls`: array of your LinkedIn searches
   - Authentication: generic credential type
   - Generic auth type: **HTTP Query Auth**
   - Attach Apify credential.

8. **Connect**
   - `Schedule Trigger -> Set Profile`
   - `Schedule Trigger -> Scrape Linkedin w/Apify`

9. **Add Filter node `Remove Big Recruiters`**
   - Add multiple conditions:
     - `companyName != Recruiter Company 1`
     - etc.
   - Use exact company names you want excluded.

10. **Add Filter node `Keep only [Your Country]`**
    - Condition:
      - `location contains YOUR_COUNTRY`
    - Replace with your actual country string.

11. **Add Markdown node `Job Description`**
    - Input HTML: use `descriptionHtml` from the filtered item.
    - Destination key: `job_description`
    - Important: use a correct expression referencing the actual prior node, for example:
      - `{{ $('Keep only [Your Country]').item.json.descriptionHtml }}`
    - Do **not** leave the broken `Keep only CH` reference if your node is named differently.

12. **Connect**
    - `Scrape Linkedin w/Apify -> Remove Big Recruiters`
    - `Remove Big Recruiters -> Keep only [Your Country]`
    - `Keep only [Your Country] -> Job Description`

---

## C. Build first-stage AI screening

13. **Add OpenRouter model node `Cheap Model`**
   - Type: **OpenRouter Chat Model**
   - Attach OpenRouter credential.
   - Optionally set a fixed cheap model explicitly for reproducibility.

14. **Add LLM Chain node `GateKeeper`**
   - Type: **Basic LLM Chain**
   - Paste the strict recruiter prompt.
   - Input job description with `{{ $json.job_description }}`
   - Enable `Continue on Error` if you want behavior to match the template.
   - Connect the model input from `Cheap Model`.

15. **Add Merge node `Merge w/ Original Data`**
   - Mode: `combine`
   - Combine by: `position`

16. **Add Switch node `Keep Matches`**
   - Rule: text contains `"match": true`
   - Left value should reference GateKeeper raw text.

17. **Connect**
   - `Job Description -> GateKeeper`
   - `Cheap Model -> GateKeeper` via AI language model connection
   - `Job Description -> Merge w/ Original Data` input 0
   - `GateKeeper -> Merge w/ Original Data` input 1
   - `Merge w/ Original Data -> Keep Matches`

---

## D. Build second-stage scoring and notification

18. **Add OpenRouter model node `Better Model`**
   - Set model to `openai/gpt-4.1`
   - Attach OpenRouter credential.

19. **Add LLM Chain node `Job Match Scorer`**
   - Paste the detailed scoring prompt.
   - Reference full profile:
     - `{{ $('Set Profile').first().json.Profile }}`
   - Reference job description:
     - `{{ $('Job Description').item.json.job_description }}`
   - Require JSON-only output.

20. **Add Code node `Parse in JSON`**
   - Paste the provided JavaScript that:
     - finds first `{`
     - finds last `}`
     - slices
     - `JSON.parse`
     - merges parsed fields into output JSON

21. **Add Merge node `Merge w/ Original Data1`**
   - Mode: `combine`
   - Combine by: `position`

22. **Add Switch node `Keep Matches1`**
   - Condition:
     - `match_score >= 80`

23. **Add Google Sheets node `Add Roles to apply`**
   - Operation: `append`
   - Sheet: your validated sheet
   - Mapping mode: auto-map input data
   - Attach Google Sheets OAuth2 credential

24. **Add Telegram node `Send a text message`**
   - Set your `chatId`
   - Set message text with your spreadsheet URL
   - Disable attribution if desired

25. **Connect**
   - `Keep Matches -> Job Match Scorer`
   - `Better Model -> Job Match Scorer` via AI model connection
   - `Job Match Scorer -> Parse in JSON`
   - `Keep Matches -> Merge w/ Original Data1` input 0
   - `Parse in JSON -> Merge w/ Original Data1` input 1
   - `Merge w/ Original Data1 -> Keep Matches1`
   - `Keep Matches1 -> Add Roles to apply`
   - `Add Roles to apply -> Send a text message`

---

## E. Build the spreadsheet-triggered application branch

26. **Add Google Sheets Trigger**
   - Event: `rowUpdate`
   - Polling: every hour at minute 10
   - Select the same spreadsheet and sheet
   - Attach Google Sheets Trigger credential

27. **Add Set node `Set Profile1`**
   - Duplicate the same profile string from `Set Profile`

28. **Add Google Sheets node `Get Job Description`**
   - Operation: read/get rows
   - Add filter:
     - `Status = apply`
   - Use same spreadsheet and sheet as above

29. **Connect**
   - `Google Sheets Trigger -> Set Profile1`
   - `Set Profile1 -> Get Job Description`

---

## F. Build resume generation

30. **Add OpenAI node `Create Resume `**
   - Model: `gpt-5.1`
   - Response/text format: JSON object
   - Paste the full resume-and-cover-letter prompt
   - Reference:
     - `{{ $('Set Profile1').first().json.Profile }}`
     - `{{ $json.job_description }}`
   - Attach OpenAI credential

31. **Add Switch node `Select Language`**
   - Rule 1:
     - output key `EN`
     - language equals `EN`
   - Rule 2:
     - output key `DE`
     - language equals `DE`
   - Expression path:
     - `{{ $json.output[0].content[0].text.meta.language }}`

32. **Connect**
   - `Get Job Description -> Create Resume `
   - `Create Resume  -> Select Language`

---

## G. Build English template branch

33. **Add Google Drive node `Make a Copy of EN Resume`**
   - Operation: `copy`
   - Source file ID: English resume template
   - New name:
     - `Resume_{{ $('Get Job Description').item.json.companyName }}_{{ $('Get Job Description').item.json.row_number }}`

34. **Add Google Docs node `Update all Resume Variables`**
   - Operation: `update`
   - Target document: copied EN resume
   - Add replace-all actions for:
     - `{{summary}}`
     - `{{experience_1.title}}`
     - `{{experience_1.company}}`
     - `{{experience_1.date}}`
     - `{{experience_1.bullets}}`
     - same for experience 2 and 3
     - `{{skills_tech}}`
     - `{{skills_industry}}`
     - `{{skills_soft}}`

35. **Add Google Drive node `Make a Copy of EN CL`**
   - Operation: `copy`
   - Source file ID: English cover-letter template
   - New name:
     - `CoverLetter_{{ $('Get Job Description').item.json.companyName }}_{{ $('Get Job Description').item.json.row_number }}`

36. **Add Google Docs node `Update all CL Variables`**
   - Replace:
     - `{{salutation}}`
     - `{{body_text}}`
     - `{{date}}`
     - `{{companyName}}`

37. **Add Google Sheets node `Update Link & Status`**
   - Operation: `update`
   - Match on column `id`
   - Set:
     - `id`
     - `Resume = https://docs.google.com/document/d/<resumeDocId>`
     - `Cover Letter = https://docs.google.com/document/d/<coverLetterDocId>`
     - `Status = created`

38. **Connect**
   - `Select Language (EN) -> Make a Copy of EN Resume`
   - `Make a Copy of EN Resume -> Update all Resume Variables`
   - `Update all Resume Variables -> Make a Copy of EN CL`
   - `Make a Copy of EN CL -> Update all CL Variables`
   - `Update all CL Variables -> Update Link & Status`

---

## H. Build German template branch

39. **Add Google Drive node `Make a Copy of DE Resume`**
   - Copy German resume template
   - Name:
     - `Lebenslauf_{{ companyName }}_{{ row_number }}`

40. **Add Google Docs node `Update all Lebenslauf Variables`**
   - Same field replacements as English resume branch.

41. **Add Google Drive node `Make a Copy of DE Motivation`**
   - Copy German cover-letter template
   - Name:
     - `Anschreiben_{{ companyName }}_{{ row_number }}`

42. **Add Google Docs node `Update all Anschreiben Variables`**
   - Replace:
     - `{{salutation}}`
     - `{{body_text}}`
     - `{{date}}`
     - `{{companyName}}`

43. **Add Google Sheets node `Update Link & Status for DE`**
   - Update matching row by `id`
   - Set:
     - `Resume`
     - `Cover Letter`
     - `Status = created`

44. **Connect**
   - `Select Language (DE) -> Make a Copy of DE Resume`
   - `Make a Copy of DE Resume -> Update all Lebenslauf Variables`
   - `Update all Lebenslauf Variables -> Make a Copy of DE Motivation`
   - `Make a Copy of DE Motivation -> Update all Anschreiben Variables`
   - `Update all Anschreiben Variables -> Update Link & Status for DE`

---

## I. Important fixes before production use

45. **Fix the broken node reference**
   - In `Job Description`, replace `$('Keep only CH')` with the actual node name.

46. **Customize all placeholders**
   - `YOUR_GEO_ID`
   - `YOUR_KEYWORD_1`
   - `YOUR_KEYWORD_2`
   - `YOUR_COUNTRY`
   - spreadsheet IDs
   - document/template file IDs
   - Telegram chat ID
   - recruiter blacklist names
   - GateKeeper prompt profile lines and reject rules

47. **Consider hardening the workflow**
   - Parse GateKeeper JSON before filtering rather than string-matching.
   - Add deduplication before Google Sheets append.
   - Replace positional merges with ID-based merges if possible.
   - Add explicit model names to both OpenRouter nodes.
   - Keep `Set Profile` and `Set Profile1` synchronized.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow contains two independent triggers: one scheduled and one spreadsheet-driven. | Architecture / operational design |
| The `Job Description` node likely contains a stale expression referencing `Keep only CH` instead of `Keep only [Your Country]`. | Critical implementation note |
| The sheet named `Validated` acts as both storage for shortlisted jobs and the control surface for application generation via `Status = apply`. | Google Sheets design |
| The resume generation supports EN and DE only by default. Additional languages require extending the `Select Language` switch and duplicating template branches. | Language support |
| Several sticky notes mention “Excel,” but the actual implementation uses Google Sheets. | Documentation consistency note |
| OpenRouter is used for screening/scoring, while OpenAI is used for final application document generation. | AI provider split |
| Template placeholders in Google Docs must exactly match the replacement tokens configured in the Google Docs nodes. | Template dependency |