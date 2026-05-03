Screen job applications from Google Forms with Google Gemini and email results

https://n8nworkflows.xyz/workflows/screen-job-applications-from-google-forms-with-google-gemini-and-email-results-14138


# Screen job applications from Google Forms with Google Gemini and email results

# 1. Workflow Overview

This workflow automates first-pass screening of job applications collected in Google Sheets from a Google Form. It detects newly added rows, skips applicants already processed, enriches each record with a role-specific job description, asks Google Gemini to score the candidate, writes the assessment back into the sheet, and sends email notifications based on the score.

Typical use cases:
- Automating early HR triage for inbound applications
- Standardizing candidate screening using AI
- Updating a tracking sheet with screening results
- Routing shortlisted vs. rejected candidates into different communication paths

## 1.1 Trigger & Intake

The workflow starts with a Google Sheets Trigger configured to watch for newly added rows every minute. It then re-reads the full sheet and filters out rows already marked as processed.

## 1.2 Candidate Data Preparation & AI Scoring

For each unprocessed row, the workflow extracts normalized applicant fields, maps the selected role to a built-in job description, and sends a prompt to Google Gemini via an AI Agent node. Gemini is expected to return raw JSON containing the score and qualitative assessment.

## 1.3 AI Output Parsing & Result Construction

The raw AI output is parsed into structured fields. If parsing fails, the workflow falls back to a basic default score/grade and still computes a final applicant status based on score threshold.

## 1.4 Google Sheets Update

The workflow builds a direct Google Sheets API request targeting columns J–R of the exact row and writes all assessment fields back in one PUT request, including a processed flag.

## 1.5 Decision Routing & Email Notifications

After the sheet is updated, an IF node checks whether the score is at least 7. Shortlisted candidates trigger an HR alert plus a candidate invitation email after a short delay; rejected candidates receive a decline email.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Filter

### Overview
This block detects new application rows and prevents duplicate processing. Although the trigger fires on row creation, the workflow re-fetches all rows and applies its own processed-flag logic for reliability and state awareness.

### Nodes Involved
- Google Sheets Trigger
- Read All Rows
- Filter Unprocessed Rows

### Node Details

#### Google Sheets Trigger
- **Type and technical role:** `n8n-nodes-base.googleSheetsTrigger` — polling trigger for new rows in a Google Sheet.
- **Configuration choices:**
  - Event: `rowAdded`
  - Polling frequency: every minute
  - Document ID set to a specific Google Sheet
  - Sheet selected by internal sheet identifier/list value
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:**
  - Entry point of the workflow
  - Outputs to `Read All Rows`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Google OAuth2 credential missing or expired
  - Wrong document or sheet selected
  - Polling delay means applications are not processed instantly
  - Trigger may detect the new row, but downstream logic still depends on reading the full sheet correctly
- **Sub-workflow reference:** None

#### Read All Rows
- **Type and technical role:** `n8n-nodes-base.googleSheets` — reads the current state of the target sheet.
- **Configuration choices:**
  - Same spreadsheet and worksheet as the trigger
  - Default read operation behavior, with no additional options enabled
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Google Sheets Trigger`
  - Output to `Filter Unprocessed Rows`
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**
  - Auth failure on Google Sheets OAuth2
  - Reading the full sheet may become slow on large datasets
  - If column headers change, downstream field extraction may break
  - If the row numbering metadata is absent or altered, later update logic may fail
- **Sub-workflow reference:** None

#### Filter Unprocessed Rows
- **Type and technical role:** `n8n-nodes-base.code` — per-item JavaScript filter and normalization step.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Trims keys and values into a normalized object `r`
  - Requires `Email Address` to be present
  - Checks processed state from either:
    - `col_18` (column R)
    - `Processed` named header
  - Returns `null` to skip invalid/already processed rows
- **Key expressions or variables used:**
  - `row['col_18']`
  - `r['Processed']`
  - `r['Email Address']`
- **Input and output connections:**
  - Input from `Read All Rows`
  - Output to `Extract Fields & Load JD`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Rows without email are silently skipped
  - If processed flag is not exactly `'1'`, the row is treated as unprocessed
  - If sheet columns are reordered and header-based names differ, logic may miss expected fields
  - Trimming keys helps with accidental whitespace in headers, but not with renamed headers
- **Sub-workflow reference:** None

---

## 2.2 Candidate Data Preparation & AI Assessment

### Overview
This block extracts candidate details, maps the selected job role to a predefined job description, then asks Gemini to score the candidate. The AI interaction is structured so downstream nodes can parse a JSON response.

### Nodes Involved
- Extract Fields & Load JD
- AI Resume Scorer
- Google Gemini Chat Model

### Node Details

#### Extract Fields & Load JD
- **Type and technical role:** `n8n-nodes-base.code` — transforms sheet row data into a normalized candidate payload for AI scoring.
- **Configuration choices:**
  - Reads the first incoming item
  - Trims all row keys
  - Extracts:
    - position
    - name
    - email
    - experience
    - skills
    - `row_number`
  - Uses a hardcoded `JD_MAP` to convert role names into compact job descriptions
  - Falls back to `'General professional evaluation.'` when no role match exists
  - Returns the normalized row plus `jd`
- **Key expressions or variables used:**
  - `r['Select the Position You Are Applying For']`
  - `r['Full Name']`
  - `r['Email Address']`
  - `r['Years of Experience']`
  - `r['Relevant Skills']`
  - `row['row_number']`
  - `JD_MAP[position]`
- **Input and output connections:**
  - Input from `Filter Unprocessed Rows`
  - Output to `AI Resume Scorer`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If expected form column headers change, extracted values may become empty
  - `row_number` is essential for the sheet update later; if unavailable, updates may target an invalid range
  - Any role not in `JD_MAP` receives a generic evaluation prompt
- **Sub-workflow reference:** None

#### AI Resume Scorer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — AI Agent node that sends the prompt to the connected LLM and returns generated text.
- **Configuration choices:**
  - Prompt type: defined manually
  - User prompt includes `{{ $json.jd }}`
  - Requests JSON with:
    - `score` (1–10)
    - `grade`
    - `strengths`
    - `weaknesses`
    - `recommendation`
    - `summary`
  - System message explicitly instructs:
    - raw JSON only
    - single line
- **Key expressions or variables used:**
  - `{{ $json.jd }}`
- **Input and output connections:**
  - Main input from `Extract Fields & Load JD`
  - AI language model input from `Google Gemini Chat Model`
  - Main output to `Parse AI Output`
- **Version-specific requirements:** Type version `3.1`; requires LangChain-compatible AI nodes in the n8n instance.
- **Edge cases or potential failure types:**
  - LLM may still return invalid JSON despite instructions
  - Prompt currently includes the JD but does not explicitly inject all applicant attributes into the prompt text; depending on agent behavior, this may reduce scoring quality unless the node implicitly includes item context
  - Rate limiting or quota issues from Gemini
  - Output schema is not enforced beyond prompt instructions
- **Sub-workflow reference:** None

#### Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — chat model provider node supplying Gemini to the AI Agent.
- **Configuration choices:**
  - Default options only
  - Requires Google Gemini API credential/API key
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected via `ai_languageModel` to `AI Resume Scorer`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Missing/invalid Gemini credential
  - Quota exhaustion, safety filtering, transient provider errors
  - Model defaults may vary across n8n versions or provider configuration
- **Sub-workflow reference:** None

---

## 2.3 AI Output Parsing & Result Construction

### Overview
This block converts the AI response into structured fields usable by the rest of the workflow. It also derives a business status and records the processing timestamp.

### Nodes Involved
- Parse AI Output

### Node Details

#### Parse AI Output
- **Type and technical role:** `n8n-nodes-base.code` — parses AI response text and merges it with candidate data.
- **Configuration choices:**
  - Reads raw AI output from `output`
  - Retrieves original candidate data from `Extract Fields & Load JD` using node reference
  - Default fallback object:
    - `score: 5`
    - `grade: 'Maybe'`
  - Attempts `JSON.parse(aiRaw)` inside `try/catch`
  - Computes:
    - `status = 'Shortlisted'` if score ≥ 7
    - otherwise `status = 'Rejected'`
  - Adds `processed_at` using current ISO timestamp
- **Key expressions or variables used:**
  - `$input.first().json.output`
  - `$('Extract Fields & Load JD').first().json`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input from `AI Resume Scorer`
  - Output to `Build Sheet Request`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Invalid JSON from Gemini triggers fallback values silently
  - If AI returns score as text instead of number, later numeric comparison may behave unexpectedly
  - If referenced node name changes, `$('Extract Fields & Load JD')` will break
  - Missing `output` field from the AI node leads to fallback behavior
- **Sub-workflow reference:** None

---

## 2.4 Sheet Update

### Overview
This block prepares and executes a direct Google Sheets API update for the applicant row. It writes the score, qualitative feedback, status, processing timestamp, and processed marker in a single API request.

### Nodes Involved
- Build Sheet Request
- Update Sheet Row

### Node Details

#### Build Sheet Request
- **Type and technical role:** `n8n-nodes-base.code` — constructs the request URL and JSON body for the Google Sheets Values API.
- **Configuration choices:**
  - Uses `row_number` to build range `Form responses 1!J{row}:R{row}`
  - Builds nine output values in order:
    1. score
    2. grade
    3. strengths
    4. weaknesses
    5. recommendation
    6. summary
    7. status
    8. processed_at
    9. `'1'` processed flag
  - Stores the request body in `_requestBody`
  - Stores the API endpoint in `_url`
- **Key expressions or variables used:**
  - `d.row_number`
  - `encodeURIComponent(range)`
  - hardcoded spreadsheet ID
  - hardcoded sheet tab name `Form responses 1`
- **Input and output connections:**
  - Input from `Parse AI Output`
  - Output to `Update Sheet Row`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Wrong tab name causes API failure
  - If `row_number` is `0` or missing, update range becomes invalid
  - Hardcoded spreadsheet ID must match all other Google Sheets nodes
  - Long AI text values may still write successfully, but spreadsheet formatting/usability may degrade
- **Sub-workflow reference:** None

#### Update Sheet Row
- **Type and technical role:** `n8n-nodes-base.httpRequest` — performs the actual Google Sheets API `PUT` call.
- **Configuration choices:**
  - Method: `PUT`
  - URL from `={{ $json._url }}`
  - Sends JSON body from `={{ JSON.stringify($json._requestBody) }}`
  - Body mode: JSON
  - Authentication: predefined credential type
  - Credential type: `googleSheetsOAuth2Api`
- **Key expressions or variables used:**
  - `{{ $json._url }}`
  - `{{ JSON.stringify($json._requestBody) }}`
- **Input and output connections:**
  - Input from `Build Sheet Request`
  - Output to `Check Score ≥ 7`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**
  - OAuth scopes may be insufficient for direct Sheets API updates
  - Invalid URL/body format produces HTTP 4xx errors
  - Network timeouts or Google API quota errors
  - If the response body differs from expectations, downstream still only uses incoming item context, not response content
- **Sub-workflow reference:** None

---

## 2.5 Decision Routing & Email Notifications

### Overview
This block routes applicants into shortlist or rejection communication paths based on score threshold. Shortlisted applicants trigger both an HR alert and a delayed candidate notification, while rejected applicants receive a decline email.

### Nodes Involved
- Check Score ≥ 7
- Alert HR Team
- Wait 5s
- Notify Shortlisted Candidate
- Notify Rejected Candidate

### Node Details

#### Check Score ≥ 7
- **Type and technical role:** `n8n-nodes-base.if` — branching logic on numeric score.
- **Configuration choices:**
  - Condition type: number
  - Operation: greater than or equal
  - Left value: `={{ $json.score }}`
  - Right value: `7`
  - Strict type validation enabled
- **Key expressions or variables used:**
  - `{{ $json.score }}`
- **Input and output connections:**
  - Input from `Update Sheet Row`
  - True output to:
    - `Alert HR Team`
    - `Wait 5s`
  - False output to:
    - `Notify Rejected Candidate`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - If `score` is a string instead of number, strict validation may affect comparison behavior
  - If score is missing, item may fail condition evaluation or route unexpectedly
- **Sub-workflow reference:** None

#### Alert HR Team
- **Type and technical role:** `n8n-nodes-base.emailSend` — sends an internal HR notification email for shortlisted candidates.
- **Configuration choices:**
  - Static `toEmail`: `user@example.com`
  - Static `fromEmail`: `user@example.com`
  - Subject: `Shortlist Alert: {name} ({score}/10)`
  - HTML body includes score and strengths
- **Key expressions or variables used:**
  - `{{ $json.name }}`
  - `{{ $json.score }}`
  - `{{ $json.strengths }}`
- **Input and output connections:**
  - Input from true branch of `Check Score ≥ 7`
  - No downstream node
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Email transport not configured in n8n
  - Static HR mailbox must be replaced
  - Missing `name` field may produce blank subject text
  - HTML content is minimal and may not meet company formatting standards
- **Sub-workflow reference:** None

#### Wait 5s
- **Type and technical role:** `n8n-nodes-base.wait` — intentional delay before sending the shortlisted candidate email.
- **Configuration choices:**
  - Default wait behavior; despite the node name, no explicit duration is visible in the JSON parameters
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from true branch of `Check Score ≥ 7`
  - Output to `Notify Shortlisted Candidate`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**
  - Important discrepancy: the node is named “Wait 5s” but no wait time is configured in the exported parameters; actual behavior should be verified in the n8n editor
  - Wait nodes may require resumable execution support depending on deployment mode
- **Sub-workflow reference:** None

#### Notify Shortlisted Candidate
- **Type and technical role:** `n8n-nodes-base.emailSend` — sends a positive candidate notification.
- **Configuration choices:**
  - `toEmail`: dynamic from applicant email
  - `fromEmail`: static sender address
  - Subject: `Interview Invitation - {name}`
  - HTML body congratulates the candidate and includes the score
- **Key expressions or variables used:**
  - `{{ $json.email }}`
  - `{{ $json.name }}`
  - `{{ $json.score }}`
- **Input and output connections:**
  - Input from `Wait 5s`
  - No downstream node
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Applicant email may be invalid or missing despite earlier filtering
  - Sender mailbox must be replaced and supported by the n8n email transport
  - Current content is simplistic and may need localization or branding
- **Sub-workflow reference:** None

#### Notify Rejected Candidate
- **Type and technical role:** `n8n-nodes-base.emailSend` — sends a rejection email for non-shortlisted applicants.
- **Configuration choices:**
  - `toEmail`: dynamic from applicant email
  - `fromEmail`: static sender address
  - Subject: `Application Update`
  - HTML body contains a concise decline message
- **Key expressions or variables used:**
  - `{{ $json.email }}`
- **Input and output connections:**
  - Input from false branch of `Check Score ≥ 7`
  - No downstream node
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Same email transport/sender concerns as other email nodes
  - Message is generic and contains no opt-in for future opportunities
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Canvas documentation note |  |  | ## 🤖 AI Resume Screener & Automated HR Notifier Automatically scores job applicants using Google Gemini and routes personalised emails to HR and candidates. How it works: Polls a Google Sheet every minute → skips already-processed rows → maps the applied role to a Job Description → asks Gemini to score the candidate 1–10 → writes the analysis back to the sheet → sends an HR alert and candidate email based on the score. Setup checklist: Google Sheets OAuth2 credential → connect to Trigger and HTTP Request nodes; Google Gemini API key → connect to the Gemini Chat Model node; Replace Sheet ID `1iv-8fToAzxBm8ht5vg-UP_p_hQzK4-....` in all Google Sheet nodes; Update `fromEmail` / `toEmail` in the three Send Email nodes; Extend `JD_MAP` in the Code node with your own open roles |
| Step 1 — Trigger & Filter | Sticky Note | Canvas documentation note |  |  | ### 📥 Step 1 — Trigger & Filter Google Sheets Trigger polls every minute and fires when a new row is added. Read All Rows re-fetches the full sheet so the filter has the latest state of every row. Filter Unprocessed Rows skips any row where column R (`col_18`) is already `'1'`, preventing duplicate processing. |
| Step 2 — AI Assessment | Sticky Note | Canvas documentation note |  |  | ### 🧠 Step 2 — AI Assessment Extract Fields & Load JD pulls the candidate's name, email, position, experience, and skills. It then looks up the matching Job Description from the built-in `JD_MAP`. AI Resume Scorer sends a structured prompt to Gemini requesting a raw JSON response containing: `score` (1–10), `grade`, `strengths`, `weaknesses`, `recommendation`, and `summary`. Google Gemini Chat Model is the sub-node that powers the AI agent above. Parse AI Output safely parses Gemini's JSON string and adds a `status` field: `'Shortlisted'` if score ≥ 7, otherwise `'Rejected'`. |
| Step 3 — Sheet Update | Sticky Note | Canvas documentation note |  |  | ### 📊 Step 3 — Sheet Update Build Sheet Request constructs the Sheets API PUT URL and body, targeting columns J–R on the exact row number of the applicant. Update Sheet Row calls the Google Sheets API directly via HTTP Request (using OAuth2) to write all 9 fields in a single operation — including setting column R to `'1'` as the processed flag. |
| Step 4 — Routing & Emails | Sticky Note | Canvas documentation note |  |  | ### 📧 Step 4 — Score Routing & Emails Check Score ≥ 7 branches the flow: ✅ True (shortlist path) - Alert HR Team — notifies the HR inbox with the candidate's score and strengths. - Wait 5s — small delay before the candidate email goes out. - Notify Shortlisted Candidate — sends the applicant a congratulations and interview invitation. ❌ False (rejection path) - Notify Rejected Candidate — sends a polite, professional decline email. |
| Google Sheets Trigger | n8n-nodes-base.googleSheetsTrigger | Poll for newly added rows |  | Read All Rows | ### 📥 Step 1 — Trigger & Filter Google Sheets Trigger polls every minute and fires when a new row is added. Read All Rows re-fetches the full sheet so the filter has the latest state of every row. Filter Unprocessed Rows skips any row where column R (`col_18`) is already `'1'`, preventing duplicate processing. |
| Read All Rows | n8n-nodes-base.googleSheets | Read all rows from the applications sheet | Google Sheets Trigger | Filter Unprocessed Rows | ### 📥 Step 1 — Trigger & Filter Google Sheets Trigger polls every minute and fires when a new row is added. Read All Rows re-fetches the full sheet so the filter has the latest state of every row. Filter Unprocessed Rows skips any row where column R (`col_18`) is already `'1'`, preventing duplicate processing. |
| Filter Unprocessed Rows | n8n-nodes-base.code | Skip processed rows and rows without email | Read All Rows | Extract Fields & Load JD | ### 📥 Step 1 — Trigger & Filter Google Sheets Trigger polls every minute and fires when a new row is added. Read All Rows re-fetches the full sheet so the filter has the latest state of every row. Filter Unprocessed Rows skips any row where column R (`col_18`) is already `'1'`, preventing duplicate processing. |
| Extract Fields & Load JD | n8n-nodes-base.code | Normalize applicant fields and map role to job description | Filter Unprocessed Rows | AI Resume Scorer | ### 🧠 Step 2 — AI Assessment Extract Fields & Load JD pulls the candidate's name, email, position, experience, and skills. It then looks up the matching Job Description from the built-in `JD_MAP`. AI Resume Scorer sends a structured prompt to Gemini requesting a raw JSON response containing: `score` (1–10), `grade`, `strengths`, `weaknesses`, `recommendation`, and `summary`. Google Gemini Chat Model is the sub-node that powers the AI agent above. Parse AI Output safely parses Gemini's JSON string and adds a `status` field: `'Shortlisted'` if score ≥ 7, otherwise `'Rejected'`. |
| AI Resume Scorer | @n8n/n8n-nodes-langchain.agent | Ask Gemini to score the candidate and return JSON | Extract Fields & Load JD; Google Gemini Chat Model | Parse AI Output | ### 🧠 Step 2 — AI Assessment Extract Fields & Load JD pulls the candidate's name, email, position, experience, and skills. It then looks up the matching Job Description from the built-in `JD_MAP`. AI Resume Scorer sends a structured prompt to Gemini requesting a raw JSON response containing: `score` (1–10), `grade`, `strengths`, `weaknesses`, `recommendation`, and `summary`. Google Gemini Chat Model is the sub-node that powers the AI agent above. Parse AI Output safely parses Gemini's JSON string and adds a `status` field: `'Shortlisted'` if score ≥ 7, otherwise `'Rejected'`. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provide Gemini LLM to the AI Agent |  | AI Resume Scorer | ### 🧠 Step 2 — AI Assessment Extract Fields & Load JD pulls the candidate's name, email, position, experience, and skills. It then looks up the matching Job Description from the built-in `JD_MAP`. AI Resume Scorer sends a structured prompt to Gemini requesting a raw JSON response containing: `score` (1–10), `grade`, `strengths`, `weaknesses`, `recommendation`, and `summary`. Google Gemini Chat Model is the sub-node that powers the AI agent above. Parse AI Output safely parses Gemini's JSON string and adds a `status` field: `'Shortlisted'` if score ≥ 7, otherwise `'Rejected'`. |
| Parse AI Output | n8n-nodes-base.code | Parse Gemini JSON and derive shortlist/reject status | AI Resume Scorer | Build Sheet Request | ### 🧠 Step 2 — AI Assessment Extract Fields & Load JD pulls the candidate's name, email, position, experience, and skills. It then looks up the matching Job Description from the built-in `JD_MAP`. AI Resume Scorer sends a structured prompt to Gemini requesting a raw JSON response containing: `score` (1–10), `grade`, `strengths`, `weaknesses`, `recommendation`, and `summary`. Google Gemini Chat Model is the sub-node that powers the AI agent above. Parse AI Output safely parses Gemini's JSON string and adds a `status` field: `'Shortlisted'` if score ≥ 7, otherwise `'Rejected'`. |
| Build Sheet Request | n8n-nodes-base.code | Build Google Sheets API URL and payload for row update | Parse AI Output | Update Sheet Row | ### 📊 Step 3 — Sheet Update Build Sheet Request constructs the Sheets API PUT URL and body, targeting columns J–R on the exact row number of the applicant. Update Sheet Row calls the Google Sheets API directly via HTTP Request (using OAuth2) to write all 9 fields in a single operation — including setting column R to `'1'` as the processed flag. |
| Update Sheet Row | n8n-nodes-base.httpRequest | Write assessment fields back to the sheet via Sheets API | Build Sheet Request | Check Score ≥ 7 | ### 📊 Step 3 — Sheet Update Build Sheet Request constructs the Sheets API PUT URL and body, targeting columns J–R on the exact row number of the applicant. Update Sheet Row calls the Google Sheets API directly via HTTP Request (using OAuth2) to write all 9 fields in a single operation — including setting column R to `'1'` as the processed flag. |
| Check Score ≥ 7 | n8n-nodes-base.if | Branch shortlisted vs rejected path | Update Sheet Row | Alert HR Team; Wait 5s; Notify Rejected Candidate | ### 📧 Step 4 — Score Routing & Emails Check Score ≥ 7 branches the flow: ✅ True (shortlist path) - Alert HR Team — notifies the HR inbox with the candidate's score and strengths. - Wait 5s — small delay before the candidate email goes out. - Notify Shortlisted Candidate — sends the applicant a congratulations and interview invitation. ❌ False (rejection path) - Notify Rejected Candidate — sends a polite, professional decline email. |
| Alert HR Team | n8n-nodes-base.emailSend | Send shortlist notification to HR | Check Score ≥ 7 |  | ### 📧 Step 4 — Score Routing & Emails Check Score ≥ 7 branches the flow: ✅ True (shortlist path) - Alert HR Team — notifies the HR inbox with the candidate's score and strengths. - Wait 5s — small delay before the candidate email goes out. - Notify Shortlisted Candidate — sends the applicant a congratulations and interview invitation. ❌ False (rejection path) - Notify Rejected Candidate — sends a polite, professional decline email. |
| Wait 5s | n8n-nodes-base.wait | Delay before emailing shortlisted candidate | Check Score ≥ 7 | Notify Shortlisted Candidate | ### 📧 Step 4 — Score Routing & Emails Check Score ≥ 7 branches the flow: ✅ True (shortlist path) - Alert HR Team — notifies the HR inbox with the candidate's score and strengths. - Wait 5s — small delay before the candidate email goes out. - Notify Shortlisted Candidate — sends the applicant a congratulations and interview invitation. ❌ False (rejection path) - Notify Rejected Candidate — sends a polite, professional decline email. |
| Notify Shortlisted Candidate | n8n-nodes-base.emailSend | Send shortlist/interview invitation email to candidate | Wait 5s |  | ### 📧 Step 4 — Score Routing & Emails Check Score ≥ 7 branches the flow: ✅ True (shortlist path) - Alert HR Team — notifies the HR inbox with the candidate's score and strengths. - Wait 5s — small delay before the candidate email goes out. - Notify Shortlisted Candidate — sends the applicant a congratulations and interview invitation. ❌ False (rejection path) - Notify Rejected Candidate — sends a polite, professional decline email. |
| Notify Rejected Candidate | n8n-nodes-base.emailSend | Send rejection email to candidate | Check Score ≥ 7 |  | ### 📧 Step 4 — Score Routing & Emails Check Score ≥ 7 branches the flow: ✅ True (shortlist path) - Alert HR Team — notifies the HR inbox with the candidate's score and strengths. - Wait 5s — small delay before the candidate email goes out. - Notify Shortlisted Candidate — sends the applicant a congratulations and interview invitation. ❌ False (rejection path) - Notify Rejected Candidate — sends a polite, professional decline email. |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence in n8n.

## Prerequisites
1. Create a Google Form that stores responses in a Google Sheet.
2. Ensure the destination sheet contains:
   - the original Google Form fields
   - enough empty columns from J to R for the workflow outputs
3. Prepare credentials:
   - **Google Sheets OAuth2** credential with access to the spreadsheet
   - **Google Gemini** API credential for the Gemini model node
   - **Email sending setup** supported by n8n for the Email Send nodes
4. Confirm your worksheet tab name. In this workflow it is hardcoded as:
   - `Form responses 1`
5. Note the spreadsheet ID and replace it consistently everywhere.

## Suggested sheet column usage
This workflow writes to columns J–R:
1. J = score
2. K = grade
3. L = strengths
4. M = weaknesses
5. N = recommendation
6. O = summary
7. P = status
8. Q = processed_at
9. R = processed flag (`1`)

## Build steps

1. **Create a new workflow** in n8n.

2. **Add a Google Sheets Trigger node** named `Google Sheets Trigger`.
   - Node type: `Google Sheets Trigger`
   - Event: `Row Added`
   - Polling: `Every Minute`
   - Select your Google Sheets OAuth2 credential
   - Set the target spreadsheet ID
   - Select the correct sheet/tab

3. **Add a Google Sheets node** named `Read All Rows`.
   - Node type: `Google Sheets`
   - Operation: read rows from the same spreadsheet/tab
   - Use the same Google Sheets OAuth2 credential
   - Point it to the same spreadsheet and worksheet
   - Leave advanced options at default unless your sheet structure requires otherwise

4. **Connect** `Google Sheets Trigger` → `Read All Rows`.

5. **Add a Code node** named `Filter Unprocessed Rows`.
   - Mode: `Run Once For Each Item`
   - Paste this logic conceptually:
     - trim keys and values
     - skip rows without `Email Address`
     - inspect column R either by `col_18` or by header `Processed`
     - if processed equals `'1'`, return nothing
     - otherwise pass the row through
   - Use this exact behavior if you want parity with the exported workflow.

6. **Connect** `Read All Rows` → `Filter Unprocessed Rows`.

7. **Add a Code node** named `Extract Fields & Load JD`.
   - Purpose: normalize applicant data and map role to JD
   - Extract these headers from the row:
     - `Select the Position You Are Applying For`
     - `Full Name`
     - `Email Address`
     - `Years of Experience`
     - `Relevant Skills`
   - Also capture `row_number`
   - Create a `JD_MAP` object with entries such as:
     - Frontend Developer
     - Backend Developer
     - UI/UX Designer
     - Project Manager
     - Digital Marketing Executive
   - For unmapped roles, use fallback text like:
     - `General professional evaluation.`
   - Return the normalized row plus `jd`

8. **Connect** `Filter Unprocessed Rows` → `Extract Fields & Load JD`.

9. **Add an AI Agent node** named `AI Resume Scorer`.
   - Node type: `AI Agent`
   - Prompt type: define manually
   - User prompt:
     - `You are an expert HR recruiter. Score this candidate based on JD: {{ $json.jd }}. Return JSON with score (1-10), grade, strengths, weaknesses, recommendation, and summary.`
   - System message:
     - `HR recruiter AI. Return raw JSON only. Single line.`
   - Keep output in its standard text field

10. **Add a Google Gemini Chat Model node** named `Google Gemini Chat Model`.
    - Node type: `Google Gemini Chat Model`
    - Attach your Gemini credential/API key
    - Default model settings are acceptable for parity unless you want to explicitly select model/version

11. **Connect the model to the agent** using the AI language model connection:
    - `Google Gemini Chat Model` → `AI Resume Scorer`

12. **Connect** `Extract Fields & Load JD` → `AI Resume Scorer`.

13. **Add a Code node** named `Parse AI Output`.
   - Read the AI output text from the agent output field
   - Reference the earlier normalized data from `Extract Fields & Load JD`
   - Try to parse the AI text as JSON
   - On parse failure, use fallback:
     - `score = 5`
     - `grade = "Maybe"`
   - Create:
     - `status = "Shortlisted"` if score >= 7
     - else `status = "Rejected"`
   - Add:
     - `processed_at = new Date().toISOString()`
   - Return merged candidate + AI data

14. **Connect** `AI Resume Scorer` → `Parse AI Output`.

15. **Add a Code node** named `Build Sheet Request`.
   - Build the row range as:
     - `Form responses 1!J{row_number}:R{row_number}`
   - Build the values array in this exact order:
     1. score
     2. grade
     3. strengths
     4. weaknesses
     5. recommendation
     6. summary
     7. status
     8. processed_at
     9. `'1'`
   - Store:
     - `_requestBody = { range, values: [values] }`
     - `_url = https://sheets.googleapis.com/v4/spreadsheets/YOUR_SPREADSHEET_ID/values/{encodedRange}?valueInputOption=RAW`

16. **Connect** `Parse AI Output` → `Build Sheet Request`.

17. **Add an HTTP Request node** named `Update Sheet Row`.
   - Method: `PUT`
   - URL: expression using `{{$json._url}}`
   - Authentication: predefined credential type
   - Credential type: `Google Sheets OAuth2 API`
   - Send body: enabled
   - Body type: JSON
   - JSON body expression:
     - `{{ JSON.stringify($json._requestBody) }}`
   - Ensure the Google credential has Sheets write access

18. **Connect** `Build Sheet Request` → `Update Sheet Row`.

19. **Add an IF node** named `Check Score ≥ 7`.
   - Set a numeric condition:
     - Left value: `{{ $json.score }}`
     - Operator: greater than or equal
     - Right value: `7`
   - Keep strict type validation in mind; score should ideally be numeric

20. **Connect** `Update Sheet Row` → `Check Score ≥ 7`.

21. **Add an Email Send node** named `Alert HR Team`.
   - Configure your email transport/credential as required by your n8n setup
   - To: HR inbox, e.g. `user@example.com`
   - From: approved sender address
   - Subject:
     - `Shortlist Alert: {{ $json.name }} ({{ $json.score }}/10)`
   - HTML body:
     - `Candidate scored {{ $json.score }}. Strengths: {{ $json.strengths }}`

22. **Connect the true branch** of `Check Score ≥ 7` → `Alert HR Team`.

23. **Add a Wait node** named `Wait 5s`.
   - Configure an actual wait duration of 5 seconds manually in the editor
   - Important: the exported JSON does not visibly contain the delay setting, so verify this explicitly

24. **Connect the true branch** of `Check Score ≥ 7` → `Wait 5s`.

25. **Add an Email Send node** named `Notify Shortlisted Candidate`.
   - To: `{{ $json.email }}`
   - From: approved sender address
   - Subject:
     - `Interview Invitation - {{ $json.name }}`
   - HTML body:
     - `Congrats! You were shortlisted with a score of {{ $json.score }}.`

26. **Connect** `Wait 5s` → `Notify Shortlisted Candidate`.

27. **Add an Email Send node** named `Notify Rejected Candidate`.
   - To: `{{ $json.email }}`
   - From: approved sender address
   - Subject:
     - `Application Update`
   - HTML body:
     - `Thank you for applying. We won't be moving forward at this time.`

28. **Connect the false branch** of `Check Score ≥ 7` → `Notify Rejected Candidate`.

29. **Optional but recommended:** add the same sticky notes used in the exported workflow to document each block for maintainability.

30. **Replace all placeholders and hardcoded values:**
   - Spreadsheet ID in:
     - Google Sheets Trigger
     - Read All Rows
     - Build Sheet Request URL
   - Sheet tab name in `Build Sheet Request`
   - HR email and sender email in all Email Send nodes
   - `JD_MAP` contents for your actual open roles

31. **Test with a sample row** in the sheet.
   - Confirm the row contains the expected form headers
   - Confirm `row_number` is present in the Google Sheets read output
   - Confirm columns J–R are updated
   - Confirm only rows without processed flag are handled

32. **Activate the workflow** once validation is complete.

## Credential setup notes

### Google Sheets OAuth2
- Used by:
  - `Google Sheets Trigger`
  - `Read All Rows`
  - `Update Sheet Row`
- Required capabilities:
  - read spreadsheet rows
  - update spreadsheet values through the Sheets API

### Google Gemini
- Used by:
  - `Google Gemini Chat Model`
- Must be linked to the Gemini model node that feeds the AI Agent

### Email delivery
- Used by:
  - `Alert HR Team`
  - `Notify Shortlisted Candidate`
  - `Notify Rejected Candidate`
- The exported workflow shows sender/recipient fields but not a dedicated SMTP credential object; configure according to your n8n deployment’s supported email sending method.

## Input/output expectations

### Input
A new Google Form submission stored as a row in the target sheet, with headers including:
- `Full Name`
- `Email Address`
- `Select the Position You Are Applying For`
- `Years of Experience`
- `Relevant Skills`

### Intermediate AI output
Gemini is expected to return a one-line JSON string like:
- `score`
- `grade`
- `strengths`
- `weaknesses`
- `recommendation`
- `summary`

### Final outputs
- Updated Google Sheet row, columns J–R
- HR alert email if shortlisted
- Candidate shortlist email if shortlisted
- Candidate rejection email if not shortlisted

## Important implementation caveats
- The AI prompt only explicitly includes `jd`; if you want stronger screening quality, include candidate experience and skills directly in the prompt.
- The workflow relies on header names exactly matching the current Google Form output.
- The hardcoded tab name `Form responses 1` must match your sheet.
- The processed flag logic expects `'1'` in column R or a `Processed` column.
- The `Wait 5s` node name may not reflect actual configuration unless you set it manually.
- Since the workflow reads all rows after each new row event, large sheets may affect performance.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| 🤖 AI Resume Screener & Automated HR Notifier: Automatically scores job applicants using Google Gemini and routes personalised emails to HR and candidates. | Workflow overview note |
| How it works: Polls a Google Sheet every minute → skips already-processed rows → maps the applied role to a Job Description → asks Gemini to score the candidate 1–10 → writes the analysis back to the sheet → sends an HR alert and candidate email based on the score. | Workflow overview note |
| Setup checklist: Google Sheets OAuth2 credential → connect to Trigger and HTTP Request nodes. | Workflow setup note |
| Setup checklist: Google Gemini API key → connect to the Gemini Chat Model node. | Workflow setup note |
| Setup checklist: Replace Sheet ID `1iv-8fToAzxBm8ht5vg-UP_p_hQzK4-....` in all Google Sheet nodes. | Workflow setup note |
| Setup checklist: Update `fromEmail` / `toEmail` in the three Send Email nodes. | Workflow setup note |
| Setup checklist: Extend `JD_MAP` in the Code node with your own open roles. | Workflow setup note |