Grade and deliver multi-course assignment feedback with GPT-4o, Google Drive, Slack, and Gmail

https://n8nworkflows.xyz/workflows/grade-and-deliver-multi-course-assignment-feedback-with-gpt-4o--google-drive--slack--and-gmail-12731


# Grade and deliver multi-course assignment feedback with GPT-4o, Google Drive, Slack, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name (in JSON):** Multi-Course Assignment Ingestion, Grading, and Feedback Automation System  
**Provided title:** Grade and deliver multi-course assignment feedback with GPT-4o, Google Drive, Slack, and Gmail

**Purpose:**  
Automates ingestion of student assignment submissions from either a webhook (e.g., Google Forms) or a scheduled LMS API poll, normalizes and validates the submission payload, checks for duplicates in PostgreSQL, applies late penalties, runs plagiarism detection and grading with GPT-4o (via n8n LangChain nodes), stores results in PostgreSQL, generates branded HTML feedback, converts it to PDF, uploads artifacts to Google Drive, and notifies instructors/students via Slack and Gmail. It also routes exceptional cases (high plagiarism, late, low score/manual review) to alert channels.

**Primary use cases:**
- Multi-course grading automation with consistent rubric-based feedback
- Plagiarism risk triage and instructor escalation
- Automated feedback PDF creation + Drive storage + email delivery
- Central gradebook and analytics event logging

### 1.1 Entry & Configuration
- **Entry points:** Webhook (form submissions) and Schedule Trigger (LMS polling)
- Centralized config via a Set node (“Workflow Configuration”) containing placeholders and thresholds

### 1.2 Data Collection & Merge
- Fetch pending submissions from LMS API
- Merge with webhook-submitted form data (intended design)

### 1.3 Normalization, Validation, and Deduplication
- Normalize fields across sources
- Validate required fields and email format
- Check duplicates in PostgreSQL

### 1.4 Late Penalty & Rate Limiting
- Compute late status and penalty
- Apply a configurable wait delay before downstream API calls

### 1.5 AI Processing (Plagiarism + Grading)
- Plagiarism agent (GPT-4o) + structured output parsing
- Grading agent (GPT-4o) + structured output parsing

### 1.6 Score Finalization & Exception Routing
- Apply late penalty to score
- Determine exception type (high plagiarism / late / manual review / normal)
- Route to alerts and/or normal processing

### 1.7 Persistence, Feedback Rendering, Distribution, Analytics
- Store full submission record + update gradebook (PostgreSQL)
- Generate HTML feedback → convert to PDF → upload to Drive
- Notify instructors (Slack + email), send student feedback email, log analytics event

---

## 2. Block-by-Block Analysis

### Block 2.1 — Entry Points & Configuration
**Overview:** Accepts submissions from a webhook and/or periodically polls an LMS endpoint. Loads workflow-wide configuration constants (API URLs/keys, thresholds, Drive/Slack targets).  
**Nodes involved:**  
- Google Forms Webhook  
- LMS API Polling Schedule  
- Workflow Configuration

#### Node: Google Forms Webhook
- **Type/role:** `Webhook` — HTTP entry point for POST submissions.
- **Key configuration:**
  - **Path:** `assignment-submission`
  - **Method:** `POST`
  - **Response mode:** `lastNode` (the response is produced at the end of the webhook path, but in this workflow only the validation error path explicitly responds).
- **Connections:**
  - Output → **Workflow Configuration**
- **Potential failures / edge cases:**
  - If no node returns a response in the “success” path, webhook callers may hang or get a default response (depends on n8n webhook settings). Only the validation-failure path has `Respond to Webhook`.
  - Payload shape may differ from expected normalized fields.

#### Node: LMS API Polling Schedule
- **Type/role:** `Schedule Trigger` — periodic trigger to poll LMS.
- **Key configuration:** every **15 minutes**.
- **Connections:** Output → **Fetch LMS Assignments**
- **Potential failures / edge cases:**
  - Overlapping runs if executions take longer than 15 minutes (depends on n8n concurrency settings).

#### Node: Workflow Configuration
- **Type/role:** `Set` — central configuration holder (placeholders).
- **Key configuration values (interpreted):**
  - `lmsApiUrl` / `lmsApiKey` placeholders
  - `plagiarismThreshold` = `0.75`
  - `latePenaltyPercent` = `10` (per day late; later capped at 100%)
  - `driveFolderId`, `slackChannelInstructors`, `slackChannelAlerts` placeholders
  - `pdfConversionApiUrl` = `https://api.html2pdf.app/v1/generate`
  - `maxRetries` = `3` (note: not implemented elsewhere)
  - `rateLimitDelayMs` = `1000`
  - **Include other fields:** enabled
- **Connections:** Output → **Merge Form and API Data** (input 1)
- **Potential failures / edge cases:**
  - Placeholder values must be replaced or downstream nodes will fail (HTTP calls, Drive upload, Slack channel routing).
  - Secrets are stored directly in node parameters (e.g., `lmsApiKey`), which may be undesirable vs. n8n credentials/environment variables.

---

### Block 2.2 — LMS Fetch + Merge
**Overview:** Polls LMS for pending submissions, then merges with the configuration stream. Intended to also merge webhook submission data, but the current wiring merges LMS results with config only.  
**Nodes involved:**  
- Fetch LMS Assignments  
- Merge Form and API Data

#### Node: Fetch LMS Assignments
- **Type/role:** `HTTP Request` — pulls pending submissions from LMS.
- **Key configuration:**
  - **URL:** `{{ lmsApiUrl }}/assignments/submissions`
  - **Query:** `status=pending`, `limit=50`
  - **Auth:** Generic header auth via `Authorization: Bearer {{ lmsApiKey }}`
  - **Timeout:** 30s
- **Connections:** Output → **Merge Form and API Data** (input 0)
- **Potential failures / edge cases:**
  - Invalid URL/key → 401/403
  - LMS rate limiting → 429
  - Response shape mismatch (affects normalization)
  - Timeout on large payloads

#### Node: Merge Form and API Data
- **Type/role:** `Merge` — combines two input streams.
- **Key configuration:** default merge behavior (not explicitly set).
- **Inputs:**
  - Input 0: **Fetch LMS Assignments**
  - Input 1: **Workflow Configuration**
- **Connections:** Output → **Normalize Submission Data**
- **Potential failures / edge cases:**
  - **Design mismatch:** Webhook payload is not connected here; the webhook currently only feeds config, not submission data. As a result, webhook submissions are not actually merged/processed unless the webhook payload is embedded elsewhere (it isn’t).
  - Default merge mode may not produce the intended 1:1 pairing of config with each submission item. Common patterns are “merge by index” or “append”; if config is a single item and LMS returns many, you may get only partial outputs or unexpected pairing.

---

### Block 2.3 — Normalization & Validation
**Overview:** Normalizes variable field names from different sources into a standard schema and validates required fields (FERPA-oriented). Invalid payloads return HTTP 400 (webhook path).  
**Nodes involved:**  
- Normalize Submission Data  
- Validate Submission  
- Check Validation Status  
- Handle Validation Error  
- Respond to Form Submission

#### Node: Normalize Submission Data
- **Type/role:** `Set` — maps/standardizes fields.
- **Key mappings (examples):**
  - `studentId` = `student_id` or `studentId`
  - `studentEmail` = `student_email` or `email`
  - `submissionText` = `submission_text` or `text` or `content`
  - `submittedAt` defaults to `now` if missing
  - `submissionId` = `id` or `{{ $runIndex }}-{{ $now.toMillis() }}`
- **Include other fields:** enabled
- **Connections:** Output → **Validate Submission**
- **Potential failures / edge cases:**
  - If LMS returns nested structures (e.g., `student.email`), mapping will miss it.
  - `deadline` may be missing; later late-penalty code uses `new Date(data.deadline)` which can become “Invalid Date”.

#### Node: Validate Submission
- **Type/role:** `Code` — validates required fields and email format; produces `isValid` and `validationErrors`.
- **Validation rules:**
  - Requires `studentId`, `studentEmail`, `courseId`, `assignmentId`
  - Requires either `submissionText` or `submissionFile`
  - Valid email regex check
- **Connections:** Output → **Check Validation Status**
- **Potential failures / edge cases:**
  - If upstream produces multiple items, code validates all.
  - Email regex is basic and may reject some valid addresses (rare).
  - Missing/invalid `deadline` is not validated here (but used later).

#### Node: Check Validation Status
- **Type/role:** `IF` — branches by `isValid == true`.
- **True output →** Check Duplicate Submission  
- **False output →** Handle Validation Error
- **Edge cases:**
  - Expression uses `$('Validate Submission').item.json.isValid` rather than `$json.isValid`. In multi-item contexts, `.item` may refer to a specific item and can mis-evaluate depending on n8n execution data mapping. Safer is `={{ $json.isValid }}`.

#### Node: Handle Validation Error
- **Type/role:** `Set` — prepares an error payload:
  - `status = validation_failed`
  - `message = "Submission validation failed: ..."`
  - `timestamp = now`
- **Connections:** Output → **Respond to Form Submission**
- **Edge cases:** If this path is triggered by the scheduled LMS polling (not webhook), it still tries to “respond to webhook” (which won’t exist), causing an execution error.

#### Node: Respond to Form Submission
- **Type/role:** `Respond to Webhook` — returns HTTP 400 JSON.
- **Key configuration:**
  - Response code: `400`
  - Body: `{ success: false, message, errors }`
- **Connections:** none (terminal)
- **Edge cases:**
  - Only works when execution originated from a webhook trigger.
  - In schedule-triggered runs, this node will fail because there is no webhook request context.

---

### Block 2.4 — Deduplication & Late Penalty
**Overview:** Prevents duplicate grading by checking a submissions table, then calculates late penalties and waits a configurable delay to reduce downstream rate limits.  
**Nodes involved:**  
- Check Duplicate Submission  
- Is Duplicate?  
- Calculate Deadline Penalty  
- Rate Limit Delay

#### Node: Check Duplicate Submission
- **Type/role:** `Postgres` — query to see if a submission already exists.
- **Query:** counts rows where `(student_id, assignment_id, course_id)` match.
- **Parameterization approach:** uses `queryReplacement` with:
  - `studentId, assignmentId, courseId`
- **Connections:** Output → **Is Duplicate?**
- **Potential failures / edge cases:**
  - `queryReplacement` formatting is nonstandard: `={{ $json.studentId }},={{ $json.assignmentId }},={{ $json.courseId }}`. Depending on node behavior, this can break parameter binding. Typically you’d use proper parameter arrays or use expressions in a safer way.
  - Missing DB credentials or table → runtime error.

#### Node: Is Duplicate?
- **Type/role:** `IF` — checks if count > 0.
- **Routing:**
  - In the workflow connections, the **false** branch goes to **Calculate Deadline Penalty** (i.e., “not duplicate” continues).
  - The **true** branch is unconnected (duplicate submissions are silently dropped).
- **Edge cases:**
  - `leftValue` references `$('Check Duplicate Submission').item.json.count`. Similar multi-item `.item` risk as above.
  - `count` often comes back as a string in Postgres drivers; numeric comparison may behave unexpectedly unless cast.

#### Node: Calculate Deadline Penalty
- **Type/role:** `Code` — computes:
  - `isLate`, `daysLate`, `penaltyPercent = min(daysLate * latePenaltyPercent, 100)`
  - `submissionStatus` set to `late` or `on_time`
- **Dependencies:** uses `latePenaltyPercent` from **Workflow Configuration**
- **Connections:** Output → **Rate Limit Delay**
- **Edge cases:**
  - If `deadline` is missing/invalid, `new Date(data.deadline)` becomes invalid and comparisons yield false, treating as on-time.
  - Timezone differences between submittedAt and deadline.

#### Node: Rate Limit Delay
- **Type/role:** `Wait` — delay to mitigate rate limits.
- **Key configuration:** `amount = rateLimitDelayMs` (default 1000ms).
- **Connections:** Output → **Upload Submission to Drive**
- **Edge cases:** Wait node increases workflow duration; may conflict with schedule frequency if many items.

---

### Block 2.5 — Artifact Storage (Submission) + AI Plagiarism Detection
**Overview:** Stores raw submission text in Drive and runs plagiarism detection via a GPT-4o-powered agent with structured output.  
**Nodes involved:**  
- Upload Submission to Drive  
- Plagiarism Check Agent  
- OpenAI Model - Plagiarism  
- Plagiarism Result Parser

#### Node: Upload Submission to Drive
- **Type/role:** `Google Drive` — uploads a text artifact (intended).
- **Key configuration:**
  - File name: `submission_<submissionId>_<studentId>.txt`
  - Target folder: `driveFolderId` from config
  - Drive: “My Drive”
- **Credentials:** Google Drive OAuth2
- **Connections:** Output → **Plagiarism Check Agent**
- **Important gap / edge case:**
  - The node does not specify **file content** (binary/data). As configured, it may create an empty file or fail (depending on node defaults). Typically you must provide `File Content`/binary property or “Content” field.
  - Output field used later: in DB insertion it references `drive_file_id = {{ $json.id }}` which assumes Drive node returns `id`.

#### Node: Plagiarism Check Agent
- **Type/role:** `LangChain Agent` — prompts GPT model for plagiarism scoring.
- **Prompt includes:** submission text, courseId, assignmentId
- **System message:** plagiarism detection guidance + FERPA note
- **Output:** uses a structured output parser (connected via `ai_outputParser`)
- **Connections:**
  - Main output → **AI Grading Agent**
  - `ai_languageModel` input ← **OpenAI Model - Plagiarism**
  - `ai_outputParser` input ← **Plagiarism Result Parser**
- **Edge cases:**
  - Large submissions can exceed token limits; consider truncation/summarization.
  - Model may produce invalid JSON; parser node will error.

#### Node: OpenAI Model - Plagiarism
- **Type/role:** `lmChatOpenAi` — model provider for the agent.
- **Model:** `gpt-4o`
- **Options:** maxTokens 2000, temperature 0.3
- **Credentials:** OpenAI API
- **Connections:** to Plagiarism Check Agent via `ai_languageModel`
- **Edge cases:** API rate limits, 401 auth, timeouts.

#### Node: Plagiarism Result Parser
- **Type/role:** `Structured Output Parser` — enforces schema:
  - `plagiarismScore` number [0..1]
  - `concerns` array of strings
  - `recommendation` string (accept / flag_for_review / reject)
- **Connections:** to Plagiarism Check Agent via `ai_outputParser`
- **Edge cases:** Any deviation fails parsing; add retry/fallback if needed.

---

### Block 2.6 — AI Grading + Score Finalization
**Overview:** Grades the submission using a rubric, parses structured results, then applies late penalty and computes exception flags.  
**Nodes involved:**  
- AI Grading Agent  
- OpenAI Model - Grading  
- Grading Result Parser  
- Combine Grading Results

#### Node: AI Grading Agent
- **Type/role:** `LangChain Agent` — rubric-based grading and feedback generation.
- **Prompt includes:** submission text, courseId, assignmentId, plagiarismScore, late status
- **System message:** grading rubric (content/structure/evidence/writing)
- **Connections:**
  - Main output → **Combine Grading Results**
  - `ai_languageModel` input ← **OpenAI Model - Grading**
  - `ai_outputParser` input ← **Grading Result Parser**
- **Edge cases:** same token/format risks as plagiarism agent.

#### Node: OpenAI Model - Grading
- **Type/role:** `lmChatOpenAi`
- **Model:** `gpt-4o`
- **Options:** maxTokens 3000, temperature 0.5
- **Connections:** to AI Grading Agent via `ai_languageModel`

#### Node: Grading Result Parser
- **Type/role:** `Structured Output Parser`
- **Key configuration issue:** `schemaType` is “manual” but **no schema is provided** in the node parameters. Without a schema, parsing cannot reliably occur.
- **Connections:** to AI Grading Agent via `ai_outputParser`
- **Expected fields downstream (must be produced by parsed grading output):**
  - `overallScore`
  - `contentQuality`, `structureOrganization`, `evidenceSupport`, `writingQuality`
  - `strengths` (array), `areasForImprovement` (array), `suggestions` (array)
  - `detailedFeedback`
- **Edge cases:** This is a major functional gap: downstream nodes reference these fields and will fail if they don’t exist.

#### Node: Combine Grading Results
- **Type/role:** `Set` — computes final scoring and review flags.
- **Key expressions:**
  - `finalScore = max(0, overallScore * (1 - penaltyPercent/100))`
  - `needsReview = plagiarismScore > plagiarismThreshold OR overallScore < 50`
  - `exceptionType`:
    - `high_plagiarism` if above threshold
    - else `late_submission` if late
    - else `manual_review` if overallScore < 50
    - else `normal`
  - `gradedAt = now`
- **Connections:** Output → **Route by Exception Type**
- **Edge cases:**
  - If `overallScore` missing, finalScore becomes NaN and DB inserts/notifications break.

---

### Block 2.7 — Exception Routing & Alerts
**Overview:** Routes submissions based on exception type; sends Slack alerts for non-normal cases while still storing records on the “Normal” branch.  
**Nodes involved:**  
- Route by Exception Type  
- Alert - High Plagiarism  
- Alert - Late Submission  
- Alert - Manual Review Needed  
- Store Submission Record

#### Node: Route by Exception Type
- **Type/role:** `Switch` — four named outputs:
  - High Plagiarism, Late Submission, Manual Review, Normal
- **Condition:** `exceptionType` exact match
- **Connections:**
  - High Plagiarism → **Alert - High Plagiarism**
  - Late Submission → **Alert - Late Submission**
  - Manual Review → **Alert - Manual Review Needed**
  - Normal → **Store Submission Record**
- **Edge case (logic consequence):**
  - For exception cases, the flow **does not continue** to store/update gradebook/generate feedback unless you additionally connect those alert nodes onward. As built, only “normal” proceeds to persistence and feedback generation.

#### Node: Alert - High Plagiarism
- **Type/role:** `Slack` — alert message with concerns list and recommendation.
- **Channel:** `slackChannelAlerts` from config
- **Credentials:** Slack OAuth2
- **Terminal:** no outgoing connection
- **Edge cases:** Slack auth/channel ID invalid; message contains user data (FERPA considerations).

#### Node: Alert - Late Submission
- **Type/role:** `Slack` — late penalty details.
- **Channel:** alerts channel
- **Terminal**

#### Node: Alert - Manual Review Needed
- **Type/role:** `Slack` — low-score manual review notice.
- **Channel:** alerts channel
- **Terminal**

#### Node: Store Submission Record
- **Type/role:** `Postgres` — inserts submission grading record into `public.submissions`.
- **Columns stored (high level):**
  - Student/course/assignment identifiers and metadata
  - Submission text, timestamps, late status/penalty
  - Plagiarism score, overall and final scores, exception flags
  - Drive file id (`drive_file_id = {{$json.id}}` from Drive upload output)
- **Connections:** Output → **Update Gradebook**
- **Edge cases:**
  - Storing `submission_text` in DB may have privacy/data-retention implications.
  - Field mapping assumes Drive node returned `id` and grading produced all required fields.

---

### Block 2.8 — Gradebook Update, Feedback PDF, Distribution, Analytics
**Overview:** Updates gradebook, generates an HTML feedback document, converts to PDF via external API, uploads PDF to Drive, notifies instructors and student, and logs an analytics event.  
**Nodes involved:**  
- Update Gradebook  
- Generate Feedback HTML  
- Convert HTML to PDF  
- Upload Feedback PDF  
- Notify Instructors - Slack  
- Notify Instructors - Email  
- Send Student Feedback Email  
- Log Analytics Event

#### Node: Update Gradebook
- **Type/role:** `Postgres` — upsert into `gradebook`.
- **Query:** insert with `ON CONFLICT (student_id, course_id, assignment_id) DO UPDATE ...`
- **Replacements:** studentId, courseId, assignmentId, finalScore, gradedAt, graded_by = `AI_SYSTEM`
- **Connections:** Output → **Generate Feedback HTML**
- **Edge cases:**
  - `graded_by` is injected as `AI_SYSTEM` without quoting in replacements; may break SQL unless treated as literal string properly.
  - Conflict key must exist as unique constraint.

#### Node: Generate Feedback HTML
- **Type/role:** `HTML` — renders a styled HTML feedback report.
- **Key dependencies:** expects fields like:
  - `finalScore`, `penaltyPercent`, `isLate`, `daysLate`, `gradedAt`
  - criterion scores: `contentQuality`, `structureOrganization`, `evidenceSupport`, `writingQuality`
  - arrays: `strengths`, `areasForImprovement`, `suggestions`
  - `detailedFeedback`
- **Connections:** Output → **Convert HTML to PDF**
- **Edge cases:**
  - If arrays are missing/not arrays, `.map()` calls will throw expression errors.
  - HTML node outputs `html` field which is used by PDF conversion.

#### Node: Convert HTML to PDF
- **Type/role:** `HTTP Request` — calls html2pdf API and receives a file.
- **Key configuration:**
  - POST to `pdfConversionApiUrl`
  - JSON body includes `html` and PDF options (A4 + margins)
  - Response format: **file** (binary)
- **Connections:** Output → **Upload Feedback PDF**
- **Edge cases:**
  - html2pdf may require an API key; not provided here.
  - Large HTML may exceed service limits.
  - Binary handling must match what Drive upload expects next.

#### Node: Upload Feedback PDF
- **Type/role:** `Google Drive` — uploads PDF file.
- **Key configuration:**
  - Name: `feedback_<submissionId>_<studentId>.pdf`
  - Folder: `driveFolderId`
  - Simplify output: enabled (so fields like `webViewLink` are more accessible)
- **Connections:** Output → (fan-out)
  - **Notify Instructors - Slack**
  - **Notify Instructors - Email**
  - **Send Student Feedback Email**
  - **Log Analytics Event**
- **Edge cases:**
  - Must correctly map binary property from Convert HTML to PDF into Drive node input (not visible in JSON). If not set, upload fails or creates empty file.
  - Relies on `webViewLink` for Slack message.

#### Node: Notify Instructors - Slack
- **Type/role:** `Slack` — posts graded summary and PDF link.
- **Channel:** `slackChannelInstructors`
- **Message includes:** student name, course, assignment, final score, late flag, `webViewLink`
- **Edge cases:** uses emoji characters; otherwise standard.

#### Node: Notify Instructors - Email
- **Type/role:** `Gmail` — sends instructor email.
- **To:** placeholder `<Instructor Email Address>`
- **Subject:** `Assignment Graded: <courseId> - <assignmentId>`
- **Body:** summary + mentions feedback uploaded
- **Edge cases:** no attachment sent to instructor (only student path attempts attachment).

#### Node: Send Student Feedback Email
- **Type/role:** `Gmail` — sends feedback email to student.
- **To:** `studentEmail`
- **Email type:** text
- **Attachments:** configured as `attachmentsBinary` but **no binary property is specified**, so attachment may be missing.
- **Body includes:** top strengths and areas to improve (first 3 each)
- **Edge cases:**
  - If arrays missing, `.slice()` will error.
  - Need to set attachment binary property from PDF conversion or Drive download step.

#### Node: Log Analytics Event
- **Type/role:** `Postgres` — inserts into `public.analytics_events`.
- **Event:** `submission_graded`
- **Stores:** score, late flag, course/assignment/student, exception_type, plagiarism_score, timestamp
- **Edge cases:** schema must exist with compatible types.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Forms Webhook | Webhook | Entry point for form submissions | — | Workflow Configuration | ## Data Collection  \nFetches raw data from Google Sheets, SQL databases, and REST APIs using scheduled triggers.\n**Why** - Centralizes dispersed business metrics into a single processing pipeline. |
| LMS API Polling Schedule | Schedule Trigger | Time-based trigger to poll LMS | — | Fetch LMS Assignments | ## Data Collection  \nFetches raw data from Google Sheets, SQL databases, and REST APIs using scheduled triggers.\n**Why** - Centralizes dispersed business metrics into a single processing pipeline. |
| Workflow Configuration | Set | Central config constants and placeholders | Google Forms Webhook | Merge Form and API Data | ## Data Collection  \nFetches raw data from Google Sheets, SQL databases, and REST APIs using scheduled triggers.\n**Why** - Centralizes dispersed business metrics into a single processing pipeline. |
| Fetch LMS Assignments | HTTP Request | Pull pending submissions from LMS API | LMS API Polling Schedule | Merge Form and API Data | ## Data Collection  \nFetches raw data from Google Sheets, SQL databases, and REST APIs using scheduled triggers.\n**Why** - Centralizes dispersed business metrics into a single processing pipeline. |
| Merge Form and API Data | Merge | Combine submission stream with config stream | Fetch LMS Assignments, Workflow Configuration | Normalize Submission Data | ## Data Transformation  \nCleanses, normalizes, and calculates KPIs using JavaScript code and mathematical functions.\n**Why** - Ensures data consistency and generates standardized metrics for cross-functional comparison. |
| Normalize Submission Data | Set | Normalize fields into a standard schema | Merge Form and API Data | Validate Submission | ## Data Transformation  \nCleanses, normalizes, and calculates KPIs using JavaScript code and mathematical functions.\n**Why** - Ensures data consistency and generates standardized metrics for cross-functional comparison. |
| Validate Submission | Code | Validate required fields and email format | Normalize Submission Data | Check Validation Status | ## Data Transformation  \nCleanses, normalizes, and calculates KPIs using JavaScript code and mathematical functions.\n**Why** - Ensures data consistency and generates standardized metrics for cross-functional comparison. |
| Check Validation Status | IF | Branch: valid vs invalid | Validate Submission | Check Duplicate Submission; Handle Validation Error | ## Data Transformation  \nCleanses, normalizes, and calculates KPIs using JavaScript code and mathematical functions.\n**Why** - Ensures data consistency and generates standardized metrics for cross-functional comparison. |
| Handle Validation Error | Set | Prepare webhook error response payload | Check Validation Status (false) | Respond to Form Submission |  |
| Respond to Form Submission | Respond to Webhook | Return HTTP 400 to webhook caller | Handle Validation Error | — |  |
| Check Duplicate Submission | Postgres | Detect duplicate submissions | Check Validation Status (true) | Is Duplicate? | ## Data Transformation  \nCleanses, normalizes, and calculates KPIs using JavaScript code and mathematical functions.\n**Why** - Ensures data consistency and generates standardized metrics for cross-functional comparison. |
| Is Duplicate? | IF | Stop duplicates; continue non-duplicates | Check Duplicate Submission | Calculate Deadline Penalty (false branch only) | ## Data Transformation  \nCleanses, normalizes, and calculates KPIs using JavaScript code and mathematical functions.\n**Why** - Ensures data consistency and generates standardized metrics for cross-functional comparison. |
| Calculate Deadline Penalty | Code | Compute late status and penalty | Is Duplicate? (not duplicate) | Rate Limit Delay | ## Data Transformation  \nCleanses, normalizes, and calculates KPIs using JavaScript code and mathematical functions.\n**Why** - Ensures data consistency and generates standardized metrics for cross-functional comparison. |
| Rate Limit Delay | Wait | Add delay to reduce rate limits | Calculate Deadline Penalty | Upload Submission to Drive | ## Data Transformation  \nCleanses, normalizes, and calculates KPIs using JavaScript code and mathematical functions.\n**Why** - Ensures data consistency and generates standardized metrics for cross-functional comparison. |
| Upload Submission to Drive | Google Drive | Store submission artifact in Drive | Rate Limit Delay | Plagiarism Check Agent | ## AI Analysis  \nRoutes transformed data to OpenAI models for trend analysis, forecasting, and narrative generation.\n**Why** - Provides intelligent insights beyond basic calculations, highlighting patterns humans might overlook. |
| Plagiarism Check Agent | LangChain Agent | Plagiarism scoring and concerns extraction | Upload Submission to Drive | AI Grading Agent | ## AI Analysis  \nRoutes transformed data to OpenAI models for trend analysis, forecasting, and narrative generation.\n**Why** - Provides intelligent insights beyond basic calculations, highlighting patterns humans might overlook. |
| OpenAI Model - Plagiarism | OpenAI Chat Model | LLM backend for plagiarism agent | — | Plagiarism Check Agent (ai_languageModel) | ## AI Analysis  \nRoutes transformed data to OpenAI models for trend analysis, forecasting, and narrative generation.\n**Why** - Provides intelligent insights beyond basic calculations, highlighting patterns humans might overlook. |
| Plagiarism Result Parser | Structured Output Parser | Enforce plagiarism JSON schema | — | Plagiarism Check Agent (ai_outputParser) | ## AI Analysis  \nRoutes transformed data to OpenAI models for trend analysis, forecasting, and narrative generation.\n**Why** - Provides intelligent insights beyond basic calculations, highlighting patterns humans might overlook. |
| AI Grading Agent | LangChain Agent | Rubric-based grading + feedback generation | Plagiarism Check Agent | Combine Grading Results | ## AI Analysis  \nRoutes transformed data to OpenAI models for trend analysis, forecasting, and narrative generation.\n**Why** - Provides intelligent insights beyond basic calculations, highlighting patterns humans might overlook. |
| OpenAI Model - Grading | OpenAI Chat Model | LLM backend for grading agent | — | AI Grading Agent (ai_languageModel) | ## AI Analysis  \nRoutes transformed data to OpenAI models for trend analysis, forecasting, and narrative generation.\n**Why** - Provides intelligent insights beyond basic calculations, highlighting patterns humans might overlook. |
| Grading Result Parser | Structured Output Parser | Enforce grading JSON schema (missing schema in config) | — | AI Grading Agent (ai_outputParser) | ## AI Analysis  \nRoutes transformed data to OpenAI models for trend analysis, forecasting, and narrative generation.\n**Why** - Provides intelligent insights beyond basic calculations, highlighting patterns humans might overlook. |
| Combine Grading Results | Set | Apply penalty; compute final score and exception type | AI Grading Agent | Route by Exception Type | ## AI Analysis  \nRoutes transformed data to OpenAI models for trend analysis, forecasting, and narrative generation.\n**Why** - Provides intelligent insights beyond basic calculations, highlighting patterns humans might overlook. |
| Route by Exception Type | Switch | Route exceptions vs normal path | Combine Grading Results | Alert - High Plagiarism; Alert - Late Submission; Alert - Manual Review Needed; Store Submission Record | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Alert - High Plagiarism | Slack | Alert instructors/admins of plagiarism risk | Route by Exception Type | — | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Alert - Late Submission | Slack | Alert on late submissions and penalty | Route by Exception Type | — | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Alert - Manual Review Needed | Slack | Alert for manual review | Route by Exception Type | — | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Store Submission Record | Postgres | Persist submission + grading results | Route by Exception Type (Normal) | Update Gradebook | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Update Gradebook | Postgres | Upsert grade into gradebook | Store Submission Record | Generate Feedback HTML | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Generate Feedback HTML | HTML | Render feedback report as HTML | Update Gradebook | Convert HTML to PDF | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Convert HTML to PDF | HTTP Request | Convert HTML to PDF via external service | Generate Feedback HTML | Upload Feedback PDF | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Upload Feedback PDF | Google Drive | Upload PDF to Drive and expose link | Convert HTML to PDF | Notify Instructors - Slack; Notify Instructors - Email; Send Student Feedback Email; Log Analytics Event | ## Dashboard Generation  \nPopulates pre-designed templates with processed data and AI-generated insights.\n**Why** - Creates stakeholder-specific views without requiring individual report customization. |
| Notify Instructors - Slack | Slack | Post grading summary to instructor channel | Upload Feedback PDF | — |  |
| Notify Instructors - Email | Gmail | Email instructors about graded submission | Upload Feedback PDF | — |  |
| Send Student Feedback Email | Gmail | Email student with feedback (intended PDF attachment) | Upload Feedback PDF | — |  |
| Log Analytics Event | Postgres | Store analytics event | Upload Feedback PDF | — |  |
| Sticky Note | Sticky Note | Documentation (canvas note) | — | — |  |
| Sticky Note1 | Sticky Note | Documentation (canvas note) | — | — |  |
| Sticky Note2 | Sticky Note | Documentation (canvas note) | — | — |  |
| Sticky Note3 | Sticky Note | Documentation (canvas note) | — | — |  |
| Sticky Note4 | Sticky Note | Documentation (canvas note) | — | — |  |
| Sticky Note5 | Sticky Note | Documentation (canvas note) | — | — |  |
| Sticky Note6 | Sticky Note | Documentation (canvas note) | — | — |  |

> Note: Sticky notes in this workflow describe a *business intelligence dashboard* scenario and do not match the actual assignment grading workflow logic. They are included verbatim in the table as requested.

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger: “Google Forms Webhook”**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: `assignment-submission`
   - Response mode: **Last Node**
   - Save and note the test/production webhook URLs.

2) **Create Trigger: “LMS API Polling Schedule”**
   - Node type: **Schedule Trigger**
   - Run every **15 minutes**

3) **Create “Workflow Configuration” (Set node)**
   - Add fields:
     - `lmsApiUrl` (string)
     - `lmsApiKey` (string) (prefer env var/credential in production)
     - `plagiarismThreshold` (number) = 0.75
     - `latePenaltyPercent` (number) = 10
     - `driveFolderId` (string)
     - `slackChannelInstructors` (string or channel ID)
     - `slackChannelAlerts` (string or channel ID)
     - `pdfConversionApiUrl` (string) = `https://api.html2pdf.app/v1/generate`
     - `maxRetries` (number) = 3
     - `rateLimitDelayMs` (number) = 1000
   - Enable **Include Other Fields**.

4) **Connect triggers**
   - Connect **Google Forms Webhook → Workflow Configuration**
   - Connect **LMS API Polling Schedule → Fetch LMS Assignments** (created next)

5) **Create “Fetch LMS Assignments” (HTTP Request)**
   - Method: **GET** (default)
   - URL: `{{ $json.lmsApiUrl }}/assignments/submissions` (or reference config node directly)
   - Add query params: `status=pending`, `limit=50`
   - Authentication: **Header Auth** (or generic)
   - Header: `Authorization: Bearer <lmsApiKey>`
   - Timeout: 30000 ms

6) **Create “Merge Form and API Data” (Merge)**
   - Connect **Fetch LMS Assignments → Merge (Input 0)**
   - Connect **Workflow Configuration → Merge (Input 1)**
   - Choose a merge mode that matches your intent:
     - If you want to **attach config to every LMS item**, use a pattern like **Merge by position** with config replicated, or avoid Merge and instead reference config node via expressions.
   - (Recommended fix) Also connect the **Webhook output** carrying submission payload into this merge or a parallel normalization path.

7) **Create “Normalize Submission Data” (Set)**
   - Add mappings for:
     - `studentId`, `studentEmail`, `studentName`, `courseId`, `assignmentId`
     - `submissionText`, `submissionFile`, `submittedAt`, `deadline`, `submissionId`
   - Enable **Include Other Fields**.
   - Connect **Merge → Normalize**.

8) **Create “Validate Submission” (Code)**
   - Paste validation JS (required fields + email regex).
   - Connect **Normalize → Validate**.

9) **Create “Check Validation Status” (IF)**
   - Condition: `{{ $json.isValid }} equals true` (recommended over referencing other node’s `.item`)
   - True branch → **Check Duplicate Submission**
   - False branch → **Handle Validation Error**

10) **Create “Handle Validation Error” (Set)**
   - Fields: `status`, `message`, `timestamp`
   - Connect from IF false branch.

11) **Create “Respond to Form Submission” (Respond to Webhook)**
   - Response code: **400**
   - Response: JSON: `{ success: false, message: {{$json.message}}, errors: {{$json.validationErrors}} }`
   - Connect **Handle Validation Error → Respond to Webhook**
   - (Recommended) Add an IF before this node to ensure it runs only for webhook-triggered executions.

12) **Create “Check Duplicate Submission” (Postgres)**
   - Configure PostgreSQL credentials in n8n (host/db/user/password/SSL).
   - Operation: **Execute Query**
   - Query: `SELECT COUNT(*) as count FROM submissions WHERE student_id = $1 AND assignment_id = $2 AND course_id = $3`
   - Provide parameters safely (use the node’s parameter UI, not a comma-joined string).
   - Connect from IF true branch.

13) **Create “Is Duplicate?” (IF)**
   - Condition: `{{ Number($json.count) }} > 0`
   - False (not duplicate) → **Calculate Deadline Penalty**
   - True → optionally add a “Stop and Respond” path (log or webhook response)

14) **Create “Calculate Deadline Penalty” (Code)**
   - Compute late status, `daysLate`, `penaltyPercent` using config `latePenaltyPercent`.
   - Connect **Is Duplicate? (false) → Calculate Deadline Penalty**.

15) **Create “Rate Limit Delay” (Wait)**
   - Wait amount: `{{ $('Workflow Configuration').first().json.rateLimitDelayMs }}`
   - Connect **Calculate Deadline Penalty → Rate Limit Delay**.

16) **Create “Upload Submission to Drive” (Google Drive)**
   - Credentials: Google Drive OAuth2
   - Operation: upload/create file
   - Folder: `driveFolderId`
   - File name: `submission_<submissionId>_<studentId>.txt`
   - Ensure you actually pass file content:
     - Create binary from `submissionText` (via “Move/Convert to File” pattern) and map it to Drive upload.
   - Connect **Rate Limit Delay → Upload Submission to Drive**.

17) **Create “Plagiarism Check Agent” (LangChain Agent)**
   - Add prompt text containing submission text + course/assignment
   - Add a system message describing plagiarism scoring
   - Enable structured output parsing

18) **Create “OpenAI Model - Plagiarism”**
   - Credentials: OpenAI API key
   - Model: `gpt-4o`
   - Connect **OpenAI Model - Plagiarism → Plagiarism Check Agent** via `ai_languageModel`

19) **Create “Plagiarism Result Parser”**
   - Provide manual JSON schema with `plagiarismScore`, `concerns[]`, `recommendation`
   - Connect **Plagiarism Result Parser → Plagiarism Check Agent** via `ai_outputParser`

20) **Create “AI Grading Agent” (LangChain Agent)**
   - Prompt includes: submission + plagiarismScore + late status
   - System message: rubric and required outputs
   - Enable structured output parsing
   - Connect **Plagiarism Check Agent → AI Grading Agent** (main)

21) **Create “OpenAI Model - Grading”**
   - Model: `gpt-4o`, maxTokens ~3000, temperature ~0.5
   - Connect to **AI Grading Agent** via `ai_languageModel`

22) **Create “Grading Result Parser”**
   - Add a **manual schema** (required to match downstream HTML):
     - `overallScore` (number 0-100)
     - `contentQuality`, `structureOrganization`, `evidenceSupport`, `writingQuality` (numbers)
     - `strengths`, `areasForImprovement`, `suggestions` (arrays of strings)
     - `detailedFeedback` (string)
   - Connect to **AI Grading Agent** via `ai_outputParser`

23) **Create “Combine Grading Results” (Set)**
   - Compute `finalScore`, `scoreBeforePenalty`, `needsReview`, `exceptionType`, `gradedAt`
   - Connect **AI Grading Agent → Combine Grading Results**

24) **Create “Route by Exception Type” (Switch)**
   - Rules on `exceptionType`:
     - `high_plagiarism`, `late_submission`, `manual_review`, `normal`
   - Connect **Combine Grading Results → Switch**

25) **Create Slack alert nodes**
   - “Alert - High Plagiarism”, “Alert - Late Submission”, “Alert - Manual Review Needed”
   - Credentials: Slack OAuth2
   - Channel: `slackChannelAlerts`
   - Connect each switch output to its alert node
   - (Recommended) After alerting, also connect to persistence/feedback if you still want grading artifacts generated for exceptions.

26) **Create “Store Submission Record” (Postgres Insert)**
   - Table: `public.submissions`
   - Map all relevant columns (student/course/assignment, scores, flags, timestamps, drive_file_id, text)
   - Connect **Switch (Normal) → Store Submission Record**

27) **Create “Update Gradebook” (Postgres Execute Query)**
   - Upsert query into `gradebook`
   - Connect **Store Submission Record → Update Gradebook**

28) **Create “Generate Feedback HTML” (HTML node)**
   - Paste HTML template and ensure referenced fields exist.
   - Connect **Update Gradebook → Generate Feedback HTML**

29) **Create “Convert HTML to PDF” (HTTP Request)**
   - POST to `pdfConversionApiUrl`
   - Body contains HTML and PDF options
   - Response: file/binary
   - Connect **Generate Feedback HTML → Convert HTML to PDF**

30) **Create “Upload Feedback PDF” (Google Drive)**
   - Upload binary from previous node
   - Name: `feedback_<submissionId>_<studentId>.pdf`
   - Folder: `driveFolderId`
   - Connect **Convert HTML to PDF → Upload Feedback PDF**

31) **Create distribution nodes (fan-out)**
   - “Notify Instructors - Slack” (channel: instructors channel)
   - “Notify Instructors - Email” (Gmail OAuth2; to instructor address)
   - “Send Student Feedback Email” (Gmail OAuth2; to studentEmail; attach PDF binary properly)
   - “Log Analytics Event” (Postgres insert into analytics_events)
   - Connect **Upload Feedback PDF →** each of these nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes describe an automated business intelligence reporting/dashboard workflow (Google Sheets + KPIs + executive dashboards). | These notes do not align with the actual assignment grading automation implemented by the nodes. |
| Workflow contains placeholders for LMS URL/key, Drive folder ID, and Slack channels, which must be replaced before activation. | Workflow Configuration node fields. |
| Several wiring/implementation gaps likely prevent successful end-to-end runs (webhook payload not processed, grading parser schema missing, attachment not mapped, exception branches don’t continue). | Review blocks 2.2, 2.6, 2.7, 2.8. |