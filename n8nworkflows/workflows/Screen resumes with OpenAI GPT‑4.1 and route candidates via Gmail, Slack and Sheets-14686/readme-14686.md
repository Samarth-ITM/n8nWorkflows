Screen resumes with OpenAI GPT‑4.1 and route candidates via Gmail, Slack and Sheets

https://n8nworkflows.xyz/workflows/screen-resumes-with-openai-gpt-4-1-and-route-candidates-via-gmail--slack-and-sheets-14686


# Screen resumes with OpenAI GPT‑4.1 and route candidates via Gmail, Slack and Sheets

# 1. Workflow Overview

This workflow automates resume intake and first-pass candidate screening using OpenAI, then routes each candidate into one of three outcomes:

- **Accepted**: send a positive email and log the result in Google Sheets
- **Borderline**: notify HR in Slack for manual review and log the result
- **Rejected**: send a rejection email and log the result

Its main use case is inbound resume screening for a single target role defined in the workflow configuration. A candidate submits a resume through a webhook, the PDF text is extracted, an AI model scores the resume against the configured job role, and score thresholds determine the next action.

## 1.1 Input Reception and Workflow Configuration
This block receives the incoming application payload through a webhook and sets the configurable business parameters used later in the workflow, including the target job role and score thresholds.

## 1.2 Resume Extraction and Data Preparation
This block extracts text from the uploaded PDF resume and normalizes key candidate fields such as name, email, and extracted resume text.

## 1.3 AI Evaluation
This block sends the candidate context and resume text to an AI agent backed by OpenAI GPT-4.1-mini and forces a structured JSON output containing score, decision, and reason.

## 1.4 Score-Based Routing
This block applies threshold logic using n8n IF nodes. First it checks whether the score qualifies for automatic acceptance; if not, it checks whether the candidate is at least borderline.

## 1.5 Accepted Candidate Handling
Accepted candidates receive a positive Gmail message and are logged to Google Sheets with status `accepted`.

## 1.6 Borderline Candidate Handling
Borderline candidates are escalated to HR via Slack and logged to Google Sheets with status `escalated`.

## 1.7 Rejected Candidate Handling
Rejected candidates receive a rejection email and are logged to Google Sheets with status `rejected`.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Workflow Configuration

### Overview
This block starts the workflow when a resume is submitted to the webhook endpoint. It also defines reusable workflow variables such as the target role and the two routing thresholds.

### Nodes Involved
- Resume Upload Webhook
- Workflow Configuration

### Node Details

#### Resume Upload Webhook
- **Type and technical role:** `n8n-nodes-base.webhook` — entry point for external HTTP submissions.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `resume-upload`
  - Response mode: `lastNode`
- **Key expressions or variables used:** None directly in parameters.
- **Input and output connections:**
  - No input node; this is a workflow entry point.
  - Outputs to `Workflow Configuration`.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Incorrect HTTP method from client
  - Missing uploaded binary file for downstream PDF extraction
  - Missing body fields such as `candidateName` or `candidateEmail`
  - Large PDF payloads may hit webhook or instance limits
  - Because response mode is `lastNode`, the HTTP caller waits for the entire workflow to finish; long AI/API delays can cause timeout issues
- **Sub-workflow reference:** None.

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set` — injects configurable constants into the item stream.
- **Configuration choices:**
  - Adds:
    - `jobRole` as a placeholder string to be replaced with the actual role
    - `acceptanceThreshold` = `75`
    - `borderlineThreshold` = `50`
  - `includeOtherFields` is enabled, so incoming webhook data is preserved
- **Key expressions or variables used:** Static assignments only.
- **Input and output connections:**
  - Input from `Resume Upload Webhook`
  - Output to `Extract Resume Text`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Placeholder job role left unchanged, causing poor AI scoring quality
  - Misconfigured thresholds, e.g. borderline higher than acceptance, causing routing logic inconsistencies
- **Sub-workflow reference:** None.

---

## 2.2 Resume Extraction and Data Preparation

### Overview
This block converts the uploaded PDF into plain text, then creates a cleaner data structure for downstream AI analysis by extracting candidate metadata and standardizing defaults.

### Nodes Involved
- Extract Resume Text
- Prepare Resume Data

### Node Details

#### Extract Resume Text
- **Type and technical role:** `n8n-nodes-base.extractFromFile` — extracts text from a binary PDF.
- **Configuration choices:**
  - Operation: `pdf`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from `Workflow Configuration`
  - Output to `Prepare Resume Data`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - No binary file present in webhook item
  - Uploaded file is not actually a valid PDF
  - Scanned/image-based PDFs may extract poorly or yield little/no text
  - Corrupted PDFs can fail parsing
- **Sub-workflow reference:** None.

#### Prepare Resume Data
- **Type and technical role:** `n8n-nodes-base.set` — maps extracted text and request body fields into normalized fields for AI scoring.
- **Configuration choices:**
  - Adds:
    - `resumeText = {{ $json.text }}`
    - `candidateName = {{ $json.body.candidateName || "Unknown" }}`
    - `candidateEmail = {{ $json.body.candidateEmail || "" }}`
  - `includeOtherFields` is enabled
- **Key expressions or variables used:**
  - `{{ $json.text }}`
  - `{{ $json.body.candidateName || "Unknown" }}`
  - `{{ $json.body.candidateEmail || "" }}`
- **Input and output connections:**
  - Input from `Extract Resume Text`
  - Output to `AI Resume Scorer`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - `text` missing if extraction fails silently
  - `body` missing if the webhook request was not formed as expected
  - Empty candidate email leads to downstream Gmail failure
  - Unknown candidate name is handled gracefully with a fallback
- **Sub-workflow reference:** None.

---

## 2.3 AI Evaluation

### Overview
This block submits the candidate’s role target, name, and extracted resume text to an AI agent. The agent uses a structured parser to return machine-readable scoring output.

### Nodes Involved
- AI Resume Scorer
- OpenAI Chat Model
- Structured Output Parser

### Node Details

#### AI Resume Scorer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt execution with an attached LLM and structured parser.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - User text combines:
    - configured job role
    - candidate name
    - resume text
  - System message instructs the model to:
    - score 0–100
    - assess skills, experience, education, overall fit
    - produce `accept`, `reject`, or `borderline`
    - remain objective and unbiased
  - Output parser enabled
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.jobRole }}`
  - `{{ $json.candidateName }}`
  - `{{ $json.resumeText }}`
- **Input and output connections:**
  - Main input from `Prepare Resume Data`
  - AI language model input from `OpenAI Chat Model`
  - AI output parser input from `Structured Output Parser`
  - Main output to `Check Score - Accept`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - OpenAI authentication or quota errors
  - LLM timeout or rate limiting
  - Resume text too long for model context window
  - Model output not matching required schema, causing parser failure
  - Decision generated by model may not align with threshold-based routing; thresholds take precedence in actual routing
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model provider for the AI agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - No extra model options configured
  - No built-in tools enabled
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No main input
  - Connected to `AI Resume Scorer` through the `ai_languageModel` port
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Invalid or missing OpenAI credentials
  - Model availability changes in the OpenAI account or region
  - API usage limits
- **Sub-workflow reference:** None.

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema output from the AI.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `score` as number
    - `decision` as enum: `accept`, `reject`, `borderline`
    - `reason` as string
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No main input
  - Connected to `AI Resume Scorer` through the `ai_outputParser` port
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Parser failure if the model returns malformed JSON or wrong types
  - `score` may be outside expected range if the model behaves unexpectedly; there is no explicit post-validation node
- **Sub-workflow reference:** None.

---

## 2.4 Score-Based Routing

### Overview
This block converts the AI score into operational workflow branches. It first checks whether the candidate is accepted, then routes non-accepted candidates into borderline or rejection flows.

### Nodes Involved
- Check Score - Accept
- Check Score - Borderline

### Node Details

#### Check Score - Accept
- **Type and technical role:** `n8n-nodes-base.if` — threshold gate for automatic acceptance.
- **Configuration choices:**
  - Condition: `score >= acceptanceThreshold`
  - Uses strict type validation
- **Key expressions or variables used:**
  - `{{ $json.score }}`
  - `{{ $('Workflow Configuration').first().json.acceptanceThreshold }}`
- **Input and output connections:**
  - Input from `AI Resume Scorer`
  - True output to `Send Acceptance Email`
  - False output to `Check Score - Borderline`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Missing or non-numeric `score`
  - Acceptance threshold accidentally configured as text or invalid number
- **Sub-workflow reference:** None.

#### Check Score - Borderline
- **Type and technical role:** `n8n-nodes-base.if` — secondary threshold gate for manual review.
- **Configuration choices:**
  - Condition: `score >= borderlineThreshold`
  - Uses strict type validation
- **Key expressions or variables used:**
  - `{{ $json.score }}`
  - `{{ $('Workflow Configuration').first().json.borderlineThreshold }}`
- **Input and output connections:**
  - Input from false branch of `Check Score - Accept`
  - True output to `Notify HR - Borderline`
  - False output to `Send Rejection Email`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - If borderline threshold is set too low or too high, many candidates may be misrouted
  - Since accept is checked first, any score above acceptance threshold never reaches this node
- **Sub-workflow reference:** None.

---

## 2.5 Accepted Candidate Handling

### Overview
This block contacts accepted candidates by email and records the decision in Google Sheets for tracking and auditing.

### Nodes Involved
- Send Acceptance Email
- Log to Google Sheets - Accept

### Node Details

#### Send Acceptance Email
- **Type and technical role:** `n8n-nodes-base.gmail` — sends acceptance email through Gmail.
- **Configuration choices:**
  - Recipient: `{{ $json.candidateEmail }}`
  - Subject: `Congratulations! Next Steps in Your Application`
  - Message includes:
    - candidate name
    - score
    - configured job role
- **Key expressions or variables used:**
  - `{{ $json.candidateEmail }}`
  - `{{ $json.candidateName }}`
  - `{{ $json.score }}`
  - `{{ $('Workflow Configuration').first().json.jobRole }}`
- **Input and output connections:**
  - Input from true branch of `Check Score - Accept`
  - Output to `Log to Google Sheets - Accept`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Empty or invalid email address
  - Gmail OAuth/credential issues
  - Sending limits or mailbox restrictions
  - Deliverability/spam issues not visible to n8n
- **Sub-workflow reference:** None.

#### Log to Google Sheets - Accept
- **Type and technical role:** `n8n-nodes-base.googleSheets` — append or update candidate tracking row.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `email`
  - Sheet and document are placeholders to be filled in
  - Mapped columns:
    - `candidate_name`
    - `email`
    - `role`
    - `score`
    - `status = accepted`
    - `notes = reason`
    - `timestamp = {{ $now.toISO() }}`
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.jobRole }}`
  - `{{ $json.candidateEmail }}`
  - `{{ $json.reason }}`
  - `{{ $json.score }}`
  - `{{ $now.toISO() }}`
  - `{{ $json.candidateName }}`
- **Input and output connections:**
  - Input from `Send Acceptance Email`
  - No downstream node
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Invalid Google Sheets credentials
  - Wrong document ID or sheet name
  - Missing `email` can create duplicate/unmatchable rows
  - Sheet schema mismatch if headers differ from expected names
- **Sub-workflow reference:** None.

---

## 2.6 Borderline Candidate Handling

### Overview
This block escalates uncertain cases to HR using Slack rather than emailing the candidate, then records the escalation in Google Sheets.

### Nodes Involved
- Notify HR - Borderline
- Log to Google Sheets - Borderline

### Node Details

#### Notify HR - Borderline
- **Type and technical role:** `n8n-nodes-base.slack` — posts a manual review request to a Slack channel.
- **Configuration choices:**
  - Sends a formatted message including:
    - candidate name
    - email
    - job role
    - score
    - AI decision
    - reason
  - Channel is selected by placeholder value
- **Key expressions or variables used:**
  - `{{ $json.candidateName }}`
  - `{{ $json.candidateEmail }}`
  - `{{ $('Workflow Configuration').first().json.jobRole }}`
  - `{{ $json.score }}`
  - `{{ $json.decision }}`
  - `{{ $json.reason }}`
- **Input and output connections:**
  - Input from true branch of `Check Score - Borderline`
  - Output to `Log to Google Sheets - Borderline`
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases or potential failure types:**
  - Invalid Slack credentials
  - Wrong channel name or ID
  - Missing email/name fields reduces usefulness for HR
- **Sub-workflow reference:** None.

#### Log to Google Sheets - Borderline
- **Type and technical role:** `n8n-nodes-base.googleSheets` — records borderline/escalated candidates.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `email`
  - Status is hardcoded to `escalated`
  - Other fields mirror the accept/reject logging structure
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.jobRole }}`
  - `{{ $json.candidateEmail }}`
  - `{{ $json.reason }}`
  - `{{ $json.score }}`
  - `{{ $now.toISO() }}`
  - `{{ $json.candidateName }}`
- **Input and output connections:**
  - Input from `Notify HR - Borderline`
  - No downstream node
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Same Google Sheets risks as other logging nodes
  - Empty email may prevent intended upsert matching
- **Sub-workflow reference:** None.

---

## 2.7 Rejected Candidate Handling

### Overview
This block sends a rejection email to low-scoring candidates and writes the result to Google Sheets.

### Nodes Involved
- Send Rejection Email
- Log to Google Sheets - Reject

### Node Details

#### Send Rejection Email
- **Type and technical role:** `n8n-nodes-base.gmail` — sends the rejection message through Gmail.
- **Configuration choices:**
  - Recipient: `{{ $json.candidateEmail }}`
  - Subject: `Thank You for Your Application`
  - Message references the configured job role and provides a polite rejection
- **Key expressions or variables used:**
  - `{{ $json.candidateEmail }}`
  - `{{ $json.candidateName }}`
  - `{{ $('Workflow Configuration').first().json.jobRole }}`
- **Input and output connections:**
  - Input from false branch of `Check Score - Borderline`
  - Output to `Log to Google Sheets - Reject`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Empty or invalid email address
  - Gmail auth/rate-limit issues
- **Sub-workflow reference:** None.

#### Log to Google Sheets - Reject
- **Type and technical role:** `n8n-nodes-base.googleSheets` — stores rejected candidates in the tracking sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `email`
  - Status is hardcoded to `rejected`
  - Same document and sheet placeholders as the other Sheets nodes
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.jobRole }}`
  - `{{ $json.candidateEmail }}`
  - `{{ $json.reason }}`
  - `{{ $json.score }}`
  - `{{ $now.toISO() }}`
  - `{{ $json.candidateName }}`
- **Input and output connections:**
  - Input from `Send Rejection Email`
  - No downstream node
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Incorrect sheet column names
  - Missing email disrupts append/update matching behavior
- **Sub-workflow reference:** None.

---

## 2.8 Sticky Notes / In-Canvas Documentation

### Overview
These nodes provide visual documentation inside the workflow editor. They do not affect execution but are important for operator understanding and should be preserved when reproducing the workflow.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote` — global explanatory note.
- **Configuration choices:** Contains a full workflow summary and setup checklist.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the input/configuration area.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the resume processing area.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Empty visual grouping note.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the borderline flow.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the rejection branch.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the acceptance branch.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the AI evaluation area.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the routing area.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the low-score/borderline area.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Resume Upload Webhook | webhook | Receives resume submissions by HTTP POST |  | Workflow Configuration | ## How it works<br>This workflow automates resume screening using AI. When a candidate uploads a resume via webhook, the system extracts the text and evaluates it against a target job role.<br>An AI agent scores the candidate (0–100) based on skills, experience, education, and overall fit. Based on thresholds, candidates are automatically categorized as accepted, rejected, or borderline.<br>Accepted candidates receive a success email, rejected candidates receive a polite rejection, and borderline candidates are flagged for manual HR review via Slack.<br>All outcomes are logged into Google Sheets for tracking and audit.<br>## Setup steps<br>- Configure webhook for resume upload<br>- Add OpenAI API credentials<br>- Set Gmail and Slack credentials<br>- Connect Google Sheets for logging<br>- Define job role and score thresholds<br>## Input Layer<br>Webhook receives resume and Define job role and scoring thresholds |
| Workflow Configuration | set | Defines job role and score thresholds | Resume Upload Webhook | Extract Resume Text | ## How it works<br>This workflow automates resume screening using AI. When a candidate uploads a resume via webhook, the system extracts the text and evaluates it against a target job role.<br>An AI agent scores the candidate (0–100) based on skills, experience, education, and overall fit. Based on thresholds, candidates are automatically categorized as accepted, rejected, or borderline.<br>Accepted candidates receive a success email, rejected candidates receive a polite rejection, and borderline candidates are flagged for manual HR review via Slack.<br>All outcomes are logged into Google Sheets for tracking and audit.<br>## Setup steps<br>- Configure webhook for resume upload<br>- Add OpenAI API credentials<br>- Set Gmail and Slack credentials<br>- Connect Google Sheets for logging<br>- Define job role and score thresholds<br>## Input Layer<br>Webhook receives resume and Define job role and scoring thresholds |
| Extract Resume Text | extractFromFile | Extracts text from uploaded PDF resume | Workflow Configuration | Prepare Resume Data | ## How it works<br>This workflow automates resume screening using AI. When a candidate uploads a resume via webhook, the system extracts the text and evaluates it against a target job role.<br>An AI agent scores the candidate (0–100) based on skills, experience, education, and overall fit. Based on thresholds, candidates are automatically categorized as accepted, rejected, or borderline.<br>Accepted candidates receive a success email, rejected candidates receive a polite rejection, and borderline candidates are flagged for manual HR review via Slack.<br>All outcomes are logged into Google Sheets for tracking and audit.<br>## Setup steps<br>- Configure webhook for resume upload<br>- Add OpenAI API credentials<br>- Set Gmail and Slack credentials<br>- Connect Google Sheets for logging<br>- Define job role and score thresholds |
| Prepare Resume Data | set | Normalizes extracted text and candidate fields | Extract Resume Text | AI Resume Scorer | ## How it works<br>This workflow automates resume screening using AI. When a candidate uploads a resume via webhook, the system extracts the text and evaluates it against a target job role.<br>An AI agent scores the candidate (0–100) based on skills, experience, education, and overall fit. Based on thresholds, candidates are automatically categorized as accepted, rejected, or borderline.<br>Accepted candidates receive a success email, rejected candidates receive a polite rejection, and borderline candidates are flagged for manual HR review via Slack.<br>All outcomes are logged into Google Sheets for tracking and audit.<br>## Setup steps<br>- Configure webhook for resume upload<br>- Add OpenAI API credentials<br>- Set Gmail and Slack credentials<br>- Connect Google Sheets for logging<br>- Define job role and score thresholds<br>## Resume Processing<br>Extract and structure resume text data |
| AI Resume Scorer | @n8n/n8n-nodes-langchain.agent | Scores the resume and returns structured decision data | Prepare Resume Data; OpenAI Chat Model; Structured Output Parser | Check Score - Accept | ## AI Evaluation<br>Score candidate and generate decision |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the GPT model used by the AI agent |  | AI Resume Scorer | ## AI Evaluation<br>Score candidate and generate decision |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output from AI |  | AI Resume Scorer | ## AI Evaluation<br>Score candidate and generate decision |
| Check Score - Accept | if | Routes high-scoring candidates to acceptance flow | AI Resume Scorer | Send Acceptance Email; Check Score - Borderline | ## Decision Routing<br>Route candidates based on score thresholds |
| Check Score - Borderline | if | Routes non-accepted candidates to borderline or rejection flow | Check Score - Accept | Notify HR - Borderline; Send Rejection Email | ## Decision Routing<br>Route candidates based on score thresholds<br>## flow for low or borderline scored resumes |
| Send Acceptance Email | gmail | Sends acceptance email | Check Score - Accept | Log to Google Sheets - Accept | ## accepted resumes<br>Send acceptance email and log to sheets |
| Log to Google Sheets - Accept | googleSheets | Logs accepted candidate outcome | Send Acceptance Email |  | ## accepted resumes<br>Send acceptance email and log to sheets |
| Notify HR - Borderline | slack | Sends Slack notification for manual HR review | Check Score - Borderline | Log to Google Sheets - Borderline | ## Borderline Flow<br>Notify HR in Slack for manual review |
| Log to Google Sheets - Borderline | googleSheets | Logs escalated borderline candidate outcome | Notify HR - Borderline |  | ## Borderline Flow<br>Notify HR in Slack for manual review |
| Send Rejection Email | gmail | Sends rejection email | Check Score - Borderline | Log to Google Sheets - Reject | ## Rejection email and logging <br>Send rejection email and log to sheets |
| Log to Google Sheets - Reject | googleSheets | Logs rejected candidate outcome | Send Rejection Email |  | ## Rejection email and logging <br>Send rejection email and log to sheets |
| Sticky Note | stickyNote | Canvas documentation |  |  |  |
| Sticky Note1 | stickyNote | Canvas documentation |  |  |  |
| Sticky Note2 | stickyNote | Canvas documentation |  |  |  |
| Sticky Note3 | stickyNote | Canvas documentation |  |  |  |
| Sticky Note4 | stickyNote | Canvas documentation |  |  |  |
| Sticky Note5 | stickyNote | Canvas documentation |  |  |  |
| Sticky Note6 | stickyNote | Canvas documentation |  |  |  |
| Sticky Note7 | stickyNote | Canvas documentation |  |  |  |
| Sticky Note8 | stickyNote | Canvas documentation |  |  |  |
| Sticky Note9 | stickyNote | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node** named `Resume Upload Webhook`.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `resume-upload`
   - Response Mode: `Last Node`
   - Expect the incoming request to include:
     - a PDF file as binary data
     - optional body fields such as `candidateName` and `candidateEmail`

3. **Add a Set node** named `Workflow Configuration`.
   - Enable keeping other input fields.
   - Add these fields:
     - `jobRole` = your target role, for example `Software Engineer`
     - `acceptanceThreshold` = `75`
     - `borderlineThreshold` = `50`
   - Connect `Resume Upload Webhook -> Workflow Configuration`.

4. **Add an Extract From File node** named `Extract Resume Text`.
   - Operation: `PDF`
   - Connect `Workflow Configuration -> Extract Resume Text`.
   - Ensure the incoming binary property from the webhook is the one this node will read. If your upload uses a non-default binary field name, adjust the node accordingly.

5. **Add another Set node** named `Prepare Resume Data`.
   - Enable keeping other input fields.
   - Add:
     - `resumeText` = `{{ $json.text }}`
     - `candidateName` = `{{ $json.body.candidateName || "Unknown" }}`
     - `candidateEmail` = `{{ $json.body.candidateEmail || "" }}`
   - Connect `Extract Resume Text -> Prepare Resume Data`.

6. **Add an AI Agent node** named `AI Resume Scorer`.
   - Type: LangChain Agent
   - Prompt type: define inside the node
   - In the main text/prompt field, use:
     - `Job Role: {{ $('Workflow Configuration').first().json.jobRole }}`
     - `Candidate Name: {{ $json.candidateName }}`
     - `Resume Text: {{ $json.resumeText }}`
   - Set the system message to instruct the model to:
     - evaluate resume fit objectively
     - score 0–100
     - produce a decision of `accept`, `reject`, or `borderline`
     - provide a brief reason
   - Enable structured output parsing.
   - Connect `Prepare Resume Data -> AI Resume Scorer`.

7. **Add an OpenAI Chat Model node** named `OpenAI Chat Model`.
   - Type: OpenAI Chat Model
   - Model: `gpt-4.1-mini`
   - Add OpenAI credentials.
   - Connect this node to the AI Agent using the **AI language model** connection, not the main connection.

8. **Add a Structured Output Parser node** named `Structured Output Parser`.
   - Schema type: manual
   - Use a JSON schema requiring:
     - `score` as number
     - `decision` as string enum with `accept`, `reject`, `borderline`
     - `reason` as string
   - Connect this node to the AI Agent using the **AI output parser** connection.

9. **Add an IF node** named `Check Score - Accept`.
   - Condition type: number
   - Expression left value: `{{ $json.score }}`
   - Operation: `greater than or equal`
   - Right value: `{{ $('Workflow Configuration').first().json.acceptanceThreshold }}`
   - Use strict type validation if available.
   - Connect `AI Resume Scorer -> Check Score - Accept`.

10. **Add another IF node** named `Check Score - Borderline`.
    - Condition type: number
    - Expression left value: `{{ $json.score }}`
    - Operation: `greater than or equal`
    - Right value: `{{ $('Workflow Configuration').first().json.borderlineThreshold }}`
    - Connect the **false** output of `Check Score - Accept` to this node.

11. **Add a Gmail node** named `Send Acceptance Email`.
    - Operation: send email
    - To: `{{ $json.candidateEmail }}`
    - Subject: `Congratulations! Next Steps in Your Application`
    - Body should include:
      - candidate name
      - score
      - job role from `Workflow Configuration`
    - Add Gmail credentials.
    - Connect the **true** output of `Check Score - Accept` to this node.

12. **Add a Google Sheets node** named `Log to Google Sheets - Accept`.
    - Operation: `Append or Update`
    - Set Google Sheets credentials.
    - Select your spreadsheet document ID.
    - Select your sheet name, for example `Candidates`.
    - Use `email` as the matching column.
    - Map these columns:
      - `candidate_name` = `{{ $json.candidateName }}`
      - `email` = `{{ $json.candidateEmail }}`
      - `role` = `{{ $('Workflow Configuration').first().json.jobRole }}`
      - `score` = `{{ $json.score }}`
      - `status` = `accepted`
      - `notes` = `{{ $json.reason }}`
      - `timestamp` = `{{ $now.toISO() }}`
    - Connect `Send Acceptance Email -> Log to Google Sheets - Accept`.

13. **Add a Slack node** named `Notify HR - Borderline`.
    - Operation: send message to channel
    - Choose the target channel, for example `#hr-reviews`
    - Message should include:
      - candidate name
      - email
      - role
      - score
      - decision
      - reason
    - Add Slack credentials.
    - Connect the **true** output of `Check Score - Borderline` to this node.

14. **Add a Google Sheets node** named `Log to Google Sheets - Borderline`.
    - Operation: `Append or Update`
    - Use the same spreadsheet and matching column `email`
    - Map:
      - `candidate_name` = `{{ $json.candidateName }}`
      - `email` = `{{ $json.candidateEmail }}`
      - `role` = `{{ $('Workflow Configuration').first().json.jobRole }}`
      - `score` = `{{ $json.score }}`
      - `status` = `escalated`
      - `notes` = `{{ $json.reason }}`
      - `timestamp` = `{{ $now.toISO() }}`
    - Connect `Notify HR - Borderline -> Log to Google Sheets - Borderline`.

15. **Add a Gmail node** named `Send Rejection Email`.
    - Operation: send email
    - To: `{{ $json.candidateEmail }}`
    - Subject: `Thank You for Your Application`
    - Body should politely reject the candidate and reference the configured job role.
    - Use the same Gmail credentials as the acceptance email node.
    - Connect the **false** output of `Check Score - Borderline` to this node.

16. **Add a Google Sheets node** named `Log to Google Sheets - Reject`.
    - Operation: `Append or Update`
    - Use the same spreadsheet and matching column `email`
    - Map:
      - `candidate_name` = `{{ $json.candidateName }}`
      - `email` = `{{ $json.candidateEmail }}`
      - `role` = `{{ $('Workflow Configuration').first().json.jobRole }}`
      - `score` = `{{ $json.score }}`
      - `status` = `rejected`
      - `notes` = `{{ $json.reason }}`
      - `timestamp` = `{{ $now.toISO() }}`
    - Connect `Send Rejection Email -> Log to Google Sheets - Reject`.

17. **Add credentials** for all external services:
    - **OpenAI**: API key with access to `gpt-4.1-mini`
    - **Gmail**: OAuth2 or supported Gmail auth in n8n
    - **Slack**: workspace app/token with permission to post to the selected channel
    - **Google Sheets**: OAuth2 or service account with access to the spreadsheet

18. **Create the target Google Sheet structure** before testing.
    - Recommended headers:
      - `candidate_name`
      - `email`
      - `role`
      - `score`
      - `status`
      - `notes`
      - `timestamp`

19. **Optional but recommended: add canvas notes** matching the original workflow:
    - Input Layer
    - Resume Processing
    - AI Evaluation
    - Decision Routing
    - accepted resumes
    - Borderline Flow
    - Rejection email and logging
    - flow for low or borderline scored resumes
    - global “How it works / Setup steps” note

20. **Test the workflow** with a sample multipart/form-data POST request.
    - Include a real text-based PDF
    - Include `candidateName` and `candidateEmail`
    - Confirm:
      - PDF extraction succeeds
      - AI returns valid structured output
      - routing matches score thresholds
      - Gmail/Slack/Sheets integrations all work

21. **Replace all placeholders** before production use:
    - `jobRole`
    - Slack channel
    - Google Sheets document ID
    - Google Sheets sheet name

### Expected Input/Output Behavior
- **Input:** webhook POST containing a resume PDF and optional candidate metadata
- **Intermediate output:** AI returns JSON-like fields `score`, `decision`, and `reason`
- **Final output:** because the webhook uses `lastNode` response mode, the HTTP caller receives the output from the last executed branch node, typically one of the Google Sheets logging nodes

### Important Rebuild Notes
- This workflow has **one entry point only**: `Resume Upload Webhook`
- There are **no sub-workflows**
- The AI decision text is generated but **routing is based on numeric thresholds**, not the AI’s `decision` field
- If you want consistency, consider adding validation to ensure the model’s decision matches the threshold logic

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates resume screening using AI. When a candidate uploads a resume via webhook, the system extracts the text and evaluates it against a target job role. | Global workflow note |
| An AI agent scores the candidate (0–100) based on skills, experience, education, and overall fit. Based on thresholds, candidates are automatically categorized as accepted, rejected, or borderline. | Global workflow note |
| Accepted candidates receive a success email, rejected candidates receive a polite rejection, and borderline candidates are flagged for manual HR review via Slack. | Global workflow note |
| All outcomes are logged into Google Sheets for tracking and audit. | Global workflow note |
| Configure webhook for resume upload | Setup note |
| Add OpenAI API credentials | Setup note |
| Set Gmail and Slack credentials | Setup note |
| Connect Google Sheets for logging | Setup note |
| Define job role and score thresholds | Setup note |
| Input Layer — Webhook receives resume and Define job role and scoring thresholds | Canvas note |
| Resume Processing — Extract and structure resume text data | Canvas note |
| AI Evaluation — Score candidate and generate decision | Canvas note |
| Decision Routing — Route candidates based on score thresholds | Canvas note |
| Borderline Flow — Notify HR in Slack for manual review | Canvas note |
| Rejection email and logging — Send rejection email and log to sheets | Canvas note |
| accepted resumes — Send acceptance email and log to sheets | Canvas note |
| flow for low or borderline scored resumes | Canvas note |

### Additional Implementation Notes
- The workflow assumes the resume is uploaded as a **PDF**. Other formats are not handled.
- The workflow tolerates missing `candidateName` by defaulting to `Unknown`, but does **not** safely handle missing email for Gmail delivery.
- The workflow logs all outcomes in the same Google Sheet using `email` as the upsert key.
- Since the webhook waits for the final node, production deployments may want asynchronous acknowledgement if response time becomes too high.
- One sticky note is visually present but empty (`Sticky Note3`), so it has no documentation content to preserve.