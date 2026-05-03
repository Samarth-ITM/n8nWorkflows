Screen CVs against job descriptions with Gmail, easybits, Airtable and Slack

https://n8nworkflows.xyz/workflows/screen-cvs-against-job-descriptions-with-gmail--easybits--airtable-and-slack-14295


# Screen CVs against job descriptions with Gmail, easybits, Airtable and Slack

# 1. Workflow Overview

This workflow automates CV screening from inbound Gmail messages. It watches for unread emails with likely application-related subjects, downloads a PDF attachment, sends the CV to easybits.tech for structured extraction, retrieves an open job from Airtable, computes a skill-match score, routes the candidate by threshold, stores the result in Airtable, and posts a Slack notification.

Its main use case is lightweight applicant pre-screening for recruiters or hiring teams that want to reduce manual first-pass review. It is designed around one active open role at a time, using the first Airtable job record where `status = "Open"`.

## 1.1 Input Reception
The workflow begins with a Gmail trigger that polls for unread application emails containing attachments and specific subject keywords.

## 1.2 Email Download and Attachment Preparation
After detecting a message, it fetches the full Gmail message payload, retrieves a specific attachment, and prepares a base64 PDF payload for the external extraction API.

## 1.3 AI-Based CV Extraction
The prepared PDF is sent to the easybits.tech extraction pipeline, which returns structured candidate information.

## 1.4 Job Retrieval
The workflow queries Airtable for the first open job to obtain the active role requirements, especially the `tech_stack`.

## 1.5 CV Scoring
A Code node compares extracted CV content against the job tech stack and computes a simple percentage score based on matched skills.

## 1.6 Routing by Screening Threshold
An IF node splits candidates into two paths based on the score threshold.

## 1.7 Candidate Persistence and Recruiter Notification
Each branch writes the candidate to Airtable with a status and posts a Slack message summarizing the screening result.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block detects new candidate emails in Gmail. It is the workflow entry point and controls which messages enter the screening pipeline.

### Nodes Involved
- `Gmail: Watch CV Emails`

### Node Details

#### Gmail: Watch CV Emails
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger for Gmail messages.
- **Configuration choices:**
  - Uses advanced mode (`simple: false`)
  - Polls every minute
  - Query filter:
    - `has:attachment`
    - subject contains one of: `Application`, `CV`, `Bewerbung`, `Resume`
    - `after:2026/03/12`
  - Restricts to unread messages
- **Key expressions or variables used:**
  - Gmail query string in `filters.q`
- **Input and output connections:**
  - Entry point of the workflow
  - Outputs to `Download Email & Attachment`
- **Version-specific requirements:**
  - Node version `1.3`
  - Requires Gmail OAuth2 credentials configured in n8n
- **Edge cases or potential failure types:**
  - OAuth token expiration or missing Gmail scopes
  - Query too restrictive, causing no executions
  - Hardcoded `after:` date eventually excluding future desired emails
  - Messages with attachments but no PDF may still pass the trigger
- **Sub-workflow reference:** None

---

## Block 2 — Email Download and Attachment Preparation

### Overview
This block retrieves the full Gmail message and downloads one attachment assumed to be the candidate PDF. It then validates size and exposes the base64 payload expected by the extraction API.

### Nodes Involved
- `Download Email & Attachment`
- `Convert PDF to Base64`
- `Prepare PDF Payload`

### Node Details

#### Download Email & Attachment
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Direct call to Gmail REST API to fetch the full message.
- **Configuration choices:**
  - GET request to:
    `https://gmail.googleapis.com/gmail/v1/users/me/messages/{{ $json.id }}?format=full`
  - Uses predefined Gmail OAuth2 credential
- **Key expressions or variables used:**
  - `{{ $json.id }}` from the Gmail trigger output
- **Input and output connections:**
  - Input from `Gmail: Watch CV Emails`
  - Output to `Convert PDF to Base64`
- **Version-specific requirements:**
  - Node version `4.3`
- **Edge cases or potential failure types:**
  - Gmail API auth failure
  - Message deleted between trigger and fetch
  - Message payload structure may differ from expectations
- **Sub-workflow reference:** None

#### Convert PDF to Base64
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads a Gmail attachment by attachment ID.
- **Configuration choices:**
  - GET request to:
    `https://gmail.googleapis.com/gmail/v1/users/me/messages/{{ $json.id }}/attachments/{{ $json.payload.parts[1].body.attachmentId }}`
  - Uses predefined Gmail OAuth2 credential
  - Assumes attachment lives at `payload.parts[1]`
- **Key expressions or variables used:**
  - `{{ $json.id }}`
  - `{{ $json.payload.parts[1].body.attachmentId }}`
- **Input and output connections:**
  - Input from `Download Email & Attachment`
  - Output to `Prepare PDF Payload`
- **Version-specific requirements:**
  - Node version `4.3`
- **Edge cases or potential failure types:**
  - `payload.parts[1]` may not exist
  - Attachment may not be the second MIME part
  - Message may contain inline images or alternate body parts that shift indices
  - Non-PDF attachment can still be downloaded
- **Sub-workflow reference:** None

#### Prepare PDF Payload
- **Type and technical role:** `n8n-nodes-base.code`  
  Validates attachment size and returns a minimal JSON object with base64 content.
- **Configuration choices:**
  - Reads `data` from the first input item
  - Computes approximate size in KB from base64 string length
  - Throws an error if base64 string length exceeds `800000`
  - Returns:
    - `base64`
    - `sizeKB`
- **Key expressions or variables used:**
  - `$input.first().json.data`
- **Input and output connections:**
  - Input from `Convert PDF to Base64`
  - Output to `easybits: Extract CV Data`
- **Version-specific requirements:**
  - Node version `2`
- **Edge cases or potential failure types:**
  - Missing `data` field if Gmail API response shape is unexpected
  - Size check is approximate and based on encoded length, not exact file bytes
  - Error intentionally stops the workflow for oversized CVs
- **Sub-workflow reference:** None

---

## Block 3 — AI-Based CV Extraction

### Overview
This block sends the encoded PDF to easybits.tech. The API is expected to return structured candidate data such as name, email, LinkedIn URL, skills, education, and employment details.

### Nodes Involved
- `easybits: Extract CV Data`

### Node Details

#### easybits: Extract CV Data
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the CV file to an external AI extraction API.
- **Configuration choices:**
  - POST request to:
    `https://extractor.easybits.tech/api/pipelines/8ppfebcZMIysqGMaUzli`
  - JSON body:
    - `files`: array containing one `data:application/pdf;base64,...` URL
  - Uses generic credential type `httpCustomAuth`
- **Key expressions or variables used:**
  - `{{ $json.base64 }}`
- **Input and output connections:**
  - Input from `Prepare PDF Payload`
  - Output to `Airtable - Fetch Open Job`
- **Version-specific requirements:**
  - Node version `4.3`
  - Requires custom auth credential with the easybits API key
- **Edge cases or potential failure types:**
  - Invalid or expired API key
  - Wrong pipeline ID
  - API timeout or quota limitation
  - Invalid PDF or malformed base64
  - Response schema may differ from what downstream nodes expect
- **Sub-workflow reference:** None

---

## Block 4 — Job Retrieval

### Overview
This block selects the active open role from Airtable. The scoring node depends on this record to determine which technologies should be matched against the candidate CV.

### Nodes Involved
- `Airtable - Fetch Open Job`

### Node Details

#### Airtable - Fetch Open Job
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Searches the Jobs table for records matching an Airtable formula.
- **Configuration choices:**
  - Operation: `search`
  - Base ID: configured explicitly
  - Table: `Jobs`
  - Formula: `status = "Open"`
- **Key expressions or variables used:**
  - No expressions; static Airtable formula
- **Input and output connections:**
  - Input from `easybits: Extract CV Data`
  - Output to `Score CV vs Job Description `
- **Version-specific requirements:**
  - Node version `2.1`
  - Requires Airtable Personal Access Token credentials
- **Edge cases or potential failure types:**
  - Wrong base or table ID
  - No records with `status = "Open"`
  - Multiple open records may yield ambiguous selection behavior
  - `tech_stack` missing or not in the expected array/list format
- **Sub-workflow reference:** None

---

## Block 5 — CV Scoring

### Overview
This block merges the extracted CV data and the Airtable job data, then computes a simple skill-match percentage. It also emits a compact summary used later by Slack.

### Nodes Involved
- `Score CV vs Job Description `

### Node Details

#### Score CV vs Job Description 
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript scoring logic.
- **Configuration choices:**
  - Reads extraction data from:
    - `$node["easybits: Extract CV Data"].json.data`
  - Reads the job record from:
    - `$node["Airtable - Fetch Open Job"].json`
  - Converts the extracted CV object into a lowercased string
  - Reads `job.tech_stack || []`
  - Counts matches where each skill string is found in the CV text
  - Computes:
    - `match_score`
    - `matched_count`
    - `total_possible`
    - `details`
    - `name`
    - `email`
    - `linkedin`
- **Key expressions or variables used:**
  - `$node["easybits: Extract CV Data"].json.data`
  - `$node["Airtable - Fetch Open Job"].json`
- **Input and output connections:**
  - Input from `Airtable - Fetch Open Job`
  - Output to `Route by Score `
- **Version-specific requirements:**
  - Node version `2`
- **Edge cases or potential failure types:**
  - Assumes `easybits` response includes `.json.data`
  - Assumes Airtable record exposes `tech_stack` directly as iterable array
  - String matching may produce false positives, e.g. `R` matching unrelated text if used as a skill
  - If `tech_stack` is a comma-separated string instead of array, `forEach` may fail
  - Emits field names that do not match downstream expectations
- **Sub-workflow reference:** None

**Important implementation issue:**  
This node returns `match_score`, `name`, `email`, and `linkedin`, but several downstream nodes expect fields like `matchScore`, `candidate_name`, `contact_email`, `linkedin_url`, and many other extracted fields. As currently built, the workflow has a data contract mismatch.

---

## Block 6 — Routing by Screening Threshold

### Overview
This block routes candidates to one of two branches based on score. In intent, high-scoring candidates are shortlisted and lower-scoring candidates are marked for review.

### Nodes Involved
- `Route by Score `

### Node Details

#### Route by Score 
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branch node.
- **Configuration choices:**
  - Number comparison `gte`
  - Left value: `{{ $json.matchScore }}`
  - Right value: `60`
- **Key expressions or variables used:**
  - `{{ $json.matchScore }}`
- **Input and output connections:**
  - Input from `Score CV vs Job Description `
  - True output to `Airtable: Save Reviewed Candidate`
  - False output to `Airtable: Save Review Candidate`
- **Version-specific requirements:**
  - Node version `2.3`
- **Edge cases or potential failure types:**
  - Upstream node provides `match_score`, not `matchScore`
  - Because of strict type validation, undefined values may route unexpectedly or fail comparison logic
  - Threshold is hardcoded
- **Sub-workflow reference:** None

**Important implementation issue:**  
The IF node checks `matchScore`, but the scoring node emits `match_score`. Unless corrected, this condition will not evaluate as intended.

---

## Block 7 — Candidate Persistence and Recruiter Notification

### Overview
This block stores candidates in Airtable and sends Slack messages. It is split into two branches, but the naming and payloads are inconsistent with the intended shortlisted/review semantics.

### Nodes Involved
- `Airtable: Save Reviewed Candidate`
- `Slack: Notify For Review`
- `Airtable: Save Review Candidate`
- `Slack: Notify For Review1`

### Node Details

#### Airtable: Save Reviewed Candidate
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Creates a candidate record in Airtable.
- **Configuration choices:**
  - Operation: `create`
  - Base ID configured explicitly
  - Table: `Candidates`
  - Sets `status` to `Shortlisted`
  - Attempts to map many candidate fields such as:
    - `skills`
    - `languages`
    - `match_score`
    - `availability`
    - `linkedin_url`
    - `contact_email`
    - `candidate_name`
    - employment and education fields
  - Typecast enabled
- **Key expressions or variables used:**
  - Expressions like `{{ $json.skills.join(', ') }}`
  - `{{ $json.matchScore }}`
  - `{{ $json.candidate_name }}`
- **Input and output connections:**
  - Input from true branch of `Route by Score `
  - Output to `Slack: Notify For Review`
- **Version-specific requirements:**
  - Node version `2.1`
  - Requires Airtable PAT credentials
- **Edge cases or potential failure types:**
  - Most mapped fields are absent from the scoring node output
  - `.join(', ')` fails if `skills` or `languages` are undefined or not arrays
  - `matchScore` mismatch with actual `match_score`
  - Airtable schema mismatch or typecasting issues
- **Sub-workflow reference:** None

#### Slack: Notify For Review
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a structured Slack Incoming Webhook payload.
- **Configuration choices:**
  - POST to a Slack webhook URL
  - Sends Block Kit payload with fields:
    - candidate name
    - score
    - title
    - company
    - email
    - matched job
- **Key expressions or variables used:**
  - `{{ $('Score CV vs Job Description ').item.json.candidate_name }}`
  - `{{ $('Score CV vs Job Description ').item.json.matchScore }}`
  - `{{ $('Score CV vs Job Description ').item.json.current_job_title }}`
  - `{{ $('Score CV vs Job Description ').item.json.current_company_name }}`
  - `{{ $('Score CV vs Job Description ').item.json.contact_email }}`
  - `{{ $('Score CV vs Job Description ').item.json.jobTitle }}`
- **Input and output connections:**
  - Input from `Airtable: Save Reviewed Candidate`
  - No downstream node
- **Version-specific requirements:**
  - Node version `4.3`
- **Edge cases or potential failure types:**
  - Webhook URL invalid or revoked
  - Many referenced fields are not emitted by the scoring node
  - Slack may reject malformed JSON if expressions resolve badly
- **Sub-workflow reference:** None

#### Airtable: Save Review Candidate
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Creates a candidate record in Airtable for the lower-score branch.
- **Configuration choices:**
  - Operation: `create`
  - Same base and table as above
  - Sets `status` to `In Review`
  - Uses the same field mappings as the shortlisted node
- **Key expressions or variables used:**
  - Same mapping pattern as above
- **Input and output connections:**
  - Input from false branch of `Route by Score `
  - Output to `Slack: Notify For Review1`
- **Version-specific requirements:**
  - Node version `2.1`
- **Edge cases or potential failure types:**
  - Same field mismatch issues as the shortlisted Airtable node
- **Sub-workflow reference:** None

#### Slack: Notify For Review1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a plain-text Slack message for the lower-score branch.
- **Configuration choices:**
  - POST to a second Slack webhook URL
  - Sends text including:
    - candidate name
    - score
    - role
    - matched skills
    - LinkedIn URL
- **Key expressions or variables used:**
  - `{{ $node['Score CV vs Job Description '].json.name }}`
  - `{{ $node['Score CV vs Job Description '].json.match_score }}`
  - `{{ $node['Airtable - Fetch Open Job'].json.job_title }}`
  - `{{ $node['Score CV vs Job Description '].json.details }}`
  - `{{ $node['Score CV vs Job Description '].json.linkedin }}`
- **Input and output connections:**
  - Input from `Airtable: Save Review Candidate`
  - No downstream node
- **Version-specific requirements:**
  - Node version `4.3`
- **Edge cases or potential failure types:**
  - Webhook invalid or rate-limited
  - Airtable job record may not expose `job_title` exactly as expected
  - This node uses the scoring node’s actual field names more consistently than the other Slack node
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note — Header | n8n-nodes-base.stickyNote | Canvas documentation and setup guidance |  |  | ## 📄 CV & Resume Screener Powered by Gmail · easybits.tech · Airtable · Slack Automatically reads CVs from Gmail, extracts structured candidate data using AI, scores each applicant against your open job requirements, saves results to Airtable, and notifies your recruiter on Slack — all without manual review. / Prerequisites: Gmail OAuth2; easybits.tech API key; Airtable Jobs and Candidates tables; Slack Incoming Webhooks. / Customisation: score threshold, email filter, scoring logic, Airtable fields. |
| Sticky Note — Step 1 | n8n-nodes-base.stickyNote | Canvas note for Gmail trigger setup |  |  | ### Step 1 · Get Email & Attachment / Gmail Trigger polls every minute for unread emails matching `has:attachment subject:(Application OR CV OR Resume OR Bewerbung)` / Setup required: Gmail OAuth2; adjust `after:` date and subject keywords. |
| Sticky Note — Step 2 | n8n-nodes-base.stickyNote | Canvas note for Gmail message and attachment retrieval |  |  | ### Step 2 · Download & Prepare PDF / Downloads the full email via Gmail API, extracts the PDF attachment, and base64-encodes it. / File size limit: PDFs over ~600 KB will be rejected. / Attachment index `parts[1]` assumes the PDF is the second part. |
| Sticky Note — Step 3 | n8n-nodes-base.stickyNote | Canvas note for easybits extraction |  |  | ### Step 3 · Extract CV with easybits / Sends the base64 PDF to the easybits.tech AI pipeline and returns structured JSON. / Setup required: add API key to `httpCustomAuth`; replace pipeline ID; [easybits.tech](https://easybits.tech) |
| Sticky Note — Step 4 | n8n-nodes-base.stickyNote | Canvas note for Airtable job retrieval |  |  | ### Step 4 · Fetch Open Job / Queries Airtable for the first record where `status = "Open"` to get active job requirements. / Setup required: update Airtable Base ID and Jobs table ID; ensure `status` and `tech_stack` fields; run the Job Description Parser workflow first. |
| Sticky Note — Step 5 | n8n-nodes-base.stickyNote | Canvas note for scoring logic |  |  | ### Step 5 · Score CV vs Job / Compares extracted CV text against each item in `tech_stack`. / Scoring formula: `score = matched skills ÷ total required skills × 100` / Customise weighted scoring if needed. |
| Sticky Note — Step 6 | n8n-nodes-base.stickyNote | Canvas note for threshold routing |  |  | ### Step 6 · Route by Score / ≥ 60% → Shortlisted / < 60% → In Review / Threshold can be changed in the IF node. |
| Sticky Note — Step 7 | n8n-nodes-base.stickyNote | Canvas note for Airtable save and Slack notify |  |  | ### Step 7 · Save to Airtable & Notify on Slack / Creates a Candidates record with either `Shortlisted` or `In Review` status. / Posts candidate summary to Slack. / Setup required: update Airtable IDs and Slack webhook URLs. |
| Gmail: Watch CV Emails | n8n-nodes-base.gmailTrigger | Poll Gmail for unread application emails with attachments |  | Download Email & Attachment | ### Step 1 · Get Email & Attachment / Gmail Trigger polls every minute for unread emails matching `has:attachment subject:(Application OR CV OR Resume OR Bewerbung)` / Setup required: Gmail OAuth2; adjust `after:` date and subject keywords. |
| Download Email & Attachment | n8n-nodes-base.httpRequest | Fetch full Gmail message payload | Gmail: Watch CV Emails | Convert PDF to Base64 | ### Step 2 · Download & Prepare PDF / Downloads the full email via Gmail API, extracts the PDF attachment, and base64-encodes it. / File size limit: PDFs over ~600 KB will be rejected. / Attachment index `parts[1]` assumes the PDF is the second part. |
| Convert PDF to Base64 | n8n-nodes-base.httpRequest | Download Gmail attachment data | Download Email & Attachment | Prepare PDF Payload | ### Step 2 · Download & Prepare PDF / Downloads the full email via Gmail API, extracts the PDF attachment, and base64-encodes it. / File size limit: PDFs over ~600 KB will be rejected. / Attachment index `parts[1]` assumes the PDF is the second part. |
| Prepare PDF Payload | n8n-nodes-base.code | Validate attachment size and expose base64 payload | Convert PDF to Base64 | easybits: Extract CV Data | ### Step 2 · Download & Prepare PDF / Downloads the full email via Gmail API, extracts the PDF attachment, and base64-encodes it. / File size limit: PDFs over ~600 KB will be rejected. / Attachment index `parts[1]` assumes the PDF is the second part. |
| easybits: Extract CV Data | n8n-nodes-base.httpRequest | Send CV PDF to AI extraction pipeline | Prepare PDF Payload | Airtable - Fetch Open Job | ### Step 3 · Extract CV with easybits / Sends the base64 PDF to the easybits.tech AI pipeline and returns structured JSON. / Setup required: add API key to `httpCustomAuth`; replace pipeline ID; [easybits.tech](https://easybits.tech) |
| Airtable - Fetch Open Job | n8n-nodes-base.airtable | Search Airtable Jobs table for open role | easybits: Extract CV Data | Score CV vs Job Description  | ### Step 4 · Fetch Open Job / Queries Airtable for the first record where `status = "Open"` to get active job requirements. / Setup required: update Airtable Base ID and Jobs table ID; ensure `status` and `tech_stack` fields; run the Job Description Parser workflow first. |
| Score CV vs Job Description  | n8n-nodes-base.code | Compute skill match score against active job | Airtable - Fetch Open Job | Route by Score  | ### Step 5 · Score CV vs Job / Compares extracted CV text against each item in `tech_stack`. / Scoring formula: `score = matched skills ÷ total required skills × 100` / Customise weighted scoring if needed. |
| Route by Score  | n8n-nodes-base.if | Branch candidate by score threshold | Score CV vs Job Description  | Airtable: Save Reviewed Candidate; Airtable: Save Review Candidate | ### Step 6 · Route by Score / ≥ 60% → Shortlisted / < 60% → In Review / Threshold can be changed in the IF node. |
| Airtable: Save Reviewed Candidate | n8n-nodes-base.airtable | Create shortlisted candidate record | Route by Score  | Slack: Notify For Review | ### Step 7 · Save to Airtable & Notify on Slack / Creates a Candidates record with either `Shortlisted` or `In Review` status. / Posts candidate summary to Slack. / Setup required: update Airtable IDs and Slack webhook URLs. |
| Slack: Notify For Review | n8n-nodes-base.httpRequest | Send Slack notification for true branch | Airtable: Save Reviewed Candidate |  | ### Step 7 · Save to Airtable & Notify on Slack / Creates a Candidates record with either `Shortlisted` or `In Review` status. / Posts candidate summary to Slack. / Setup required: update Airtable IDs and Slack webhook URLs. |
| Airtable: Save Review Candidate | n8n-nodes-base.airtable | Create in-review candidate record | Route by Score  | Slack: Notify For Review1 | ### Step 7 · Save to Airtable & Notify on Slack / Creates a Candidates record with either `Shortlisted` or `In Review` status. / Posts candidate summary to Slack. / Setup required: update Airtable IDs and Slack webhook URLs. |
| Slack: Notify For Review1 | n8n-nodes-base.httpRequest | Send Slack notification for false branch | Airtable: Save Review Candidate |  | ### Step 7 · Save to Airtable & Notify on Slack / Creates a Candidates record with either `Shortlisted` or `In Review` status. / Posts candidate summary to Slack. / Setup required: update Airtable IDs and Slack webhook URLs. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `CV screening`.

2. **Add a Gmail Trigger node**
   - Type: `Gmail Trigger`
   - Name: `Gmail: Watch CV Emails`
   - Connect Gmail OAuth2 credentials.
   - Set polling to every minute.
   - Use advanced mode, not simple mode.
   - Set search query to:
     - `has:attachment subject:(Application OR CV OR Bewerbung OR Resume) after:2026/03/12`
   - Set read status filter to `unread`.

3. **Add an HTTP Request node to fetch the full Gmail message**
   - Type: `HTTP Request`
   - Name: `Download Email & Attachment`
   - Method: `GET`
   - URL:
     - `=https://gmail.googleapis.com/gmail/v1/users/me/messages/{{ $json.id }}?format=full`
   - Authentication: `Predefined Credential Type`
   - Credential type: `gmailOAuth2`
   - Use the same Gmail credential as the trigger.
   - Connect from `Gmail: Watch CV Emails`.

4. **Add an HTTP Request node to fetch the Gmail attachment**
   - Type: `HTTP Request`
   - Name: `Convert PDF to Base64`
   - Method: `GET`
   - URL:
     - `=https://gmail.googleapis.com/gmail/v1/users/me/messages/{{ $json.id }}/attachments/{{ $json.payload.parts[1].body.attachmentId }}`
   - Authentication: `Predefined Credential Type`
   - Credential type: `gmailOAuth2`
   - Connect from `Download Email & Attachment`.
   - Note: this assumes the PDF attachment is at `parts[1]`. If your Gmail MIME structure differs, inspect a sample execution and adjust the path.

5. **Add a Code node to validate and prepare the PDF**
   - Type: `Code`
   - Name: `Prepare PDF Payload`
   - Use JavaScript.
   - Paste logic equivalent to:
     - read `data` from the Gmail attachment response
     - estimate size
     - reject payloads larger than about 800000 base64 characters
     - return `{ base64, sizeKB }`
   - Connect from `Convert PDF to Base64`.

6. **Create easybits authentication**
   - In n8n credentials, create `HTTP Custom Auth`.
   - Name it something meaningful.
   - Configure it according to easybits.tech API requirements using your API key.
   - Make sure the auth method matches the easybits endpoint expectation.

7. **Add an HTTP Request node for CV extraction**
   - Type: `HTTP Request`
   - Name: `easybits: Extract CV Data`
   - Method: `POST`
   - URL:
     - `https://extractor.easybits.tech/api/pipelines/8ppfebcZMIysqGMaUzli`
   - Replace the pipeline ID with your own pipeline if needed.
   - Body type: JSON
   - JSON body:
     - `files` = array containing `data:application/pdf;base64,{{ $json.base64 }}`
   - Authentication: `Generic Credential Type`
   - Generic auth type: `httpCustomAuth`
   - Connect from `Prepare PDF Payload`.

8. **Prepare Airtable**
   - Create an Airtable base.
   - Add a `Jobs` table and a `Candidates` table.
   - In `Jobs`, include at least:
     - `status`
     - `tech_stack`
     - optionally `job_title`
   - In `Candidates`, create all fields you plan to map.
   - Generate an Airtable Personal Access Token with appropriate permissions.
   - Add the credential in n8n.

9. **Add an Airtable node to fetch the open job**
   - Type: `Airtable`
   - Name: `Airtable - Fetch Open Job`
   - Operation: `Search`
   - Select your base
   - Select the `Jobs` table
   - Filter formula:
     - `status = "Open"`
   - Connect from `easybits: Extract CV Data`.

10. **Add a Code node for scoring**
    - Type: `Code`
    - Name: `Score CV vs Job Description `
    - Configure JavaScript to:
      - read extraction data from `easybits: Extract CV Data`
      - read the open job from `Airtable - Fetch Open Job`
      - turn extracted CV data into lowercased text
      - iterate through `tech_stack`
      - count matches
      - calculate percentage score
    - The current workflow uses output fields:
      - `match_score`
      - `matched_count`
      - `total_possible`
      - `details`
      - `name`
      - `email`
      - `linkedin`
    - Connect from `Airtable - Fetch Open Job`.

11. **Recommended correction before proceeding**
    - To make the rest of the workflow function, either:
      - change this Code node to also emit:
        - `matchScore`
        - `candidate_name`
        - `contact_email`
        - `linkedin_url`
        - any other Airtable/Slack-required fields
      - or update all downstream nodes to use the fields actually returned.
    - Without this correction, the workflow will not work reliably.

12. **Add an IF node for routing**
    - Type: `IF`
    - Name: `Route by Score `
    - Set a numeric comparison:
      - left value: candidate score
      - operation: greater than or equal
      - right value: `60`
    - In the supplied workflow, left value is `{{ $json.matchScore }}`, but if you keep the current scoring output it should instead be `{{ $json.match_score }}`.
    - Connect from `Score CV vs Job Description `.

13. **Add Airtable node for shortlisted candidates**
    - Type: `Airtable`
    - Name: `Airtable: Save Reviewed Candidate`
    - Operation: `Create`
    - Base: your Airtable base
    - Table: `Candidates`
    - Map candidate fields.
    - Set `status` to `Shortlisted`.
    - Enable typecast if useful.
    - Connect from the true output of `Route by Score `.

14. **Add Slack webhook for shortlisted branch**
    - In Slack, create an Incoming Webhook for the desired channel.
    - Add an `HTTP Request` node:
      - Name: `Slack: Notify For Review`
      - Method: `POST`
      - URL: your webhook URL
      - Body type: JSON
      - Use either Block Kit or plain text
    - If using the current structure, make sure the expressions reference fields that actually exist.
    - Connect from `Airtable: Save Reviewed Candidate`.

15. **Add Airtable node for lower-score candidates**
    - Type: `Airtable`
    - Name: `Airtable: Save Review Candidate`
    - Operation: `Create`
    - Base: same base
    - Table: `Candidates`
    - Map the same candidate fields
    - Set `status` to `In Review`
    - Connect from the false output of `Route by Score `.

16. **Add Slack webhook for in-review branch**
    - Create a second Slack Incoming Webhook if you want a different channel.
    - Add an `HTTP Request` node:
      - Name: `Slack: Notify For Review1`
      - Method: `POST`
      - URL: second webhook URL
      - Body type: JSON
      - Message text can include candidate name, score, role, matched skills, and LinkedIn
    - Connect from `Airtable: Save Review Candidate`.

17. **Test with sample emails**
    - Send a Gmail message matching the subject filter.
    - Attach a PDF CV.
    - Confirm:
      - Gmail trigger fires
      - Gmail message structure matches `payload.parts[1]`
      - easybits returns the expected schema
      - Airtable returns an open job with `tech_stack`
      - score is computed
      - records are created
      - Slack notifications are delivered

18. **Harden the workflow**
    - Recommended improvements:
      - dynamically detect the PDF attachment instead of hardcoding `parts[1]`
      - normalize field names across nodes
      - handle missing `tech_stack`
      - support multiple open jobs instead of first match only
      - mark processed Gmail messages as read or labeled to avoid duplicate processing

### Credential Configuration Summary

- **Gmail OAuth2**
  - Used by:
    - `Gmail: Watch CV Emails`
    - `Download Email & Attachment`
    - `Convert PDF to Base64`

- **easybits HTTP Custom Auth**
  - Used by:
    - `easybits: Extract CV Data`

- **Airtable Personal Access Token**
  - Used by:
    - `Airtable - Fetch Open Job`
    - `Airtable: Save Reviewed Candidate`
    - `Airtable: Save Review Candidate`

- **Slack Incoming Webhooks**
  - Used directly as URLs in:
    - `Slack: Notify For Review`
    - `Slack: Notify For Review1`

### Sub-workflow Setup
This workflow does not contain any n8n Execute Workflow or sub-workflow nodes.

However, it depends operationally on a separate process mentioned in the notes:

- **Job Description Parser workflow**
  - Expected purpose: populate the Airtable `Jobs` table before CV screening runs
  - Expected output: at least one `Jobs` record with:
    - `status = "Open"`
    - `tech_stack`
    - preferably `job_title`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| easybits.tech is the external AI service used for CV extraction. | https://easybits.tech |
| The workflow notes state that the Airtable `Jobs` table should be populated first by a separate Job Description Parser flow. | Internal dependency mentioned in canvas notes |
| The design intention is: Gmail intake → PDF extraction → AI parsing → Airtable job lookup → score → Airtable save → Slack notify. | Workflow-level context |
| Current implementation contains field-name mismatches between scoring output and downstream Airtable/Slack nodes. This should be corrected before production use. | Important implementation note |
| The Gmail query includes a hardcoded `after:2026/03/12` filter, which may unintentionally exclude valid emails depending on run date. | Trigger configuration note |